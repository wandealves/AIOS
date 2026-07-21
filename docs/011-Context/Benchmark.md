---
Documento: Benchmark
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0115
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 011 · Context — Benchmark

> Este documento especifica a **metodologia de benchmark** do
> `ContextService`: cargas, ambiente, KPIs e **resultados-alvo**. Todo
> resultado-alvo aqui é **idêntico** às metas fixadas na §7.2 do
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) e em
> [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md) —
> divergência é defeito deste documento, nunca do brief. O *gate* de
> qualidade (Compression Ratio + Δ Task Completion Rate) é executado em
> conjunto com o benchmark de tarefas do `035-Benchmark` (módulo global de
> avaliação, ainda não fan-out); aqui apenas se define a metodologia
> específica de `011-Context`. Palavras normativas conforme RFC 2119/8174.

---

## 1. Objetivo e escopo

O benchmark do `011-Context` mede, de forma **reprodutível**, quatro
dimensões que juntas expressam a missão do módulo (brief §1.1 — gerenciador
de janela de contexto análogo ao subsistema de cache/paginação de um SO):

1. **Desempenho** — latência (`CacheLookup`, `Assemble` por caminho) e
   *throughput* de `assemble` por réplica.
2. **Eficácia de compressão** — **Context Compression Ratio** média, sem
   sacrificar a Δ **Task Completion Rate**.
3. **Eficiência de cache semântico** — *hit ratio* em regime permanente e
   custo evitado (`cost_saved_usd`) sobre custo bruto estimado.
4. **Precisão** — erro de estimativa de tokens e taxa de falso-*hit* do
   cache.

Este documento **NÃO** mede corretude funcional isolada (isso é
[Testing.md](./Testing.md)) nem resiliência a falha (isso é
[FailureRecovery.md](./FailureRecovery.md) §9/[Testing.md](./Testing.md)
§9) — mede **comportamento sob carga e em regime permanente**, sempre
comparado a um alvo numérico fixo.

---

## 2. Metodologia

### 2.1 Princípios

- **Determinístico onde possível.** Testes de **latência/throughput** usam
  `ModelRouterClient`/`MemoryClient` com latência sintética controlada
  (percentis configuráveis) para isolar o custo próprio do
  `ContextAssembler` do ruído de rede externa — reprodutibilidade em
  primeiro lugar.
- **Realista onde necessário.** Testes de **qualidade** (Compression Ratio,
  Δ Task Completion Rate, falso-*hit*) usam o modelo de sumarização/
  embedding real roteado pelo `017` (via `summarization_model_hint`/
  `embedding.model_hint` de produção), pois a métrica de negócio só é
  significativa com o comportamento real do modelo.
- **A/B pareado.** A Δ Task Completion Rate é sempre medida como a
  diferença entre uma execução **com** compressão/cache (`Context` ativo)
  e uma execução **baseline** (sem compressão, contexto bruto truncado
  apenas por corte simples) sobre o **mesmo** conjunto de tarefas —
  nunca comparada entre corpora diferentes.
- **Regime permanente vs. partida a frio.** Métricas de cache (*hit ratio*,
  custo evitado) são reportadas separadamente para a janela de
  **aquecimento** (primeiros N minutos, cache vazio) e para o **regime
  permanente** (pós-aquecimento) — o SLO (NFR-006 ≥ 0,35) aplica-se **apenas**
  ao regime permanente, consistente com [NonFunctionalRequirements.md](./NonFunctionalRequirements.md)
  §4.

### 2.2 Fórmulas normativas (idênticas ao catálogo de métricas)

```
Context Compression Ratio  = 1 − tokens_in / tokens_raw          (por bundle; NFR-004: média ≥ 0.50)
Δ Task Completion Rate     = rate_com_contexto − rate_baseline    (NFR-005: ≥ −0.02 absoluto)
Cache Hit Ratio            = hits / (hits + misses)               (regime permanente; NFR-006: ≥ 0.35)
Custo evitado / bruto      = cost_saved_usd / custo_bruto_estimado (NFR-007: ≥ 0.25)
Erro de estimativa tokens  = |tokens_estimados − tokens_exatos| / tokens_exatos  (NFR-010: ≤ 0.02)
Falso-hit ratio            = hits_inadequados / hits_totais        (amostragem; NFR-011: ≤ 0.005)
```

