# §1 HealthScore

> **Status:** 🟢 **Documentado e validado empiricamente.** 7 issues mergeadas em 4 sessões (ARI-192, ARI-191, ARI-160, ARI-157, ARI-158, ARI-159, ARI-215), fórmula consolidada em `formula_version="v1.0-2026-05"`, 4 classifications (`healthy`, `degraded`, `critical`, `unknown`), janela temporal explícita (15min default, parametrizada), p95 substituindo avg como entrada da fórmula, persistência completa em `health_score_history` (15 colunas, tabela autoauditável), invariante D13 (função pura intocada) validada empiricamente em 4 issues consecutivas. **Quarto conceito da plataforma Arius a alcançar 🟢 estrito** (depois de §3 Circuit Breakers, §6 Heartbeat e §10 Onion Architecture).
>
> **Última atualização:** 2026-05-08 (sessão N+5 do plano §1 — ARI-159 + ARI-215 mergeadas, persistência completa)

---

## 1.1 — O que é

**HealthScore** é um índice numérico (0-100) que sintetiza **saúde operacional de um agente em janela temporal explícita**, destilando telemetria de múltiplas dimensões (erros, latência, heartbeat, circuit breakers) em **uma decisão operacional única** acompanhada de classificação semântica.

### Problema concreto que resolve

Sem HealthScore, observability é cega:

- Dashboards podem mostrar verde para agente que nunca enviou heartbeat (caso real — `arius-seed-agent` retornava `score=80 / classification=healthy` apesar de `last_heartbeat_at IS NULL`)
- Equipe operacional precisa cruzar 5+ métricas mentalmente (error rate, p95, last heartbeat, CBs abertos, half-open) para responder "este agente está saudável?"
- Cliente Pulso (Camada 2 — operador/SRE de tenant) não tem sinal único, queryable, auditável

**HealthScore é a camada semântica** que destila telemetria operacional em decisão. Foundational do Pulso Camada 2: HealthScore alimenta dashboards, alertas, SLOs derivados, decisões de paging.

### Diferença com QualityScore (§2 — Cenário C)

HealthScore mede **saúde operacional** (sistema está respondendo? está rápido? está vivo?). QualityScore (§2, futuro) mede **qualidade da resposta do agente** (a resposta está correta? útil? alucina?).

