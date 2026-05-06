# §10 Onion Architecture (C1-C4)

> **Status:** 🟢 **Documentado e validado empiricamente.** Cartografia cross-repo executada em 2026-05-06 (15min): estrutura de camadas mapeada em ambos os repos, regra de dependência validada via grep (zero violações), 8 arquivos representativos verificados. Drift estrutural entre repos documentado honestamente. **Terceiro conceito da plataforma Arius a alcançar 🟢 estrito.**
>
> **Última atualização:** 2026-05-06 (cartografia leve cross-repo)

---

## 10.1 — O que é

**Onion Architecture** é um padrão arquitetural de organização de código em **camadas concêntricas**, onde dependências apontam **APENAS para o centro**. Camadas externas dependem de camadas internas via interfaces; camadas internas não conhecem nem dependem de camadas externas.

### 4 camadas canônicas

| Camada | Nome | Responsabilidade | Conhece quem? |
| -- | -- | -- | -- |
| **C1** | Domain | Entidades, value objects, contratos, exceções, interfaces (Protocols) | Ninguém — coração puro |
| **C2** | Domain Services | Lógica de negócio pura entre múltiplos domain objects | Apenas C1 |
| **C3** | Application | Use cases (orquestração), workflows, services de aplicação | C1 + C2 (via interfaces) |
| **C4** | Infrastructure | Adapters concretos (DB, HTTP, LLM, queue), implementações de interfaces de C1 | C1 + C2 + C3 (implementa interfaces) |

**Camada de presentation** (HTTP routers, CLI, entry points) é tradicionalmente classificada como **C5 separada** OU **parte de C4**. Ambas as classificações são defensáveis. Drift entre os 2 repos da plataforma Arius documentado em §10.2.

### Regra de dependência única

Único invariante que sustenta Onion:

> **`from src.{outer_layer} import ...` é PROIBIDO dentro de qualquer arquivo em `src/{inner_layer}/`.**

Concretamente:

- ❌ `src/domain/` NUNCA importa de `src/domain_services/`, `src/application/`, `src/infrastructure/`
- ❌ `src/domain_services/` NUNCA importa de `src/application/`, `src/infrastructure/`
- ❌ `src/application/` NUNCA importa de `src/infrastructure/` (acesso APENAS via interfaces de C1)
- ✅ `src/infrastructure/` PODE importar de C1, C2, C3 (implementa interfaces, executa use cases)

### Por que importa

**Sem regra de dependência:** mudança em SQL/HTTP/LLM provider força recompilação/refatoração de domínio. Domínio fica acoplado a tecnologia. Testes de domínio precisam de DB real ou mocks sofisticados. Substituir adapter (ex: trocar Postgres por DynamoDB, OpenAI por Anthropic) vira migração épica.

**Com regra de dependência:** domínio é o que NUNCA muda — entidades, regras, contratos. Adapters concretos (C4) podem ser trocados sem tocar lógica de negócio. Testes de domínio são puros (sem mocks de I/O). Um novo provider entra como nova implementação de interface, zero impacto no resto.

**Onion ≠ DDD ≠ Clean Architecture**, mas todos compartilham essa regra. Aplicar Onion **estritamente** já entrega 80% do valor arquitetural.

---

## 10.2 — Como implementamos

### Cartografia cross-repo (2026-05-06)

**agent-standard** (`~/Arius/arius-agent-standard/src/`):

```
src/
├── domain/                    # C1
│   ├── contracts/             #   schemas Pydantic (cb_state, heartbeat)
│   ├── entities/              #   agent_state, message
│   ├── exceptions.py
│   ├── interfaces.py          #   Protocols (ILLMAdapter, etc.)
│   └── outcomes.py
├── domain_services/           # C2
│   ├── routing.py             #   pure routing (route_on_error, route_by_intent)
│   ├── state_validation.py
│   ├── taxonomy.py
│   └── tracing_helpers.py
├── application/               # C3
│   ├── nodes/                 #   nodes do LangGraph (entry, processing, synthesis)
│   ├── pipeline/              #   graph.py
│   └── services/              #   agent_service.py orquestra
├── infrastructure/            # C4
│   ├── config/
│   ├── llm/                   #   adapters (anthropic, openai, groq, gemini, factory)
│   ├── observability/
│   ├── persistence/           #   postgres_checkpointer.py
│   ├── prompts/
│   └── resilience/            #   circuit_breaker.py
└── presentation/              # C5 (explícita)
    ├── api/                   #   dependencies, health, routes, schemas
    └── app.py                 #   entry FastAPI
```

