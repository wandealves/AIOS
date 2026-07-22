---
Documento: Benchmark
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0071, ADR-0073, ADR-0077
RFCs relacionados: RFC-0001
Depende de: ./NonFunctionalRequirements.md, ./Testing.md, ./Scalability.md
---

# 007-Agent-Runtime — Benchmark

> Metodologia, cargas e resultados-alvo dos benchmarks formais do Agent
> Runtime. Todo alvo numérico é **idêntico** ao das metas de
> `./NonFunctionalRequirements.md` — este documento define **como** medir,
> não redefine **quanto**.

## Índice

1. Metodologia
2. Ambiente de benchmark (`perf-lab`)
3. Benchmark — Cold Start
4. Benchmark — Overhead de Sandbox
5. Benchmark — Latência de Passo do Loop
6. Benchmark — Throughput de Eventos
7. Benchmark — Densidade de Sessões
8. Benchmark — Retomada (Resume)
9. Comparação com baseline
10. Reprodutibilidade
11. Referências

---

## 1. Metodologia

- Todo benchmark **DEVE** rodar em ambiente isolado (`perf-lab`, sem
  tráfego concorrente de outros times), com no mínimo 3 execuções
  independentes; o resultado reportado é a **mediana das medianas** (para
  reduzir ruído de execução única) e o **p99 da pior execução** (para
  garantir que o pior caso observado também é reportado).
- Toda carga usa `AgentSpec` sintético e ferramentas/modelos determinísticos
  (`golden responses`) para isolar a latência do **runtime** da latência de
  dependências externas variáveis — a menos que o benchmark seja
  explicitamente "fim-a-fim com dependência real" (indicado por benchmark).
- Resultados são publicados como série temporal (regressão de desempenho
  entre versões é tão importante quanto o valor absoluto).

---

## 2. Ambiente de Benchmark (`perf-lab`)

| Recurso | Especificação |
|---------|-----------------|
| Nó de execução | 32 vCPU / 64 GiB RAM, kernel Linux com suporte completo a `seccomp-bpf`/`cgroups v2`. |
| Rede | Mesma AZ que NATS/Redis/MinIO de teste (latência de rede desprezável no isolamento da métrica). |
| `runtime.pool.warm_size` | Variável por benchmark (0 para "cold", ≥ 8 para "warm"). |
| Dependências | NATS/JetStream, Redis, MinIO dedicados ao `perf-lab` (sem contenção com staging/produção). |

---

## 3. Benchmark — Cold Start

**Meta:** p99 boot→ready ≤ 250 ms (warm) / ≤ 900 ms (cold) — NFR-001.

| Carga | Descrição |
|-------|-------------|
| C1 — Warm burst | 200 chamadas `Boot` em rajada com `warm_size ≥ 8`, medindo `aios_runtime_cold_start_ms{warm="true"}`. |
| C2 — Cold burst | 50 chamadas `Boot` com `warm_size=0`, forçando boot completo (namespaces+seccomp+cgroups+AgentSpec) a cada chamada. |
| C3 — Warm sob pressão | Rajada C1 repetida com o pool quente já parcialmente consumido (simula pico real de demanda). |

**KPI:** p50/p95/p99 de `aios_runtime_cold_start_ms`, taxa de
`AIOS-RUNTIME-0001` (pool sem slot) sob C3.

---

## 4. Benchmark — Overhead de Sandbox

**Meta:** overhead de CPU ≤ 5%; overhead de latência por passo ≤ 8 ms — NFR-002.

| Carga | Descrição |
|-------|-------------|
| D1 — Baseline sem sandbox | Loop ReAct sintético executado **sem** `seccomp`/`cgroups`/namespaces (ambiente de comparação apenas, nunca produção). |
| D2 — Com sandbox (perfil `net-restricted`) | Mesmo loop sintético, com todos os controles de `./Security.md` §6 ativos. |

**KPI:** `(CPU_D2 − CPU_D1) / CPU_D1` (`aios_runtime_sandbox_overhead_ratio`);
`(latência_passo_D2 − latência_passo_D1)` em ms.

---