**Cenário C** ([ARI-156](https://linear.app/arius-ai/issue/ARI-156) / [ARI-177](https://linear.app/arius-ai/issue/ARI-177) — ratificado): HealthScore e QualityScore são **dimensões independentes**, não combináveis em score único. Um agente pode ter:

- `health=95 / quality=40` — sistema saudável respondendo besteira (alucinação consistente)
- `health=20 / quality=92` — sistema instável mas respostas corretas quando responde

Forçar combinação em score único apaga sinal operacional. Decisão consciente: separar.

### Inputs e outputs

| Categoria | Item | Tipo | Origem |
| -- | -- | -- | -- |
| **Input** | `error_rate` | float ∈ [0, 1] | KPIResult (Langfuse traces na janela) |
| **Input** | `p95_latency_ms` | int ≥ 0 | KPIResult (Langfuse traces na janela) |
| **Input** | `heartbeat_age_s` | int ≥ 0 | `now() - agent.last_heartbeat_at` |
| **Input** | `cb_open_count` | int ≥ 0 | snapshot de `agent_circuit_breakers` |
| **Input** | `cb_half_open_count` | int ≥ 0 | idem |
| **Constante** | `latency_threshold_ms` | int (default 8000) | config — `LATENCY_THRESHOLD_MS` |
| **Constante** | `heartbeat_timeout_s` | int (default 120) | config — `HEARTBEAT_TIMEOUT_SECONDS` |
| **Constante** | `window_minutes` | int (default 15, range 1-1440) | config — `HEARTBEAT_WINDOW_MINUTES` |
| **Output** | `HealthScore` (value object) | — | retorno de `compute_health_score` |

`HealthScore` value object carrega: `score: int \| None`, `classification: Literal["healthy","degraded","critical","unknown"]`, `breakdown: dict[str, int]` (5 chaves de penalty), metadados (`error_rate`, `p95_latency_ms`, `avg_latency_ms`, `heartbeat_age_s`, `cb_open_count`, `cb_half_open_count`, `formula_version`, `thresholds_used`, `calculated_at`).

---

## 1.2 — Como implementamos

### Fórmula (`formula_version="v1.0-2026-05"`)

`compute_health_score` (`src/domain_services/health_score.py`) é **função pura matemática**:

```python
score = 100
score -= error_penalty       # = error_rate * 50          → até -50
score -= latency_penalty     # = min(1.0, p95 / threshold) * 30  → até -30
score -= heartbeat_penalty   # = min(1.0, age / timeout) * 20    → até -20
score -= cb_open_count * 30
score -= cb_half_open_count * 15
score = max(0, min(100, int(score)))   # clamp final
```

**Decomposição conceitual de cada penalty:**

| Penalty | Peso máx | Sinal operacional |
| -- | -- | -- |
| `error_penalty` | 50 | erros são o sinal mais crítico de saúde — peso dominante |
| `latency_penalty` | 30 | latência percebida pelo usuário — segundo sinal mais crítico |
| `heartbeat_penalty` | 20 | liveness — importa, mas heartbeat=120s não significa "agente morto" instantâneo |
| `cb_open` | 30/CB | dependência confirmadamente fora — saúde degradada por design |
| `cb_half_open` | 15/CB | dependência em recovery — degradação parcial |

**Status dos pesos:** `50/30/20/30/15` são **chutes razoáveis baseados em SRE intuition**, não calibração estatística. Drift documentado em §1.3 + decisão de calibrar pós-6 meses de histórico real (Médio prazo em "Notas e direção futura").

### As 4 classifications

| Score | Classification | Significado |
| -- | -- | -- |
| 80-100 | `healthy` | Saudável (override por CB pode rebaixar para `degraded` mantendo score) |
| 50-79 | `degraded` | Degradação visível — atenção operacional |
| 0-49 | `critical` | Crítico — ação imediata |
| `None` | `unknown` | Sem dados suficientes para classificar — distinção epistêmica |

**Override por CB:** se há CB aberto, classificação **mínima** é `degraded` (mesmo que score calculado caia em faixa `healthy`). Score numérico **não muda** — apenas o label semântico. Rationale: "CB aberto = dependência fora = não é saudável", independente do score matemático.

**`unknown` ≠ score=0** ([ARI-160](https://linear.app/arius-ai/issue/ARI-160), 2026-05-07). Distinção epistêmica:

- `score=0 / classification=critical` → "tenho dados, são todos ruins"
- `score=None / classification=unknown` → "não tenho dados suficientes para opinar"

Antes da [ARI-160](https://linear.app/arius-ai/issue/ARI-160), agentes sem heartbeat ou sem traces caíam em `score=80 / healthy` por default vazio das penalties — falso positivo crítico para Pulso Camada 2. `arius-seed-agent` foi o caso vivo que disparou a issue.

### Casos `unknown` (3)

Lógica de classificação `unknown` vive em **C3 Application** (`compute_health_score.py` use case), **não** em C2 Domain Services (função pura). Decisão D5 da sessão N+2 (2026-05-07): manter função pura intocada, lógica unknown como pré-cálculo no use case.

| # | Condição | Comportamento |
| -- | -- | -- |
| 1 | `agent.last_heartbeat_at IS NULL` | **Short-circuit** — retorna `unknown` sem chamar Langfuse nem CB snapshot |
| 2 | `kpis.has_data == False` AND `heartbeat_age_s > heartbeat_timeout_s` | **After-langfuse** — só decide `unknown` depois de confirmar zero traces na janela |
| 3 | Langfuse indisponível AND `heartbeat_age_s > heartbeat_timeout_s` | **After-langfuse** — degradação dupla (sem KPIs externas + sem heartbeat recente) |

Casos 2 e 3 só são `unknown` se heartbeat também está stale. Heartbeat fresco + zero traces = agente vivo mas ocioso, não desconhecido — score calcula normalmente com `error_rate=0`.

**Caso 1 vivo:** `arius-seed-agent` foi inserido via seed do Observatory sem nunca enviar heartbeat. Pré-[ARI-160](https://linear.app/arius-ai/issue/ARI-160) retornava `score=80 / healthy`. Pós-[ARI-160](https://linear.app/arius-ai/issue/ARI-160) retorna `score=None / classification=unknown`.

### Janela temporal — 15min default

Janela aplica **APENAS em `langfuse.get_traces()`** (Q4 da sessão N+3, 2026-05-08). Não filtra CB states (sempre estado atual) nem heartbeat (sempre `now() - last_heartbeat_at`).

**Configuração** (`config.py`):

```python
heartbeat_window_minutes: int = Field(
    default=15,
    ge=1,
    le=1440,
    description="Janela temporal para KPIs de HealthScore (minutos). Default 15. Range 1-1440 (24h)."
)
```

**Anchor temporal:** `datetime.now(UTC)` no use case (Q7), não no domain service. Mesma chamada propaga para Langfuse query (`from_=now-window`, `to=now`) e para `calculated_at` da persistência (Q15 da sessão N+4).

**Por que 15min:**

- ~7 heartbeats de sinal sob `HEARTBEAT_TIMEOUT=120s` (estatística suficiente sem ser volátil)
- Alinhado com SRE Book 5-30min para "operational health" window
- Reativo (~15min para detectar degradação) sem ruído de janelas curtas (1min seria sensível demais a outliers)
- Range válido 1-1440min permite tuning por necessidade operacional sem reescrita

Decisão consciente: janela única, não multi-window multi-burn-rate. Drift documentado em §1.3.

### p95 vs avg — long tail visível

Pré-[ARI-157](https://linear.app/arius-ai/issue/ARI-157) (2026-05-08): fórmula consumia `avg_latency_ms` como entrada de `latency_penalty`. **Avg esconde long tail.**

Cenário concreto: agente com 100 traces, 95 retornando em 200ms e 5 travando em 30s.

- `avg ≈ 1690ms` → `latency_penalty ≈ min(1, 1690/8000) * 30 = 6.3` → score ainda alto
- `p95 = 30000ms` → `latency_penalty = 1.0 * 30 = 30` (penalty máximo) → score reflete degradação real

**Decisão (Q5 sessão N+3, ratificada):** p95 substituí avg como **entrada da fórmula**. `avg_latency_ms` permanece em `KPIResult` (consumer real em SLO engine — `slos.py:78` ainda usa avg para SLOs declarativos), mas HealthScore usa apenas p95.

**Separação de papéis:** `KPIResult` carrega todas as métricas calculadas a partir dos traces (avg, p95, error rate). `HealthScore` carrega apenas as métricas que **entram na fórmula**. `avg_latency_ms` aparece em `health_score_history` por motivo histórico (Q5 — preservar para análise retrospectiva mesmo não entrando na fórmula).

### Persistência — schema completo pós-[ARI-159](https://linear.app/arius-ai/issue/ARI-159) + [ARI-215](https://linear.app/arius-ai/issue/ARI-215)

`health_score_history` (Postgres, append-only) é **tabela autoauditável**: cada linha reproduz cálculo sem depender de config externa. Schema:

| Coluna | Tipo | Nullable | Origem | Notas |
| -- | -- | -- | -- | -- |
| `id` | uuid | ✗ | gerado | PK |
| `agent_id` | uuid | ✗ | FK `agents.id` | índice composto com `calculated_at` |
| `score` | int | ✓ | calculado | NULL quando `classification=unknown` |
| `classification` | text | ✗ | calculado | uma das 4 |
| `error_rate` | numeric | ✗ | observado | input da fórmula |
| `p95_latency_ms` | int | ✗ | observado | input da fórmula ([ARI-157](https://linear.app/arius-ai/issue/ARI-157)) |
| `avg_latency_ms` | int | ✗ | observado | histórico — não entra na fórmula (Q5) |
| `heartbeat_age_s` | int | ✗ | observado | input da fórmula |
| `cb_open_count` | int | ✗ | observado | input da fórmula |
| `cb_half_open_count` | int | ✗ | observado | input da fórmula |
| `penalty_breakdown` | jsonb | ✗ | calculado | 5 chaves; `{}` quando `unknown` (Q16) |
| `formula_version` | text | ✗ | constante | `"v1.0-2026-05"` |
| `thresholds_used` | jsonb | ✗ | constante | 3 chaves: `latency_threshold_ms`, `heartbeat_timeout_s`, `window_minutes` |
| `score_type` | text | ✗ | constante | `"health"` (default) — distingue de futuro `"quality"` |
| `calculated_at` | timestamp | ✗ | autoritativo do use case | [ARI-215](https://linear.app/arius-ai/issue/ARI-215) — mesmo `datetime.now(UTC)` que ancora janela Langfuse |

**Caso `unknown`:** `score=NULL`, `penalty_breakdown={}` (Q16 — vazio, não 5 chaves zeradas), demais metadados populados normalmente (`error_rate=0`, `p95_latency_ms=0`, etc. — sentinelas que indicam "sem dados"). `formula_version` e `thresholds_used` populados igualmente — útil para forense de "quando isto era unknown, qual versão da fórmula estava ativa?".

**Tabela autoauditável significa:** dada uma linha, é possível reproduzir o score via `compute_health_score(error_rate, p95, age, open, half_open, **thresholds_used)` sem consultar config atual. Decisão consciente — protege análise retrospectiva contra mudança de fórmula/threshold.

### Persistência session-per-task ([ARI-191](https://linear.app/arius-ai/issue/ARI-191), 2026-05-07)

Pré-[ARI-191](https://linear.app/arius-ai/issue/ARI-191): persistência reaproveitava session SQLAlchemy do request HTTP. Compute do score acontece em background task (`asyncio.create_task` no use case `ComputeHealthScore`); session do request poderia já estar fechada quando task tentasse persistir.

**Solução:** session-per-task. Use case abre session nova via `async with session_factory() as session` dentro da task, commita, fecha. Independente do ciclo de vida do request.

**Trade-off:** não há transacionalidade entre cálculo HTTP e persistência. Aceito conscientemente — persistência é informação histórica best-effort, não transação de domínio (mesmo padrão de `ProcessHeartbeat` em §6).

**Validação empírica via [ARI-216](https://linear.app/arius-ai/issue/ARI-216):** teste de integração com DB real está em backlog (bloqueada por [ARI-214](https://linear.app/arius-ai/issue/ARI-214)). Suite atual cobre comportamento via mock; smoke validation manual em desenvolvimento confirma que `health_score_history` recebe linhas para chamadas concorrentes sem session leak.

### Snapshot dead code removido ([ARI-192](https://linear.app/arius-ai/issue/ARI-192), 2026-05-07)

Pré-[ARI-192](https://linear.app/arius-ai/issue/ARI-192): caminho legado salvava snapshot de HealthScore dentro de `agent_snapshots` (campos `health_score`, `health_classification`). Caminho nunca foi consumido após `health_score_history` virar fonte de verdade.

**Limpeza:** colunas removidas via Alembic migration; lógica de write em `ProcessHeartbeat` extinta. Decisão limpou contrato de `agent_snapshots` (responsabilidade restrita a counters delta — §6) e elimina ambiguidade "qual é a fonte de verdade?".

**Lição transferível:** dead code de migrações arquiteturais (HealthScore migrou de "campo em snapshot" para "tabela própria") merece remoção explícita, não tolerância silenciosa. Caso similar registrado em [ARI-211](https://linear.app/arius-ai/issue/ARI-211) (`error_count` cleanup em §6).

### Invariante D13 — função pura intocada

**D13** (decisão arquitetural transversal): `compute_health_score` é **função pura matemática** — recebe inputs primitivos, retorna value object. Lógica condicional (unknown classification, override por CB, persistência) vive em **C3 Application** (use case), não em C2 Domain Services.

**Validação empírica em 4 issues consecutivas (Padrão H aplicado a invariante):**

| Issue | Mudança em `compute_health_score` |
| -- | -- |
| [ARI-160](https://linear.app/arius-ai/issue/ARI-160) (4ª classification `unknown`) | **Zero diff** — lógica unknown ficou em C3 |
| [ARI-191](https://linear.app/arius-ai/issue/ARI-191) (session-per-task) | **Zero diff** — mudança puramente em C3/C4 |
| [ARI-157](https://linear.app/arius-ai/issue/ARI-157) + [ARI-158](https://linear.app/arius-ai/issue/ARI-158) (p95 + janela) | **Rename de parâmetro** (`avg_latency_ms` → `p95_latency_ms`); cálculo idêntico |
| [ARI-159](https://linear.app/arius-ai/issue/ARI-159) (persistência completa) | **Adiciona campo no retorno** (`avg_latency_ms` no value object) — info já calculada em C3, apenas exposta no retorno |

D13 sustentou-se sem custo: cada issue resolveu drift estrutural sem tocar matemática. **Property tests P6, P7, P11, P12 preservados sem mudança de lógica** (apenas rename em P11 ao mudar p95).

**P13 novo (introduzido em [ARI-159](https://linear.app/arius-ai/issue/ARI-159)):** invariante matemático `sum(breakdown.values()) == 100 - score` (antes do clamp final). Garante que `penalty_breakdown` é decomposição exaustiva e fiel do score.

**Lição transferível:** segregar matemática pura (C2) de orquestração (C3) reduz blast radius de mudanças. Padrão validado para extensão a futuro `compute_quality_score` (§2 Cenário C) — função pura espelhada com lógica de classificação no use case.

### Decisões registradas

- [ARI-156](https://linear.app/arius-ai/issue/ARI-156) (Done) — Cenário C ratificado (HealthScore × QualityScore como dimensões independentes)
- [ARI-157](https://linear.app/arius-ai/issue/ARI-157) (Done — mergeada 2026-05-08) — p95 substitui avg na fórmula
- [ARI-158](https://linear.app/arius-ai/issue/ARI-158) (Done — mergeada 2026-05-08) — janela temporal explícita 15min default + parametrização
- [ARI-159](https://linear.app/arius-ai/issue/ARI-159) (Done — mergeada 2026-05-08) — persistência completa em `health_score_history` (15 colunas)
- [ARI-160](https://linear.app/arius-ai/issue/ARI-160) (Done — mergeada 2026-05-07) — 4ª classification `unknown` + 3 casos
- [ARI-177](https://linear.app/arius-ai/issue/ARI-177) (Done) — Cenário C consolidado em decisão arquitetural durável
- [ARI-191](https://linear.app/arius-ai/issue/ARI-191) (Done — mergeada 2026-05-07) — session-per-task na persistência
- [ARI-192](https://linear.app/arius-ai/issue/ARI-192) (Done — mergeada 2026-05-07) — snapshot dead code removido
- [ARI-215](https://linear.app/arius-ai/issue/ARI-215) (Done — mergeada 2026-05-08) — `calculated_at` consolidado como autoritativo do use case
- [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (Done — mergeada 2026-05-09) — popular `total_cost_usd` em `agent_snapshots` via enrichment background task. Cost agora consumível por SLO eval e Pulso Camada 2 (cost continua dimensão separada de HealthScore — §6).
- [ARI-214](https://linear.app/arius-ai/issue/ARI-214) (Backlog) — fixture de DB real para testes de integração (bloqueia [ARI-216](https://linear.app/arius-ai/issue/ARI-216))
- [ARI-216](https://linear.app/arius-ai/issue/ARI-216) (Backlog Medium) — teste de integração [ARI-191](https://linear.app/arius-ai/issue/ARI-191) com DB real
- [ARI-217](https://linear.app/arius-ai/issue/ARI-217) (Backlog Medium 3pts) — smoke tests Postman atualizados
- [ARI-218](https://linear.app/arius-ai/issue/ARI-218) (Backlog Medium 2pts) — investigar heartbeat staleness do `agent-standard`

---

## 1.3 — Está funcionando?

### Evidência empírica disponível

**Suite consolidada (2026-05-08, pós-[ARI-159](https://linear.app/arius-ai/issue/ARI-159) + [ARI-215](https://linear.app/arius-ai/issue/ARI-215)):**

- Observatory: **226/226 testes passing** (suite cresceu de 212 → 226 entre as 7 issues do plano §1, +14 no total)
- Property tests cobrindo invariantes da fórmula:
  - **P6** — score sempre clampa em [0, 100]
  - **P7** — classification consistente com faixa de score
  - **P11** — penalty de latência usa p95 (rename pós-[ARI-157](https://linear.app/arius-ai/issue/ARI-157), lógica preservada)
  - **P12** — heartbeat_penalty satura em `min(1.0, age/timeout)`
  - **P13** ([ARI-159](https://linear.app/arius-ai/issue/ARI-159)) — `sum(breakdown) == 100 - score` (decomposição fiel)
- Unit tests cenário-a-cenário cobrindo:
  - 3 casos `unknown` (short-circuit + after-langfuse × 2)
  - Override de CB sobre `healthy` calculado
  - Janela temporal aplicada apenas em Langfuse query
  - Persistência com `score=NULL` e `breakdown={}` no caso `unknown`

**Smoke validation operacional (acumulado):**

- Pós-[ARI-160](https://linear.app/arius-ai/issue/ARI-160) (2026-05-07): `arius-seed-agent` retorna `score=None / classification=unknown` (validação manual via `GET /intelligence/agents/arius-seed-agent/health-score`). Pré-[ARI-160](https://linear.app/arius-ai/issue/ARI-160) retornaria `score=80 / healthy`.
- Pós-[ARI-191](https://linear.app/arius-ai/issue/ARI-191) (2026-05-07): chamadas concorrentes ao endpoint não geram session leak; `health_score_history` recebe linhas independentes do ciclo de vida do request HTTP.
- Pós-[ARI-157](https://linear.app/arius-ai/issue/ARI-157) + [ARI-158](https://linear.app/arius-ai/issue/ARI-158) (2026-05-08): agente real com long tail (5% traces > 20s) retorna score refletindo degradação (~60-70 vs ~85-90 pré-fix). Janela 15min produz sinal mais reativo que janela 24h sem ruído.
- Pós-[ARI-159](https://linear.app/arius-ai/issue/ARI-159) (2026-05-08): `health_score_history` tem 15 colunas populadas; query SQL reproduz score via `compute_health_score(...)` sem precisar de config atual (validação de "tabela autoauditável").
- Pós-[ARI-215](https://linear.app/arius-ai/issue/ARI-215) (2026-05-08): `calculated_at` na linha persistida bate exatamente com anchor de `from_/to` da query Langfuse correspondente (auditoria temporal sem drift).

### Pipeline end-to-end (validado)

```
GET /intelligence/agents/{slug}/health-score
  → ComputeHealthScore.execute (C3)
  → anchor = datetime.now(UTC)
  → if agent.last_heartbeat_at IS NULL → return UNKNOWN (caso 1)
  → kpis = await langfuse.get_traces(slug, from_=anchor-window, to=anchor)
  → if not kpis.has_data and heartbeat_age > timeout → return UNKNOWN (caso 2/3)
  → cb_states = cb_repo.get_states_for_agent(slug)
  → score_obj = compute_health_score(  # função pura D13
       error_rate=kpis.error_rate,
       p95_latency_ms=kpis.p95_latency_ms,
       heartbeat_age_s=heartbeat_age,
       cb_open_count=cb_states.count_open(),
       cb_half_open_count=cb_states.count_half_open(),
       latency_threshold_ms=config.latency_threshold_ms,
       heartbeat_timeout_s=config.heartbeat_timeout_s,
    )
  → if score_obj.classification == "healthy" and cb_open_count > 0:
       score_obj = score_obj.with_classification("degraded")  # override
  → asyncio.create_task(persist_to_history(score_obj, calculated_at=anchor))  # ARI-191 + ARI-215
  → return HealthScoreResponse(...)
```

### Sinais de problema conhecidos

| Issue | Prioridade | Impacto |
| -- | -- | -- |
| [ARI-216](https://linear.app/arius-ai/issue/ARI-216) | Medium | Teste de integração [ARI-191](https://linear.app/arius-ai/issue/ARI-191) com DB real ainda em backlog (bloqueada por [ARI-214](https://linear.app/arius-ai/issue/ARI-214)). Mock cobre comportamento; defesa em profundidade pendente. |
| [ARI-217](https://linear.app/arius-ai/issue/ARI-217) | Medium 3pts | Smoke tests Postman desatualizados. Não bloqueia funcionalidade — testes unitários (226 passing) cobrem; smoke é defesa em profundidade adicional. |
| [ARI-218](https://linear.app/arius-ai/issue/ARI-218) | Medium 2pts | `agent-standard` (instância real em ambiente dev) com heartbeat staleness intermitente. Investigação separada — pode ser config local ou bug latente do loop. |
| [ARI-213](https://linear.app/arius-ai/issue/ARI-213) | ✅ Done 09/05/2026 | `total_cost_usd` populado via enrichment background task. Pulso Camada 2 desbloqueada. Continua não impactando HealthScore (cost é dimensão separada — §6). |

### Drift documentado (detalhado)

§1 alcançou 🟢 **com drift consciente**, não com perfeição. Seis limitações registradas honestamente, cada uma com rationale e gatilho de melhoria:

#### Pesos da fórmula não-calibrados

- **Estado atual:** `50/30/20/30/15` são chutes razoáveis baseados em SRE intuition (erros > latência > heartbeat; CB aberto pesa mais que half-open).
- **Por que aceitamos hoje:** sem 30+ dias de histórico real em produção, calibração estatística é especulação. Otimizar contra dataset sintético ou sentido comum cria viés.
- **Quando melhorar:** após ~6 meses de dados em `health_score_history`, análise de correlação `penalty_breakdown` × incidentes reais (CB que abriu de fato, latência que gerou ticket, agente que cliente reportou como degradado). Bumpar `formula_version` para `v2.x-YYYY-MM` quando recalibrar — `formula_version` na tabela protege análise retrospectiva.

#### Threshold global, não por agente/tenant

- **Estado atual:** `latency_threshold_ms=8000` fixo para todos os agentes. `heartbeat_timeout_s=120` idem.
- **Por que aceitamos hoje:** agentes hoje na plataforma são similares (FAQ, transação curta) — threshold único é proxy razoável. Adicionar dimensão "threshold por agente" antes de ter 5+ tipos de agente é over-engineering.
- **Quando melhorar:** quando customização por sector/tenant entrar (multi-tenancy §8). Threshold pode virar atributo de `agent` ou `tenant` na DB. Schema atual já permite — `thresholds_used` é jsonb, basta passar valores diferentes para `compute_health_score` por agente.

#### Confiança estatística não diferenciada

- **Estado atual:** `score=82` baseado em 5 traces vs 5000 traces tem **mesma semântica** no value object. `unknown` cobre apenas o extremo (zero traces ou zero heartbeat).
- **Por que aceitamos hoje:** `unknown` já cobre o caso patológico (zero dados); refinar entre "5 traces" e "5000 traces" é refinamento de v2. Cliente Pulso Camada 1 (operador/SRE) não precisa dessa granularidade ainda.
- **Quando melhorar:** adicionar campo `confidence_score: float ∈ [0,1]` ou `min_samples: int` ao value object. Schema da tabela permite extensão (jsonb metadados).

#### Sem multi-window multi-burn-rate

- **Estado atual:** janela única 15min default. Não há "30min + 6h" combinados como SRE Book sugere para alerting maduro (cap. 5 — multi-window multi-burn-rate alerts).
- **Por que aceitamos hoje:** janela única já dá sinal operacional honesto. Multi-window é refinamento que faz sentido quando alerting automation entrar (paging on burn rate exige múltiplas janelas para reduzir falso-positivo).
- **Quando melhorar:** quando alerting amadurecer e burn rate virar input de paging. Provavelmente coincide com SLOs §4 ganhar status 🟢.

#### Cost em `agent_snapshots` — resolvido em [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (09/05/2026)

- **Estado atual:** `total_cost_usd` populado via background task post-heartbeat. Use case `EnrichSnapshotKPIs` em Observatory consome traces Langfuse v4 via `kpi_calculator` e faz UPDATE no snapshot recém-criado.
- **Princípio arquitetural preservado:** cost continua dimensão separada de HealthScore (Cenário C). Resolução muda o estado do dado (populado vs zerado), não muda o princípio. HealthScore não absorve cost — §1 mantém foco em saúde operacional.
- **Desbloqueio:** SLO eval consegue ler cost real (antes lia 0, gerando bug silencioso). Pulso Camada 2 (cliente direto) consegue ver tendência de custo acumulado por agente via `agent_snapshots`.

#### Smoke tests Postman desatualizados ([ARI-217](https://linear.app/arius-ai/issue/ARI-217))

- **Estado atual:** coleção Postman não foi atualizada após [ARI-157](https://linear.app/arius-ai/issue/ARI-157)/[ARI-158](https://linear.app/arius-ai/issue/ARI-158)/[ARI-159](https://linear.app/arius-ai/issue/ARI-159) — alguns asserts batem em schema antigo do `HealthScoreResponse`.
- **Por que aceitamos hoje:** testes unitários (226 passing) cobrem funcionalidade. Postman é **defesa em profundidade** (smoke contra ambiente real), não substituto.
- **Quando melhorar:** [ARI-217](https://linear.app/arius-ai/issue/ARI-217) — atualizar coleção alinhada com schema atual + adicionar caso `unknown` ao smoke.

---

## 1.4 — Standard envia o necessário?

### Contrato cliente-servidor

HealthScore **não tem contrato direto agent → Observatory** — é cálculo derivado no Observatory a partir de **3 fontes que o agent já alimenta** (cobertas em §6 Heartbeat e §3 Circuit Breakers):

| Fonte | Dimensão | Documentado em | Status do contrato |
| -- | -- | -- | -- |
| `agent.last_heartbeat_at` | Liveness (heartbeat_age_s) | §6 Heartbeat | ✅ funcional |
| `agent_circuit_breakers` (estado atual) | CBs (open_count, half_open_count) | §3 Circuit Breakers | ✅ funcional |
| Langfuse traces (janela 15min) | Telemetria (error_rate, p95_latency_ms, avg_latency_ms) | §7 Telemetry (🔴 não documentado ainda) | ✅ funcional, contrato pendente formalização em §7 |

### Endpoint que materializa HealthScore

`GET /intelligence/agents/{slug}/health-score` — implementado em `src/infrastructure/routers/intelligence.py`:

```python
class HealthScoreResponse(BaseModel):
    agent_slug: str
    score: int | None                # NULL quando unknown
    classification: Literal["healthy", "degraded", "critical", "unknown"]
    breakdown: dict[str, int]        # 5 chaves; {} quando unknown
    error_rate: float
    p95_latency_ms: int
    avg_latency_ms: int              # ARI-159 — histórico, não input
    heartbeat_age_s: int
    cb_open_count: int
    cb_half_open_count: int
    formula_version: str             # "v1.0-2026-05"
    thresholds_used: dict[str, int]  # 3 chaves
    calculated_at: datetime          # ARI-215
    window_minutes: int              # ARI-158 — eco do parâmetro de janela usado
```

Endpoint dispara `ComputeHealthScore` que persiste linha em `health_score_history` (background task — [ARI-191](https://linear.app/arius-ai/issue/ARI-191)) e retorna response síncrona com snapshot do cálculo. **Cliente Pulso Camada 2 consome esse endpoint.**

### Mudanças consolidadas pelo plano §1

| Aspecto | Pré-fix | Pós-fix | Issue |
| -- | -- | -- | -- |
| Classification cobre zero-data? | Não — caía em `healthy` | Sim — retorna `unknown` | [ARI-160](https://linear.app/arius-ai/issue/ARI-160) |
| Latência reflete long tail? | Não — usava avg | Sim — usa p95 | [ARI-157](https://linear.app/arius-ai/issue/ARI-157) |
| Janela explícita? | Não — implícita "todos os traces recentes" | Sim — 15min parametrizável | [ARI-158](https://linear.app/arius-ai/issue/ARI-158) |
| Persistência audita o cálculo? | Parcial — 8 colunas | Completa — 15 colunas autoauditáveis | [ARI-159](https://linear.app/arius-ai/issue/ARI-159) |
| `calculated_at` reflete anchor real? | Não — gerado no commit, drift vs Langfuse query | Sim — autoritativo do use case | [ARI-215](https://linear.app/arius-ai/issue/ARI-215) |
| Persistência session-safe? | Não — reaproveitava session do request | Sim — session-per-task | [ARI-191](https://linear.app/arius-ai/issue/ARI-191) |
| Snapshot legado convivia? | Sim — `agent_snapshots.health_score` (dead code) | Não — limpo via Alembic | [ARI-192](https://linear.app/arius-ai/issue/ARI-192) |

Sete drifts identificados e fechados em 4 sessões (2026-05-07 a 2026-05-08). Cada um com lição transferível para futuros conceitos da plataforma.

### Conclusão da seção 1.4

Contrato **funcional para Camada 1** (operador/SRE de admin Arius) e **funcional para Camada 2** (cliente Pulso direto consumindo `GET /intelligence/agents/{slug}/health-score`). [ARI-213](https://linear.app/arius-ai/issue/ARI-213) (cost zerado em `agent_snapshots`, resolvida 09/05/2026) nunca impactou HealthScore porque cost é dimensão separada (decisão arquitetural deliberada — §1 não absorve cost, espelhando Cenário C HealthScore × QualityScore).

Próximas evoluções (calibração de pesos, threshold por agente, confidence score, multi-window) não são bloqueios para 🟢 — são **roadmap** de v2 documentado em §1.3 drift.

---

## Caminho para 🟢 (Interpretação A estrita)

§1 alcançou 🟢 em 2026-05-08 com todos os itens fechados:

- [X] §1.1, §1.2, §1.3, §1.4 redigidos com base em fato (não memória)
- [X] [ARI-156](https://linear.app/arius-ai/issue/ARI-156) + [ARI-177](https://linear.app/arius-ai/issue/ARI-177) (Cenário C ratificado — HealthScore × QualityScore independentes)
- [X] [ARI-160](https://linear.app/arius-ai/issue/ARI-160) mergeada (4ª classification `unknown` + 3 casos, 2026-05-07)
- [X] [ARI-191](https://linear.app/arius-ai/issue/ARI-191) mergeada (session-per-task na persistência, 2026-05-07)
- [X] [ARI-192](https://linear.app/arius-ai/issue/ARI-192) mergeada (snapshot dead code removido, 2026-05-07)
- [X] [ARI-157](https://linear.app/arius-ai/issue/ARI-157) mergeada (p95 substitui avg na fórmula, 2026-05-08)
- [X] [ARI-158](https://linear.app/arius-ai/issue/ARI-158) mergeada (janela 15min default + parametrização, 2026-05-08)
- [X] [ARI-159](https://linear.app/arius-ai/issue/ARI-159) mergeada (persistência completa em 15 colunas, 2026-05-08)
- [X] [ARI-215](https://linear.app/arius-ai/issue/ARI-215) mergeada (`calculated_at` consolidado autoritativo, 2026-05-08)
- [X] Fórmula consolidada em `formula_version="v1.0-2026-05"` com decomposição fiel via P13
- [X] Invariante D13 validada empiricamente em 4 issues consecutivas (zero diff, rename, adição de campo)
- [X] Tabela `health_score_history` autoauditável (reproduz cálculo sem config externa)
- [X] Drift documentado conscientemente (6 limitações com rationale + gatilho de melhoria)
- [X] Suite consolidada validada (Observatory 226 passing, +14 testes pelo plano §1)
- [X] Smoke validation operacional acumulado das 7 issues
- [X] 4 issues abertas registradas como gaps conscientes ([ARI-216](https://linear.app/arius-ai/issue/ARI-216), [ARI-217](https://linear.app/arius-ai/issue/ARI-217), [ARI-218](https://linear.app/arius-ai/issue/ARI-218), [ARI-213](https://linear.app/arius-ai/issue/ARI-213))

**Tempo real:** 5 sessões úteis (2026-05-06 a 2026-05-08). 7 issues mergeadas + 4 issues registradas como roadmap + invariante D13 validada empiricamente.

---

## Notas e direção futura

### Curto prazo (issues abertas hoje)

| Issue | Escopo | Bloqueio |
| -- | -- | -- |
| [ARI-216](https://linear.app/arius-ai/issue/ARI-216) | Teste de integração [ARI-191](https://linear.app/arius-ai/issue/ARI-191) com DB real | Bloqueada por [ARI-214](https://linear.app/arius-ai/issue/ARI-214) (fixture de DB real) |
| [ARI-217](https://linear.app/arius-ai/issue/ARI-217) (Medium 3pts) | Smoke tests Postman atualizados (incluir caso `unknown`) | — |
| [ARI-218](https://linear.app/arius-ai/issue/ARI-218) (Medium 2pts) | Investigar heartbeat staleness do `agent-standard` em ambiente dev | Pode ser config local; investigação isolada |

Nenhuma das três bloqueia 🟢 de §1 — todas são defesa em profundidade ou investigação operacional.

### Médio prazo (próximos 6 meses)

- **Calibração empírica de pesos** após 30+ dias de histórico em produção. Análise de correlação `penalty_breakdown` × incidentes reais. Bumpar `formula_version` para `v2.x-YYYY-MM`. Tabela `health_score_history` protege análise retrospectiva — `formula_version` por linha permite comparar cohorts pré/pós-recalibração.
- **Threshold por agente/tenant** quando multi-tenancy §8 entrar. Schema atual permite — `thresholds_used` é jsonb, basta source diferente por agente. Mudança puramente em C3.
- **Confidence score ou min_samples** para diferenciar `score=82 / 5 traces` de `score=82 / 5000 traces`. Cliente Pulso Camada 2 pode querer filtrar baixa confiança.

### Longo prazo (refinamentos avançados)

- **Multi-window multi-burn-rate** (SRE Book cap. 5) quando alerting automation entrar. Burn rate combinado em "5min + 1h" é input padrão para paging maduro.
- **Alerting automation** (page on burn rate) — provavelmente coincide com SLOs §4 ganhar 🟢.
- **Drift detection** — HealthScore mudou significativamente vs baseline histórico do mesmo agente? Sinal proativo de degradação antes de score cair em `degraded`.
- **§2 QualityScore** como dimensão independente (Cenário C ratificado em [ARI-156](https://linear.app/arius-ai/issue/ARI-156) / [ARI-177](https://linear.app/arius-ai/issue/ARI-177)). Função pura espelhada `compute_quality_score` reutilizando padrão D13. Pré-requisito: agent-spec definido + telemetria de qualidade (judge LLM, feedback humano, eval automatizado).

### Para o SDK arius-agent-core (Fase 3)

`AriusHealthScore` (futuro) deve abstrair:

- **Função pura matemática como SDK export** (`compute_health_score` reutilizável por outros agents da plataforma sem fork)
- **Configuração de pesos como parâmetros**, não hardcode (lição de drift de pesos não-calibrados — SDK não pode prescrever pesos universais)
- **`formula_version` versionado pelo SDK** — clientes do SDK herdam versão; release notes documentam recalibrações
- **Persistência em `health_score_history` como contrato opcional** — SDK fornece schema canônico; consumer decide se persiste em Postgres, ClickHouse, ou em lugar nenhum
- **Hooks de override** (ex: classification override por CB é pattern transferível mas pode ter regras específicas por domínio)

### Para Pulso (Camada 2)

- HealthScore alimenta dashboards, alertas, SLOs derivados de Pulso Camada 2 (cliente direto = operador/SRE de tenant Arius)
- **Tabela autoauditável** é diferencial vendável: cliente pode reconstruir cálculo histórico sem confiar em config server-side mutável
- **`unknown` distinto de `score=0`** evita falso positivo em onboarding (agente novo sem traces ainda não é "crítico")
- **Roadmap de calibração** dá ao cliente confiança de que pesos vão evoluir baseado em dados reais, não chutes permanentes

### Aprendizado de processo — função pura como invariante prospectivo

§3 (CB) e §6 (Heartbeat) validaram **regra de dependência Onion** como invariante prospectivo (cf. §10). §1 (HealthScore) valida **função pura matemática separada de orquestração** (D13) como invariante prospectivo dentro de C2/C3.

**Lição transferível para §2 QualityScore, §4 SLOs, §5 Audit Log:** quando há cálculo agregando múltiplos inputs em valor único, separar matemática (C2) de orquestração (C3). Reduz blast radius de drift estrutural — 4 issues consecutivas em §1 não tocaram a função pura, mesmo introduzindo `unknown` (mudança de tipo de retorno semântica), p95 (mudança de input semântica) e persistência completa (mudança de superfície do value object).

### Aprendizado arquitetural — drift documentado vs drift escondido

§1 alcança 🟢 com **6 limitações documentadas honestamente**, não com perfeição. Padrão validado em §6 (4 issues registradas como gaps) e §10 (drift estrutural cross-repo). **Tornar drift visível é parte do produto** (ADR-000 Implicação 2 aplicada à própria documentação): cliente Pulso Camada 2 lendo §1 entende o que está calibrado, o que é chute consciente e quando esperar evolução. Drift escondido é dívida; drift documentado é roadmap.

### Aprendizado de método — cartografia profunda em conceito agregador

Diferente de §10 (invariante binário, cartografia leve em ~12min) e §6 (arqueologia de conceito antigo, ~3h), §1 exigiu **5 sessões úteis** porque é conceito agregador: combina 3 fontes (heartbeat, CBs, Langfuse), 2 camadas Onion (C2 função pura + C3 orquestração), 1 schema de persistência completo, 1 contrato HTTP derivado e 1 decisão arquitetural transversal (Cenário C HealthScore × QualityScore). **Conceitos agregadores merecem múltiplas sessões** — tentar fechar em uma única corre risco de drift escondido.

### Marco arquitetural

§1 HealthScore é o **quarto conceito da plataforma Arius a alcançar 🟢 estrito** (depois de §3 Circuit Breakers, §6 Heartbeat e §10 Onion Architecture). Combinado com os anteriores, estabelece padrão validado de:

- **Função pura como invariante** (D13) — transferível para §2 QualityScore
- **Tabela autoauditável** com `formula_version` + `thresholds_used` — transferível para §4 SLOs
- **`unknown` como classificação epistêmica distinta de score baixo** — transferível para qualquer score derivado da plataforma
- **Drift documentado em 6 dimensões** com rationale + gatilho de melhoria — modelo para §2, §4, §5, §7

Status epistêmico cumulativo da plataforma: **4/10 🟢, 0/10 🟡, 5/10 🔴, 1/10 ⚪** (depois de migração para GitHub).

---

## Histórico

- **2026-05-06 (sessão N — plano §1 consolidado):** Plano implementacional do §1 HealthScore consolidado a partir de cartografia inicial. 7 issues abertas: [ARI-160](https://linear.app/arius-ai/issue/ARI-160), [ARI-191](https://linear.app/arius-ai/issue/ARI-191), [ARI-192](https://linear.app/arius-ai/issue/ARI-192), [ARI-157](https://linear.app/arius-ai/issue/ARI-157), [ARI-158](https://linear.app/arius-ai/issue/ARI-158), [ARI-159](https://linear.app/arius-ai/issue/ARI-159), [ARI-215](https://linear.app/arius-ai/issue/ARI-215). Cenário C (HealthScore × QualityScore independentes) ratificado em [ARI-156](https://linear.app/arius-ai/issue/ARI-156) / [ARI-177](https://linear.app/arius-ai/issue/ARI-177). Status inicial: 🔴.
- **2026-05-07 (sessão N+1, N+2 — primeira tríade fechada):** [ARI-192](https://linear.app/arius-ai/issue/ARI-192) (snapshot dead code removido), [ARI-191](https://linear.app/arius-ai/issue/ARI-191) (session-per-task na persistência), [ARI-160](https://linear.app/arius-ai/issue/ARI-160) (4ª classification `unknown` + 3 casos) mergeadas. Caso vivo `arius-seed-agent` corrigido (pré: `score=80 / healthy`; pós: `score=None / unknown`). Invariante D13 validada nas 3 issues (zero diff em `compute_health_score`).
- **2026-05-08 sessão N+3 — p95 + janela ([ARI-157](https://linear.app/arius-ai/issue/ARI-157) + [ARI-158](https://linear.app/arius-ai/issue/ARI-158) mergeadas):** p95 substitui avg na fórmula (long tail visível). Janela temporal explícita 15min default, parametrizada via `HEARTBEAT_WINDOW_MINUTES` (range 1-1440min). Anchor `datetime.now(UTC)` no use case (Q7). Invariante D13 sustentou-se com rename de parâmetro apenas.
- **2026-05-08 sessão N+4 — persistência completa ([ARI-159](https://linear.app/arius-ai/issue/ARI-159) + [ARI-215](https://linear.app/arius-ai/issue/ARI-215) mergeadas):** `health_score_history` ganha 15 colunas (tabela autoauditável). `calculated_at` consolidado como autoritativo do use case (mesmo anchor da query Langfuse). `penalty_breakdown={}` no caso `unknown` (Q16). P13 novo introduzido (`sum(breakdown) == 100 - score`). Invariante D13 sustentou-se com adição de campo no retorno (`avg_latency_ms` exposto, info já calculada em C3).
- **2026-05-08 sessão N+5 (final) — §1 → 🟢:** Reference Arius §1 HealthScore redigido no `erickmarinho-notebook` no padrão validado por §3, §6, §10. Estrutura §1.1-§1.4 + Caminho 🟢 + Notas + Histórico. Drift documentado em 6 dimensões (pesos não-calibrados, threshold global, confiança não diferenciada, sem multi-window, cost zerado, smoke Postman desatualizado). 4 issues abertas registradas como roadmap consciente. **§1 HealthScore alcança 🟢 estrito — quarto conceito da plataforma Arius com esse status.** Padrão "função pura matemática como invariante" estabelecido como transferível para §2, §4, §5.
- **2026-05-09 — Cost dimension consumível em [ARI-213](https://linear.app/arius-ai/issue/ARI-213):** `total_cost_usd` agora populado em `agent_snapshots` via enrichment background task post-heartbeat (detalhes em §6). Princípio arquitetural Cenário C preservado: HealthScore continua não absorvendo cost. Mudança é de estado de dado (populado vs zerado), não de fórmula. SLO eval em cost passa a funcionar (antes era bug silencioso lendo 0). Pulso Camada 2 ganha série temporal de custo consumível por dashboards.
