---
Documento: RFC.md
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007, ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104
RFCs relacionados: RFC-0001
Depende de: 001-Architecture, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 025-Audit, 026-Cost-Optimizer, 003-RFC
---

# 010-Memory — Índice de RFCs

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
> módulo — apenas **especializados** para o domínio de memória hierárquica de
> sete camadas, consolidação dirigida e esquecimento controlado.

---

## 1. RFC Consumida (Baseline)

| RFC | Título | Status | Relação com o Memory |
|-----|--------|--------|----------------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline consumida, não redefinida.** O Memory herda integralmente: formato de URN (§5.1, tipo `memory` já registrado em RFC-0001 §8), envelope de evento CloudEvents (§5.2), convenção de subjects (§5.3, domínio `memory`), envelope de erro RFC 7807 (§5.4, faixa `AIOS-MEM-0001`..`0099`), semântica de idempotência (§5.5, `Idempotency-Key` em toda mutação), cabeçalhos de correlação obrigatórios (§5.6) e regras de versionamento de API (§5.7, pacote gRPC `aios.memory.v1`). Toda RFC específica do Memory (§2 abaixo) DEVE citar a seção pertinente da RFC-0001 em vez de reespecificá-la. |

### 1.1 Mapeamento de uso da RFC-0001 pelo Memory

| Seção da RFC-0001 | Contrato | Uso concreto no Memory |
|--------------------|----------|------------------------------|
| §5.1 URN | `urn:aios:<tenant>:<tipo>:<id>` | Todo `MemoryItem.id` (ULID interno) é exposto externamente como `urn:aios:<tenant>:memory:<id>` (tipo `memory` já reservado em RFC-0001 §8); `source_urn` e `parent_ids` referenciam a mesma convenção para linhagem e proveniência. |
| §5.2 Envelope de Evento (CloudEvents) | Estrutura `{id, source, type, time, datacontenttype, subject, tenant, data, traceparent}` | Payload publicado pelo `OutboxPublisher` para todos os 12 tipos de evento do §6 do `_DESIGN_BRIEF.md` (`item.stored`, `item.consolidated`, `item.forgotten`, `item.recalled`, `item.decayed`, `item.archived`, `item.purged`, `consolidation.started`, `consolidation.completed`, `consolidation.failed`, `consolidation.rolledback`, `quota.exceeded`). |
| §5.3 Convenção de Subjects | `aios.<tenant>.<dominio>.<entidade>.<acao>` | Domínio reservado `memory` (§0 do brief); entidades `item` e `consolidation` (ex.: `aios.<tenant>.memory.item.stored`, `aios.<tenant>.memory.consolidation.completed`). |
| §5.4 Envelope de Erro (RFC 7807) | `type/title/status/detail/instance/code/retriable` | Toda resposta de erro do `MemoryApiFacade` (`API.md` §5), com `code` no domínio reservado `MEM` (`AIOS-MEM-0001`..`AIOS-MEM-0060`, faixa completa reservada `0001`..`0099`). |
| §5.5 Idempotência | `Idempotency-Key`, retenção ≥ 24h | `IdempotencyStore` (Redis) cobre `remember`, `consolidate`, `forget`, `PurgeItem`, `RollbackConsolidation`; retenção mínima 24h conforme `IdempotencyRecord.expires_at` (brief §3.2). |
| §5.6 Correlação | `traceparent`, `X-AIOS-Tenant` | Extraídos e validados por `MemoryPep` em toda requisição; `X-AIOS-Tenant` divergente do claim autenticado é rejeitado com `AIOS-MEM-0005`; propagados a todo evento e span OTel emitido por `MemoryTelemetry`. |
| §5.7 Versionamento | Versionamento semântico de API | Pacote gRPC `aios.memory.v1`; base REST `/v1/memory`. Evolução da estratégia de fusão de recall (RRF↔weighted, ver ADR-0104) ou da política de esquecimento default NÃO incrementa versão de API — é transparente ao contrato externo, apenas configuração (`memory.recall.fusion`). |

---

## 2. RFCs Propostas por Este Módulo

> **Estado de redação.** A RFC abaixo está **planejada** e seu conteúdo
> pretendido está descrito no `./_DESIGN_BRIEF.md` (especialmente §3 modelo de
> dados, §4 máquinas de estado, §5 superfície de API e §6 catálogo de
> eventos), que é a fonte única de verdade até que a RFC seja redigida como
> arquivo próprio em `../003-RFC/` no formato IETF completo e promovida de
> `Draft` para `Accepted` conforme o ciclo de vida `Draft → Discussion → Last
> Call → Accepted`. O arquivo não existe fisicamente em `../003-RFC/` no
> momento desta versão do índice.