**Observatory** (`~/Arius/arius-observatory/src/`):

```
src/
├── domain/                    # C1 (flat, sem subpastas)
│   ├── entities.py
│   ├── exceptions.py
│   ├── interfaces.py          #   Protocols (ILangfusePort, IAgentRepo, etc.)
│   └── value_objects.py
├── domain_services/           # C2
│   ├── cb_metrics.py
│   ├── extract_hierarchy.py
│   ├── health_score.py
│   ├── kpi_calculator.py
│   ├── status_engine.py       #   compute_status (pura)
│   └── taxonomy.py
├── application/               # C3 (flat — 1 módulo por use case)
│   ├── agents.py              #   ProcessHeartbeat, RegisterAgent
│   ├── intelligence.py        #   ComputeHealthScore
│   ├── observations.py
│   ├── process_cb_transition.py
│   ├── scores.py
│   ├── slos.py
│   └── sweep_stale_agents.py  #   ARI-210 (background sweep)
└── infrastructure/            # C4
    ├── adapters/              #   langfuse_client.py
    ├── auth.py
    ├── config.py
    ├── db.py
    ├── dependencies.py
    ├── models.py              #   SQLAlchemy ORM
    ├── repositories/          #   7 repos (agent, audit, cb, snapshot, last_counters, slo, score)
    ├── routers/               #   6 routers FastAPI (agents, health, intelligence, langfuse, observations, slos)
    └── schemas.py             #   Pydantic request/response

# Entry point: main.py na raiz do repo (fora de src/)
```

### Drift estrutural cross-repo (documentado honestamente)

Cartografia revelou **3 diferenças factuais** entre os 2 repos. Nenhuma viola Onion — todas são convenções de organização interna defensáveis.

| Aspecto | agent-standard | Observatory | Natureza do drift |
| -- | -- | -- | -- |
| **C1 Domain** | Subpastas (`contracts/`, `entities/`) | Flat (`entities.py`, `value_objects.py`) | Convenção |
| **C3 Application** | Subpastas por padrão LangGraph (`nodes/`, `pipeline/`, `services/`) | Flat por use case (`agents.py`, `intelligence.py`, ...) | Convenção |
| **Presentation** | C5 explícita (`src/presentation/`) | Routers em C4 (`src/infrastructure/routers/`) + `main.py` na raiz | **Estrutural** |

**Drift de presentation é o mais notável.** Não é violação:

- agent-standard trata HTTP como "interação com mundo externo" → camada explícita
- Observatory trata routers como "infrastructure que adapta protocolo HTTP para use cases" → parte de C4

Ambas as classificações são defensáveis em literatura clássica de Onion. Decisão consciente: **documentar drift sem forçar uniformização**. Re-avaliar quando SDK arius-agent-core (Fase 3) for extraído — SDK pode forçar convergência se compartilhar código de presentation entre agents.

### Regra de dependência (validada empiricamente)

Cartografia executou queries explícitas em ambos os repos para validar regra de dependência:

| Query | agent-standard | Observatory |
| -- | -- | -- |
| `domain` importa de `infrastructure`? | ✅ zero | ✅ zero |
| `domain` importa de `application`? | ✅ zero | ✅ zero |
| `domain` importa de `domain_services`? | ✅ zero | ✅ zero |
| `domain_services` importa de `infrastructure`? | ✅ zero | ✅ zero |
| `domain_services` importa de `application`? | ✅ zero | ✅ zero |
| `application` importa de `infrastructure`? | ✅ zero | ✅ zero |
| `application` importa de `presentation`? | ✅ zero | n/a |

**Zero violações em ambos os repos.** C3/Application acessa C4 **apenas via interfaces de C1 + DI** (Dependency Injection no construtor). Padrão Onion respeitado estritamente.

### Exemplos representativos por camada

#### agent-standard

**C1 Domain — `src/domain/contracts/heartbeat.py`:**
- Schemas Pydantic (`CounterSnapshot`, `CircuitBreakerSnapshot`, `HeartbeatPayload`)
- Imports: apenas `pydantic` e `src.domain.contracts.cb_state`
- ✅ Domain puro

**C2 Domain Services — `src/domain_services/routing.py`:**
- Funções puras `route_on_error`, `route_by_intent`
- Imports: `src.domain.entities.agent_state`
- ✅ Apenas C1

**C3 Application — `src/application/services/agent_service.py`:**
- Orquestra pipeline LangGraph com `@observe()` Langfuse
- Imports: `src.application.services._context`, `src.domain.exceptions`
- ✅ C1 + C3 interno (sem C4)

