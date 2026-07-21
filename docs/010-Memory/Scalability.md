---
Documento: Scalability (Modelo de Escala e Concorrência)
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100, ADR-0101, ADR-0105, ADR-0106, ADR-0107, ADR-0108
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 001-Architecture, 005-Database, 006-Kernel, 008-Lifecycle, 017-Model-Router, 020-Messaging (NATS/JetStream), 026-Cost-Optimizer, 027-HA-DR, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Scalability

> Este documento deriva da **§10 (Escalabilidade e Concorrência)** do
> `_DESIGN_BRIEF.md` e das metas de escala dos NFRs (§7.2). **NÃO PODE
> contradizê-lo**. Sharding, limites-alvo e política de locks são canônicos.
> Reutiliza os contratos da [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md)
> e o modelo de shard `hash(tenant,agent) mod N` de
> [001-Architecture](../001-Architecture/Architecture.md). Palavras normativas
> conforme RFC 2119/8174.

---

## 1. Objetivo e metas de escala

O `MemoryService` (010) DEVE escalar de um punhado de agentes até **milhões de
agentes** por meio de particionamento determinístico, separação estado quente/frio
e consolidação/poda que mantêm o crescimento **sub-linear**. As metas canônicas
(brief §7.2/§10) são:

| Meta | Valor-alvo | Requisito |
|------|-----------|-----------|
| Vetores por tenant | ≥ 10^8 com p99 ANN ≤ 150 ms | NFR-012 |
| Throughput de escrita | ≥ 5.000 `remember`/s por shard | NFR-004 |
| Throughput de leitura | ≥ 2.000 `recall`/s por shard | NFR-004 |
| Latência `recall` ANN | p99 ≤ 150 ms (Semantic/Episodic) | NFR-003 |
| Latência `recall` grafo | p99 ≤ 250 ms | NFR-003 |
| Uso vs. cota | `aios_memory_usage_ratio` < 1,0 (poda mantém ≤ 100%) | NFR-009 |
| Agentes suportados | milhões (via *cold agents* reidratáveis) | ADR-0107, brief §10 |

---

## 2. Modelo de escala horizontal

O serviço é **stateless** no plano de controle (.NET 10) e escala horizontalmente
por réplicas atrás do YARP; o estado vive nos backends particionados. A dimensão
crítica é o **estado**, não a computação.

```
                         ┌──────────────── YARP / Gateway ────────────────┐
                         │   roteia por shard = hash(tenant_id, agent_id)  │
                         └───────┬───────────────┬───────────────┬────────┘
                                 │               │               │
                        ┌────────▼─────┐ ┌───────▼──────┐ ┌──────▼───────┐
                        │ MemorySvc r1 │ │ MemorySvc r2 │ │ MemorySvc rN │  (stateless, N réplicas)
                        └───┬───┬───┬──┘ └──┬───┬───┬───┘ └──┬───┬───┬───┘
                            │   │   │       │   │   │        │   │   │
        ┌───────────────────┘   │   └───────┼───┼───┼────────┘   │   └────────────┐
        ▼                       ▼           ▼   ▼   ▼            ▼                ▼
   ┌─────────┐          ┌──────────────┐  ┌───────────────┐  ┌────────────┐  ┌────────┐
   │ Redis   │          │ PostgreSQL   │  │ pgvector HNSW │  │ Apache AGE │  │ MinIO  │
   │ (hot,   │          │ shard 0..N   │  │ part./tenant  │  │ (grafo)    │  │ (blobs)│
   │  locks) │          │ (RLS/tenant) │  │ Episodic:tempo│  │            │  │ +hash  │
   └─────────┘          └──────────────┘  └───────────────┘  └────────────┘  └────────┘
       ▲                                                                          ▲
       │ estado quente (Working/Short-Term)              estado frio (cold agents / snapshots)
       └──────────────────────────────────────────────────────────────────────────┘
```

- **Réplicas do serviço** escalam com a taxa de requisições (CPU-bound na fusão/
  re-rank do `RecallEngine`).