| RFC | Título | Status | Escopo pretendido |
|-----|--------|--------|----------------------|
| RFC-0100 | Memory Consolidation & Forgetting Contract | Planned (Draft a redigir) | Especificação normativa completa do contrato de consolidação dirigida por `023-Learning` e do contrato de esquecimento controlado consumido/produzido por múltiplos módulos: a máquina de estados de `MemoryItem` (§4.1 do brief, `INGESTED→ACTIVE→CONSOLIDATING→CONSOLIDATED→...→FORGOTTEN→PURGED`); a máquina de estados de `ConsolidationJob` (§4.2 do brief, `PENDING→SNAPSHOTTING→RUNNING→VALIDATING→COMMITTED` com trilha de `ROLLING_BACK`/`ROLLED_BACK`); o formato canônico de `ConsolidationVersion` e a semântica de rollback (defesa contra *catastrophic forgetting*); as cinco estratégias de `ForgettingPolicy` (`ttl\|decay\|lru\|quota\|rtbf`) e sua precedência; e os eventos `aios.<tenant>.memory.item.*`/`consolidation.*` (§6 do brief) como contrato público consumido por `023-Learning`, `025-Audit`, `026-Cost-Optimizer`. |

### 2.1 Justificativa da RFC (por que é uma RFC e não uma seção da Architecture.md)

| RFC | Justificativa |
|-----|----------------|
| RFC-0100 | O contrato de consolidação e esquecimento é a superfície de integração entre `010-Memory` e `023-Learning` (que **dirige** a consolidação sem executá-la, R4/N2 do brief) e é consumido por `025-Audit` (trilha de `forgotten`/`purged`) e `026-Cost-Optimizer` (reação a `quota.exceeded`) — uma mudança de forma unilateral (ex.: alterar quando um rollback é disparado, ou renomear um estado de `ConsolidationJob`) quebraria consumidores externos, exigindo o processo de comentário público (`Draft → Discussion → Last Call`) típico de RFC, diferente de uma decisão pontual de ADR. Além disso, o contrato de esquecimento tem implicação regulatória direta (LGPD/GDPR, RTBF) que exige especificação formal e estável, não uma nota de implementação. |

### 2.2 Estrutura obrigatória que a RFC deste módulo DEVE seguir

Conforme `../003-RFC/README.md`, a RFC proposta DEVE conter, nesta ordem:

1. **Abstract** — resumo de 1 parágrafo do contrato de consolidação e
   esquecimento especificado.
2. **Status & terminologia** (RFC 2119/8174) — palavras normativas.
3. **Motivação** — por que o contrato existe (referenciar `_DESIGN_BRIEF.md`
   §1 R4/R5, e o risco de *catastrophic forgetting* identificado em ADR-0007).
4. **Terminologia** — reuso de `../040-Glossary/Glossary.md`, sem redefinir
   termos (`Forgetting`, `Episodic Memory`, `Semantic Memory`, `Procedural
   Memory`, `Working Memory`, `Long-Term Memory`, `Knowledge Graph`).
5. **Especificação (normativa)** — o contrato em si: máquina de estados de
   `MemoryItem` (§4.1 do brief), máquina de estados de `ConsolidationJob`
   (§4.2 do brief), formato de `ConsolidationVersion`, precedência de
   `ForgettingPolicy`, e o gatilho/critério de rollback automático (NFR-011:
   regressão de Memory Recall Rate ≥ 0,01 dispara rollback).
6. **Considerações de segurança** — referenciar `Security.md` deste módulo
   (fronteira PEP/PDP em consolidação/esquecimento, §12.1/§12.2 do brief).
7. **Considerações de privacidade (LGPD/GDPR)** — referenciar `_DESIGN_BRIEF.md`
   §12.3 (base legal por item, `legal_hold`, direito ao esquecimento/RTBF,
   minimização de PII em eventos).
8. **Considerações de registro (IANA-like)** — reserva de domínio de erro
   `MEM`, subjects `memory.item.*`/`memory.consolidation.*`, versões de
   `dataschema` dos eventos de consolidação/esquecimento.
