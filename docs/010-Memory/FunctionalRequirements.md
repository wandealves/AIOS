---
Documento: FunctionalRequirements
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007, ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Requisitos Funcionais

> Derivado de `./_DESIGN_BRIEF.md` §7.1. Os IDs `FR-001..FR-014` são **canônicos** e
> reutilizados literalmente nos demais documentos (`./Requirements.md`, `./UseCases.md`,
> `./Testing.md`). Nenhum FR aqui contradiz o brief — apenas o expande com detalhe,
> componentes responsáveis e verificação. Palavras normativas conforme RFC 2119/8174.
> Contratos centrais (URN, erro, idempotência) vêm da RFC-0001, não são redefinidos.

## 1. Convenções

- **Prioridade (MoSCoW):** Must (obrigatório para GA), Should (importante), Could (desejável).
- **Origem:** seção do brief que gera o requisito.
- **Verificação:** como o critério de aceite é comprovado (liga-se a `./Testing.md`).
- **Componentes:** `PascalCase` conforme `./Architecture.md` §5.
- Códigos de erro `AIOS-MEM-NNNN` conforme brief §5.2 e `../004-API/Errors.md`.

## 2. Tabela consolidada de Requisitos Funcionais

| ID | Requisito (MoSCoW) | Critério de aceite | Origem |
|----|--------------------|--------------------|--------|
| **FR-001** (Must) | `remember(item, layer)` **DEVE** persistir o item na camada e retornar URN + estado. | Item consultável por `GET`; evento `memory.item.stored` emitido; idempotente por `Idempotency-Key`. | R1, brief §7.1 |
| **FR-002** (Must) | `recall(query, filtros)` **DEVE** retornar lista rankeada com scores por relevância. | Top-k ordenado; filtros aplicados; `mode` honrado. | R3, brief §7.1 |
| **FR-003** (Must) | Busca vetorial ANN via `pgvector`/HNSW **DEVE** operar nas camadas Semantic/Episodic. | Recall@10 ≥ 0,95 vs. busca exata em bench (NFR-002). | R3, brief §7.1 |
| **FR-004** (Must) | O roteamento **DEVE** levar o item à camada correta por `kind`/hint. | Item aterrissa na camada prevista; erro `AIOS-MEM-0020` se incompatível. | R2, brief §7.1 |
| **FR-005** (Must) | `consolidate()` **DEVE** promover itens entre camadas de forma versionada. | `ConsolidationVersion` gravada antes; eventos de job emitidos. | R4, brief §7.1 |
| **FR-006** (Must) | `forget(policy)` **DEVE** aplicar `ttl`/`decay`/`lru`/`quota`/`rtbf`. | Itens elegíveis → `FORGOTTEN`; contagem retornada; `legal_hold` respeitado. | R5, brief §7.1 |
| **FR-007** (Must) | O rollback de consolidação **DEVE** restaurar a versão anterior. | Estado idêntico à pré-imagem; evento `memory.consolidation.rolledback`. | R4, brief §7.1 |
| **FR-008** (Must) | O módulo **DEVE** impor cotas por (tenant, agente, camada) com backpressure. | Escrita negada com `AIOS-MEM-0010`/`0011` ao exceder. | R6, brief §7.1 |
| **FR-009** (Must) | Blobs grandes **DEVEM** ser externalizados para MinIO. | Conteúdo > limite vira `content_ref`; hash content-addressed. | R8, brief §7.1 |
| **FR-010** (Must) | Embeddings **DEVEM** ser obtidos exclusivamente via Model Router (017). | Nenhuma chamada direta a LLM; `embedding_model` registrado. | R7, brief §7.1 |
| **FR-011** (Must) | O direito ao esquecimento (RTBF) **DEVE** operar por item/agente/tenant. | Purge rastreável; evento `memory.item.purged`; auditoria (025). | R10, brief §7.1 |
| **FR-012** (Should) | Itens `ARCHIVED` **DEVERIAM** ser reidratados sob recall. | Recall retorna item frio reidratado transparentemente. | R2/R3, brief §7.1 |
| **FR-013** (Should) | O grafo de memória do agente (Apache AGE) **DEVERIA** servir recall por travessia. | `mode=graph` retorna itens conectados. | R3, brief §7.1 |
| **FR-014** (Could) | A consolidação **PODE** deduplicar semanticamente (merge por similaridade). | Duplicatas mescladas mantendo linhagem `parent_ids`. | R4, brief §7.1 |

## 3. Detalhamento por requisito

