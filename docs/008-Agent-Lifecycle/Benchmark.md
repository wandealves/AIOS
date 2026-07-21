---
Documento: Benchmark
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (FSM canônica), ADR-0081 (Hibernação), ADR-0083 (WarmPool), ADR-0084 (Lease/fencing), ADR-0085 (Migração/saga)
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (a propor — Cold Agent Hibernation & Checkpoint/Restore Protocol)
Depende de: 035-Benchmark, 024-Observability, 009-Scheduler, 007-Agent-Runtime, 027-Cluster
---

# 008-Agent-Lifecycle — Benchmark

> Este documento define a **metodologia de benchmark** do módulo 008 — como
> medir, reproduzir e comparar seu desempenho contra as metas normativas do
> `_DESIGN_BRIEF.md` §7.2 (NFR-001..NFR-013) e §10 (escalabilidade rumo a
> milhões de agentes). Segue o arcabouço metodológico geral de
> `../035-Benchmark/README.md`, especializado para spawn, hibernação,
> checkpoint/restore e migração.

## 1. Objetivo

Prover uma metodologia **reprodutível** para responder, a cada release:

1. O módulo 008 atinge `p99 ≤ 250 ms` (`p50 ≤ 80 ms`) de spawn/materialização
   sob carga realista, com e sem WarmPool (NFR-001)?
2. O módulo sustenta `≥ 50.000 transições/s` por cluster (NFR-008)?
3. O checkpoint mantém `p99 ≤ 500 ms` para working memory de até 64 MiB
   (NFR-003), e a migração `p99 ≤ 2 s` no mesmo DC (NFR-010)?
4. O sistema sustenta `≥ 10⁶` agentes com `≤ 5%` ativos em RAM
   simultaneamente, com `wake` sub-250 ms mesmo nessa escala (NFR-004, §10 do
   brief)?
5. A serialização por lease (§10 do brief) escala horizontalmente por shard
   sem lock global, mantendo baixa contenção sob concorrência adversarial?

## 2. Metodologia

### 2.1 Princípios

- **Reprodutibilidade em primeiro lugar**: toda carga DEVE ser expressa como
  script versionado (k6/NBomber/ghz) com seed determinístico para geração de
  dados sintéticos; resultados DEVEM incluir hash do commit do módulo 008
  testado e hash do script de carga.
- **Isolamento de variável**: cada benchmark varia **uma** dimensão por vez
  (ex.: apenas RPS de spawn, ou apenas nº de agentes hibernados, ou apenas
  tamanho de working memory) para permitir atribuição causal de regressões.
- **Ambiente representativo, não de produção**: ambiente `staging` dedicado a
  benchmark (ver `../035-Benchmark/README.md`), dimensionado de forma
  documentada e estável entre execuções (mesma classe de VM/CPU/rede/disco).
- **Warm-up obrigatório**: 2 minutos de carga descartada antes da coleta,
  para estabilizar pools de conexão, cache de decisão de política e JIT/AOT
  do .NET 10.

### 2.2 Ambiente de Referência

| Componente | Especificação de referência |
|------------|-------------------------------|
| Agent Lifecycle Service | 1 réplica, 4 vCPU / 8 GiB RAM, .NET 10 AOT, sem *co-location* com dependências |
| PostgreSQL 16 + pgvector + Apache AGE | 4 vCPU / 16 GiB RAM, SSD NVMe, streaming replication desabilitada (medição de nó único; HA medido separadamente em `Scalability.md`) |
| Redis | 2 vCPU / 4 GiB RAM, modo standalone (ACB quente + leases) |
| MinIO | 4 vCPU / 8 GiB RAM, SSD NVMe, single-node (armazenamento de blobs de checkpoint) |
| NATS/JetStream | 3 nós (cluster mínimo), armazenamento em disco |
| Dependências simuladas (`006`,`007`,`009`,`022`,`027`) | *Stub services* com latência configurável (distribuição realista, não latência zero) para isolar o desempenho do módulo 008 do desempenho de terceiros |
| Rede | Mesma zona de disponibilidade (latência de rede interna < 1 ms p50) para migração "mesmo DC"; par de zonas para migração cross-AZ (workload W6) |
| Gerador de carga | k6 (REST) / ghz ou NBomber (gRPC), executando fora da réplica do módulo 008 |

