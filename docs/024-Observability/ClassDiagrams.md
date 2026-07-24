---
Documento: ClassDiagrams
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0243, ADR-0244, ADR-0245
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: Architecture.md, StateMachine.md, API.md
---

# 024-Observability — Diagramas de Classe e Contratos

> As interfaces abaixo são **conceituais** (este repositório é de documentação, não de
> implementação) e servem para fixar as fronteiras testáveis do módulo, referenciadas
> em `./Testing.md`. Nomes de componentes são idênticos aos de `./Architecture.md` §3.1.

## 1. Visão estrutural do caminho de ingestão

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │                     TelemetryIngestGateway                           │
   │  + Ingest(batch: OtlpBatch, ctx: EmitterContext): IngestAck          │
   │  - ValidateTenant(batch, certIdentity): void                         │
   │  - ApplyRateLimit(tenantId): void                                    │
   │  invariante: NUNCA bloqueia o emissor (fail-open, ADR-0243)          │
   └────────────────────────────────┬─────────────────────────────────────┘
                                    │ ◆ (composição)
                                    ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                        SignalAdmission                               │
   │  + Admit(signals: Signal[]): AdmissionResult                         │
   │  invariante: sinal desconhecido quarentena SÓ AQUELE sinal           │
   └──┬──────────────┬──────────────┬───────────────┬─────────────────────┘
      │              │              │               │
      ▼              ▼              ▼               ▼
 ┌──────────┐ ┌─────────────┐ ┌────────────┐ ┌──────────────────┐
 │ Signal   │ │ Cardinality │ │PiiRedactor │ │ ResourceEnricher │
 │ Registry │ │   Guard     │ │            │ │                  │
 └──────────┘ └─────────────┘ └────────────┘ └──────────────────┘
                     │
                     ▼  (fila assíncrona — o ACK já foi devolvido)
   ┌──────────────────────────────────────────────────────────────────────┐
   │   MetricPipeline  ·  TracePipeline  ·  LogPipeline  ·  TraceSampler  │
   └──────────────────────────────────┬───────────────────────────────────┘
                                      ▼
                              ┌───────────────┐
                              │ TenantRouter  │
                              └───────────────┘
```

## 2. Interfaces do caminho de ingestão

```
interface ITelemetryIngestGateway {
  IngestAck Ingest(OtlpBatch batch, EmitterContext ctx);
  // IngestAck NUNCA carrega instrução de retry/backoff (ADR-0243)
}

interface ISignalAdmission {
  AdmissionResult Admit(Signal[] signals, string tenantId);
  // AdmissionResult = { Signal[] accepted, QuarantinedSignal[] quarantined }
}

interface ISignalRegistry {
  SignalDescriptor? Lookup(string name);                  // cache local + PostgreSQL
  SignalDescriptor  Register(SignalDescriptorDraft draft); // valida RFC-0240
  void              SetState(string name, SignalState target, long expectedVersion);
}

interface ICardinalityGuard {
  CardinalityVerdict Check(Signal signal, string tenantId);
  // Verdict = { Ok | SoftThresholdExceeded | HardLimitExceeded(action) }
  long ActiveSeries(string tenantId, string? signalName);
}

interface IPiiRedactor {
  Signal Redact(Signal signal, DataClass dataClass, RedactionMode mode);
  // executa ANTES de qualquer persistência (ADR-0249)
}

interface IResourceEnricher {
  Signal Enrich(Signal signal, NodeContext node);
  // service.name · service.version · deployment.environment
  // aios.tenant · aios.replica · aios.shard
}

interface ITraceSampler {
  SamplingDecision DecideTail(Trace assembled);
  // ERROR em qualquer span  ⇒ Keep(1.0)
  // duração > tail_keep_slow_ms ⇒ Keep(1.0)
  // caso contrário ⇒ Keep(ratio)
}

interface ITenantRouter {
  StorageTarget RouteWrite(Signal signal);
  QueryScope    RestrictRead(QueryRequest req, string authenticatedTenant);
}
```

### 2.1 Contrato de dados da ingestão (RFC-0240)

```
record OtlpBatch {
  ResourceAttributes resource;   // DEVE conter aios.tenant e service.name
  Signal[]           signals;    // métricas, spans ou logs
}

