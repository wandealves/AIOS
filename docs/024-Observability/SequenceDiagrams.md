---
Documento: SequenceDiagrams
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0242, ADR-0243, ADR-0244, ADR-0245
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: UseCases.md, API.md, Events.md, StateMachine.md
---

# 024-Observability — Diagramas de Sequência

> Todos os fluxos propagam `traceparent` (W3C Trace Context) e `X-AIOS-Tenant`
> conforme `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6. Timeouts indicados são
> os defaults de `./Configuration.md`.

## 1. UC-001 — Ingestão OTLP (caminho quente)

```
 Serviço      otel-agent      otel-gateway    SignalAdmission  Cardinality  Pipelines
 emissor      (por nó)                                          Guard
   │             │                 │                │              │           │
   │ export OTLP │                 │                │              │           │
   │────────────▶│ buffer em disco │                │              │           │
   │◀── ACK ─────│ (RPO 1 min)     │                │              │           │
   │  (local,    │                 │                │              │           │
   │   ~1 ms)    │ forward (mTLS)  │                │              │           │
   │             │────────────────▶│ valida tenant  │              │           │
   │             │                 │ vs. certificado│              │           │
   │             │                 │───────┐        │              │           │
   │             │                 │◀──────┘ ok     │              │           │
   │             │                 │ admite sinais  │              │           │
   │             │                 │───────────────▶│              │           │
   │             │                 │                │ conta séries │           │
   │             │                 │                │─────────────▶│           │
   │             │                 │                │◀── dentro ───│           │
   │             │                 │◀── admitido ───│  do orçamento│           │
   │             │                 │ redige PII + enriquece recurso│           │
   │             │                 │───────┐                       │           │
   │             │                 │◀──────┘                       │           │
   │             │                 │ ENFILEIRA ────────────────────────────────▶│
   │             │◀── ACK ─────────│  (ACK aqui, NÃO após a escrita)           │
   │             │  (p99 ≤ 50 ms)  │                                │          │
   │             │                 │                     escreve ───────────────▶ Prometheus
   │             │                 │                     assíncrono ──────────────▶ Tempo
   │             │                 │                                ─────────────▶ Seq
```

> O ACK ocorre **após o enfileiramento**, nunca após a escrita no armazenamento.
> Esperar a persistência transformaria a latência do Prometheus na latência do serviço
> observado (ADR-0243).

## 2. UC-001 (A1) — Saturação: descarte sem backpressure

```
 Serviço        otel-gateway            fila            SelfTelemetry      NATS
 emissor
   │                 │                    │                   │              │
   │ export OTLP     │                    │                   │              │
   │────────────────▶│ enfileirar         │                   │              │
   │                 │───────────────────▶│ FILA CHEIA        │              │
   │                 │◀── rejeitada ──────│ (obs.ingest.      │              │
   │                 │                    │  queue_size)      │              │
   │                 │ DESCARTA o lote    │                   │              │
   │                 │ aios_obs_dropped_total +N ────────────▶│              │
   │◀── 200 OK ──────│                    │                   │              │
   │  (emissor NÃO   │                    │                   │              │
   │   é bloqueado)  │ ratio > 1%?        │                   │              │
   │                 │───────┐            │                   │              │
   │                 │◀──────┘ sim        │                   │              │
   │                 │ telemetry.ingest.degraded ────────────────────────────▶│
   │                 │                    │                   │              │