- **Backends** escalam com o volume de dados (por shard/tenant).
- Uma réplica PODE servir qualquer tenant, mas roteamento por shard maximiza
  localidade de cache e reduz *fan-out* de leitura.

---

## 3. Particionamento e sharding (ADR-0108)

### 3.1 Estratégia canônica (brief §10)

| Dimensão | Estratégia | Chave de partição |
|----------|-----------|-------------------|
| Shard global | `shard = hash(tenant_id, agent_id) mod N` | (tenant, agent) |
| Vetores (Semantic) | Particionados por tenant; HNSW por partição | tenant |
| Episodic | Particionamento `RANGE` **mensal** por tempo | tempo (`created_at`) |
| Knowledge Graph (AGE) | Subgrafo por (tenant, agente) | (tenant, agent) |
| Blobs (MinIO) | *content-addressed* por `content_hash`; prefixo por tenant | tenant + hash |
| Cotas | Contadores por (tenant, agente, camada) | (tenant, agent, layer) |

### 3.2 Diagrama de particionamento

```
tenant "acme"
   ├── agent A ──hash──▶ shard 3 ──▶ PG shard3 (RLS acme) + HNSW part(acme) + AGE subgraph(acme:A)
   ├── agent B ──hash──▶ shard 7 ──▶ PG shard7 (RLS acme) + HNSW part(acme) + AGE subgraph(acme:B)
   └── ...

Episodic (qualquer agente) → partição por tempo:
   episodic_2026_05 │ episodic_2026_06 │ episodic_2026_07 (quente) │ ...
   └── partições frias movíveis p/ MinIO (ARCHIVED); poda por retenção (365d default)
```

### 3.3 Rebalanceamento

- Aumentar `N` (número de shards) exige *rehash*; DEVE ser feito com migração
  online por tenant, sem *downtime* (janela de dupla-escrita + *cutover*).
- Reindex HNSW é **online** (brief §10): `memory.hnsw.ef_search` é recarregável;
  `memory.hnsw.m` / `ef_construction` exigem reindex (não recarregáveis — §8 brief).
- Migração para índice vetorial externo está prevista como caminho evolutivo
  (ADR-0005 / ADR-0101) caso pgvector/HNSW atinja o teto por tenant.

---

## 4. Estado quente vs. frio (caminho para milhões de agentes)

A chave para milhões de agentes é que **agentes suspensos não consomem RAM**
(*cold agents*, glossário). A memória segue o ciclo de vida do agente (brief §6.2):

| Sinal (evento consumido) | Ação de memória | Efeito de escala |
|--------------------------|-----------------|------------------|
| `agent.lifecycle.terminated` | Expira Working/Short-Term do agente | Libera Redis |
| `agent.lifecycle.suspended` | Move estado quente → cold (PG/MinIO) | Libera RAM; agente vira *cold* |
| `context.recall.requested` (agente reativado) | Reidrata `ARCHIVED` → `ACTIVE` sob demanda | RAM só p/ agentes ativos |

```
   milhões de agentes registrados
            │
   ┌────────┴─────────┐
   │ ativos (quentes) │  ← Redis: Working + Short-Term (minoria a qualquer instante)
   └────────┬─────────┘
            │ suspend
            ▼
   ┌──────────────────┐  reidrata sob recall   ┌──────────────────┐
   │ cold (PG/MinIO)  │───────────────────────▶│ quente novamente │
   │ Long/Sem/Epi/KG  │                         └──────────────────┘
   └──────────────────┘
```

Assim o custo de RAM escala com **agentes ativos simultâneos**, não com o total de
agentes — habilitando escala a milhões (ADR-0107 fronteira Runtime↔Memory; ADR-0108).

---

## 5. Concorrência e locks

### 5.1 Modelo de concorrência (brief §10)