record Signal {
  SignalKind          kind;      // Metric | Span | LogEvent
  string              name;      // aios_<subsistema>_<nome>_<unidade> | <componente>.<evento>
  Map<string,string>  labels;    // subconjunto de allowed_labels
  object              value;     // datapoint | span | log record
  string              traceparent;
}

record IngestAck {
  bool accepted;                 // sempre true em operação normal (fail-open)
  int  droppedCount;             // informativo — NÃO é instrução de retry
}
```

**Invariantes de contrato:**

- (C1) `IngestAck` **nunca** instrui retry ou backoff: fail-open (ADR-0243).
- (C2) `labels ⊆ descriptor.allowed_labels`; labels fora do conjunto são removidos,
  não causam rejeição do sinal.
- (C3) Nenhum label pertence a `forbidden_labels` (identidade) — verificado no
  registro **e** em runtime (ADR-0242).
- (C4) Todo `Signal` persistido passou por `IPiiRedactor` (ADR-0249).
- (C5) Todo `Signal` persistido tem `service.name`, `aios.tenant` e
  `deployment.environment` preenchidos pelo `ResourceEnricher`.
- (C6) `traceparent` propagado conforme RFC-0001 §5.6 — sem ele, não há pivô
  log↔trace.

## 3. Estruturas de SLO e alerta

```
   ┌──────────────────────┐  avalia   ┌─────────────────────────┐
   │     SloEngine        │──────────▶│  ErrorBudgetLedger      │
   │ + Evaluate(slo):     │           │ - sliValue              │
   │     BudgetEntry      │           │ - budgetConsumed        │
   │ + BurnRate(w): decimal│          │ - burnRate1h / 6h       │
   └──────────┬───────────┘           └─────────────────────────┘
              │ alimenta
              ▼
   ┌──────────────────────┐   produz   ┌─────────────────────────┐
   │   AlertEvaluator     │───────────▶│     AlertInstance       │
   │ + Evaluate(rule):    │            │ - state (FSM §4)        │
   │     AlertInstance?   │            │ - fingerprint (dedupe)  │
   │ + Transition(inst,to)│            │ - escalationLevel       │
   └──────────┬───────────┘            └───────────┬─────────────┘
              │ ◆                                   │ silenciada por
              ▼                                     ▼
   ┌──────────────────────┐              ┌─────────────────────┐
   │     AlertRouter      │              │      Silence        │
   │ + Route(inst): void  │              │ - expiresAt (NN)    │
   │ + Silence(matcher)   │              │ - reason (NN)       │
   │ + Escalate(inst)     │              └─────────────────────┘
   └──────────────────────┘
              │ exige
              ▼
   ┌──────────────────────┐
   │   RunbookRegistry    │  ← alerta sem runbook é inválido (ADR-0247)
   └──────────────────────┘
```

```
interface ISloEngine {
  BudgetEntry Evaluate(SloDefinition slo, TimeWindow window);
  decimal     BurnRate(SloDefinition slo, TimeSpan window);
  bool        IsBudgetExhausted(SloDefinition slo);
}

interface IAlertEvaluator {
  AlertInstance? Evaluate(AlertRule rule, EvaluationContext ctx);
  void           Transition(AlertInstance inst, AlertState target, long expectedVersion);
  // guarda de T-02: rule.RunbookUrn DEVE resolver para um runbook válido
}

interface IAlertRouter {
  void Route(AlertInstance inst);        // agrupa · deduplica · entrega
  void Escalate(AlertInstance inst);     // T-08
  bool IsSilenced(AlertInstance inst);   // suprime NOTIFICAÇÃO, não avaliação (I3)
}

interface IRunbookRegistry {
  Runbook? Resolve(string runbookUrn);
  bool     IsStale(Runbook rb, TimeSpan maxAge);   // 180 dias (NFR-016)
}

interface IQueryGateway {
  QueryResult Query(QueryRequest req, AuthContext ctx);
  // PEP: consulta PolicyClient antes de tocar o armazenamento
}

interface IRetentionManager {
  void Downsample(RetentionTier from, RetentionTier to);
  void Purge(RetentionTier tier, DateTimeOffset olderThan);
  ErasureReceipt Erase(SubjectRef subject);   // direito ao esquecimento (UC-015)
}

