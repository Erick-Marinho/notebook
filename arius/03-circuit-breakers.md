# §3 Circuit Breakers

> **Status:** 🟢 **Documentado e validado empiricamente.** 8 issues mergeadas ([ARI-204](https://linear.app/arius-ai/issue/ARI-204), [ARI-200](https://linear.app/arius-ai/issue/ARI-200), [ARI-205](https://linear.app/arius-ai/issue/ARI-205), [ARI-165](https://linear.app/arius-ai/issue/ARI-165), [ARI-164](https://linear.app/arius-ai/issue/ARI-164), [ARI-206](https://linear.app/arius-ai/issue/ARI-206), [ARI-207](https://linear.app/arius-ai/issue/ARI-207), [ARI-209](https://linear.app/arius-ai/issue/ARI-209)), bug crítico extinto, pipeline cross-repo validado end-to-end, vocabulário tipado, persistência cross-restart, push transitions em tempo real, 5 métricas operacionais funcionais (3 básicas + 2 de calibração com `reset_timeout_seconds` propagado), pool integration via callback nativo. **Primeiro conceito da plataforma Arius a alcançar 🟢 estrito.**
>
> **Última atualização:** 2026-05-02 (sessão 2 — pós [ARI-209](https://linear.app/arius-ai/issue/ARI-209) cross-repo)

---

## 3.1 — O que é

**Circuit Breaker (CB)** é um padrão de resiliência que protege um sistema de **falhas sustentadas em uma dependência externa**, transformando falhas lentas (timeouts) em falhas rápidas (rejeição imediata) quando a dependência está comprovadamente indisponível.

### Problema concreto que resolve

Sem CB, quando uma dependência externa começa a falhar (API LLM com timeout, Postgres inacessível), o sistema continua tentando — cada chamada espera o timeout completo (segundos), ocupa threads/event loop, esgota pool de conexões, acumula latência percebida pelo usuário, e pode gerar **death spiral** (cada request lento ocupa recurso → fila acumula → mais lento ainda → cascading failure).

**CB transforma esse comportamento.** Após N falhas consecutivas, CB "abre o circuito" — próximas chamadas falham em microssegundos sem nem tocar a dependência. Sistema reconhece falha rápido, libera recursos, mantém latência previsível, evita amplificar problema.

### Os 3 estados

| Estado | Comportamento da chamada | Significado |
| -- | -- | -- |
| **closed** | Chamada vai para a dependência real | Operação normal — CB observa e conta falhas |
| **open** | Chamada falha imediatamente (`CircuitBreakerError`) | Dependência marcada como indisponível — sem chamadas reais |
| **half-open** | UMA chamada de teste passa por vez | Sonda controlada para detectar recuperação |

**Por que 3 estados, não 2:** sem `half-open`, o sistema oscilaria entre `open` e `closed` toda vez que o `reset_timeout` expirasse. `half-open` é o "buffer de probe" — permite uma sonda controlada antes de liberar tráfego. Sucesso na sonda → `closed` (libera tudo). Falha na sonda → volta para `open` (espera novo `reset_timeout`).

### Trade-offs de configuração

CB tem 2 parâmetros principais que **calibram sensibilidade vs tolerância**:

| Parâmetro | Muito baixo | Muito alto |
| -- | -- | -- |
| `fail_max` | Falsos positivos por falhas transitórias (hiccups, timeout pontual) | Sistema queima recursos esperando timeout antes de detectar falha real |
| `reset_timeout` | Martela dependência em recuperação (atrapalha sua subida) | Desperdiça tempo sem usar dependência que já voltou |

**CB não é defesa contra falhas — é tradutor entre falha real e degradação tolerável.** Configuração ruim de CB causa mais problema que ausência de CB. Configuração certa exige conhecer a **natureza** do recurso protegido (latência típica, tipo de falha, criticidade, fallback disponível).

---
## 3.2 — Como implementamos

### Cartografia empírica (2026-04-30)

**Biblioteca:** `pybreaker 1.4.1` (apenas no agent-standard).

**Observatory NÃO instancia CBs** — apenas **representa o estado recebido** via dataclasses, persiste em tabelas (`agent_circuit_breakers`, `agent_circuit_breaker_history`), expõe via API e calcula métricas agregadas (`CBMetrics`).

**Total no agent-standard: 5 CBs**

| CB | Atributo | `fail_max` | `reset_timeout` | Fallback |
| -- | -- | -- | -- | -- |
| `anthropic` | class attr `_cb` em `AnthropicAdapter` | 5 | 30s | Nenhum — propaga `CircuitBreakerError` |
| `openai` | class attr `_cb` | 5 | 30s | Nenhum |
| `groq` | class attr `_cb` | 5 | 30s | Nenhum |
| `gemini` | class attr `_cb` | 5 | 30s | Nenhum |
| `postgres` | `_postgres_cb` (module-level) | 3 | 60s | Dict in-memory (`PostgresCheckpointer`) |

**Por que Postgres é diferente** (mais sensível para abrir, mais paciente para tentar voltar):

1. **Cardinalidade:** LLMs são 4 (alternativos), Postgres é único (sem fallback automático). Detectar rápido tem mais valor.
2. **Tipo de falha:** Postgres em rede local raramente tem ruído — quando falha, **tende a ser problema real** (container caiu, OOM, deadlock). LLM externo tem mil falhas transitórias toleráveis.
3. **Latência da chamada:** Falha de Postgres é instantânea (`connection refused`). Falha de LLM espera timeout (segundos). `fail_max=3` em Postgres ainda é detecção rápida; em LLM seria precoce.
4. **Tempo de recuperação:** Postgres recuperando = container reiniciar + WAL replay + reabrir conexões + possível migration. API LLM recuperando = só status code voltar a 200. Daí `reset_timeout=60s` vs `30s`.

### Factory padronizada

`src/infrastructure/resilience/circuit_breaker.py:241-267` — `create_circuit_breaker(name, fail_max=5, reset_timeout=30)`:

- Cria instância `pybreaker.CircuitBreaker`
- Instala `_LoggingListener` automaticamente
- Registra em `_cb_registry: dict[str, tuple[CircuitBreaker, _LoggingListener]]`

Registry global permite snapshot de todos os CBs para `/ready` e heartbeat.

### Wrapper async — `call_async_with_breaker` (corrigido em 2026-05-01)

**Histórico crítico:** o wrapper original tinha bug estrutural — chamava `state.before_call(func, ...)` que invocava `func` sincronamente como parte da transição open → half-open. Como `func` é corrotina, retornava objeto `Coroutine` sem levantar exceção, pybreaker interpretava como "sucesso fictício" e fechava o CB sem validar recuperação real.

Bug [ARI-202](https://linear.app/arius-ai/issue/ARI-202) (descoberto em 2026-04-30 via Padrão H — validação operacional empírica) → fix tático definido no ADR-001 (Caminho A) → implementado em [ARI-204](https://linear.app/arius-ai/issue/ARI-204), mergeado em main em 2026-05-01.

**Lógica corrigida:**

- **CLOSED:** `await func()` direto. Sucesso reseta counter. Falha incrementa; se atinge `fail_max`, on_failure transiciona para OPEN.
- **OPEN:** se `reset_timeout` não expirado, levanta `CircuitBreakerError` sem chamar func. Se expirado, transiciona para HALF_OPEN via `cb.half_open()` standalone (método público), depois `await func()` real.
- **HALF_OPEN:** `await func()` direto. Sucesso → CB fecha (success_threshold=1). Falha → on_failure reabre CB e levanta `CircuitBreakerError`.

**Decisões registradas no fix (ADR-001):**

1. Manter chamada para `listener.before_call()` — contrato público da lib
2. `CircuitBreakerError` prioritizado sobre exceção original quando `_handle_error` levanta — adapters esperam essa exceção
3. Race condition entre snapshot de estado e execução aceita conscientemente (mesma race existe em `pybreaker.cb.call()` síncrono)

**Caminho E (complementar):** [ARI-203](https://linear.app/arius-ai/issue/ARI-203) monitora PR #103 upstream do pybreaker (mensal) — quando v1.5.0 release com `cb.acall()` oficial, plano de migração entra em vigor.

---
### Vocabulário tipado de estados (resolvido em 2026-05-01 — [ARI-200](https://linear.app/arius-ai/issue/ARI-200) mergeada)

- `pybreaker` emite `new_state.name` como `"open"`, `"half-open"`, `"closed"` (string crua)
- agent-standard agora aplica `CBState(StrEnum)` em `CircuitBreakerSnapshot.state` e `HealthResponse.llm_circuit_breaker` (escopo Opção 4 da [ARI-200](https://linear.app/arius-ai/issue/ARI-200))
- Wire-format permanece string (`"closed"`/`"open"`/`"half-open"`) via `model_dump(mode="json")` — compatível com `Literal[...]` que Observatory já tem na borda HTTP
- Quebra de contrato consciente: `HealthResponse.llm_circuit_breaker` antes era `str` com sentinela `"unknown"`; agora é `CBState | None` — `null` substitui sentinela
- Observatory mantém `state: str` no domain (decisão Opção 4 — unificação cross-repo será feita quando AriusCB for extraído no SDK arius-agent-core, Fase 3 [ARI-201](https://linear.app/arius-ai/issue/ARI-201))
- Padrão G aplicado: typo silencioso (`"opened"` em vez de `"open"`) é rejeitado por Pydantic ao desserializar

### Persistência de estado entre restarts (resolvido em 2026-05-02 — [ARI-165](https://linear.app/arius-ai/issue/ARI-165) mergeada)

- agent-standard antes: estado vivia somente em memória do processo. Restart zerava `_cb_registry`, todos CBs nasciam `closed` mesmo com incidente em curso.
- agent-standard agora: no startup, `_seed_cbs_from_observatory()` consulta `GET /agents/{slug}/circuit-breakers` (endpoint que já existia no Observatory) e seedea CBs com estado conhecido.
- **Graceful degradation:** se Observatory está down/timeout/erro, agent loga warning estruturado e segue startup com CBs em default (closed). Funcionalidade preservada — só perde benefício de persistência.
- Smoke 3 validou empiricamente: após restart com CB anthropic em open, `/ready` mostra `warn` IMEDIATAMENTE (CB nasceu open via seed, antes de qualquer chamada `/chat`).
- Observatory continua persistindo em duas tabelas (`agent_circuit_breakers` upsert + `agent_circuit_breaker_history` append-only) — fonte de verdade para o seed.

### Push de transições em tempo real (resolvido em 2026-05-02 — [ARI-164](https://linear.app/arius-ai/issue/ARI-164) mergeada cross-repo)

Antes da [ARI-164](https://linear.app/arius-ai/issue/ARI-164): transições eram visíveis ao Observatory **só via heartbeat** (60s default). Transições rápidas (`open → closed` em 35s) podiam não ser registradas.

Agora:

- Endpoint novo no Observatory: `POST /agents/{slug}/circuit-breakers/transitions` retornando 202
- `_PushTransitionListener` no agent-standard (separado do `_LoggingListener` — Decisão 5) dispara push fire-and-forget via `asyncio.create_task` quando state_change ocorre
- Schema completo do payload: `name, from_state, to_state, fail_count, transitioned_at, last_error_type, last_error_message`
- Persistência em `agent_circuit_breaker_history` (campos extras `from_state` e `last_error_*` vão para `audit_log` — Decisão O2, sem migration de schema v1)
- **Wiring tardio:** `wire_push_listeners()` chamado no lifespan APÓS `_seed_cbs_from_observatory()` — ordem crítica preservada (durante seed, `cb.open()` não dispara push porque listener ainda não está instalado, evitando bug sutil de "anthropic abriu agora" para estado que era apenas restore)
- Decisões arquiteturais (5): fire-and-forget (Decisão 1) + drop com warning se Observatory down (Decisão 2) + sem retry (Decisão 3) + schema completo (Decisão 4) + listener separado (Decisão 5)
- Smoke 1 validou: transição `closed → open` apareceu em history em <10s, sem aguardar heartbeat (60s)
- Smoke 2 validou graceful degradation: Observatory parado, agent emitiu `cb_push_transition_failed`, NÃO crashou

---
### Métricas operacionais de calibração (resolvido em 2026-05-02 — [ARI-206](https://linear.app/arius-ai/issue/ARI-206) mergeada)

`CBMetrics` no Observatory ganhou 2 campos novos para validar configuração de CB ao longo do tempo:

- `flap_rate: float | None` — proporção de aberturas seguidas de fechamento curto (< 2× `reset_timeout`) sobre total de aberturas. Sinaliza `fail_max` muito baixo se taxa alta.
- `half_open_first_attempt_success_rate: float | None` — proporção de probes half-open que passam direto na **primeira tentativa pós-abertura**. Sinaliza `reset_timeout` conservador demais se taxa alta.

Decisões algorítmicas finalizadas (Etapa 2 da [ARI-206](https://linear.app/arius-ai/issue/ARI-206)):

- **Fonte de dados:** `agent_circuit_breaker_history` (snapshots a cada heartbeat — cobertura 100%, resolução temporal limitada a ~60s). audit_log com `from_state` foi descartado pois cobertura ainda baixa.
- **flap_rate Versão A:** ciclos curtos / total aberturas. "Ciclo curto" = abertura seguida de fechamento dentro de 2× `reset_timeout`.
- **half_open_first_attempt_success_rate Leitura X:** estritamente o **primeiro probe após cada transição closed → open**. Subsequentes probes do mesmo ciclo de recovery são ignorados (algoritmo usa flag `outcome_resolved_in_cycle` para bloquear re-disparo).
- **Janela temporal:** per-SLO via `window_hours` (padrão herdado dos 3 campos existentes).
- **Casos limite:** `None` quando indefinido (zero amostras no denominador).
- **Threshold mínimo:** 2 ciclos completos para flap_rate, 3 transições half-open para success_rate.

**Validação empírica:** 202 testes passing no Observatory após merge — incluindo property tests P14a-e + 14 unit tests cenário-a-cenário. Cenário ambíguo (`closed → open → half-open → open → half-open → closed`) explicitamente coberto via `test_multiple_attempts_in_same_recovery_only_count_first` validando rate=0.0 (Leitura X estrita).

**Aprendizado de processo:** o cenário ambíguo foi descoberto durante decisão arquitetural Etapa 2 e ratificado conscientemente. Algoritmo proposto inicialmente tinha bug que apenas trace caso-a-caso revelou. Padrão H validou-se recursivamente — não só ao código de produção, mas ao algoritmo articulado em design.

### Pool psycopg — callback `check` (resolvido em 2026-05-02 — [ARI-207](https://linear.app/arius-ai/issue/ARI-207) mergeada)

Anomalia operacional descoberta durante validação do Cenário B re-executado pós-fix [ARI-204](https://linear.app/arius-ai/issue/ARI-204) (2026-05-01): após restart do container Postgres, recovery do CB postgres demorava ~2× o `reset_timeout` configurado. Diagnóstico inicial atribuiu ao pool psycopg, mas cartografia empírica em [ARI-207](https://linear.app/arius-ai/issue/ARI-207) revelou causa-raiz mais sutil.

**Diagnóstico real (Etapa 1 da [ARI-207](https://linear.app/arius-ai/issue/ARI-207)):**

- Pool isolado dreniou 10 conexões BAD em ~10s (sem CB no caminho)
- Os "60s adicionais" do Cenário B eram **interação CB↔pool**, não pool intrínseco:
  - 3 falhas → CB OPEN por 60s
  - Durante OPEN, nenhuma chamada chega ao pool → BADs cacheadas não são consumidas/evictadas
  - T+60s: half-open probe único pode pegar BAD do pool → reabre CB → +60s
- Pool sem callback `check` (default `None`) só descarta BADs **reativamente** (via `transaction_status=UNKNOWN` em putconn)

**Caminho E escolhido (não previsto nos 4 caminhos originais):**

`AsyncConnectionPool.check_connection` (staticmethod) é literalmente desenhado para ser passado como callback `check` no construtor. Documentação interna do psycopg-pool endossa: "available for client usage, for instance as !check callback when a pool is created."

**Implementação:** 1 linha em `src/infrastructure/persistence/postgres_checkpointer.py`:

```python
pool = AsyncConnectionPool(
    self._connection_string,
    kwargs={...},
    min_size=1,
    max_size=10,
    timeout=timeout_s,
    open=False,
    # ARI-207: validação proativa de conexões BAD em cada getconn
    check=AsyncConnectionPool.check_connection,
)
```

**Por que Caminho E é melhor que os 4 originais:**

| Caminho | Custo | Resolve causa? | Acopla CB↔pool? | Cobertura |
| -- | -- | -- | -- | -- |
| **E (callback check)** | 1 linha | Sim | **Não** | Universal |
| A (pool.check() em listener) | ~5 linhas | Parcial | Sim | Só pós-CB |
| B (aumentar reset_timeout) | 1 linha | Não (sintoma) | Não | Indireto |
| C (custom health check) | ~20 linhas | Sim | Sim | Reinventa roda |

**Trade-off documentado:** ~0.5-2ms por `getconn` (query vazia "ping"). Negligível em rede local. Ganho: BADs nunca chegam ao caller, recovery cai de O(múltiplos ciclos CB) para O(1 RTT do ping).

**Validação empírica (smoke Cenário B re-executado pós-[ARI-207](https://linear.app/arius-ai/issue/ARI-207)):**

- Probe único pós-restart Postgres + reset_timeout: HTTP 200 em 3.09s (de primeira)
- CB postgres recovery clean: `closed → open → half-open → closed` (sem ciclos extras)
- Suite: 387 testes passing (387 ← 386, +1 teste novo `test_pool_configured_with_check_callback`)

### Propagação de configuração estática no heartbeat (resolvido em 2026-05-02 — [ARI-209](https://linear.app/arius-ai/issue/ARI-209) mergeada cross-repo)

Gap descoberto durante [ARI-206](https://linear.app/arius-ai/issue/ARI-206): `flap_rate` em `CBMetrics` precisa de `reset_timeout_seconds` para definir "ciclo curto" (= 2× reset_timeout). Calculador aceita o parâmetro, mas **caller `EvaluateSLO._evaluate_cb_metric` no Observatory passava sempre `None`** porque a info não chegava via heartbeat. Resultado: `flap_rate` retornava `None` em produção mesmo com history suficiente.

**Solução cross-repo (deploy em 2 etapas — Decisão 5):**

**Parte 1 — Observatory (mergeada primeiro, retro-compat):**

- `CircuitBreakerStateRequest` ganha `fail_max: Optional[int]` e `reset_timeout_seconds: Optional[int]` (Optional para retro-compat com agents antigos)
- `CircuitBreakerStateResponse` (GET endpoint) expõe os campos (Decisão 4a)
- `CircuitBreakerState` (domain) ganha 2 campos com `default=None`
- Migration Alembic `d4e5f6a7b8c9_add_cb_config_columns`: 2 colunas Integer NULL em `agent_circuit_breakers`
- `AgentCircuitBreakerModel` + `_state_to_entity` + `upsert_many` propagam
- Router mapping + `ProcessHeartbeat` re-stamp + audit log payload (Decisão 4b) propagam
- `_evaluate_cb_metric` em `slos.py:251` lê `reset_timeout_seconds` do `cb_state` corrente (lookup por nome em `current_states`)
- GET endpoint herda os campos via `model_validate(cb)` com `from_attributes=True` (sem código novo)

**Parte 2 — agent-standard (mergeada após Parte 1):**

- `get_circuit_breaker_states()` popula `fail_max` e `reset_timeout_seconds` direto dos atributos públicos `cb.fail_max` e `cb.reset_timeout` (precedente: `_LoggingListener._emit_opened_event` já lia esses atributos)
- Coerção `int(cb.reset_timeout)` para Decisão 2 (int, não float — granularidade de segundo é suficiente)
- `CircuitBreakerSnapshot` Pydantic recebe campos como **Required** (`int = Field(ge=1)`) — Decisão 1, agent SEMPRE tem o dado disponível, falha de validação é bug que merece ser detectado

**Decisões consolidadas (Etapa 2 da [ARI-209](https://linear.app/arius-ai/issue/ARI-209)):**

| # | Decisão | Valor |
|---|---|---|
| 1 | Tipos no agent-standard | Required (`int = Field(ge=1)`) |
| 2 | Tipo de `reset_timeout_seconds` | `int` (não float) |
| 3 | Migration | NULL puro (precedente de `last_state_change`) |
| 4a | `CircuitBreakerStateResponse` (GET) | Expõe campos novos |
| 4b | Audit log payload | Inclui campos novos |
| 5 | Estratégia de deploy | Observatory primeiro |
| 6 | Migration | Manual seguindo padrão Alembic existente |

**Pipeline cross-repo validado end-to-end:**

````
agent-standard registry
  → get_circuit_breaker_states (dict com 6 keys)
  → CircuitBreakerSnapshot Pydantic Required
  → POST /agents/{slug}/heartbeat
  → CircuitBreakerStateRequest Optional (retro-compat)
  → router mapping
  → ProcessHeartbeat re-stamp
  → cb_repo.upsert_many
  → tabela agent_circuit_breakers (colunas populadas)
  → _evaluate_cb_metric lê cb_state.reset_timeout_seconds
  → calculate_cb_metrics produz flap_rate=0.0
````

**Validação empírica (smoke cross-repo pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209)):**

- Após heartbeat, Observatory persiste:
  - 4 LLMs: `fail_max=5, reset_timeout_seconds=30`
  - postgres: `fail_max=3, reset_timeout_seconds=60`
- SLO `flap_rate:anthropic` em janela 24h retornou `measured_value=0.0 / status=ok` (pré-fix retornaria `null / insufficient_data`)
- Suite agent-standard: 390 testes passing (387 → 390, +3 novos)
- Suite Observatory: 205 testes passing (202 → 205, +3 novos)

**Aprendizado durante implementação:** teste e2e em Parte 1 detectou que `ProcessHeartbeat._restamp` recriava `CircuitBreakerState` sem propagar campos novos — bug que minha instrução não cobria. **Padrão H validou-se mais uma vez** — testes empíricos > raciocínio sobre código. Mesma classe de detecção da [ARI-164](https://linear.app/arius-ai/issue/ARI-164) (instance vs class attribute) e [ARI-206](https://linear.app/arius-ai/issue/ARI-206) (algoritmo Leitura X). **Lição:** instruções para implementação cross-cutting devem incluir teste e2e que exercita pipeline completo — bugs em fronteiras (re-stamp, re-mapping) só aparecem com smoke real.

### Listeners (extensão pybreaker → mundo Arius)

`pybreaker` por si só não sabe nada do Arius (Langfuse, Observatory, logger). **Listeners são o ponto de extensão** que ligam o evento mecânico (transição de estado) ao sinal observável.

**Listeners atuais (todos os 5 CBs):**

- `_LoggingListener` (genérico): `failure(cb, exc)` armazena `last_error`; `state_change(cb, old, new)` atualiza `last_state_change`, loga WARNING, emite evento Langfuse (`circuit_breaker_opened`/`half_open`/`recovered`). Falha ao emitir Langfuse não propaga ([ARI-166](https://linear.app/arius-ai/issue/ARI-166)) — degrada silenciosamente.
- `_PushTransitionListener` ([ARI-164](https://linear.app/arius-ai/issue/ARI-164), todos os 5 CBs): `state_change(cb, old, new)` dispara `asyncio.create_task` fire-and-forget enviando POST para Observatory. `failure(cb, exc)` armazena `last_error` para enriquecer payload subsequente.
- `_PostgresCBListener` (adicional, só no Postgres CB): `state_change` loga ERROR/INFO específico do contexto Postgres + fallback in-memory.

**Por que múltiplos listeners além do log local:** **leitores diferentes**. Log local é para dev debugando incidente em vivo. Langfuse é para análise retrospectiva de trace por engenheiro de IA. Observatory (heartbeat + push) é para operador/SRE em monitoramento contínuo. ADR-000 Implicação 2 ("tradução é produto") em ação: mesmo evento técnico, vocabulários diferentes por audiência.

### Decisões registradas

- [ARI-152](https://linear.app/arius-ai/issue/ARI-152) (Done) — `PostgresCheckpointer` + listener + fallback in-memory
- [ARI-154](https://linear.app/arius-ai/issue/ARI-154) (Done) — heartbeat envia counters + CB schema completo no audit_log payload
- [ARI-162](https://linear.app/arius-ai/issue/ARI-162) (Done) — vocabulário CB alinhado cross-repo (parcial; [ARI-200](https://linear.app/arius-ai/issue/ARI-200) completou em 2026-05-01)
- [ARI-164](https://linear.app/arius-ai/issue/ARI-164) (Done — mergeada 2026-05-02) — push-based de transições cross-repo (resolveu janela cega entre heartbeats)
- [ARI-165](https://linear.app/arius-ai/issue/ARI-165) (Done — mergeada 2026-05-02) — persistência de estado entre restarts via seed do Observatory
- [ARI-200](https://linear.app/arius-ai/issue/ARI-200) (Done — mergeada 2026-05-01) — Padrão G aplicado ao vocabulário CB no agent-standard
- [ARI-201](https://linear.app/arius-ai/issue/ARI-201) (Backlog, agent-core) — AriusCB com cardinalidade flexível para SDK Fase 3
- [ARI-202](https://linear.app/arius-ai/issue/ARI-202) (Done) — bug original do `call_async_with_breaker` (causa-raiz documentada)
- [ARI-203](https://linear.app/arius-ai/issue/ARI-203) (Backlog Low) — monitoramento PR #103 upstream pybreaker (cadência mensal)
- [ARI-204](https://linear.app/arius-ai/issue/ARI-204) (Done — mergeada 2026-05-01) — implementação do fix ADR-001
- [ARI-205](https://linear.app/arius-ai/issue/ARI-205) (Done — mergeada 2026-05-01) — reforço de 6 testes com spy counter
- [ARI-206](https://linear.app/arius-ai/issue/ARI-206) (Done — mergeada 2026-05-02) — flap_rate + half_open_first_attempt_success_rate em CBMetrics
- [ARI-207](https://linear.app/arius-ai/issue/ARI-207) (Done — mergeada 2026-05-02) — pool psycopg callback `check` (Caminho E)
- [ARI-208](https://linear.app/arius-ai/issue/ARI-208) (Backlog Medium) — adicionar mypy ao Observatory (paridade com agent-standard)
- [ARI-209](https://linear.app/arius-ai/issue/ARI-209) (Done — mergeada 2026-05-02 cross-repo) — propagação de `reset_timeout_seconds` no heartbeat
- ADR-001 do agent-standard — fix tático para `call_async_with_breaker` quebrado em half-open com funções async (primeiro ADR físico do agent-standard)

---
## 3.3 — Está funcionando?

### Evidência empírica disponível (atualizada 2026-05-02 sessão 2 pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209))

**Smoke validation operacional pós-fix [ARI-204](https://linear.app/arius-ai/issue/ARI-204) (2026-05-01):**

- **Cenário A re-executado** (chave Anthropic inválida real):
  - 5 falhas reais → `closed → open` ✅
  - 32s aguardo + probe falho real → `half-open → open` ✅ (antes era `→ closed` por bug [ARI-202](https://linear.app/arius-ai/issue/ARI-202))
  - `RuntimeWarning: coroutine ... was never awaited` desapareceu dos logs
  - Comportamento agora reflete modelo articulado em §3.1
- **Cenário B re-executado** (Postgres parado):
  - CB postgres abre após 3 falhas (`closed → open`) ✅
  - Fallback in-memory funciona durante OPEN (5 chats retornam 200) ✅
  - Probe falho REABRE CB (probe 1 com pool BAD) ✅ — bug extinto neste ponto também
  - Probe sucedido FECHA CB (probe 2 com pool fresh) ✅
  - Anomalia secundária descoberta: pool psycopg ~60s de cache ([ARI-207](https://linear.app/arius-ai/issue/ARI-207))
- **Cenário C** (DNS errado): cancelado pós-descoberta de bug — não traria info nova; mesmo fluxo do A.

**Smoke validation operacional pós-[ARI-200](https://linear.app/arius-ai/issue/ARI-200) (2026-05-01):**

- Smoke `/health`, `/ready`: vocabulário tipado funciona, wire-format string preservado.
- Heartbeat enviado para Observatory: payload válido, estado serializa como `"open"` puro (não `<CBState.OPEN: 'open'>`).
- Observatory persiste como `Literal["closed", "open", "half-open"]` via borda HTTP — compatível.

**Smoke validation operacional pós-[ARI-165](https://linear.app/arius-ai/issue/ARI-165) (2026-05-02):**

- Smoke 1 (Observatory up, restart agent): seed atua em startup, CBs nascem com estado conhecido.
- Smoke 2 (Observatory down): graceful degradation, log `cb_seed_failed` com `ConnectError`, agent sobe normal.
- Smoke 3 (estado preservado entre restarts): `/ready` mostra `warn` IMEDIATAMENTE após restart, sem chamar `/chat`.

**Smoke validation operacional pós-[ARI-164](https://linear.app/arius-ai/issue/ARI-164) (2026-05-02):**

- Smoke 1 (push em tempo real): transição `closed → open` apareceu em `agent_circuit_breaker_history` + `audit_log` em <10s, sem aguardar heartbeat (60s).
- Smoke 2 (Observatory down): agent emitiu `cb_push_transition_failed` para transições, NÃO crashou.

**Validação algorítmica pós-[ARI-206](https://linear.app/arius-ai/issue/ARI-206) (2026-05-02):**

- 202 testes passing no Observatory (200 anteriores + 2 cenários novos)
- Property tests P14a-e validam invariantes (bounds [0,1] ou None)
- 14 unit tests cenário-a-cenário cobrem: ciclos curtos / longos / mistos, sucesso/falha/mistura, múltiplas tentativas no mesmo ciclo (rate=0.0), 2 ciclos separados (rate=1.0)
- Bug detectado durante validação caso-a-caso (algoritmo retornava 1.0 quando deveria ser 0.0 no cenário ambíguo) — corrigido com flag `outcome_resolved_in_cycle` antes de merge

**Smoke validation operacional pós-[ARI-207](https://linear.app/arius-ai/issue/ARI-207) (2026-05-02):**

- **Cenário B re-re-executado** (Postgres parado + restart, agora com callback `check` ativo):
  - 3 falhas → CB postgres OPEN ✅
  - Fallback in-memory funciona durante OPEN ✅
  - Postgres restart durante OPEN, aguarda reset_timeout=60s
  - Probe único pós-reset_timeout: **HTTP 200 em 3.09s de primeira** ✅
  - CB recovery clean: `closed → open → half-open → closed` SEM ciclos extras ✅
  - Tempo total de recovery: ~63s (60s reset_timeout + 3s probe) — agora determinístico, antes era ~120s+ (multi-ciclo)

**Smoke validation cross-repo pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209) (2026-05-02):**

- Após heartbeat (60s default), Observatory persiste corretamente os campos novos:
  - 4 LLMs (anthropic, openai, groq, gemini): `fail_max=5, reset_timeout_seconds=30`
  - postgres: `fail_max=3, reset_timeout_seconds=60`
- **SLO `flap_rate:anthropic` em janela 24h: `measured_value=0.0 / status=ok`** ✅ (pré-fix retornaria `null / insufficient_data`)
- Pipeline end-to-end validado: agent registry → snapshot → POST heartbeat → router mapping → re-stamp → upsert → tabela → `_evaluate_cb_metric` → `calculate_cb_metrics` → `flap_rate=0.0`
- Bug detectado durante smoke Parte 1 (re-stamp em `ProcessHeartbeat` dropava campos antes de upsert) — corrigido antes de merge

**Suite local consolidada (pós-todas-as-issues):**

- agent-standard: **390/390 testes passing + 1 skipped** (suite cresceu de 219 → 362 → 371 → 378 → 386 → 387 → 390 entre as 7 issues do agent-standard)
- Observatory: **205/205 testes passing** (suite cresceu de 152 → 183 → 200 → 202 → 205 entre as 4 issues do Observatory)

### Métricas operacionais para validar configuração ao longo do tempo

`CBMetrics` no Observatory **calcula 5 métricas (atualizado 2026-05-02 pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209)):**

**Métricas básicas (existentes):**

- `cb_open_duration_minutes` — tempo total que CB ficou em open na janela
- `cb_open_count` — quantas vezes CB transicionou para open na janela
- `cb_availability_pct` — % do tempo NÃO em "open" na janela (half-open conta como "available")

**Métricas de calibração ([ARI-206](https://linear.app/arius-ai/issue/ARI-206) — mergeada, [ARI-209](https://linear.app/arius-ai/issue/ARI-209) propagou config estática):**

- `flap_rate` (`float | None`) — proporção de aberturas seguidas de fechamento curto. Alta flap_rate (>50%) sugere `fail_max` muito baixo. **Retorna valor real em produção pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209)** (anteriormente sempre `None`).
- `half_open_first_attempt_success_rate` (`float | None`) — proporção de probes half-open que passam de primeira. Taxa muito alta (>90%) sugere `reset_timeout` conservador demais.

Suportadas como SLOs via `metric="flap_rate:<cb_name>"` ou `metric="half_open_first_attempt_success_rate:<cb_name>"` em `EvaluateSLO._evaluate_cb_metric` — `None` retornado pela métrica vira `insufficient_data` no SLOEvaluation.

### Sinais de problema conhecidos

- **[ARI-203](https://linear.app/arius-ai/issue/ARI-203) monitoramento contínuo:** PR #103 upstream pybreaker para `cb.acall()` oficial — verificação mensal. Quando v1.5.0 release, ADR-001 prevê plano de migração.
- **[ARI-208](https://linear.app/arius-ai/issue/ARI-208) (Backlog):** mypy ainda não está configurado no Observatory — paridade com agent-standard pendente. Não bloqueia §3.

---
## 3.4 — Standard envia o necessário?

### Contrato cliente-servidor (heartbeat + push)

**O que o standard envia (atualizado 2026-05-02 pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209)):**

Via heartbeat periódico (60s default, range 5-300s configurável):

```python
class CircuitBreakerSnapshot(BaseModel):
    name: str
    state: CBState  # tipado via Enum (ARI-200) — wire-format string preservado
    fail_count: int  # >= 0
    last_state_change: str | None  # ISO datetime ou None
    fail_max: int  # ARI-209 — Required no agent-standard
    reset_timeout_seconds: int  # ARI-209 — Required no agent-standard
```

Via push de transições em tempo real ([ARI-164](https://linear.app/arius-ai/issue/ARI-164)):

```python
{
    "name": str,
    "from_state": "closed" | "open" | "half-open",
    "to_state": "closed" | "open" | "half-open",
    "fail_count": int,
    "transitioned_at": datetime,
    "last_error_type": str | None,
    "last_error_message": str | None,
}
```

**Endpoints destino no Observatory:**

- `POST /agents/{slug}/heartbeat` (existente, schema agora aceita campos novos como Optional para retro-compat)
- `POST /agents/{slug}/circuit-breakers/transitions` (novo via [ARI-164](https://linear.app/arius-ai/issue/ARI-164), retorna 202)
- `GET /agents/{slug}/circuit-breakers` (existente, agora expõe `fail_max` e `reset_timeout_seconds` — Decisão 4a da [ARI-209](https://linear.app/arius-ai/issue/ARI-209))

### Gaps no contrato

1. **Audit log não retorna em queries de history:** `from_state` e `last_error_*` ficam em `audit_log` (Decisão O2 da [ARI-164](https://linear.app/arius-ai/issue/ARI-164)), exigem JOIN para queries futuras de "transições por motivo de erro". **Não bloqueia operação atual** — gap arquitetural a tratar quando audit_log v2 (compliance-grade com hash encadeado) for implementado.

### Conclusão da seção 3.4

Contrato **funcional e completo** para Camada 1 × Operador/SRE (operação básica do Arius admin) **e** Camada 2 × Cliente Pulso direto (consumo de métricas operacionais incluindo `flap_rate` em produção). Próximos passos do contrato pertencem a outras issues (audit log v2, AriusCB no SDK).

---

## Caminho para 🟢 (Interpretação A estrita)

§3 alcançou 🟢 em 2026-05-02 com todos os itens fechados:

- [x] §3.1, §3.2, §3.3, §3.4 redigidos com base em fato (não memória)
- [x] Bug [ARI-202](https://linear.app/arius-ai/issue/ARI-202) diagnosticado e corrigido
- [x] Decisão arquitetural de fix registrada (ADR-001)
- [x] [ARI-204](https://linear.app/arius-ai/issue/ARI-204) mergeada em main
- [x] Cenário A re-executado pós-fix
- [x] Cenário B re-executado pós-fix ([ARI-204](https://linear.app/arius-ai/issue/ARI-204))
- [x] [ARI-200](https://linear.app/arius-ai/issue/ARI-200) mergeada (vocabulário tipado no agent-standard)
- [x] [ARI-205](https://linear.app/arius-ai/issue/ARI-205) mergeada (reforço de 6 testes com spy counter)
- [x] [ARI-165](https://linear.app/arius-ai/issue/ARI-165) mergeada (persistência de estado via seed do Observatory)
- [x] [ARI-164](https://linear.app/arius-ai/issue/ARI-164) mergeada cross-repo (push transitions)
- [x] [ARI-206](https://linear.app/arius-ai/issue/ARI-206) mergeada (flap_rate + half_open_first_attempt_success_rate)
- [x] [ARI-207](https://linear.app/arius-ai/issue/ARI-207) mergeada (pool psycopg callback `check` — Caminho E)
- [x] Cenário B re-re-executado pós-[ARI-207](https://linear.app/arius-ai/issue/ARI-207) (smoke validou recovery clean em ~63s determinísticos)
- [x] Cenário C re-executado pós-todos-os-fixes (cancelado conscientemente — mesmo fluxo do Cenário A, não traz info nova)
- [x] [ARI-209](https://linear.app/arius-ai/issue/ARI-209) mergeada cross-repo (propagação de `reset_timeout_seconds` no heartbeat — flap_rate retorna valor real em produção)

**Tempo real:** 4 dias úteis (2026-04-29 a 2026-05-02). 8 issues mergeadas + 1 ADR físico + bug crítico extinto + pipeline cross-repo validado end-to-end.

---

## Notas e direção futura

### Para o SDK arius-agent-core (Fase 3 — AriusCB + AriusRetry)

`AriusCB` deve abstrair pybreaker mantendo:

- **Cardinalidade flexível** (1..N CBs por agente, [ARI-201](https://linear.app/arius-ai/issue/ARI-201))
- **Vocabulário tipado por padrão** (`CBState` exportado pelo SDK, importado por agent-standard E Observatory — unifica decisão Opção 4 da [ARI-200](https://linear.app/arius-ai/issue/ARI-200))
- **Listener interface clara** (Langfuse + Observatory push + logger como defaults opcionais)
- **Configuração por CB explícita** (não constantes hardcoded)
- Não acoplar Observatory diretamente — interface de reporting
- **Padrão de spy counter como obrigatório em testes** (lição de [ARI-205](https://linear.app/arius-ai/issue/ARI-205))
- **Wiring tardio de listeners cross-instance preservado** (lição de [ARI-164](https://linear.app/arius-ai/issue/ARI-164))
- **Métricas de calibração nativas** (lição de [ARI-206](https://linear.app/arius-ai/issue/ARI-206): `flap_rate` + `half_open_first_attempt_success_rate` fazem parte de "ciclo de vida saudável" de qualquer CB)
- **Pool integration as configuration, not coupling** (lição de [ARI-207](https://linear.app/arius-ai/issue/ARI-207))
- **Configuração estática propagada via heartbeat** (lição de [ARI-209](https://linear.app/arius-ai/issue/ARI-209): `fail_max` e `reset_timeout_seconds` viajam no payload por padrão; auditoria de drift de configuração é capacidade nativa do SDK)

### Para Pulso (Camada 2)

As 5 métricas operacionais (`cb_open_duration_minutes`, `cb_open_count`, `cb_availability_pct`, `flap_rate`, `half_open_first_attempt_success_rate`) alimentam decisão de "reconfigure seu CB" como recomendação ao cliente. Todas funcionais em produção pós-[ARI-209](https://linear.app/arius-ai/issue/ARI-209).

### Investigação pendente (Notas de Estudo)

Como Langfuse v4 self-hosted persiste eventos. Confirmar se vão para ClickHouse ou PostgreSQL, estrutura de query disponível, indexação por `level`.

### Aprendizado de processo (Padrão H validou-se em 4 níveis)

- **Nível 1 — código de produção ([ARI-202](https://linear.app/arius-ai/issue/ARI-202)):** suite verde com mock NÃO bastou para detectar bug. Apenas validação operacional empírica (chave Anthropic real) revelou.
- **Nível 2 — design algorítmico ([ARI-206](https://linear.app/arius-ai/issue/ARI-206)):** algoritmos de máquina de estados com múltiplos branches dependentes de flags merecem trace caso-a-caso ANTES de virar instrução. Bug em algoritmo proposto só apareceu com cenário ambíguo.
- **Nível 3 — cartografia arquitetural ([ARI-207](https://linear.app/arius-ai/issue/ARI-207)):** decisões com 4 caminhos articulados merecem cartografia de Etapa 1 antes de escolha. Caminho E (1 linha) emergiu da investigação empírica, não estava nos 4 originais.
- **Nível 4 — pipeline cross-repo ([ARI-209](https://linear.app/arius-ai/issue/ARI-209)):** instruções para implementação cross-cutting devem incluir teste e2e que exercita pipeline completo. Bug em re-stamp do `ProcessHeartbeat` só apareceu via teste e2e cross-repo.

### Aprendizado arquitetural consolidado

Consistência entre instâncias merece atenção especial. [ARI-164](https://linear.app/arius-ai/issue/ARI-164) (instance vs class attribute), [ARI-209](https://linear.app/arius-ai/issue/ARI-209) (re-stamp dropando campos) — ambos eram bugs em "fronteiras de instância" (factory→instâncias / DTO→domain→DTO). Lição: para CBs e estado compartilhado em geral, sempre testar com pipeline completo, não só unidades isoladas.

### Aprendizado de método — validação cross-source na migração de documentação

Durante a migração do §3 do Linear para o GitHub (2026-05-02), Codex no VS Code pegou divergências em referências de path/linha reconstruídas a partir de transcript:

- Factory `create_circuit_breaker` real está em `circuit_breaker.py:241-267` (não `:145-171` como reconstruído)
- Pool psycopg callback `check` real está na linha 143 de `postgres_checkpointer.py`
- `_PostgresCBListener` vive em `postgres_checkpointer.py` (não em `circuit_breaker.py`)

**Lição:** descrições de código reconstruídas via memória ou transcript devem ser **validadas contra o código real** antes de virar documentação canônica. Padrão H aplicado à documentação. Convenção a adotar para próximos conceitos: instruções para Codex incluem "abra arquivo X e valide Y antes do append".

### Marco arquitetural

§3 Circuit Breakers é o **primeiro conceito da plataforma Arius a alcançar 🟢 estrito**. Estabelece padrão de qualidade para os 6 conceitos centrais restantes (HealthScore, QualityScore, SLOs, Audit Log, Heartbeat, Telemetry) e os 3 arquiteturais (Multi-tenancy, PEP, Onion Architecture).

Em 2026-05-02 sessão 4, §3 foi migrado do Linear (que comprometeu o save monolítico) para o GitHub `erickmarinho-notebook` em 7 chunks via Codex no VS Code, com validação cross-source contra código real em cada chunk.

---

## Histórico

- **2026-04-30 (criação):** §3 redigido pela primeira vez no Linear como parte do Reference Arius. Cartografia empírica via Kiro + formalização conceitual. Status inicial: 🟡.
- **2026-05-01 ([ARI-204](https://linear.app/arius-ai/issue/ARI-204) + [ARI-200](https://linear.app/arius-ai/issue/ARI-200) + [ARI-205](https://linear.app/arius-ai/issue/ARI-205) mergeadas):** Bug crítico [ARI-202](https://linear.app/arius-ai/issue/ARI-202) diagnosticado e corrigido (primeiro ADR físico do agent-standard). Padrão G aplicado ao vocabulário CB. Reforço de testes com spy counter.
- **2026-05-02 sessão 1 ([ARI-165](https://linear.app/arius-ai/issue/ARI-165) + [ARI-164](https://linear.app/arius-ai/issue/ARI-164) mergeadas):** Persistência de estado via seed. Push transitions cross-repo.
- **2026-05-02 sessão 2 ([ARI-206](https://linear.app/arius-ai/issue/ARI-206) + [ARI-207](https://linear.app/arius-ai/issue/ARI-207) mergeadas):** Métricas operacionais de calibração. Pool psycopg callback `check` (Caminho E descoberto via cartografia).
- **2026-05-02 sessão 2 final (§3 → 🟢 — [ARI-209](https://linear.app/arius-ai/issue/ARI-209) mergeada cross-repo):** Propagação de configuração estática via heartbeat. Bug em re-stamp detectado por teste e2e. **§3 Circuit Breakers alcança 🟢 estrito — primeiro conceito da plataforma Arius com esse status.** Padrão H validou-se em 4 níveis. Total: 8 issues mergeadas em 4 dias úteis.
- **2026-05-02 sessão 4 — migração para GitHub:** §3 movido do Linear para `erickmarinho-notebook` em 7 chunks via Codex no VS Code. Validação cross-source contra código real em cada chunk corrigiu 3 divergências de path/linha reconstruídas do transcript da sessão da manhã. Aprendizado de método registrado.