Estas fórmulas são **as mesmas** de [Metrics.md](./Metrics.md) §3/§5 — o
benchmark não introduz uma definição alternativa.

---

## 3. Ambiente de benchmark

```
┌─────────────────────────── Ambiente de Benchmark (staging isolado) ───────────────────────────┐
│                                                                                                  │
│   Gerador de carga (k6-like)                                                                    │
│        │ perfis de workload (§4)                                                                │
│        ▼                                                                                        │
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐    │
│  │ context-service · N réplicas (mesmo perfil de recursos de Deployment.md §3: 1-2 vCPU,   │    │
│  │ 1-2 GiB por réplica) · HPA desabilitado durante a corrida (contagem de réplicas fixa)    │    │
│  └───┬───────────┬────────────────┬───────────────────┬────────────────────────────────────┘    │
│      ▼           ▼                ▼                   ▼                                        │
│  ┌────────┐ ┌───────────┐  ┌──────────────┐   ┌──────────────────┐                              │
│  │ Redis  │ │PostgreSQL │  │ MinIO         │   │ NATS JetStream    │                              │
│  │ (L1)   │ │+pgvector  │  │ (blobs)       │   │ (eventos context) │                              │
│  └────────┘ └───────────┘  └──────────────┘   └──────────────────┘                              │
│      ▲                                                                                          │
│  ModelRouterClient/MemoryClient: modo "sintético controlado" (§2.1) para bench de latência,      │
│  modo "real via 017/010" para bench de qualidade (compressão/cache/Δ Task Completion Rate)       │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

| Parâmetro | Valor |
|-----------|-------|
| Réplicas do `context-service` | 3 (fixo, sem HPA durante a corrida) — mesmo perfil de [Deployment.md](./Deployment.md) §3 |
| Isolamento | Tenant sintético dedicado (`bench-<run_id>`), nunca tráfego de produção |
| Rede | Mesma topologia de mTLS interna de produção ([Security.md](./Security.md)) |
| Ferramenta de carga | Gerador HTTP/gRPC parametrizável (perfil constante, rampa, *burst*) — versão fixada por execução (§9) |
| Duração por corrida | Aquecimento (10 min) + medição (30 min) + resfriamento (5 min) — ver §8 |

---

## 4. Cargas de teste (perfis de *workload*)

| Perfil | Descrição | Mistura *hit*/*miss* | Objetivo |
|--------|-----------|------------------------|----------|
| **W1 — Cold start** | Cache vazio, 100% *miss*, sem sumarização (contexto cabe sem compressão agressiva) | 0% hit | Medir NFR-002 em condição mais adversa de custo de `retrieve+rank+dedup`. |
| **W2 — Cold + sumarização** | Cache vazio, 100% *miss*, contexto excede orçamento e requer 1 chamada de sumarização | 0% hit | Medir NFR-003 (caminho dominado por RTT do modelo de sumarização). |
| **W3 — Warm cache** | Cache pré-aquecido com *working set* representativo | ~90% hit (sintético, para isolar NFR-001) | Medir NFR-001 em isolamento (não representa regime permanente real). |
| **W4 — Misto realista** | Distribuição de prompts recorrentes + únicos, replay de padrão de tráfego sintético multi-tenant | ~35–45% hit (alvo de regime permanente) | Medir NFR-006/NFR-007/NFR-009 em condição representativa de produção. |
| **W5 — Rajada (*burst*)** | Pico 5× a taxa sustentada por 60 s, sobre W4 em regime permanente | variável | Validar *backpressure* (`AIOS-CTX-0015`) e ausência de degradação de latência fora do *burst*. |
| **W6 — Qualidade (offline)** | Conjunto golden de tarefas do `035-Benchmark`, execução pareada com/sem compressão | N/A (não é teste de carga) | Medir Compression Ratio, Δ Task Completion Rate, falso-*hit* (§6). |

```
Taxa
 │                              ┌── W5 burst (5x) ──┐
 │                              │                    │
 │        W4 sustentado ────────┴────────────────────┴──── W4 sustentado
 │       ╱
 │  W1/W2/W3 (isolados, sequenciais, cache resetado entre perfis)
 └──────────────────────────────────────────────────────────────────▶ tempo
