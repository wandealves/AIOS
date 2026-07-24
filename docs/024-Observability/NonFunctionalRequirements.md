---
Documento: NonFunctionalRequirements
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0242, ADR-0243, ADR-0245, ADR-0248
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §7.2, 005-Database, 025-Audit, 029-Operations
---

# 024-Observability — Requisitos Não-Funcionais

> Metas idênticas às de `./_DESIGN_BRIEF.md` §7.2. Os mesmos números aparecem em
> `./Monitoring.md`, `./Benchmark.md`, `./FailureRecovery.md`, `./Configuration.md` e
> `./Scalability.md` — divergência numérica entre documentos é defeito.

## 1. Tabela de requisitos não-funcionais

| ID | Atributo | Meta (SLO) | SLI / método de verificação | Teste |
|----|----------|-----------|-----------------------------|-------|
| NFR-001 | Desempenho (ingestão) | p99 de aceitação de lote OTLP ≤ **50 ms**. | `aios_obs_ingest_latency_ms`. | T-OBS-101 |
| NFR-002 | Overhead no observado | ≤ **3%** de CPU e ≤ **1 ms** adicionados ao p99 do serviço instrumentado. | Benchmark comparativo com e sem instrumentação. | T-OBS-102 |
| NFR-003 | Throughput | ≥ **2.000.000** *datapoints*/s e ≥ **200.000** spans/s por réplica de coletor. | Benchmark `./Benchmark.md` §3. | T-OBS-103 |
| NFR-004 | Disponibilidade (ingestão) | ≥ **99,9%**. | Uptime probe; *error budget* mensal. | T-OBS-104 |
| NFR-005 | Disponibilidade (alerta) | ≥ **99,95%** — o caminho de alerta é mais crítico que o de consulta. | Probe sintético de alerta ponta a ponta. | T-OBS-105 |
| NFR-006 | Perda de telemetria | ≤ **0,1%** em operação normal; **100%** do descarte contabilizado sob saturação. | `aios_obs_dropped_total` / `aios_obs_received_total`. | T-OBS-106 |
| NFR-007 | Frescor | Sinal visível na consulta em ≤ **10 s** (p99) após a emissão. | Probe sintético fim a fim. | T-OBS-107 |
| NFR-008 | Tempo de detecção | Alerta de queima rápida dispara em ≤ **2 min** do início da degradação. | Injeção controlada de falha. | T-OBS-108 |
| NFR-009 | Cardinalidade | ≤ **10⁶** séries ativas por tenant; **0** labels de identidade em produção. | `aios_obs_active_series`; varredura do registro. | T-OBS-109 |
| NFR-010 | Escalabilidade | ≥ **10⁶** agentes e ≥ **500** serviços emissores sem crescimento super-linear de custo. | Teste de escala `./Scalability.md` §5. | T-OBS-110 |
| NFR-011 | Recuperação | **RTO ≤ 15 min** (plano de controle), **RPO ≤ 5 min** para SLOs/regras/alertas; telemetria bruta com perda máxima de **1 min** (buffer em disco). | DR drill trimestral. | T-OBS-111 |
| NFR-012 | Retenção | Métricas: raw **15 d**, 5 min/**90 d**, 1 h/**400 d**. Traces: **7 d**. Logs: **14 d**. Aderência de **100%**. | Auditoria de expurgo. | T-OBS-112 |
| NFR-013 | Latência de consulta | p95 de consulta de painel ≤ **2 s**; consulta acima do orçamento é abortada. | `aios_obs_query_latency_ms`. | T-OBS-113 |
| NFR-014 | Privacidade | **0** ocorrências de PII em sinal classificado `operational`. | Varredura automatizada contínua (FR-007). | T-OBS-114 |
| NFR-015 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key` (RFC-0001 §5.5). | T-OBS-115 |
| NFR-016 | Cobertura de runbook | **100%** das regras ativas com runbook válido e revisado nos últimos **180 dias**. | Consulta ao registro; alerta de runbook envelhecido. | T-OBS-116 |
| NFR-017 | Isolamento de tenant | **0** consultas que atravessem a fronteira de tenant. | RLS + validação estrutural; `AIOS-OBS-0003` monitorado. | T-OBS-117 |

## 2. Atributos de qualidade por categoria

| Categoria | NFRs | Comentário |
|-----------|------|-----------|
| Desempenho | NFR-001, NFR-003, NFR-013 | O caminho de ingestão é otimizado para vazão; o de consulta, para previsibilidade. |
| Impacto no sistema observado | NFR-002, NFR-006 | O requisito mais importante do módulo: **não atrapalhar**. |
| Disponibilidade | NFR-004, NFR-005 | Assimetria deliberada: alerta (99,95%) é mais crítico que ingestão (99,9%). |
| Escalabilidade | NFR-009, NFR-010 | O limite é **séries**, não bytes. |
| Recuperação | NFR-011 | RPO diferenciado entre plano de controle e telemetria bruta. |
| Custo e ciclo de vida do dado | NFR-012 | Retenção em camadas é decisão de custo tanto quanto de utilidade. |
| Privacidade | NFR-014 | Redação na borda; PII não chega ao disco. |
| Operabilidade | NFR-008, NFR-016 | Detectar rápido **e** saber o que fazer. |
| Consistência | NFR-015, NFR-017 | Idempotência e isolamento como propriedades verificáveis. |

## 3. Orçamento de latência da ingestão (p99)

```
   aceitação do lote OTLP ≤ 50 ms
   ├── TelemetryIngestGateway (TLS, parse, validação) ....... 8 ms
   ├── SignalAdmission (lookup em cache do registro) ........ 3 ms
   ├── CardinalityGuard (contador em Redis, amostrado) ...... 4 ms
   ├── PiiRedactor (varredura de atributos) ................. 6 ms
   ├── ResourceEnricher ..................................... 2 ms
   ├── enfileiramento para o pipeline (assíncrono) .......... 1 ms
   └── margem para GC e picos ............................... 26 ms

   O ACK ao emissor ocorre APÓS o enfileiramento — nunca após a escrita
   no armazenamento. Esperar a persistência transformaria a latência do
   Prometheus na latência do serviço observado (ADR-0243).
```

## 4. Assimetria deliberada de disponibilidade

| Caminho | SLO | Justificativa |
|---------|-----|---------------|
| Ingestão | 99,9% | Perder alguns minutos de telemetria degrada a visibilidade, não o produto. O buffer local do `otel-agent` cobre a janela curta. |
| **Alerta** | **99,95%** | Um sistema que não alerta é um sistema em que ninguém sabe que há problema. É a única capacidade cuja falha é invisível por definição. |
| Consulta | 99,5% (informativo) | Consulta indisponível atrapalha a investigação, mas não impede a detecção. |

## 5. Riscos e trade-offs

| Risco | Impacto | Mitigação | NFR afetado |
|-------|---------|-----------|-------------|
| Fail-open esconde perda de dado | Decisão tomada sobre telemetria incompleta | `dropped_total` obrigatório + `telemetry.ingest.degraded` acima de 1% | NFR-006 |
| Explosão de cardinalidade | Armazenamento saturado; consultas lentas | Orçamento + quarentena antes da série existir | NFR-009 |
| *Tail sampling* concentra spans em um gateway | Réplica perdida degrada amostragem | Roteamento consistente por `hash(trace_id)`; degradação parcial, sem corrupção | NFR-003 |
| Retenção curta de traces | Investigação antiga sem trace | Métricas agregadas e logs cobrem a janela longa | NFR-012 |
| Consulta abusiva satura o armazenamento | Painéis lentos para todos | Orçamento de tempo e de séries varridas por tenant | NFR-013 |
| Alerta ruidoso leva a silêncio permanente | Cegueira operacional | Histerese, deduplicação, silêncio com prazo obrigatório | NFR-008, NFR-016 |

## 6. Referências

- Fonte: `./_DESIGN_BRIEF.md` §7.2
- Métricas citadas: `./Metrics.md` · Alertas: `./Monitoring.md`
- Metodologia de medição: `./Benchmark.md` · Testes: `./Testing.md`
- Rastreabilidade: `./Requirements.md`

*Fim de `NonFunctionalRequirements.md`.*
