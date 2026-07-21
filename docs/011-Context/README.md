---
Documento: Índice do Módulo
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 011-Context
Depende de: 010,017,024
---

# AIOS — Módulo 011 · Context

> **Propósito.** Gerência de janela de contexto: compressão, sumarização, recuperação seletiva e cache semântico.

> **Status:** Draft. O fan-out do módulo está concluído: os 26 documentos
> obrigatórios foram produzidos conforme `../_templates/MODULE_TEMPLATE.md` e em
> conformidade com `../003-RFC/RFC-0001-Architecture-Baseline.md`, derivados do
> `_DESIGN_BRIEF.md`. Refinamentos rumo a `Stable` seguem o Definition of Done abaixo.

## Dependências

Este módulo depende de: **010,017,024**. Reutilize as definições do
`../040-Glossary/Glossary.md` e os contratos centrais da RFC-0001 (URN, envelope
de evento, envelope de erro, idempotência, correlação).

## Documentos obrigatórios (26)

| # | Documento | Status | Descrição |
|---|-----------|--------|-----------|
| 1 | `Vision.md` | 🟨 Draft | Propósito, escopo, personas, princípios e não-objetivos do módulo. |
| 2 | `Architecture.md` | 🟨 Draft | C4 (contexto→componente) em ASCII, componentes internos e padrões. |
| 3 | `Requirements.md` | 🟨 Draft | Índice de requisitos e matriz de rastreabilidade. |
| 4 | `FunctionalRequirements.md` | 🟨 Draft | Tabela FR-NNN com critérios de aceite (MoSCoW). |
| 5 | `NonFunctionalRequirements.md` | 🟨 Draft | Tabela NFR-NNN com SLO/SLI e método de verificação. |
| 6 | `UseCases.md` | 🟨 Draft | Casos de uso UC-NNN (ator/fluxos/exceções). |
| 7 | `SequenceDiagrams.md` | 🟨 Draft | Sequências ASCII dos fluxos críticos (feliz + falha). |
| 8 | `ClassDiagrams.md` | 🟨 Draft | Estruturas/interfaces/contratos e invariantes (ASCII). |
| 9 | `StateMachine.md` | 🟨 Draft | Estados, transições, guardas e ações. |
| 10 | `Database.md` | 🟨 Draft | Modelo físico, DDL, índices, particionamento, retenção. |
| 11 | `API.md` | 🟨 Draft | Contratos REST (OpenAPI) e gRPC (proto), erros, idempotência. |
| 12 | `Events.md` | 🟨 Draft | Catálogo de eventos NATS (subject, schema, semântica). |
| 13 | `Configuration.md` | 🟨 Draft | Chaves de config, defaults, faixas e escopo. |
| 14 | `Deployment.md` | 🟨 Draft | Topologia, recursos, réplicas, health/readiness, rollout. |
| 15 | `Security.md` | 🟨 Draft | AuthN/Z, threat model STRIDE, secrets, mTLS, LGPD/GDPR. |
| 16 | `Monitoring.md` | 🟨 Draft | Dashboards, alertas Prometheus, SLO/SLA e runbooks. |
| 17 | `Logging.md` | 🟨 Draft | Log estruturado (Serilog→Seq), campos e correlação. |
| 18 | `Metrics.md` | 🟨 Draft | Catálogo de métricas OTel/Prometheus (nome/tipo/labels). |
| 19 | `Testing.md` | 🟨 Draft | Estratégia de testes (unit/integration/contract/e2e/chaos). |
| 20 | `Benchmark.md` | 🟨 Draft | Metodologia, cargas, KPIs e resultados-alvo. |
| 21 | `FailureRecovery.md` | 🟨 Draft | FMEA, detecção, recuperação, idempotência, RTO/RPO. |
| 22 | `Scalability.md` | 🟨 Draft | Modelo de escala, sharding, concorrência e limites. |
| 23 | `Examples.md` | 🟨 Draft | Exemplos executáveis (CLI/SDK/API) comentados. |
| 24 | `FAQ.md` | 🟨 Draft | Perguntas frequentes e armadilhas comuns. |
| 25 | `ADR.md` | 🟨 Draft | Índice de ADRs que afetam o módulo (link p/ 002-ADR). |
| 26 | `RFC.md` | 🟨 Draft | Índice de RFCs relacionadas (link p/ 003-RFC). |

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
