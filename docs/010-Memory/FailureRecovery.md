---
Documento: Failure Recovery (FMEA e Estratégia de Recuperação)
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100, ADR-0102, ADR-0103, ADR-0105, ADR-0106, ADR-0108
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 001-Architecture, 005-Database, 017-Model-Router, 020-Messaging (NATS/JetStream), 024-Observability, 025-Audit, 026-Cost-Optimizer, 027-HA-DR, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Failure Recovery

> Este documento deriva da **§9 (Modos de Falha e Estratégia de Recuperação)** do
> `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**. Todos os códigos de erro
> (`AIOS-MEM-NNNN`), metas de RTO/RPO e princípios de idempotência reutilizam os
> contratos centrais da [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md)
> (§5.4 erros, §5.5 idempotência) sem redefini-los. Palavras normativas conforme
> RFC 2119/8174.

---

## 1. Objetivo e escopo

Este documento especifica a **Análise de Modos de Falha e Efeitos (FMEA)** do
`MemoryService` (010), com detecção, isolamento, recuperação, idempotência,
política de *retries*/*backoff*, uso de *Dead-Letter Queue* (DLQ), metas de
**RTO/RPO** por camada e as regras de **degradação graciosa**.

O `MemoryService` é o gerenciador de memória cognitiva hierárquica do AIOS
(análogo a MMU + swap + filesystem). Sua resiliência é assimétrica **por design**:
camadas duráveis (Long-Term, Semantic, Procedural, Episodic, Knowledge Graph)
têm garantias fortes (RPO ≤ 5 min); a Working Memory é **efêmera e reconstruível**
(perda tolerada, NFR-008). O documento é normativo para operação e para os
*runbooks* referenciados em [Monitoring.md](./Monitoring.md).

---

## 2. Princípios de recuperação (invariantes)

Estes princípios governam TODA a matriz de falhas da §5:

1. **P1 — Idempotência universal.** Toda mutação (`remember`, `consolidate`,
   `forget`, `PurgeItem`) DEVE ser idempotente por `Idempotency-Key`
   (RFC-0001 §5.5). Repetições retornam o resultado memoizado do `IdempotencyStore`
   (Redis, ≥ 24h), nunca reaplicam o efeito.
2. **P2 — Retry com backoff exponencial + jitter.** Falhas transitórias
   (`retriable = true`) DEVEM ser repetidas com backoff exponencial e *jitter*
   para evitar *thundering herd*. Falhas `retriable = false` NÃO DEVEM ser
   repetidas — falham rápido e retornam ao chamador.
3. **P3 — Outbox garante atomicidade DB+evento.** Todo evento de domínio é gravado
   na tabela `OutboxEvent` na mesma transação da mutação; o `OutboxPublisher`
   reenvia *at-least-once*. RPO de evento = 0.
4. **P4 — Consumidores deduplicam por `event.id`.** Entrega *at-least-once*
   implica que consumidores DEVEM deduplicar (RFC-0001 §5.2).
5. **P5 — Degradação graciosa antes de falha total.** A indisponibilidade de um
   backend NÃO DEVE derrubar operações que podem ser servidas por outro caminho
   (ex.: grafo indisponível → *recall* cai para `hybrid` sem grafo).
6. **P6 — Consolidação sempre reversível.** Nenhuma consolidação muta itens sem
   antes gravar uma `ConsolidationVersion` (pré-imagem); qualquer regressão de
   **Memory Recall Rate** dispara *rollback* automático (NFR-011, ADR-0102).
7. **P7 — Isolamento de falha por tenant/shard.** Uma falha em um shard ou tenant
   NÃO DEVE se propagar a outros (padrão *bulkhead*, ver [Scalability.md](./Scalability.md)).

---

## 3. Detecção e classificação de falhas

### 3.1 Sinais de detecção

| Mecanismo | O que detecta | Origem |
|-----------|---------------|--------|
| Health/readiness probes | Backend caído (Redis/PG/AGE/MinIO/Model Router) | [Deployment.md](./Deployment.md) |
| Timeouts por dependência | Latência acima do orçamento (ex.: embedding, ANN) | `EmbeddingClient`, `RecallEngine` |
| Circuit breaker | Sequência de falhas a um dependente (abre o circuito) | `EmbeddingClient` (017), `KnowledgeGraphAdapter` |
| Contadores do `QuotaManager` | Estouro de cota / saturação de *inflight* | `QuotaManager` |
| Validação pós-consolidação | Regressão de Recall Rate (A/B por versão) | `ConsolidationEngine` (NFR-011) |
| Outbox `status = pending` antigo | Evento não publicado após crash | `OutboxPublisher` |
| Contador de re-entregas JetStream | *Poison message* (N falhas de consumo) | `RetentionScheduler`/consumers |
| Réplica/lag de streaming PG | Divergência do standby (risco de RPO) | 005/027 |

### 3.2 Taxonomia de erros (ligada ao catálogo §5.2 do brief)

| Classe | Códigos | Retriable | Estratégia base |
|--------|---------|-----------|-----------------|
| Cliente (contrato) | `AIOS-MEM-0001`, `0002`, `0020`, `0021` | não | Rejeitar; corrigir requisição. |
| Autorização/tenant | `AIOS-MEM-0004`, `0005` | não | *Default deny*; auditar (025). |
| Idempotência/consolidação (conflito) | `AIOS-MEM-0003`, `0040`, `0042` | não | Retornar memoizado / recusar concorrência. |
| Cota/backpressure | `AIOS-MEM-0010`, `0011` | sim (429) | Represar + *backoff*; podar (ForgettingEngine). |
| Dependência externa | `AIOS-MEM-0030` (Model Router), `0031` (backend camada) | sim | Degradar + *retry* + circuit breaker. |
| Consolidação (falha) | `AIOS-MEM-0041` | sim | *Rollback* automático à versão anterior. |
| Legal | `AIOS-MEM-0050` (legal hold), `0051` (RTBF aceito) | não | Bloquear/agendar expurgo rastreável. |
| Interno | `AIOS-MEM-0060` | sim | *Retry* limitado; alertar. |

---

## 4. Política de retries, backoff e circuit breaking

### 4.1 Parâmetros normativos

| Parâmetro | Valor recomendado | Aplica-se a |
|-----------|-------------------|-------------|
| Tentativas máximas (dependência externa) | 5 | `EmbeddingClient` (017), MinIO, AGE |
| Backoff base | 100 ms | todas as classes retriable |
| Multiplicador | 2,0 (exponencial) | todas |
| Teto de backoff | 30 s | todas |
| Jitter | ± 20% (full jitter recomendado) | todas |
| Timeout por tentativa (embedding) | 2 s | `EmbeddingClient` |
| Timeout por tentativa (ANN recall) | orçamento NFR-003 (≤ 150 ms) | `RecallEngine` |
| Limiar do circuit breaker | 50% de falhas em janela de 20 chamadas | 017, AGE |
| Half-open após | 10 s | circuito do dependente |
| Máx. re-entregas antes de DLQ | 5 | consumidores NATS/JetStream |

> **Regra:** operações **não idempotentes por natureza** não existem neste módulo —
> toda mutação usa `Idempotency-Key`. Portanto o *retry* (P2) é **sempre seguro**
> desde que a mesma chave seja reutilizada. Um *retry* com chave nova é uma nova
> operação lógica e NÃO DEVE ser emitido pelo cliente como reintento.

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
     │ aberto?        │        │ ou 503 AIOS-MEM-0031      │
     └──────┬────────┘         └──────────────────────────┘
            │ não
            ▼
     ┌───────────────┐   não   ┌──────────────────────────┐
     │ tentativas <   │───────▶│ DLQ (consumo) ou FAILED   │
     │ máx?           │        │ (item) + alerta           │
     └──────┬────────┘         └──────────────────────────┘
            │ sim
            ▼
     backoff = min(teto, base·2^n) ± jitter → repete com MESMA Idempotency-Key
```

