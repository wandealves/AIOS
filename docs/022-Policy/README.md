---
Documento: Índice do Módulo
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0220..0229 (a propor); ADR-0008, ADR-0010 (globais)
RFCs relacionados: RFC-0001 (§5.8); RFC-0220, RFC-0221 (a propor)
Depende de: 001,003,005,020,021,024,025,026
---

# AIOS — Módulo 022 · Policy

> **Propósito.** Policy Engine (PDP): RBAC+ABAC, default deny, avaliação, simulação e testes de política.

> **Status:** Draft. O fan-out do módulo está concluído: os 26 documentos
> obrigatórios foram produzidos conforme `../_templates/MODULE_TEMPLATE.md` e em
> conformidade com `../003-RFC/RFC-0001-Architecture-Baseline.md`, derivados do
> `_DESIGN_BRIEF.md`. Refinamentos rumo a `Stable` seguem o Definition of Done abaixo.

## Fronteira em uma frase

O `../021-Security/` responde **quem é** o solicitante; o **022** responde **se ele
pode**; o PEP do módulo chamador **impede ou deixa passar**; o `../025-Audit/` registra
**que aconteceu**.

## Dependências

Este módulo depende de: **021-Security** (identidade e atributos assertáveis,
assinatura do bundle), **025-Audit** (trilha imutável), **026-Cost-Optimizer**
(atributos de orçamento), **005-Database** (schema `policy`), **020-Communication**
(NATS/JetStream e request-reply) e **024-Observability**.
É consumido como **PDP** por **todos** os módulos que possuem PEP: `004-API`,
`006-Kernel`, `007-Agent-Runtime`, `008-Agent-Lifecycle`, `009-Scheduler`,
`010-Memory`, `011-Context`, `015-Tool-Manager`, `017-Model-Router`,
`020-Communication` e `021-Security`.
Reutilize as definições do `../040-Glossary/Glossary.md` e os contratos centrais da
RFC-0001 (URN, envelope de evento, envelope de erro, idempotência, correlação).

## Documentos obrigatórios (26)

| # | Documento | Status | Descrição |
|---|-----------|--------|-----------|
| 1 | [`Vision.md`](Vision.md) | 🟨 Draft | Propósito, escopo, personas, princípios e não-objetivos do módulo. |
| 2 | [`Architecture.md`](Architecture.md) | 🟨 Draft | C4 (contexto→componente) em ASCII, componentes internos e padrões. |
| 3 | [`Requirements.md`](Requirements.md) | 🟨 Draft | Índice de requisitos e matriz de rastreabilidade. |
| 4 | [`FunctionalRequirements.md`](FunctionalRequirements.md) | 🟨 Draft | FR-001..FR-023 com critérios de aceite (MoSCoW). |
| 5 | [`NonFunctionalRequirements.md`](NonFunctionalRequirements.md) | 🟨 Draft | NFR-001..NFR-016 com SLO/SLI e método de verificação. |
| 6 | [`UseCases.md`](UseCases.md) | 🟨 Draft | UC-001..UC-015 (ator/fluxos/exceções). |
| 7 | [`SequenceDiagrams.md`](SequenceDiagrams.md) | 🟨 Draft | Sequências ASCII dos fluxos críticos (feliz + falha). |
| 8 | [`ClassDiagrams.md`](ClassDiagrams.md) | 🟨 Draft | Estruturas/interfaces/contratos e invariantes (ASCII). |
| 9 | [`StateMachine.md`](StateMachine.md) | 🟨 Draft | FSM do `PolicyBundle`, precedência de decisão e invariantes. |
| 10 | [`Database.md`](Database.md) | 🟨 Draft | Schema `policy`: DDL, RLS, particionamento, retenção. |
| 11 | [`API.md`](API.md) | 🟨 Draft | REST (OpenAPI), gRPC (proto), NATS request-reply, erros `AIOS-POL-*`. |
| 12 | [`Events.md`](Events.md) | 🟨 Draft | Catálogo de eventos NATS do domínio `policy`. |
| 13 | [`Configuration.md`](Configuration.md) | 🟨 Draft | Chaves `AIOS_POL_*`, defaults, faixas e escopo. |
| 14 | [`Deployment.md`](Deployment.md) | 🟨 Draft | Topologia, recursos, réplicas, health/readiness, rollout. |
| 15 | [`Security.md`](Security.md) | 🟨 Draft | AuthN/Z, autogoverno, threat model STRIDE, LGPD/GDPR. |
| 16 | [`Monitoring.md`](Monitoring.md) | 🟨 Draft | Dashboards, alertas Prometheus, SLO e runbooks RB-P-01..12. |
| 17 | [`Logging.md`](Logging.md) | 🟨 Draft | Log estruturado, journal de decisões e trilha. |
| 18 | [`Metrics.md`](Metrics.md) | 🟨 Draft | Catálogo de métricas `aios_pol_*`. |
| 19 | [`Testing.md`](Testing.md) | 🟨 Draft | Testes do módulo × testes de política; gates de qualidade. |
| 20 | [`Benchmark.md`](Benchmark.md) | 🟨 Draft | Metodologia, cargas B-01..B-10, KPIs e alvos. |
| 21 | [`FailureRecovery.md`](FailureRecovery.md) | 🟨 Draft | FMEA F-01..F-15, degradação graciosa, RTO/RPO. |
| 22 | [`Scalability.md`](Scalability.md) | 🟨 Draft | Modelo de escala, limites e caminho para milhões de agentes. |
| 23 | [`Examples.md`](Examples.md) | 🟨 Draft | Exemplos executáveis (CLI/SDK/API) comentados. |
| 24 | [`FAQ.md`](FAQ.md) | 🟨 Draft | Perguntas frequentes e armadilhas comuns. |
| 25 | [`ADR.md`](ADR.md) | 🟨 Draft | Índice de ADRs (ADR-0220..0229 a propor). |
| 26 | [`RFC.md`](RFC.md) | 🟨 Draft | Índice de RFCs (RFC-0220, RFC-0221 a propor). |

Fonte única de verdade interna: [`_DESIGN_BRIEF.md`](_DESIGN_BRIEF.md).

## Definition of Done

Ver checklist de qualidade em `../_templates/MODULE_TEMPLATE.md`. Um documento
só passa de `🟨 Draft` para `✅ Stable` quando cumpre o checklist e reutiliza o
glossário e os contratos da RFC-0001 sem redefini-los.

## Referências canônicas

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§5.8)
- Decisão-mãe: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`
