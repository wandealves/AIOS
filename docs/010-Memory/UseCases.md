---
Documento: UseCases
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# Casos de Uso — Módulo 010-Memory

> Este documento deriva integralmente do
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) (§1, §4, §5, §6, §7, §9) e **NÃO PODE
> contradizê-lo**. Contratos centrais (URN, envelope de evento, envelope de erro,
> idempotência, correlação, subjects) são **reutilizados** de
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md), nunca redefinidos.
> Palavras normativas conforme RFC 2119/8174 (**DEVE**, **NÃO DEVE**, **DEVERIA**, **PODE**).

## 1. Convenções e atores

Cada caso de uso segue o formato **ator / pré-condições / fluxo principal / fluxos
alternativos / exceções / pós-condições**, e é rastreável até os requisitos
funcionais ([`FunctionalRequirements.md`](./FunctionalRequirements.md)) da §7.1 do
brief. Toda mutação DEVE portar `Idempotency-Key`, `X-AIOS-Tenant` e `traceparent`
(RFC-0001 §5.5, §5.6). Toda operação DEVE ser autorizada pelo `MemoryPep` junto ao
PDP (022) em regime *default deny*.

### 1.1 Atores

| Ator | Tipo | Descrição |
|------|------|-----------|
| **AgentRuntime** | Primário (via API) | Runtime Python (plano de dados) que chama a API do `MemoryService` por gRPC/NATS; NÃO acessa datastores diretamente (brief §1.2, fronteira crítica). |
| **ContextService (011)** | Primário | Solicita recall seletivo para montar a janela de contexto. |
| **LearningService (023)** | Primário | Dirige consolidação e rollback de políticas. |
| **SecurityService (021/025)** | Primário | Dispara expurgo por RTBF (direito ao esquecimento). |
| **AgentLifecycle (006/008)** | Primário | Emite eventos de término/suspensão que expiram ou esfriam memória. |
| **RetentionScheduler** | Interno | Agenda decaimento/consolidação/expurgo (cron + reação a eventos). |
| **Operador de Plataforma** | Secundário | Consulta estatísticas/cotas e configura políticas. |
| **PolicyEngine (022)** | Suporte | PDP consultado pelo `MemoryPep`. |
| **ModelRouter (017)** | Suporte | Gera embeddings sob demanda. |
| **CostOptimizer (026)** | Suporte | Recebe consumo/cotas. |

### 1.2 Índice de casos de uso

| UC | Nome | Ator primário | FR de origem |
|----|------|---------------|--------------|
| UC-001 | Lembrar item (`remember`) | AgentRuntime | FR-001, FR-004, FR-009, FR-010 |
| UC-002 | Recuperar memória rankeada (`recall`) | AgentRuntime / ContextService | FR-002, FR-003, FR-012, FR-013 |
| UC-003 | Consolidar memória (`consolidate`) | LearningService | FR-005, FR-014 |
| UC-004 | Reverter consolidação (`rollback`) | LearningService | FR-007 |
| UC-005 | Esquecer por política (`forget`) | RetentionScheduler / AgentRuntime | FR-006 |
| UC-006 | Expurgar item por RTBF | SecurityService | FR-011 |
| UC-007 | Consultar item por URN (`getItem`) | AgentRuntime | FR-001 |
| UC-008 | Consultar estatísticas e cotas (`getStats`) | Operador | FR-008 |
| UC-009 | Reagir ao término/suspensão de agente | AgentLifecycle | FR-006, FR-012 |
| UC-010 | Aplicar backpressure/cota na escrita | QuotaManager | FR-008 |

---

## 2. UC-001 — Lembrar item (`remember`)

- **Ator primário:** AgentRuntime.
- **Atores de suporte:** MemoryPep, PolicyEngine (022), LayerRouter, QuotaManager,
  EmbeddingClient, ModelRouter (017), LayerStores, BlobStoreAdapter, OutboxPublisher.
