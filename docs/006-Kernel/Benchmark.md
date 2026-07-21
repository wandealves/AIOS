---
Documento: Benchmark
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0061 (ACB/OCC), ADR-0062 (Cotas), ADR-0063 (PEP/PDP), ADR-0065 (Sharding), ADR-0066 (Outbox)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 035-Benchmark, 024-Observability, 009-Scheduler, 022-Policy
---

# 006-Kernel — Benchmark

> Este documento define a **metodologia de benchmark** do Kernel — como medir,
> reproduzir e comparar seu desempenho contra as metas normativas do
> `_DESIGN_BRIEF.md` §7.2 (NFR-001..NFR-012) e §10 (escalabilidade). Segue o
> arcabouço metodológico geral de `../035-Benchmark/README.md`, especializado
> para a superfície de syscalls do Kernel.

## 1. Objetivo

Prover uma metodologia **reprodutível** para responder, a cada release:

1. O Kernel atinge p99 de `spawn` ≤ 250 ms e de syscalls de controle ≤ 20 ms
   sob carga realista (NFR-001, NFR-002)?
2. O Kernel sustenta ≥ 20.000 syscalls/s por réplica (NFR-004)?
3. O PEP mantém decisão ≤ 8 ms (cache hit) / ≤ 25 ms (cache miss) sob carga
   concorrente (NFR-003)?
4. O sistema de cotas mantém overshoot ≤ 1% sob concorrência adversarial
   (NFR-009)?
5. O Kernel sustenta ≥ 10⁶ ACBs majoritariamente `Hibernated` com custo
   sub-linear (NFR-006, §10 do brief)?

## 2. Metodologia

### 2.1 Princípios

- **Reprodutibilidade em primeiro lugar**: toda carga DEVE ser expressa como
  script versionado (k6/NBomber/ghz) com seed determinístico para geração de
  dados sintéticos; resultados DEVEM incluir hash do commit do Kernel testado
  e hash do script de carga.
- **Isolamento de variável**: cada benchmark varia **uma** dimensão por vez
  (ex.: apenas RPS, ou apenas nº de ACBs residentes, ou apenas taxa de
  cache-miss do PEP) para permitir atribuição causal de regressões.
- **Ambiente representativo, não de produção**: ambiente `staging` dedicado a
  benchmark (ver `../035-Benchmark/README.md`), dimensionado de forma
  documentada e estável entre execuções (mesma classe de VM/CPU/rede).
- **Warm-up obrigatório**: 2 minutos de carga descartada antes da coleta, para
  estabilizar caches de decisão do PEP, pools de conexão e JIT/AOT do .NET 10.

### 2.2 Ambiente de Referência

| Componente | Especificação de referência |
|------------|-------------------------------|
| Kernel Service | 1 réplica, 4 vCPU / 8 GiB RAM, .NET 10 AOT, sem *co-location* com dependências |
| PostgreSQL 16 + pgvector + Apache AGE | 4 vCPU / 16 GiB RAM, SSD NVMe, streaming replication desabilitada (medição de nó único; HA medido separadamente em `Scalability.md`) |
| Redis | 2 vCPU / 4 GiB RAM, modo standalone |
| NATS/JetStream | 3 nós (cluster mínimo), armazenamento em disco |
| Dependências simuladas (`009`,`022`,`008`,`010`,`011`,`012`,`015`,`017`) | *Stub services* com latência configurável (distribuição realista, não latência zero) para isolar o desempenho do Kernel do desempenho de terceiros |
| Rede | Mesma zona de disponibilidade (latência de rede interna < 1 ms p50) |
| Gerador de carga | k6 (REST) / ghz ou NBomber (gRPC), executando fora da réplica do Kernel |

### 2.3 Cargas de Trabalho (Workload Mix)