**C4 Infrastructure — `src/infrastructure/llm/anthropic_adapter.py`:**
- Implementa `ILLMAdapter` (interface de C1) com pybreaker + tenacity
- Imports: `src.domain.exceptions`, `src.domain.interfaces`, `src.infrastructure.resilience`
- ✅ C4 implementa interface de C1, depende de outros módulos de C4

#### Observatory

**C1 Domain — `src/domain/interfaces.py`:**
- Protocols (`ILangfusePort`, `IAgentRepo`, etc.)
- Imports: apenas `src.domain.entities`, `src.domain.value_objects`
- ✅ Domain puro

**C2 Domain Services — `src/domain_services/cb_metrics.py`:**
- Função pura `calculate_cb_metrics`
- Imports: `src.domain.value_objects`
- ✅ Apenas C1

**C3 Application — `src/application/observations.py`:**
- Use case `GetClientOverview` recebe `IAgentRepo` e `ILangfusePort` por DI
- Imports: `src.domain.interfaces`, `src.domain_services.extract_hierarchy`
- ✅ C1 + C2 (via interfaces, sem C4)

**C4 Infrastructure — `src/infrastructure/repositories/agent_repository.py`:**
- Implementa `IAgentRepo` com SQLAlchemy
- Imports: `src.domain.entities`, `src.domain.value_objects`, `src.infrastructure.models`
- ✅ C4 implementa interface de C1, depende de outros módulos de C4

---

## 10.3 — Está funcionando?

### Evidência empírica disponível (cartografia 2026-05-06)

**Validação de regra de dependência (cross-repo):**

7 queries `grep -rn` executadas em ambos os repos cobrindo todas as direções proibidas (C1 → outras camadas, C2 → camadas externas, C3 → C4, C3 → presentation). **Resultado: zero violações em ambos os repos.**

**Validação de exemplos representativos:**

8 arquivos representativos (4 por repo, 1 por camada) inspecionados manualmente. Todos os imports respeitam regra de dependência:

- C1 importa apenas pydantic e outros módulos de C1
- C2 importa apenas C1
- C3 importa C1 + C2 (via interfaces)
- C4 implementa interfaces de C1, depende de C1 + C2 + outros módulos de C4

**Padrão Onion respeitado estritamente em código de produção.**

### Consequências práticas observadas

- **Substituição de adapters trivial:** ARI-209 propagou `fail_max`/`reset_timeout_seconds` cross-repo sem tocar lógica de domínio. ARI-211 removeu campo de schema sem refactor de StatusEngine. ARI-212 envolveu loop em try/except sem mudar use case. ARI-210 introduziu primeira background task no Observatory sem violação arquitetural — segue padrão do `ProcessHeartbeat`.
- **Testes de domínio puros:** Property tests em `test_status_engine_property.py`, `test_health_score_property.py`, `test_cb_metrics.py` rodam **sem mocks de I/O**. C1 e C2 são testáveis isoladamente (lição transferível para futuros conceitos).
- **DI no construtor é convenção universal:** Todos os use cases (`ProcessHeartbeat`, `ComputeHealthScore`, `SweepStaleAgents`, `RegisterAgent`) recebem dependências (`IAgentRepo`, `IAuditLogRepo`, `ILangfusePort`) por construtor. Trocar implementação concreta = trocar uma linha no `dependencies.py`.

### Sinais de problema (zero detectados em 2026-05-06)

- Violações de regra de dependência: ❌ nenhuma
- Imports circulares: ❌ nenhum (consequência da regra)
- Acoplamento de domain a adapter concreto: ❌ nenhum (uso de interfaces)
- Drift estrutural cross-repo: ✅ presente, **documentado conscientemente em §10.2** como decisão consciente, não regressão

### Onion como **invariante prospectivo**

Todo conceito futuro da plataforma Arius (HealthScore §1, QualityScore §2, SLOs §4, Audit Log §5, Telemetry §7, Multi-tenancy §8, PEP §9) **deve respeitar regra de dependência ao ser implementado**. Cartografia §10 valida que padrão funcionou para conceitos atuais (CB §3, Heartbeat §6) — valida que método é viável.

---

## 10.4 — SDK extrai o necessário?

### Contexto: SDK arius-agent-core (Fase 3 — futuro)

Plataforma Arius prevê extração de SDK (`arius-agent-core`) a partir do `arius-agent-standard` quando padrão validar em produção. **§10 Onion é pré-requisito para SDK** — SDK precisa expor abstrações de C1 (interfaces, contracts) e fornecer adapters padrão de C4, sem prescrever C3 (use cases customizados pelo agent específico).

