---
Documento: Configuration
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0116, ADR-0117
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 001-Architecture, 005-Database, 017-Model-Router, 022-Policy, 024-Observability, 027-Cluster, 040-Glossary
---

# AIOS — Módulo 011 · Context — Configuration

> Este documento **deriva integralmente** da **§8 (Chaves de Configuração
> Principais)** do [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) e **NÃO PODE
> contradizê-lo**. Toda chave, *default*, faixa válida, escopo e
> recarregabilidade listados aqui são **idênticos** aos do brief; qualquer
> divergência é defeito deste documento, não do brief. Contratos centrais
> (correlação, identidade, envelope de erro) vêm de
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) e **não são
> redefinidos** aqui. Palavras normativas conforme RFC 2119/8174.

---

## 1. Convenções

| Aspecto | Regra |
|---------|-------|
| Namespace de chave | Toda chave de domínio usa o formato pontuado `context.<grupo>.<nome>` (ex.: `context.cache.similarity_threshold`). |
| Variável de ambiente | Cada chave é espelhada como `AIOS_CONTEXT_<GRUPO>_<NOME>` (maiúsculas, `.`→`_`), lida por `IOptionsMonitor<ContextOptions>` (.NET 10). Exceção documentada: `context.embedding.dimensions` mapeia para `AIOS_CONTEXT_EMBEDDING_DIM` (forma abreviada já fixada em [Deployment.md](./Deployment.md) §5). |
| Escopo | `global` (única instância válida no processo), `tenant` (override por `tenant_id`), `agent` (override por `agent_urn`), `task_type` (override por rótulo de tipo de tarefa). Onde múltiplos escopos aparecem (ex. `tenant/agent`), a resolução segue a **precedência de especificidade** (§3). |
| Recarregável | **sim** = pode mudar em runtime sem reiniciar o processo, propagando por um dos dois mecanismos do §4; **não** = exige *rollout* dedicado (nunca no mesmo passo de um deploy de código, [Deployment.md](./Deployment.md) §6). |
| Fonte de verdade | A tabela da §2 é a **cópia literal** da §8 do brief; nenhuma chave nova é inventada sem primeiro atualizar o brief — a única exceção documentada é `context.telemetry.cache_miss_sample_rate` (§2.1), já referenciada por [Events.md](./Events.md) §7 mas ausente da tabela original do brief (gap sinalizado no relatório final do agente que escreveu este documento). |
| Validação | Toda chave é validada na leitura/escrita (tipo, faixa, enum); violação retorna `AIOS-CTX-0008` (`InvalidBudgetProfile`) quando a chave pertence a um `BudgetProfile`, ou é rejeitada no *bootstrap*/`PUT` de configuração equivalente para as demais (RFC-0001 §7, envelope RFC 7807). |
| Segredos | Nenhum segredo (DSN, credenciais, chaves de API) usa o *namespace* `context.*` ou variáveis `AIOS_CONTEXT_*` de domínio — segredos são injetados por `secretRef` do cofre (021), nunca em texto claro nem registrados em log (ver [Security.md](./Security.md) §3, [Logging.md](./Logging.md) §5). |

---

## 2. Tabela canônica de chaves (idêntica ao brief §8)