- **Objetivo:** persistir um `MemoryItem` na camada correta e devolver seu URN e estado.
- **Rastreabilidade:** FR-001, FR-004, FR-009, FR-010; NFR-001.

### 2.1 Pré-condições

1. O chamador DEVE estar autenticado (OIDC/mTLS, RFC-0001 §6) e portar
   `X-AIOS-Tenant`, `Idempotency-Key` e `traceparent`.
2. O `tenant_id` do recurso DEVE coincidir com o contexto autenticado.
3. A camada alvo (`layer`) DEVE ser compatível com o `kind` do item.

### 2.2 Fluxo principal (caminho feliz)

1. AgentRuntime envia `POST /v1/memory/items` (ou gRPC `Remember`) com `{content, layer, kind, tags, legal_basis, retention_class, ...}`.
2. `MemoryPep` valida `Idempotency-Key` no `IdempotencyStore`, impõe o tenant e consulta o PDP (022) — decisão *allow*.
3. `LayerRouter` classifica o item, decide inline vs. blob (`memory.item.inline_max_bytes`) e aciona `QuotaManager`.
4. `QuotaManager` confirma cota disponível para `(tenant, agente, camada)`.
5. Se a camada exigir embedding (Semantic/Episodic), `EmbeddingClient` solicita o vetor ao ModelRouter (017) e o persiste em `pgvector`.
6. O `LayerStore` correspondente persiste o item; o estado transita `INGESTED → ACTIVE` ([`StateMachine.md`](./StateMachine.md) §2).
7. `OutboxPublisher` grava a linha de outbox e publica `aios.<t>.memory.item.stored`.
8. A API retorna `201/OK` com `urn:aios:<tenant>:memory:<ULID>` e `state=ACTIVE`.

### 2.3 Fluxos alternativos

- **A1 — Requisição repetida (idempotência):** se o `Idempotency-Key` já existe com o mesmo payload, o passo 2 retorna o resultado memoizado (≥ 24 h) sem re-executar 3–7.
- **A2 — Conteúdo grande:** no passo 3, se `bytes > memory.item.inline_max_bytes`, `BlobStoreAdapter` externaliza para MinIO (content-addressed por `content_hash`) e grava `content_ref` (FR-009).
- **A3 — Camada sem embedding:** para Working/Short-Term/Long-Term/Procedural, o passo 5 é omitido.
- **A4 — ModelRouter indisponível:** o item é persistido em `state=INGESTED` sem embedding e o vetor é gerado em backlog (brief §9 F3), sem perda.

### 2.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| Schema/DTO inválido | `AIOS-MEM-0001` | 400 | Rejeita; nada persistido. |
| Idempotency-Key com payload divergente | `AIOS-MEM-0003` | 409 | Rejeita. |
| PDP nega | `AIOS-MEM-0004` | 403 | Rejeita; auditoria (025). |
| Tenant divergente | `AIOS-MEM-0005` | 403 | Rejeita. |
| Cota de camada excedida | `AIOS-MEM-0010` | 429 | Retriable; dispara poda. |
| Backpressure ativo | `AIOS-MEM-0011` | 429 | Retriable. |
| Camada incompatível com `kind` | `AIOS-MEM-0020` | 422 | Rejeita. |
| Dimensão de embedding incompatível | `AIOS-MEM-0021` | 422 | Rejeita. |
| Falha ao obter embedding | `AIOS-MEM-0030` | 502 | Retriable; ver A4. |
| Backend de camada indisponível | `AIOS-MEM-0031` | 503 | Retriable. |

### 2.5 Pós-condições

1. Em sucesso, o item é consultável por `GET /v1/memory/items/{id}` em `state=ACTIVE`.
2. Exatamente um evento `aios.<t>.memory.item.stored` DEVE ser publicado (at-least-once via Outbox).
3. O `QuotaManager` reflete o novo consumo de itens/bytes/vetores.

---

## 3. UC-002 — Recuperar memória rankeada (`recall`)

