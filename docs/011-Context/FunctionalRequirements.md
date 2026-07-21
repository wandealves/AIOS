---
Documento: FunctionalRequirements
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability, 025-Audit, 021-Security, 005-Database, 040-Glossary
---

# AIOS — Módulo 011 · Context — Requisitos Funcionais

> Este documento deriva integralmente de [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md)
> §7.1 e **NÃO PODE contradizê-lo**. Os IDs `FR-001..FR-014` são **canônicos** e
> DEVEM ser reutilizados literalmente em [`Requirements.md`](./Requirements.md),
> [`UseCases.md`](./UseCases.md) e nos demais documentos do módulo (`Testing.md`,
> `Database.md`, `API.md`, `Events.md`). Nenhum FR aqui contradiz o brief — apenas
> o expande com componentes responsáveis, regras de negócio e verificação.
> Palavras normativas conforme RFC 2119/8174 (**DEVE**, **NÃO DEVE**, **DEVERIA**,
> **PODE**). Contratos centrais (URN, envelope de evento, envelope de erro RFC 7807,
> `Idempotency-Key`, correlação `traceparent`/`X-AIOS-Tenant`) vêm de
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) e **não são redefinidos**
> aqui; apenas referenciados.

## 1. Convenções

- **Prioridade (MoSCoW):** *Must* (obrigatório para GA), *Should* (importante,
  pode atrasar sem bloquear GA), *Could* (desejável, oportunista).
- **Origem:** seção da tabela de responsabilidades (`_DESIGN_BRIEF.md` §1.2, R-01…R-10)
  que gera o requisito.
- **Verificação:** como o critério de aceite é comprovado; liga-se a
  [`Testing.md`](./Testing.md) e [`Benchmark.md`](./Benchmark.md).
- **Componentes:** nomes `PascalCase` conforme [`Architecture.md`](./Architecture.md)
  §2 do brief (`ContextApiGateway`, `ContextAssembler`, `TokenBudgeter`,
  `TokenCounter`, `SelectiveRetriever`, `RelevanceRanker`, `RedundancyEliminator`,
  `HierarchicalCompressor`, `SemanticCacheManager`, `EmbeddingClient`,
  `EvictionManager`, `BundleStore`, `ContextPolicyGuard`, `EventPublisher`,
  `ModelRouterClient`, `MemoryClient`, `TelemetryEmitter`).
- Códigos de erro `AIOS-CTX-<NNNN>` conforme brief §5.3 e [`API.md`](./API.md).
  Envelope de erro RFC 7807 herdado de RFC-0001 §7.

## 2. Tabela consolidada de Requisitos Funcionais

