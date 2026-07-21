---
Documento: NonFunctionalRequirements
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 021-Security, 022-Policy, 005-Database, 027-Cluster, 040-Glossary
---

# AIOS — Módulo 011 · Context — Requisitos Não-Funcionais

> Derivado de [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) §7.2. Os IDs
> `NFR-001..NFR-014` e suas metas (SLO/SLI) são **canônicos**: o valor que
> aparece aqui **DEVE** ser idêntico em [`Monitoring.md`](./Monitoring.md),
> [`Benchmark.md`](./Benchmark.md), [`FailureRecovery.md`](./FailureRecovery.md),
> [`Configuration.md`](./Configuration.md) e [`Scalability.md`](./Scalability.md).
> Nada aqui contradiz o brief. Palavras normativas conforme RFC 2119/8174.
> Métricas seguem o padrão `aios_<subsistema>_<nome>_<unidade>` (ver
> [`Metrics.md`](./Metrics.md)).

## 1. Convenções

- **SLO** = objetivo (meta); **SLI** = indicador medido; verificação liga-se a
  [`Testing.md`](./Testing.md)/[`Benchmark.md`](./Benchmark.md)/[`Monitoring.md`](./Monitoring.md).
- Latência em `ms` (`p50/p95/p99`); throughput em `req/s`; tamanho em `MiB/GiB`;
  custo em `USD` e `tokens`.
- Métricas-norte do módulo: **Context Compression Ratio** e **cache hit ratio**
  (`../040-Glossary/Glossary.md`), correlacionadas com a **Task Completion Rate**
  (métrica global, `024`/`035`).

## 2. Tabela consolidada de Requisitos Não-Funcionais

| ID | Atributo | Meta (SLO) | SLI / verificação |
|----|----------|-----------|-------------------|
| **NFR-001** | Desempenho — cache hit | latência `CacheLookup` p99 ≤ 20 ms. | histograma `aios_context_cache_lookup_duration_ms`; benchmark dedicado. |
| **NFR-002** | Desempenho — assembly (cache miss, sem sumarização) | latência `Assemble` p99 ≤ 150 ms. | `aios_context_assemble_duration_ms{path="miss_no_summarize"}`; teste de carga. |
| **NFR-003** | Desempenho — assembly (com 1 chamada de sumarização) | latência `Assemble` p99 ≤ 1200 ms. | `aios_context_assemble_duration_ms{path="miss_summarize"}`; teste de carga. |
| **NFR-004** | Eficácia de compressão | **Context Compression Ratio** médio ≥ 0,50. | `aios_context_compression_ratio` (histograma/gauge agregado). |
| **NFR-005** | Preservação de qualidade | Δ **Task Completion Rate** vs. baseline sem compressão ≥ −2% (absoluto). | benchmark de qualidade em `./Benchmark.md`, alinhado a `035-Benchmark`. |
| **NFR-006** | Eficiência de cache | cache hit ratio em regime permanente ≥ 0,35. | `hits/(hits+misses)` via `aios_context_cache_hit_total`/`aios_context_cache_miss_total`. |
| **NFR-007** | Economia de custo | custo evitado / custo bruto de inferência ≥ 0,25. | `aios_context_cache_cost_saved_usd` / custo estimado bruto. |
| **NFR-008** | Disponibilidade | uptime do serviço (mensal) ≥ 99,95%. | probes de liveness/readiness + *error budget* de 5xx; SLO burn-rate. |
| **NFR-009** | Escalabilidade | throughput de `assemble` ≥ 500 req/s por réplica. | teste de carga (`./Benchmark.md`, `./Scalability.md`). |
| **NFR-010** | Precisão de contagem de tokens | erro de estimativa vs. contagem exata ≤ 2%. | suite de contagem multi-tokenizer. |
| **NFR-011** | Precisão do cache semântico | taxa de falso-*hit* (resposta inadequada servida) ≤ 0,5%. | amostragem/avaliação offline com anotadores. |
| **NFR-012** | Recuperabilidade | RTO ≤ 5 min / RPO ≤ 60 s. | *chaos drill*; teste de kill de réplica/dependência. |
| **NFR-013** | Observabilidade | cobertura de traces OTel em 100% dos caminhos críticos (`assemble`, `cache lookup/store`, `compress`). | auditoria de spans; dashboard `./Monitoring.md`. |
| **NFR-014** | Isolamento multi-tenant | vazamento de cache entre tenants = 0. | testes de RLS + *fuzz* de `tenant_id`; auditoria de namespace de cache. |

