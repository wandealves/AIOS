---
Documento: Benchmark
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (admission control), ADR-0091 (score multiobjetivo), ADR-0092 (sharding), ADR-0093 (filas ZSET/aging), ADR-0094 (preempção compensável), ADR-0095 (backpressure)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 035-Benchmark, 024-Observability, 022-Policy, 026-Cost-Optimizer, 027-Cluster, 007-Agent-Runtime
---

# 009-Scheduler — Benchmark

> Este documento define a **metodologia de benchmark** do Scheduler cognitivo
> — como medir, reproduzir e comparar seu desempenho contra as metas
> normativas de `NonFunctionalRequirements.md` (NFR-001..NFR-017) e do
> `_DESIGN_BRIEF.md` §10 (escalabilidade e concorrência). Segue o arcabouço
> metodológico geral de `../035-Benchmark/README.md`, especializado para a
> superfície de decisão de escalonamento.

## 1. Objetivo

Prover uma metodologia **reprodutível** para responder, a cada release:

1. O Scheduler atinge p99 de decisão ≤ 20 ms e p50 ≤ 5 ms sob carga realista
   (NFR-001)?
2. O Scheduler sustenta ≥ 50.000 decisões/s por réplica, com escala linear ao
   adicionar réplicas (NFR-002)?
3. O sistema de cotas mantém 0 overcommit sob concorrência adversarial
   (NFR-005)?
4. A fairness garante `min_share` a todo tenant ativo mesmo sob dominância de
   um tenant (NFR-006)?
5. O overhead de preempção permanece ≤ 50 ms sob contenção (NFR-009)?
6. O Scheduler sustenta o caminho para 10⁶⁺ agentes gerenciados com
   coordenação sub-linear via sharding (NFR-004, `_DESIGN_BRIEF.md` §10)?
7. Uma mudança no número de shards (`N`) ocorre sem downtime de admissão
   (NFR-014)?

## 2. Metodologia

### 2.1 Princípios

- **Reprodutibilidade em primeiro lugar**: toda carga DEVE ser expressa como
  script versionado (k6/ghz/NBomber) com seed determinístico para geração de
  dados sintéticos (tenants, agentes, classes de serviço); resultados DEVEM
  incluir hash do commit do Scheduler testado e hash do script de carga.
- **Isolamento de variável**: cada benchmark varia **uma** dimensão por vez
  (ex.: apenas RPS, ou apenas número de tenants concorrentes, ou apenas taxa
  de preempção induzida) para permitir atribuição causal de regressões.
- **Ambiente representativo, não de produção**: ambiente `staging` dedicado a
  benchmark (ver `../035-Benchmark/README.md`), dimensionado de forma
  documentada e estável entre execuções (mesma classe de VM/CPU/rede).
- **Warm-up obrigatório**: 2 minutos de carga descartada antes da coleta,
  para estabilizar caches de decisão de política/orçamento, pools de conexão
  Redis/PostgreSQL/NATS e JIT/AOT do .NET 10.
- **Caminho quente isolado**: dependências externas (022/026/027/007) DEVEM
  ser simuladas via *stubs* com latência configurável e realista, para que o
  benchmark meça o desempenho do **próprio** Scheduler, não o das dependências.

### 2.2 Ambiente de Referência

| Componente | Especificação de referência |
|------------|-------------------------------|
| Scheduler Service | N réplicas (variável por cenário), 4 vCPU / 8 GiB RAM cada, .NET 10 AOT, sem *co-location* com dependências |
| Redis | 4 vCPU / 8 GiB RAM, modo standalone, persistência AOF desabilitada durante medição de latência pura |
| PostgreSQL 16 | 4 vCPU / 16 GiB RAM, SSD NVMe, streaming replication desabilitada (medição de nó único; HA medida separadamente em `Scalability.md`) |
| NATS/JetStream | 3 nós (cluster mínimo), armazenamento em disco |
| Dependências simuladas (`022`,`026`,`027`,`007`) | *Stub services* com latência configurável (distribuição realista, não latência zero) para isolar o desempenho do Scheduler do desempenho de terceiros |
| Rede | Mesma zona de disponibilidade (latência de rede interna < 1 ms p50) |
| Gerador de carga | k6 (REST) / ghz ou NBomber (gRPC), executando fora das réplicas do Scheduler |

### 2.3 Cargas de Trabalho (Workload Mix)