| ID | Requisito (MoSCoW) | Critério de aceite | Origem |
|----|--------------------|---------------------|--------|
| **FR-001** (Must) | O módulo **DEVE** montar um `ContextBundle` respeitando o `token_budget` do modelo alvo informado pelo `017-Model-Router`. | `tokens_in ≤ token_budget` em 100% dos bundles em estado `SERVED`. | R-01, brief §7.1 |
| **FR-002** (Must) | O módulo **DEVE** alocar o orçamento de tokens por seção (`system`, `instruction`, `memory`, `history`, `tools`) via `BudgetProfile`. | Alocação respeita os pesos configurados com tolerância ±2%; invariante `Σ alloc_* = 1.0` validada na escrita (`AIOS-CTX-0008` se violada). | R-02, brief §7.1 |
| **FR-003** (Must) | O módulo **DEVE** comprimir contexto por sumarização hierárquica map-reduce multinível. | Redução de tokens preservando itens com `relevance_score ≥ p90`; nível máximo respeitando `context.compression.max_levels`. | R-03, brief §7.1 |
| **FR-004** (Must) | O módulo **DEVE** remover redundância entre fragmentos candidatos (near-duplicate detection). | Fragmentos com similaridade de cosseno `≥ context.dedup.cosine_threshold` (default 0,95) são colapsados sem perda de item semanticamente único. | R-04, brief §7.1 |
| **FR-005** (Must) | O módulo **DEVE** recuperar seletivamente do `010-Memory` apenas os itens necessários, rankeados por relevância dentro do orçamento. | Recall limitado a `context.retrieval.top_k`; itens abaixo do orçamento remanescente são descartados antes da montagem final. | R-05, brief §7.1 |
| **FR-006** (Must) | O módulo **DEVE** manter cache semântico indexado por similaridade (Redis L1 + `pgvector` L2). | *Hit* quando `similarity ≥ similarity_threshold` e entrada não invalidada; *miss* caso contrário; `cache_outcome` registrado no bundle. | R-06, brief §7.1 |
| **FR-007** (Must) | O módulo **DEVE** contar/estimar tokens usando o tokenizer do modelo alvo (via contrato com `017`). | Erro de estimativa ≤ 2% frente à contagem exata do provedor (NFR-010). | R-07, brief §7.1 |
| **FR-008** (Must) | O módulo **DEVE** invalidar entradas de cache reagindo a eventos de mudança da origem. | Invalidação concluída em ≤ 5 s após `aios.<t>.memory.item.consolidated`/`.deleted`. | R-06/R-09, brief §7.1 |
| **FR-009** (Must) | O módulo **DEVE** emitir eventos `context.window.*` e métricas de telemetria para toda montagem. | 100% dos `assemble` concluídos emitem `window.assembled`/`window.cache_hit`/`window.cache_miss`/`window.rejected` conforme o desfecho. | R-08, brief §7.1 |
| **FR-010** (Must) | O módulo **DEVE** degradar graciosamente quando `010-Memory` ou `017-Model-Router` estiverem indisponíveis. | Retorna bundle truncado seguro (estado `DEGRADED`) em vez de falhar a chamada. | R-10, brief §7.1 |
| **FR-011** (Must) | Operações mutantes (`assemble`, `cache store`, `budget upsert`) **DEVEM** ser idempotentes por `Idempotency-Key`. | Repetição com a mesma chave e mesmo payload retorna o mesmo resultado por ≥ 24 h; payload divergente → `AIOS-CTX-0012`. | R-01, RFC-0001 §5.5 |
| **FR-012** (Should) | O módulo **DEVERIA** expurgar bundles/cache por `source_urn`/tenant sob demanda LGPD. | Expurgo rastreável emite `context.cache.evicted` e registro em auditoria (`025`). | R-09, brief §7.1/§12.3 |
| **FR-013** (Should) | Fragmentos grandes **DEVERIAM** ser externalizados para MinIO. | Fragmento com `bytes > context.limits.max_inline_bytes` é armazenado via `content_ref`; `content_inline` permanece nulo. | R-01, brief §7.1 |
| **FR-014** (Could) | O módulo **PODE** reutilizar `SummaryNode` por similaridade em vez de sumarizar novamente. | Sumário reaproveitado quando `similarity ≥ context.cache.reuse_threshold` e `expires_at` não vencido. | R-03, brief §7.1 |

## 3. Detalhamento por requisito

### FR-001 — Assembly dentro do orçamento
- **Componentes:** `ContextApiGateway` → `ContextPolicyGuard` → `ContextAssembler`
  (orquestrador) → `TokenBudgeter` → `TokenCounter` → `BundleStore` → `EventPublisher`.
- **Regras:** a requisição `POST /v1/context/assemble` **DEVE** portar
  `X-AIOS-Tenant`, `traceparent` e `Idempotency-Key` (RFC-0001 §5.5/§5.6). Se o
  contexto mínimo obrigatório (system + instruções) já exceder o `token_budget`,
  a montagem **DEVE** ser rejeitada com `AIOS-CTX-0001` (`BudgetInfeasible`), sem
  tentar compressão parcial que viole a semântica mínima. Em sucesso, o bundle
  transita `RECEIVED → … → ASSEMBLED → SERVED` ([`StateMachine.md`](./StateMachine.md) §4.1).
- **Verificação:** teste de contrato garantindo `tokens_in ≤ token_budget` em
  100% das amostras; teste de carga cobrindo NFR-002/NFR-003.

### FR-002 — Token budgeting por `BudgetProfile`
- **Componentes:** `TokenBudgeter`, `BundleStore` (leitura de `BudgetProfile`).
- **Regras:** o perfil efetivo é resolvido por escopo mais específico primeiro
  (`agent` > `task_type` > `tenant`). A soma `alloc_system + alloc_instruction +
  alloc_memory + alloc_history + alloc_tools` **DEVE** ser exatamente `1.0`
  (tolerância de arredondamento `1e-6`); violação na escrita retorna
  `AIOS-CTX-0008` (`InvalidBudgetProfile`). `reserved_output_ratio` é subtraído do
  `model_max_tokens` **antes** da distribuição por seção.
- **Verificação:** teste unitário do alocador com perfis de borda (pesos no
  limite, escopo ausente cai no default de tenant).

### FR-003 / FR-014 — Compressão hierárquica e reuso de `SummaryNode`
- **Componentes:** `HierarchicalCompressor`, `SummarizationClient` (via `017`),
  `TokenCounter`.