| Caminho | Modelo | Justificativa |
|---------|--------|---------------|
| Escrita de item (`remember`) | *Append-preferencial*; sem locks longos | Caminho quente — latência NFR-001 |
| Atualização de `MemoryItem` | *Optimistic concurrency* por `updated_at`/versão | Evita contenção; conflito raro |
| Consolidação por (tenant, agente) | **Lock distribuído** (Redis, ADR-0006) | Serializa jobs concorrentes do mesmo escopo |
| Jobs de consolidação | Idempotentes por `version_id` | *Retry* seguro (ver F6) |
| Recall | Sem lock (leitura) | Escala com réplicas de leitura |

> **Regra:** locks distribuídos são usados **apenas** para consolidação por
> (tenant, agente) — nunca no caminho quente de `remember`/`recall`. Um lock longo
> no caminho quente violaria NFR-001/003.

### 5.2 Conflito de consolidação

Dois jobs de consolidação concorrentes para o mesmo (tenant, agente) são
serializados pelo lock; se um segundo job tenta mutar a mesma versão, retorna
`AIOS-MEM-0040` (conflito de versão concorrente). O *rollback* é idempotente por
`version_id` (`AIOS-MEM-0042` se a versão não existe/já ativa).

---

## 6. Backpressure e admissão

O `QuotaManager` aplica *backpressure* antes da saturação (brief §10, F7):

```
   remember(...) ───▶ ┌──────────────────────────┐
                      │ inflight < max_inflight?  │
                      │ (memory.backpressure.     │
                      │  max_inflight = 1000)     │
                      └───────┬───────────┬───────┘
                          sim │           │ não
                              ▼           ▼
                      ┌───────────┐  ┌─────────────────────────┐
                      │ admite    │  │ 429 AIOS-MEM-0011        │
                      │ escrita   │  │ (backpressure, retriable)│
                      └─────┬─────┘  └─────────────────────────┘
                            ▼
                  ┌──────────────────────────┐
                  │ cota (itens/bytes) ok?    │──não──▶ 429 AIOS-MEM-0010 + poda (ForgettingEngine)
                  └──────────┬───────────────┘
                             │ sim
                             ▼
                        persiste + Outbox
```

| Mecanismo | Chave/limite | Código |
|-----------|--------------|--------|
| *Inflight* de escrita | `memory.backpressure.max_inflight` (1000, G) | `AIOS-MEM-0011` |
| Cota de itens/bytes/vetores | `memory.quota.max_items` (1M), `memory.quota.max_bytes` (10 GiB) | `AIOS-MEM-0010` |
| Represamento JetStream | limites de consumidor/stream | — |
| Poda automática | `ForgettingEngine` dispara ao exceder | evento `quota.exceeded` |

Consumo é reportado ao **026-Cost-Optimizer** via evento `memory.quota.exceeded`
e métrica `aios_memory_usage_ratio`.

---

## 7. Leitura em escala (CQRS)

Para sustentar ≥ 2.000 `recall`/s por shard (NFR-004) com p99 baixo (NFR-003):

| Técnica | Descrição |
|---------|-----------|
| Réplicas de leitura PG | `recall` durável servido por réplicas; primário reservado a escrita |
| Cache de resultados quentes (Redis) | Consultas recorrentes memoizadas por (query, filtros, tenant) |
| Short-Term como projeção | Serve leitura recente sem tocar Long-Term (CQRS, glossário) |
| Fusão RRF paralela | Lexical + ANN + grafo executados em paralelo; fundidos por RRF (ADR-0104) |
| `ef_search` por tenant | *Trade-off* recall×latência ajustável sem reindex (`memory.hnsw.ef_search`) |

> A qualidade de recuperação (**Memory Recall Rate ≥ 0,95**, NFR-002) NÃO DEVE ser
> sacrificada por escala: `ef_search` é o botão para equilibrar p99 vs. Recall@10
> por tenant, não um atalho para baixar recall globalmente.

---

## 8. Consolidação e poda em lote (crescimento sub-linear)

Jobs de consolidação/poda rodam **fora do caminho quente** (brief §10):

- Agendados pelo `RetentionScheduler` (cron interno + reação a eventos), idempotentes.
- Particionados por shard — paralelismo sem contenção cross-shard.
- Consolidação reduz cardinalidade (merge/dedup, FR-014) → menos vetores por tenant.
- Poda por decaimento/TTL/LRU/cota mantém `usage_ratio` < 1,0 (NFR-009).

