---
Documento: SequenceDiagrams
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0116, ADR-0118; ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability, 020-Communication
---

# 011-Context — SequenceDiagrams

> Diagramas de sequência ASCII para os fluxos críticos do `ContextService` —
> caminho feliz e caminhos de falha — conforme componentes de
> `Architecture.md`, contratos de `_DESIGN_BRIEF.md` §5/§6 e a máquina de
> estados de `StateMachine.md`. Setas `───▶` denotam chamada síncrona
> (gRPC/REST, aguarda resposta); setas `┄┄▶` denotam mensagem assíncrona
> (evento NATS/JetStream); `- - -` denota resposta/retorno.

## Índice

1. Convenções de notação
2. Fluxo 1 (feliz) — Assemble com *cache miss* completo (retrieve→rank→dedup→compress)
3. Fluxo 2 (feliz) — Assemble com *cache hit* (fast path)
4. Fluxo 3 (falha) — Orçamento infeasível (`REJECTED`)
5. Fluxo 4 (falha) — Policy Engine indisponível (*default deny*)
6. Fluxo 5 (falha) — `010-Memory` timeout no recall (`DEGRADED`)
7. Fluxo 6 (falha) — `017-Model-Router` indisponível para limites/tokenizer
8. Fluxo 7 (falha) — Falha de sumarização com *fallback* de truncamento
9. Fluxo 8 (falha) — Backend de cache indisponível (`BYPASS`)
10. Fluxo 9 (feliz) — Idempotência: retry de `Assemble` com mesma `Idempotency-Key`
11. Fluxo 10 (feliz) — Invalidação de cache por evento externo (`memory.item.consolidated`)
12. Fluxo 11 (falha) — Tenant divergente (`TenantMismatch`)
13. Fluxo 12 (feliz) — Expurgo LGPD por `source_urn` (direito ao esquecimento)
14. Timeouts e SLOs por fluxo

---

## 1. Convenções de Notação

```
Participante A     Participante B
     │                    │
     │──── msg síncrona ─▶│   (aguarda resposta; timeout aplicável)
     │◀- - - resposta - - -│
     │┄┄┄▶ evento async ┄┄▶│   (publish/consume via NATS/JetStream, sem aguardar)
     │                    │
     ┌───────────────┐
     │ processamento  │   (caixa de atividade/computação local)
     └───────────────┘
```

Todos os fluxos assumem os cabeçalhos obrigatórios de RFC-0001 §5.6
(`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key` em mutações) mesmo quando
omitidos do diagrama por brevidade.

---

## 2. Fluxo 1 (Feliz) — Assemble com *Cache Miss* Completo

**Participantes:** AgentRuntime(007), ContextApiGateway, ContextPolicyGuard,
PolicyEngine(022), SemanticCacheManager, TokenBudgeter,
ModelRouterClient→ModelRouter(017), SelectiveRetriever,
MemoryClient→Memory(010), RelevanceRanker, RedundancyEliminator,
HierarchicalCompressor, BundleStore, EventPublisher→NATS/JetStream.

