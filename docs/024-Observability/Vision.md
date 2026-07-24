---
Documento: Vision
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0001, ADR-0010, ADR-0241, ADR-0243, ADR-0245, ADR-0246
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: 000-Vision, 001-Architecture, 022-Policy, 025-Audit, 029-Operations
---

# 024-Observability — Vision

## 1. Propósito

O 024-Observability é o **sistema nervoso sensorial** do AIOS: o subsistema que torna
visível e mensurável o comportamento de todos os demais módulos, e que converte essa
visibilidade em **sinal acionável** — SLI, error budget, alerta e runbook.

Ele materializa a `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md` e
o requisito transversal da `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.8: *todo
serviço DEVE emitir traces/metrics/logs OTel com os campos de correlação*. O 024 é
quem recebe, admite, governa, armazena e serve esse fluxo.

Na analogia de sistema operacional (`../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md`),
este módulo é o subsistema de **instrumentação e contadores de desempenho** —
elevado à escala de uma plataforma multi-tenant com 10⁶ agentes autônomos, onde o
problema deixa de ser "coletar" e passa a ser "coletar sem quebrar sob a própria
coleta".

## 2. Problema que resolve

| Patologia sem o módulo | Sintoma observável | Custo |
|------------------------|--------------------|-------|
| **Telemetria emergente** | Cada serviço inventa nomes, unidades e labels. | Dashboards que não compõem; nenhuma consulta cruza dois módulos. |
| **Explosão de cardinalidade** | Alguém usa `agent_urn` como label; 10⁶ séries nascem de uma vez. | O observador passa a custar mais que o observado e derruba o armazenamento. |
| **Observabilidade que derruba o observado** | Coletor lento aplica backpressure; o serviço trava esperando exportar. | Uma falha de telemetria vira uma indisponibilidade de produto. |
| **Alerta sem dono nem procedimento** | `403 no dashboard vermelho` às 3h da manhã, sem runbook. | O alerta é silenciado em vez de tratado; o próximo incidente passa despercebido. |
| **SLO como slide** | O objetivo existe em apresentação, não em dado. | Nenhuma decisão de release ou de política é tomada com base em error budget. |

## 3. Escopo

### 3.1 Dentro do escopo (in)

- **Pipeline OTLP** de ingestão de métricas, traces e logs, sob mTLS e correlação W3C.
- **Registro de sinais** (`SignalDescriptor`): toda métrica, span e evento de log é
  declarado, versionado e admitido antes de ser armazenado.
- **Governança de cardinalidade**: orçamento por tenant e por serviço, proibição de
  labels de identidade, quarentena automática do sinal infrator.
- **Redação de PII na borda**, antes de qualquer persistência.
- **Amostragem** *tail-based* de traces, retendo 100% dos erros e das caudas.
- **Armazenamento e retenção em camadas** (Prometheus, Tempo/Jaeger, Seq) com
  *downsampling*.
- **SLI/SLO e ledger de error budget**; regras de alerta *multi-window multi-burn-rate*.
- **Ciclo de vida do alerta**: disparo, notificação, reconhecimento, silêncio, resolução.
- **Dashboards e runbooks como código** versionado.
- **Consulta unificada** com isolamento por tenant, atuando como PEP perante o
  `../022-Policy/`.

### 3.2 Fora do escopo (out)

| Não faz | Dono real |
|---------|-----------|
| Ser **prova de auditoria** (imutável, completa, legal) | `../025-Audit/` |
| Ser **dono** de qualquer métrica de negócio | módulo emissor |
| Decidir autorização de consulta | `../022-Policy/` |
| Responder ao incidente (executar runbook, acionar plantão) | `../029-Operations/` |
| Decidir rollback de deploy ou de política | `../028-Deployment/`, `../022-Policy/` |
| Definir orçamento financeiro ou custo de inferência | `../026-Cost-Optimizer/` |
| Emitir identidade dos coletores | `../021-Security/` |
| Detectar fraude ou anomalia de segurança | `../021-Security/`, `../025-Audit/` |
| Aprender com a telemetria para ajustar agentes | `../023-Learning/` |
| Guardar conteúdo de payload de negócio | módulo dono do dado |

A lista completa e normativa está em `./_DESIGN_BRIEF.md` §1.3.

## 4. Personas atendidas

| Persona | Necessidade | Como o módulo atende |
|---------|-------------|----------------------|
| **Serviço emissor** (todos os módulos) | Instrumentar sem pensar em infraestrutura, e sem risco de a telemetria travá-lo. | SDK OTel + coletor local; fail-open com descarte contabilizado (NFR-002, NFR-006). |
| **SRE de plantão** (`../029-Operations/`) | Ser acordado só pelo que importa, com procedimento em mãos. | Alerta *multi-burn-rate*, runbook obrigatório, silêncio com prazo, deduplicação por `fingerprint`. |
| **Dono de serviço** | Saber se seu serviço está dentro do objetivo e quanto budget resta. | SLO versionado, ledger de error budget, dashboards como código. |
| **Engenheiro depurando** | Ir do log ao trace ao painel sem perder o contexto. | Correlação `trace_id`/`span_id`/`tenant_id` obrigatória (RFC-0001 §5.6); consulta unificada. |
| **Plataforma / arquitetura** | Impedir que a telemetria cresça sem limite. | Registro de sinais, orçamento de cardinalidade, retenção em camadas. |
| **DPO / auditoria** | Garantir que a telemetria não vire repositório de PII. | Redação na borda, `data_class` por sinal, direito ao esquecimento sobre telemetria. |
| **`../028-Deployment/` e `../022-Policy/`** | Um sinal objetivo para decidir avanço ou rollback. | `telemetry.budget.exhausted` e `telemetry.alert.firing`. |

## 5. Princípios de design

1. **Cardinalidade é o recurso escasso, não volume.** O limite é séries ativas, e ele
   é imposto no registro — antes de a série existir (ADR-0242).
2. **A observabilidade nunca derruba o observado.** Sob saturação, descarta-se
   telemetria e contabiliza-se o descarte; jamais se aplica backpressure ao emissor
   (ADR-0243).
3. **Telemetria é contrato declarado, não emergente.** Sinal não registrado é
   quarentenado (ADR-0241).
4. **PII é redigida na borda.** Redigir no armazenamento significa que o dado já
   esteve em claro em disco (ADR-0249).
5. **Erro e cauda nunca são amostrados fora.** *Tail-based sampling* retém 100% do que
   explica uma falha (ADR-0244).
6. **SLO é dado versionado; error budget é sinal de primeira classe.** É ele, não a
   intuição, que decide se um release avança (ADR-0245).
7. **Alerta sem runbook é inválido.** Publicação sem `runbook_urn` é rejeitada
   (ADR-0247).
8. **Ausência de dado não é saúde.** Alerta sem dado vai a `Expired`, nunca a
   `Resolved` (invariante I6).
9. **Quem observa também é observado.** O `SelfTelemetry` reporta por um caminho que
   não depende do pipeline que ele observa.

## 6. Não-objetivos explícitos

- **Não** é auditoria: é amostrável, tem retenção limitada e pode perder dados sob
  pressão — as três coisas que a trilha do `../025-Audit/` não pode ter (ADR-0246).
- **Não** é um banco de dados de propósito geral: séries, traces e logs **não** vivem
  no PostgreSQL, que guarda apenas o plano de controle da observabilidade.
- **Não** oferece "modo confiável" com backpressure em produção:
  `obs.ingest.fail_mode = closed` existe só para diagnóstico em laboratório.
- **Não** substitui o journal de decisões do `../022-Policy/` nem o registro de custo
  do `../026-Cost-Optimizer/` por sua própria série temporal.
- **Não** guarda conteúdo de negócio: prompt, resposta de LLM e item de memória não
  entram em log, span ou métrica.
- **Não** decide nada operacional: dispara e roteia; a resposta é de
  `../029-Operations/`.

## 7. Relação com a visão global do AIOS

`../000-Vision/Vision.md` lista **observabilidade** entre os princípios do sistema —
"telemetria e proveniência em todo caminho". O 024 é onde esse princípio deixa de ser
declaração e vira propriedade verificável: cada módulo emite, este módulo admite e
governa, e o resultado é um sistema cujo comportamento pode ser medido, comparado com
um objetivo e questionado quando desvia.

```
   000-Vision ──── "telemetria e proveniência em todo caminho"
        │
        ├── ADR-0010 ──▶ observabilidade e auditoria POR CONSTRUÇÃO
        │                       │
        │              RFC-0001 §5.8 ──▶ todo serviço emite OTel correlacionado
        │                       │
        └──────────────▶  024-OBSERVABILITY (este módulo)
                                │
              todos os módulos ─┼─ 025-Audit (prova imutável, disjunta)
                                │
                        029-Operations (quem responde ao sinal)
```

## 8. Critérios de sucesso do módulo

| Critério | Meta | Verificação |
|----------|------|-------------|
| Overhead no serviço observado | ≤ 3% CPU e ≤ 1 ms no p99 | NFR-002 |
| Telemetria nunca bloqueia o emissor | 0 casos de backpressure | NFR-002, FR-002 |
| Cardinalidade sob controle | ≤ 10⁶ séries/tenant; 0 labels de identidade | NFR-009 |
| Detecção rápida | Alerta de queima rápida em ≤ 2 min | NFR-008 |
| Todo alerta acionável | 100% com runbook válido e revisado | NFR-016 |
| Nenhuma PII em sinal operacional | 0 ocorrências | NFR-014 |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- Arquitetura do módulo: `./Architecture.md` · Requisitos: `./Requirements.md`

*Fim de `Vision.md`.*