- **Regras:** a compressão **DEVE** operar em no máximo
  `context.compression.max_levels` níveis (map por lote → reduce hierárquico).
  Cada nível **DEVE** preservar fragmentos com `relevance_score` no percentil
  ≥ 90 sem sumarização agressiva (extractive antes de abstractive). Antes de
  sumarizar novamente, o `HierarchicalCompressor` **DEVERIA** consultar
  `SummaryNode`s existentes por similaridade de embedding; se
  `similarity ≥ context.cache.reuse_threshold` e não expirado, reutiliza (FR-014)
  em vez de chamar o modelo de sumarização — reduzindo custo (NFR-007). Falha do
  modelo de sumarização **DEVE** acionar fallback determinístico de truncamento
  (`AIOS-CTX-0004`, ver FM-04 em `_DESIGN_BRIEF.md` §9).
- **Verificação:** bench de **Context Compression Ratio** (NFR-004) e Δ Task
  Completion Rate (NFR-005) em `./Benchmark.md`.

### FR-004 — Remoção de redundância
- **Componentes:** `RedundancyEliminator`, `EmbeddingClient`.
- **Regras:** fragmentos com similaridade de cosseno `≥
  context.dedup.cosine_threshold` **DEVEM** ser colapsados em um único fragmento
  representativo, preservando o `source_urn` de maior `relevance_score` e
  registrando os demais como `included=false`. A deduplicação **NÃO DEVE**
  remover o único fragmento que cobre uma entidade/fato exclusivo (checagem de
  cobertura mínima antes do colapso).
- **Verificação:** suite de *near-duplicate* com corpus rotulado; taxa de falso
  colapso (perda de item único) medida e reportada em `./Testing.md`.

### FR-005 — Recuperação seletiva do `010-Memory`
- **Componentes:** `SelectiveRetriever`, `MemoryClient`, `RelevanceRanker`.
- **Regras:** a chamada a `010-Memory` (`recall`) **DEVE** respeitar
  `context.retrieval.memory_deadline_ms` e limitar-se a
  `context.retrieval.top_k` candidatos. O `RelevanceRanker` combina similaridade
  semântica, recência e prioridade declarada para ordenar candidatos antes da
  fase de compressão. Timeout do `010` **DEVE** acionar o estado `DEGRADED`
  (`AIOS-CTX-0005`), nunca bloquear indefinidamente.
- **Verificação:** teste de contrato com `MemoryClient` mockado (latência e
  timeout); teste de ranking com dataset rotulado.

### FR-006 / FR-008 — Cache semântico e invalidação event-driven
- **Componentes:** `SemanticCacheManager`, `EvictionManager`, `EmbeddingClient`,
  Redis (L1), `pgvector` (L2).
- **Regras:** o lookup **DEVE** primeiro tentar `prompt_hash` (fast path exato)
  e, na ausência, busca ANN por `prompt_embedding` (HNSW, slow path); um *hit*
  exige `similarity ≥ similarity_threshold` **e** `invalidated_at IS NULL`. Ao
  consumir `aios.<tenant>.memory.item.consolidated`/`.deleted` ou
  `aios.<tenant>.knowledge.node.updated`, o `SemanticCacheManager` **DEVE**
  invalidar em ≤ 5 s toda entrada cujo `scope_ref`/proveniência sobreponha o
  `source_urn` afetado, emitindo `context.cache.invalidated`.
- **Verificação:** teste de latência de invalidação; teste de falso-hit
  (NFR-011) por amostragem/eval offline.

### FR-007 — Contagem/estimativa de tokens
- **Componentes:** `TokenCounter`, `ModelRouterClient` (registry de tokenizers).
- **Regras:** o `TokenCounter` **DEVE** usar o tokenizer exato do `model_id`
  quando disponível via `017`; na ausência, **DEVE** aplicar um estimador
  calibrado com erro ≤ 2% (NFR-010) e sinalizar a estimativa como aproximada nos
  metadados de resposta. Indisponibilidade do tokenizer **DEVE** retornar
  `AIOS-CTX-0003`.
- **Verificação:** suite de contagem comparando estimativa vs. contagem exata
  em corpus de referência multi-idioma.

### FR-009 — Emissão de eventos e métricas
- **Componentes:** `EventPublisher` (outbox transacional), `TelemetryEmitter`.
- **Regras:** toda transição terminal de `ContextBundle`
  (`SERVED`/`REJECTED`/`FAILED`) **DEVE** publicar exatamente um evento
  correspondente via outbox (at-least-once + dedup por `event.id`, RFC-0001
  §5.3/§5.4). Métricas `aios_context_*` **DEVEM** ser emitidas mesmo em
  caminhos degradados/rejeitados.