---

## 5. Matriz FMEA (canônica — §9 do brief)

Reproduz e detalha os modos F1..F10 do brief. Nenhuma linha diverge do brief.

| # | Modo de falha | Detecção | Isolamento | Recuperação | Idempotência / retry | RTO / RPO |
|---|---------------|----------|------------|-------------|----------------------|-----------|
| **F1** | Redis (Working/hot) indisponível | health probe / timeout | *Bulkhead*: só afeta caminho quente; camadas duráveis seguem por PG | *Fail-open* p/ camadas duráveis; Working reconstruível; degradação graciosa (§6) | leitura com *retry*; Working **sem** *retry* | RTO ≤ 1 min; RPO Working = tolerado (NFR-008) |
| **F2** | PostgreSQL primário cai | lag de replicação / erro | Failover isola nó falho | Failover p/ standby (027); reprocessa Outbox pendente | escrita idempotente por `Idempotency-Key` | RTO ≤ 15 min; RPO ≤ 5 min |
| **F3** | Model Router (017) indisponível | erro `AIOS-MEM-0030` / circuit breaker | Enfileira sem bloquear escrita | Persiste item `state=INGESTED` sem embedding; gera embedding em *backlog* | *retry* backoff exp.; *batch* | sem perda; embedding **eventual** |
| **F4** | MinIO indisponível | erro S3 / timeout | Represa apenas blobs; inline segue | Mantém conteúdo inline se possível; represa blob; *retry* | *put* idempotente por `content_hash` | RPO ≤ 5 min |
| **F5** | Apache AGE / grafo falha | erro de consulta / circuit breaker | Isola só o caminho de grafo | *Recall* cai p/ `hybrid` sem grafo (degradação); `mode=graph` → `AIOS-MEM-0031` | *retry* | sem perda |
| **F6** | Consolidação corrompe/regressa Recall Rate | validação NFR-011 (A/B por versão) falha | *Rollback* isola a versão ruim | *Rollback* automático à `ConsolidationVersion` anterior; evento `consolidation.rolledback` | job idempotente por `version_id` | estado restaurado (RPO = 0 lógico) |
| **F7** | Estouro de cota | contador `QuotaManager` | *Backpressure* no shard/tenant | 429 `AIOS-MEM-0011` (inflight) / `AIOS-MEM-0010` (cota); dispara poda (ForgettingEngine) | 429 *retriable* | — |
| **F8** | Evento não publicado (crash pós-commit) | Outbox `status=pending` | Isolado na tabela outbox | Outbox *relay* reenvia *at-least-once* | dedup por `event.id` | RPO evento = 0 (Outbox) |
| **F9** | Duplicidade de escrita | `Idempotency-Key` repetida | — | Retorna resultado memoizado (`IdempotencyStore`) | dedup ≥ 24h | — |
| **F10** | *Poison message* (consumo) | N falhas de re-entrega | DLQ isola a mensagem | Move p/ DLQ JetStream; alerta; inspeção/replay manual | — | — |