```

---

## 5. KPIs e metas — mapeamento direto aos NFR

| KPI | Métrica-fonte ([Metrics.md](./Metrics.md)) | Meta (idêntica ao NFR) | Perfil de carga |
|-----|-----------------------------------------------|---------------------------|--------------------|
| Latência `CacheLookup` p99 | `aios_context_cache_lookup_duration_ms` | ≤ 20 ms | W3 |
| Latência `Assemble` p99 (miss, sem sumarização) | `aios_context_assemble_duration_ms{path="miss_no_summarize"}` | ≤ 150 ms | W1 |
| Latência `Assemble` p99 (miss, com sumarização) | `aios_context_assemble_duration_ms{path="miss_summarize"}` | ≤ 1200 ms | W2 |
| Throughput `assemble` por réplica | `rate(aios_context_assemble_total)` | ≥ 500 req/s | W4 |
| Context Compression Ratio (média) | `aios_context_compression_ratio` | ≥ 0,50 | W6 |
| Δ Task Completion Rate | `aios_context_task_completion_delta_ratio` | ≥ −0,02 (absoluto) | W6 |
| Cache hit ratio (regime permanente) | `aios_context_cache_hit_total`/`cache_miss_total` | ≥ 0,35 | W4 (pós-aquecimento) |
| Custo evitado / bruto | `aios_context_cache_cost_saved_usd` | ≥ 0,25 | W4 (pós-aquecimento) |
| Erro de estimativa de tokens | `aios_context_token_estimate_error_ratio` | ≤ 2% | W6 (corpus multi-idioma) |
| Falso-*hit* de cache | `aios_context_cache_false_hit_ratio` | ≤ 0,5% | W6 (amostragem/eval offline) |
| Disponibilidade sob rajada | `aios_context:error_ratio:rate5m` | error budget não estourado durante W5 | W5 |

Todas as metas acima são **cópias literais** de
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md) §2 —
qualquer alteração de meta **DEVE** primeiro passar pelo brief §7.2.

---

## 6. Metodologia de qualidade (Compression Ratio × Task Completion Rate)

O gate de qualidade (também descrito em [Testing.md](./Testing.md) §8) segue
este protocolo determinado pelo perfil **W6**:

```
1. Selecionar N tarefas do conjunto golden (035-Benchmark), estratificadas
   por task_type (ex.: suporte, planejamento, code-review, pesquisa).
2. Para cada tarefa, executar DUAS vezes:
     (a) baseline: contexto bruto truncado por corte simples (sem
         HierarchicalCompressor/RedundancyEliminator/SemanticCacheManager)
     (b) contexto Context-011: pipeline completo (budget→retrieve→rank→
         dedup→compress), cache desabilitado na primeira passada para não
         contaminar a comparação com reuso de respostas cacheadas.
3. Registrar, por tarefa: sucesso/fracasso (Task Completion), tokens_raw,
   tokens_in, compression_ratio.
4. Agregar:
     compression_ratio_media = média(compression_ratio) sobre (b)
     task_completion_rate_baseline = sucessos(a) / N
     task_completion_rate_context  = sucessos(b) / N
     delta_task_completion = task_completion_rate_context − task_completion_rate_baseline