9. **Compatibilidade e versionamento** — regras de evolução sem quebra (ex.:
   adicionar uma nova `ForgettingPolicy.strategy` é aditivo; renomear um
   estado existente da máquina de `ConsolidationJob` exige nova major de RFC).
10. **Referências** — normativas (RFC-0001) e informativas (ADRs do §2 de
    `ADR.md` deste módulo, em especial ADR-0102 e ADR-0103).
11. **Apêndices** — exemplos completos de payload de evento
    (`consolidation.completed`, `item.forgotten`) e de resposta de
    `RollbackConsolidation` (reaproveitar exemplos de `API.md`/`Events.md`).

---

## 3. Rastreabilidade RFC → Componente → Requisito

| RFC | Componente(s) primário(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) | ADR(s) de suporte |
|-----|---------------------------------------------------------|--------------------------------|----------------------|
| RFC-0001 (baseline) | `MemoryApiFacade`, `OutboxPublisher`, `IdempotencyStore`, `MemoryTelemetry` | FR-001, FR-002, FR-010, NFR-005, NFR-013 | ADR-0004, ADR-0006, ADR-0008, ADR-0010 |
| RFC-0100 (proposta) | `ConsolidationEngine`, `ConsolidationVersionManager`, `ForgettingEngine`, `RetentionScheduler` | FR-005, FR-006, FR-007, FR-011, FR-014, NFR-011 | ADR-0102, ADR-0103, ADR-0109 |

---

## 4. Impacto de Mudanças Futuras na RFC-0001 sobre o Memory

| Cenário de mudança na RFC-0001 | Impacto esperado no Memory | Mitigação |
|----------------------------------|------------------------------|-----------|
| Alteração do formato de URN (§5.1) | Migração de `MemoryItem.id`/`source_urn`/`parent_ids` e de toda referência URN em eventos e respostas de API. | Migração versionada com dupla-leitura (RFC-0001 §9); ADR de suporte antes de aplicar; reindexação de índices por hash/URN afetados. |
| Alteração do envelope de evento CloudEvents (§5.2) | Todo payload publicado pelo `OutboxPublisher` (12 tipos de evento, §6 do brief) precisa ser revalidado contra o novo schema. | Versionamento de `dataschema` (`Events.md` §versionamento); consumidores (`023`, `024`, `025`, `026`, `011`) toleram campos aditivos por contrato (RFC-0001 §5.7). |
| Alteração da semântica de idempotência (§5.5) | `IdempotencyStore` e sua janela de retenção podem precisar de ajuste; a garantia de "no máximo um efeito por `Idempotency-Key`" (R1/FR-001 do brief) depende diretamente desta seção. | Retenção configurável já ≥ mínimo exigido (24h); recarregável em runtime; `ForgettingEngine`/`ConsolidationEngine` já tratam re-execução como no-op idempotente por `version_id`. |
| Novo cabeçalho de correlação obrigatório (§5.6) | `MemoryPep` precisa extrair, validar e propagar o novo cabeçalho a spans OTel e a todo evento emitido. | Extração de cabeçalhos é centralizada no `MemoryPep` (único PEP do módulo) — baixo *fan-out* de mudança. |
| Mudança nas regras de versionamento de API (§5.7) | Pacote gRPC `aios.memory.v1` e base REST `/v1/memory` podem precisar de nova major coexistindo por ≥ 2 majors. | As quatro operações canônicas (`remember/recall/consolidate/forget`) são estáveis desde ADR-0007; mudanças internas de estratégia (fusão, política de esquecimento) já são absorvidas por configuração, não por versão de API. |

---

## 5. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `RFC-0100-Memory-Consolidation-Forgetting-Contract.md`
   em `../003-RFC/` seguindo a estrutura obrigatória da §2.2 e o
   `../003-RFC/TEMPLATE.md`.
2. A especificação **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste
   módulo nem redefinir contratos já centrais na RFC-0001; qualquer
   necessidade de divergência exige atualizar o brief primeiro.
3. Ciclo de comentário público interno: `Draft` → `Discussion` (revisão dos
   módulos consumidores listados na §3 — em especial `023-Learning`
   (dirige/consome rollback), `025-Audit` (trilha de forgotten/purged),
   `021-Security` (RTBF/`legal_hold`), `026-Cost-Optimizer` (reação a
   `quota.exceeded`)) → `Last Call` (janela final de objeções) → `Accepted`.
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

*Fim do índice de RFCs do módulo 010-Memory.*
