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
10. Onion Architecture (C1-C4) 🟡

---

## Histórico

- **2026-05-02:** Migração do Reference Arius do Linear (que comprometeu o save monolítico) para `erickmarinho-notebook`. §6 Heartbeat migrado primeiro (cartografia arqueológica de 8 áreas, 4 issues criadas como gaps registrados, validação cross-source contra código real durante migração — 2 divergências corrigidas). §3 Circuit Breakers migrado em seguida (8 issues mergeadas, primeiro conceito a alcançar 🟢, validação cross-source corrigiu 3 divergências de path/linha reconstruídas do transcript). 2 conceitos centrais agora 🟢 estrito.
