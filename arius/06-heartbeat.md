# §6 Heartbeat

> **Status:** 🟢 **Documentado e validado empiricamente.** Cartografia cross-repo completa em 8 áreas (2026-05-02), schema canônico mapeado, pipeline `ProcessHeartbeat` documentado linha-a-linha, drift histórico registrado, 4 gaps identificados e registrados como issues ([ARI-210](https://linear.app/arius-ai/issue/ARI-210) stale detection resolvido em 2026-05-06 / [ARI-211](https://linear.app/arius-ai/issue/ARI-211) error_count cleanup resolvido em 2026-05-05 / [ARI-212](https://linear.app/arius-ai/issue/ARI-212) try/except no loop resolvido em 2026-05-06 / [ARI-213](https://linear.app/arius-ai/issue/ARI-213) enrichment de agent_snapshots resolvido em 2026-05-09). **Segundo conceito da plataforma Arius a alcançar 🟢 estrito.**
>
> **Última atualização:** 2026-05-09 (ARI-213 — enrichment de agent_snapshots populado via background task)

---

## 6.1 — O que é

**Heartbeat** é o pulse periódico que o agent envia ao Observatory para reportar **estado operacional vivo** — combinação de:

- **Liveness signal** ("estou aqui, ativo")
- **Snapshot de subdomínios** (Circuit Breakers, counters de requests/errors, version)
- **Trigger de side effects no Observatory** (atualização de `agent.status`, persistência de history, audit log, agregação de counters)

**Natureza transversal:** diferente de Circuit Breakers (conceito de domínio com semântica própria), Heartbeat é **transporte** que carrega snapshots de outros conceitos. Por isso §6 referencia §3 (CB), §1 (HealthScore quando documentado), §7 (Telemetry quando documentado) sem repetir o que já está documentado nessas seções.

### Diferença com Push de transições ([ARI-164](https://linear.app/arius-ai/issue/ARI-164))

Heartbeat não é o único canal agent → Observatory. Push de transições (`POST /agents/{slug}/circuit-breakers/transitions`) complementa heartbeat para **eventos discretos rápidos** (transições de CB que podem acontecer em <60s). Tabela comparativa em §6.4.

### Por que 60s default?

Trade-off entre:

- **Frequência alta** (5s, 10s): mais responsivo, mais carga no Observatory, mais rows em `agent_circuit_breaker_history`
- **Frequência baixa** (300s = 5min): leve, mas janela cega grande para detectar problemas

60s é compromisso típico de healthcheck de infra (mesmo padrão de Kubernetes liveness probes default).

---
## 6.2 — Como implementamos

### Schema canônico do payload (cross-repo)

**agent-standard envia** (`src/domain/contracts/heartbeat.py`):

```python
class CounterSnapshot(BaseModel):
    requests_total: int = Field(ge=0)
    errors_total: int = Field(ge=0)
    process_started_at: str = Field(description="ISO 8601 UTC timestamp; changes on process restart")


class CircuitBreakerSnapshot(BaseModel):
    name: str
    state: CBState                              # closed | open | half-open
    fail_count: int = Field(ge=0)
    last_state_change: str | None = None
    fail_max: int = Field(ge=1)                 # ARI-209 Required
    reset_timeout_seconds: int = Field(ge=1)    # ARI-209 Required


class HeartbeatPayload(BaseModel):
    version: str                                # Required
    circuit_breakers: list[CircuitBreakerSnapshot]  # Required (lista, pode ser vazia)
    counters: CounterSnapshot                   # Required
```

**Observatory aceita** (`src/infrastructure/schemas.py`):

```python
class CountersRequest(BaseModel):
    requests_total: int = Field(ge=0)
    errors_total: int = Field(ge=0)
    process_started_at: datetime                # tipo DIFERENTE: datetime, não str


class CircuitBreakerStateRequest(BaseModel):
    name: str
    state: Literal["closed", "open", "half-open"]
    fail_count: int = Field(default=0, ge=0)
    last_state_change: Optional[datetime] = None
    fail_max: Optional[int] = Field(default=None, ge=1)            # ARI-209 Optional retro-compat
    reset_timeout_seconds: Optional[int] = Field(default=None, ge=1)


class HeartbeatRequest(BaseModel):
    version: str | None = None                  # Optional
    circuit_breakers: list[CircuitBreakerStateRequest] | None = None
    counters: CountersRequest | None = None
```

Diferença Required (cliente) vs Optional (servidor) é **decisão arquitetural deliberada** para retro-compat: agents antigos que ainda não enviam todos os campos não quebram.

### Frequência e ciclo de vida (loop de envio)

`src/presentation/app.py:173-196` — `_heartbeat_loop`:

```python
async def _heartbeat_loop(client, agent_slug, agent_version, interval_seconds) -> None:
    while True:
        await asyncio.sleep(interval_seconds)            # sleep PRIMEIRO
        payload = HeartbeatPayload(...)
        await client.heartbeat(agent_slug, payload.model_dump(mode="json"))
```

**Configuração** (`settings.py:90`):

```python
heartbeat_interval_seconds: int = Field(
    default=60,
    ge=5,
    le=300,
    description="Intervalo entre heartbeats ao Observatory (segundos). Default 60. Range válido: 5-300."
)
```

- **Env var:** `HEARTBEAT_INTERVAL_SECONDS` (resolvida via pydantic-settings)
- **Range:** 5-300s confirmado, default 60s
- **Origem:** [ARI-152](https://linear.app/arius-ai/issue/ARI-152)

**Ordem do startup com importância semântica:**

1. Checkpointer abre pool Postgres (CB `postgres` registrado)
2. `client.register(...)` — POST /agents/register
3. `_seed_cbs_from_observatory(...)` ([ARI-165](https://linear.app/arius-ai/issue/ARI-165))
4. `wire_push_listeners(...)` ([ARI-164](https://linear.app/arius-ai/issue/ARI-164)) — **DEPOIS do seed** (senão seed dispara push fantasma — lição arquitetural §3.2)
5. Cria task `_heartbeat_loop` via `asyncio.create_task`
6. Primeiro heartbeat: só sai depois de `interval_seconds` de uptime (sleep antes de send)

**Ciclo de vida da task:**

- Iniciada no lifespan
- Cancelada via `task.cancel()` no shutdown
- **Sem jitter na cadência** (existe apenas no retry interno do client)
- **Sem backoff na cadência** (idem)

**Recuperação se loop crashar:** o cliente captura exceptions de rede; o loop em si **tem try/except envolvendo a iteração** (ARI-212 resolvida 2026-05-06). Falha em uma iteração emite log estruturado (`heartbeat_iteration_failed`) + Langfuse event (defesa em profundidade) e próxima iteração roda normal. Loop NUNCA morre silenciosamente — apenas via `task.cancel()` no shutdown.

---
### Pipeline completo no Observatory (`ProcessHeartbeat`)

`POST /agents/{slug}/heartbeat` → `ProcessHeartbeat.execute` (`src/application/agents.py:180-297`).

**8 passos em ordem:**

1. `agent_repo.get_agent_by_slug(slug)` → 404 se None
2. `agent.last_heartbeat_at = now`
3. Se `payload.version ≠ None` → `agent.version = payload.version`
4. `agent.status = compute_status(last_heartbeat_at, now, timeout_seconds)` (StatusEngine)
5. `agent_repo.update_agent(agent)` (write síncrona, parte da transação principal)
6. **Bloco try isolado** — Se `payload.circuit_breakers and self._cb_repo`:
   - Stamp com `agent_id` real + `now`
   - `cb_repo.upsert_many(stamped)` — atualiza `agent_circuit_breakers` (state atual)
   - `cb_repo.append_transitions(transitions)` — append em `agent_circuit_breaker_history`
   - `except Exception: logger.warning("circuit_breaker upsert failed...")` — best-effort
7. **Bloco try isolado** — Se `payload.counters` (com `snapshot_repo` + `last_counters_repo`):
   - `_process_counters` → delta + `create_snapshot` OU audit anomaly
   - `last_counters_repo.upsert(...)` — sempre, mesmo em anomalia
   - `except Exception: logger.warning("counters processing failed...")` — best-effort
8. **Bloco try isolado** — Append em `audit_log` action=`heartbeat` com payload completo (slug, status, version, CBs com `fail_max`/`reset_timeout`, counters)
   - `except Exception: logger.warning("audit_log write failed...")` — best-effort

**Garantias transacionais:**

- **Sem Unit of Work explícito.** Cada bloco try/except é independente.
- A SQLAlchemy session do `get_db` é commit-per-request (FastAPI dependency). Falha "silenciosa" via `logger.warning` NÃO faz rollback — apenas suprime a exception. Inserts já feitos antes da falha permanecem se a session committar.
- **Caso patológico documentado:** `agent.status` atualizado (passo 5) mas CBs falham (passo 6). Status fica avançado vs CBs no banco.

**Decisão arquitetural deliberada (Decisão 5 da §6):** **best-effort em todos os ramos, não Unit of Work atômica**. Rationale: heartbeat é **fonte de informação operacional, não transação de domínio**. "Heartbeat parcial chegou" é melhor que "heartbeat inteiro perdido por causa de 1 campo". Trade-off aceito conscientemente: falha parcial possível (status atualizado mas CBs não, ou CBs sim mas counters não). Quem precisa de garantia transacional não usa heartbeat — usa endpoint específico do conceito.

**Retorno ao agent:** `AgentResponse` completo (id, system_id, slug, name, description, version, base_url, status, last_heartbeat_at, registered_at). Agent atual **não consome** o body (apenas `await client.heartbeat` sem inspeção — `app.py:196`).

### Confiabilidade e recuperação (cliente)

**Cliente HTTP** (`observatory_client.py`):

```python
httpx.AsyncClient(base_url=..., headers=..., timeout=10.0)   # 10s default
```

**Retry policy:**

```python
AsyncRetrying(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=0.5, max=2) + wait_random(0, 0.1),
    reraise=True,
)
```

- **3 tentativas, exponential 0.5-2s + random jitter 0-0.1s**
- Captura `HTTPStatusError` e `RequestError` — ambos retried
- Após 3 falhas: `_post_silently` captura Exception → `logger.warning("Observatory offline...")` → retorna `None`
- **Loop NUNCA crasha** — `client.heartbeat()` nunca propaga exception

**Comportamento se Observatory está down:**

| Cenário | Comportamento |
| -- | -- |
| Connection refused / timeout | 3 retries → log warning → drop silencioso (return None) |
| HTTP 5xx | Retry → drop silencioso |
| HTTP 4xx | Retry (mesma policy — sem distinção) → drop |
| Observatory volta | Próximo heartbeat re-sincroniza estado completo (CBs, counters absolutos) |

**Buffer/idempotência:**

- **Sem buffer local de reenvio.** Heartbeats perdidos são perdidos.
- **Idempotência implícita parcial:**
  - `last_heartbeat_at`: sobrescrito com `now` do **servidor** (não do payload) → re-envio só atrasa o relógio do servidor
  - CBs: `upsert_many` por `(agent_id, name)` → idempotente
  - Counters: delta calculado contra `agent_last_counters` → re-envio do MESMO snapshot gera delta=0 (não é anomalia)
  - `agent_circuit_breaker_history`: **não-idempotente** — cada heartbeat appenda nova linha por CB, sem dedup
  - `audit_log`: idem, append-only sem dedup

Sem métricas internas no agent sobre heartbeats falhos. Apenas log.

### Counters: subdomínio interno do payload

Counters fazem parte do payload de heartbeat mas têm vida própria.

**No agent** (`request_counters.py`):

- Module-level singleton: `_counters: dict[str, int] = {"requests_total": 0, "errors_total": 0}` + `_process_started_at: datetime`
- `threading.Lock` para incrementos
- **Acumulado cumulativo desde o boot do processo. NUNCA reseta em runtime.**
- `increment_request(outcome)`:
  - `requests_total += 1` sempre
  - `errors_total += 1` só se `outcome ∈ {ERROR_LLM, ERROR_VALIDATION}` (decidido em [ARI-150](https://linear.app/arius-ai/issue/ARI-150))
  - `refused_out_of_scope`, `degraded_fallback_prompt` **NÃO contam** como erro
- Reset detectado pelo Observatory via mudança de `process_started_at` (restart do agent)
- Chamado em `agent_service.py:195`

**No Observatory** (`agents.py:299-409`):

- **Persistência em 2 tabelas:**
  - `agent_last_counters` (1 linha por agente, `ON CONFLICT DO UPDATE` → upsert atômico, multi-replica safe)
  - `agent_snapshots` (delta por heartbeat — `period_start`, `period_end`, `total_traces=delta.requests`, `error_count=delta.errors`)

**Lógica de delta** (`agents.py:340-376`):

| Caso | Comportamento |
| -- | -- |
| `previous is None` | Primeiro heartbeat — delta = absoluto, `is_reset=False` |
| `current.process_started_at ≠ previous.process_started_at` | Reset detectado — delta = absoluto novo, `is_reset=True` |
| Caso normal | `delta = current - previous` |
| `delta.requests < 0` ou `delta.errors < 0` | Anomalia — audit `action="agent_counters_anomaly"`, NÃO cria snapshot |
| Sempre | `last_counters_repo.upsert(current)` |

**Decisões arquiteturais:**

- **Best-effort O2** ([ARI-154](https://linear.app/arius-ai/issue/ARI-154)): falha em counters não bloqueia heartbeat (try/except isolado em passo 7).
- **Multi-replica safety** via Postgres `ON CONFLICT DO UPDATE`.
- **Granularidade:** snapshot por heartbeat (60s default). Não há janela rolling explícita — o "intervalo" é o gap entre heartbeats consecutivos.

> **Gap resolvido — [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (Medium 4pts, 09/05/2026)**
>
> Os 3 campos `avg_latency_ms`, `p95_latency_ms`, `total_cost_usd` em `agent_snapshots` são populados via background task post-heartbeat. Use case `EnrichSnapshotKPIs` em `src/application/enrich_snapshot.py` consome traces Langfuse v4 via `kpi_calculator` e faz UPDATE no snapshot recém-criado.
>
> Pattern session-per-task (ARI-191) primeira aplicação consciente no Observatory. D-A: bg task vs síncrono — escolhido bg task para preservar Decisão 5 ("heartbeat é informação, não transação"). D-B: lifecycle no shutdown não tratado, aceitável porque enrichment é informação histórica best-effort (próximo heartbeat re-cria snapshot ~60s depois).
>
> Comentário enganoso "TECH DEBT: virá de OTel Metrics no futuro" removido. Fonte real é Langfuse v4, não OTel — não há roadmap planejado para OTel Metrics.

---
### Push transitions (relação com heartbeat)

| Aspecto | Push ([ARI-164](https://linear.app/arius-ai/issue/ARI-164)) | Heartbeat |
| -- | -- | -- |
| Trigger | `pybreaker.state_change` | Loop a cada 60s |
| Endpoint | `POST /agents/{slug}/circuit-breakers/transitions` (202) | `POST /agents/{slug}/heartbeat` (200) |
| Retry | Nenhum (Decisão 3 ADR-164) | 3× exponencial |
| Falha | `logger.warning(extra={"fallback": "heartbeat_will_eventually_sync"})` | Drop silencioso após 3 |
| Persistência | `agent_circuit_breaker_history` (append) + `audit_log` | `agent_circuit_breakers` (state atual) + `agent_circuit_breaker_history` (append) |
| Atualiza state atual? | Não | **Sim (`upsert_many`)** |
| Schema | `name, from_state, to_state, fail_count, transitioned_at, last_error_type, last_error_message` | `CircuitBreakerSnapshot` (sem `from_state`, sem error info) |

**Cenário de divergência:**

- Push `open → closed` falha → history tem o estado pré-falha + heartbeat seguinte traz `state=closed`
- **Fonte de verdade do state atual = heartbeat** (sempre)
- **Fonte de verdade do histórico** = combinação push + heartbeat. Push falhado = lacuna no history.
- `last_error_type` / `last_error_message` SÓ chegam via push. Heartbeat não os carrega — se push falha, esse contexto se perde.

### Endpoints derivados (consumidores do heartbeat)

**No agent-standard** (`src/presentation/api/health.py`):

| Endpoint | Tipo | Side effects | Resposta |
| -- | -- | -- | -- |
| `GET /health` | Liveness, sem I/O | nenhum | `HealthResponse` (status="ok", version, agent_slug, llm_provider, llm_circuit_breaker, langfuse_connected) |
| `GET /ready` ([ARI-152](https://linear.app/arius-ai/issue/ARI-152)) | Readiness ativa | probes `checkpointer.get`, `observatory.health`, lê CB states locais | `ReadinessResponse` (status: ready/degraded/not_ready), HTTP 200/200/503 |

`/ready` agrega: fail em qualquer check → 503 `not_ready`; warn → 200 `degraded`; tudo ok/skipped → 200 `ready`.

**No Observatory:**

| Endpoint | Retorno |
| -- | -- |
| `GET /agents/{slug}` | `AgentResponse` (status, last_heartbeat_at, version, ...) |
| `GET /agents/{slug}/status` | `AgentStatusResponse` (status, last_heartbeat_at, circuit_breakers) |
| `GET /agents/{slug}/circuit-breakers` | `list[CircuitBreakerStateResponse]` |
| `GET /intelligence/agents/{slug}/health-score` | `HealthScoreResponse` (calcula `heartbeat_age_s = now - last_heartbeat_at` on demand) |

### Stale detection — resolvida via background sweep

`compute_status` (StatusEngine) continua sendo uma função pura chamada dentro de `ProcessHeartbeat.execute` (`agents.py:191`) quando heartbeats chegam. Quando heartbeats param, o Observatory agora tem sweep server-side.

**Stale detection ativa via background sweep (ARI-210 resolvida 2026-05-06):**

`SweepStaleAgents` em `src/application/sweep_stale_agents.py` é executado por background task no lifespan a cada `SWEEP_STALE_AGENTS_INTERVAL_SECONDS` (default 30s). Query usa índice composto `ix_agents_status_last_heartbeat_at`:

`WHERE status='healthy' AND last_heartbeat_at < now - HEARTBEAT_TIMEOUT_SECONDS`

Cada agent marcado como `unhealthy` gera audit_log com action `agent_marked_unhealthy` contendo `previous_status`, `last_heartbeat_at`, `heartbeat_age_seconds`, `timeout_threshold_seconds`. Loop sobrevive a exceptions (lição ARI-212).

Latência máxima de detecção: `HEARTBEAT_TIMEOUT_SECONDS + SWEEP_STALE_AGENTS_INTERVAL_SECONDS` = 150s default.

**Convenção `timeout = 2× interval` (Decisão 3 da §6):** documentada explicitamente para proteger contra alteração silenciosa em uma das pontas. Agent default 60s × 2 = Observatory threshold 120s. Quem alterar uma ponta deve avaliar a outra.

### Audit log: action=`heartbeat` como contrato implícito

Cada heartbeat gera um `audit_log` entry com payload completo do que o agente enviou (não pós-merge). Útil para reconstruir cronologia. **Capacidade implícita do sistema** que não estava documentada antes desta cartografia — vale conhecimento explícito porque cliente Pulso pode usar para forense de incidentes.

### Decisões registradas

- [ARI-149](https://linear.app/arius-ai/issue/ARI-149) — schema `HeartbeatPayload` Pydantic + counters
- [ARI-150](https://linear.app/arius-ai/issue/ARI-150) — vocabulário `AgentOutcome` (define o que conta como erro em counters)
- [ARI-152](https://linear.app/arius-ai/issue/ARI-152) — `heartbeat_interval_seconds` configurável + endpoint `/ready` + `HEARTBEAT_TIMEOUT_SECONDS=120`
- [ARI-154](https://linear.app/arius-ai/issue/ARI-154) — persistência de `agent_last_counters` + delta + audit anomaly
- [ARI-164](https://linear.app/arius-ai/issue/ARI-164) — push de transição CB (complementar — fonte de verdade de state atual continua sendo heartbeat)
- [ARI-165](https://linear.app/arius-ai/issue/ARI-165) — seed de CBs no startup via Observatory (consumidor indireto de info que heartbeat também transporta)
- [ARI-209](https://linear.app/arius-ai/issue/ARI-209) — propagação de `fail_max` / `reset_timeout_seconds` via heartbeat
- [ARI-210](https://linear.app/arius-ai/issue/ARI-210) (Done — mergeada 2026-05-06) — stale detection via background sweep (`SweepStaleAgents`). Marca agent.status='unhealthy' quando heartbeats param. Audit log com action `agent_marked_unhealthy`. Loop com try/except (lição ARI-212).
- [ARI-211](https://linear.app/arius-ai/issue/ARI-211) — cleanup aplicado: `error_count` removido de `HeartbeatRequest`
- [ARI-212](https://linear.app/arius-ai/issue/ARI-212) (Done — mergeada 2026-05-06) — try/except + log estruturado + Langfuse event no `_heartbeat_loop`. Loop sobrevive a `ValidationError` em Pydantic, `RuntimeError` em registry, e qualquer outra `Exception`. CancelledError preservada (shutdown intencional continua funcionando).
- [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (Done — mergeada 2026-05-09) — popular `avg_latency_ms`, `p95_latency_ms`, `total_cost_usd` em `agent_snapshots` via enrichment background task

---

## 6.3 — Está funcionando?

### Evidência empírica disponível

**Smoke validation operacional (acumulado das issues mergeadas):**

- [ARI-152](https://linear.app/arius-ai/issue/ARI-152) smoke `/health`, `/ready` cobre liveness/readiness do agent
- [ARI-154](https://linear.app/arius-ai/issue/ARI-154) counters via heartbeat: validados em produção desde merge — `agent_last_counters` populado, `agent_snapshots` criados a cada heartbeat
- [ARI-164](https://linear.app/arius-ai/issue/ARI-164) push transitions complementar: smoke 1 validou que transição `closed → open` aparece em <10s sem aguardar heartbeat (60s); smoke 2 validou graceful degradation
- [ARI-165](https://linear.app/arius-ai/issue/ARI-165) seed via Observatory: smoke 3 validou que `/ready` mostra `warn` IMEDIATAMENTE após restart com CB anthropic em open (estado preservado)
- [ARI-209](https://linear.app/arius-ai/issue/ARI-209) propagação cross-repo de configuração estática: smoke validou Observatory persiste `fail_max=5/3, reset_timeout_seconds=30/60` após heartbeat; SLO `flap_rate:anthropic` retorna `measured_value=0.0 / status=ok`

**Pipeline cross-repo end-to-end** (validado em [ARI-209](https://linear.app/arius-ai/issue/ARI-209)):

```
agent loop _heartbeat_loop (60s)
  → HeartbeatPayload Pydantic Required
  → POST /agents/{slug}/heartbeat (httpx + retry 3×)
  → Observatory router → HeartbeatRequest Optional
  → ProcessHeartbeat.execute (8 passos best-effort)
  → side effects:
     - agent.last_heartbeat_at + status (StatusEngine)
     - cb_repo.upsert_many + append_transitions
     - counters delta + agent_snapshots + last_counters
     - audit_log action=heartbeat
  → response 200 (AgentResponse) — agent não inspeciona
```

**Validação algorítmica:** counters delta calculation testado em `tests/application/test_process_heartbeat.py` (Observatory) — cobre primeiro heartbeat, reset detectado, anomalia, caso normal.

**Suite local consolidada:**

- agent-standard: **390/390 testes passing + 1 skipped** (heartbeat coberto em testes de `_heartbeat_loop` + `ObservatoryClient`)
- Observatory: **205/205 testes passing** (heartbeat coberto em testes do endpoint + `ProcessHeartbeat` + counters delta)

### Sinais de problema conhecidos

Nenhum sinal de problema conhecido pendente após resolução de [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (09/05/2026). Sinais futuros serão registrados conforme aparecerem.

**Defesa em profundidade ARI-210 + ARI-212 completa (2026-05-06):** agent-side (loop sobrevive a exceptions) + server-side (sweep marca agents stale como unhealthy). Cliente Pulso direto agora vê estado real do agent em `GET /agents/{slug}` mesmo se agent crashar ou rede cair. Latência max de detecção: ~150s.

---
## 6.4 — Standard envia o necessário?

### Contrato cliente-servidor

**O que o standard envia hoje** (mapeado em §6.2):

```python
HeartbeatPayload {
    version: str (Required)
    circuit_breakers: list[CircuitBreakerSnapshot] {
        name, state, fail_count, last_state_change,
        fail_max (ARI-209), reset_timeout_seconds (ARI-209)
    }
    counters: CounterSnapshot {
        requests_total, errors_total, process_started_at
    }
}
```

**Endpoint destino:** `POST /agents/{slug}/heartbeat` → `ProcessHeartbeat` → `AgentResponse` (200) ou `AgentNotFoundError` (404).

**Confiabilidade de transporte:** 3 retries com exponential backoff + jitter, drop silencioso após falha. Sem buffer local. Próximo heartbeat re-sincroniza estado completo.

### Drift histórico documentado

| Campo | Cliente envia | Servidor aceita | Status |
| -- | -- | -- | -- |
| `version` | Required | Optional | Histórico inofensivo |
| `circuit_breakers` | Required (lista, pode `[]`) | Optional `None` | Idem |
| `counters` | Required | Optional | Idem |
| `fail_max`, `reset_timeout_seconds` | Required ([ARI-209](https://linear.app/arius-ai/issue/ARI-209)) | Optional (retro-compat) | Decisão deliberada |
| `process_started_at` | str ISO | datetime | Pydantic faz coerce — sem rachadura |
| `last_state_change` | str ISO | datetime | Idem |

### Gaps no contrato (registrados como issues)

1. **`last_error_*` só via push:** heartbeat não carrega contexto de erro do CB. Se push falha (sem retry), essa dimensão de auditoria se perde. Heartbeat seguinte sincroniza state+fail_count mas não a causa.

### Conclusão da seção 6.4

Contrato **funcional para Camada 1** × Operador/SRE (operação básica do Arius admin). Camada 2 × Cliente Pulso direto: [ARI-210](https://linear.app/arius-ai/issue/ARI-210) e [ARI-213](https://linear.app/arius-ai/issue/ARI-213) resolvidas (09/05/2026). Visualização de tendência de latência e custo acumulado por agente está habilitada — `agent_snapshots` fornece série temporal completa. [ARI-211](https://linear.app/arius-ai/issue/ARI-211) foi resolvida como cleanup; [ARI-212](https://linear.app/arius-ai/issue/ARI-212) segue como robustez sem bloquear o contrato.

---

## Caminho para 🟢 (Interpretação A estrita)

§6 alcançou 🟢 em 2026-05-02 com todos os itens fechados:

- [X] §6.1, §6.2, §6.3, §6.4 redigidos com base em fato (cartografia cross-repo em 8 áreas)
- [X] Schema canônico mapeado cross-repo (`HeartbeatPayload` + `HeartbeatRequest`)
- [X] Pipeline `ProcessHeartbeat` documentado linha-a-linha (8 passos best-effort)
- [X] Decisões arquiteturais registradas (best-effort vs UoW, convenção `timeout = 2× interval`, push vs heartbeat fonte de verdade)
- [X] Drift histórico identificado e documentado
- [X] 4 gaps identificados e registrados como issues:
  - [X] [ARI-210](https://linear.app/arius-ai/issue/ARI-210) (stale detection — High)
  - [X] [ARI-211](https://linear.app/arius-ai/issue/ARI-211) (`error_count` cleanup — Low)
  - [X] [ARI-212](https://linear.app/arius-ai/issue/ARI-212) (try/except no loop — High)
  - [X] [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (latency/cost zerados — Medium)
- [X] Suite local consolidada validada (Observatory 205, agent-standard 390)
- [X] Pipeline cross-repo validado end-to-end via smoke da [ARI-209](https://linear.app/arius-ai/issue/ARI-209)

**Tempo real:** 1 sessão de cartografia + redação (2026-05-02 sessão 3, ~3h). 4 issues criadas como gaps registrados.

---

## Notas e direção futura

### Para o SDK arius-agent-core (Fase 3)

`AriusHeartbeat` (futuro) deve abstrair:

- **Loop com try/except externo nativo** (resolve classe [ARI-212](https://linear.app/arius-ai/issue/ARI-212))
- **Schema versionado** (Required progressivo conforme features amadurecem — [ARI-209](https://linear.app/arius-ai/issue/ARI-209) estabeleceu padrão)
- **Hooks pre-send/post-send** para clientes customizarem payload sem fork do SDK
- **Buffering local opcional** (desligado por default, ativável para casos críticos)
- **Configuração estática propagada por padrão** (lição de [ARI-209](https://linear.app/arius-ai/issue/ARI-209))
- **Best-effort em side effects no servidor** (lição da §6 atual)
- **Métricas de saúde do próprio loop** (counter de iterations, last_send_at, last_error) — protege contra death silenciosa

### Para Pulso (Camada 2)

- Cliente direto precisa de **[ARI-210](https://linear.app/arius-ai/issue/ARI-210) resolvida** para confiar em `agent.status` sem checar `last_heartbeat_at` manualmente
- **[ARI-213](https://linear.app/arius-ai/issue/ARI-213) resolvida (09/05/2026)** desbloqueou visualização de tendência de latência e custo acumulado por agente. `agent_snapshots` agora carrega série temporal completa consumível por dashboards Pulso e SLO eval
- Capacidade de auditoria via `audit_log action=heartbeat` é **diferencial vendável** (rastreabilidade compliance-grade)

### Aprendizado de processo (cartografia de §6 vs §3)

- **§3 era greenfield documental** (CB documentado conforme construído)
- **§6 era arqueologia** (Heartbeat existe desde Bloco 1, acumulou camadas via 6 ARIs)
- **Cartografia em arqueologia revela mais drift:** §6 expôs `error_count` dead code, latency/cost zerados, stale detection ausente, loop sem try/except — gaps que estavam invisíveis antes da investigação sistemática
- **Lição:** documentar conceitos com idade ≥ 6 meses exige cartografia mais densa que documentar conceitos novos. Drift acumula silenciosamente.

### Aprendizado arquitetural — `ProcessHeartbeat` best-effort

- Decisão "sem Unit of Work" foi tomada implicitamente em [ARI-152](https://linear.app/arius-ai/issue/ARI-152) / [ARI-154](https://linear.app/arius-ai/issue/ARI-154) (try/except isolados em cada bloco)
- Cartografia tornou explícita a decisão e seu rationale: **heartbeat é informação, não transação**
- Padrão transferível: **operações que combinam múltiplos subdomínios via "pulse" merecem best-effort por ramo**, não atomic. Quem precisa de transação usa endpoint específico do conceito.

### Aprendizado de método — validação cross-source na migração de documentação

Durante a migração do §6 do Linear para o GitHub (2026-05-02), Codex no VS Code pegou 2 divergências em descrições reconstruídas a partir de transcript:

- Path real `src/presentation/api/health.py` (não `src/presentation/health.py`)
- Endpoint real `GET /intelligence/agents/{slug}/health-score` (não `GET /agents/{slug}/health-score`)

**Lição:** descrições de código reconstruídas via memória ou transcript devem ser **validadas contra o código real** antes de virar documentação canônica. Padrão H aplicado à documentação. Convenção a adotar para próximos conceitos (§1, §2, §4, §5, §7): instruções para Codex incluem "abra arquivo X e valide Y antes do append".

### Marco arquitetural

§6 Heartbeat é o **segundo conceito da plataforma Arius a alcançar 🟢 estrito**. Combinado com §3, estabelece padrão validado para os outros 5 conceitos centrais (HealthScore, QualityScore, SLOs, Audit Log, Telemetry). Tempo de cada conceito a 🟢 vai variar — §3 levou 4 dias úteis (greenfield + bug crítico + cross-repo), §6 levou 1 sessão (arqueologia + 4 gaps registrados sem implementação imediata).

---

## Histórico

- **2026-05-02 sessão 3 — §6 → 🟢:** Cartografia completa de Heartbeat em 8 áreas cross-repo. 8 descobertas críticas: stale detection ausente, `error_count` dead code, timeout 1:2 sem documentação, `_heartbeat_loop` sem try/except, sem Unit of Work em `ProcessHeartbeat`, `last_error_*` só via push, latency/cost zerados, `audit_log heartbeat` como contrato implícito. 6 decisões arquiteturais consolidadas. 4 issues criadas: [ARI-210](https://linear.app/arius-ai/issue/ARI-210), [ARI-211](https://linear.app/arius-ai/issue/ARI-211), [ARI-212](https://linear.app/arius-ai/issue/ARI-212), [ARI-213](https://linear.app/arius-ai/issue/ARI-213). "Best-effort by branch" tornado padrão arquitetural transferível. **§6 Heartbeat alcança 🟢 estrito — segundo conceito da plataforma Arius com esse status.**
- **2026-05-02 sessão 4 — migração para GitHub:** Reference movido do Linear (que comprometeu o save monolítico) para `erickmarinho-notebook` no GitHub. Migração feita em 5 chunks via Codex no VS Code, com validação cross-source contra código real em cada chunk. 2 divergências corrigidas (path do health.py, prefixo do endpoint health-score). Aprendizado de método registrado.
- **2026-05-08 — [ARI-218](https://linear.app/arius-ai/issue/ARI-218) absorvida (não-bug):** investigação read-only em 3 etapas (cartografia cross-source, audit_log do Observatory, validação isolada de `logging.lastResort` em container efêmero) confirmou Cenário C — interrupção externa do processo (Docker daemon, SIGKILL, ou hibernação da máquina dev), não bug do `_heartbeat_loop`. Sub-hipóteses A, B', B-a, B-b, B-c todas rejeitadas com evidência independente. Validação positiva das afirmações "Loop NUNCA crasha" e "Loop NUNCA morre silenciosamente" do §6.2. Subproduto técnico: `logging.lastResort` confirmado ativo na imagem do agent-standard (Python 3.13.13, `<_StderrHandler <stderr> (WARNING)>`) — `logger.warning/error/exception` emergem em stderr mesmo sem `basicConfig`/`dictConfig`. Recalibra framing de [ARI-220](https://linear.app/arius-ai/issue/ARI-220) de "app cega" para "logging não-estruturado" (urgência mantida — JSON estruturado continua necessário para Pulso Camada 2 e §7 Telemetry).
- **2026-05-09 — [ARI-213](https://linear.app/arius-ai/issue/ARI-213) resolvida:** enrichment background task post-heartbeat popula `avg_latency_ms`, `p95_latency_ms`, `total_cost_usd` em `agent_snapshots` via `kpi_calculator` (Langfuse v4). Pattern session-per-task primeira aplicação consciente no Observatory. D-A: bg task escolhido vs síncrono. D-B: lifecycle no shutdown não tratado (aceitável para enrichment best-effort). Schema cross-repo intocado, agent-standard intocado, migration desnecessária. Bug silencioso em SLO eval (latency/cost sempre zerados) resolvido. Pulso Camada 2 desbloqueada. Estimativa final 4pts (recalibrada de 3pts originais devido a pattern bg task ser primeira aplicação).