- **Ator primário:** AgentRuntime ou ContextService (011).
- **Atores de suporte:** MemoryPep, RecallEngine, LayerStores, KnowledgeGraphAdapter, EmbeddingClient, MemoryTelemetry.
- **Objetivo:** retornar lista `RecallResult` rankeada por relevância/recência/saliência/escopo.
- **Rastreabilidade:** FR-002, FR-003, FR-012, FR-013; NFR-002, NFR-003.

### 3.1 Pré-condições

1. Chamador autenticado, com `X-AIOS-Tenant` e `traceparent`.
2. Consulta com filtros válidos: `agent_id`, `layer[]`, `kind[]`, `tags[]`, `k`, `min_score`, `time_range`, `include_forgotten=false`, `mode` (`vector|lexical|graph|hybrid`).

### 3.2 Fluxo principal (caminho feliz)

1. Chamador envia `POST /v1/memory/recall` com `query` + filtros.
2. `MemoryPep` autoriza a leitura (PDP 022) e impõe o tenant (RLS).
3. `RecallEngine` executa a busca conforme `mode`: lexical, ANN vetorial (`pgvector`/HNSW), travessia de grafo (Apache AGE) ou híbrida.
4. Para `mode` vetorial/híbrido, `EmbeddingClient` obtém o embedding da `query`.
5. `RecallEngine` funde os candidatos por **RRF** (`memory.recall.fusion=rrf`) e re-rankeia por relevância/recência/saliência/escopo.
6. Aplica `k` (top-k, default `memory.recall.default_k`) e `min_score` (default `memory.recall.min_score`).
7. `MemoryTelemetry` registra a **Memory Recall Rate** (`aios_memory_recall_rate`) e emite `aios.<t>.memory.item.recalled` (amostrado).
8. A API retorna a lista `RecallResult` com scores.

### 3.3 Fluxos alternativos

- **A1 — Item ARCHIVED:** se um resultado está `ARCHIVED` (cold/MinIO), `RecallEngine` reidrata o conteúdo transparentemente (FR-012; transição `ARCHIVED → ACTIVE`).
- **A2 — `mode=graph`:** a recuperação percorre `KnowledgeEdge` via openCypher (FR-013).
- **A3 — Recall seletivo do Context:** quando disparado por `aios.<t>.context.recall.requested`, o resultado é devolvido ao ContextService (011) que monta a janela.
- **A4 — Grafo indisponível:** em `mode=hybrid`, o recall degrada para busca sem grafo (brief §9 F5), mantendo resposta.

### 3.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| Filtros/DTO inválidos | `AIOS-MEM-0001` | 400 | Rejeita. |
| PDP nega | `AIOS-MEM-0004` | 403 | Rejeita. |
| Tenant divergente | `AIOS-MEM-0005` | 403 | Rejeita. |
| Dimensão de embedding incompatível | `AIOS-MEM-0021` | 422 | Rejeita. |
| Falha de embedding (modo vetorial) | `AIOS-MEM-0030` | 502 | Retriable. |
| Backend/grafo indisponível em `mode=graph` | `AIOS-MEM-0031` | 503 | Retriable; ver A4 para híbrido. |

### 3.5 Pós-condições

1. A lista retornada respeita `k`, `min_score` e `include_forgotten=false` (itens `FORGOTTEN`/`PURGED` NÃO DEVEM aparecer salvo pedido explícito).
2. A métrica `aios_memory_recall_rate` é atualizada; nenhum estado de item é mutado (exceto `access_count`/`last_access_at` e possível reidratação A1).

---

## 4. UC-003 — Consolidar memória (`consolidate`)

- **Ator primário:** LearningService (023).
- **Atores de suporte:** MemoryPep, ConsolidationEngine, ConsolidationVersionManager, LayerStores, OutboxPublisher, RetentionScheduler.
- **Objetivo:** promover/mesclar/deduplicar itens entre camadas de forma versionada e reversível.
- **Rastreabilidade:** FR-005, FR-014; NFR-011.

### 4.1 Pré-condições

