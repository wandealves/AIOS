---
Documento: Failure Recovery (FMEA e Estratégia de Recuperação)
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0114, ADR-0115, ADR-0116, ADR-0118
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 001-Architecture, 005-Database, 010-Memory, 017-Model-Router, 020-Communication, 024-Observability, 025-Audit, 027-Cluster, 040-Glossary
---

# AIOS — Módulo 011 · Context — Failure Recovery

> Este documento deriva da **§9 (Modos de Falha e Estratégia de Recuperação)**
> do `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**. Todos os códigos de erro
> (`AIOS-CTX-NNNN`), metas de RTO/RPO e princípios de idempotência reutilizam
> os contratos centrais da
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) (§5.4 erros, §5.5
> idempotência) sem redefini-los. Palavras normativas conforme RFC 2119/8174.

---

## 1. Objetivo e escopo

Este documento especifica a **Análise de Modos de Falha e Efeitos (FMEA)** do
`ContextService` (011), com detecção, isolamento, recuperação, idempotência,
política de *retries*/*backoff*, uso de *Dead-Letter Queue* (DLQ), metas de
**RTO/RPO** e as regras de **degradação graciosa** (truncamento seguro).

O `ContextService` é o gerenciador de janela de contexto do AIOS (análogo
cognitivo ao subsistema de cache/paginação de um SO clássico). Sua resiliência
segue um princípio central do brief §1.3 e §9: **é preferível entregar um
contexto viável e degradado a falhar a montagem inteira** — desde que a
degradação seja sinalizada e nunca mascarada. Camadas duráveis
(`ContextBundle`/`ContextFragment`/`SemanticCacheEntry` L2/`BudgetProfile`) têm
RPO ≤ 60 s; o cache L1 (Redis) é **hot state** e tolera perda total sem
inconsistência (reconstrói-se por *cache-fill*). O documento é normativo para
operação e para os *runbooks* referenciados em
[Monitoring.md](./Monitoring.md).

---

## 2. Princípios de recuperação (invariantes)

Estes princípios governam TODA a matriz de falhas da §5.

1. **P1 — Idempotência universal.** Toda mutação (`assemble`, `cache store`,
   `budget upsert`) DEVE ser idempotente por `Idempotency-Key` (RFC-0001 §5.5).
   Repetições retornam o resultado memoizado (Redis, ≥ 24h), nunca reaplicam o
   efeito; divergência de payload sob a mesma chave retorna `AIOS-CTX-0012`
   (409, `IdempotencyConflict`).
2. **P2 — Retry com backoff exponencial + jitter.** Falhas transitórias
   (`retriable = true`) DEVEM ser repetidas com backoff exponencial e *jitter*
   para evitar *thundering herd*. Falhas `retriable = false` NÃO DEVEM ser
   repetidas — falham rápido e retornam ao chamador.
3. **P3 — Outbox garante atomicidade DB+evento.** Todo evento de domínio
   (`context.window.*`, `context.cache.*`, `context.budget.*`) é gravado na
   mesma transação da mutação; o `EventPublisher` reenvia *at-least-once* a
   partir do outbox. RPO de evento ≤ 60 s (FM-07).
4. **P4 — Consumidores deduplicam por `event.id`.** Entrega *at-least-once*
   implica que o `ContextService`, ao consumir eventos de `010`/`017`/`019`/
   `006`/`022` (brief §6.2), DEVE deduplicar (RFC-0001 §5.2).
5. **P5 — Degradação graciosa antes de falha total.** A indisponibilidade de
   uma dependência (`010`, `017`, Redis) NÃO DEVE impedir a entrega de um
   contexto viável quando `context.degraded.allow_truncate_fallback=true`
   (estado `DEGRADED`, brief §4.1).
6. **P6 — Compressão nunca viola o orçamento.** Falha de sumarização recorre a
   truncamento determinístico (*fallback*), mas o bundle final DEVE respeitar
   `tokens_in ≤ token_budget` (FR-001) — nunca é entregue um bundle acima do
   orçamento por causa de uma falha.
7. **P7 — Isolamento de falha por tenant/shard.** Uma falha em um shard,
   tenant ou dependência de um tenant NÃO DEVE se propagar a outros (padrão
   *bulkhead*, ver [Scalability.md](./Scalability.md)).
8. **P8 — Cache é sempre um atalho, nunca a única fonte de verdade.** Uma
   falha do cache (L1 ou L2) DEVE degradar para o caminho de montagem
   completo (`MISS`/`BYPASS`) — nunca bloqueia `assemble` (FM-03).

---

## 3. Detecção e classificação de falhas

### 3.1 Sinais de detecção

| Mecanismo | O que detecta | Origem |
|-----------|-----------------|--------|
| Health/readiness probes | Backend caído (Redis/PostgreSQL/MinIO/`017`/`010`) | [Deployment.md](./Deployment.md) |
| Timeouts por dependência | Latência acima do orçamento (`context.retrieval.memory_deadline_ms`, chamadas ao `017`) | `MemoryClient`, `ModelRouterClient` |
| Circuit breaker | Sequência de falhas a um dependente (abre o circuito) | `ModelRouterClient` (`017`), `MemoryClient` (`010`) |
| Contadores de rate-limit | Estouro de `context.ratelimit.assemble_rps_per_tenant` | `ContextApiGateway` |
| Validação de orçamento | `tokens(system+instrução) > token_budget` | `TokenBudgeter` |
| Outbox `status = pending` antigo | Evento não publicado após crash | `EventPublisher` |
| Contador de re-entregas JetStream | *Poison message* (N falhas de consumo) | consumidores de eventos (§6.2 do brief) |
| Auditoria de `similarity`/falso-hit | Taxa de falso-hit acima do limiar (amostragem/eval offline) | NFR-011, `ADR-0119` |

### 3.2 Taxonomia de erros (ligada ao catálogo §5.3 do brief)

| Classe | Códigos | Retriable | Estratégia base |
|--------|---------|-----------|-------------------|
| Cliente (contrato) | `AIOS-CTX-0007`, `0008`, `0013`, `0014` | não | Rejeitar; corrigir requisição. |
| Autorização/tenant | `AIOS-CTX-0011` | não | *Default deny*; auditar (025). |
| Idempotência | `AIOS-CTX-0012` | não | Retornar memoizado / recusar conflito de payload. |
| Orçamento infeasível | `AIOS-CTX-0001` | não | Rejeitar (`REJECTED`); sugerir modelo maior ao `017`. |
| Rate-limit | `AIOS-CTX-0015` | sim (429) | Repetir com backoff; respeitar `Retry-After`. |
| Dependência externa (`017`) | `AIOS-CTX-0002`, `0003`, `0009`, `0010` | sim | Degradar + *retry* + circuit breaker. |
| Dependência externa (`010`) | `AIOS-CTX-0005` | sim | `DEGRADED`: contexto parcial (system+history). |
| Cache backend | `AIOS-CTX-0006` | sim | *Bypass* de cache; segue caminho `MISS`. |
| Compressão | `AIOS-CTX-0004` | sim | *Fallback* de truncamento determinístico. |

---

## 4. Política de retries, backoff e circuit breaking

### 4.1 Parâmetros normativos

| Parâmetro | Valor recomendado | Aplica-se a |
|-----------|---------------------|---------------|
| Tentativas máximas (dependência externa) | 5 | `ModelRouterClient` (`017`), `MemoryClient` (`010`) |
| Backoff base | 100 ms | todas as classes retriable |
| Multiplicador | 2,0 (exponencial) | todas |
| Teto de backoff | 2 s | `ModelRouterClient` (alinhado a FM-01: 100 ms→2 s, 5x) |
| Jitter | ± 20% (*full jitter* recomendado) | todas |
| Timeout por tentativa (recall `010`) | `context.retrieval.memory_deadline_ms` (120 ms) | `MemoryClient` |
| Timeout por tentativa (sumarização `017`) | orçamento de NFR-003 (≤ 1200 ms total) | `ModelRouterClient` (sumarização) |
| Limiar do circuit breaker | 50% de falhas em janela de 20 chamadas | `017`, `010` |
| Half-open após | 10 s | circuito do dependente |
| Máx. re-entregas antes de DLQ | 5 | consumidores NATS/JetStream |

> **Regra:** operações **não idempotentes por natureza** não existem neste
> módulo — toda mutação usa `Idempotency-Key`. Portanto o *retry* (P2) é
> **sempre seguro** desde que a mesma chave seja reutilizada. Um *retry* com
> chave nova é uma nova operação lógica e NÃO DEVE ser emitido pelo cliente
> como reintento.

### 4.2 Fluxo ASCII de decisão de retry

```
      falha em operação
             │
             ▼
     ┌───────────────┐   não   ┌──────────────────────────┐
     │ retriable?    │────────▶│ retorna erro ao chamador  │
     │ (§3.2)        │         │ (RFC 7807, sem detail PII)│
     └──────┬────────┘         └──────────────────────────┘
            │ sim
            ▼
     ┌───────────────┐   sim   ┌──────────────────────────┐
     │ circuito       │───────▶│ degradação graciosa (§6)  │
     │ aberto?        │        │ ou erro retriable ao cham.│
     └──────┬────────┘         └──────────────────────────┘
            │ não
            ▼
     ┌───────────────┐   não   ┌──────────────────────────┐
     │ tentativas <   │───────▶│ DLQ (consumo) ou DEGRADED  │
     │ máx?           │        │/FAILED (bundle) + alerta   │
     └──────┬────────┘         └──────────────────────────┘
            │ sim
            ▼
     backoff = min(teto, base·2^n) ± jitter → repete com MESMA Idempotency-Key
