---
Documento: Scalability
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0002, ADR-0006, ADR-0242, ADR-0244, ADR-0248
RFCs relacionados: RFC-0001, RFC-0240
Depende de: _DESIGN_BRIEF.md §10, NonFunctionalRequirements.md, Benchmark.md, Deployment.md
---

# 024-Observability — Escalabilidade

## 1. O problema

Com 10⁶ agentes e ~500 serviços emissores, o volume bruto de telemetria é grande, mas
gerenciável: coletores escalam horizontalmente. **O que não escala é cardinalidade.**

```
   ❌ 1 agente = 1 série                 ✅ agente vive em trace/log, não em label
      10⁶ agentes ⇒ 10⁶+ séries             métricas agregam por tenant/serviço/shard
      (o observador custa mais que o         cardinalidade cresce com a TOPOLOGIA,
       observado)                            não com a POPULAÇÃO
```

Uma única linha de código que usa `agent_urn` como label de métrica cria 10⁶ séries
temporais. Nenhuma quantidade de réplicas resolve isso — o custo já foi pago no
momento em que a série nasceu.

## 2. Estratégias

| Dimensão | Estratégia |
|----------|-----------|
| Coletores | *Stateless* e horizontais; escala por CPU e por profundidade de fila. |
| Duas camadas de coleta | `otel-agent` por nó (latência ~1 ms para o emissor, buffer em disco) → `otel-gateway` central (agregação, *tail sampling*, roteamento). |
| *Tail-based sampling* | Exige que todos os spans de um trace cheguem ao mesmo gateway: **roteamento consistente por `hash(trace_id)`**. |
| Cardinalidade | O limite real não é bytes: é **séries ativas**. Orçamento por tenant e por sinal, com quarentena automática (ADR-0242). |
| Métricas de longo prazo | *Downsampling* em camadas (raw → 5 min → 1 h) com objeto em MinIO; consultas longas leem a camada agregada. |
| Consulta | Orçamento de tempo e de séries varridas; painéis pesados usam *recording rules* pré-calculadas em vez de agregação ao vivo. |
| Isolamento multi-tenant | Rótulo de tenant obrigatório e `TenantRouter`; tenant ruidoso não degrada os demais. |
| Concorrência do registro | OCC por `version`; descritor **imutável** após `Registered` elimina disputa no caminho quente. |
| Contadores de cardinalidade | Redis com janela deslizante, amostrados — contar exatamente toda série a cada datapoint custaria mais que armazená-la. |
| Alertas | Avaliação particionada por regra; deduplicação por `fingerprint` evita tempestade de notificação. |
| Rumo a milhões | 10⁶ agentes ⇒ dezenas de milhões de *datapoints*/s se cada agente virar série. Sustentado por: **agentes nunca são label**, *tail sampling*, logs amostrados por nível, e o registro de sinais como ponto único de contenção. |

## 3. Modelo de escala

```
   Camada 0 — SDK no emissor
   ┌──────────────────────────────────────────────────────────────┐
   │  head sampling (10%)  ·  agregação local de métricas          │
   │  custo no observado: ≤ 3% CPU, ≤ 1 ms p99 (NFR-002)          │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼ OTLP local (~1 ms)
   Camada 1 — otel-agent (1 por nó, DaemonSet)
   ┌──────────────────────────────────────────────────────────────┐
   │  buffer em disco 2 GiB  ·  absorve indisponibilidade curta    │
   │  escala AUTOMATICAMENTE com o cluster (1 por nó)              │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼ roteamento consistente por hash(trace_id)
   Camada 2 — otel-gateway (3–12 réplicas, HPA)
   ┌──────────────────────────────────────────────────────────────┐
   │  admissão · cardinalidade · PII · tail sampling · roteamento  │
   │  escala LINEAR: +réplica = +200.000 spans/s (NFR-003)         │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼
   Camada 3 — armazenamentos (escala por retenção e cardinalidade)
   ┌──────────────────────────────────────────────────────────────┐
   │  Prometheus (raw 15 d) → objeto/MinIO (5 min 90 d, 1 h 400 d) │
   │  Tempo/Jaeger (7 d, amostrado) · Seq (14 d)                   │
   └──────────────────────────────────────────────────────────────┘
```

O `otel-agent` é o único componente que **não** precisa de política de escala: sendo
DaemonSet, cresce com o cluster por construção.

## 4. Concorrência e locks