5. GATE: compression_ratio_media >= 0.50  E  delta_task_completion >= -0.02
```

Para **falso-*hit*** (independente do fluxo acima): amostrar M respostas
servidas por cache em produção/staging (W4) e submeter cada par
(*prompt* original, resposta cacheada servida) a um avaliador — LLM-juiz
calibrado com *rubric* fixa ou anotação humana — classificando `adequada`/
`inadequada`; `falso_hit_ratio = inadequadas / M`.

---

## 7. Resultados-alvo consolidados (linha de base para regressão)

| Dimensão | Meta | NFR | Observação |
|----------|------|-----|-------------|
| `CacheLookup` p99 | ≤ 20 ms | NFR-001 | Medido isoladamente (W3); em W4 misto, cauda pode ser um pouco maior por contenção compartilhada — reportar ambas. |
| `Assemble` p99 (miss, sem sumarização) | ≤ 150 ms | NFR-002 | W1. |
| `Assemble` p99 (miss, com sumarização) | ≤ 1200 ms | NFR-003 | W2; dominado por RTT do modelo de sumarização via `017`. |
| Throughput `assemble`/réplica | ≥ 500 req/s | NFR-009 | W4, 3 réplicas fixas — throughput total esperado ≥ 1500 req/s agregado (referência, não é o SLO por réplica). |
| Compression Ratio médio | ≥ 0,50 | NFR-004 | W6; reportado **sempre** junto de Δ Task Completion Rate. |
| Δ Task Completion Rate | ≥ −2% (abs.) | NFR-005 | W6; gate bloqueante de promoção (Testing.md §11.1). |
| Cache hit ratio (regime permanente) | ≥ 0,35 | NFR-006 | W4, pós-aquecimento (após 10 min). |
| Custo evitado/bruto | ≥ 0,25 | NFR-007 | W4, pós-aquecimento. |
| Erro de estimativa de tokens | ≤ 2% | NFR-010 | W6, corpus multi-idioma via `017`. |
| Falso-*hit* | ≤ 0,5% | NFR-011 | W4 amostrado + avaliação offline. |
| Disponibilidade sob rajada (W5) | *error budget* não consumido além do orçamento diário | NFR-008 | Verifica que `AIOS-CTX-0015` (rate-limit) absorve o excesso sem 5xx. |

---

## 8. Protocolo de execução

| Fase | Duração | Ação |
|------|---------|------|
| **Seed** | — | Popular `tenant_id` sintético, `BudgetProfile` padrão, e (para W3/W4) pré-carregar cache com *working set* representativo. |
| **Aquecimento** | 10 min | Tráfego no perfil-alvo, métricas **descartadas** da agregação de KPI (exceto para validar que o aquecimento converge). |
| **Medição** | 30 min | Tráfego sustentado no perfil-alvo; métricas agregadas por percentil (`p50/p95/p99`) e por janela de 1 min. |
| **Resfriamento** | 5 min | Verificar que a taxa de erro/latência retorna ao basal após o fim da carga (detecta vazamento de recursos/*backlog*). |
| **Repetição** | ≥ 3 corridas por perfil | Reduz variância; resultado reportado é a **mediana das medianas de p99** entre corridas, com desvio explicitado. |

```
seed ──▶ aquecimento(10min, descartado) ──▶ medição(30min, agregada) ──▶ resfriamento(5min) ──▶ relatório
                                                     │
                                          repetir ≥3x por perfil (W1..W5)
```

---

## 9. Reprodutibilidade

- **Versionamento do gerador de carga e dos perfis W1–W6**: cada corrida
  referencia a *tag* de versão do gerador e o *hash* do arquivo de definição
  de perfil (workload-as-code), publicados junto ao relatório (§10).
- **Sementes determinísticas** para geração de tráfego sintético (W1–W5) —
  mesma semente produz a mesma sequência de `tenant`/`agent`/tamanho de
  contexto candidato, isolando variação real de infraestrutura.
- **Corpus golden versionado** (W6, `035-Benchmark`) — mudança de corpus
  gera uma nova linha de base explícita, nunca comparada silenciosamente
  contra uma linha de base de corpus anterior.
- **`model_id`/`summarization_model_hint`/`embedding.model_hint` fixados**
  por corrida — mudança de modelo roteado pelo `017` invalida comparação
  direta com corridas anteriores (deve ser tratada como nova linha de base).
- **Ambiente isolado**: nenhuma corrida de benchmark compartilha réplicas ou
  backends com tráfego de produção ou com testes de caos ([Testing.md](./Testing.md)
  §9) simultâneos.
- Resultados brutos (séries de latência, *logs* de execução) são retidos
  por, no mínimo, 2 ciclos de release, para permitir comparação de
  regressão entre versões consecutivas.

---

## 10. Relatório de benchmark (formato canônico)

```
=== AIOS · 011-Context · Benchmark Report ===
run_id:            bench-2026-07-21-001
generator_version: loadgen@1.4.2
workload_hash:     sha256:9f2c...ab31
model_id:          gpt-x-mini (fixo)
replicas:          3

--- W1 (cold, sem sumarização) ---
assemble p50/p95/p99 (ms): 61 / 118 / 143      [meta p99 <= 150]   PASS
error_rate:                0.02%