```

---

## 5. Matriz FMEA (canônica — §9 do brief)

Reproduz e detalha os modos FM-01..FM-10 do brief. Nenhuma linha diverge do
brief.

| # | Modo de falha | Detecção | Isolamento | Recuperação | Idempotência / retry | RTO / RPO |
|---|-----------------|----------|------------|---------------|--------------------------|-----------|
| **FM-01** | `017-Model-Router` indisponível (limites/tokenizer) | timeout/circuit breaker | *Bulkhead*: só afeta `TokenBudgeter`/`TokenCounter` | Usa limites cacheados; estado `BUDGET_ALLOCATED` com defaults; `AIOS-CTX-0002` se sem cache | *Retry* idempotente c/ backoff exp. (100 ms→2 s, 5x) | RTO ≤ 30 s |
| **FM-02** | `010-Memory` recall timeout | *deadline* `context.retrieval.memory_deadline_ms` | Isola apenas o estágio `RETRIEVING` | Estado `DEGRADED`: monta com contexto parcial (system+history) | *Retry* parcial idempotente | RPO 0 (efêmero) |
| **FM-03** | Redis/`pgvector` (cache) down | health probe/erro | *Bulkhead*: caminho de cache isolado do caminho de montagem | *Bypass* de cache (`cache_outcome=BYPASS`); segue caminho `MISS`; `AIOS-CTX-0006` | Sem *retry* no caminho quente | RTO ≤ 60 s |
| **FM-04** | Falha de sumarização | erro do modelo (`017`) | Isola `HierarchicalCompressor` | *Fallback* `truncate` determinístico; `AIOS-CTX-0004`; ainda respeita orçamento (P6) | Idempotente por bundle | RPO 0 |
| **FM-05** | Embedding indisponível | erro `017` | Isola `EmbeddingClient` | *Fallback* para `prompt_hash` (*exact-match*) no cache; *ranking* por recência | *Retry* idempotente | RTO ≤ 60 s |
| **FM-06** | Orçamento infeasível | `tokens(system+instr) > budget` | — (rejeição determinística) | `REJECTED` + `AIOS-CTX-0001`; sugere modelo maior ao `017` | Determinístico | — |
| **FM-07** | Falha ao publicar evento | erro NATS | Isolado na tabela de outbox | **Outbox**: persistir evento na transação; `EventPublisher` re-tenta do outbox | Dedup por `event.id` | RPO ≤ 60 s |
| **FM-08** | *Cache poisoning* / falso-hit | eval offline + auditoria de `similarity` | Isola cluster de entradas afetado | Elevar `similarity_threshold`; invalidar cluster afetado; quarentena | Invalidação idempotente | RTO ≤ 5 min |
| **FM-09** | Conflito de idempotência | chave repetida, payload ≠ | — | `AIOS-CTX-0012`; não reexecuta | Registro de idempotência 24h | — |
| **FM-10** | Perda de nó do serviço | probe/orquestrador | *Stateless*: réplicas assumem | Estado quente em Redis reconstrói-se; verdade em PostgreSQL | N/A | RTO ≤ 5 min / RPO ≤ 60 s |

### 5.1 Falhas derivadas adicionais (consistentes com o brief)

| # | Modo de falha | Detecção | Recuperação | Idempotência / retry |
|---|-----------------|----------|---------------|--------------------------|
| FM-11 | Payload de entrada excede `context.limits.max_payload_bytes` | validação no `ContextApiGateway` | Rejeita com `AIOS-CTX-0013` (413) | não retriable |
| FM-12 | `BudgetProfile` com pesos que não somam 1.0 | validação de escrita (invariante §3.4 do brief) | Rejeita com `AIOS-CTX-0008` (422) | não retriable |
| FM-13 | `bundle_id` inexistente/expirado em `GetBundle` | consulta a `BundleStore` | Retorna `AIOS-CTX-0007` (404) | não retriable |
| FM-14 | Estouro de rate-limit por tenant | contador do `ContextApiGateway` | Rejeita com `AIOS-CTX-0015` (429) + `Retry-After` | *retry* pelo cliente após backoff |

---

## 6. Degradação graciosa (matriz de *fallback*)

O `ContextService` DEVE preferir uma resposta **parcial e viável** a uma falha
total, sempre marcando a resposta como degradada (estado `DEGRADED` na
máquina de estado do `ContextBundle`, brief §4.1, e *span* OTel anotado).

| Cenário | Modo pleno | Modo degradado | Sinalização |
|---------|------------|-------------------|---------------|
| `017` fora (FM-01) | Limites de modelo em tempo real | Limites cacheados; `BUDGET_ALLOCATED` com defaults | `AIOS-CTX-0002` só se sem cache; alerta |
| `010` fora/lento (FM-02) | Recall seletivo completo (top-K) | Contexto parcial: system+instrução+history, sem memória recuperada | estado `DEGRADED`; `AIOS-CTX-0005` |
| Cache fora (FM-03) | `CacheLookup` L1/L2 | *Bypass*; segue direto ao caminho de montagem completo | `cache_outcome=BYPASS`; `AIOS-CTX-0006` |
| Sumarização falha (FM-04) | Compressão hierárquica *map-reduce* | Truncamento determinístico, ainda dentro do orçamento | `AIOS-CTX-0004`; `compression_method=truncate` |
| Embedding fora (FM-05) | Cache por similaridade (`prompt_embedding`) | Cache por `prompt_hash` (exato); *ranking* por recência em vez de similaridade | `AIOS-CTX-0009` só se ambos os caminhos falharem |

> **Regra de correção sob degradação:** a métrica **Context Compression
> Ratio** (NFR-004) e o **Task Completion Rate** (NFR-005) reportados DEVEM
> refletir a degradação (não mascarar). Uma resposta degradada que reduza
> essas métricas abaixo do SLO DEVE disparar alerta em
> [Monitoring.md](./Monitoring.md).

---

## 7. RTO / RPO por componente

Consolidação das metas NFR-012 e da matriz FMEA (§9 do brief). O cache L1 é
deliberadamente efêmero — troca durabilidade por latência.

| Componente/estado | Backend | RPO alvo | RTO alvo | Estratégia de recuperação |
|----------------------|---------|----------|----------|-------------------------------|
| Cache L1 | Redis (RAM) | **tolerado** (reconstrói via cache-fill) | ≤ 60 s | *Bypass* seguido de reconstrução sob demanda (FM-03) |
| Cache L2 (`SemanticCacheEntry`) | PostgreSQL + `pgvector` | ≤ 60 s | ≤ 5 min | Failover streaming p/ standby (`027`) |
| `ContextBundle`/`ContextFragment` | PostgreSQL | ≤ 60 s | ≤ 5 min | Failover streaming; bundles são efêmeros por design (`expires_at`) |
| `BudgetProfile` | PostgreSQL | ≤ 60 s | ≤ 5 min | Failover streaming |
| `SummaryNode` | PostgreSQL + `pgvector` | ≤ 60 s (tolerável — cache de reuso) | ≤ 5 min | Reconstrução por nova sumarização se perdido |
| Blobs de fragmentos | MinIO | ≤ 60 s | ≤ 5 min | Replicação de bucket |
| Serviço (control plane) | .NET 10 | — | ≤ 5 min (NFR-012) | Réplicas *stateless*; *drill* de failover |
| Eventos de domínio | Outbox + JetStream | ≤ 60 s (FM-07) | ≤ 1 min | *Relay* reenvia pendentes |

Disponibilidade-alvo do serviço: **≥ 99,95%** (NFR-008).

---

## 8. DLQ e recuperação de consumo

O `ContextService` consome eventos (brief §6.2) via JetStream:
`memory.item.consolidated/deleted`, `model.registry.updated`,
`knowledge.node.updated`, `agent.lifecycle.suspended/terminated`,
`policy.decision.updated`. O tratamento de *poison messages* segue:

```
consumidor NATS/JetStream (ex.: memory.item.consolidated)
        │ processa evento (idempotente por event.id)
        ▼
   ┌─────────┐  ok   ┌───────────┐
   │ handler │──────▶│ ack       │
   └────┬────┘       └───────────┘
        │ erro (ex.: falha ao invalidar SummaryNode/cache)
        ▼
   ┌──────────────┐  n < 5   redelivery (backoff)
   │ nak + retry  │─────────▶ volta ao consumidor
   └──────┬───────┘
          │ n ≥ 5
          ▼
   ┌──────────────────────┐
   │ DLQ (stream dedicada)│──▶ alerta 024 + inspeção/replay manual
   └──────────────────────┘