```
Runtime  ApiGw    PolicyGuard  Policy(022)  CacheMgr   Budgeter  ModelRouterClient  Retriever  Memory(010)  Ranker/Dedup  Compressor  BundleStore  EventPub  NATS
  │ Assemble(req,   │             │             │           │           │                  │           │              │             │            │           │
  │  idemKey)        │             │             │           │           │                  │           │              │             │            │           │
  ├────────────────▶│             │             │           │           │                  │           │              │             │            │           │
  │                  │ valida envelope/idem       │           │           │                  │           │              │             │            │           │
  │                  ├────────────▶│             │           │           │                  │           │              │             │            │           │
  │                  │             │ Authorize(assemble) ───▶│           │                  │           │              │             │            │           │
  │                  │             │◀- allow - - │           │           │                  │           │              │             │            │           │
  │                  ├──────────────────────────▶│ Lookup(prompt,model)  │                  │           │              │             │            │           │
  │                  │             │             │ fast path (hash): MISS│                  │           │              │             │            │           │
  │                  │             │             │ slow path (ANN): MISS │                  │           │            │             │            │           │
  │                  │◀── cache_outcome=MISS ─────│           │           │                  │           │              │             │            │           │
  │                  │                                         ┄┄▶ context.window.cache_miss (via EventPub, ver nota) │             │            │           │
  │                  ├───────────────────────────────────────▶│ AllocateAsync(req)          │           │              │             │            │           │
  │                  │             │             │           ├──────────▶│ GetModelLimitsAsync/GetTokenizerAsync      │             │            │           │
  │                  │             │             │           │◀─ limits/tokenizer ──────────│           │              │             │            │           │
  │                  │             │             │           │ resolve BudgetProfile; valida Σpesos=1.0                │             │            │           │
  │                  │◀── TokenBudget ─────────────────────────│           │                  │           │              │             │            │           │
  │                  ├─────────────────────────────────────────────────────────────────────▶│ RetrieveAsync(query,   │              │             │            │           │
  │                  │             │             │           │           │                  │  topK, deadlineMs)     │              │             │            │           │
  │                  │             │             │           │           │                  ├───────────▶│ recall seletivo         │             │            │           │
  │                  │             │             │           │           │                  │◀─ candidatos ─────────│             │            │           │
  │                  │◀── candidatos ─────────────────────────────────────────────────────────│           │              │             │            │           │
  │                  ├──────────────────────────────────────────────────────────────────────────────────▶│ RankAsync + DeduplicateAsync│             │            │           │
  │                  │             │             │           │           │                  │           │              │ ranked/deduped│           │            │           │
  │                  │◀── fragments (ranked, deduped) ────────────────────────────────────────────────────│              │             │            │           │
  │                  ├──────────────────────────────────────────────────────────────────────────────────────────────▶│ CompressAsync(fragments,│            │           │
  │                  │             │             │           │           │                  │           │              │  targetTokens)          │            │           │
  │                  │             │             │           │           │  SummarizeAsync(text, target, modelHint)  │              │             │            │           │
  │                  │             │             │           │           ├───────────────────────────────────────────▶│ (017: modelo barato)     │            │           │
  │                  │             │             │           │           │◀── sumário ────────────────────────────────│              │             │            │           │
  │                  │◀── tokens_in ≤ token_budget (ASSEMBLED) ─────────────────────────────────────────────────────────│             │            │           │
  │                  ├───────────────────────────────────────────────────────────────────────────────────────────────────────────▶│ SaveAsync(bundle,     │           │
  │                  │             │             │           │           │                  │           │              │             │  fragments) [outbox]  │           │
  │                  │             │             │           │           │                  │           │              │             ├───────────▶│ PublishAsync(      │
  │                  │             │             │           │           │                  │           │              │             │            │  context.window.assembled)│
  │                  │             │             │           │           │                  │           │              │             │            │      ┄┄┄▶│ aios.<t>.context.window.assembled
  │◀── 200 OK (ContextBundle) ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────│
```

**Notas:** o SLO deste caminho (sem chamada de sumarização) é **p99 ≤ 150 ms**
(NFR-002); com uma chamada de sumarização, **p99 ≤ 1200 ms** (NFR-003). O
evento `context.window.cache_miss` é emitido de forma independente do
`assembled` (dois eventos distintos, ver `Events.md`).

---

## 3. Fluxo 2 (Feliz) — Assemble com *Cache Hit* (Fast Path)

```
Runtime   ContextApiGateway   ContextPolicyGuard   SemanticCacheManager    EventPublisher
  │ Assemble(req, idemKey)         │                     │                       │
  ├───────────────────────────────▶│                    │                       │
  │                                ├───────────────────▶│ Authorize(assemble)   │
  │                                │◀── allow ───────────┤                       │
  │                                ├────────────────────────────────────────────▶│ Lookup(prompt,model)
  │                                │                     │  prompt_hash exato → HIT (fast path)             │
  │                                │◀── cache_outcome=HIT, entry, tokensSaved ───│                       │
  │                                ├─ persist bundle (cache_outcome=HIT) ────────────────────────────────▶│ RecordAndPublish
  │                                │                                                                       ┄┄▶ context.window.cache_hit
  │◀── 200 OK (ContextBundle, servido do cache) ────────│                       │
```

**Guarda:** `prompt_hash` idêntico ao de uma entrada `INDEXED`/`SERVING` não
invalidada. **SLO:** latência de `CacheLookup` p99 ≤ 20 ms (NFR-001) — este é
o caminho mais rápido do módulo, pois evita `RETRIEVING`/`RANKING`/
`COMPRESSING` por completo.

---

## 4. Fluxo 3 (Falha) — Orçamento Infeasível