1. Existe demanda de consolidação: chamada `consolidate()`, evento `aios.<t>.learning.consolidation.requested`, `consolidation_threshold` atingido, ou janela agendada.
2. Há cota e política que permitam a promoção `from_layer → to_layer`.

### 4.2 Fluxo principal (caminho feliz)

1. Gatilho cria um `ConsolidationJob` em `PENDING` ([`StateMachine.md`](./StateMachine.md) §3).
2. `RetentionScheduler` admite o job; lock distribuído (Redis) por `(tenant, agente)` serializa jobs concorrentes.
3. `ConsolidationVersionManager` grava a pré-imagem (`ConsolidationVersion`) — job em `SNAPSHOTTING` (invariante: snapshot antes de mutar).
4. `ConsolidationEngine` promove/mescla/deduplica itens — job em `RUNNING`; os `MemoryItem` transitam `ACTIVE → CONSOLIDATING`; emite `aios.<t>.memory.consolidation.started`.
5. Validação de integridade/linhagem/Recall Rate — job em `VALIDATING` (NFR-011: pós ≥ pré − 0,01).
6. Marca a versão ativa — job em `COMMITTED`; itens `CONSOLIDATING → CONSOLIDATED`; emite `aios.<t>.memory.item.consolidated` e `aios.<t>.memory.consolidation.completed`.
7. A API (chamada síncrona inicial) retorna `job_id`; o status é acompanhado por `GET /v1/memory/jobs/{id}`.

### 4.3 Fluxos alternativos

- **A1 — Deduplicação semântica:** no passo 4, duplicatas por similaridade são mescladas mantendo linhagem `parent_ids` (FR-014).
- **A2 — Threshold automático:** quando `memory.consolidation.auto=true` e `consolidation_threshold` é atingido, o job é criado sem chamada explícita.

### 4.4 Exceções

| Condição | Código/estado | Efeito |
|----------|---------------|--------|
| Cota/política negam | `REJECTED` (`AIOS-MEM-0010`) | Job terminal rejeitado. |
| Conflito de versão concorrente | `AIOS-MEM-0040` (409) | Rejeita ou serializa via lock. |
| Falha durante `RUNNING`/`VALIDATING` | `ROLLING_BACK → FAILED`/`ROLLED_BACK` (`AIOS-MEM-0041`) | Rollback automático à versão anterior; itens `CONSOLIDATING → ACTIVE`; emite `consolidation.failed`/`rolledback`. |
| Regressão de Recall Rate (NFR-011) | `ROLLING_BACK → ROLLED_BACK` | Reverte; emite `consolidation.rolledback`. |

### 4.5 Pós-condições

1. Em `COMMITTED`, uma `ConsolidationVersion` ativa existe e os itens estão `CONSOLIDATED`.
2. Em falha, o estado é idêntico à pré-imagem (rollback), com evento `consolidation.failed` ou `rolledback`.
3. A retenção de versões respeita `memory.consolidation.version.retention`.

---

## 5. UC-004 — Reverter consolidação (`rollback`)

- **Ator primário:** LearningService (023) ou Operador.
- **Atores de suporte:** ConsolidationVersionManager, ConsolidationEngine, OutboxPublisher.
- **Objetivo:** restaurar o estado de uma consolidação anterior (defesa contra *catastrophic forgetting*).
- **Rastreabilidade:** FR-007; NFR-011.

### 5.1 Pré-condições

1. Existe uma `ConsolidationVersion` anterior válida e não ativa para o alvo.
2. O chamador está autorizado (PDP 022).

### 5.2 Fluxo principal

1. Chamador envia `POST /v1/memory/consolidate/{jobId}/rollback` (ou consome `aios.<t>.learning.policy.rolledback`).
2. `ConsolidationVersionManager` localiza a versão anterior.
3. `ConsolidationEngine` restaura os itens à pré-imagem; job entra em `ROLLING_BACK → ROLLED_BACK`.
4. `OutboxPublisher` emite `aios.<t>.memory.consolidation.rolledback`.
5. A API retorna o resultado da reversão.