| Carga | Descrição | Mix de verbos | Uso |
|-------|-----------|-----------------|-----|
| **W1 — Controle puro** | Alta taxa de `suspend`/`resume`/`kill`/`get_quota`, sem I/O de recurso. | 40% `get_quota`, 20% `suspend`, 20% `resume`, 20% `kill` | Mede NFR-002, NFR-004 (piso). |
| **W2 — Spawn em rajada** | Simula *burst* de criação de agentes (ex.: início de um workflow com sub-agentes). | 100% `spawn` em rajadas de 100–1000 concorrentes | Mede NFR-001, saga de admissão (ADR-0064). |
| **W3 — Mix realista de produção** | Distribuição observada em ambientes de referência (estimada até haver telemetria real). | 10% `spawn`, 5% `kill`, 10% `suspend`/`resume`, 15% `get_quota`, 20% `remember`, 15% `recall`, 10% `plan`, 10% `invoke_tool`, 5% `route_model` | Mede NFR-004 sob mix heterogêneo. |
| **W4 — Cota adversarial** | Múltiplos agentes do mesmo tenant competindo pelo mesmo `subject_urn` de cota, perto do limite. | 100% syscalls que consomem cota (tokens/tools) | Mede NFR-009 (overshoot). |
| **W5 — PEP sob pressão de cache-miss** | TTL do cache de decisão reduzido artificialmente / alta diversidade de capabilities. | Syscalls privilegiadas variadas com `kernel.pep.decision_cache_ttl_ms=0` | Mede NFR-003 (pior caso, cache miss). |
| **W6 — Escala de ACBs hibernados** | População crescente de ACBs majoritariamente `Hibernated` (10³ → 10⁶), medindo custo de `get_quota`/`resume` sob essa população. | `resume` esporádico + `get_quota` de amostra aleatória | Mede NFR-006 e §10 (sharding). |

### 2.4 KPIs Coletados

| KPI | Fonte (métrica) | Unidade |
|-----|-------------------|---------|
| Latência p50/p95/p99 por verbo | `aios_kernel_syscall_latency_ms`, `aios_kernel_spawn_latency_ms` | ms |
| Throughput sustentado (RPS) | `rate(aios_kernel_syscall_total[1m])` | syscalls/s |
| Taxa de erro | `aios_kernel_syscall_total{status="error"}` / total | % |
| Latência de decisão do PEP (hit/miss) | `aios_kernel_pep_decision_ms` | ms |
| Overshoot de cota | `aios_kernel_quota_overshoot_ratio` | % |
| Conflitos de OCC / s | `aios_kernel_occ_conflicts_total` | conflitos/s |
| Lag do Outbox sob carga | `aios_kernel_outbox_lag_ms` | ms |
| Utilização de CPU/RAM da réplica | métricas de infraestrutura (cAdvisor/node-exporter) | % / MiB |
| Custo por 1k syscalls (infra) | derivado (custo da réplica ÷ throughput) | USD |

## 3. Resultados-Alvo (Metas de Aceitação)

> Uma execução de benchmark **PASSA** se, para a carga correspondente, **todos**
> os KPIs abaixo forem atendidos com a réplica de referência (§2.2).

| Carga | KPI | Meta | Rastreia |
|-------|-----|------|----------|
| W1 | p99 latência controle | ≤ 20 ms | NFR-002 |
| W1 | Throughput sustentado | ≥ 20.000 syscalls/s | NFR-004 |
| W2 | p99 latência `spawn` | ≤ 250 ms | NFR-001 |
| W2 | Taxa de `Failed` por timeout de admissão sob rajada de 1000 | ≤ 0,5% | robustez da saga (ADR-0064) |
| W3 | Throughput sustentado (mix realista) | ≥ 20.000 syscalls/s por réplica | NFR-004 |
| W3 | Taxa de erro | ≤ 0,1% | qualidade geral |
| W4 | Overshoot de cota sob concorrência | ≤ 1% | NFR-009 |
| W5 | p99 decisão PEP (cache miss) | ≤ 25 ms | NFR-003 |
| W6 | Latência de `get_quota`/`resume` com 10⁶ ACBs (90% `Hibernated`) | degradação ≤ 10% vs. baseline com 10³ ACBs | NFR-006, sub-linearidade (§10 do brief) |
| Todas | Lag do Outbox sob carga sustentada | ≤ `kernel.outbox.relay_interval_ms` × 5 | NFR-007 (proxy de saúde, não de perda) |

## 4. Comparação com Baseline

- Todo release candidato DEVE ser comparado contra o **baseline da última
  versão `Stable`** publicada, usando o mesmo ambiente de referência (§2.2) e
  os mesmos scripts de carga (versionados).
- Regressão de p99 > **10%** em qualquer KPI da §3, sem justificativa de
  trade-off documentada (ex.: nova feature de segurança com custo aceito),
  **DEVE** bloquear o release (gate de qualidade, ver `./Testing.md` §7).
