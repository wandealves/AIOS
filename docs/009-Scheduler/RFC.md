---
Documento: RFC.md
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0009, ADR-0090, ADR-0091, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001
Depende de: 001-Architecture, 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster, 003-RFC
---

# 009-Scheduler — Índice de RFCs

> **Escopo deste documento.** Este documento **NÃO** especifica protocolo
> algum por si — é um **índice navegável** das *Requests for Comments* (RFC)
> que especificam contratos consumidos ou propostos por este módulo. O
> conteúdo normativo pleno reside em `../003-RFC/`, no formato IETF definido
> por `../003-RFC/TEMPLATE.md` (Abstract · Status/Terminologia · Motivação ·
> Terminologia · Especificação · Segurança · Privacidade · Registros IANA-like
> · Compatibilidade · Referências · Apêndices). Este arquivo **DEVE** ser
> mantido sincronizado com `./_DESIGN_BRIEF.md` §11 e com `../003-RFC/README.md`.

> **Regra de não-redefinição.** Contratos centrais já especificados pela
> RFC-0001 (URN `urn:aios:<tenant>:<tipo>:<id>`, envelope de evento
> CloudEvents, subjects `aios.<tenant>.<dominio>.<entidade>.<acao>`, envelope
> de erro RFC 7807 `AIOS-<DOMINIO>-<NNNN>`, `Idempotency-Key`, cabeçalhos
> `traceparent`/`X-AIOS-Tenant`) **NÃO SÃO** redefinidos por nenhuma RFC deste
> módulo — apenas **especializados** para o domínio de admission control,
> priorização/custo, placement, preempção e backpressure.

---

## 1. RFC Consumida (Baseline)

| RFC | Título | Status | Relação com o Scheduler |
|-----|--------|--------|----------------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline consumida, não redefinida.** O Scheduler herda integralmente: formato de URN (§5.1), envelope de evento CloudEvents (§5.2), convenção de subjects (§5.3), envelope de erro RFC 7807 (§5.4), semântica de idempotência (§5.5), cabeçalhos de correlação obrigatórios (§5.6) e regras de versionamento de API (§5.7). Toda RFC específica do Scheduler (§2 abaixo) DEVE citar a seção pertinente da RFC-0001 em vez de reespecificá-la. |

### 1.1 Mapeamento de uso da RFC-0001 pelo Scheduler

| Seção da RFC-0001 | Contrato | Uso concreto no Scheduler |
|--------------------|----------|------------------------------|
| §5.1 URN | `urn:aios:<tenant>:<tipo>:<id>` | Campos `task_urn`/`agent_urn` de `SchedulingRequest`; `placement_target`/`instance` referenciam a mesma URN em erros e eventos. |
| §5.2 Envelope de Evento (CloudEvents) | Estrutura `{id, source, type, time, datacontenttype, subject, tenant, data, traceparent}` | Payload publicado pelo `EventPublisher` para todos os 9 tipos de evento do §6 do `_DESIGN_BRIEF.md` (`admitted`, `rejected`, `scheduled`, `preempted`, `resumed`, `completed`, `failed`, `backpressure.changed`, `decision.recorded`). |
| §5.3 Convenção de Subjects | `aios.<tenant>.<dominio>.<entidade>.<acao>` | Domínio primário `task`, entidade `execution` (ex.: `aios.<tenant>.task.execution.scheduled`); domínio `scheduler` para sinais internos (`aios.<tenant>.scheduler.backpressure.changed`, `aios.<tenant>.scheduler.decision.recorded`). |
| §5.4 Envelope de Erro (RFC 7807) | `type/title/status/detail/instance/code/retriable` | Toda resposta de erro do `SchedulingGateway` (`API.md` §5), com `code` no domínio reservado `SCHED` (`AIOS-SCHED-0001`..`AIOS-SCHED-0012`). |
| §5.5 Idempotência | `Idempotency-Key`, retenção ≥ 24h | `IdempotencyStore` (Redis+PostgreSQL); retenção configurável via `scheduler.idempotency.ttl_hours` (default 24h, faixa 24–168h). Cobre `Submit`, `Cancel`, `Preempt`, `ReportRuntimeSignal`. |
| §5.6 Correlação | `traceparent`, `X-AIOS-Tenant` | Extraídos e validados pelo `SchedulingGateway` (PEP) em toda requisição; propagados a todo evento e span OTel emitido pelo `SchedulerTelemetry`; `X-AIOS-Tenant` divergente do claim autenticado é rejeitado com `AIOS-SCHED-0004`. |
| §5.7 Versionamento | Versionamento semântico de API | Pacote gRPC `aios.scheduler.v1`; base REST `/v1/scheduler`. Troca de `ISchedulingPolicy` (heurística↔RL, ver ADR-0096) NÃO incrementa versão de API — é transparente ao contrato externo. |