| Carga | Descrição | Mix/parâmetros | Uso |
|-------|-----------|-----------------|-----|
| **W1 — Admissão pura** | Alta taxa de `Submit` sem preempção, sem deadline, capacidade abundante. | 100% `Submit`, `service_class` uniforme `INTERACTIVE` | Mede NFR-001 (piso de latência), NFR-002 (piso de throughput). |
| **W2 — Mix realista de classes** | Distribuição observada/estimada de classes de serviço. | 40% `INTERACTIVE`, 30% `BATCH`, 20% `BACKGROUND`, 10% `SYSTEM` | Mede NFR-001/NFR-002 sob mix heterogêneo, incluindo EDF. |
| **W3 — Cota adversarial** | Múltiplos agentes do mesmo tenant competindo pelo mesmo `(tenant, service_class)` perto do limite de `max_concurrency`. | 100% `Submit` no limite de cota, alta concorrência | Mede NFR-005 (0 overcommit). |
| **W4 — Fairness multi-tenant** | Um tenant dominante (80% do tráfego) e múltiplos tenants minoritários. | N tenants, 1 dominante, resto uniforme | Mede NFR-006 (`min_share` respeitado). |
| **W5 — Preempção sob contenção** | Capacidade escassa forçando preempções frequentes entre classes de prioridade distintas. | Mix com `INTERACTIVE` de alta prioridade sistematicamente colidindo com `BATCH` em execução | Mede NFR-009 (overhead de preempção) e ausência de *thrashing* (ADR-0094). |
| **W6 — Backpressure sustentado** | Carga acima da capacidade de dispatch, forçando níveis `defer`/`reject`. | Rajada sustentada de `Submit` além da capacidade de shards disponível | Mede FR-005/comportamento de `BackpressureController` e `Retry-After`. |
| **W7 — Escala de agentes/shards** | População crescente de tarefas/agentes geridos (10³ → 10⁶), medindo custo por decisão sob essa população. | `Submit` esporádico + leitura de `StreamQueueState` de amostra aleatória, `N` (shards) escalado proporcionalmente | Mede NFR-004 e `_DESIGN_BRIEF.md` §10 (sub-linearidade). |
| **W8 — Rebalanceamento de shards** | Mudança de `scheduler.shards.count` sob carga sustentada. | Alteração de `N` de 64 para 128 durante W1 em execução | Mede NFR-014 (0 downtime, fração remapeada ≈ `1/N`). |

### 2.4 KPIs Coletados

| KPI | Fonte (métrica) | Unidade |
|-----|-------------------|---------|
| Latência p50/p95/p99 de decisão | `aios_scheduler_decision_latency_ms` | ms |
| Throughput sustentado (decisões/s) | `rate(aios_scheduler_requests_total[1m])` | decisões/s |
| Taxa de rejeição por código | `aios_scheduler_admission_rejected_total` / total | % |
| Overcommit de slot | `aios_scheduler_quota_overcommit_total` | contagem (deve ser 0) |
| Fairness share por tenant | `aios_scheduler_fairness_share_ratio` | ratio |
| Overhead de preempção | `aios_scheduler_preemption_ms` | ms |
| Preempções em thrashing | `aios_scheduler_preemption_cooldown_blocked_total` | contagem |
| Nível/pressão de backpressure | `aios_scheduler_backpressure_level`, `aios_scheduler_backpressure_pressure_ratio` | enum / ratio |
| Lag do outbox sob carga | `aios_scheduler_outbox_lag_ms` | ms |
| Fração de chaves remapeadas no rebalanceamento | `aios_scheduler_shard_rebalance_keys_moved_ratio` | ratio |
| Utilização de CPU/RAM da réplica | métricas de infraestrutura (cAdvisor/node-exporter) | % / MiB |
| Custo por 1k decisões (infra) | derivado (custo da réplica ÷ throughput) | USD |

## 3. Resultados-Alvo (Metas de Aceitação)

> Uma execução de benchmark **PASSA** se, para a carga correspondente,
> **todos** os KPIs abaixo forem atendidos com a réplica de referência (§2.2).

| Carga | KPI | Meta | Rastreia |
|-------|-----|------|----------|
| W1 | p99 latência de decisão | ≤ 20 ms | NFR-001 |
| W1 | p50 latência de decisão | ≤ 5 ms | NFR-001 |
| W1 | Throughput sustentado por réplica | ≥ 50.000 decisões/s | NFR-002 |
| W2 | Throughput sustentado (mix realista) | ≥ 50.000 decisões/s por réplica | NFR-002 |
| W2 | Taxa de erro (não-negócio, ex.: `AIOS-SCHED-0012`) | ≤ 0,1% | qualidade geral |
| W3 | Overcommit de slot sob concorrência | 0 (zero) | NFR-005 |
| W4 | Fairness share do menor tenant | ≥ `scheduler.fairness.min_share` (default 1%) | NFR-006 |
| W5 | p99 overhead de preempção | ≤ 50 ms | NFR-009 |
| W5 | Preempções bloqueadas por cooldown vs. total | sem crescimento sustentado indicando *thrashing* não contido | ADR-0094 |
| W6 | Backpressure sinaliza `reject` com `Retry-After` coerente antes de degradação de latência | pressão ≥ 0.95 → 100% das rejeições com `retryAfterMs` presente | FR-005 |
| W7 | Latência de decisão com 10⁶ tarefas/agentes geridos | degradação ≤ 10% vs. baseline com 10³ | NFR-004, sub-linearidade (`_DESIGN_BRIEF.md` §10) |
| W8 | Downtime de admissão durante rebalanceamento (`N`: 64→128) | 0 (zero) | NFR-014 |
| W8 | Fração de chaves remapeadas | ≈ `1/N` (não 100%) | ADR-0092 |
| Todas | Lag do outbox sob carga sustentada | ≤ `scheduler.decision.timeout_ms` × 5 | NFR-007 (proxy de saúde, não de perda) |

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

