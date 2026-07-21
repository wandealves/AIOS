---
Documento: RFC.md
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0007, ADR-0110, ADR-0111, ADR-0112, ADR-0113
RFCs relacionados: RFC-0001
Depende de: 001-Architecture, 010-Memory, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 003-RFC
---

# 011-Context — Índice de RFCs

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
> módulo — apenas **especializados** para o domínio de montagem de janela de
> contexto, compressão hierárquica, recuperação seletiva e cache semântico.

---

## 1. RFC Consumida (Baseline)

| RFC | Título | Status | Relação com o Context |
|-----|--------|--------|----------------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline consumida, não redefinida.** O Context herda integralmente: formato de URN (§5.1 — este módulo **propõe** ao registro de `004-API/Resources.md` os novos `<tipo>` `context`, `ctxfrag` e `ctxcache`, ver §2 abaixo), envelope de evento CloudEvents (§5.2), convenção de subjects (§5.3, domínio `context`), envelope de erro RFC 7807 (§5.4, faixa `AIOS-CTX-0001`..`0015`, reservável até `0099`), semântica de idempotência (§5.5, `Idempotency-Key` em `assemble`/`compress`/`cache store`/`budget upsert`), cabeçalhos de correlação obrigatórios (§5.6) e regras de versionamento de API (§5.7, pacote gRPC `aios.context.v1`). Toda RFC específica do Context (§2 abaixo) DEVE citar a seção pertinente da RFC-0001 em vez de reespecificá-la. |

### 1.1 Mapeamento de uso da RFC-0001 pelo Context

| Seção da RFC-0001 | Contrato | Uso concreto no Context |
|--------------------|----------|------------------------------|
| §5.1 URN | `urn:aios:<tenant>:<tipo>:<id>` | `ContextBundle.bundle_id` exposto como `urn:aios:<tenant>:context:<ulid>`; `ContextFragment.fragment_id` como `urn:aios:<tenant>:ctxfrag:<ulid>`; `SemanticCacheEntry.cache_id` como `urn:aios:<tenant>:ctxcache:<ulid>` — os três `<tipo>` são **propostos por este módulo** ao registro central (RFC-0011, §2). `BudgetProfile` reutiliza o `<tipo>` já registrado `policy` (subtipo lógico `context-budget`), sem propor novo tipo. |
| §5.2 Envelope de Evento (CloudEvents) | Estrutura `{id, source, type, time, datacontenttype, subject, tenant, data, traceparent}` | Payload publicado pelo `EventPublisher` (via outbox transacional) para os 8 tipos de evento emitidos do §6 do `_DESIGN_BRIEF.md` (`window.assembled`, `window.compressed`, `window.cache_hit`, `window.cache_miss`, `window.rejected`, `cache.evicted`, `cache.invalidated`, `budget.updated`). |
| §5.3 Convenção de Subjects | `aios.<tenant>.<dominio>.<entidade>.<acao>` | Domínio reservado `context` (§0 do brief); entidades `window`, `cache` e `budget` (ex.: `aios.<tenant>.context.window.assembled`, `aios.<tenant>.context.cache.invalidated`, `aios.<tenant>.context.budget.updated`). |
| §5.4 Envelope de Erro (RFC 7807) | `type/title/status/detail/instance/code/retriable` | Toda resposta de erro do `ContextApiGateway` (`API.md` §5), com `code` no domínio reservado `CTX` (`AIOS-CTX-0001`..`AIOS-CTX-0015` já catalogados no brief §5.3, faixa completa reservada até `0099`). |
| §5.5 Idempotência | `Idempotency-Key`, retenção ≥ 24h | Cobre `POST /v1/context/assemble`, `POST /v1/context/compress`, `POST /v1/context/cache/entries`, `POST /v1/context/cache/invalidate` e `PUT /v1/context/budgets/{scope}` (FR-011); repetição com a mesma chave e mesmo payload retorna o mesmo resultado por ≥ 24h; payload divergente sob a mesma chave produz `AIOS-CTX-0012` (IdempotencyConflict). |
| §5.6 Correlação | `traceparent`, `X-AIOS-Tenant` | Extraídos e validados por `ContextApiGateway`/`ContextPolicyGuard` em toda requisição; `X-AIOS-Tenant` divergente do claim autenticado é rejeitado com `AIOS-CTX-0011` (TenantMismatch); propagados a todo evento e span OTel emitido por `TelemetryEmitter`. |
| §5.7 Versionamento | Versionamento semântico de API | Pacote gRPC `aios.context.v1`; base REST `/v1/context`. Evolução do método de compressão default (`hierarchical`↔`extractive`, ver ADR-0110) ou do limiar de similaridade default (ADR-0112) NÃO incrementa versão de API — é transparente ao contrato externo, apenas configuração (`context.compression.method`, `context.cache.similarity_threshold`). |

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
| RFC-0011 | Context Assembly & Semantic Cache Contract | Planned (Draft a redigir) | Especificação normativa completa do contrato de montagem de contexto (`assemble`) e de cache semântico consumido por todo agente/tarefa do AIOS: a máquina de estados de `ContextBundle` (§4.1 do brief, `RECEIVED→POLICY_CHECK→CACHE_LOOKUP→BUDGET_ALLOCATED→RETRIEVING→RANKING→COMPRESSING→ASSEMBLED→SERVED`, com ramos `REJECTED`/`DEGRADED`/`FAILED`/`EXPIRED`); a máquina de estados de `SemanticCacheEntry` (§4.2 do brief, `PENDING→INDEXED→SERVING→STALE→INVALIDATED→EVICTED`); a definição normativa da métrica **Context Compression Ratio** (`1 - tokens_in/tokens_raw`, reutilizada do `../040-Glossary/Glossary.md`); a semântica de chave de cache por similaridade (fast path `prompt_hash` exato vs. slow path `prompt_embedding` por cosseno com `similarity_threshold`); e o registro dos novos `<tipo>` de URN `context`, `ctxfrag` e `ctxcache` para `004-API/Resources.md`. |