---

## 2. RFCs Propostas por Este Módulo

> **Estado de redação.** As RFCs abaixo estão **planejadas** e seu conteúdo
> pretendido está descrito no `./_DESIGN_BRIEF.md` (especialmente §3 modelo de
> dados, §4 máquina de estados, §5 superfície de API e §6 catálogo de eventos),
> que é a fonte única de verdade até que cada RFC seja redigida como arquivo
> próprio em `../003-RFC/` no formato IETF completo e promovida de `Draft` para
> `Accepted` conforme o ciclo de vida `Draft → Discussion → Last Call →
> Accepted`. Nenhum destes arquivos existe fisicamente em `../003-RFC/` no
> momento desta versão do índice.

| RFC | Título | Status | Escopo pretendido |
|-----|--------|--------|----------------------|
| RFC-0090 | Scheduling Decision Contract & Service-Class Model | Planned (Draft a redigir) | Especificação normativa completa do contrato de decisão de escalonamento: as **quatro classes de serviço** (`INTERACTIVE`, `BATCH`, `BACKGROUND`, `SYSTEM`) e seus pesos/prioridades default; o formato canônico de `SchedulingRequest`/`SchedulingDecision` (§3 do brief); a função de score `C = α·L + β·$ + γ·E`; a máquina de estados de tarefa sob a ótica do Scheduler (§4 do brief); e os eventos `aios.<tenant>.task.execution.*` (§6 do brief) como contrato público consumido por `006-Kernel`, `007-Agent-Runtime`, `025-Audit`. |
| RFC-0091 | Learned Scheduling Policy (RL) Interface & Feature Contract | Planned (Draft a redigir) | Especificação normativa da interface `ISchedulingPolicy` (ADR-0096) e do **contrato de features** extraído pelo `FeatureExtractor` (carga, profundidade de fila, custo estimado, folga de SLA, histórico de decisão) consumido tanto pela `HeuristicPolicy` (v1) quanto pela `LearnedPolicy` (v2, RL/`023-Learning`); regras de compatibilidade que garantem que a troca v1→v2 não altera API/eventos externos (FR-013). |

### 2.1 Justificativa de cada RFC (por que é uma RFC e não uma seção da Architecture.md)

| RFC | Justificativa |
|-----|----------------|
| RFC-0090 | O contrato de decisão de escalonamento (classes de serviço, score, estados, eventos) é consumido por **múltiplos módulos** fora da fronteira do Scheduler (`006-Kernel` submete `SchedulingRequest`; `007-Agent-Runtime` reage a `scheduled`/`preempted`; `025-Audit` consome `decision.recorded`) — uma mudança de forma unilateral quebraria consumidores externos, exigindo o processo de comentário público (`Draft → Discussion → Last Call`) típico de RFC, diferente de uma decisão pontual de ADR. |
| RFC-0091 | O contrato de features e a interface `ISchedulingPolicy` são a superfície de integração entre o Scheduler e `023-Learning` (treino da política RL) — formalizar esse contrato como RFC garante que a evolução de v1 (heurística) para v2 (aprendida) seja **aditiva e compatível**, sem exigir renegociação bilateral a cada iteração de modelo. |

### 2.2 Estrutura obrigatória que cada RFC deste módulo DEVE seguir

Conforme `../003-RFC/README.md`, ambas as RFCs propostas DEVEM conter, nesta
ordem:

1. **Abstract** — resumo de 1 parágrafo do contrato especificado.
2. **Status & terminologia** (RFC 2119/8174) — palavras normativas.
3. **Motivação** — por que o contrato existe (referenciar `_DESIGN_BRIEF.md` §0/§1).
4. **Terminologia** — reuso de `../040-Glossary/Glossary.md`, sem redefinir termos
   (`Admission Control`, `Backpressure`, `Placement`, `Preemption`, `Shard`).
5. **Especificação (normativa)** — o contrato em si (modelo de decisão/classes
   de serviço para RFC-0090; interface de política/features para RFC-0091).
6. **Considerações de segurança** — referenciar `Security.md` deste módulo
   (fronteira PEP/PDP, §12.1/§12.2 do brief).
7. **Considerações de privacidade (LGPD/GDPR)** — referenciar `_DESIGN_BRIEF.md`
   §12.3 (minimização via `payload_ref`, `feature_vector` sem PII bruta).
8. **Considerações de registro (IANA-like)** — reserva de domínio de erro
   `SCHED`, subjects `task.execution.*`/`scheduler.*`, versões de `dataschema`.
9. **Compatibilidade e versionamento** — regras de evolução sem quebra (ex.:
   troca de política v1→v2 é transparente à API/eventos).