## 5. Benchmark — Latência de Passo do Loop

**Meta:** p95 de latência de passo (excluindo LLM/tool) ≤ 15 ms — NFR-003.

| Carga | Descrição |
|-------|-------------|
| E1 — Loop com tool/model mockados de latência zero | Isola a latência intrínseca de `PerceptionModule→ReasoningModule→ActionExecutor→ObservationCollector`. |
| E2 — Loop com tool/model de latência realista (via *golden responses* com atraso simulado) | Mede a composição total (para comparação com SLO fim-a-fim, não para o SLO de NFR-003 em si, que exclui LLM/tool). |

**KPI:** `aios_runtime_step_latency_ms` (p50/p95/p99) de E1.

---

## 6. Benchmark — Throughput de Eventos

**Meta:** ≥ 5.000 msg/s por nó sem *backpressure* — NFR-007.

| Carga | Descrição |
|-------|-------------|
| F1 — Rajada sustentada | Múltiplas sessões simultâneas gerando `task.execution.step` continuamente por ≥ 15 min, medindo taxa de publicação e `aios_runtime_events_outbox_pending`. |

**KPI:** `rate(aios_runtime_events_published_total[1m])`; backlog máximo
observado em `aios_runtime_events_outbox_pending`.

---

## 7. Benchmark — Densidade de Sessões

**Meta:** ≥ 500 sessões concorrentes por nó — NFR-004.

| Carga | Descrição |
|-------|-------------|
| G1 — Rampa de sessões `busy` | Materialização progressiva de sessões até `runtime.pool.max_sessions_per_node`, medindo degradação de `aios_runtime_step_latency_ms` e `aios_runtime_session_rss_mib` em função da densidade. |

**KPI:** densidade máxima sustentável antes de violar NFR-002/NFR-003;
RAM total do nó em função do nº de sessões `busy`.

---

## 8. Benchmark — Retomada (`Resume`)

**Meta:** RTO ≤ 30 s — NFR-006.

| Carga | Descrição |
|-------|-------------|
| H1 — Resume em massa após falha simulada de nó | N sessões `suspended` retomadas simultaneamente em novas instâncias após "morte" simulada do nó original. |

**KPI:** `aios_runtime_resume_latency_ms` (p50/p95/p99);
`aios_runtime_lease_conflicts_total` (deve ser 0 em condições normais).

---

## 9. Comparação com Baseline

| Dimensão | Baseline de referência | Fonte |
|-----------|----------------------------|-------|
| Cold start | Processo Python "nu" (sem sandbox, sem pool) iniciando o mesmo `CognitiveLoopEngine`. | Benchmark D1/C2. |
| Overhead de sandbox | Mesmo *workload* sem `seccomp`/`cgroups`/namespaces. | Benchmark D1. |
| Densidade de sessões | Densidade calculada apenas por limite de CPU/RAM bruto do nó (sem considerar overhead de sandbox). | Cálculo teórico documentado em `./Scalability.md`. |

Toda regressão ≥ 10% em qualquer KPI acima **DEVE** ser investigada antes de
promover uma versão a produção (gate de `./Testing.md` §9).

---

## 10. Reprodutibilidade

- Todo benchmark é definido como um script versionado (`perf/benchmarks/*.py`,
  fora do escopo deste repositório de documentação, mas referenciado aqui
  por nome de arquivo canônico) com seed determinística para geração de
  carga sintética.
- Resultados são publicados com: versão do runtime testada, commit hash,
  especificação exata do ambiente (`perf-lab`), e os três valores de cada
  execução (não apenas a média/mediana final).
- Qualquer alteração de metodologia (ex.: nova ferramenta de carga) **DEVE**
  ser registrada com data e motivo, para que comparações históricas
  permaneçam válidas ou sejam explicitamente marcadas como "quebra de
  série".

---

## 11. Referências

- Metas numéricas (fonte): `./NonFunctionalRequirements.md`.
- Estratégia geral de testes: `./Testing.md`.
- Modelo de escala e cálculo de densidade: `./Scalability.md`.
- Catálogo de métricas usadas como KPI: `./Metrics.md`.