### 5.1 Falhas derivadas adicionais (consistentes com o brief)

| # | Modo de falha | Detecção | Recuperação | Idempotência / retry |
|---|---------------|----------|-------------|----------------------|
| F11 | Conflito de idempotência (mesma key, payload divergente) | `IdempotencyStore` | Rejeita com `AIOS-MEM-0003` (409); não reaplica | não retriable |
| F12 | Dimensão de embedding incompatível com índice | validação no `LayerRouter` | Rejeita com `AIOS-MEM-0021` (422); reindex requer nova config (`memory.embedding.dim`) | não retriable |
| F13 | Tentativa de purgar item em `legal_hold` | `ForgettingEngine` | Bloqueia com `AIOS-MEM-0050` (423), salvo RTBF autorizado (§4.1 invariante do brief) | não retriable |
| F14 | *Rollback* impossível (versão inexistente/já ativa) | `ConsolidationVersionManager` | Rejeita com `AIOS-MEM-0042` (409) | não retriable |

---

## 6. Degradação graciosa (matriz de *fallback*)

O `MemoryService` DEVE preferir uma resposta **parcial e correta** a uma falha
total, sempre marcando a resposta como degradada (campo `degraded=true` no
`RecallResult` e *span* OTel anotado).

| Cenário | Modo pleno | Modo degradado | Sinalização |
|---------|-----------|----------------|-------------|
| Redis fora (F1) | *recall* quente + durável | apenas camadas duráveis (PG); Working reconstruída sob demanda | `degraded=true`; alerta |
| AGE fora (F5) | `hybrid` = lexical + ANN + grafo | `hybrid` sem grafo; `mode=graph` explícito → `AIOS-MEM-0031` | erro só quando grafo é exigido |
| Model Router fora (F3) | embedding síncrono | item `INGESTED` sem embedding; *recall* lexical até *backfill* | evento adiado; *backlog* |
| MinIO fora (F4) | blob externalizado | mantém inline (se ≤ limite); represa externalização | `AIOS-MEM-0031` só se inline inviável |
| Réplica de leitura fora | *recall* em réplica | *recall* no primário (com *rate-limit*) | maior latência; alerta |
| Saturação (F7) | escrita imediata | represa admissão; 429; poda | `AIOS-MEM-0011` |