## 3. Agrupamento por atributo de qualidade

| Atributo | NFRs | Observação |
|----------|------|------------|
| Performance / latência | NFR-001, NFR-002, NFR-003 | Três SLOs distintos por caminho: *hit*, *miss* sem sumarização, *miss* com 1 sumarização. |
| Qualidade funcional | NFR-004, NFR-005, NFR-011 | Compressão e cache não podem degradar a tarefa além do limite acordado. |
| Eficiência de custo | NFR-006, NFR-007 | Cache é a principal alavanca de redução de reinferência (missão do módulo, brief §1.1). |
| Disponibilidade / resiliência | NFR-008, NFR-012 | Detalhados em [`FailureRecovery.md`](./FailureRecovery.md); serviço stateless (brief §10). |
| Escalabilidade | NFR-009 | Sharding por `hash(tenant_id, agent_id)`; ver [`Scalability.md`](./Scalability.md). |
| Precisão | NFR-010, NFR-011 | Tokens e similaridade de cache são a base de todas as demais metas. |
| Observabilidade | NFR-013 | Traces OTel obrigatórios (RFC-0001 §5.6/§5.8). |
| Segurança / isolamento | NFR-014 | RLS por tenant; namespace de cache por tenant; ver [`Security.md`](./Security.md). |

## 4. Detalhamento e método de verificação

### NFR-001 / NFR-002 / NFR-003 — Latência por caminho
As três metas de latência **DEVEM** ser medidas separadamente por *label* de
caminho (`path`) no histograma `aios_context_assemble_duration_ms`, pois os
custos são estruturalmente diferentes: um *cache hit* (NFR-001) é dominado por
I/O de Redis (~sub-ms) ou busca HNSW em `pgvector`; um *miss* sem sumarização
(NFR-002) é dominado por recall ao `010` e ranking; um *miss* com sumarização
(NFR-003) inclui uma chamada síncrona ao modelo de sumarização roteado pelo
`017` (tipicamente um modelo barato, ADR-0115), cujo RTT domina o orçamento de
latência. Violação sustentada de qualquer um dos três SLOs por 15 minutos
**DEVE** disparar alerta (`./Monitoring.md`).

### NFR-004 / NFR-005 — Compressão sem regressão de qualidade
A **Context Compression Ratio** é definida como
`1 − tokens_in/tokens_raw` (idêntica ao campo `compression_ratio` de
`ContextBundle`, brief §3.1) e **DEVE** ter média ≥ 0,50 em regime permanente,
respeitando o piso configurável `context.compression.min_compression_ratio`
(default 0,30) por `BudgetProfile`. Entretanto, compressão agressiva sem
controle de qualidade é uma **não-meta**: a Δ Task Completion Rate frente a um
baseline sem compressão **NÃO DEVE** cair mais que 2 pontos percentuais
absolutos (NFR-005), medida no benchmark de tarefas do `035-Benchmark`. Estas
duas métricas **DEVEM** ser reportadas juntas — otimizar uma sem a outra é
considerado defeito de release (gate de qualidade, `./Testing.md`).

### NFR-006 / NFR-007 — Eficiência e economia do cache semântico
O cache semântico é a principal alavanca de redução de **reinferências**
(brief §1.1). Em regime permanente (após período de aquecimento configurável),
o *hit ratio* **DEVE** atingir ≥ 0,35 e a fração de custo evitado sobre custo
bruto estimado **DEVE** atingir ≥ 0,25. Ambas as métricas são acumuladas por
tenant e por `model_id`, permitindo detectar tenants/tarefas com baixa taxa de
reuso (candidatos a ajuste de `similarity_threshold` ou de `BudgetProfile`).

### NFR-008 / NFR-012 — Disponibilidade e recuperabilidade
O serviço **DEVE** manter uptime mensal ≥ 99,95% (~21,9 min de indisponibilidade
tolerada por mês). Por ser **stateless** (verdade em PostgreSQL, estado quente em
Redis — brief §10), a perda de uma réplica **NÃO DEVE** exigir RTO acima de
5 minutos, e a perda de dados em trânsito (bundles efêmeros, cache) **NÃO DEVE**
exceder RPO de 60 segundos, garantido pelo padrão **Outbox** transacional (FM-07,
brief §9) e pela natureza efêmera/reconstruível de `ContextBundle` e
`SemanticCacheEntry`.