--- W2 (cold, com sumarização) ---
assemble p50/p95/p99 (ms): 410 / 980 / 1150    [meta p99 <= 1200]  PASS

--- W3 (warm, hit sintético) ---
cache_lookup p50/p95/p99 (ms): 2 / 9 / 18      [meta p99 <= 20]    PASS

--- W4 (misto realista, regime permanente) ---
throughput/replica (req/s):     512            [meta >= 500]       PASS
cache_hit_ratio:                0.38            [meta >= 0.35]      PASS
cost_saved_ratio:               0.29            [meta >= 0.25]      PASS

--- W5 (burst 5x) ---
rate_limited_rejections:        842 (esperado; AIOS-CTX-0015)
5xx_during_burst:                0              [meta: 0]           PASS

--- W6 (qualidade, corpus golden N=480) ---
compression_ratio_media:        0.54            [meta >= 0.50]      PASS
delta_task_completion:          -0.011          [meta >= -0.02]     PASS
token_estimate_error:           0.014           [meta <= 0.02]      PASS
false_hit_ratio (amostra M=200):0.004           [meta <= 0.005]     PASS

=== Resultado consolidado: GO (todas as metas atingidas) ===
```

Toda corrida gera este relatório, versionado junto ao artefato de release e
referenciado no *changelog* — divergência de **qualquer** linha em relação à
meta correspondente bloqueia a promoção a produção
([Testing.md](./Testing.md) §11.1).

---

## 11. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-------------------|--------------------------|
| Latência sintética de `ModelRouterClient`/`MemoryClient` (W1–W5) não reflete variação real de rede em produção. | Perfis de latência sintética calibrados periodicamente contra amostras reais de produção (p50/p95/p99 observados via [Monitoring.md](./Monitoring.md)); revisão trimestral. |
| Corpus golden (W6) pequeno demais para detectar regressões sutis de qualidade. | N mínimo por *task_type* definido em conjunto com `035-Benchmark`; ampliação incremental do corpus a cada ciclo. |
| Variância entre corridas mascarando regressão real. | ≥ 3 repetições por perfil, mediana reportada, desvio explicitado (§8); alerta se desvio > limiar configurado. |
| Custo de rodar W6 (avaliação por LLM-juiz) a cada release. | Amostragem estatística (M por perfil) em vez de corpus completo em toda corrida; corpus completo reservado a releases *major*. |
| Benchmark "verde" mas produção diverge (SLO real abaixo do medido). | [Monitoring.md](./Monitoring.md) monitora os mesmos SLI em produção continuamente; divergência sustentada entre benchmark e produção é tratada como defeito de calibração do benchmark (§11, ação de revisão). |
| **Alternativa descartada:** medir Δ Task Completion Rate apenas com avaliação humana. | Rejeitada isoladamente por custo/latência de ciclo — adotado LLM-juiz calibrado como avaliador primário, com auditoria humana periódica por amostragem (mesma abordagem de NFR-011). |
| **Alternativa descartada:** um único perfil de carga "genérico" cobrindo todos os NFR de latência. | Rejeitada — os três caminhos (hit/miss sem sumarização/miss com sumarização) têm perfis de custo estruturalmente diferentes ([NonFunctionalRequirements.md](./NonFunctionalRequirements.md) §4); perfis separados (W1/W2/W3) isolam cada meta. |

---

## 12. Ver também

- [`_DESIGN_BRIEF.md` §7.2](./_DESIGN_BRIEF.md) — metas canônicas de NFR.
- [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md) — detalhamento
  e método de verificação por NFR.
- [`Metrics.md`](./Metrics.md) — métricas-fonte e *recording rules* usadas
  nos KPIs.
- [`Monitoring.md`](./Monitoring.md) — SLO/SLA/*error budget* em produção
  contínua (complementar ao benchmark periódico).
- [`Testing.md`](./Testing.md) — *gate* de CI/CD que consome este benchmark
  (§11.1) e testes funcionais/qualidade relacionados.
- [`Scalability.md`](./Scalability.md) — limites teóricos e gargalos
  previstos referenciados pelos perfis de carga.
- Glossário: [Context Compression Ratio, Task Completion Rate, Semantic Cache](../040-Glossary/Glossary.md).