### 5.3 Fluxos alternativos

- **A1 — Disparo por evento de Learning:** o consumo de `aios.<t>.learning.policy.rolledback` reverte a consolidação associada à versão.

### 5.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| Versão inexistente/já ativa | `AIOS-MEM-0042` | 409 |
| PDP nega | `AIOS-MEM-0004` | 403 |
| Backend indisponível | `AIOS-MEM-0031` | 503 |

### 5.5 Pós-condições

1. O estado dos itens é idêntico à `ConsolidationVersion` anterior.
2. Evento `consolidation.rolledback` publicado; auditoria (025) registra a reversão.

---

## 6. UC-005 — Esquecer por política (`forget`)

- **Ator primário:** RetentionScheduler (cron) ou AgentRuntime.
- **Atores de suporte:** ForgettingEngine, QuotaManager, LayerStores, OutboxPublisher.
- **Objetivo:** aplicar `ttl|decay|lru|quota|rtbf` e mover itens elegíveis a `FORGOTTEN`.
- **Rastreabilidade:** FR-006; NFR-009.

### 6.1 Pré-condições

1. Existe uma `ForgettingPolicy` habilitada (`enabled=true`) para o escopo (`tenant|agent|layer`).
2. Itens candidatos existem (por `decay_score < θ`, `expires_at` vencido, LRU ou cota).

### 6.2 Fluxo principal

1. Gatilho: `POST /v1/memory/forget` ou job agendado.
2. `ForgettingEngine` seleciona itens elegíveis conforme `strategy`.
3. Itens transitam `DECAYING → FORGET_PENDING → FORGOTTEN`, com tombstone criado.
4. `OutboxPublisher` emite `aios.<t>.memory.item.forgotten` (e `decayed`/`archived` conforme o caso).
5. Após `memory.forget.grace_period`, itens `FORGOTTEN → PURGED` (purge físico + remoção de vetor/blob).
6. A API retorna a contagem de itens afetados.

### 6.3 Fluxos alternativos

- **A1 — Decaimento contínuo:** jobs periódicos reduzem `decay_score`; itens abaixo de `memory.decay.threshold` entram em `DECAYING` e emitem `item.decayed`.
- **A2 — Poda por cota:** disparada por `AIOS-MEM-0010`, prioriza LRU/menor saliência (NFR-009: uso ≤ 100% da cota).
- **A3 — Arquivamento:** itens frios `CONSOLIDATED → ARCHIVED` movem conteúdo para MinIO e emitem `item.archived`.

### 6.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| Item sob `legal_hold` | `AIOS-MEM-0050` | 423 | Esquecimento bloqueado (salvo RTBF autorizado). |
| PDP nega | `AIOS-MEM-0004` | 403 | Rejeita. |
| Backend indisponível | `AIOS-MEM-0031` | 503 | Retriable. |

### 6.5 Pós-condições

1. Itens elegíveis estão em `FORGOTTEN` (tombstone + linhagem preservada para auditoria) ou `PURGED` após grace period.
2. `legal_hold` NÃO DEVE ir a `PURGED` salvo RTBF autorizado (invariante §4.1 do brief).
3. `QuotaManager` reflete a liberação de espaço.

---

## 7. UC-006 — Expurgar item por RTBF (direito ao esquecimento)

- **Ator primário:** SecurityService (021/025).
- **Atores de suporte:** MemoryPep, ForgettingEngine, BlobStoreAdapter, OutboxPublisher.
- **Objetivo:** remover de forma rastreável dados de um sujeito (LGPD/GDPR).
- **Rastreabilidade:** FR-011; NFR-010.

### 7.1 Pré-condições

1. Existe base legal para o expurgo (RTBF autorizado); pode incidir mesmo sobre `legal_hold`.
2. Alvo identificado por item/agente/tenant.

### 7.2 Fluxo principal

