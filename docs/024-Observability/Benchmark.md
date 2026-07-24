---
Documento: Benchmark
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0242, ADR-0243, ADR-0244, ADR-0248
RFCs relacionados: RFC-0001, RFC-0240
Depende de: NonFunctionalRequirements.md, Metrics.md, Testing.md, Scalability.md
---

# 024-Observability — Benchmark

## 1. Objetivo

Medir, de forma reprodutível, se o plano de telemetria sustenta as metas de
`./NonFunctionalRequirements.md` — em especial NFR-001 (p99 ≤ 50 ms), **NFR-002
(overhead ≤ 3% CPU e ≤ 1 ms no observado)**, NFR-003 (vazão), NFR-007 (frescor),
NFR-009 (cardinalidade) e NFR-013 (consulta).

O KPI mais importante deste módulo não é a sua própria vazão: é **quanto ele custa ao
sistema que observa**. Um coletor rapidíssimo que adiciona 5 ms ao p99 de cada serviço
instrumentado é um mau coletor — a telemetria passaria a ser a maior fonte de latência
do AIOS.

## 2. Ambiente de referência

| Item | Especificação |
|------|---------------|
| Nó de aplicação | 8 vCPU, 16 GiB, SSD NVMe |
| `otel-agent` | 1 por nó, 1 vCPU / 512 MiB, buffer 2 GiB |
| `otel-gateway` medido | 1 réplica isolada (6 vCPU, 12 GiB) para NFR-003 |
| Prometheus | 8 vCPU, 32 GiB, SSD dedicado |
| Tempo / Seq | 4 vCPU, 16 GiB cada |
| PostgreSQL | 4 vCPU, 16 GiB |
| Gerador de carga | Gerador OTLP em nó separado; k6 para consulta |
| Rede | RTT intra-cluster ≤ 0,3 ms |
| Fixture de sinais | `signals-baseline` (50 descritores registrados) |

Toda medição descarta os primeiros 120 s (aquecimento de JIT, caches e *scrape*
inicial) e reporta mediana de 5 execuções.

## 3. Cargas e KPIs

| # | Carga | Perfil | KPI | Alvo |
|---|-------|--------|-----|------|
| B-01 | Ingestão de métricas | 2.000.000 dp/s, 50 sinais | p99 de aceitação | ≤ **50 ms** |
| B-02 | Ingestão de traces | 200.000 spans/s, traces de 12 spans | p99 de aceitação; spans/s | ≤ 50 ms; ≥ **200.000/s** |
| B-03 | Ingestão de logs | 100.000 logs/s | p99 de aceitação | ≤ 50 ms |
| B-04 | **Overhead no observado** | Serviço de referência com e sem instrumentação | Δ CPU; Δ p99 | ≤ **3%**; ≤ **1 ms** |
| B-05 | **Não-interferência sob saturação** | 5× a carga nominal por 2 min | Δ p99 do emissor | **0** (inalterado) |
| B-06 | Frescor fim a fim | Probe sintético sob carga nominal | p99 emissão→consulta | ≤ **10 s** |
| B-07 | Cardinalidade | 10³ → 10⁶ séries ativas | p95 de consulta; memória do Prometheus | crescimento **sub-linear** até 10⁶ |
| B-08 | *Tail sampling* | 1.000 traces/s, 5% erro, 5% lento | % retido; memória de montagem | 100% erro/cauda; memória estável |
| B-09 | Consulta de painel | 20 painéis típicos concorrentes | p95 | ≤ **2 s** |
| B-10 | *Downsampling* | 15 d de dados raw → 5 min | duração; impacto no p99 de ingestão | ≤ 30 min; impacto ≤ 5% |
| B-11 | Avaliação de SLO | 500 SLOs a cada 60 s | duração do ciclo | ≤ 20 s |
| B-12 | Latência de alerta | Degradação injetada | detecção→notificação | ≤ **2 min** (NFR-008) |

## 4. B-04 e B-05 — os dois testes que definem o módulo

### 4.1 B-04 — overhead no observado

```
   Serviço de referência (006-Kernel sintético), 3 execuções:

   (a) SEM instrumentação          → baseline de CPU e p99
   (b) COM SDK OTel, sem coletor   → custo do SDK isolado
   (c) COM SDK + otel-agent local  → custo total

   Métricas comparadas:
     Δ CPU  = CPU(c) − CPU(a)   →  alvo ≤ 3%
     Δ p99  = p99(c) − p99(a)   →  alvo ≤ 1 ms
```

Se `Δ p99` exceder 1 ms, a causa quase sempre é **export síncrono** ou lote pequeno
demais no SDK — não o coletor.

### 4.2 B-05 — não-interferência sob saturação

```
   t=0     carga nominal            → p99 do emissor registrado (baseline)
   t=60s   carga 5× no gateway      → gateway satura, fila enche
   t=120s  medir novamente o p99 do EMISSOR
   t=180s  carga volta ao nominal

   Critério: p99 do emissor em t=120s == p99 em t=0 (dentro do ruído)
             aios_obs_dropped_total > 0  (a perda é real e contabilizada)
             nenhum 429/503 devolvido ao emissor
```

Este é o teste que valida a decisão arquitetural mais importante do módulo
(ADR-0243). Se o p99 do emissor subir sob saturação do coletor, o *fail-open* não está
funcionando e a telemetria virou um acoplamento de disponibilidade.