| Release | Data | p99 decisão (ms) | RPS sustentado | Overcommit | Fairness (min tenant) | Preempção p99 (ms) | Observações |
|---------|------|--------------------|-------------------|--------------|--------------------------|-----------------------|--------------|
| `baseline` (v0.1, projeção pré-implementação) | 2026-07-20 | 20 (meta) | 50.000 (meta) | 0 (meta) | ≥1% (meta) | 50 (meta) | Metas normativas do brief/NFRs; ainda sem execução real — ver §6. |

> **Nota de honestidade metodológica.** Como o módulo 009-Scheduler está em
> fase de especificação (`Status: Draft`), a tabela acima registra **metas**,
> não **medições**. A primeira execução real do benchmark DEVE ocorrer contra
> uma implementação de referência e popular esta tabela com dados reais antes
> que o documento avance para `Status: Review`.

## 5. Reprodutibilidade

Todo relatório de benchmark DEVE incluir:

| Item obrigatório | Descrição |
|--------------------|-----------|
| Hash do commit do Scheduler | Versão exata testada (SemVer + SHA). |
| Hash/versão dos scripts de carga | Script k6/ghz/NBomber versionado em `../035-Benchmark/`. |
| Especificação do ambiente | Conforme §2.2, incluindo classe de VM/CPU/rede real usada (pode divergir do "de referência" — deve ser declarado). |
| Duração do warm-up e da coleta | Mínimo 2 min warm-up + 10 min de coleta estável. |
| Seed de geração de dados sintéticos | Para reprodutibilidade exata do mix de tenants/agentes/classes. |
| Configuração de `scheduler.*` usada | Todas as chaves da `_DESIGN_BRIEF.md` §8 relevantes (ex.: `scheduler.shards.count`, `scheduler.decision.timeout_ms`, `scheduler.backpressure.*`, `scheduler.weights.*`). |
| Estado das dependências simuladas | Distribuição de latência configurada nos *stubs* de `022`/`026`/`027`/`007`. |
| Motor de política ativo | `scheduler.policy.engine` (`heuristic` ou `rl`) — resultados DEVEM ser reportados separadamente por motor quando ambos forem testados. |

## 6. Execução (Pipeline de Benchmark)

```
Release candidato taggeado
        │
        ▼
Provisiona ambiente de referência (§2.2) — infraestrutura como código
        │
        ▼
Executa W1..W8 sequencialmente (isolamento de variável) ── warm-up 2min cada
        │
        ▼
Coleta KPIs via OTel Collector → Prometheus (retenção de longo prazo)
        │
        ▼
Compara contra baseline anterior (§4) ── regressão > 10%? ──sim──▶ bloqueia release
        │ não
        ▼
Publica relatório versionado em ../035-Benchmark/results/009-scheduler/<versao>.md
        │
        ▼
Atualiza tabela de comparação histórica (§4) e dashboard de tendência
```

## 7. Riscos e Limitações Conhecidas

| Risco | Mitigação |
|-------|-----------|
| *Stubs* de dependência não refletem latência real de `022`/`026`/`027`/`007` em produção | Calibrar periodicamente a distribuição de latência dos stubs com dados reais de `Monitoring.md` desses módulos. |
| Benchmark de W7 (10⁶ agentes/tarefas) é caro e demorado — pode não rodar em toda release | Executar W7 completo trimestralmente; execuções de release usam amostra reduzida (10⁴–10⁵) com extrapolação documentada. |
| Ambiente de benchmark divergente do de produção (topologia de rede, HA) | Declarar divergências explicitamente no relatório (§5); resultados são direcionais, não uma garantia de SLA em produção. |
| Contenção de "vizinho ruidoso" na infraestrutura de nuvem compartilhada | Usar instâncias dedicadas/isoladas para o ambiente de benchmark, não compartilhadas com outras cargas. |
| Resultados de `LearnedPolicy` (v2, RL) dependem de modelo treinado específico | Reportar sempre a versão do modelo/`policy_version` junto ao resultado; benchmark de `rl` é complementar ao de `heuristic`, nunca substitui. |
| W8 (rebalanceamento) tem efeito colateral de redistribuir carga real entre shards | Executar em janela isolada, sem sobreposição com outras cargas do mesmo lote de benchmark. |

## 8. Referências

- Metodologia geral de benchmark do AIOS: `../035-Benchmark/README.md`
- Metas normativas de origem: `./NonFunctionalRequirements.md`, `_DESIGN_BRIEF.md` §7.2, §10
- Catálogo de métricas usado como KPI: `./Metrics.md`
- Estratégia de testes de carga complementar (gate de CI): `./Testing.md`
- Modelo de escalabilidade e sharding: `_DESIGN_BRIEF.md` §10 (`Scalability.md`, quando produzido)
- Dashboards para acompanhamento contínuo: `./Monitoring.md`