1. Gatilho: `DELETE /v1/memory/items/{id}` (`PurgeItem`) ou evento `aios.<t>.security.rtbf.requested`.
2. `MemoryPep` autoriza o expurgo RTBF.
3. `ForgettingEngine` cria tombstone e agenda o purge (`FORGET_PENDING → FORGOTTEN → PURGED`).
4. `BlobStoreAdapter` remove blobs em MinIO; o vetor `pgvector` é removido.
5. `OutboxPublisher` emite `aios.<t>.memory.item.purged`.
6. A API retorna `202 Accepted` (`AIOS-MEM-0051`), operação assíncrona.

### 7.3 Fluxos alternativos

- **A1 — RTBF por agente/tenant:** o expurgo cobre todos os itens do escopo, em lote.

### 7.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| Item inexistente | `AIOS-MEM-0002` | 404 |
| Autorização RTBF ausente | `AIOS-MEM-0004` | 403 |
| Backend indisponível | `AIOS-MEM-0031` | 503 |

### 7.5 Pós-condições

1. Conteúdo, vetor e blob do alvo são fisicamente removidos após purge; tombstone mantido para auditoria RTBF.
2. Evento `item.purged` consumido pelo Audit (025).

---

## 8. UC-007 — Consultar item por URN (`getItem`)

- **Ator primário:** AgentRuntime.
- **Objetivo:** recuperar um `MemoryItem` por seu URN.
- **Rastreabilidade:** FR-001.

### 8.1 Pré-condições

1. Chamador autenticado com `X-AIOS-Tenant`.

### 8.2 Fluxo principal

1. `GET /v1/memory/items/{id}` (ou gRPC `GetItem`).
2. `MemoryPep` autoriza a leitura e impõe RLS por tenant.
3. O `LayerStore` retorna o item (reidratando de MinIO se `ARCHIVED`).

### 8.3 Fluxos alternativos

- **A1 — Item ARCHIVED:** reidratação transparente do conteúdo frio.

### 8.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| Item inexistente | `AIOS-MEM-0002` | 404 |
| PDP nega | `AIOS-MEM-0004` | 403 |
| Tenant divergente | `AIOS-MEM-0005` | 403 |

### 8.5 Pós-condições

1. Item retornado sem mutação de estado (exceto `access_count`/`last_access_at`).

---

## 9. UC-008 — Consultar estatísticas e cotas (`getStats`)

- **Ator primário:** Operador de Plataforma.
- **Atores de suporte:** QuotaManager, MemoryTelemetry.
- **Objetivo:** expor uso por camada/tenant/agente e a Memory Recall Rate.
- **Rastreabilidade:** FR-008; NFR-002, NFR-009.

### 9.1 Pré-condições

1. Chamador autorizado a ler estatísticas do tenant.

### 9.2 Fluxo principal

1. `GET /v1/memory/stats` (ou `GET /v1/memory/layers`).
2. `QuotaManager` agrega itens/bytes/vetores; `MemoryTelemetry` fornece `aios_memory_recall_rate` e `aios_memory_usage_ratio`.
3. A API retorna as métricas por escopo.

### 9.3 Fluxos alternativos

- **A1 — Config por camada:** `GET /v1/memory/layers` retorna a `MemoryLayerConfig` vigente.

### 9.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| PDP nega | `AIOS-MEM-0004` | 403 |
| Erro interno | `AIOS-MEM-0060` | 500 |

### 9.5 Pós-condições

1. Estatísticas retornadas refletem o consumo corrente; nenhuma mutação.

---

## 10. UC-009 — Reagir ao término/suspensão de agente

- **Ator primário:** AgentLifecycle (006/008) via evento.
- **Atores de suporte:** WorkingMemoryStore, ShortTermMemoryStore, BlobStoreAdapter, ForgettingEngine.
- **Objetivo:** expirar memória volátil ou esfriar estado quente ao encerrar/suspender um agente.
- **Rastreabilidade:** FR-006, FR-012; NFR-008.

### 10.1 Pré-condições

1. Chega `aios.<t>.agent.lifecycle.terminated` ou `.suspended`.

### 10.2 Fluxo principal