### 2.3 Cargas de Trabalho (Workload Mix)

| Carga | Descrição | Mix de operações | Uso |
|-------|-----------|-----------------|-----|
| **W1 — Transições de controle puro** | Alta taxa de `suspend`/`resume`/`GetState`, sem I/O de checkpoint/migração. | 40% `GetState`, 30% `suspend`, 30% `resume` | Mede NFR-002, NFR-008 (piso). |
| **W2 — Spawn em rajada** | Simula *burst* de criação de agentes (ex.: início de um workflow com sub-agentes), com e sem WarmPool aquecido. | 100% `spawn`, em rajadas de 100–1000 concorrentes, variando `warmpool.min_per_shard` | Mede NFR-001 (p50/p99), FR-014. |
| **W3 — Mix realista de produção** | Distribuição observada em ambientes de referência (estimada até haver telemetria real). | 15% `spawn`, 10% `terminate`, 20% `suspend`/`resume`, 25% `hibernate`, 20% `wake`, 5% `migrate`, 5% `checkpoint` | Mede NFR-008 sob mix heterogêneo. |
| **W4 — Checkpoint/restore por tamanho** | Varia o tamanho da working memory (1 MiB → 256 MiB) medindo duração de checkpoint/restore. | 100% `checkpoint` seguido de `restore` | Mede NFR-003 e sensibilidade a `checkpoint.codec`. |
| **W5 — Migração sob concorrência** | Migrações concorrentes por nó, próximo ao limite de `migration.max_concurrent_per_node`. | 100% `migrate`, mesmo DC e cross-AZ | Mede NFR-010, saturação de rede/storage (§10 do brief). |
| **W6 — Escala de agentes hibernados** | População crescente de agentes majoritariamente `Hibernated` (10³ → 10⁶), medindo custo de `wake`/`GetState` sob essa população. | `wake` esporádico + `GetState` de amostra aleatória | Mede NFR-004, NFR-011 e sub-linearidade do sharding (§10 do brief). |
| **W7 — Contenção de lease adversarial** | Múltiplos clientes emitindo comandos concorrentes para o mesmo `agent_id`. | 100% operações mutantes concorrentes no mesmo agente | Mede `aios_lifecycle_lease_contention_ratio`, correção de FR-010. |

### 2.4 KPIs Coletados

| KPI | Fonte (métrica) | Unidade |
|-----|-------------------|---------|
| Latência p50/p95/p99 de spawn/wake | `aios_lifecycle_spawn_duration_ms`, `aios_lifecycle_wake_duration_ms` | ms |
| Latência p50/p95/p99 de transição de controle | `aios_lifecycle_transition_duration_ms` | ms |
| Latência p50/p95/p99 de checkpoint/restore | `aios_lifecycle_checkpoint_duration_ms`, `aios_lifecycle_restore_duration_ms` | ms |
| Latência p50/p95/p99 de migração | `aios_lifecycle_migration_duration_ms` | ms |
| Throughput sustentado (transições/s) | `rate(aios_lifecycle_transitions_total[1m])` | transições/s |
| Taxa de erro por código | `aios_lifecycle_operations_total{status="error"}` / total | % |
| % de agentes em `Hibernated` sob população-alvo | `aios_lifecycle_agents_total{state="Hibernated"} / aios_lifecycle_agents_total` | % |
| RSS agregado de agentes hibernados | `aios_lifecycle_cold_agents_ram_bytes` | MiB |
| Tamanho do índice quente por agente frio | `aios_lifecycle_hot_index_size_bytes` | KiB |
| Contenção de lease | `aios_lifecycle_lease_contention_ratio` | % |
| Lag do Outbox sob carga | `aios_lifecycle_outbox_lag_ms` | ms |
| Utilização de CPU/RAM da réplica | métricas de infraestrutura (cAdvisor/node-exporter) | % / MiB |
| Custo por 1k materializações (infra) | derivado (custo da réplica ÷ throughput) | USD |