interface ISelfTelemetry {
  HealthSnapshot Snapshot();   // exposto por caminho INDEPENDENTE do pipeline
}
```

## 4. Relações entre componentes

| Relação | Tipo | Semântica |
|---------|------|-----------|
| `TelemetryIngestGateway` ◆── `SignalAdmission` | Composição | Não há rota de ingestão que ignore a admissão. |
| `SignalAdmission` ◇── `SignalRegistry` | Agregação | O registro é compartilhado e cacheado; sua indisponibilidade **não** interrompe a ingestão (cache local). |
| `SignalAdmission` ──▶ `CardinalityGuard` | Dependência | Contagem amostrada, fora do caminho crítico por lote. |
| `MetricPipeline` ──▶ `TenantRouter` | Dependência | Todo write passa pelo roteador; não há escrita sem rótulo de tenant. |
| `AlertEvaluator` ◆── `AlertInstance` | Composição | O ciclo de vida da instância pertence ao avaliador. |
| `AlertRouter` ──▶ `RunbookRegistry` | Dependência | A notificação carrega o link do runbook; sem ele não há transição a `Firing`. |
| `SloEngine` ──▶ `SignalRegistry` | Dependência | SLO só referencia sinal `Registered` (invariante S3). |
| `QueryGateway` ◆── `PolicyClient` | Composição | O PEP não é alcançável fora do gateway de consulta. |
| `SelfTelemetry` ⊘ pipeline | **Ausência** de dependência | Deliberada: a meta-observabilidade não pode depender do que observa. |

## 5. Invariantes estruturais

| ID | Invariante | Consequência do rompimento |
|----|-----------|----------------------------|
| S1 | O ACK ao emissor ocorre após o **enfileiramento**, nunca após a persistência. | A latência do armazenamento viraria latência do serviço observado (NFR-002 cai). |
| S2 | Nenhum caminho de ingestão devolve backpressure ao emissor. | Falha de telemetria vira indisponibilidade de produto (ADR-0243). |
| S3 | Nenhum sinal é persistido sem passar por `PiiRedactor`. | PII em claro em disco (NFR-014 cai). |
| S4 | Nenhuma escrita ocorre sem `TenantRouter`. | Vazamento cross-tenant (NFR-017 cai). |
| S5 | `SelfTelemetry` não depende do pipeline de ingestão. | A única falha que ela não reportaria seria a do próprio pipeline. |
| S6 | `AlertRouter.IsSilenced` suprime notificação, **nunca** avaliação. | O silêncio esconderia a resolução tanto quanto o problema (invariante I3). |
| S7 | `SignalDescriptor` é imutável a partir de `Registered`. | Séries históricas mudariam de significado retroativamente. |

## 6. Modelo de domínio (entidades persistidas)

```
   ┌────────────────────┐        ┌────────────────────┐
   │  SignalDescriptor  │        │ CardinalityBudget  │
   │ - name / version   │        │ - maxActiveSeries  │
   │ - allowedLabels    │        │ - hardAction       │
   │ - forbiddenLabels  │        └────────────────────┘
   │ - state (FSM §7)   │
   │ - retentionTier ───┼──────▶ ┌────────────────────┐
   └─────────┬──────────┘        │  RetentionPolicy   │
             │ referenciado por  │ - retainFor        │
             ▼                   │ - downsampleTo     │
   ┌────────────────────┐        └────────────────────┘
   │   SloDefinition    │ 1    * ┌────────────────────┐
   │ - sliQuery         │────────│ ErrorBudgetEntry   │
   │ - objective        │        │ - burnRate1h / 6h  │
   └─────────┬──────────┘        └────────────────────┘
             │ slo_urn
             ▼
   ┌────────────────────┐ 1    * ┌────────────────────┐ *   * ┌──────────┐
   │     AlertRule      │────────│   AlertInstance    │───────│ Silence  │
   │ - expr / forDur.   │        │ - state / finger.  │       └──────────┘
   │ - runbookUrn (NN) ─┼──▶ ┌──────────┐
   └────────────────────┘    │ Runbook  │
                              └──────────┘
   ┌────────────────────┐
   │     Dashboard      │──[signalsUsed]──▶ SignalDescriptor
   └────────────────────┘
```

Detalhamento físico (tipos, constraints, índices) em `./Database.md`.

## 7. Referências

- Componentes: `./Architecture.md` §3.1 · FSM: `./StateMachine.md`
- Contratos externos: `./API.md` · `./Events.md`
- Fonte única de verdade: `./_DESIGN_BRIEF.md` §2 e §5
- Testes das fronteiras: `./Testing.md` §3

*Fim de `ClassDiagrams.md`.*