- Resultados históricos DEVEM ser armazenados (série temporal) para permitir
  visualizar tendência de degradação/melhoria entre releases — dashboard
  dedicado em `../035-Benchmark/README.md`.

| Release | Data | p99 spawn (ms) | p99 controle (ms) | RPS sustentado | Overshoot cota | Observações |
|---------|------|------------------|----------------------|------------------|-------------------|--------------|
| `baseline` (v0.1, projeção pré-implementação) | 2026-07-20 | 250 (meta) | 20 (meta) | 20.000 (meta) | ≤1% (meta) | Metas normativas do brief; ainda sem execução real — ver §6. |

> **Nota de honestidade metodológica.** Como o módulo 006-Kernel está em fase de
> especificação (`Status: Draft`), a tabela acima registra **metas**, não
> **medições**. A primeira execução real do benchmark DEVE ocorrer contra uma
> implementação de referência e popular esta tabela com dados reais antes que
> o documento avance para `Status: Review`.

## 5. Reprodutibilidade

Todo relatório de benchmark DEVE incluir:

| Item obrigatório | Descrição |
|--------------------|-----------|
| Hash do commit do Kernel | Versão exata testada (SemVer + SHA). |
| Hash/versão dos scripts de carga | Script k6/NBomber/ghz versionado em `../035-Benchmark/`. |
| Especificação do ambiente | Conforme §2.2, incluindo classe de VM/CPU/rede real usada (pode divergir do "de referência" — deve ser declarado). |
| Duração do warm-up e da coleta | Mínimo 2 min warm-up + 10 min de coleta estável. |
| Seed de geração de dados sintéticos | Para reprodutibilidade exata do mix de tenants/agentes. |
| Configuração de `kernel.*` usada | Todas as chaves da `_DESIGN_BRIEF.md` §8 relevantes (ex.: `kernel.pep.decision_cache_ttl_ms`, `kernel.acb.shard_count`). |
| Estado das dependências simuladas | Distribuição de latência configurada nos *stubs* de `009`/`022`/`008`/`010`/`011`/`012`/`015`/`017`. |

## 6. Execução (Pipeline de Benchmark)

```
Release candidato taggeado
        │
        ▼
Provisiona ambiente de referência (§2.2) — infraestrutura como código
        │
        ▼
Executa W1..W6 sequencialmente (isolamento de variável) ── warm-up 2min cada
        │
        ▼
Coleta KPIs via OTel Collector → Prometheus (retenção de longo prazo)
        │
        ▼
Compara contra baseline anterior (§4) ── regressão > 10%? ──sim──▶ bloqueia release
        │ não
        ▼
Publica relatório versionado em ../035-Benchmark/results/006-kernel/<versao>.md
        │
        ▼
Atualiza tabela de comparação histórica (§4) e dashboard de tendência
```

## 7. Riscos e Limitações Conhecidas

| Risco | Mitigação |
|-------|-----------|
| *Stubs* de dependência não refletem latência real de `009`/`022` em produção | Calibrar periodicamente a distribuição de latência dos stubs com dados reais de `Monitoring.md` desses módulos. |
| Benchmark de W6 (10⁶ ACBs) é caro e demorado — pode não rodar em toda release | Executar W6 completo trimestralmente; execuções de release usam amostra reduzida (10⁴–10⁵) com extrapolação documentada. |
| Ambiente de benchmark divergente do de produção (topologia de rede, HA) | Declarar divergências explicitamente no relatório (§5); resultados são direcionais, não uma garantia de SLA em produção. |
| Contenção de "vizinho ruidoso" na infraestrutura de nuvem compartilhada | Usar instâncias dedicadas/isoladas para o ambiente de benchmark, não compartilhadas com outras cargas. |

## 8. Referências

- Metodologia geral de benchmark do AIOS: `../035-Benchmark/README.md`
- Metas normativas de origem: `_DESIGN_BRIEF.md` §7.2 (NFR), §10 (escalabilidade)
- Catálogo de métricas usado como KPI: `./Metrics.md`
- Estratégia de testes de carga complementar (gate de CI): `./Testing.md`
- Modelo de escalabilidade e sharding: `./Scalability.md`
- Dashboards para acompanhamento contínuo: `./Monitoring.md`