### 2.1 Justificativa da RFC (por que é uma RFC e não uma seção da Architecture.md)

| RFC | Justificativa |
|-----|----------------|
| RFC-0011 | O contrato de `assemble`/cache semântico é a superfície de integração entre `011-Context` e **todo** módulo que invoca um LLM no AIOS (`007-Agent-Runtime`, `012-Planning`, `013`-em-diante) — uma mudança de forma unilateral (ex.: renomear um estado de `ContextBundle`, alterar a fórmula de **Context Compression Ratio**, ou mudar a semântica de `cache_outcome`) quebraria consumidores externos e métricas já pactuadas (`024-Observability`, `035-Benchmark`), exigindo o processo de comentário público (`Draft → Discussion → Last Call`) típico de RFC, diferente de uma decisão pontual de ADR. Além disso, a proposta de três novos `<tipo>` de URN ao registro central (`004-API/Resources.md`) é, por definição da RFC-0001 §8, um ato de registro que exige RFC/ADR de módulo — não pode ser feita apenas em `Architecture.md`. |

### 2.2 Estrutura obrigatória que a RFC deste módulo DEVE seguir

Conforme `../003-RFC/README.md`, a RFC proposta DEVE conter, nesta ordem:

1. **Abstract** — resumo de 1 parágrafo do contrato de montagem de contexto e
   cache semântico especificado.
2. **Status & terminologia** (RFC 2119/8174) — palavras normativas.
3. **Motivação** — por que o contrato existe (referenciar `_DESIGN_BRIEF.md`
   §1 R-01/R-06, e a fronteira explícita com `010-Memory` §1.3 que evita
   redecidir consolidação/esquecimento).
4. **Terminologia** — reuso de `../040-Glossary/Glossary.md`, sem redefinir
   termos (`Context Window`, `Context Compression Ratio`, `Semantic Cache`,
   `Cognitive Resource`, `Hot State`).
5. **Especificação (normativa)** — o contrato em si: máquina de estados de
   `ContextBundle` (§4.1 do brief), máquina de estados de `SemanticCacheEntry`
   (§4.2 do brief), fórmula e método de amostragem da **Context Compression
   Ratio**, semântica de fast path (`prompt_hash`)/slow path
   (`prompt_embedding`) do cache, e o invariante de alocação de `BudgetProfile`
   (`alloc_system + alloc_instruction + alloc_memory + alloc_history +
   alloc_tools = 1.0`).
6. **Considerações de segurança** — referenciar `Security.md` deste módulo
   (fronteira PEP/PDP em operações privilegiadas, isolamento de cache por
   tenant, §12.1/§12.2 do brief).
7. **Considerações de privacidade (LGPD/GDPR)** — referenciar `_DESIGN_BRIEF.md`
   §12.3 (minimização de PII em eventos e logs, `content_ref` para conteúdo
   sensível, expurgo rastreável por `source_urn`/tenant propagado de
   `memory.item.deleted`).
8. **Considerações de registro (IANA-like)** — reserva de domínio de erro
   `CTX`, subjects `context.window.*`/`context.cache.*`/`context.budget.*`,
   versões de `dataschema` dos eventos, e os três novos `<tipo>` de URN
   (`context`, `ctxfrag`, `ctxcache`).