### NFR-009 — Escalabilidade horizontal
Cada réplica stateless **DEVE** sustentar ≥ 500 req/s de `assemble` sob carga
mista (hit/miss), com sharding de cache/bundles por
`hash(tenant_id, agent_id) mod context.shard.count` (brief §10). O modelo de
escala progressivo (1 → 10³ → 10⁵ → 10⁶⁺ agentes) está detalhado em
[`Scalability.md`](./Scalability.md).

### NFR-010 / NFR-011 — Precisão de tokens e de cache
A estimativa de tokens **DEVE** ter erro ≤ 2% frente à contagem exata do
tokenizer do provedor (fallback quando o tokenizer exato não está disponível
via `017`). A taxa de falso-*hit* do cache semântico — servir uma resposta
semanticamente inadequada por *hit* incorreto — **DEVE** permanecer ≤ 0,5%,
verificada por amostragem periódica com avaliação humana/LLM-juiz offline;
violação sustentada aciona o modo de falha FM-08 (cache poisoning, brief §9) e
eleva `similarity_threshold`.

### NFR-013 / NFR-014 — Observabilidade e isolamento
Todo caminho crítico (`assemble`, `cache lookup/store`, `compress`,
`invalidate`) **DEVE** produzir um trace OTel com `tenant_id`, `trace_id` e
`bundle_id`/`cache_id` como atributos (RFC-0001 §5.6). O isolamento
multi-tenant **DEVE** ser absoluto: **zero** vazamento de cache entre tenants,
garantido por RLS em todas as tabelas (brief §3), namespace de cache por
`tenant_id` e verificação por *fuzz testing* de `tenant` divergente
(`AIOS-CTX-0011`).

## 5. Exemplo — verificação de NFR-004/NFR-006 (compressão e cache)

```
# Agregação diária (Prometheus recording rule):
aios_context_compression_ratio:avg1d
  = avg_over_time(aios_context_compression_ratio[1d])
GATE: aios_context_compression_ratio:avg1d >= 0.50   # senão falha o release gate

aios_context_cache_hit_ratio:avg1d
  = sum(rate(aios_context_cache_hit_total[1d]))
    / (sum(rate(aios_context_cache_hit_total[1d])) + sum(rate(aios_context_cache_miss_total[1d])))
GATE: aios_context_cache_hit_ratio:avg1d >= 0.35      # em regime permanente (pós-aquecimento)

# Alerta (./Monitoring.md):
ALERT ContextCompressionRatioBelowSLO
  IF aios_context_compression_ratio:avg1d < 0.50 for 6h
  THEN page on-call + inspecionar BudgetProfile e amostra de bundles recentes.

ALERT ContextCacheHitRatioBelowSLO
  IF aios_context_cache_hit_ratio:avg1d < 0.35 for 6h
  THEN page on-call + revisar similarity_threshold e padrões de prompt por tenant.
```

## 6. Rastreabilidade e riscos

Cada NFR liga-se a componentes de [`Architecture.md`](./Architecture.md) §2 e a
testes em [`Testing.md`](./Testing.md)/[`Benchmark.md`](./Benchmark.md); a matriz
consolidada está em [`Requirements.md`](./Requirements.md) §4.

| Risco | NFR afetado | Mitigação | ADR |
|-------|-------------|-----------|-----|
| Sumarização agressiva reduz Task Completion Rate | NFR-004, NFR-005 | Piso `min_compression_ratio` + preservação de itens `relevance_score ≥ p90`; benchmark de qualidade como gate. | ADR-0110, ADR-0115 |
| Cache semântico serve resposta inadequada (falso-*hit*) | NFR-011 | Limiar de similaridade calibrado + auditoria de amostragem + invalidação event-driven. | ADR-0111, ADR-0112, ADR-0119 |
| Vazamento de cache entre tenants | NFR-014 | RLS + namespace por tenant + testes de *fuzz*. | ADR-0111, ADR-0119 |
| Latência de sumarização eleva p99 de `assemble` | NFR-003 | Roteamento a modelo barato via `017`; fallback truncate determinístico. | ADR-0113, ADR-0115, ADR-0118 |
| Perda de nó degrada disponibilidade | NFR-008, NFR-012 | Serviço stateless + Outbox + réplicas sem afinidade. | ADR-0111, ADR-0116 |
| Estimativa de tokens imprecisa causa `BudgetInfeasible` espúrio | NFR-010 | Registro de tokenizers multi-provedor com contagem exata quando disponível. | ADR-0114 |