### FR-001 — `remember(item, layer)`
- **Componentes:** MemoryApiFacade → MemoryPep → LayerRouter → (QuotaManager, EmbeddingClient) → LayerStore correspondente → OutboxPublisher.
- **Regras:** a operação **DEVE** exigir `Idempotency-Key`, `X-AIOS-Tenant` e `traceparent` (RFC-0001 §5.5/§5.6). A resposta **DEVE** conter o URN `urn:aios:<tenant>:memory:<ULID>` e o `state` (`INGESTED`/`ACTIVE`). Se a cota estourar, **DEVE** retornar `AIOS-MEM-0010`.
- **Verificação:** teste de contrato + integração (`./Testing.md`), verificando persistência, emissão de `memory.item.stored` e idempotência (mesma key → mesmo resultado).

### FR-002 / FR-003 — `recall` híbrido e ANN
- **Componentes:** RecallEngine (fusão RRF + re-rank), LayerStores, KnowledgeGraphAdapter, EmbeddingClient.
- **Regras:** filtros **DEVEM** incluir `agent_id`, `layer[]`, `kind[]`, `tags[]`, `k`, `min_score`, `time_range`, `include_forgotten=false`, `mode` (`vector|lexical|graph|hybrid`); paginação por cursor (brief §5.1). O re-rank **DEVE** combinar relevância, recência, saliência e escopo.
- **Verificação:** bench de recall (`./Benchmark.md`) mede Recall@10 ≥ 0,95 (NFR-002); teste de filtros e ordenação.

### FR-004 — Roteamento por camada
- **Regras:** o `LayerRouter` **DEVE** rejeitar par (`layer`,`kind`) incompatível com `AIOS-MEM-0020` e dimensão de embedding incompatível com `AIOS-MEM-0021`.

### FR-005 / FR-007 / FR-014 — Consolidação, rollback e dedup
- **Componentes:** ConsolidationEngine, ConsolidationVersionManager, RetentionScheduler.
- **Regras:** toda transição a `CONSOLIDATING` **DEVE** gravar `ConsolidationVersion` antes de mutar (invariante do brief §4.1). O rollback **DEVE** ser idempotente por `version_id` e emitir `memory.consolidation.rolledback`. Rollback impossível → `AIOS-MEM-0042`.
- **Verificação:** teste A/B de Recall Rate pré/pós (NFR-011); teste de restauração de pré-imagem.

### FR-006 / FR-011 — Esquecimento e RTBF
- **Componentes:** ForgettingEngine, RetentionScheduler, BlobStoreAdapter.
- **Regras:** `legal_hold` **NÃO DEVE** ser purgado salvo RTBF autorizado (`AIOS-MEM-0050` bloqueia; `AIOS-MEM-0051` aceita/agenda). O purge físico **DEVE** remover vetor + blob + criar tombstone após `memory.forget.grace_period`.
- **Verificação:** teste de RTBF end-to-end com auditoria (025) e evento `memory.item.purged`.

### FR-008 — Cotas e backpressure
- **Regras:** QuotaManager **DEVE** contabilizar itens/bytes/vetores e sinalizar `AIOS-MEM-0011` (backpressure) / `AIOS-MEM-0010` (cota) e emitir `memory.quota.exceeded`.

### FR-009 / FR-010 — Blobs e embeddings
- **Regras:** conteúdo acima de `memory.item.inline_max_bytes` **DEVE** ir para MinIO como `content_ref` content-addressed. Falha ao obter embedding do 017 **DEVE** retornar `AIOS-MEM-0030` e enfileirar o item sem embedding (`state=INGESTED`).

### FR-012 / FR-013 — Reidratação e grafo
- **Regras:** recall sobre item `ARCHIVED` **DEVERIA** reidratar de MinIO/PG de forma transparente; `mode=graph` **DEVERIA** percorrer `KnowledgeEdge` (openCypher). Falha do grafo → degrada para `hybrid` sem grafo (`AIOS-MEM-0031` se `mode=graph` explícito).

## 4. Exemplo — verificação de FR-001 (request/response)

```
POST /v1/memory/items
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZB0M8Q7K2M4N6P8R0S2T4V
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{ "layer": "short_term", "kind": "fact",
  "content": {"text": "Cliente prefere contato por e-mail"},
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6", "tags": ["preferencia","contato"] }

→ 201 Created
{ "id": "urn:aios:acme:memory:01J9ZB0N2C4E6F8G0H2J4K6M8",
  "state": "ACTIVE", "layer": "short_term" }
# Efeito: evento aios.acme.memory.item.stored publicado via Outbox.
# Repetir com a MESMA Idempotency-Key → mesmo 201 e mesmo URN (RFC-0001 §5.5).
```

## 5. Rastreabilidade

A matriz completa `FR → UseCase → Teste` está em `./Requirements.md`. Cada FR
liga-se a pelo menos um `UC-NNN` (`./UseCases.md`) e a casos de teste em
`./Testing.md`. As metas numéricas citadas (Recall@10, Recall Rate) pertencem aos
NFRs em `./NonFunctionalRequirements.md` e **DEVEM** manter valores idênticos.
