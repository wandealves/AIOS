---
Documento: Architecture
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0010, ADR-0241, ADR-0243, ADR-0244, ADR-0248
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: 001-Architecture, 005-Database, 020-Communication, 021-Security, 022-Policy, 025-Audit, 027-Cluster
---

# 024-Observability — Architecture

## 1. Visão C4 — Nível 1 (Contexto)

```
   ┌───────────────────────────────────────────────────────────────────────────┐
   │  TODOS OS MÓDULOS — emitem telemetria por construção (ADR-0010)          │
   │  004 · 005 · 006 · 007 · 008 · 009 · 010 · 011 · … · 022 · 025 · 026     │
   └──────────────┬────────────────────────────────────────────────────────────┘
                  │ OTLP (gRPC :4317 / HTTP :4318) · mTLS · traceparent
                  ▼
   ┌───────────────────────────────────────────────────────────────────────────┐
   │              024-OBSERVABILITY — PLANO DE TELEMETRIA (este módulo)         │
   │  admite · governa cardinalidade · redige PII · amostra · armazena         │
   │  calcula SLI/SLO · dispara e roteia alertas · publica dashboards/runbooks │
   └───┬──────────┬────────────┬─────────────┬──────────────┬─────────────────┘
       │          │            │             │              │
       ▼          ▼            ▼             ▼              ▼
   Prometheus  Tempo/Jaeger   Seq      Alertmanager    PostgreSQL
   (métricas)   (traces)     (logs)   (notificação)   (plano de controle)
       │          │            │             │
       └──────────┴────────────┴─────────────┴──▶ 029-Operations (plantão)
                                                  028-Deployment (gate de release)
                                                  022-Policy (gate de publicação)
```

## 2. Visão C4 — Nível 2 (Contêiner)

| Contêiner | Tecnologia | Papel | Justificativa |
|-----------|-----------|-------|---------------|
| `otel-agent` (DaemonSet) | OTel Collector | Coleta local por nó, buffer em disco, primeiro *hop* de baixa latência. | Isola o emissor da rede; sustenta o RPO de 1 min de telemetria bruta (NFR-011). |
| `otel-gateway` | OTel Collector + processadores próprios | Admissão, cardinalidade, redação de PII, *tail sampling*, roteamento. | O *tail-based sampling* exige que todos os spans de um trace convirjam (§7). |
| `observability-api` | .NET 10 (ASP.NET Core + gRPC) | Registro de sinais, SLO, alertas, silêncios, dashboards, consulta unificada. | Control plane em .NET (ADR-0003). |
| `slo-evaluator` | .NET 10 (worker) | Avaliação periódica de SLI/SLO e escrita do ledger de error budget. | Carga periódica isolada do caminho de ingestão. |
| `alert-evaluator` | .NET 10 (worker) | Avaliação de regras, FSM do alerta, escalonamento. | Caminho crítico independente do de consulta (NFR-005). |
| `observability-outbox-relay` | .NET 10 (worker) | Publica o outbox em JetStream. | At-least-once sem 2PC (padrão do repositório). |
| Prometheus (+ longo prazo) | Prometheus 2.x + objeto em MinIO | Séries temporais, *recording rules*, *downsampling*. | Padrão do AIOS (`../001-Architecture/` §4). |
| Tempo / Jaeger | Grafana Tempo ou Jaeger | Armazenamento de traces. | Compatível com OTLP; escolha operacional em `../028-Deployment/`. |
| Seq | Seq | Log estruturado (Serilog). | Padrão do AIOS. |
| Alertmanager | Alertmanager | Entrega, agrupamento e silêncio de notificações. | Integra com Prometheus e com o roteamento próprio. |
| PostgreSQL (schema `telemetry`) | PostgreSQL 16 | **Plano de controle** da observabilidade. | Unificação relacional (ADR-0005); RLS por `tenant_id`. |
| Redis | Redis 7 | Contadores de cardinalidade e limites de ingestão. | Estado quente (ADR-0006). |
| NATS / JetStream | NATS 2.x | Eventos `aios.<tenant>.telemetry.*`. | Barramento primário (ADR-0004). |

