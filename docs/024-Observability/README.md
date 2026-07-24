---
Documento: Índice do Módulo
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0240..0249 (a propor); ADR-0008, ADR-0010 (globais)
RFCs relacionados: RFC-0001 (§5.6, §5.8, §7); RFC-0240, RFC-0241 (a propor)
Depende de: 001,003,005,020,021,022,025,027,028,029
---

# AIOS — Módulo 024 · Observability

> **Propósito.** Telemetria OTel: métricas (Prometheus), traces, logs (Serilog/Seq), dashboards e SLOs.

> **Status:** Draft. O fan-out do módulo está concluído: os 26 documentos
> obrigatórios foram produzidos conforme `../_templates/MODULE_TEMPLATE.md` e em
> conformidade com `../003-RFC/RFC-0001-Architecture-Baseline.md`, derivados do
> `_DESIGN_BRIEF.md`. Refinamentos rumo a `Stable` seguem o Definition of Done abaixo.

## Fronteira em uma frase

O **024** responde **o que está acontecendo agora e como o sistema se comporta**; o
`../025-Audit/` responde **o que aconteceu e pode ser provado**. A telemetria é
amostrável, tem retenção limitada e pode perder dados sob pressão; a trilha de
auditoria não pode nenhuma das três coisas.

## Dependências

Este módulo depende de: **021-Security** (mTLS dos emissores, chave de tokenização de
PII), **022-Policy** (PDP das consultas), **025-Audit** (fronteira telemetria × prova),
**005-Database** (schema `telemetry`), **020-Communication** (NATS/JetStream),
**027-Cluster** e **028-Deployment** (atributos de recurso e topologia) e
**029-Operations** (plantão e runbooks).
É consumido por **todos** os módulos, que emitem métricas, traces e logs por construção
(ADR-0010).
Reutilize as definições do `../040-Glossary/Glossary.md` e os contratos centrais da
RFC-0001 (URN, envelope de evento, envelope de erro, idempotência, correlação).

## Documentos obrigatórios (26)

| # | Documento | Status | Descrição |
|---|-----------|--------|-----------|
| 1 | [`Vision.md`](Vision.md) | 🟨 Draft | Propósito, escopo, personas, princípios e não-objetivos do módulo. |
| 2 | [`Architecture.md`](Architecture.md) | 🟨 Draft | C4 (contexto→componente) em ASCII, componentes internos e padrões. |
| 3 | [`Requirements.md`](Requirements.md) | 🟨 Draft | Índice de requisitos e matriz de rastreabilidade. |
| 4 | [`FunctionalRequirements.md`](FunctionalRequirements.md) | 🟨 Draft | FR-001..FR-024 com critérios de aceite (MoSCoW). |
| 5 | [`NonFunctionalRequirements.md`](NonFunctionalRequirements.md) | 🟨 Draft | NFR-001..NFR-017 com SLO/SLI e método de verificação. |
| 6 | [`UseCases.md`](UseCases.md) | 🟨 Draft | UC-001..UC-015 (ator/fluxos/exceções). |
| 7 | [`SequenceDiagrams.md`](SequenceDiagrams.md) | 🟨 Draft | Sequências ASCII dos fluxos críticos (feliz + falha). |
| 8 | [`ClassDiagrams.md`](ClassDiagrams.md) | 🟨 Draft | Estruturas/interfaces/contratos e invariantes (ASCII). |
| 9 | [`StateMachine.md`](StateMachine.md) | 🟨 Draft | FSM do `AlertInstance`, ciclo do `SignalDescriptor` e invariantes. |
| 10 | [`Database.md`](Database.md) | 🟨 Draft | Schema `telemetry` (plano de controle): DDL, RLS, partições, retenção. |
| 11 | [`API.md`](API.md) | 🟨 Draft | OTLP (ingestão), REST/gRPC (consulta e administração), erros `AIOS-OBS-*`. |
| 12 | [`Events.md`](Events.md) | 🟨 Draft | Catálogo de eventos NATS do domínio `telemetry`. |
| 13 | [`Configuration.md`](Configuration.md) | 🟨 Draft | Chaves `AIOS_OBS_*`, defaults, faixas e escopo. |
| 14 | [`Deployment.md`](Deployment.md) | 🟨 Draft | Topologia agente+gateway, recursos, health/readiness, rollout. |
| 15 | [`Security.md`](Security.md) | 🟨 Draft | AuthN/Z, assimetria de `fail_mode`, STRIDE, LGPD/GDPR. |
| 16 | [`Monitoring.md`](Monitoring.md) | 🟨 Draft | Dashboards, alertas Prometheus, SLO e runbooks RB-O-01..13. |
| 17 | [`Logging.md`](Logging.md) | 🟨 Draft | Log estruturado próprio e operação do `LogPipeline`. |
| 18 | [`Metrics.md`](Metrics.md) | 🟨 Draft | Catálogo de métricas `aios_obs_*`. |
| 19 | [`Testing.md`](Testing.md) | 🟨 Draft | Não-interferência, perda contabilizada, probes sintéticos. |
| 20 | [`Benchmark.md`](Benchmark.md) | 🟨 Draft | Metodologia, cargas B-01..B-12, KPIs e alvos. |
| 21 | [`FailureRecovery.md`](FailureRecovery.md) | 🟨 Draft | FMEA F-01..F-17, degradação graciosa, RTO/RPO. |
| 22 | [`Scalability.md`](Scalability.md) | 🟨 Draft | Cardinalidade como recurso escasso; caminho para milhões de agentes. |
| 23 | [`Examples.md`](Examples.md) | 🟨 Draft | Exemplos executáveis (SDK/CLI/API) comentados. |
| 24 | [`FAQ.md`](FAQ.md) | 🟨 Draft | Perguntas frequentes e armadilhas comuns. |
| 25 | [`ADR.md`](ADR.md) | 🟨 Draft | Índice de ADRs (ADR-0240..0249 a propor). |
| 26 | [`RFC.md`](RFC.md) | 🟨 Draft | Índice de RFCs (RFC-0240, RFC-0241 a propor). |

Fonte única de verdade interna: [`_DESIGN_BRIEF.md`](_DESIGN_BRIEF.md).

## Definition of Done

Ver checklist de qualidade em `../_templates/MODULE_TEMPLATE.md`. Um documento
só passa de `🟨 Draft` para `✅ Stable` quando cumpre o checklist e reutiliza o
glossário e os contratos da RFC-0001 sem redefini-los.

## Referências canônicas

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md` (§4 stack de observabilidade)
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§5.6, §5.8, §7)
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
