---
Documento: Índice do Módulo
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0250..0259 (a propor); ADR-0008, ADR-0010 (globais)
RFCs relacionados: RFC-0001 (§5.8, §7); RFC-0250, RFC-0251 (a propor)
Depende de: 001,003,005,020,021,022,024,027,028,029
---

# AIOS — Módulo 025 · Audit

> **Propósito.** Auditoria imutável event-sourced: proveniência de decisões, compliance e retenção.

> **Status:** Draft. O fan-out do módulo está concluído: os 26 documentos
> obrigatórios foram produzidos conforme `../_templates/MODULE_TEMPLATE.md` e em
> conformidade com `../003-RFC/RFC-0001-Architecture-Baseline.md`, derivados do
> `_DESIGN_BRIEF.md`. Refinamentos rumo a `Stable` seguem o Definition of Done abaixo.

## Fronteira em uma frase

O `../024-Observability/` responde **o que está acontecendo agora e como o sistema se
comporta**; o **025** responde **o que aconteceu e pode ser provado**. Telemetria é
amostrável, tem retenção curta e pode perder dados sob pressão; a trilha de auditoria
não pode nenhuma das três coisas.

## As três teses do módulo

1. **Imutabilidade é verificável, não declarada** — cadeia de hash + selo de Merkle
   assinado + cópia WORM: adulterar exige comprometer três sistemas distintos.
2. **A lacuna é o defeito mais perigoso** — um registro adulterado quebra o hash e é
   detectado; um registro que nunca chegou não deixa rastro. Completude é medida.
3. **Apagar conteúdo, nunca a prova de existência** — o apagamento criptográfico
   destrói a chave do titular; `seq`, hashes e `payload_digest` permanecem.

## Dependências

Este módulo depende de: **021-Security** (assinatura de selo, chaves por titular,
mTLS), **022-Policy** (PDP de consulta e governança), **005-Database** (schema `audit`,
réplica síncrona), **020-Communication** (JetStream), **024-Observability** (fronteira
telemetria × prova), **027-Cluster**, **028-Deployment** e **029-Operations**.
É consumido por **todos** os módulos, que emitem registro de auditoria por construção
(RFC-0001 §5.8, ADR-0010); o `../005-Database/` consome `audit.legalhold.applied` para
suspender expurgo.
Reutilize as definições do `../040-Glossary/Glossary.md` (*Event Sourcing*) e os
contratos centrais da RFC-0001 (URN, envelope de evento, envelope de erro,
idempotência, correlação).

## Documentos obrigatórios (26)

| # | Documento | Status | Descrição |
|---|-----------|--------|-----------|
| 1 | [`Vision.md`](Vision.md) | 🟨 Draft | Propósito, escopo, personas, princípios e não-objetivos do módulo. |
| 2 | [`Architecture.md`](Architecture.md) | 🟨 Draft | C4 (contexto→componente) em ASCII, componentes internos e padrões. |
| 3 | [`Requirements.md`](Requirements.md) | 🟨 Draft | Índice de requisitos e matriz de rastreabilidade. |
| 4 | [`FunctionalRequirements.md`](FunctionalRequirements.md) | 🟨 Draft | FR-001..FR-023 com critérios de aceite (MoSCoW). |
| 5 | [`NonFunctionalRequirements.md`](NonFunctionalRequirements.md) | 🟨 Draft | NFR-001..NFR-017 com SLO/SLI e método de verificação. |
| 6 | [`UseCases.md`](UseCases.md) | 🟨 Draft | UC-001..UC-015 (ator/fluxos/exceções). |
| 7 | [`SequenceDiagrams.md`](SequenceDiagrams.md) | 🟨 Draft | Sequências ASCII dos fluxos críticos (feliz + falha). |
| 8 | [`ClassDiagrams.md`](ClassDiagrams.md) | 🟨 Draft | Estruturas/interfaces/contratos e invariantes (ASCII). |
| 9 | [`StateMachine.md`](StateMachine.md) | 🟨 Draft | FSM do `AuditRecord`, ciclo do `LegalHold` e invariantes. |
| 10 | [`Database.md`](Database.md) | 🟨 Draft | Schema `audit`: DDL, *trigger* de imutabilidade, RLS, partições. |
| 11 | [`API.md`](API.md) | 🟨 Draft | REST/gRPC de ingestão, consulta, prova e governança; erros `AIOS-AUD-*`. |
| 12 | [`Events.md`](Events.md) | 🟨 Draft | Catálogo de eventos NATS do domínio `audit`. |
| 13 | [`Configuration.md`](Configuration.md) | 🟨 Draft | Chaves `AIOS_AUD_*`, defaults, faixas e escopo. |
| 14 | [`Deployment.md`](Deployment.md) | 🟨 Draft | Topologia, réplica síncrona, WORM, health/readiness, rollout. |
| 15 | [`Security.md`](Security.md) | 🟨 Draft | Defesa em profundidade da imutabilidade, STRIDE, LGPD/GDPR. |
| 16 | [`Monitoring.md`](Monitoring.md) | 🟨 Draft | Dashboards, alertas Prometheus, SLO e runbooks RB-A-01..13. |
| 17 | [`Logging.md`](Logging.md) | 🟨 Draft | Log de aplicação × trilha; correlação; o que nunca logar. |
| 18 | [`Metrics.md`](Metrics.md) | 🟨 Draft | Catálogo de métricas `aios_aud_*`. |
| 19 | [`Testing.md`](Testing.md) | 🟨 Draft | Testes de crash, adversariais (A-01..A-10) e verificação offline. |
| 20 | [`Benchmark.md`](Benchmark.md) | 🟨 Draft | Metodologia, cargas B-01..B-15, KPIs e o custo do RPO 0. |
| 21 | [`FailureRecovery.md`](FailureRecovery.md) | 🟨 Draft | FMEA F-01..F-17, degradação graciosa, RTO/RPO 0. |
| 22 | [`Scalability.md`](Scalability.md) | 🟨 Draft | Cadeia particionada; caminho para bilhões de registros. |
| 23 | [`Examples.md`](Examples.md) | 🟨 Draft | Exemplos executáveis (CLI/SDK/API) comentados. |
| 24 | [`FAQ.md`](FAQ.md) | 🟨 Draft | Perguntas frequentes e armadilhas comuns. |
| 25 | [`ADR.md`](ADR.md) | 🟨 Draft | Índice de ADRs (ADR-0250..0259 a propor). |
| 26 | [`RFC.md`](RFC.md) | 🟨 Draft | Índice de RFCs (RFC-0250, RFC-0251 a propor). |

Fonte única de verdade interna: [`_DESIGN_BRIEF.md`](_DESIGN_BRIEF.md).

## Definition of Done

Ver checklist de qualidade em `../_templates/MODULE_TEMPLATE.md`. Um documento
só passa de `🟨 Draft` para `✅ Stable` quando cumpre o checklist e reutiliza o
glossário e os contratos da RFC-0001 sem redefini-los.

## Referências canônicas

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md` (*Event Sourcing*)
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§5.8, §7)
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