## 5. Resultados-alvo (baseline de referência)

| KPI | Alvo | Baseline esperado | Limite de regressão |
|-----|------|-------------------|---------------------|
| B-01 p99 | ≤ 50 ms | ~18 ms | +15% |
| B-02 spans/s | ≥ 200.000 | ~245.000 | −10% |
| B-04 Δ CPU | ≤ 3% | ~1,8% | +0,5 p.p. |
| B-04 Δ p99 | ≤ 1 ms | ~0,3 ms | +0,2 ms |
| B-05 Δ p99 sob saturação | 0 | 0 | **qualquer aumento reprova** |
| B-06 frescor p99 | ≤ 10 s | ~4 s | +25% |
| B-07 p95 de consulta @ 10⁶ séries | ≤ 2 s | ~1,4 s | +20% |
| B-08 retenção de erro | 100% | 100% | **qualquer perda reprova** |
| B-09 p95 | ≤ 2 s | ~1,1 s | +20% |
| B-10 duração | ≤ 30 min | ~14 min | +30% |
| B-11 ciclo de SLO | ≤ 20 s | ~7 s | +30% |
| B-12 detecção→notificação | ≤ 2 min | ~55 s | +30% |

Regressão além do limite **bloqueia o merge** (`./Testing.md` §8). Dois KPIs têm
tolerância **zero**: B-05 (não-interferência) e B-08 (retenção de erro) — ambos
correspondem a garantias, não a metas de desempenho.

## 6. Metodologia da medição de cardinalidade (B-07)

```
   séries ativas:  10³ → 10⁴ → 10⁵ → 10⁶
        │
        ├─ medir p95 de consulta de painel típico
        ├─ medir memória residente do Prometheus
        └─ medir p99 de ingestão

   Esperado: consulta e memória crescem SUB-LINEARMENTE até 10⁶;
             ingestão praticamente constante (a cardinalidade pesa
             no ARMAZENAMENTO, não no coletor).

   Além de 10⁶: o orçamento (obs.cardinality.max_series_per_tenant)
   deve QUARENTENAR antes que a degradação apareça — e isso também
   é medido: o tempo entre ultrapassar o teto e o sinal ser descartado.
```

## 7. Comparação com alternativas

| Arranjo | p99 no observado | Volume armazenado | Detecção de erro | Escolha |
|---------|------------------|-------------------|------------------|---------|
| **Agente + gateway, tail sampling** (adotado) | +0,3 ms | ~12% dos traces | 100% dos erros | ADR-0244 |
| Export direto ao gateway (sem agente) | +2,1 ms | igual | igual | Descartado: latência de rede no caminho do emissor |
| Head sampling puro a 10% | +0,3 ms | ~10% dos traces | **~10% dos erros** | Descartado: perde justamente o que importa |
| Sem amostragem | +0,3 ms | 100% | 100% | Descartado: custo de armazenamento inviável em 10⁶ agentes |
| Backpressure ao emissor sob saturação | **+∞ (bloqueia)** | 100% | 100% | Descartado: falha de telemetria vira indisponibilidade (ADR-0243) |

A linha do *head sampling* puro é a mais instrutiva: ele custa o mesmo e armazena o
mesmo volume, mas retém apenas 10% dos traces com erro — que são exatamente os raros e
os úteis.

## 8. Reprodutibilidade

| Item | Regra |
|------|-------|
| Fixtures | Versionadas (`signals-baseline`, `traffic-nominal`, `traces-mixed`) com digest publicado |
| Sementes | Geração de carga com semente fixa |
| Configuração | Perfil de produção (`./Configuration.md` §11), exceto `head_sample_ratio` fixado em 0,1 |
| Registro | Cada execução grava: versão do serviço, descritores ativos, ambiente, resultados brutos |
| Frequência | A cada release e semanalmente em homologação |
| Publicação | Resultados anexados ao release em `../035-Benchmark/` |

## 9. Interpretação dos resultados

| Observação | Diagnóstico provável | Ação |
|------------|----------------------|------|
| B-04 Δ p99 alto, Δ CPU baixo | Export síncrono ou lote pequeno no SDK | Ajustar `BatchSpanProcessor`/`PeriodicExportingMetricReader` |
| B-05 p99 do emissor sobe | *Fail-open* não está funcionando | **Bloqueia release**: investigar backpressure no receptor |
| B-01 p99 sobe, fila estável | `PiiRedactor` com regra custosa | Revisar padrões de redação |
| B-02 spans/s cai, memória sobe | Janela de montagem de trace longa demais | Reduzir janela; verificar traces órfãos |
| B-07 consulta degrada antes de 10⁶ | Cardinalidade real acima da nominal | Verificar `labels_dropped_total` e descritores |
| B-08 erro não retido | Ordem dos processadores no gateway | *Tail sampler* **depois** da montagem completa |
| B-10 downsampling lento | I/O do objeto (MinIO) | Paralelizar por camada; verificar rede |

## 10. Referências

- Metas: `./NonFunctionalRequirements.md` · Métricas: `./Metrics.md`
- Testes: `./Testing.md` · Escala: `./Scalability.md`
- Metodologia global de benchmark: `../035-Benchmark/`

*Fim de `Benchmark.md`.*