```

**O que NÃO acontece:** o gateway **não** devolve `429`, **não** segura a conexão e
**não** faz o SDK do emissor gastar CPU em retry. A perda é real, é medida e é
anunciada — mas é do observador, nunca do observado.

## 3. UC-003 — Explosão de cardinalidade e quarentena

```
 Deploy novo   otel-gateway   CardinalityGuard   SignalRegistry    NATS     Dono do sinal
     │              │                │                  │            │            │
     │ métrica com  │                │                  │            │            │
     │ label novo   │                │                  │            │            │
     │─────────────▶│ admite         │                  │            │            │
     │              │───────────────▶│ conta séries     │            │            │
     │              │                │ 780k / 1M (78%)  │            │            │
     │              │◀── ok ─────────│                  │            │            │
     │  … minutos …                  │                  │            │            │
     │─────────────▶│───────────────▶│ 820k (82%)       │            │            │
     │              │                │ > soft_threshold │            │            │
     │              │                │ alerta preventivo ───────────▶│───────────▶│
     │  … minutos …                  │                  │            │            │
     │─────────────▶│───────────────▶│ 1,02M (>100%)    │            │            │
     │              │                │ hard_action=quarantine        │            │
     │              │                │─────────────────▶│ Registered │            │
     │              │                │                  │ → Quarantined           │
     │              │◀── QUARENTENA ─│                  │            │            │
     │              │ descarta o sinal│                 │            │            │
     │              │ telemetry.cardinality.exceeded ───────────────▶│───────────▶│
     │              │ telemetry.signal.quarantined ─────────────────▶│           │
     │              │                │                  │            │            │
     │              │       alertas que dependem do sinal ⇒ Expired (T-09)        │
     │              │       (NUNCA Resolved — invariante I6/S3)                   │
```

## 4. UC-005 — *Tail-based sampling*

```
 Serviço A     Serviço B     otel-gateway (hash(trace_id))    TraceSampler   TraceStore
    │              │                    │                          │             │
    │ span root    │                    │                          │             │
    │───────────────────────────────────▶│ acumula                 │             │
    │              │ span filho          │                          │             │
    │              │────────────────────▶│ acumula                 │             │
    │              │ span filho (ERROR)  │                          │             │
    │              │────────────────────▶│ acumula                 │             │
    │              │                    │ fim da janela de montagem│             │
    │              │                    │─────────────────────────▶│             │
    │              │                    │      status=ERROR?  sim  │             │
    │              │                    │◀── RETÉM 100% ───────────│             │
    │              │                    │ escreve ──────────────────────────────▶│
    │              │                    │                          │             │
    │  ── outro trace, sem erro, 40 ms ──                          │             │
    │───────────────────────────────────▶│─────────────────────────▶│             │
    │              │                    │  sem erro ∧ < 1000 ms    │             │
    │              │                    │◀── amostra (10%) ────────│             │
    │              │                    │ descarta (90% dos casos) │             │
```

Retém-se **o que explica falha**, não uma fatia estatística cega — que é justamente o
defeito do *head sampling* puro (ADR-0244).

## 5. UC-007 + UC-008 — De SLI degradado a plantão acordado

```
 SloEngine        ledger        AlertEvaluator     AlertRouter    Alertmanager   SRE
     │               │                │                 │              │          │
     │ avalia SLI (60 s)             │                 │              │          │
     │──────────────▶│ grava lançamento                │              │          │
     │  budget_consumed=0,31          │                 │              │          │
     │  burn_1h=15,2  burn_6h=6,4    │                 │              │          │
     │───────────────────────────────▶│ limiar CRITICAL │              │          │
     │               │                │ (14,4 ∧ 6)      │              │          │
     │               │                │ T-01 → Pending  │              │          │
     │               │                │ … for_duration …│              │          │
     │               │                │ T-02 → Firing   │              │          │
     │               │                │ TRANSAÇÃO: estado + outbox     │          │
     │               │                │────────────────▶│ agrupa+dedupe│          │
     │               │                │                 │─────────────▶│ notifica │
     │               │                │                 │              │─────────▶│
     │               │                │                 │  + link do RUNBOOK      │
     │               │                │                 │              │          │
     │  … sem ack em obs.alert.escalation_after_s (15 min) …           │          │
     │               │                │ T-08 escalona   │─────────────▶│─────────▶│ nível 2
     │               │                │ telemetry.alert.escalated ─────────────────▶ NATS