### O que SDK deve abstrair (Fase 3)

Por camada Onion:

| Camada | SDK exporta | SDK NÃO exporta |
| -- | -- | -- |
| **C1 Domain** | Interfaces (`ILLMAdapter`, `IObservatoryClient`, `ICheckpointer`), contracts Pydantic (`HeartbeatPayload`, `CBState`), exceptions canônicas | Entidades específicas de domínio (cada agent define as suas) |
| **C2 Domain Services** | Funções puras transferíveis (ex: `compute_status`, `calculate_cb_metrics` se virarem patterns universais) | Lógica de routing específica (cada agent define sua taxonomy) |
| **C3 Application** | Hooks, base classes opcionais (ex: `BaseHeartbeatLoop`) | Use cases do agent específico — **agent é dono da orquestração** |
| **C4 Infrastructure** | Adapters padrão (LLM providers, CB com pybreaker, Langfuse client, Observatory client) | Adapters específicos do domínio do agent |

### Drift de presentation — implicação para SDK

Cartografia documentou drift entre repos: agent-standard tem `src/presentation/` explícita; Observatory mistura presentation em C4. **SDK precisará tomar posição** quando for extraído:

- **Opção A:** SDK **força convenção** — define `src/presentation/` como camada explícita, padronizando agents futuros
- **Opção B:** SDK **abstrai a decisão** — fornece factory `create_app(use_cases=...)` agnóstico de onde routers vivem
- **Opção C:** SDK **documenta convenção opinativa** mas não força (exemplo de referência apenas)

Decisão fica para Fase 3 (quando SDK for materializado). Documentação atual reconhece o drift sem antecipar a escolha.

### Pré-requisitos arquiteturais do SDK

Antes de SDK ser extraído com confiança:

- ✅ Onion respeitada estritamente (validado em 2026-05-06 — pré-requisito atendido)
- ✅ Interfaces de C1 estáveis (CB, Heartbeat, observability — atendido)
- ⚠️ Padrão de DI uniforme (atendido em ambos os repos via construtor)
- 🔴 Agent-spec definido (não iniciado — pré-requisito de QualityScore §2 também)
- 🔴 Convenção de presentation (decisão pendente — drift atual aceito)

### Conclusão da seção 10.4

**SDK não pode ser extraído antes que agent-spec esteja definido** (decisão arquitetural já tomada). Onion como invariante está **pronta para sustentar SDK** quando momento chegar — drift de presentation é única decisão pendente, e fica para Fase 3.

---

## Caminho para 🟢 (Interpretação A estrita)

§10 alcançou 🟢 em 2026-05-06 com todos os itens fechados:

- [X] §10.1, §10.2, §10.3, §10.4 redigidos com base em fato (cartografia leve cross-repo)
- [X] Estrutura real de C1-C4 mapeada em ambos os repos (agent-standard + Observatory)
- [X] Regra de dependência validada empiricamente via 7 queries `grep -rn` cross-repo
- [X] Zero violações de regra de dependência detectadas
- [X] 8 arquivos representativos verificados (4 por repo, 1 por camada)
- [X] Drift estrutural cross-repo identificado e documentado honestamente (3 diferenças factuais)
- [X] Drift de presentation registrado conscientemente (decisão: não forçar uniformização agora)
- [X] DI no construtor confirmado como convenção universal em ambos os repos
- [X] Onion validada como invariante prospectivo para conceitos futuros (§1, §2, §4, §5, §7, §8, §9)
- [X] Pré-requisitos arquiteturais para SDK (Fase 3) inventariados
- [X] Decisão sobre ADR físico de Onion (não criar — Reference Arius é fonte)
- [X] §10 migrado do Linear para GitHub (cartografia leve substitui conteúdo raso original)

**Tempo real:** ~12 minutos de cartografia + ~30 minutos de redação cross-repo. Cartografia mais leve da plataforma — sem decisões arquiteturais finas, apenas validação empírica do que já estava implementado.

---

## Notas e direção futura

### Onion como invariante prospectivo (não decisão pontual)

Diferente de §3 (CB) e §6 (Heartbeat), §10 não é "conceito que evolui via issues". Onion é **invariante** — implementação respeita ou não respeita; não há gradiente. Cartografia 2026-05-06 validou que invariante está sendo respeitado em ambos os repos. Próximos conceitos da plataforma (HealthScore, QualityScore, SLOs, Audit Log, Telemetry, Multi-tenancy, PEP) **devem** respeitar ao serem implementados.