- **Verificação:** auditoria de cobertura de eventos por estado terminal
  (100% esperado, NFR-013 análogo de observabilidade).

### FR-010 — Degradação graciosa
- **Componentes:** `ContextAssembler`, `ModelRouterClient`, `MemoryClient`.
- **Regras:** indisponibilidade de `017` (limites/tokenizer) **DEVE** usar o
  último limite cacheado válido; indisponibilidade de `010` (timeout de recall)
  **DEVE** montar o bundle apenas com `system`+`instruction`+`history`
  disponíveis localmente, respeitando o orçamento. Em ambos os casos o bundle
  transita para `DEGRADED` e é servido, nunca falha silenciosamente — o
  chamador recebe `degraded=true` nos metadados.
- **Verificação:** teste de caos desligando `010`/`017` em ambiente de
  integração; verificação de que `Assemble` retorna 200 com `degraded=true`.

### FR-011 — Idempotência
- **Componentes:** `ContextApiGateway` (registro de idempotência).
- **Regras:** `assemble`, `cache/entries` (`POST`) e `budgets/{scope}` (`PUT`)
  **DEVEM** aceitar `Idempotency-Key`; repetição com o mesmo payload dentro da
  janela de retenção (≥ 24 h) **DEVE** retornar o resultado memoizado sem
  reexecutar efeitos colaterais; payload divergente com a mesma chave **DEVE**
  retornar `AIOS-CTX-0012` (409).
- **Verificação:** teste de replay com mesma/chave-payload divergente.

### FR-012 — Expurgo LGPD
- **Componentes:** `BundleStore`, `SemanticCacheManager`, `EventPublisher`.
- **Regras:** `POST /v1/context/cache/invalidate` com escopo `source_urn`/tenant
  **DEVERIA** remover/expurgar entradas de cache e bundles residuais associados,
  emitindo `context.cache.evicted` com `cause=rtbf` e registro em `025-Audit`.
  Propagado automaticamente por `aios.<tenant>.memory.item.deleted`.
- **Verificação:** teste end-to-end de expurgo com trilha de auditoria
  verificável.

### FR-013 — Offload de fragmentos grandes
- **Componentes:** `BundleStore`, MinIO.
- **Regras:** fragmentos com `bytes > context.limits.max_inline_bytes`
  **DEVERIAM** ser gravados em MinIO com `content_ref` (endereçamento por
  conteúdo), mantendo `content_inline = NULL`. A leitura do bundle **DEVE**
  reidratar transparentemente o conteúdo referenciado.
- **Verificação:** teste de limite de tamanho com fragmento sintético acima e
  abaixo do limiar.

## 4. Exemplo — verificação de FR-001 e FR-011 (request/response)

```
POST /v1/context/assemble
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZC0P5R7T2V4X6Z8B0D2F4H
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "task_urn": "urn:aios:acme:task:01J9Z9R7J8L3N5P7Q9S1T3U5W7",
  "model_id": "gpt-5-mini",
  "budget_profile_scope": "task_type:support-ticket",
  "sections": {
    "system": "Você é o agente de suporte da AmazingSoft...",
    "instruction": "Resuma o histórico e proponha a próxima ação.",
    "history_ref": "urn:aios:acme:ctxfrag:01J9Z9S8K9M4P6R8S1U3V5X7Y9"
  }
}

→ 200 OK
{
  "bundle_id": "urn:aios:acme:context:01J9ZC0Q7S9U2W4Y6A8C0E2G4J",
  "status": "SERVED",
  "cache_outcome": "MISS",
  "model_id": "gpt-5-mini",
  "token_budget": 12000,
  "tokens_raw": 18734,
  "tokens_in": 8990,
  "compression_ratio": 0.5199,
  "degraded": false
}
# Efeitos: aios.acme.context.window.assembled publicado via Outbox.
# Repetir com a MESMA Idempotency-Key → mesmo 200 e mesmo bundle_id (RFC-0001 §5.5).
# Payload divergente com a MESMA key → 409 AIOS-CTX-0012.
```

## 5. Rastreabilidade

A matriz completa `FR → UseCase → Teste` está em [`Requirements.md`](./Requirements.md)
§4. Cada FR liga-se a pelo menos um `UC-NNN` em [`UseCases.md`](./UseCases.md) e a
casos de teste em `./Testing.md`. As metas numéricas citadas (compression ratio,
erro de estimativa de tokens, latência de invalidação) pertencem aos NFRs em
[`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md) e **DEVEM**
manter valores idênticos aos do brief §7.2.