10. **Referências** — normativas (RFC-0001) e informativas (ADRs do §2 de
    `ADR.md` deste módulo).
11. **Apêndices** — exemplos completos de request/response e payloads de
    evento (reaproveitar exemplos de `API.md`/`Events.md`).

---

## 3. Rastreabilidade RFC → Componente → Requisito

| RFC | Componente(s) primário(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) | ADR(s) de suporte |
|-----|---------------------------------------------------------|--------------------------------|----------------------|
| RFC-0001 (baseline) | `SchedulingGateway`, `EventPublisher`, `IdempotencyStore`, `SchedulerTelemetry` | FR-007, FR-008, FR-016, NFR-010, NFR-011, NFR-012 | ADR-0004, ADR-0006, ADR-0008, ADR-0010 |
| RFC-0090 (proposta) | `AdmissionController`, `PriorityCostEvaluator`, `DispatchLoop`, `DecisionJournal` | FR-001, FR-002, FR-006, FR-007, FR-011, FR-012 | ADR-0090, ADR-0091, ADR-0097 |
| RFC-0091 (proposta) | `ISchedulingPolicy`, `HeuristicPolicy`, `LearnedPolicy`, `FeatureExtractor` | FR-013, NFR-013 | ADR-0091, ADR-0096 |

---

## 4. Impacto de Mudanças Futuras na RFC-0001 sobre o Scheduler

| Cenário de mudança na RFC-0001 | Impacto esperado no Scheduler | Mitigação |
|----------------------------------|------------------------------|-----------|
| Alteração do formato de URN (§5.1) | Migração de `task_urn`/`agent_urn` em `SchedulingRequest`, `SchedulingDecision`, `QueueEntry` (chave/hash lateral no Redis) e de todo evento `task.execution.*`. | Migração versionada com dupla-leitura (RFC-0001 §9); ADR de suporte antes de aplicar; reindexação das ZSETs por novo formato de membro. |
| Alteração do envelope de evento CloudEvents (§5.2) | Todo payload publicado pelo `EventPublisher` (9 tipos de evento, §6 do brief) precisa ser revalidado contra o novo schema. | Versionamento de `dataschema` (`Events.md` §versionamento); consumidores (`006`, `007`, `023`, `025`) toleram campos aditivos por contrato (RFC-0001 §5.7). |
| Alteração da semântica de idempotência (§5.5) | `IdempotencyStore` e sua janela de retenção (`scheduler.idempotency.ttl_hours`) podem precisar de ajuste; garantia de "no máximo uma admissão efetiva" (R-08 do brief) depende diretamente desta seção. | Retenção configurável já ≥ mínimo exigido (24h); hot-reload de config (`scheduler.idempotency.ttl_hours` é recarregável). |
| Novo cabeçalho de correlação obrigatório (§5.6) | `SchedulingGateway` precisa extrair, validar e propagar o novo cabeçalho a spans OTel e a `SchedulingDecision.traceparent`. | Extração de cabeçalhos é centralizada no `SchedulingGateway` (único PEP do módulo) — baixo *fan-out* de mudança. |
| Mudança nas regras de versionamento de API (§5.7) | Pacote gRPC `aios.scheduler.v1` e base REST `/v1/scheduler` podem precisar de nova major coexistindo por ≥ 2 majors. | `ISchedulingPolicy` já isola a lógica de decisão da superfície de API (ADR-0096) — mudança de versão de API não exige reescrever a política. |

---

## 5. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `RFC-00NN-<slug>.md` em `../003-RFC/` seguindo a
   estrutura obrigatória da §2.2 e o `../003-RFC/TEMPLATE.md`.
2. A especificação **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste
   módulo nem redefinir contratos já centrais na RFC-0001; qualquer
   necessidade de divergência exige atualizar o brief primeiro.
3. Ciclo de comentário público interno: `Draft` → `Discussion` (revisão dos
   módulos consumidores listados na §3 — em especial `006-Kernel`,
   `007-Agent-Runtime`, `022-Policy`, `023-Learning`, `025-Audit`) →
   `Last Call` (janela final de objeções) → `Accepted`.
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da RFC e neste
   índice; `../003-RFC/README.md` global é atualizado.
5. RFCs aceitas evoluem apenas por **RFC sucessora** que a torna
   `Obsoleted by RFC-XXXX` — nunca por edição retroativa do texto normativo.

---

## 6. Referências

- Índice global de RFCs: `../003-RFC/README.md`
- Template de RFC: `../003-RFC/TEMPLATE.md`
- Contrato baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §3, §4, §5, §6, §11
- Índice de ADRs do módulo: `./ADR.md`
- API do módulo: `./API.md`
- Eventos do módulo: `./Events.md`

*Fim do índice de RFCs do módulo 009-Scheduler.*