9. **Compatibilidade e versionamento** — regras de evolução sem quebra (ex.:
   adicionar um novo `compression_method` é aditivo; renomear um estado
   existente da máquina de `ContextBundle` ou mudar a fórmula da **Context
   Compression Ratio** exige nova major de RFC, dado o impacto em métricas
   pactuadas de terceiros consumidores).
10. **Referências** — normativas (RFC-0001) e informativas (ADRs do §2 de
    `ADR.md` deste módulo, em especial ADR-0110, ADR-0111 e ADR-0112).
11. **Apêndices** — exemplos completos de payload de evento
    (`window.assembled`, `window.cache_hit`) e de resposta de
    `POST /v1/context/assemble` (reaproveitar exemplos de `API.md`/`Events.md`).

---

## 3. Rastreabilidade RFC → Componente → Requisito

| RFC | Componente(s) primário(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) | ADR(s) de suporte |
|-----|---------------------------------------------------------|--------------------------------|----------------------|
| RFC-0001 (baseline) | `ContextApiGateway`, `EventPublisher`, `ContextPolicyGuard`, `TelemetryEmitter` | FR-009, FR-011, NFR-008, NFR-013, NFR-014 | ADR-0004, ADR-0006, ADR-0008, ADR-0010 |
| RFC-0011 (proposta) | `ContextAssembler`, `SemanticCacheManager`, `TokenBudgeter`, `BundleStore` | FR-001, FR-002, FR-006, NFR-004, NFR-006 | ADR-0110, ADR-0111, ADR-0112, ADR-0113 |

---

## 4. Impacto de Mudanças Futuras na RFC-0001 sobre o Context

| Cenário de mudança na RFC-0001 | Impacto esperado no Context | Mitigação |
|----------------------------------|------------------------------|-----------|
| Alteração do formato de URN (§5.1) | Migração de `bundle_id`/`fragment_id`/`cache_id` e de toda referência URN em eventos, `source_urn` e respostas de API. | Migração versionada com dupla-leitura (RFC-0001 §9); os três `<tipo>` propostos (`context`, `ctxfrag`, `ctxcache`) já isolam o impacto ao módulo, sem afetar `<tipo>` de outros módulos. |
| Alteração do envelope de evento CloudEvents (§5.2) | Todo payload publicado pelo `EventPublisher` (8 tipos emitidos, §6 do brief) precisa ser revalidado contra o novo schema. | Versionamento de `dataschema` (`Events.md` §versionamento); consumidores (`010`, `017`, `022`, `024`, `025`) toleram campos aditivos por contrato (RFC-0001 §5.7). |
| Alteração da semântica de idempotência (§5.5) | O comportamento de `Idempotency-Key` em `assemble`/`compress`/`cache store`/`budget upsert` (FR-011) depende diretamente desta seção; janela de retenção pode precisar de ajuste. | Retenção já ≥ mínimo exigido (24h) via registro de idempotência efêmero em Redis; caminho quente (`Assemble`) já trata re-execução como no-op determinístico por `bundle_id`. |
| Novo cabeçalho de correlação obrigatório (§5.6) | `ContextApiGateway`/`ContextPolicyGuard` precisam extrair, validar e propagar o novo cabeçalho a spans OTel e a todo evento emitido. | Extração de cabeçalhos é centralizada no `ContextApiGateway` (única superfície de entrada do módulo) — baixo *fan-out* de mudança. |
| Mudança nas regras de versionamento de API (§5.7) | Pacote gRPC `aios.context.v1` e base REST `/v1/context` podem precisar de nova major coexistindo por ≥ 2 majors. | As operações canônicas (`assemble/compress/cache lookup/cache store/estimate tokens/budget`) são estáveis desde o brief §5; mudanças internas de estratégia (método de compressão, limiar de similaridade) já são absorvidas por configuração, não por versão de API (ver §1.1 acima). |

---

## 5. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `RFC-0011-Context-Assembly-Semantic-Cache-Contract.md`
   em `../003-RFC/` seguindo a estrutura obrigatória da §2.2 e o
   `../003-RFC/TEMPLATE.md`.
2. A especificação **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste
   módulo nem redefinir contratos já centrais na RFC-0001; qualquer
   necessidade de divergência exige atualizar o brief primeiro.
3. Ciclo de comentário público interno: `Draft` → `Discussion` (revisão dos
   módulos consumidores listados na §3 — em especial `010-Memory` (fonte do
   `recall` seletivo, fronteira de sumarização efêmera vs. memória canônica),
   `017-Model-Router` (limites de janela, tokenizer, embedding, modelo de
   sumarização), `022-Policy`/`021-Security` (PEP/PDP, isolamento de cache por
   tenant), `024-Observability` (consumidor da métrica **Context Compression
   Ratio**)) → `Last Call` (janela final de objeções) → `Accepted`.
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

*Fim do índice de RFCs do módulo 011-Context.*