```

## 6. UC-011 (A1) — Ausência de dado ≠ resolução

```
 Alvo de coleta    otel-gateway    AlertEvaluator            NATS         SRE
      │                 │                 │                    │           │
      │ métricas ok     │                 │                    │           │
      │────────────────▶│────────────────▶│ condição verdadeira│           │
      │                 │                 │ estado = Firing    │           │
      │                 │                 │                    │           │
      │   ✗ ALVO CAI (deixa de reportar)  │                    │           │
      │                 │                 │ sem dado …         │           │
      │                 │                 │ … no_data_timeout (10 min) …   │
      │                 │                 │ T-09 → Expired     │           │
      │                 │                 │ telemetry.alert.nodata ───────▶│
      │                 │                 │                    │           │
      │  ❌ o que NÃO acontece: estado → Resolved                          │
      │     (o alvo que caiu deixou de reportar exatamente a               │
      │      métrica que provaria que ele caiu — invariante I6)            │
```

## 7. UC-012 — Consulta com PEP e isolamento de tenant

```
   SRE          QueryGateway      PolicyClient     022-Policy     TenantRouter   Prometheus
    │                 │                 │               │               │            │
    │ PromQL          │                 │               │               │            │
    │────────────────▶│ DecisionRequest │               │               │            │
    │                 │────────────────▶│  Evaluate     │               │            │
    │                 │                 │──────────────▶│               │            │
    │                 │                 │◀── allow ─────│               │            │
    │                 │◀────────────────│               │               │            │
    │                 │ restringe ao tenant do solicitante ────────────▶│            │
    │                 │                 │               │               │ query      │
    │                 │                 │               │               │───────────▶│
    │                 │ orçamento: max_duration_s=30, max_series=5M     │            │
    │                 │◀────────────────────────────────────────────────────────────│
    │◀── resultado ───│                 │               │               │            │
    │                 │                 │               │               │            │
    │  ── caso PDP indisponível ──                      │               │            │
    │────────────────▶│────────────────▶│   ⏱ timeout   │               │            │
    │                 │◀── CB aberto ───│               │               │            │
    │◀── NEGADO ──────│ fail_mode=closed (consulta)     │               │            │
    │                 │ ↳ a INGESTÃO continua normalmente (assimetria §Security §2)  │
```

## 8. UC-015 — Direito ao esquecimento sobre telemetria

```
 021-Security      NATS      RetentionManager   Prometheus  Tempo   Seq    025-Audit
      │              │              │                │        │      │         │
      │ rtbf.requested             │                │        │      │         │
      │─────────────▶│─────────────▶│ localiza ocorrências   │      │         │
      │              │              │───────────────▶│       │      │         │
      │              │              │────────────────────────▶│     │         │
      │              │              │─────────────────────────────▶│         │
      │              │              │ pseudonimiza/expurga    │      │         │
      │              │              │  (agregados sem identificação preservados)│
      │              │              │ comprovante ────────────────────────────▶│
      │              │              │                │        │      │         │
```

## 9. Fluxo de falha — PostgreSQL indisponível

```
   otel-gateway     SignalRegistry(cache)    PostgreSQL     observability-api
        │                    │                    │                 │
        │ admite sinal       │                    │                 │
        │───────────────────▶│ cache local válido │                 │
        │◀── admitido ───────│  (INGESTÃO SEGUE)  │                 │
        │                    │                    │                 │
        │                    │      ✗ indisponível│                 │
        │                    │                    │◀── registro ────│ RegisterSignal
        │                    │                    │  falha          │
        │                    │                    │─────────────────▶│ AIOS-OBS-0011
        │                    │                    │                 │
        │  ✅ ingestão e alerta continuam · ❌ administração e SLO param
```

A hierarquia de degradação (`./FailureRecovery.md` §5) é visível aqui: perde-se
primeiro a capacidade de **mudar** a observabilidade, nunca a de **observar**.

## 10. Referências

- Casos de uso: `./UseCases.md` · FSM: `./StateMachine.md`
- Contratos: `./API.md` · `./Events.md`
- Falhas: `./FailureRecovery.md` · Alertas: `./Monitoring.md`
- Convenções de diagrama: `../_templates/MODULE_TEMPLATE.md`

*Fim de `SequenceDiagrams.md`.*