## 3. Resultados-Alvo (Metas de Aceitação)

> Uma execução de benchmark **PASSA** se, para a carga correspondente,
> **todos** os KPIs abaixo forem atendidos com a réplica de referência
> (§2.2).

| Carga | KPI | Meta | Rastreia |
|-------|-----|------|----------|
| W1 | p99 latência de transição de controle | ≤ 20 ms | NFR-002 |
| W1 | Throughput sustentado | ≥ 50.000 transições/s | NFR-008 |
| W2 | p99 latência `spawn` (WarmPool aquecido) | ≤ 250 ms, p50 ≤ 80 ms | NFR-001 |
| W2 | p99 latência `spawn` (WarmPool frio/miss) | ≤ 1500 ms (`spawn.timeout_ms`) | limite de degradação aceitável |
| W3 | Throughput sustentado (mix realista) | ≥ 50.000 transições/s por cluster | NFR-008 |
| W3 | Taxa de erro | ≤ 0,1% | qualidade geral |
| W4 | p99 duração de checkpoint (working memory ≤ 64 MiB) | ≤ 500 ms | NFR-003 |
| W4 | Integridade de checkpoint/restore | 0 corrupção não detectada | NFR-009 |
| W5 | p99 duração de migração (agente ≤ 64 MiB, mesmo DC) | ≤ 2 s | NFR-010 |
| W5 | Perda de trabalho aceito durante migração | 0 | invariante de migração |
| W6 | % de agentes em `Hibernated` a 10⁶ | ≥ 95% | NFR-004 |
| W6 | Latência de `wake` com 10⁶ agentes (90% `Hibernated`) | degradação ≤ 10% vs. baseline com 10³ agentes | NFR-004, sub-linearidade (§10 do brief) |
| W6 | Tamanho do índice quente por agente frio | ≤ 4 KiB | NFR-011 |
| W7 | Contenção de lease sob concorrência adversarial | rejeições esperadas (não é falha), mas 0 violação de serialização (INV1) | FR-010 |
| Todas | Lag do Outbox sob carga sustentada | ≤ `outbox.publish_batch`-equivalente em tempo, proxy de saúde | FR-008 (proxy de saúde, não de perda) |

## 4. Comparação com Baseline

- Todo release candidato DEVE ser comparado contra o **baseline da última
  versão `Stable`** publicada, usando o mesmo ambiente de referência (§2.2) e
  os mesmos scripts de carga (versionados).
- Regressão de p99 > **10%** em qualquer KPI da §3, sem justificativa de
  trade-off documentada (ex.: nova cifragem com custo aceito), **DEVE**
  bloquear o release (gate de qualidade, ver `./Testing.md` §7).
- Resultados históricos DEVEM ser armazenados (série temporal) para permitir
  visualizar tendência de degradação/melhoria entre releases — dashboard
  dedicado em `../035-Benchmark/README.md`.

| Release | Data | p99 spawn (ms) | p99 transição (ms) | p99 checkpoint (ms) | p99 migração (ms) | % frios @10⁶ | Observações |
|---------|------|------------------|----------------------|------------------------|----------------------|----------------|--------------|
| `baseline` (v0.1, projeção pré-implementação) | 2026-07-20 | 250 (meta) | 20 (meta) | 500 (meta) | 2000 (meta) | ≥95% (meta) | Metas normativas do brief; ainda sem execução real — ver §6. |