O efeito combinado é **crescimento sub-linear** do estado por agente: sem
consolidação/poda, o número de itens cresceria linearmente com a atividade; com
elas, converge para o *working set* relevante.

---

## 9. Limites teóricos e gargalos previstos

| Recurso | Limite prático (por unidade) | Gargalo | Mitigação / caminho evolutivo |
|---------|------------------------------|---------|-------------------------------|
| Vetores por tenant | ~10^8 (NFR-012) | Tamanho do índice HNSW em RAM | Particionamento por tenant; índice externo (ADR-0101) |
| Escrita por shard | ~5.000/s (NFR-004) | I/O de PG + Outbox | Mais shards (aumentar N); *batch* de embeddings |
| Leitura por shard | ~2.000/s (NFR-004) | CPU de fusão/re-rank | Réplicas de leitura + cache Redis |
| Grafo (AGE) | travessia p99 ≤ 250 ms | Profundidade de travessia | Limitar profundidade; degradar p/ `hybrid` (F5) |
| Redis (hot) | RAM do cluster | *Working set* de agentes ativos | *Cold agents*; TTL curto (Working 15 min) |
| MinIO | capacidade S3 | Throughput de objeto | *Content-addressed* dedup; retenção alinhada |

---

## 10. Riscos e alternativas

| Risco | Impacto | Mitigação | Alternativa descartada |
|-------|---------|-----------|------------------------|
| *Hot tenant* satura um shard | contenção localizada | *Backpressure* por tenant; sub-sharding por agente; cotas (026) | Shard único global (rejeitado: SPOF/limite) |
| Índice HNSW estoura RAM em 10^8 vetores | p99 ANN degrada | Partição por tenant; índice externo (ADR-0101/ADR-0005) | Busca exata (rejeitado: viola NFR-003/012) |
| *Rehash* ao crescer N causa *downtime* | indisponibilidade | Migração online por tenant (dupla-escrita + cutover) | *Stop-the-world* (rejeitado: viola NFR-005) |
| Lock de consolidação vira gargalo | jobs enfileiram | Lock só por (tenant, agente); jobs em lote paralelos por shard | Lock global (rejeitado: serializa tudo) |
| Cache Redis inconsistente com PG | recall stale | TTL curto + invalidação por evento `stored`/`consolidated` | Cache sem invalidação (rejeitado: correção) |
| Reidratação de *cold agent* adiciona latência | p99 recall pior no reativo | Pré-aquecimento sob `lifecycle.suspended`; reidratação assíncrona | Manter tudo quente (rejeitado: inviável a milhões) |

---

## 11. Rastreabilidade

| Requisito | Coberto por (esta doc) |
|-----------|------------------------|
| NFR-004 (throughput por shard) | §2, §7, §9 |
| NFR-009 (crescimento controlado) | §6, §8 |
| NFR-012 (10^8 vetores/tenant) | §3, §9 |
| FR-008 (cotas + backpressure) | §6 |
| FR-012 (reidratação ARCHIVED) | §4 |
| FR-014 (dedup semântica) | §8 |

---

## 12. Referências

- [_DESIGN_BRIEF.md §10](./_DESIGN_BRIEF.md) — fonte única (sharding, locks, backpressure).
- [001-Architecture §12](../001-Architecture/Architecture.md) — modelo de shard global.
- [ADR.md](./ADR.md) — ADR-0101/0105/0106/0107/0108.
- [FailureRecovery.md](./FailureRecovery.md) — *bulkhead*, degradação, RTO/RPO.
- [Database.md](./Database.md) — particionamento físico e índices.
- [Configuration.md](./Configuration.md) — chaves `memory.hnsw.*`, `memory.quota.*`, `memory.backpressure.*`.
- Glossário: [Shard, Cold Agent, Backpressure, CQRS, Hot State](../040-Glossary/Glossary.md).