```
Runtime   ContextApiGateway   TokenBudgeter    ModelRouterClient    EventPublisher
  │ Assemble(req)    │                │                  │                  │
  ├─────────────────▶│               │                  │                  │
  │  (policy_check + cache_lookup=MISS já concluídos, ver Fluxo 1)          │
  │                  ├──────────────▶│ AllocateAsync(req)                  │
  │                  │               ├─────────────────▶│ GetModelLimitsAsync
  │                  │               │◀── modelMaxTokens=4096 ─────────────│
  │                  │  tokens(system+instruction) = 4200 > token_budget viável │
  │                  │  guarda falha: orçamento mínimo não cabe no modelo alvo   │
  │                  ├─ RecordAndPublish(status=REJECTED,                   │
  │                  │    code=AIOS-CTX-0001) ──────────────────────────────▶│
  │                  │                                                       ┄┄▶ context.window.rejected
  │◀── 422 Unprocessable Entity ─────┤                  │                  │
  │   { code: AIOS-CTX-0001, retriable: false }          │                  │
```

**Guarda:** `tokens(system + instruction) > token_budget` mínimo viável
(FR-002). **Erro:** `AIOS-CTX-0001` (HTTP 422, `retriable=false`) — o
chamador DEVE resubmeter com um modelo de janela maior ou reduzir o
conteúdo obrigatório (`system`/`instruction`).

---

## 5. Fluxo 4 (Falha) — Policy Engine Indisponível (*Default Deny*)

```
Runtime   ContextApiGateway   ContextPolicyGuard    Policy Engine(022)   EventPublisher
  │ Assemble(req)     │                    │                   │                  │
  ├──────────────────▶│                   │                   │                  │
  │                   ├──────────────────▶│ Authorize(assemble) ─▶│               │
  │                   │                   │      (timeout, sem resposta em decision.timeout_ms)
  │                   │                   │◀── TimeoutException ─│               │
  │                   │  fail-safe: default deny (RFC-0001 §5.8)                  │
  │                   ├─ RecordAndPublish(status=REJECTED,                        │
  │                   │    code=AIOS-CTX-0011) ────────────────────────────────────▶│
  │                   │                                                             ┄┄▶ context.window.rejected
  │◀── 403 Forbidden ──┤                   │                   │                  │
  │   { code: AIOS-CTX-0011, retriable: false }                │                  │
```

**Guarda:** ausência de resposta do PDP dentro do *timeout* de decisão é
tratada como **negação** — nunca como aprovação implícita (RFC-0001 §5.8).
**Erro:** `AIOS-CTX-0011` (HTTP 403, `retriable=false`) — assim como no
Scheduler (009), este é o único ramo em que a segurança prevalece sobre a
disponibilidade, sem retry automático oferecido ao cliente.

---

## 6. Fluxo 5 (Falha) — `010-Memory` Timeout no Recall (`DEGRADED`)

```
ContextAssembler   SelectiveRetriever   MemoryClient    Memory(010)    HierarchicalCompressor   EventPublisher
      │ RetrieveAsync(query, topK,   │                 │                │                          │
      │   deadlineMs=120)              │                │                │                          │
      ├───────────────────────────────▶│               │                │                          │
      │                                ├──────────────▶│ Recall(agentUrn, query, topK) ─▶│         │
      │                                │                │      (excede 120 ms; sem resposta)         │
      │                                │◀── DeadlineExceeded ─────────────│                          │
      │  guarda: deadline excedido → estado DEGRADED (AIOS-CTX-0005)                                 │
      │◀── candidatos parciais (apenas system+history já disponíveis) ───│                          │
      ├─ prossegue para RANKING/COMPRESSING com o conjunto parcial                                   │
      ├───────────────────────────────────────────────────────────────▶│ CompressAsync(parcial)     │
      │◀── ASSEMBLED (contexto reduzido, dentro do orçamento) ──────────│                          │
      ├─ RecordAndPublish(status=SERVED, degraded=true) ────────────────────────────────────────────▶│
      │                                                                                                ┄┄▶ context.window.assembled (flag degraded)
```

**Guarda:** ausência de resposta de `010-Memory` dentro de
`context.retrieval.memory_deadline_ms` (default 120 ms). **Erro/telemetria:**
`AIOS-CTX-0005` registrado internamente; o bundle ainda é `SERVED` (FR-010 —
degradação graciosa NÃO é falha do chamador), mas com contexto reduzido
(apenas `system`/`history` disponíveis sem recall de memória).