```
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     por NÓ
   │ otel-agent   │   │ otel-agent   │   │ otel-agent   │  (DaemonSet,
   │ + disk buf   │   │ + disk buf   │   │ + disk buf   │   buffer local)
   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
          └──────────────────┼──────────────────┘
                             │ roteamento consistente por hash(trace_id)
                  ┌──────────▼───────────┐
                  │    otel-gateway      │  (N réplicas)
                  │ admissão·cardinalid. │
                  │ PII·tail sampling    │
                  └──┬────────┬──────┬───┘
                     ▼        ▼      ▼
              Prometheus   Tempo    Seq
                     ▲        ▲      ▲
   ┌─────────────────┴────────┴──────┴─────────────────────────────────┐
   │  observability-api (registro·SLO·alertas·consulta·PEP)            │
   └───┬───────────────────┬───────────────────┬───────────────────────┘
       │                   │                   │
  ┌────▼─────┐      ┌──────▼───────┐    ┌──────▼────────────────┐
  │PostgreSQL│      │slo-evaluator │    │ alert-evaluator       │
  │telemetry │      │(ledger)      │    │ → Alertmanager → NATS │
  └──────────┘      └──────────────┘    └───────────────────────┘
```

## 3. Visão C4 — Nível 3 (Componente)

```
    OTLP gRPC :4317 / HTTP :4318 (mTLS)          REST /v1/observability (admin/consulta)
                    │                                          │
   ┌────────────────▼──────────────────────────────────────────▼─────────────────┐
   │                 OBSERVABILITY SERVICE (024 · .NET 10 + OTel Collector)        │
   │                                                                               │
   │  ┌──────────────────────────┐        ┌────────────────────────────────────┐  │
   │  │ TelemetryIngestGateway   │        │  QueryGateway (PromQL/TraceQL/Seq) │  │
   │  │ (fail-open · rate limit) │        │  + PolicyClient (PEP) ─────────────┼──┼─▶ 022
   │  └──────────┬───────────────┘        └────────────────┬───────────────────┘  │
   │             ▼                                          │                      │
   │  ┌──────────────────────┐   ┌──────────────────┐      │                      │
   │  │  SignalAdmission     │◀─▶│  SignalRegistry  │◀─────┘                      │
   │  └──────────┬───────────┘   └────────┬─────────┘                              │
   │             │                        │                                        │
   │  ┌──────────▼───────────┐   ┌────────▼─────────┐   ┌────────────────────────┐│
   │  │  CardinalityGuard    │   │   PiiRedactor    │   │   ResourceEnricher     ││
   │  └──────────┬───────────┘   └────────┬─────────┘   └───────────┬────────────┘│
   │             └───────────┬────────────┴─────────────────────────┘             │
   │                          ▼                                                    │
   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
   │   │MetricPipeline│  │ TracePipeline│  │  LogPipeline │  │  TraceSampler   │  │
   │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────────────────┘  │
   │          │          ┌──────▼─────────────────▼──────┐                         │
   │          │          │        TenantRouter           │                         │
   │          │          └───────────────┬───────────────┘                         │
   │   ┌──────▼──────────┐               │            ┌──────────────────────────┐ │
   │   │   SloEngine     │               │            │   RetentionManager       │ │
   │   └──────┬──────────┘               │            └──────────────────────────┘ │
   │   ┌──────▼──────────┐  ┌────────────▼───────┐   ┌──────────────────────────┐ │
   │   │ AlertEvaluator  │─▶│    AlertRouter     │──▶│ DashboardRegistry        │ │
   │   └─────────────────┘  └────────────────────┘   │ RunbookRegistry          │ │
   │                                                  └──────────────────────────┘ │
   │  ┌─────────────────────────────────────────────────────────────────────────┐ │
   │  │ EventEmitter (Outbox → JetStream)  ·  SelfTelemetry (caminho separado)   │ │
   │  └─────────────────────────────────────────────────────────────────────────┘ │
   └──────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Responsabilidade por componente

| Componente | Responsabilidade | Depende de |
|------------|------------------|-----------|
| **TelemetryIngestGateway** | Receptor OTLP sob mTLS; valida envelope, extrai `traceparent` e `tenant`, limita por tenant e **nunca bloqueia o emissor**. | SignalAdmission, ResourceEnricher |
| **SignalAdmission** | Admite ou rejeita contra o registro; quarentena o desconhecido sem derrubar o lote. | SignalRegistry, CardinalityGuard |
| **SignalRegistry** | Registro autoritativo dos `SignalDescriptor` (FSM em `./StateMachine.md` §5). | → PostgreSQL |
| **CardinalityGuard** | Orçamento de séries por tenant e por sinal; quarentena e evento. | SignalRegistry; → Redis |
| **PiiRedactor** | Redige/tokeniza atributos pessoais **antes** de persistir. | SignalRegistry |
| **ResourceEnricher** | Injeta `service.name`, `service.version`, `deployment.environment`, `aios.tenant`, `aios.replica`, `aios.shard`. | → 027, 028 |
| **TraceSampler** | *Head sampling* barato no SDK + *tail-based* no gateway (100% de erro e cauda). | SloEngine |
| **MetricPipeline** | Agregação, *drop* de série quarentenada, *recording rules*, roteamento. | CardinalityGuard; → Prometheus |
| **TracePipeline** | Montagem e deduplicação de span; escrita no `TraceStore`. | TraceSampler; → Tempo/Jaeger |
| **LogPipeline** | Normalização Serilog, correlação, roteamento. | PiiRedactor; → Seq |
| **TenantRouter** | Isolamento multi-tenant do dado de telemetria. | PolicyClient |
| **SloEngine** | SLI, SLO versionado e ledger de error budget; *burn rate*. | SignalRegistry; → PostgreSQL, Prometheus |
| **AlertEvaluator** | Regras, histerese, FSM do alerta. | SloEngine, SignalRegistry |
| **AlertRouter** | Agrupa, deduplica, silencia, escalona e entrega. | AlertEvaluator, EventEmitter; → Alertmanager |
| **DashboardRegistry** | Dashboards como código; anotações de rollout e de política. | → Grafana |
| **RunbookRegistry** | Runbooks versionados; alerta sem runbook é inválido. | AlertEvaluator |
| **QueryGateway** | Consulta unificada com PEP, limites de custo e timeout. | PolicyClient, TenantRouter |
| **RetentionManager** | Camadas, *downsampling*, expurgo, direito ao esquecimento. | → Prometheus, Tempo, Seq, MinIO |
| **SelfTelemetry** | Meta-observabilidade por caminho independente do pipeline. | → Prometheus (scrape direto) |
| **PolicyClient** | PEP de consulta e administração; *fail-closed*. | → 022-Policy |
| **EventEmitter** | Outbox transacional → JetStream. | → PostgreSQL, NATS |

## 4. Fronteiras arquiteturais

| Fronteira | Contrato | Documento |
|-----------|----------|-----------|
| Emissores → 024 | OTLP + convenções semânticas (RFC-0240) | `./API.md` §2 |
| 024 → `029-Operations` | `telemetry.alert.*` + runbook referenciado | `./Events.md` |
| 024 → `028-Deployment` | `telemetry.budget.exhausted` (gate de release) | `./Events.md` |
| 024 → `022-Policy` | PEP de consulta; consumo de `policy.bundle.updated` | `./Security.md` |
| 024 → `025-Audit` | Fatos de governança (silêncio, alteração de SLO) | `../025-Audit/` |
| 024 ↔ `021-Security` | mTLS dos emissores; RTBF sobre telemetria | `../021-Security/` |

> A fronteira mais importante é com o `../025-Audit/`: **telemetria é amostrável e
> perecível; auditoria não é** (ADR-0246). Um sistema que usa a mesma loja para as
> duas coisas acaba ou com uma auditoria que perde eventos, ou com uma telemetria cara
> demais para ser completa.

## 5. Padrões arquiteturais adotados

| Padrão | Onde | Por quê | ADR |
|--------|------|---------|-----|
| **Agente + gateway (duas camadas de coleta)** | `otel-agent` → `otel-gateway` | Baixa latência para o emissor; agregação e *tail sampling* no centro. | ADR-0244 |
| **Schema registry de telemetria** | `SignalRegistry` | Telemetria como contrato declarado, não emergente. | ADR-0241 |
| **Admission control** | `SignalAdmission` + `CardinalityGuard` | Impedir a série antes de ela existir — depois, o custo já foi pago. | ADR-0242 |
| **Fail-open com descarte contabilizado** | `TelemetryIngestGateway` | A observabilidade não pode derrubar o observado. | ADR-0243 |
| **Tail-based sampling** | `TraceSampler` | Reter 100% do que explica falha, amostrar o resto. | ADR-0244 |
| **Multi-window multi-burn-rate** | `SloEngine`, `AlertEvaluator` | Separar queima rápida de degradação lenta. | ADR-0245 |
| **Retenção em camadas com downsampling** | `RetentionManager` | Custo proporcional ao valor do dado ao longo do tempo. | ADR-0248 |
| **Everything as code** | `DashboardRegistry`, `RunbookRegistry`, `AlertRule` | Revisão, histórico e teste para painel, alerta e procedimento. | ADR-0247 |
| **Transactional Outbox** | `EventEmitter` | Transição de alerta e evento na mesma transação (invariante I5). | ADR-0004 |
| **Meta-observabilidade fora de banda** | `SelfTelemetry` | A falha do pipeline não pode ser invisível por depender do pipeline. | ADR-0010 |

## 6. Alternativas descartadas

| Alternativa | Por que foi descartada | ADR |
|-------------|------------------------|-----|
| **Telemetria sem registro prévio** ("aceita tudo, resolve depois") | Cardinalidade só é detectada depois de paga; nomes e unidades divergem entre módulos. | ADR-0241 |
| **Backpressure no emissor sob saturação** | Transforma falha de telemetria em indisponibilidade de produto. | ADR-0243 |
| **Head sampling puro (taxa fixa)** | Descarta justamente os traces com erro, que são os raros e os úteis. | ADR-0244 |
| **Reter tudo, sem amostragem nem downsampling** | Custo cresce com a população de agentes; inviável em 10⁶. | ADR-0248 |
| **Séries temporais no PostgreSQL** | O observador viraria o maior consumidor de I/O do sistema. | ADR-0005 |
| **Usar a trilha de auditoria como armazenamento de telemetria** | Ou a auditoria perde eventos, ou a telemetria fica cara demais. | ADR-0246 |
| **APM proprietário** | *Vendor lock-in* e perda do controle sobre cardinalidade e retenção. | ADR-0010 |
| **Alerta sem runbook obrigatório** | O plantonista silencia em vez de tratar. | ADR-0247 |

## 7. Decisões estruturais e seus riscos

| Decisão | Risco assumido | Mitigação |
|---------|----------------|-----------|
| Fail-open na ingestão | Perda silenciosa de telemetria | `dropped_total` obrigatório + evento `ingest.degraded` acima do limiar (FR-002, NFR-006). |
| *Tail sampling* central | Todos os spans de um trace precisam do mesmo gateway | Roteamento consistente por `hash(trace_id)`; réplica perdida degrada amostragem, não corrompe traces. |
| Registro obrigatório de sinal | Atrito para o emissor adicionar métrica | Registro é operação de API idempotente, versionada e automatizável no pipeline do módulo dono. |
| Quarentena automática | Perda temporária de visibilidade do sinal infrator | Quarentena é **reversível** (`./StateMachine.md` §5) e notifica o dono. |
| Retenção curta de traces (7 d) | Investigação antiga sem trace | Métricas agregadas e logs cobrem a janela longa; o trace é ferramenta de incidente recente. |
| PDP no caminho de consulta | Consulta indisponível se o PDP cair | *Fail-closed* deliberado **apenas** na consulta; a ingestão nunca depende do PDP (§`./Security.md` §2). |

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` (§2 componentes, §10 escalabilidade)
- Arquitetura global: `../001-Architecture/Architecture.md` §4
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`, `./ADR.md`
- Detalhamento: `./ClassDiagrams.md`, `./SequenceDiagrams.md`, `./Database.md`,
  `./API.md`, `./Events.md`, `./Scalability.md`

*Fim de `Architecture.md`.*