**Mecanismo de manutenção:** cartografia §10 pode ser re-executada a qualquer momento (queries `grep -rn` são determinísticas). Se violação aparecer no futuro, é regressão arquitetural — vira issue prioritária.

### Drift de presentation — re-avaliação na Fase 3

Cartografia documentou drift entre repos:

- agent-standard: `src/presentation/` explícita (C5)
- Observatory: routers em `src/infrastructure/routers/` + `main.py` na raiz (presentation difusa em C4)

**Decisão consciente atual (2026-05-06):** documentar drift sem forçar uniformização. Re-avaliar na Fase 3 quando SDK arius-agent-core for extraído. SDK pode forçar convergência ou abstrair decisão (Opções A/B/C em §10.4).

### ADR físico de Onion — não criar agora

Discussão registrada em §10.4: SDK Fase 3 pode ou não exigir ADR físico de Onion. Decisão atual: **Reference Arius (este documento) é fonte**. Criar ADR físico antes de SDK seria duplicação. Re-avaliar quando SDK exigir documentação formal externa (ex: clientes Pulso ou auditoria compliance-grade).

### Pré-requisitos para SDK arius-agent-core (consolidados)

| Pré-requisito | Status |
| -- | -- |
| Onion respeitada estritamente em código | ✅ validado (2026-05-06) |
| Interfaces de C1 estáveis | ✅ atendido (CB, Heartbeat, observability mapeados) |
| Padrão DI uniforme | ✅ atendido (construtor em ambos os repos) |
| Agent-spec definido | 🔴 não iniciado (pré-requisito comum a §2 QualityScore) |
| Convenção de presentation | ⚠️ drift aceito conscientemente (decisão pendente para Fase 3) |

**Conclusão:** SDK não pode ser extraído antes de **agent-spec definido** (decisão arquitetural já tomada). Onion como invariante está pronta para sustentar SDK quando momento chegar.

### Aprendizado de método — cartografia leve vs cartografia profunda

Cartografias §3 (CB) e §6 (Heartbeat) levaram ~30-40min cada e revelaram **drift acumulado** (gaps, dead code, decisões implícitas). §10 levou ~12min e revelou **drift estrutural mas sem gaps** — Onion é mais simples de validar empiricamente porque é regra binária (respeita / viola).

**Lição transferível:** cartografia de **invariantes arquiteturais** (regras binárias) é mais leve que cartografia de **conceitos que evoluem** (subdomínios com decisões). Próximos conceitos a documentar (Multi-tenancy §8, PEP §9) podem se beneficiar de cartografias leves se forem invariantes; já HealthScore §1 e SLOs §4 vão exigir cartografias profundas.

### Marco arquitetural

§10 Onion Architecture é o **terceiro conceito da plataforma Arius a alcançar 🟢 estrito** (depois de §3 Circuit Breakers e §6 Heartbeat). Estabelece precedente para cartografia leve cross-repo aplicada a invariantes arquiteturais.

Status epistêmico cumulativo da plataforma: **3/10 🟢, 0/10 🟡, 6/10 🔴, 1/10 ⚪.**

### Notas técnicas sobre formatação

Conteúdo deste documento prioriza fidelidade técnica sobre conformidade estrita ao markdownlint. Warnings de regras estilísticas (MD040 — code blocks sem linguagem; MD032 — listas sem linha em branco antes/depois) podem aparecer em algumas linhas. Conteúdo renderiza corretamente no GitHub. Issue separada de tooling para zerar warnings fica registrada se desejar uniformização total.

---

## Histórico

- **2026-04-30:** §10 criado no Linear durante construção inicial do Reference Arius (sessão 1 da ARI-186). Status inicial: 🟡 (4 frases descrevendo Onion + decisão de ADR físico futuro). Conteúdo raso, sem cartografia.
- **2026-05-06 (cartografia + migração para GitHub):** Cartografia leve cross-repo executada (~12min) via Kiro. Estrutura real de camadas mapeada em ambos os repos, regra de dependência validada via grep (zero violações), 8 exemplos representativos verificados. 1 ajuste pós-cartografia: Observatory tem 6 routers, não 7 (correção via validação cross-source antes do append). Drift estrutural documentado honestamente. **§10 alcança 🟢 estrito — terceiro conceito da plataforma Arius com esse status.** Migração do Linear (conteúdo raso) para GitHub (cartografia densa) feita em 4 chunks via Codex no VS Code, padrão validado em §3 e §6.