1. **Terminado:** o MemoryService expira Working/Short-Term do agente (Redis TTL/purge).
2. **Suspenso:** move estado quente → cold (MinIO/PG); itens `CONSOLIDATED → ARCHIVED` quando aplicável.
3. Emite `item.forgotten`/`item.archived` conforme o caso.

### 10.3 Fluxos alternativos

- **A1 — Reidratação futura:** ao ser retomado, um recall reidrata itens `ARCHIVED` (UC-002 A1).

### 10.4 Exceções

| Condição | Efeito |
|----------|--------|
| Redis indisponível (F1) | Working reconstruível; degradação graciosa; camadas duráveis preservadas. |
| Evento duplicado | Consumidor deduplica por `event.id` (idempotente). |

### 10.5 Pós-condições

1. Working/Short-Term do agente terminado é expirado (perda tolerada, NFR-008).
2. Estado do agente suspenso permanece recuperável (cold).

---

## 11. UC-010 — Aplicar backpressure/cota na escrita

- **Ator primário:** QuotaManager (interno, no caminho de `remember`).
- **Atores de suporte:** LayerRouter, ForgettingEngine, CostOptimizer (026).
- **Objetivo:** conter crescimento descontrolado e represar escrita ao saturar.
- **Rastreabilidade:** FR-008; NFR-009.

### 11.1 Pré-condições

1. Uma escrita (`remember`) está em curso; contadores de cota estão disponíveis.

### 11.2 Fluxo principal

1. `QuotaManager` verifica `(used_items, used_bytes)` contra `(limit_items, limit_bytes)`.
2. Se dentro do limite e `inflight < memory.backpressure.max_inflight`, admite a escrita.
3. Reporta consumo ao CostOptimizer (026).

### 11.3 Fluxos alternativos

- **A1 — Poda reativa:** ao exceder cota, dispara `ForgettingEngine` (poda LRU/menor saliência) e emite `aios.<t>.memory.quota.exceeded`.

### 11.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| Cota excedida | `AIOS-MEM-0010` | 429 (retriable) |
| Backpressure (inflight saturado) | `AIOS-MEM-0011` | 429 (retriable) |

### 11.5 Pós-condições

1. Escrita admitida ou represada; uso do tenant NÃO DEVE exceder a cota (`aios_memory_usage_ratio < 1,0`).
2. Evento `quota.exceeded` emitido quando aplicável.

---

## 12. Matriz de rastreabilidade UC → FR → evento

| UC | FR | Evento(s) emitido(s) | Estado(s) afetado(s) |
|----|----|----------------------|----------------------|
| UC-001 | FR-001, FR-004, FR-009, FR-010 | `item.stored` | `INGESTED→ACTIVE` |
| UC-002 | FR-002, FR-003, FR-012, FR-013 | `item.recalled` | `ARCHIVED→ACTIVE` (reidr.) |
| UC-003 | FR-005, FR-014 | `consolidation.started/completed`, `item.consolidated` | `ACTIVE→CONSOLIDATING→CONSOLIDATED` |
| UC-004 | FR-007 | `consolidation.rolledback` | `CONSOLIDATING→ACTIVE` |
| UC-005 | FR-006 | `item.forgotten/decayed/archived` | `DECAYING→FORGET_PENDING→FORGOTTEN→PURGED` |
| UC-006 | FR-011 | `item.purged` | `FORGET_PENDING→FORGOTTEN→PURGED` |
| UC-007 | FR-001 | — | — |
| UC-008 | FR-008 | — | — |
| UC-009 | FR-006, FR-012 | `item.forgotten/archived` | `CONSOLIDATED→ARCHIVED` |
| UC-010 | FR-008 | `quota.exceeded` | — |

> Referências: fluxos detalhados em [`SequenceDiagrams.md`](./SequenceDiagrams.md);
> estados e transições em [`StateMachine.md`](./StateMachine.md); contratos de API em
> [`API.md`](./API.md); eventos em [`Events.md`](./Events.md).