| Ponto | Mecanismo | Por quê |
|-------|-----------|---------|
| Ingestão | **Sem lock**; filas por pipeline, produtores independentes | O caminho quente não pode ter ponto de serialização |
| Consulta ao registro de sinais | Cache local imutável por versão | Descritor `Registered` é imutável (invariante S7) — leitura sem sincronização |
| Contagem de cardinalidade | Redis (`PFADD`/HLL amostrado), janela deslizante | Contagem aproximada é suficiente para orçamento; exata custaria mais que o dado |
| Registro de sinal | OCC por `version` | Registros concorrentes do mesmo nome: o segundo falha com `AIOS-OBS-0006` |
| Transição de alerta | OCC + índice único parcial | Impede duas instâncias abertas por `fingerprint` (invariante I1) |
| Escrita do ledger de budget | Append-only, sem update | Elimina contenção e preserva o histórico |
| *Downsampling* | Job com lock distribuído por camada | Duas execuções simultâneas produziriam agregação duplicada |

## 5. Limites teóricos e observados

| Dimensão | Limite de projeto | Fator limitante | Ação ao aproximar |
|----------|-------------------|-----------------|-------------------|
| *Datapoints*/s por réplica | 2.000.000 (NFR-003) | CPU do gateway | HPA por CPU e fila |
| Spans/s por réplica | 200.000 (NFR-003) | CPU + memória de montagem de trace | HPA; reduzir janela de montagem |
| Séries ativas por tenant | 1.000.000 (NFR-009) | Memória e índice do Prometheus | Quarentenar sinal infrator; **não** elevar o teto como primeira ação |
| Séries por sinal | 10.000 | Índice | Revisar `allowed_labels` |
| Serviços emissores | 500 (NFR-010) | Nada estrutural | — |
| Agentes | 10⁶ | Cardinalidade **se** virarem label | Regra: agente vive em trace/log |
| Retenção raw | 15 d | Disco do Prometheus | *Downsampling* + objeto |
| Consulta | 5.000.000 séries varridas | I/O do armazenamento | *Recording rules* |
| Traces retidos | ~10% + 100% dos erros | Disco do Tempo | Ajustar `tail_keep_slow_ms` |

### 5.1 Onde a escala realmente dói

O limite prático **não** é vazão de ingestão — coletores escalam. São dois outros:

1. **Cardinalidade no Prometheus.** Séries ativas determinam memória e latência de
   consulta de forma super-linear. Mitigação: orçamento + quarentena, e a regra de que
   identidade nunca é label.
2. **Montagem de trace para *tail sampling*.** O gateway precisa manter em memória
   todos os spans de traces abertos durante a janela de montagem. Mitigação: janela
   curta, roteamento consistente e limite de spans por trace.

Ambos são problemas de **memória**, não de CPU — e escalar réplicas ajuda no segundo,
mas não no primeiro.

## 6. Rumo a milhões de agentes

```
   10⁶ agentes ──▶ dezenas de milhões de datapoints/s se cada agente virar série
        │
        ├── agentes NUNCA são label  → cardinalidade cresce com topologia (nº de
        │                               serviços × shards × tenants), não com população
        ├── tail sampling            → volume de trace armazenado é fração do emitido
        ├── logs amostrados por nível→ Debug não sai de desenvolvimento
        ├── downsampling em camadas  → custo proporcional ao valor do dado no tempo
        └── registro de sinais       → ponto único onde tudo isso é imposto
```

Antipadrões que quebram a escala:

| Antipadrão | Efeito em escala | Alternativa |
|-----------|------------------|-------------|
| `agent_urn`/`task_urn` como label de métrica | 10⁶ séries de uma vez | Granularidade individual vive em trace/log |
| `trace_id` como label | Cardinalidade ilimitada e crescente | Correlação por `trace_id` no **log**, não na métrica |
| Buckets automáticos de histograma | Não caem nas fronteiras de SLO; quantil incorreto | Buckets explícitos alinhados aos limiares |
| Painel agregando ao vivo sobre 30 d | Consulta varre milhões de séries | *Recording rule* pré-calculada |
| `head_sample_ratio = 1.0` em produção | Volume de trace 10× maior sem ganho | Manter 10% + *tail sampling* de erro e cauda |
| Retenção única longa para tudo | Custo linear na janela mais longa | Camadas com *downsampling* |
| Log em `Debug` em produção | O log compete com o pipeline que o transporta | `Debug` apenas em desenvolvimento |
| Aumentar `max_series_per_tenant` ao receber alerta | Adia o problema e fixa o custo mais alto | Quarentenar o sinal infrator e corrigir o descritor |

## 7. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §10
- Medição: `./Benchmark.md` · Metas: `./NonFunctionalRequirements.md`
- Implantação e HPA: `./Deployment.md` §2.1 · Falhas: `./FailureRecovery.md`
- Plataforma: `../027-Cluster/`

*Fim de `Scalability.md`.*