---

## 7. Fluxo 6 (Falha) — `017-Model-Router` Indisponível para Limites/Tokenizer

```
TokenBudgeter   ModelRouterClient    ModelRouter(017)    Redis(limites cacheados)   EventPublisher
     │ GetModelLimitsAsync(modelId)   │                    │                            │
     ├───────────────────────────────▶│ (circuit breaker aberto; timeout)               │
     │                                │◀── CircuitOpenException ─│                     │
     │  fallback: consulta cache local de limites (TTL curto)                          │
     ├────────────────────────────────────────────────────────▶│ Get(modelId)          │
     │◀── limites cacheados (com timestamp) ─────────────────────│                      │
     │  limites cacheados encontrados e dentro da janela de confiança                    │
     ├─ prossegue BUDGET_ALLOCATED com limites cacheados (código AIOS-CTX-0002 registrado como aviso, não bloqueante) │
     │
     │  [ramo alternativo: SEM cache disponível]                                        │
     │  ├─ RecordAndPublish(status=REJECTED, code=AIOS-CTX-0002) ──────────────────────▶│
     │  │                                                                                 ┄┄▶ context.window.rejected
     │  └─ retorna 503 { code: AIOS-CTX-0002, retriable: true }
```

**Guarda:** `017` indisponível (circuit breaker aberto/timeout). **Estratégia
de recuperação:** usar limites/tokenizer cacheados (brief §9 FM-01); apenas
quando não há cache válido o pipeline retorna `AIOS-CTX-0002` (HTTP 503,
`retriable=true`) com *backoff* exponencial sugerido ao cliente
(100 ms → 2 s, 5 tentativas).

---

## 8. Fluxo 7 (Falha) — Falha de Sumarização com *Fallback* de Truncamento

```
ContextAssembler   HierarchicalCompressor   ModelRouterClient    ModelRouter(017)    TruncateCompressor
       │ CompressAsync(fragments, targetTokens)     │                  │                    │
       ├────────────────────────────────────────────▶│                │                    │
       │                                              ├───────────────▶│ SummarizeAsync(...) │
       │                                              │                │  erro do modelo de sumarização│
       │                                              │◀── SummarizationModelError ──────────│
       │  guarda: falha de compressão → AIOS-CTX-0004; aciona fallback determinístico          │
       │                                              ├─────────────────────────────────────▶│ CompressAsync(fragments, targetTokens)
       │                                              │                                        │  corta pelo limite de tokens (sem chamada de LLM)
       │                                              │◀── tokensAfter ≤ targetTokens ─────────│
       │◀── CompressionResult (method=TRUNCATE, degraded=true) ────────│                    │
       ├─ prossegue para ASSEMBLED (tokens_in ≤ token_budget garantido) │                    │
```

**Guarda:** erro no modelo de sumarização (`017`) durante
`HierarchicalCompressor.CompressAsync`. **Erro:** `AIOS-CTX-0010`
(`SummarizationModelError`, retriable) registrado internamente;
`AIOS-CTX-0004` (`CompressionFailed`) sinaliza a ativação do *fallback*.
**Invariante preservada:** todo bundle `SERVED` ainda respeita
`tokens_in ≤ token_budget` (SM-CTX-03), mesmo com qualidade de compressão
reduzida.

---

## 9. Fluxo 8 (Falha) — Backend de Cache Indisponível (`BYPASS`)

```
ContextAssembler   SemanticCacheManager   Redis(L1)   PostgreSQL+pgvector(L2)
       │ LookupAsync(prompt, modelId)   │                │                   │
       ├────────────────────────────────▶│               │                   │
       │                                 ├──────────────▶│ (timeout/erro de conexão)
       │                                 │◀── ConnectionException ───────────│
       │                                 ├────────────────────────────────────▶│ (tenta L2 diretamente)
       │                                 │◀── ConnectionException (L2 também indisponível) ─│
       │  guarda: ambos os níveis de cache indisponíveis → BYPASS (AIOS-CTX-0006)            │
       │◀── CacheLookupResult(outcome=BYPASS) ───────────│                   │
       ├─ prossegue pipeline normal (BUDGET_ALLOCATED → ... ) como se fosse MISS              │
       ├─ StoreAsync também é ignorado (best-effort) até que o backend se recupere            │
```

