# Arius

Plataforma multi-agente para ERP retail (Arius Sistemas).

## Repositórios

- [arius-agent-standard/](./arius-agent-standard/) — agent canônico (LangGraph + LLM adapters + CB)
- [arius-observatory/](./arius-observatory/) — IDP/observabilidade (FastAPI + PostgreSQL + Langfuse)

## Reference Arius — status epistêmico dos conceitos

> **Legenda:**
> - 🟢 **Documentado e validado empiricamente** — alguém pode operar sem perguntar
> - 🟡 **Parcialmente documentado** — falta evidência ou contrato
> - 🔴 **Não documentado** — apenas existe no código + memória
> - ⚪ **Não iniciado** — conceito direcionado mas sem implementação

### Conceitos centrais (Camada 1 — Persona funcional)

1. HealthScore 🔴
2. QualityScore 🔴
3. [Circuit Breakers](./03-circuit-breakers.md) 🟢
4. SLOs ⚪
5. Audit Log 🔴
6. [Heartbeat](./06-heartbeat.md) 🟢
7. Telemetry (Langfuse + Custom Traces) 🔴

### Conceitos arquiteturais

8. Multi-tenancy / PulsoHierarchy 🔴
9. Policy Enforcement Point (PEP) 🔴
10. [Onion Architecture (C1-C4)](./10-onion-architecture.md) 🟢

---

## Histórico

- **2026-05-02:** Migração inicial do Reference Arius do Linear para `erickmarinho-notebook`. §6 Heartbeat e §3 Circuit Breakers migrados via Codex no VS Code com validação cross-source.
- **2026-05-06:** §10 Onion Architecture migrado com cartografia leve cross-repo (validação de regra de dependência: zero violações em ambos os repos; drift estrutural documentado). 3 conceitos centrais agora 🟢 estrito (§3, §6, §10).
- **2026-05-08:** [ARI-218](https://linear.app/arius-ai/issue/ARI-218) absorvida no §6 Heartbeat (staleness em dev — não-bug, comportamento esperado em ambiente local). Validação positiva das afirmações §6.2; subproduto: `logging.lastResort` validado empiricamente na imagem do agent-standard.
