---
Documento: NonFunctionalRequirements
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007, ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 024-Observability, 026-Cost-Optimizer, 027-Cluster, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Requisitos Não-Funcionais

> Derivado de `./_DESIGN_BRIEF.md` §7.2. Os IDs `NFR-001..NFR-013` e suas metas
> (SLO/SLI) são **canônicos**: o valor que aparece aqui **DEVE** ser idêntico em
> `./Monitoring.md`, `./Benchmark.md`, `./FailureRecovery.md`, `./Configuration.md`
> e `./Scalability.md`. Nada aqui contradiz o brief. Palavras normativas conforme
> RFC 2119/8174. Métricas seguem o padrão `aios_<subsistema>_<nome>_<unidade>`.

## 1. Convenções

- **SLO** = objetivo (meta); **SLI** = indicador medido; verificação liga-se a `./Testing.md`/`./Benchmark.md`/`./Monitoring.md`.
- Latência em `ms` (`p50/p95/p99`); throughput em `req/s`; tamanho em `MiB/GiB`.
- Métrica-norte do módulo: **Memory Recall Rate** (`../040-Glossary/Glossary.md`).

## 2. Tabela consolidada de Requisitos Não-Funcionais

| ID | Atributo | Meta (SLO) | SLI / verificação |
|----|----------|-----------|-------------------|
| **NFR-001** | Latência `remember` | p99 ≤ 30 ms (inline, sem embedding); ≤ 120 ms (com embedding). | histograma `aios_memory_remember_duration_ms`. |
| **NFR-002** | Qualidade de recuperação | **Memory Recall Rate ≥ 0,95**; Recall@10 ANN ≥ 0,95 vs. exata. | `aios_memory_recall_rate` (bench + amostragem produção). |
| **NFR-003** | Latência `recall` | p99 ≤ 50 ms (Working/hot Redis); ≤ 150 ms (Semantic/Episodic ANN); ≤ 250 ms (grafo). | `aios_memory_recall_duration_ms` por `mode`. |
| **NFR-004** | Throughput | ≥ 5.000 `remember`/s e ≥ 2.000 `recall`/s por shard. | teste de carga (`./Benchmark.md`). |
| **NFR-005** | Disponibilidade | ≥ 99,95% do serviço (control plane). | uptime SLI; *error budget* 5xx. |
| **NFR-006** | Durabilidade (camadas duráveis) | RPO ≤ 5 min (Long-Term/Semantic/Procedural/Episodic/KG). | replicação PG streaming; teste de DR. |
| **NFR-007** | RTO | ≤ 15 min para o serviço de memória. | *drill* de failover. |
| **NFR-008** | Working Memory | perda tolerada (efêmera); RPO não garantido; reconstruível. | política declarada; teste de kill. |
| **NFR-009** | Crescimento controlado | uso por tenant **NÃO DEVE** exceder a cota; poda mantém ≤ 100% da cota. | `aios_memory_usage_ratio` < 1,0. |
| **NFR-010** | Isolamento | zero vazamento cross-tenant. | teste de RLS + *fuzz* de tenant. |
| **NFR-011** | Consolidação sem regressão | Recall Rate pós ≥ pré − 0,01, senão rollback automático. | comparação A/B por versão. |
| **NFR-012** | Escala vetorial | ≥ 10^8 vetores por tenant com p99 ANN ≤ 150 ms. | bench pgvector/HNSW + sharding. |
| **NFR-013** | Observabilidade | 100% das operações com trace OTel + `tenant_id`/`trace_id`. | auditoria de spans. |

## 3. Agrupamento por atributo de qualidade

| Atributo | NFRs | Observação |
|----------|------|------------|
| Performance / latência | NFR-001, NFR-003 | Metas por caminho (inline/embedding; hot/ANN/grafo). |
| Qualidade funcional | NFR-002, NFR-011 | Recall Rate norteia recuperação **e** consolidação. |
| Escalabilidade | NFR-004, NFR-012 | Por shard e por tenant; detalhado em `./Scalability.md`. |
| Disponibilidade / resiliência | NFR-005, NFR-006, NFR-007, NFR-008 | RTO/RPO detalhados em `./FailureRecovery.md`. |
| Custo / crescimento | NFR-009 | Cotas + poda; reporta ao 026 (`./Configuration.md`). |
| Segurança / privacidade | NFR-010 | RLS por tenant; ver `./Security.md`. |
| Observabilidade | NFR-013 | Traces OTel obrigatórios (RFC-0001 §5.6/§5.8). |