**Guarda:** indisponibilidade de Redis (L1) e/ou `pgvector` (L2). **Erro:**
`AIOS-CTX-0006` (HTTP 503, `retriable=true`) registrado na telemetria; o
pipeline **NÃO** é bloqueado — segue como se fosse `MISS`, garantindo que a
indisponibilidade do cache degrade desempenho (sem economia de tokens), mas
nunca disponibilidade (FR-010, brief §9 FM-03).

---

## 10. Fluxo 9 (Feliz) — Idempotência: Retry de `Assemble`

**Cenário:** o cliente (`Agent Runtime`) não recebe a resposta do primeiro
`Assemble` (timeout de rede) e reenvia a mesma requisição com o mesmo
`Idempotency-Key`.

```
Runtime   ContextApiGateway   IdempotencyStore    ContextAssembler
  │ Assemble(req, idemKey=K)  │                    │
  ├──────────────────────────▶│                   │
  │                           ├──TryBegin(K,hash)▶│ (Redis, não existe ainda)
  │                           │◀── lease adquirida │
  │                           ├──────────────────────────────────▶│ (segue Fluxo 1 normalmente)
  │                           │◀── ContextBundle(B1, SERVED) ─────│
  │                           ├──Complete(K, response)──▶│ (grava snapshot, TTL≥24h)
  │◀── 200 OK (bundleId=B1) │                    │
  │                           │                    │
  │  ... timeout de rede no cliente antes de ler a resposta ...
  │                           │                    │
  │ Assemble(req, idemKey=K) [retry idêntico]      │
  ├──────────────────────────▶│                   │
  │                           ├──TryBegin(K,hash)▶│ (Redis: já existe resultado)
  │                           │◀── snapshot(B1) ───│  (não reexecuta o pipeline)
  │◀── 200 OK (bundleId=B1) [mesmo resultado] │
```

**Invariante (FR-011):** nenhuma segunda montagem/persistência é criada; a
resposta é idêntica à primeira. Se o `requestHash` do retry divergir do
original para a mesma `idemKey`, o Context retorna `AIOS-CTX-0012` (HTTP 409,
conflito de idempotência) em vez do *snapshot*.

---

## 11. Fluxo 10 (Feliz) — Invalidação de Cache por Evento Externo

**Participantes:** Memory(010) (produtor), NATS/JetStream, EvictionManager,
SemanticCacheManager, EventPublisher.

```
Memory(010)   NATS/JetStream   EvictionManager    SemanticCacheManager   EventPublisher
    │ (item consolidado altera conteúdo referenciado em cache)          │
    ├─ publish ┄┄▶│ aios.<tenant>.memory.item.consolidated             │
    │              │┄┄consume┄┄▶│                                       │
    │              │             │ OnSourceChangedAsync(sourceUrn)      │
    │              │             ├──────────────────────────────────▶│ localiza entradas com scope_ref/proveniência = sourceUrn
    │              │             │                                     │ marca INVALIDATED
    │              │             │◀── entradas invalidadas (n) ────────│
    │              │             ├─ EvictAsync(entradas) ─────────────▶│ remove de L1+L2 (INVALIDATED→EVICTED)
    │              │             ├─ RecordAndPublish ──────────────────────────────────────────────────▶│
    │              │             │                                                                        ┄┄▶ aios.<tenant>.context.cache.invalidated
    │              │             │                                                                        ┄┄▶ aios.<tenant>.context.cache.evicted
```

**Guarda:** consumo do evento `aios.<tenant>.memory.item.consolidated` (ou
`memory.item.deleted`, `model.registry.updated`, `knowledge.node.updated`).
**SLO:** invalidação DEVE completar em ≤ 5 s após a publicação do evento
(FR-008) — medido fim-a-fim do `publish` externo ao `evicted` interno.

---

## 12. Fluxo 11 (Falha) — Tenant Divergente

```
Runtime   ContextApiGateway     EventPublisher
  │ Assemble(req)  [X-AIOS-Tenant: "acme", token.tenant: "globex"]      │
  ├───────────────▶│                    │
  │                │ valida: token.tenant != X-AIOS-Tenant              │
  │                │ guarda falha: isolamento de tenant violado         │
  │                ├─ RecordAndPublish(status=REJECTED,                 │
  │                │    code=AIOS-CTX-0011) ─────────────────────────▶│
  │                │                                                    ┄┄▶ context.window.rejected
  │◀── 403 Forbidden ─┤                    │
  │   { code: AIOS-CTX-0011, retriable: false } │
```