```

Regras normativas:

- Consumidores DEVEM ser idempotentes (dedup por `event.id`) — reprocessar da
  DLQ NÃO DEVE duplicar efeito (ex.: invalidar cache duas vezes é inofensivo,
  mas deduplicar evita ruído de auditoria).
- Mensagens em DLQ DEVEM ser reprocessáveis (*replay*) após correção da causa
  raiz.
- Toda entrada em DLQ DEVE emitir alerta e registro para auditoria (025).
- Eventos de **`memory.item.deleted`** (expurgo LGPD/RTBF a montante) que
  caiam em DLQ têm **prioridade máxima** de resolução — a invalidação de cache
  derivada é uma obrigação legal (brief §12.3).

---

## 9. Cenários de recuperação (runbooks resumidos)

| Cenário | Passos de recuperação (resumo) | Runbook |
|---------|-----------------------------------|---------|
| Failover de PostgreSQL (FM-10) | Promover *standby* → religar *pool* → drenar outbox `pending` → verificar RLS e lag | `RB-CTX-01` |
| `017-Model-Router` degradado (FM-01/04/05) | Confirmar circuit breaker → validar limites/tokenizers cacheados → drenar *backlog* de sumarização | `RB-CTX-02` |
| Cache backend fora (FM-03) | Confirmar *bypass* ativo → monitorar aumento de latência p99 → restaurar Redis/`pgvector` → validar reconstrução de L1 | `RB-CTX-03` |
| Falso-hit / *cache poisoning* (FM-08) | Identificar cluster de entradas via auditoria de `similarity` → elevar `similarity_threshold` → invalidar cluster → validar NFR-011 | `RB-CTX-04` |
| Drenagem de DLQ (§8) | Analisar causa → corrigir → *replay* controlado → confirmar dedup | `RB-CTX-05` |
| Expurgo LGPD sob falha (DLQ de `memory.item.deleted`) | Priorizar reprocessamento → confirmar invalidação de cache/`SummaryNode` → auditar (025) | `RB-CTX-06` |

Os *runbooks* completos vivem em [Monitoring.md](./Monitoring.md) e são
acionados pelos alertas de [Metrics.md](./Metrics.md).

---

## 10. Riscos e alternativas

| Risco | Impacto | Mitigação | Alternativa descartada |
|-------|---------|-----------|----------------------------|
| *Backlog* de sumarização cresce sob FM-01/04 prolongado | Compressão degrada para truncamento por período indefinido | Limite de tentativas + alerta; *fallback* determinístico sempre disponível (P6) | Bloquear `assemble` até sumarização (rejeitado: viola NFR-002/003) |
| Falso-hit de cache não detectado a tempo (FM-08) | Resposta inadequada servida repetidamente | Auditoria de amostragem contínua (NFR-011) + `similarity_threshold` conservador | Sem auditoria de amostragem (rejeitado: viola NFR-011) |
| Perda silenciosa de cache L1 confundida com bug de qualidade | Diagnóstico incorreto (queda de hit ratio interpretada como regressão de modelo) | Cache L1 documentado como efêmero; métrica `cache_outcome` distingue `BYPASS`/`MISS`/`HIT` | Persistir L1 (rejeitado: viola latência/NFR-001) |
| DLQ acumula eventos de expurgo LGPD (`memory.item.deleted`) | Não conformidade — cache com dado que deveria ter sido invalidado | Prioridade máxima + alerta dedicado (§8) | Descartar após N falhas (rejeitado: obrigação legal) |
| Failover de PostgreSQL excede RTO sob carga | Indisponibilidade > 5 min | *Standby* quente pré-aquecido; *drill* periódico (NFR-012) | Sem *standby* (rejeitado: viola NFR-008/012) |

---

## 11. Rastreabilidade

| Requisito | Coberto por (esta doc) |
|-----------|---------------------------|
| NFR-008 (disponibilidade ≥ 99,95%) | §7 |
| NFR-011 (falso-hit ≤ 0,5%) | §5 FM-08, §9, §10 |
| NFR-012 (RTO ≤ 5 min / RPO ≤ 60 s) | §7, §9 |
| FR-010 (degradação graciosa) | §6 |
| FR-011 (idempotência de mutações) | §2, §5 FM-09 |
| FR-008 (invalidação por evento) | §8 |

---

## 12. Referências

- [_DESIGN_BRIEF.md §9](./_DESIGN_BRIEF.md) — fonte única de verdade
  (FM-01..FM-10).
- [RFC-0001 §5.4/§5.5](../003-RFC/RFC-0001-Architecture-Baseline.md) — erros e
  idempotência.
- [Scalability.md](./Scalability.md) — *bulkhead*, sharding, backpressure.
- [StateMachine.md](./StateMachine.md) — estados `DEGRADED`/`REJECTED`/`FAILED`.
- [Monitoring.md](./Monitoring.md), [Metrics.md](./Metrics.md) — detecção e
  alertas.
- [ADR.md](./ADR.md), [RFC.md](./RFC.md) — decisões que fundamentam a
  recuperação.
- Glossário: [FMEA, DLQ, RTO/RPO, Idempotency, Outbox, Semantic Cache](../040-Glossary/Glossary.md).