| # | Chave | Env var | Tipo | Default | Faixa/Enum | Escopo | Recarregável |
|---|-------|---------|------|---------|------------|--------|:---:|
| 1 | `context.budget.reserved_output_ratio` | `AIOS_CONTEXT_BUDGET_RESERVED_OUTPUT_RATIO` | float | `0.25` | `0.05–0.60` | tenant/agent | sim |
| 2 | `context.budget.min_compression_ratio` | `AIOS_CONTEXT_BUDGET_MIN_COMPRESSION_RATIO` | float | `0.30` | `0.0–0.90` | tenant | sim |
| 3 | `context.cache.enabled` | `AIOS_CONTEXT_CACHE_ENABLED` | bool | `true` | `true\|false` | tenant/agent | sim |
| 4 | `context.cache.similarity_threshold` | `AIOS_CONTEXT_CACHE_SIMILARITY_THRESHOLD` | float | `0.92` | `0.80–0.99` | tenant/task_type | sim |
| 5 | `context.cache.reuse_threshold` | `AIOS_CONTEXT_CACHE_REUSE_THRESHOLD` | float | `0.95` | `0.85–0.99` | tenant | sim |
| 6 | `context.cache.default_ttl_seconds` | `AIOS_CONTEXT_CACHE_DEFAULT_TTL_SECONDS` | int | `3600` | `60–604800` | tenant | sim |
| 7 | `context.cache.max_entries_per_tenant` | `AIOS_CONTEXT_CACHE_MAX_ENTRIES_PER_TENANT` | int | `1_000_000` | `10³–10⁸` | tenant | sim |
| 8 | `context.cache.l1_redis_ttl_seconds` | `AIOS_CONTEXT_CACHE_L1_REDIS_TTL_SECONDS` | int | `300` | `30–3600` | global | sim |
| 9 | `context.cache.vector_index` | `AIOS_CONTEXT_CACHE_VECTOR_INDEX` | enum | `hnsw` | `hnsw\|ivfflat` | global | **não** |
| 10 | `context.compression.method` | `AIOS_CONTEXT_COMPRESSION_METHOD` | enum | `hierarchical` | `hierarchical\|extractive\|truncate` | tenant/task_type | sim |
| 11 | `context.compression.max_levels` | `AIOS_CONTEXT_COMPRESSION_MAX_LEVELS` | int | `3` | `1–5` | tenant | sim |
| 12 | `context.compression.summarization_model_hint` | `AIOS_CONTEXT_COMPRESSION_SUMMARIZATION_MODEL_HINT` | string | `cheap-small` | rótulo `017` | tenant | sim |
| 13 | `context.retrieval.top_k` | `AIOS_CONTEXT_RETRIEVAL_TOP_K` | int | `32` | `1–256` | tenant/task_type | sim |
| 14 | `context.retrieval.memory_deadline_ms` | `AIOS_CONTEXT_RETRIEVAL_MEMORY_DEADLINE_MS` | int | `120` | `20–1000` | tenant | sim |
| 15 | `context.dedup.cosine_threshold` | `AIOS_CONTEXT_DEDUP_COSINE_THRESHOLD` | float | `0.95` | `0.80–0.999` | tenant | sim |
| 16 | `context.embedding.model_hint` | `AIOS_CONTEXT_EMBEDDING_MODEL_HINT` | string | `default-embed` | rótulo `017` | tenant | sim |
| 17 | `context.embedding.dimensions` | `AIOS_CONTEXT_EMBEDDING_DIM` | int | `1536` | `256–4096` | global | **não** |
| 18 | `context.limits.max_inline_bytes` | `AIOS_CONTEXT_LIMITS_MAX_INLINE_BYTES` | int | `16384` | `1024–262144` | global | sim |
| 19 | `context.limits.max_payload_bytes` | `AIOS_CONTEXT_LIMITS_MAX_PAYLOAD_BYTES` | int | `4_194_304` | `65536–33554432` | tenant | sim |
| 20 | `context.ratelimit.assemble_rps_per_tenant` | `AIOS_CONTEXT_RATELIMIT_ASSEMBLE_RPS_PER_TENANT` | int | `1000` | `1–100000` | tenant | sim |
| 21 | `context.degraded.allow_truncate_fallback` | `AIOS_CONTEXT_DEGRADED_ALLOW_TRUNCATE_FALLBACK` | bool | `true` | `true\|false` | tenant | sim |
| 22 | `context.shard.count` | `AIOS_CONTEXT_SHARD_COUNT` | int | `64` | `1–4096` | global | **não** |

### 2.1 Chave adicional referenciada por documentos irmãos (gap do brief)

| # | Chave | Env var | Tipo | Default | Faixa | Escopo | Recarregável |
|---|-------|---------|------|---------|-------|--------|:---:|
| 23 | `context.telemetry.cache_miss_sample_rate` | `AIOS_CONTEXT_TELEMETRY_CACHE_MISS_SAMPLE_RATE` | float | `1.0` | `0.0–1.0` | global | sim |