**Guarda:** `tenant` do token autenticado DEVE igualar `X-AIOS-Tenant`
(RFC-0001 §6). **Erro:** `AIOS-CTX-0011` (HTTP 403, `retriable=false`) —
nenhuma consulta a `010`/`017`/cache é sequer iniciada; a rejeição ocorre
antes de `POLICY_CHECK` (fail-fast de isolamento multi-tenant).

---

## 13. Fluxo 12 (Feliz) — Expurgo LGPD por `source_urn`

**Participantes:** Security/Compliance (021, via API administrativa),
ContextApiGateway, ContextPolicyGuard, BundleStore, SemanticCacheManager,
EventPublisher, Audit(025).

```
Compliance(021)   ContextApiGateway   PolicyGuard   BundleStore   CacheManager   EventPublisher   Audit(025)
     │ POST /v1/context/cache/invalidate (scope, sourceUrn, idemKey)    │              │               │
     ├─────────────────────────────────▶│               │              │               │
     │                                   ├──────────────▶│ Authorize(invalidate_bulk)   │               │
     │                                   │◀── allow ──────┤              │               │
     │                                   ├──────────────────────────────▶│ purga bundles/fragments      │
     │                                   │                │              │  referenciando sourceUrn      │
     │                                   ├───────────────────────────────────────────▶│ InvalidateByScopeAsync(sourceUrn)
     │                                   │                │              │◀── n entradas invalidadas ────│
     │                                   ├─ RecordAndPublish(purge_completed) ─────────────────────────▶│
     │                                   │                                                                 ┄┄▶ context.cache.evicted (cause=purge)
     │                                   │                                                                              ┄┄▶ trilha de auditoria (025)
     │◀── 200 OK (purged: bundles=n1, fragments=n2, cache=n3) ─────────│              │               │
```

**Guarda:** operação privilegiada — DEVE ser autorizada pelo PDP
(`022`) como `invalidate_bulk`. **Rastreabilidade (FR-012):** todo expurgo
emite `context.cache.evicted` e um registro de auditoria imutável em `025`,
nunca uma remoção silenciosa — requisito de "direito ao esquecimento"
propagado a partir de `memory.item.deleted`.

---

## 14. Timeouts e SLOs por Fluxo

| Fluxo | Timeout/SLO aplicável | Comportamento no timeout |
|-------|--------------------------|------------------------------|
| Fluxo 1 (Assemble, *cache miss*, sem sumarização) | p99 ≤ 150 ms (NFR-002). | Excedente aciona degradação por dependência (Fluxos 5/6/7). |
| Fluxo 1 (Assemble, com 1 chamada de sumarização) | p99 ≤ 1200 ms (NFR-003). | Falha de sumarização aciona Fluxo 7 (*fallback* truncamento). |
| Fluxo 2 (Assemble, *cache hit*) | `CacheLookup` p99 ≤ 20 ms (NFR-001). | — |
| Fluxo 4 (PDP indisponível) | `context.policy.decision_timeout_ms`. | *Default deny* — `AIOS-CTX-0011`. |
| Fluxo 5 (Recall timeout) | `context.retrieval.memory_deadline_ms` (120 ms). | `DEGRADED`; contexto parcial servido; `AIOS-CTX-0005`. |
| Fluxo 6 (Model Router indisponível) | *circuit breaker* + *backoff* exp. (100 ms→2 s, 5x). | Limites cacheados; `AIOS-CTX-0002` se sem cache. |
| Fluxo 7 (Falha de sumarização) | Timeout do provedor de sumarização (`017`). | *Fallback* `TruncateCompressor`; `AIOS-CTX-0004`. |
| Fluxo 8 (Cache indisponível) | *health probe* de Redis/`pgvector`. | `BYPASS`; `AIOS-CTX-0006`; segue como `MISS`. |
| Fluxo 9 (Idempotência) | `context.idempotency.ttl_hours` (24h, mínimo RFC-0001 §5.5). | Resultado repetido sem nova efetivação. |
| Fluxo 10 (Invalidação por evento) | ≤ 5 s fim-a-fim (FR-008). | Excedente é tratado como incidente de observabilidade (`Monitoring.md`). |
| Fluxo 12 (Expurgo LGPD) | Sem SLO de latência estrito; DEVE ser rastreável e completo. | Falha parcial é reexecutável (idempotente por `Idempotency-Key`). |
