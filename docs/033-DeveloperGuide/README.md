---
Documento: Índice do Módulo
Módulo: 033-DeveloperGuide
Status: Skeleton
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): A definir na fase de fan-out
Depende de: 031,001
---

# AIOS — Módulo 033 · DeveloperGuide

> **Propósito.** Guia do desenvolvedor: setup, convenções, CI de documentação, lint arquitetural e contribuição.

> **Status:** Esqueleto. Este módulo faz parte do fan-out planejado após a
> aprovação da fundação canônica (`000-Vision`, `001-Architecture`, `040-Glossary`,
> `002-ADR`, `003-RFC`). Cada documento abaixo DEVE ser produzido conforme
> `../_templates/MODULE_TEMPLATE.md` e em conformidade com `../003-RFC/RFC-0001-Architecture-Baseline.md`.

## Dependências

Este módulo depende de: **031,001**. Reutilize as definições do
`../040-Glossary/Glossary.md` e os contratos centrais da RFC-0001 (URN, envelope
de evento, envelope de erro, idempotência, correlação).

## Documentos obrigatórios (26)

| # | Documento | Status | Descrição |
|---|-----------|--------|-----------|
| 1 | `Vision.md` | ⬜ Planned | Propósito, escopo, personas, princípios e não-objetivos do módulo. |
| 2 | `Architecture.md` | ⬜ Planned | C4 (contexto→componente) em ASCII, componentes internos e padrões. |
| 3 | `Requirements.md` | ⬜ Planned | Índice de requisitos e matriz de rastreabilidade. |
| 4 | `FunctionalRequirements.md` | ⬜ Planned | Tabela FR-NNN com critérios de aceite (MoSCoW). |
| 5 | `NonFunctionalRequirements.md` | ⬜ Planned | Tabela NFR-NNN com SLO/SLI e método de verificação. |
| 6 | `UseCases.md` | ⬜ Planned | Casos de uso UC-NNN (ator/fluxos/exceções). |
| 7 | `SequenceDiagrams.md` | ⬜ Planned | Sequências ASCII dos fluxos críticos (feliz + falha). |
| 8 | `ClassDiagrams.md` | ⬜ Planned | Estruturas/interfaces/contratos e invariantes (ASCII). |
| 9 | `StateMachine.md` | ⬜ Planned | Estados, transições, guardas e ações. |
| 10 | `Database.md` | ⬜ Planned | Modelo físico, DDL, índices, particionamento, retenção. |
| 11 | `API.md` | ⬜ Planned | Contratos REST (OpenAPI) e gRPC (proto), erros, idempotência. |
| 12 | `Events.md` | ⬜ Planned | Catálogo de eventos NATS (subject, schema, semântica). |
| 13 | `Configuration.md` | ⬜ Planned | Chaves de config, defaults, faixas e escopo. |
| 14 | `Deployment.md` | ⬜ Planned | Topologia, recursos, réplicas, health/readiness, rollout. |
| 15 | `Security.md` | ⬜ Planned | AuthN/Z, threat model STRIDE, secrets, mTLS, LGPD/GDPR. |
| 16 | `Monitoring.md` | ⬜ Planned | Dashboards, alertas Prometheus, SLO/SLA e runbooks. |
| 17 | `Logging.md` | ⬜ Planned | Log estruturado (Serilog→Seq), campos e correlação. |
| 18 | `Metrics.md` | ⬜ Planned | Catálogo de métricas OTel/Prometheus (nome/tipo/labels). |
| 19 | `Testing.md` | ⬜ Planned | Estratégia de testes (unit/integration/contract/e2e/chaos). |
| 20 | `Benchmark.md` | ⬜ Planned | Metodologia, cargas, KPIs e resultados-alvo. |
| 21 | `FailureRecovery.md` | ⬜ Planned | FMEA, detecção, recuperação, idempotência, RTO/RPO. |
| 22 | `Scalability.md` | ⬜ Planned | Modelo de escala, sharding, concorrência e limites. |
| 23 | `Examples.md` | ⬜ Planned | Exemplos executáveis (CLI/SDK/API) comentados. |
| 24 | `FAQ.md` | ⬜ Planned | Perguntas frequentes e armadilhas comuns. |
| 25 | `ADR.md` | ⬜ Planned | Índice de ADRs que afetam o módulo (link p/ 002-ADR). |
| 26 | `RFC.md` | ⬜ Planned | Índice de RFCs relacionadas (link p/ 003-RFC). |

## Definition of Done

Ver checklist de qualidade em `../_templates/MODULE_TEMPLATE.md`. Um documento
só passa de `⬜ Planned` para `✅ Stable` quando cumpre o checklist e reutiliza o
glossário e os contratos da RFC-0001 sem redefini-los.

## Referências canônicas

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `../002-ADR/README.md`