> **Regra de correção sob degradação:** o valor de **Memory Recall Rate** reportado
> (NFR-002) DEVE refletir a degradação (não mascarar). Uma resposta degradada que
> reduza o Recall Rate abaixo do SLO DEVE disparar alerta em [Monitoring.md](./Monitoring.md).

---

## 7. RTO / RPO por camada

Consolidação das metas NFR-006/007/008 do brief. As camadas efêmeras trocam
durabilidade por latência **deliberadamente** (ADR-0007, ADR-0100).

| Camada | Backend | RPO alvo | RTO alvo | Estratégia de recuperação |
|--------|---------|----------|----------|---------------------------|
| Working | Redis (RAM) | **tolerado** (NFR-008) | ≤ 1 min | Reconstrução sob demanda; sem garantia de dado |
| Short-Term | Redis (hot) + PG (proj.) | ≤ 5 min (projeção durável) | ≤ 5 min | Redis reconstruído da projeção PG (CQRS) |
| Long-Term | PostgreSQL | ≤ 5 min | ≤ 15 min | Failover streaming p/ standby (027) |
| Semantic | PG + pgvector (HNSW) | ≤ 5 min | ≤ 15 min | Failover PG + reindex HNSW online se necessário |
| Procedural | PostgreSQL | ≤ 5 min | ≤ 15 min | Failover streaming |
| Episodic | PG + pgvector (part. tempo) | ≤ 5 min | ≤ 15 min | Failover + restauração de partição |
| Knowledge Graph | Apache AGE | ≤ 5 min | ≤ 15 min | Failover PG (AGE é extensão do PG) |
| Blobs | MinIO | ≤ 5 min | ≤ 15 min | Replicação de bucket; `content_hash` valida integridade |
| Serviço (control plane) | .NET 10 | — | ≤ 15 min (NFR-007) | Réplicas *stateless*; *drill* de failover |
| Eventos de domínio | Outbox + JetStream | **0** (Outbox) | ≤ 1 min | *Relay* reenvia pendentes |

Disponibilidade-alvo do serviço: **≥ 99,95%** (NFR-005).

---

## 8. DLQ e recuperação de consumo

O `MemoryService` consome eventos (brief §6.2) via JetStream. O tratamento de
*poison messages* segue F10:

```
consumidor NATS/JetStream
        │ processa evento (idempotente por event.id)
        ▼
   ┌─────────┐  ok   ┌───────────┐
   │ handler │──────▶│ ack       │
   └────┬────┘       └───────────┘
        │ erro
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

- Consumidores DEVEM ser idempotentes (dedup por `event.id`) — reprocessar da DLQ
  NÃO DEVE duplicar efeito.
- Mensagens em DLQ DEVEM ser reprocessáveis (*replay*) após correção da causa raiz.
- Toda entrada em DLQ DEVE emitir alerta e registro para auditoria (025).
- Eventos de **RTBF** (`security.rtbf.requested`) que caiam em DLQ têm prioridade
  máxima de resolução (obrigação legal — LGPD/GDPR, brief §12.3).

---

## 9. Cenários de recuperação (runbooks resumidos)

| Cenário | Passos de recuperação (resumo) | Runbook |
|---------|-------------------------------|---------|
| Failover de PostgreSQL (F2) | Promover standby → religar pool → drenar Outbox `pending` → verificar RLS e lag | `RB-MEM-01` |
| Model Router degradado (F3) | Confirmar circuit breaker → drenar *backlog* de embeddings → validar `embedding_model` registrado | `RB-MEM-02` |
| Rollback de consolidação (F6) | Identificar `version_id` regressiva → `POST /consolidate/{jobId}/rollback` → validar Recall Rate ≥ pré − 0,01 | `RB-MEM-03` |
| Saturação/cota (F7) | Inspecionar `aios_memory_usage_ratio` → acionar poda → revisar cotas com 026 | `RB-MEM-04` |
| Drenagem de DLQ (F10) | Analisar causa → corrigir → *replay* controlado → confirmar dedup | `RB-MEM-05` |
| RTBF sob falha (F13/DLQ) | Priorizar expurgo → confirmar remoção de vetor + blob + tombstone → auditar (025) | `RB-MEM-06` |

Os *runbooks* completos vivem em [Monitoring.md](./Monitoring.md) e são acionados
pelos alertas de [Metrics.md](./Metrics.md).

---

## 10. Riscos e alternativas

| Risco | Impacto | Mitigação | Alternativa descartada |
|-------|---------|-----------|------------------------|
| *Backlog* de embeddings cresce sob F3 prolongado | *recall* semântico degradado por tempo indefinido | Limite de *backlog* + alerta; *fallback* lexical; priorização por saliência | Bloquear escrita até embedding (rejeitado: viola NFR-001) |
| *Rollback* automático oscilante (*flapping*) em F6 | consolidação nunca converge | Histerese + limite de tentativas → `FAILED` p/ inspeção (ADR-0102) | *Rollback* manual apenas (rejeitado: viola NFR-011) |
| Perda silenciosa de Working (F1) confundida com bug | diagnóstico errado | Working documentada como efêmera (NFR-008); métrica de reconstrução | Persistir Working (rejeitado: viola latência/ADR-0007) |
| DLQ acumula RTBF (risco legal) | não conformidade LGPD/GDPR | Prioridade máxima + alerta dedicado (§8) | Descartar após N falhas (rejeitado: obrigação legal) |
| Failover PG excede RTO sob carga | indisponibilidade > 15 min | Standby quente pré-aquecido; *drill* periódico (NFR-007) | Sem standby (rejeitado: viola NFR-005/006) |

---

## 11. Rastreabilidade

| Requisito | Coberto por (esta doc) |
|-----------|------------------------|
| NFR-005 (disponibilidade ≥ 99,95%) | §7, §6 |
| NFR-006 (RPO ≤ 5 min duráveis) | §7 |
| NFR-007 (RTO ≤ 15 min) | §7, §9 |
| NFR-008 (Working efêmera) | §5 F1, §7 |
| NFR-011 (consolidação sem regressão) | §5 F6, §10 |
| FR-007 (rollback de consolidação) | §5 F6, §9 |
| FR-011 (RTBF) | §8, §9, F13 |

---

## 12. Referências

- [_DESIGN_BRIEF.md §9](./_DESIGN_BRIEF.md) — fonte única de verdade (F1..F10).
- [RFC-0001 §5.4/§5.5](../003-RFC/RFC-0001-Architecture-Baseline.md) — erros e idempotência.
- [Scalability.md](./Scalability.md) — *bulkhead*, sharding, backpressure.
- [StateMachine.md](./StateMachine.md) — estados `FAILED`/`ROLLING_BACK`/`FORGET_PENDING`.
- [Monitoring.md](./Monitoring.md), [Metrics.md](./Metrics.md) — detecção e alertas.
- [ADR.md](./ADR.md), [RFC.md](./RFC.md) — decisões que fundamentam a recuperação.
- Glossário: [FMEA, DLQ, RTO/RPO, Idempotency, Outbox](../040-Glossary/Glossary.md).