> **Nota de rastreabilidade.** [Events.md](./Events.md) §7 já referencia
> `context.telemetry.*` como o mecanismo de amostragem de
> `context.window.cache_miss` sob alta carga ("Volume alto de
> `window.cache_miss` sob baixa taxa de acerto"). O `_DESIGN_BRIEF.md` §8 não
> lista essa chave explicitamente. Para não contradizer um documento já
> escrito, esta chave é adicionada aqui com um *default* conservador
> (`1.0` = sem amostragem, emite 100% dos eventos) e faixa segura. **Esta
> chave DEVE ser incorporada ao brief §8 na próxima revisão** — sinalizado no
> relatório final deste agente.

---

## 3. Resolução de escopo e precedência

Para chaves com mais de um escopo válido, a resolução segue a ordem de
especificidade **mais específico vence**, idêntica à regra já fixada por
[FunctionalRequirements.md](./FunctionalRequirements.md) (FR-002) para
`BudgetProfile`:

```
agent  >  task_type  >  tenant  >  global (default de fábrica)
```

```
resolve(chave, tenant_id, agent_urn?, task_type?):
    se existir override registrado para (tenant_id, agent_urn)  → usa-o
    senão se existir override para (tenant_id, task_type)       → usa-o
    senão se existir override para (tenant_id)                  → usa-o
    senão                                                        → usa o default global (§2)
```

- Chaves com escopo único `global` (`context.cache.vector_index`,
  `context.embedding.dimensions`, `context.shard.count`,
  `context.cache.l1_redis_ttl_seconds`, `context.limits.max_inline_bytes`,
  `context.telemetry.cache_miss_sample_rate`) **NÃO POSSUEM** override por
  tenant/agent — qualquer tentativa de `PUT` de override nesse escopo **DEVE**
  ser rejeitada (`AIOS-CTX-0014`, `ValidationError`).
- As cinco chaves de `alloc_*` do `BudgetProfile` (brief §3.4) não aparecem
  nesta tabela porque são **campos de entidade**, não chaves soltas — são
  resolvidas pelo mesmo algoritmo de precedência via
  `PUT /v1/context/budgets/{scope}` ([API.md](./API.md) §5.1), com a
  invariante `Σ alloc_* = 1.0` validada na escrita.

---

## 4. Mecanismos de configuração e *reload*

O `ContextService` mantém **duas camadas** de configuração, coerentes com o
padrão *stateless* do serviço ([Scalability.md](./Scalability.md) §2):

```
┌──────────────────────────────── Camada 1 — Base (por processo) ────────────────────────────────┐
│  ConfigMap/Secret montado + variáveis AIOS_CONTEXT_*  ──▶  IOptionsMonitor<ContextOptions>       │
│  Aplica-se a chaves "global" e a defaults de fábrica de todas as demais.                          │
│  Chaves NÃO recarregáveis (vector_index, embedding.dimensions, shard.count) só mudam com          │
│  rollout dedicado + migração/reindex (Deployment.md §6).                                          │
└─────────────────────────────────────────┬─────────────────────────────────────────────────────────┘
                                            │ fallback quando não há override de escopo mais específico
┌───────────────────────────────────────────▼─────────────────────────────────────────────────────────┐
│  Camada 2 — Overrides de escopo (tenant/agent/task_type)                                             │
│  Fonte durável: PostgreSQL (`BudgetProfile` para chaves de orçamento; demais chaves "sim" em          │
│  uma tabela de overrides escopados equivalente, sob RLS por tenant_id).                               │
│  Cache local (in-process, TTL curto) por réplica; invalidado por evento, NUNCA por poll agressivo:    │
│    - aios.<tenant>.context.budget.updated        (BudgetProfile alterado)                             │
│    - aios.<tenant>.policy.decision.updated        (limites de política aplicáveis mudaram)            │
└────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Regras normativas:

1. Chaves marcadas **sim** DEVEM refletir uma alteração de override
   (`PUT /v1/context/budgets/{scope}` ou operação equivalente de config
   escopada) em, no máximo, o TTL do cache local de configuração (recomendado
   ≤ 30 s) — não é necessário reiniciar réplicas.
2. Chaves marcadas **não** só mudam através de um novo *rollout* de imagem/
   `ConfigMap`, nunca por escrita em runtime; o serviço **DEVE** rejeitar
   qualquer tentativa de override de escopo para essas chaves
   (`AIOS-CTX-0014`).
3. Toda alteração de override que afete um `BudgetProfile` (chaves 1–2 da
   §2) publica `aios.<tenant>.context.budget.updated` via **outbox**
   (RFC-0001 §5.2/§5.3), disparando invalidação do cache local em todas as
   réplicas.
4. Em caso de indisponibilidade da fonte durável (PostgreSQL) para resolver
   um override de escopo, o `ContextService` **DEVE** usar o último valor
   cacheado válido e, na ausência de cache, cair para o *default* global —
   nunca falhar a operação de `assemble` por causa de configuração ausente
   (alinhado a FM-01/P5 de [FailureRecovery.md](./FailureRecovery.md)).

---

## 5. Detalhamento por grupo

### 5.1 `context.budget.*` — orçamento e reserva de saída
Controla a fração do `model_max_tokens` reservada para geração
(`reserved_output_ratio`) e o piso mínimo aceitável de
**Context Compression Ratio** (`min_compression_ratio`). Consumido pelo
`TokenBudgeter` (FR-002); a meta de compressão em regime permanente
(NFR-004 ≥ 0,50) é **maior** que o piso configurável — o piso é uma rede de
segurança contra compressão insuficiente, não a meta operacional.

### 5.2 `context.cache.*` — cache semântico (L1 Redis + L2 pgvector)
Governa o comportamento do `SemanticCacheManager`: liga/desliga por
escopo (`enabled`), limiares de aceitação de *hit* (`similarity_threshold`)
e de reuso de `SummaryNode` (`reuse_threshold`), TTLs (`default_ttl_seconds`,
`l1_redis_ttl_seconds`), cota por tenant (`max_entries_per_tenant`, base do
alerta `ContextCacheEntriesNearQuota` em [Monitoring.md](./Monitoring.md)) e o
tipo de índice ANN (`vector_index`, **não recarregável** — mudar exige
reindex, ADR-0111/ADR-0112).

### 5.3 `context.compression.*` — compressão hierárquica
Governa o `HierarchicalCompressor` (FR-003/FR-014): método ativo
(`method`), profundidade máxima da árvore *map-reduce*
(`max_levels`) e o rótulo de modelo barato de sumarização
(`summarization_model_hint`, resolvido pelo `017-Model-Router`, ADR-0115).

### 5.4 `context.retrieval.*` e `context.dedup.*` — recall e deduplicação
`context.retrieval.top_k` e `context.retrieval.memory_deadline_ms` limitam o
`SelectiveRetriever`/`MemoryClient` (FR-005); `context.dedup.cosine_threshold`
governa o `RedundancyEliminator` (FR-004).

### 5.5 `context.embedding.*` — modelo e dimensionalidade de embedding
`model_hint` é recarregável (troca de rótulo de modelo via `017`);
`dimensions` **NÃO É** recarregável porque determina a forma física da coluna
`vector(N)` em `ContextFragment`/`SemanticCacheEntry`/`SummaryNode`
([Database.md](./Database.md)) — mudança exige migração com reindex completo.

### 5.6 `context.limits.*` e `context.ratelimit.*` — proteção do caminho quente
`max_inline_bytes` decide o corte inline vs. MinIO (`BundleStore`, FR-013,
ADR-0117); `max_payload_bytes` protege o `ContextApiGateway` contra
requisições excessivas (`AIOS-CTX-0013`); `assemble_rps_per_tenant` aplica
*backpressure* por tenant (`AIOS-CTX-0015`, [Scalability.md](./Scalability.md) §6).

### 5.7 `context.degraded.*` — degradação graciosa
`allow_truncate_fallback` habilita o truncamento determinístico quando `010`/
`017` estão indisponíveis (FR-010, FM-01/FM-02/FM-04). **DEVERIA** permanecer
`true` em produção — desligá-lo transforma indisponibilidade parcial em falha
total do `assemble`, contrariando o princípio P5 de
[FailureRecovery.md](./FailureRecovery.md).

### 5.8 `context.shard.count` — particionamento
Determina o número de partições lógicas (`hash(tenant_id, agent_id) mod N`)
usadas por Redis L1 e pelo roteamento do Gateway ([Scalability.md](./Scalability.md)
§3). **NÃO recarregável**; aumentar `N` exige migração online por tenant
(dupla-escrita + *cutover*), nunca uma simples alteração de valor.

### 5.9 `context.telemetry.*` — amostragem de telemetria
`cache_miss_sample_rate` controla a fração de eventos
`context.window.cache_miss` efetivamente publicados sob alta carga (ver
§2.1); métricas Prometheus (`aios_context_cache_miss_total`) **NÃO SÃO**
amostradas — apenas o evento de domínio, que é *best-effort* por definição
([Events.md](./Events.md) §3).

---

## 6. Exemplo — configuração de processo e override de escopo

### 6.1 Variáveis de ambiente (camada base, `docker-compose`/manifesto)

```yaml
environment:
  AIOS_CONTEXT_EMBEDDING_DIM: "1536"
  AIOS_CONTEXT_CACHE_VECTOR_INDEX: "hnsw"
  AIOS_CONTEXT_RETRIEVAL_TOP_K: "32"
  AIOS_CONTEXT_SHARD_COUNT: "64"
  AIOS_CONTEXT_RATELIMIT_ASSEMBLE_RPS_PER_TENANT: "1000"
  AIOS_CONTEXT_CACHE_SIMILARITY_THRESHOLD: "0.92"
  AIOS_CONTEXT_CACHE_DEFAULT_TTL_SECONDS: "3600"
  AIOS_CONTEXT_DEGRADED_ALLOW_TRUNCATE_FALLBACK: "true"
  AIOS_CONTEXT_TELEMETRY_CACHE_MISS_SAMPLE_RATE: "1.0"
```

### 6.2 Override de escopo em runtime (chave recarregável, via `BudgetProfile`)

```
PUT /v1/context/budgets/tenant
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZD1Q9S7U3W5Y7A9C1E3G5J
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{
  "scope": "tenant",
  "reserved_output_ratio": 0.30,
  "alloc_system": 0.10,
  "alloc_instruction": 0.15,
  "alloc_memory": 0.40,
  "alloc_history": 0.25,
  "alloc_tools": 0.10,
  "min_compression_ratio": 0.35
}

→ 200 OK  { "profile_id": "01J9ZD1R...", "updated_at": "2026-07-21T10:00:00Z" }
# Efeito: publica aios.acme.context.budget.updated (outbox); réplicas invalidam
# cache local de configuração em ≤ 30 s (§4). Payload com Σ alloc_* ≠ 1.0
# retorna 422 AIOS-CTX-0008 (InvalidBudgetProfile).
```

### 6.3 Tentativa inválida — override de chave `global` (rejeitada)

```
PUT /v1/context/budgets/tenant   (tentando alterar vector_index por tenant)
→ 400 AIOS-CTX-0014 ValidationError
  "detail": "context.cache.vector_index tem escopo global; não aceita override por tenant."
```

---

## 7. Segurança e auditoria de configuração

- Toda alteração de override escopado (§4) **DEVE** passar pelo
  `ContextPolicyGuard`/PDP (`022-Policy`, operação privilegiada — brief §12.1)
  e emitir registro em [025-Audit](../025-Audit/Events.md).
- Segredos **NÃO DEVEM** trafegar por este mecanismo — ver [Security.md](./Security.md)
  §3 para o inventário de segredos e sua gestão via cofre (`021`).
- Nenhuma chave de configuração de domínio (`context.*`) contém ou referencia
  PII; valores de configuração são seguros para aparecer em logs/telemetria
  (ao contrário de segredos), consistente com [Logging.md](./Logging.md) §5.

---

## 8. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-------------------|--------------------------|
| Chave `context.telemetry.cache_miss_sample_rate` não estava no brief §8 original. | Adicionada aqui com *default* conservador (§2.1); sinalizada para incorporação formal ao brief — não decidida "à revelia" (mantém o brief como fonte única de verdade após a próxima revisão). |
| Overrides de escopo desatualizados por até o TTL do cache local (~30 s). | Invalidação *event-driven* (`context.budget.updated`) reduz a janela na prática para o tempo de propagação de evento (tipicamente < 1 s); TTL é apenas rede de segurança contra perda de evento. |
| Mudança acidental de chave **não recarregável** via override de escopo. | Validação explícita rejeita com `AIOS-CTX-0014`; chaves globais imutáveis nunca aparecem no schema de `PUT /v1/context/budgets/{scope}`. |
| Drift entre defaults documentados aqui e os efetivamente compilados no serviço. | Este documento é gerado a partir do brief §8 (fonte única); testes de contrato de configuração (`./Testing.md`) comparam o schema de opções em runtime contra esta tabela. |
| **Alternativa descartada:** um único arquivo de configuração estático sem overrides por tenant. | Rejeitada — viola a necessidade de calibrar `similarity_threshold`/`BudgetProfile` por tenant/task_type observada em NFR-004/005/006/011 (compressão e cache não podem ser "tamanho único"). |
| **Alternativa descartada:** *hot reload* por *polling* agressivo (ex.: a cada 1 s) em vez de evento. | Rejeitada — gera carga desnecessária no PostgreSQL a cada réplica; o padrão *event-driven* já é usado por todo o módulo (Outbox, §4 do brief) e é reaproveitado aqui. |

---

## 9. Ver também

- [`_DESIGN_BRIEF.md` §8](./_DESIGN_BRIEF.md) — fonte única de verdade das
  chaves.
- [FunctionalRequirements.md](./FunctionalRequirements.md) — FR-002 (precedência
  de `BudgetProfile`).
- [Database.md](./Database.md) — schema físico de `BudgetProfile` e demais
  entidades.
- [Deployment.md](./Deployment.md) — variáveis de ambiente na topologia de
  implantação; regras de rollout para chaves não recarregáveis.
- [Scalability.md](./Scalability.md) — `context.shard.*`, `context.cache.*`,
  `context.ratelimit.*` em contexto de escala.
- [Security.md](./Security.md) — gestão de segredos (fora do namespace `context.*`).
- [Events.md](./Events.md) — referência original a `context.telemetry.*`.
- [Testing.md](./Testing.md) — testes de contrato de configuração.
- Glossário: [Idempotency, Row-Level Security, Outbox](../040-Glossary/Glossary.md).