> **Nota de honestidade metodológica.** Como o módulo 008-Agent-Lifecycle está
> em fase de especificação (`Status: Draft`), a tabela acima registra
> **metas**, não **medições**. A primeira execução real do benchmark DEVE
> ocorrer contra uma implementação de referência e popular esta tabela com
> dados reais antes que o documento avance para `Status: Review`.

## 5. Reprodutibilidade

Todo relatório de benchmark DEVE incluir:

| Item obrigatório | Descrição |
|--------------------|-----------|
| Hash do commit do módulo 008 | Versão exata testada (SemVer + SHA). |
| Hash/versão dos scripts de carga | Script k6/NBomber/ghz versionado em `../035-Benchmark/`. |
| Especificação do ambiente | Conforme §2.2, incluindo classe de VM/CPU/rede/disco real usada (pode divergir do "de referência" — deve ser declarado). |
| Duração do warm-up e da coleta | Mínimo 2 min warm-up + 10 min de coleta estável (30 min para W6). |
| Seed de geração de dados sintéticos | Para reprodutibilidade exata do mix de tenants/agentes/tamanhos de working memory. |
| Configuração de `lifecycle.*` usada | Todas as chaves da `_DESIGN_BRIEF.md` §8 relevantes (ex.: `warmpool.min_per_shard`, `checkpoint.codec`, `hibernation.idle_ttl_s`, `lease.ttl_ms`). |
| Estado das dependências simuladas | Distribuição de latência configurada nos *stubs* de `006`/`007`/`009`/`022`/`027`. |

## 6. Execução (Pipeline de Benchmark)

```
Release candidato taggeado
        │
        ▼
Provisiona ambiente de referência (§2.2) — infraestrutura como código
        │
        ▼
Executa W1..W7 sequencialmente (isolamento de variável) ── warm-up 2min cada
        │
        ▼
Coleta KPIs via OTel Collector → Prometheus (retenção de longo prazo)
        │
        ▼
Compara contra baseline anterior (§4) ── regressão > 10%? ──sim──▶ bloqueia release
        │ não
        ▼
Publica relatório versionado em ../035-Benchmark/results/008-agent-lifecycle/<versao>.md
        │
        ▼
Atualiza tabela de comparação histórica (§4) e dashboard de tendência
```

## 7. Riscos e Limitações Conhecidas

| Risco | Mitigação |
|-------|-----------|
| *Stubs* de dependência não refletem latência real de `009`/`007` em produção | Calibrar periodicamente a distribuição de latência dos stubs com dados reais de `Monitoring.md` desses módulos. |
| Benchmark de W6 (10⁶ agentes) é caro e demorado — pode não rodar em toda release | Executar W6 completo trimestralmente; execuções de release usam amostra reduzida (10⁴–10⁵) com extrapolação documentada. |
| Ambiente de benchmark divergente do de produção (topologia de rede, HA, distribuição real de tamanho de working memory) | Declarar divergências explicitamente no relatório (§5); resultados são direcionais, não uma garantia de SLA em produção. |
| Migração cross-AZ tem variabilidade de rede maior que o ambiente de benchmark único-DC | Medir W5 separadamente em topologia cross-AZ real quando disponível (ver `Scalability.md`); registrar como cenário adicional não coberto pela meta mesmo-DC. |
| Contenção de "vizinho ruidoso" na infraestrutura de nuvem compartilhada | Usar instâncias dedicadas/isoladas para o ambiente de benchmark, não compartilhadas com outras cargas. |

## 8. Referências

- Metodologia geral de benchmark do AIOS: `../035-Benchmark/README.md`
- Metas normativas de origem: `_DESIGN_BRIEF.md` §7.2 (NFR), §10 (escalabilidade)
- Catálogo de métricas usado como KPI: `./Metrics.md`
- Estratégia de testes de carga complementar (gate de CI): `./Testing.md`
- Modelo de escalabilidade e sharding: `./Scalability.md`
- Dashboards para acompanhamento contínuo: `./Monitoring.md`