## 4. Detalhamento e método de verificação

### NFR-001 / NFR-003 — Latência
As metas **DEVEM** ser medidas por percentil `p99` no `MemoryTelemetry`. O caminho
`remember` sem embedding (inline) tem SLO mais estrito (≤ 30 ms) que o caminho com
embedding (≤ 120 ms), pois este depende de RTT ao Model Router (017). O `recall`
tem três SLOs por `mode`: Redis quente (≤ 50 ms), ANN (≤ 150 ms) e grafo (≤ 250 ms).
Verificação: histogramas `aios_memory_remember_duration_ms` e
`aios_memory_recall_duration_ms` com alertas em `./Monitoring.md`.

### NFR-002 / NFR-011 — Recall Rate e consolidação sem regressão
A **Memory Recall Rate** **DEVE** permanecer ≥ 0,95, medida por bench offline e
amostragem em produção (`aios_memory_recall_rate`). Uma consolidação que reduza a
taxa além de 0,01 face à pré-imagem **DEVE** disparar rollback automático (liga-se a
FR-007 e ao modo de falha F6 em `./FailureRecovery.md`).

### NFR-004 / NFR-012 — Throughput e escala vetorial
Alvos de ≥ 5.000 `remember`/s e ≥ 2.000 `recall`/s **por shard**, e ≥ 10^8 vetores
por tenant com p99 ANN ≤ 150 ms. Verificação por teste de carga e bench
`pgvector`/HNSW com sharding (`./Benchmark.md`, `./Scalability.md`).

### NFR-005..NFR-008 — Disponibilidade, durabilidade, RTO/RPO
Serviço com disponibilidade ≥ 99,95%. Camadas duráveis com RPO ≤ 5 min e RTO
≤ 15 min (failover para standby, ver `../027-Cluster/` e brief §9). A **Working Memory**
é explicitamente efêmera (NFR-008): perda tolerada, reconstruível — não conta para
RPO. Verificação por *drills* de failover e teste de kill.

### NFR-009 / NFR-010 / NFR-013 — Custo, isolamento, observabilidade
- NFR-009: `aios_memory_usage_ratio` **DEVE** ficar < 1,0; poda pelo ForgettingEngine mantém uso ≤ 100% da cota.
- NFR-010: **zero** vazamento cross-tenant, garantido por RLS + guarda de tenant (`AIOS-MEM-0005`); verificado por *fuzz* de tenant.
- NFR-013: **100%** das operações com trace OTel carregando `tenant_id`/`trace_id` (RFC-0001 §5.6).

## 5. Exemplo — verificação de NFR-002 (Recall Rate)

```
# Bench offline (./Benchmark.md): dataset rotulado com relevância conhecida.
recall_rate = itens_relevantes_recuperados / itens_relevantes_esperados
GATE: recall_rate >= 0.95           # senão o build de qualidade falha

# Produção (amostragem): MemoryTelemetry emite
aios_memory_recall_rate{tenant="acme", mode="hybrid"} = 0.962   # OK (≥ 0,95)

# Alerta (./Monitoring.md):
ALERT MemoryRecallRateBelowSLO
  IF aios_memory_recall_rate < 0.95 for 15m
  THEN page on-call + inspecionar consolidações recentes (NFR-011).
```

## 6. Rastreabilidade e riscos

Cada NFR liga-se a componentes de `./Architecture.md` §5 e a testes em `./Testing.md`
/`./Benchmark.md`; a matriz consolidada está em `./Requirements.md`.

| Risco | NFR afetado | Mitigação | ADR |
|-------|-------------|-----------|-----|
| ANN degrada além do SLO em alta escala | NFR-003, NFR-012 | HNSW por partição, `ef_search` por tenant, sharding | ADR-0101, ADR-0108 |
| Consolidação regride Recall | NFR-002, NFR-011 | Guarda + rollback automático | ADR-0102 |
| Cota estoura e custo cresce | NFR-009 | Backpressure + poda | ADR-0103, ADR-0105 |
| Falha de datastore quebra durabilidade | NFR-006, NFR-007 | Failover standby + Outbox reprocessável | ADR-0007 |
