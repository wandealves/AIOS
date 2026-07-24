---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0010 (globais, herdados); ADR-0240..ADR-0249 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — não redefinida); RFC-0240 (Telemetry Signal & Semantic Conventions), RFC-0241 (SLO & Alerting Contract), a propor por este módulo
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database (schema `telemetry`), 020-Communication (NATS), 021-Security (identidade e mTLS), 022-Policy (PDP), 025-Audit (fronteira telemetria × prova), 027-Cluster, 028-Deployment, 029-Operations (plantão e runbooks); consumido por **todos** os módulos, que emitem métricas, traces e logs por construção (ADR-0010)
---

# 024-Observability — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 024-Observability. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, **cabeçalhos de correlação W3C Trace Context**,
> minimização de dados pessoais), este documento **referencia** a
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6, §5.8 e §7, e **não os redefine**.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 0. Posição no AIOS e contrato de fronteira

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  TODOS OS MÓDULOS — emitem telemetria por construção (ADR-0010)          │
   │  004 · 005 · 006 · 007 · 008 · 009 · 010 · 011 · … · 022 · 025 · 026     │
   │  métricas `aios_<subsistema>_*` · spans OTel · logs Serilog              │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ OTLP (gRPC/HTTP) · mTLS · traceparent (RFC-0001 §5.6)
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  024-OBSERVABILITY — PLANO DE TELEMETRIA (este módulo)                    │
   │  · ADMITE o sinal (registro obrigatório) e GOVERNA a cardinalidade       │
   │  · REDIGE PII na borda, amostra, enriquece, roteia e armazena            │
   │  · calcula SLI/SLO e ERROR BUDGET; dispara e roteia ALERTAS              │
   │  · publica dashboards e runbooks como código versionado                  │
   │  · NÃO é dono de nenhuma métrica de negócio e NÃO é prova de auditoria   │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │
        ┌──────────┼───────────────┬──────────────────┐
        ▼          ▼               ▼                  ▼
   Prometheus   Tempo/Jaeger     Seq             Alertmanager
   (métricas)   (traces)       (logs)            (notificação)
        │          │               │                  │
        └──────────┴───────┬───────┴──────────────────┘
                           ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  029-OPERATIONS  — consome alertas e runbooks; opera o plantão           │
   │  025-AUDIT       — trilha IMUTÁVEL, disjunta desta (§1.3, ADR-0246)      │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Fronteira em uma frase:** *o 024 responde **o que está acontecendo agora e como o
sistema se comporta**; o `025-Audit` responde **o que aconteceu e pode ser provado**.*
A telemetria é amostrável, tem retenção limitada e pode perder dados sob pressão; a
trilha de auditoria não pode nenhuma das três coisas. Tratá-las como o mesmo
subsistema produz ou uma auditoria que perde eventos, ou uma telemetria que derruba o
sistema que observa.

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O 024-Observability é o **sistema nervoso sensorial** do AIOS — o análogo do
subsistema de instrumentação e contadores de desempenho de um sistema operacional,
elevado à escala de uma plataforma multi-tenant com 10⁶ agentes autônomos. Ele não
decide nada, não executa nada e não guarda nenhuma verdade de negócio: ele torna
**visível e mensurável** o comportamento de todos os demais módulos, e transforma essa
visibilidade em **sinal acionável** — SLI, error budget, alerta e runbook.

A primeira tese do módulo é que **cardinalidade é o recurso escasso, não volume**. Um
sistema com 10⁶ agentes não quebra a observabilidade por gerar muitos bytes: quebra
por alguém usar `agent_urn` como label de métrica e criar 10⁶ séries temporais de uma
vez. Por isso o módulo opera um **registro de sinais** com admissão: métrica não
registrada é quarentenada, label de identidade é rejeitado, e cada tenant tem um
**orçamento de séries** tão real quanto seu orçamento de tokens.

A segunda tese é que **a observabilidade nunca pode derrubar o observado**. O 024 é
deliberadamente **fail-open** — o oposto do `022-Policy`. Sob saturação, o caminho
correto é **descartar telemetria e contabilizar o descarte**, jamais aplicar
backpressure ao serviço instrumentado. Um sistema que para de atender porque o
coletor de métricas está lento trocou o problema por um pior.

A terceira tese é que **SLO é dado versionado, não slide**. Objetivo de nível de
serviço, consumo de error budget e regra de alerta são artefatos com dono, versão,
histórico e teste — e é o consumo de error budget, não a intuição, que decide se um
release prossegue (`../028-Deployment/`) ou se uma publicação de política é adiada
(`../022-Policy/`).

### 1.2 Responsabilidades (o 024-Observability DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Operar o **pipeline OTLP** de ingestão de métricas, traces e logs de todos os módulos, sob mTLS e com correlação W3C (RFC-0001 §5.6). |
| R-02 | Manter o **registro de sinais** (`SignalDescriptor`): toda métrica, span e evento de log é declarado, versionado e admitido antes de ser armazenado. |
| R-03 | Governar **cardinalidade**: orçamento de séries por tenant e por serviço, proibição de labels de identidade, quarentena automática do sinal infrator. |
| R-04 | **Redigir PII na borda de ingestão**, antes de qualquer persistência (RFC-0001 §7). |
| R-05 | Aplicar **amostragem** de traces com política *tail-based*: 100% dos traces com erro e dos que excedem o SLO de latência, amostra dos demais. |
| R-06 | Enriquecer o sinal com atributos de recurso canônicos (`service`, `tenant`, `env`, `version`, `replica`) sem que cada módulo os invente. |
| R-07 | Armazenar e servir métricas (Prometheus + longo prazo), traces (Tempo/Jaeger) e logs (Seq), com **retenção em camadas** e *downsampling*. |
| R-08 | Calcular **SLI**, avaliar **SLO** e manter o **ledger de error budget** por serviço e por janela. |
| R-09 | Avaliar **regras de alerta** com *multi-window multi-burn-rate*, e gerir o ciclo de vida do alerta (§4): disparo, reconhecimento, silêncio, resolução. |
| R-10 | **Rotear notificações** com deduplicação, agrupamento, escalonamento e silêncios — sem decidir a resposta operacional. |
| R-11 | Publicar **dashboards e runbooks como código** versionado, vinculando cada alerta a um runbook existente. |
| R-12 | Expor **consulta unificada** (métricas, traces, logs) com isolamento por tenant, atuando como **PEP** perante o PDP do `022-Policy`. |
| R-13 | Aplicar **retenção e expurgo** por camada e suportar o direito ao esquecimento sobre a telemetria (RFC-0001 §7). |
| R-14 | **Observar a si mesmo**: expor a saúde e a perda do próprio pipeline por um caminho que não dependa do pipeline (§9). |
| R-15 | Emitir eventos de plataforma (`telemetry.alert.*`, `telemetry.slo.*`, `telemetry.cardinality.*`) para `029-Operations`, `028-Deployment` e `022-Policy`. |

### 1.3 Não-Responsabilidades (o 024-Observability NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Ser **prova de auditoria**. Telemetria é amostrável, com retenção curta e perda tolerável; a trilha imutável é de outro módulo. | `025-Audit` |
| NR-02 | **Ser dono** de qualquer métrica de negócio. Cada módulo define e emite as suas; o 024 admite, governa e armazena. | módulo emissor |
| NR-03 | **Decidir autorização** de consulta. O 024 é PEP; a decisão é do PDP. | `022-Policy` |
| NR-04 | **Responder ao incidente** (executar runbook, acionar plantão, decidir mitigação). O 024 dispara e roteia; a resposta é operação. | `029-Operations` |
| NR-05 | **Decidir rollback** de deploy ou de política. O 024 fornece o sinal (error budget, taxa de erro); a decisão é do dono. | `028-Deployment`, `022-Policy` |
| NR-06 | Definir **orçamento financeiro** ou custo de inferência. O 024 mede; o custo é contabilizado por quem o gera. | `026-Cost-Optimizer` |
| NR-07 | Emitir **identidade** ou credencial para os agentes coletores. | `021-Security` |
| NR-08 | Definir **topologia de rede, nós e capacidade** do cluster. | `027-Cluster`, `028-Deployment` |
| NR-09 | Detectar **fraude, abuso ou anomalia de segurança**. O 024 fornece o sinal bruto; a detecção de segurança é de quem tem o contexto. | `021-Security`, `025-Audit` |
| NR-10 | Aprender com a telemetria para **ajustar comportamento de agente**. | `023-Learning` |
| NR-11 | Guardar **conteúdo** de payload de negócio (prompt, resposta, item de memória) em log ou span. | módulo dono do dado |
| NR-12 | Substituir o **journal de decisões** do `022` ou o registro de custo do `026` por sua própria série temporal. | `022-Policy`, `026-Cost-Optimizer` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **TelemetryIngestGateway** | Receptor OTLP (gRPC 4317 / HTTP 4318) sob mTLS; valida envelope, extrai `traceparent` e `tenant`, aplica limite por tenant e **nunca bloqueia o emissor** (fail-open). | SignalAdmission, ResourceEnricher |
| **SignalAdmission** | Admite ou rejeita o sinal contra o `SignalRegistry`: sinal desconhecido → quarentena; label proibido → *drop* do label; violação de convenção → contador + evento. | SignalRegistry, CardinalityGuard |
| **SignalRegistry** | Registro autoritativo dos `SignalDescriptor` (métrica, span, evento de log): nome, tipo, unidade, labels permitidos, buckets, dono, versão e estado (§4.4). | → PostgreSQL (`telemetry.*`) |
| **CardinalityGuard** | Orçamento de séries ativas por tenant e por serviço; detecta explosão, quarentena o sinal infrator e emite `telemetry.cardinality.exceeded`. | SignalRegistry, MetricPipeline; → Redis |
| **PiiRedactor** | Redige/tokeniza atributos e corpos de log classificados como pessoais **antes** de qualquer persistência (RFC-0001 §7, ADR-0249). | SignalRegistry |
| **ResourceEnricher** | Injeta atributos de recurso canônicos (`service.name`, `service.version`, `deployment.environment`, `aios.tenant`, `aios.replica`, `aios.shard`) de forma uniforme. | → 027-Cluster, 028-Deployment |
| **TraceSampler** | Amostragem *head* barata no SDK + **tail-based** no coletor: retém 100% dos traces com erro e dos que excedem o SLO de latência, amostra o restante. | SloEngine |
| **MetricPipeline** | Processadores de métrica: agregação temporal, *drop* de série quarentenada, cálculo de *recording rules*, roteamento por tenant. | CardinalityGuard; → Prometheus |
| **TracePipeline** | Montagem de trace, correlação por `traceparent`, deduplicação de span e escrita no `TraceStore`. | TraceSampler; → Tempo/Jaeger |
| **LogPipeline** | Normalização de log estruturado (Serilog), correlação `trace_id`/`span_id`/`tenant_id`, roteamento e escrita no `LogStore`. | PiiRedactor; → Seq |
| **TenantRouter** | Isolamento multi-tenant do dado de telemetria: rótulo de tenant obrigatório, separação de índice/stream e negação de consulta cruzada. | PolicyClient |
| **SloEngine** | Definições de SLO versionadas; cálculo contínuo de SLI; **ledger de error budget** por janela; *burn rate* multi-janela. | SignalRegistry; → PostgreSQL, Prometheus |
| **AlertEvaluator** | Avalia regras de alerta (incluindo as derivadas de SLO), aplica `for`/histerese e produz instâncias de alerta (§4). | SloEngine, SignalRegistry |
| **AlertRouter** | Agrupa, deduplica, silencia, escalona e entrega notificações; nunca decide a mitigação. | AlertEvaluator, EventEmitter; → Alertmanager |
| **DashboardRegistry** | Dashboards como código, versionados e publicados; anota eventos de rollout e de publicação de política nas linhas do tempo. | → Grafana |
| **RunbookRegistry** | Runbooks versionados em `runbooks/`; **toda regra de alerta referencia um runbook existente** — publicação sem runbook é rejeitada. | AlertEvaluator |
| **QueryGateway** | Consulta unificada (PromQL, TraceQL, Seq) com PEP por tenant, limites de custo de consulta e *timeout*. | PolicyClient, TenantRouter |
| **RetentionManager** | Camadas de retenção, *downsampling* programado, expurgo e execução do direito ao esquecimento sobre a telemetria. | → Prometheus, Tempo, Seq, MinIO |
| **SelfTelemetry** | Meta-observabilidade: saúde, fila e **perda** do próprio pipeline, exposta por um caminho independente do pipeline (§9). | → Prometheus (scrape direto) |
| **PolicyClient** | PEP das operações administrativas e de consulta; circuit breaker, cache curto, *fail-closed* **apenas** no plano de controle (§12.1). | → 022-Policy |
| **EventEmitter** | Publica eventos CloudEvents via **Outbox transacional** em `telemetry.outbox`. | → PostgreSQL, NATS/JetStream |

### 2.2 Diagrama de Componentes (ASCII)

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
   │  │ (admite/quarentena)  │   │ (descritores,FSM)│                              │
   │  └──────────┬───────────┘   └────────┬─────────┘                              │
   │             │                        │                                        │
   │  ┌──────────▼───────────┐   ┌────────▼─────────┐   ┌────────────────────────┐│
   │  │  CardinalityGuard    │   │   PiiRedactor    │   │   ResourceEnricher     ││
   │  │ (orçamento de séries)│   │ (redige na borda)│   │ (service/tenant/env)   ││
   │  └──────────┬───────────┘   └────────┬─────────┘   └───────────┬────────────┘│
   │             └───────────┬────────────┴─────────────────────────┘             │
   │                          ▼                                                    │
   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
   │   │MetricPipeline│  │ TracePipeline│  │  LogPipeline │  │  TraceSampler   │  │
   │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │ (tail-based)    │  │
   │          │                 │                 │           └─────────────────┘  │
   │          │          ┌──────▼─────────────────▼──────┐                         │
   │          │          │        TenantRouter           │                         │
   │          │          └───────────────┬───────────────┘                         │
   │   ┌──────▼──────────┐               │            ┌──────────────────────────┐ │
   │   │   SloEngine     │               │            │   RetentionManager       │ │
   │   │ (SLI + budget)  │               │            │ (camadas + downsampling) │ │
   │   └──────┬──────────┘               │            └──────────────────────────┘ │
   │   ┌──────▼──────────┐  ┌────────────▼───────┐   ┌──────────────────────────┐ │
   │   │ AlertEvaluator  │─▶│    AlertRouter     │──▶│ DashboardRegistry        │ │
   │   │ (multi-burn)    │  │ (dedupe·silêncio·  │   │ RunbookRegistry          │ │
   │   └─────────────────┘  │  escalonamento)    │   └──────────────────────────┘ │
   │                         └────────────────────┘                                │
   │  ┌─────────────────────────────────────────────────────────────────────────┐ │
   │  │ EventEmitter (Outbox → JetStream)  ·  SelfTelemetry (caminho separado)   │ │
   │  └─────────────────────────────────────────────────────────────────────────┘ │
   └────┬───────────┬───────────────┬──────────────┬───────────────┬──────────────┘
        ▼           ▼               ▼              ▼               ▼
   PostgreSQL   Prometheus     Tempo/Jaeger      Seq         Alertmanager
   (telemetry)  (+ longo prazo)  (traces)       (logs)       (notificação)
                                                                   │
                                              NATS aios.*.telemetry.*  ──▶ 029 / 028 / 022
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN, ULID) e às convenções físicas do
> `../005-Database/Database.md` §2 (CV-01..CV-12). Schema: **`telemetry`**.
>
> **Regra absoluta deste modelo:** o PostgreSQL guarda o **plano de controle da
> observabilidade** — descritores de sinal, SLOs, regras, alertas, silêncios,
> dashboards e políticas de retenção. **Nenhuma série temporal, trace ou linha de log
> é armazenada aqui**: esses vivem no Prometheus, no Tempo/Jaeger e no Seq. Guardar
> telemetria bruta em banco relacional é o erro que transforma o observador no maior
> consumidor de I/O do sistema.

### 3.1 Entidade `SignalDescriptor` — tabela `telemetry.signal_descriptor` (plataforma)

Registro canônico de todo sinal admitido. É a entidade governada pela FSM da §4.4.

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:event:<ULID>` (tipo `event`, RFC-0001 §5.1). |
| `signal_kind` | `text` | NOT NULL, CHECK ∈ {metric, span, log_event} | Natureza do sinal. |
| `name` | `text` | NOT NULL, UNIQUE(name, descriptor_version) | `aios_<subsistema>_<nome>_<unidade>` para métrica; `<componente>.<evento>` para log. |
| `descriptor_version` | `int` | NOT NULL | Versão do descritor (imutável após `Registered`). |
| `owner_module` | `text` | NOT NULL | Módulo dono (`006-Kernel`, `022-Policy`, ...). O 024 **não** é dono de sinal de negócio. |
| `metric_type` | `text` | NULL, CHECK ∈ {counter, gauge, histogram, updown_counter} | Obrigatório quando `signal_kind = metric`. |
| `unit` | `text` | NULL | Unidade (`ms`, `bytes`, `1`, `usd`) — obrigatória para métrica. |
| `allowed_labels` | `text[]` | NOT NULL default '{}' | Lista fechada de labels permitidos. |
| `forbidden_labels` | `text[]` | NOT NULL default '{}' | Labels explicitamente proibidos (identidade). |
| `bucket_boundaries` | `numeric[]` | NULL | Buckets explícitos do histograma (fronteiras alinhadas aos SLO). |
| `max_series` | `int` | NOT NULL default 10000 | Teto de séries deste sinal por tenant. |
| `data_class` | `text` | NOT NULL default 'operational', CHECK ∈ {operational, pii, sensitive} | Classificação (dirige redação e retenção). |
| `retention_tier` | `text` | NOT NULL, CHECK ∈ {raw, short, medium, long} | Camada de retenção (§11). |
| `state` | `text` | NOT NULL, CHECK ∈ SignalState | Estado da FSM (§4.4). |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(urn)`; `UNIQUE(name, descriptor_version)`; `btree(owner_module, state)`;
`btree(state) WHERE state = 'Quarantined'`; `btree(signal_kind, retention_tier)`.

> `allowed_labels` é uma **lista fechada**, não uma sugestão. É o único ponto do
> sistema em que a explosão de cardinalidade pode ser impedida antes de acontecer —
> depois que a série existe, o custo já foi pago.

### 3.2 Entidade `CardinalityBudget` — tabela `telemetry.cardinality_budget` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento. |
| `scope` | `text` | NOT NULL, CHECK ∈ {tenant, service} | Granularidade do orçamento. |
| `scope_ref` | `text` | NOT NULL | `*` para o tenant inteiro, ou o `service.name`. |
| `max_active_series` | `bigint` | NOT NULL default 1000000 | Teto de séries ativas. |
| `current_series` | `bigint` | NOT NULL default 0 | Observado na última avaliação. |
| `soft_threshold` | `numeric(4,3)` | NOT NULL default 0.800 | Fração que dispara alerta. |
| `hard_action` | `text` | NOT NULL, CHECK ∈ {quarantine, drop_labels, reject} | Ação ao estourar. |
| `evaluated_at` | `timestamptz` | NOT NULL | Última avaliação. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, scope, scope_ref)`;
`btree(tenant_id, evaluated_at DESC)`.

### 3.3 Entidade `SloDefinition` — tabela `telemetry.slo_definition` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `slo_key` | `text` | NOT NULL, UNIQUE(tenant_id,slo_key,slo_version) | Nome estável (ex.: `kernel.syscall.latency`). |
| `slo_version` | `int` | NOT NULL | Versão (imutável após publicação). |
| `owner_module` | `text` | NOT NULL | Serviço cujo nível de serviço é medido. |
| `sli_query` | `text` | NOT NULL | Expressão do SLI (PromQL) sobre sinais **registrados**. |
| `objective` | `numeric(6,4)` | NOT NULL, CHECK (objective > 0 AND objective < 1) | Alvo (ex.: 0.9995). |
| `window` | `interval` | NOT NULL default '30 days' | Janela de conformidade. |
| `budget_policy` | `jsonb` | NOT NULL default '{}' | Ações associadas ao consumo (ex.: congelar release a 50%). |
| `status` | `text` | NOT NULL, CHECK ∈ {draft, active, deprecated} | Situação. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, slo_key, slo_version)`;
`btree(tenant_id, owner_module, status)`.

### 3.4 Entidade `ErrorBudgetEntry` — tabela `telemetry.error_budget_ledger` (multi-tenant, RLS, particionada)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | parte da PK (ULID) | Identidade do lançamento. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `slo_urn` | `text` | NOT NULL, FK→telemetry.slo_definition.urn | SLO avaliado. |
| `evaluated_at` | `timestamptz` | NOT NULL, chave de partição | Momento da avaliação. |
| `sli_value` | `numeric(9,6)` | NOT NULL | SLI observado na janela. |
| `budget_consumed` | `numeric(6,4)` | NOT NULL | Fração do budget consumida (0–1). |
| `burn_rate_1h` | `numeric(9,4)` | NOT NULL | Taxa de queima na janela curta. |
| `burn_rate_6h` | `numeric(9,4)` | NOT NULL | Taxa de queima na janela longa. |
| `breached` | `boolean` | NOT NULL default false | SLO violado nesta avaliação. |

Chave primária: `PRIMARY KEY (tenant_id, evaluated_at, id)`.
Índices: `btree(tenant_id, slo_urn, evaluated_at DESC)`; `btree(breached, evaluated_at DESC)`.

### 3.5 Entidade `AlertRule` — tabela `telemetry.alert_rule` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `rule_key` | `text` | NOT NULL, UNIQUE(tenant_id,rule_key) | Nome estável (ex.: `KernelSyscallLatencyHigh`). |
| `expr` | `text` | NOT NULL | Expressão de avaliação (PromQL). |
| `for_duration` | `interval` | NOT NULL default '5 minutes' | Histerese antes de disparar. |
| `severity` | `text` | NOT NULL, CHECK ∈ {critical, warning, info} | Severidade. |
| `slo_urn` | `text` | NULL, FK→telemetry.slo_definition.urn | Preenchido quando a regra deriva de SLO. |
| `runbook_urn` | `text` | **NOT NULL**, FK→telemetry.runbook.urn | Runbook obrigatório (ADR-0247). |
| `owner_module` | `text` | NOT NULL | Dono do alerta. |
| `notify_channels` | `text[]` | NOT NULL default '{}' | Canais de destino. |
| `status` | `text` | NOT NULL, CHECK ∈ {active, disabled} | Situação. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, rule_key)`; `btree(tenant_id, severity, status)`;
`btree(runbook_urn)`; `btree(slo_urn)`.

> `runbook_urn` é `NOT NULL` **por desenho**. Um alerta sem runbook transfere ao
> plantonista, às 3h da manhã, o trabalho de descobrir o que fazer — e o resultado
> previsível é o alerta ser silenciado em vez de tratado.

### 3.6 Entidade `AlertInstance` — tabela `telemetry.alert_instance` (multi-tenant, RLS, particionada)

Entidade governada pela FSM canônica (§4).

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | parte da PK (ULID) | Identidade da instância. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `started_at` | `timestamptz` | NOT NULL, chave de partição | Início da condição. |
| `rule_urn` | `text` | NOT NULL, FK→telemetry.alert_rule.urn | Regra que disparou. |
| `fingerprint` | `text` | NOT NULL | *Hash* do conjunto de labels — chave de deduplicação. |
| `state` | `text` | NOT NULL, CHECK ∈ AlertState | Estado da FSM (§4.1). |
| `severity` | `text` | NOT NULL | Severidade efetiva (pode escalar). |
| `labels` | `jsonb` | NOT NULL default '{}' | Labels da instância (sem identidade). |
| `value` | `numeric` | NULL | Valor observado no disparo. |
| `notified_at` | `timestamptz` | NULL | Primeira notificação entregue. |
| `acknowledged_by` | `text` | NULL | Quem reconheceu. |
| `acknowledged_at` | `timestamptz` | NULL | Quando. |
| `silenced_until` | `timestamptz` | NULL | Fim do silêncio ativo. |
| `resolved_at` | `timestamptz` | NULL | Resolução. |
| `escalation_level` | `int` | NOT NULL default 0 | Nível de escalonamento atingido. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Chave primária: `PRIMARY KEY (tenant_id, started_at, id)`.
Índices: `btree(tenant_id, state, severity)`; `UNIQUE(tenant_id, fingerprint) WHERE state IN ('Pending','Firing','Acknowledged','Silenced')`;
`btree(rule_urn, started_at DESC)`.

### 3.7 Entidade `Silence` — tabela `telemetry.silence` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `matcher` | `jsonb` | NOT NULL | Seletor de labels silenciados. |
| `reason` | `text` | NOT NULL | Justificativa — obrigatória. |
| `created_by` | `text` | NOT NULL | Autor. |
| `starts_at` | `timestamptz` | NOT NULL | Início. |
| `expires_at` | `timestamptz` | **NOT NULL** | Fim — **sempre preenchido**, limitado por `obs.alert.max_silence_h`. |
| `revoked_at` | `timestamptz` | NULL | Revogação antecipada. |

Índices: `PK(urn)`; `btree(tenant_id, expires_at)`; `btree(expires_at) WHERE revoked_at IS NULL`.

> `expires_at` é `NOT NULL` pela mesma razão que o waiver do `../022-Policy/` tem
> prazo: silêncio permanente é o alerta excluído sem que ninguém tenha decidido
> excluí-lo.

### 3.8 Entidades `Dashboard` e `Runbook` — tabelas `telemetry.dashboard`, `telemetry.runbook`

| Campo (`dashboard`) | Tipo | Chave/Constraint | Descrição |
|---------------------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `dashboard_key` | `text` | NOT NULL, UNIQUE(tenant_id,dashboard_key,dashboard_version) | Nome estável. |
| `dashboard_version` | `int` | NOT NULL | Versão imutável. |
| `definition` | `jsonb` | NOT NULL | Definição como código. |
| `owner_module` | `text` | NOT NULL | Dono. |
| `signals_used` | `text[]` | NOT NULL default '{}' | Sinais referenciados — **DEVEM** estar registrados. |
| `status` | `text` | NOT NULL, CHECK ∈ {draft, published, deprecated} | Situação. |

| Campo (`runbook`) | Tipo | Chave/Constraint | Descrição |
|-------------------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `runbook_key` | `text` | NOT NULL, UNIQUE(tenant_id,runbook_key) | Ex.: `RB-K-01`. |
| `path` | `text` | NOT NULL | Caminho versionado (`runbooks/<arquivo>.md`). |
| `steps_count` | `int` | NOT NULL, CHECK (steps_count > 0) | Passos — runbook vazio não é runbook. |
| `last_reviewed_at` | `timestamptz` | NOT NULL | Última revisão (envelhecimento é alertado). |
| `owner_module` | `text` | NOT NULL | Dono. |

Índices: `dashboard`: `UNIQUE(tenant_id, dashboard_key, dashboard_version)`, `btree(status)`.
`runbook`: `UNIQUE(tenant_id, runbook_key)`, `btree(last_reviewed_at)`.

### 3.9 Entidade `RetentionPolicy` — tabela `telemetry.retention_policy` (plataforma)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:event:<ULID>`. |
| `tier` | `text` | NOT NULL, UNIQUE, CHECK ∈ {raw, short, medium, long} | Camada. |
| `signal_kind` | `text` | NOT NULL, CHECK ∈ {metric, span, log_event} | Tipo de sinal. |
| `retain_for` | `interval` | NOT NULL | Duração da retenção. |
| `downsample_to` | `interval` | NULL | Resolução após *downsampling* (métricas). |
| `storage_target` | `text` | NOT NULL | Destino (`prometheus`, `object_store`, `tempo`, `seq`). |
| `legal_basis` | `text` | NULL | Obrigatório quando o sinal é `pii` (CV-08). |

Índices: `PK(urn)`; `UNIQUE(tier, signal_kind)`.

### 3.10 Relações (ASCII)

```
   telemetry.signal_descriptor(*) ──referenciado por──▶ telemetry.slo_definition(*)
            │                                                    │
            │ retention_tier                                     │ slo_urn
            ▼                                                    ▼
   telemetry.retention_policy(1..*)              telemetry.error_budget_ledger(*)
                                                                 │
   telemetry.alert_rule(*) ──[slo_urn]──────────────────────────┘
        │  │
        │  └──[runbook_urn NOT NULL]──▶ telemetry.runbook(1)
        ▼
   telemetry.alert_instance(*) ──silenciada por──▶ telemetry.silence(*)

   telemetry.cardinality_budget(*) ──governa──▶ admissão de signal_descriptor
   telemetry.dashboard(*) ──[signals_used]────▶ telemetry.signal_descriptor(*)

   ══════════════ FORA do PostgreSQL ══════════════
   séries temporais → Prometheus (+ armazenamento de longo prazo)
   traces           → Tempo / Jaeger
   linhas de log    → Seq
```

---

## 4. Máquina de Estados Canônica do `AlertInstance`

> A entidade com ciclo de vida governado é o **alerta** — o produto final do módulo e
> a única coisa que ele entrega a um ser humano. Séries temporais não têm ciclo de
> vida (têm retenção); SLOs mudam por versão; mas o alerta nasce, é notificado,
> reconhecido, eventualmente silenciado e resolvido — e cada transição tem
> consequência operacional real.
>
> O `SignalDescriptor` tem ciclo próprio, mais simples (§4.4).

### 4.1 Estados (`AlertState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Pending` | Condição verdadeira, mas ainda dentro de `for_duration` (histerese). | não |
| `Firing` | Condição sustentada; notificação entregue ou em entrega. | não |
| `Acknowledged` | Um operador assumiu o alerta; escalonamento suspenso. | não |
| `Silenced` | Coberto por um silêncio vigente; não notifica, **continua avaliando**. | não |
| `Resolved` | Condição deixou de ser verdadeira. | **sim** |
| `Expired` | Condição não pôde ser avaliada por falta de dado além do limite tolerado. | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Pending` | Condição da regra tornou-se verdadeira | Regra em `status = active` ∧ sinais da `expr` registrados ∧ nenhuma instância aberta com o mesmo `fingerprint`. |
| T-02 | `Pending` → `Firing` | `for_duration` decorrida com a condição contínua | Condição verdadeira em **todas** as avaliações da janela ∧ regra possui `runbook_urn` válido. |
| T-03 | `Pending` → `Resolved` | Condição deixou de ser verdadeira antes de `for_duration` | Sem notificação emitida — o alerta **nunca** alcançou um humano. |
| T-04 | `Firing` → `Acknowledged` | `AcknowledgeAlert` | Capability `obs:alert:ack` ∧ `acknowledged_by` identificado. |
| T-05 | `Firing`/`Acknowledged` → `Silenced` | Silêncio vigente casa com os labels | Silêncio com `expires_at` no futuro ∧ `reason` preenchida. |
| T-06 | `Silenced` → `Firing` | Silêncio expirou ou foi revogado ∧ condição ainda verdadeira | Reavaliação confirma a condição. |
| T-07 | `Firing`/`Acknowledged`/`Silenced` → `Resolved` | Condição deixou de ser verdadeira | Confirmada por ≥ 1 janela de avaliação (evita *flapping*). |
| T-08 | `Firing` → `Firing` (escalonamento) | `escalation_after` decorrido sem `Acknowledged` | Nível de escalonamento < máximo configurado; incrementa `escalation_level`. |
| T-09 | `Pending`/`Firing`/`Acknowledged`/`Silenced` → `Expired` | Dado ausente além de `obs.alert.no_data_timeout` | Ausência de dado sustentada; emite sinal próprio — **ausência de dado não é sucesso**. |

**Invariantes:**
(I1) Existe **no máximo uma** instância aberta (`Pending`, `Firing`, `Acknowledged`,
`Silenced`) por `(tenant_id, fingerprint)` — garantido por índice único parcial (§3.6).
Duas instâncias do mesmo alerta é ruído, e ruído leva a silêncio.
(I2) Nenhuma transição para `Firing` ocorre sem `runbook_urn` válido na regra (T-02).
(I3) `Silenced` **NÃO DEVE** interromper a avaliação: silêncio suprime a **notificação**,
nunca a **observação**. Do contrário, o silêncio esconderia a resolução tanto quanto o
problema.
(I4) `Resolved` e `Expired` são terminais: a recorrência da condição cria uma **nova**
instância, com novo `started_at` — o histórico de recorrência é o dado mais útil na
análise pós-incidente.
(I5) Toda transição para `Firing` e para `Resolved` emite evento via Outbox na mesma
transação (`telemetry.alert.firing` / `telemetry.alert.resolved`).
(I6) `Expired` **NÃO DEVE** ser tratado como `Resolved`: falta de dado é uma falha do
observador, não uma melhora do observado.
(I7) Toda transição ocorre sob OCC pelo `version` esperado; conflito → `AIOS-OBS-0006`.

### 4.3 Diagrama de estados (ASCII)

```
        condição verdadeira (T-01)
   ∅ ─────────────────────────────▶ ┌───────────┐
                                     │  Pending  │
                                     └─────┬─────┘
              condição cessa (T-03)        │ for_duration decorrida (T-02)
        ┌────────────────────────────────  ┤
        │                                  ▼
        │                            ┌───────────┐  escalonamento (T-08)
        │                            │  Firing   │──────────┐
        │                            └──┬───┬────┘◀─────────┘
        │              ack (T-04)       │   │  silêncio casa (T-05)
        │            ┌──────────────────┘   └──────────────┐
        │            ▼                                      ▼
        │     ┌──────────────┐   silêncio (T-05)     ┌────────────┐
        │     │ Acknowledged │──────────────────────▶│  Silenced  │
        │     └──────┬───────┘                        └─────┬──────┘
        │            │                    expira/revoga (T-06) │
        │            │                        ┌───────────────┘
        │            │                        ▼ (condição ainda verdadeira)
        │            │                    ┌───────────┐
        │            │                    │  Firing   │
        │            │                    └───────────┘
        │  condição cessa (T-07)  │            │
        └────────────┬────────────┴────────────┘
                     ▼
              ┌────────────┐          sem dado (T-09)     ┌───────────┐
              │  Resolved  │ (terminal)  ───────────────▶ │  Expired  │ (terminal)
              └────────────┘                               └───────────┘
```

### 4.4 Ciclo de vida do `SignalDescriptor`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `Proposed` | Sinal declarado pelo módulo dono, aguardando validação de convenção. | `Registered`, `Rejected` |
| `Registered` | Admitido: nome, tipo, unidade, labels e buckets conformes; **descritor imutável**. | `Quarantined`, `Deprecated` |
| `Quarantined` | Excedeu orçamento de cardinalidade ou violou convenção em runtime: **descartado na ingestão** até correção. | `Registered`, `Retired` |
| `Rejected` | Reprovado na validação (nome fora do padrão, label proibido, unidade ausente). | — (terminal) |
| `Deprecated` | Substituído por versão nova; ainda aceito durante a janela de migração. | `Retired` |
| `Retired` | Não aceito mais; séries expurgadas conforme retenção. | — (terminal) |

```
   Proposed ──valida──▶ Registered ──cardinalidade estourou──▶ Quarantined
      │                    │    ▲                                   │
      │ falha              │    └──── corrigido e reabilitado ──────┘
      ▼                    │ nova versão                            │
   Rejected           Deprecated ──────janela de migração──────▶ Retired
   (terminal)                                                  (terminal)
```

> `Quarantined` é reversível **de propósito**: a resposta correta a uma explosão de
> cardinalidade é descartar o sinal e avisar o dono, não perder o sinal para sempre
> nem derrubar o coletor tentando absorvê-lo.

### 4.5 Avaliação de SLO e *burn rate* (não é FSM)

O `SloEngine` avalia continuamente, sem estado durável próprio além do ledger:

```
   SLI(janela) ──▶ budget_consumed = (1 - SLI) / (1 - objective)
                          │
                          ├─ burn_rate_1h ≥ 14,4  ∧ burn_rate_6h ≥ 6  ⇒ alerta CRITICAL
                          ├─ burn_rate_6h ≥ 6     ∧ burn_rate_24h ≥ 3 ⇒ alerta WARNING
                          └─ budget_consumed ≥ política ⇒ evento slo.budget.exhausted
                                                          (consumido por 028 e 022)
```

A abordagem **multi-janela/multi-taxa** existe para separar "queima rápida" de
"degradação lenta": alertar apenas no limiar absoluto avisa tarde demais; alertar
apenas na taxa instantânea produz ruído a cada oscilação.

---

## 5. Superfície de API (ingestão, consulta e administração)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.observability.v1`. Base REST: `/v1/observability`. A **ingestão** usa o
> protocolo padrão **OTLP** (não a convenção interna), por ser o contrato que os SDKs
> de todos os módulos já falam.

### 5.1 Ingestão (OTLP — padrão externo)

| Endpoint | Protocolo | Descrição |
|----------|-----------|-----------|
| `:4317` `/opentelemetry.proto.collector.metrics.v1.MetricsService/Export` | OTLP gRPC | Métricas. |
| `:4317` `/opentelemetry.proto.collector.trace.v1.TraceService/Export` | OTLP gRPC | Traces. |
| `:4317` `/opentelemetry.proto.collector.logs.v1.LogsService/Export` | OTLP gRPC | Logs. |
| `:4318` `POST /v1/{metrics,traces,logs}` | OTLP HTTP | Equivalente HTTP. |

Regras normativas da ingestão:

- O emissor **DEVE** autenticar-se por mTLS com certificado de workload (`021`).
- O `tenant` **DEVE** vir do atributo de recurso `aios.tenant`; divergente da
  identidade do certificado ⇒ rejeição (`AIOS-OBS-0003`).
- A resposta **NÃO DEVE** aplicar backpressure ao emissor: sob saturação, o gateway
  responde sucesso e **descarta** com contabilização (`aios_obs_dropped_total`) —
  ADR-0243.
- Sinal não registrado **NÃO DEVE** derrubar o lote: apenas aquele sinal é
  quarentenado (`AIOS-OBS-0002` reportado por evento, não por erro de transporte).

### 5.2 Consulta

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Consultar métricas (PromQL) | `POST /v1/observability/query/metrics` | `QueryMetrics` | sim (leitura) | `obs:query:metrics` |
| Consultar traces | `POST /v1/observability/query/traces` | `QueryTraces` | sim (leitura) | `obs:query:traces` |
| Obter trace por id | `GET /v1/observability/traces/{traceId}` | `GetTrace` | sim (leitura) | `obs:query:traces` |
| Consultar logs | `POST /v1/observability/query/logs` | `QueryLogs` | sim (leitura) | `obs:query:logs` |
| Estado de SLO e budget | `GET /v1/observability/slos/{key}` | `GetSlo` | sim (leitura) | `obs:slo:read` |

### 5.3 Administração

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Registrar sinal | `POST /v1/observability/signals` | `RegisterSignal` | sim (key) | `obs:signal:register` |
| Listar/ler sinal | `GET /v1/observability/signals[/{name}]` | `GetSignal`/`ListSignals` | sim (leitura) | `obs:signal:read` |
| Quarentenar/reabilitar sinal | `POST /v1/observability/signals/{name}:{quarantine\|reinstate}` | `SetSignalState` | sim | `obs:signal:manage` |
| Definir orçamento de cardinalidade | `PUT /v1/observability/cardinality-budgets/{scope}` | `PutCardinalityBudget` | sim | `obs:cardinality:manage` |
| Publicar SLO | `POST /v1/observability/slos` | `PutSlo` | sim (key) | `obs:slo:write` |
| Publicar regra de alerta | `PUT /v1/observability/alert-rules/{key}` | `PutAlertRule` | sim | `obs:alert:write` |
| Reconhecer alerta | `POST /v1/observability/alerts/{id}:ack` | `AcknowledgeAlert` | sim | `obs:alert:ack` |
| Criar silêncio | `POST /v1/observability/silences` | `CreateSilence` | sim (key) | `obs:silence:write` |
| Revogar silêncio | `POST /v1/observability/silences/{urn}:revoke` | `RevokeSilence` | sim | `obs:silence:write` |
| Publicar dashboard | `POST /v1/observability/dashboards` | `PutDashboard` | sim | `obs:dashboard:write` |
| Publicar runbook | `PUT /v1/observability/runbooks/{key}` | `PutRunbook` | sim | `obs:runbook:write` |
| Definir retenção | `PUT /v1/observability/retention/{tier}` | `PutRetentionPolicy` | sim | `obs:retention:manage` |

### 5.4 Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínio reservado: **`OBS`** (0001–0099), registrado no
> catálogo mantido por `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0240**).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-OBS-0001` | 404 | não | Sinal, SLO, regra, alerta, silêncio, dashboard ou runbook não encontrado. |
| `AIOS-OBS-0002` | 422 | não | Sinal não registrado ou em quarentena (rejeitado na admissão). |
| `AIOS-OBS-0003` | 403 | não | Tenant divergente da identidade do emissor/consultante. |
| `AIOS-OBS-0004` | 422 | não | Nome, tipo, unidade ou label fora da convenção (RFC-0240). |
| `AIOS-OBS-0005` | 429 | sim | Orçamento de cardinalidade excedido para o escopo. |
| `AIOS-OBS-0006` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-OBS-0007` | 409 | não | Transição de estado inválida (alerta ou sinal). |
| `AIOS-OBS-0008` | 422 | não | Regra de alerta sem runbook válido (`runbook_urn` obrigatório). |
| `AIOS-OBS-0009` | 422 | não | SLO referencia sinal não registrado ou objetivo fora de (0,1). |
| `AIOS-OBS-0010` | 429 | sim | Limite de custo/tempo de consulta excedido. |
| `AIOS-OBS-0011` | 503 | sim | Armazenamento de telemetria indisponível (consulta). |
| `AIOS-OBS-0012` | 413 | não | Lote OTLP acima do tamanho máximo aceito. |
| `AIOS-OBS-0013` | 403 | não | Silêncio sem prazo, sem justificativa ou acima do máximo permitido. |
| `AIOS-OBS-0014` | 409 | não | Descritor de sinal imutável: registre uma nova versão. |
| `AIOS-OBS-0015` | 422 | não | Retenção solicitada incompatível com a classificação do dado (`pii` sem base legal). |

> **Nota deliberada sobre a ingestão.** Os erros acima valem para as superfícies de
> **consulta e administração**. Na **ingestão**, o módulo é fail-open: problemas de
> admissão viram **contador e evento**, não erro de transporte devolvido ao emissor
> (ADR-0243). Um `429` na ingestão faria o SDK do serviço observado gastar CPU em
> retry justamente quando o sistema está sob pressão.

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3. `<dominio>` =
> **`telemetry`**, valor a ser registrado no registro de domínios mantido por
> `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0240**). Entrega
> at-least-once; dedupe por `event.id`.
>
> **Regra absoluta:** nenhum evento deste módulo transporta amostra de telemetria,
> valor de atributo classificado `pii`/`sensitive`, nem conteúdo de log. Eventos
> carregam identificadores, nomes de sinal, severidade, números agregados e horários.

### 6.1 Eventos emitidos (produtor: 024-Observability)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios.<tenant>.telemetry.alert.firing` | `aios.telemetry.alert.firing` | Alerta → `Firing` (T-02). | `TELEMETRY_ALERT` |
| `aios.<tenant>.telemetry.alert.acknowledged` | `aios.telemetry.alert.acknowledged` | Alerta → `Acknowledged` (T-04). | `TELEMETRY_ALERT` |
| `aios.<tenant>.telemetry.alert.resolved` | `aios.telemetry.alert.resolved` | Alerta → `Resolved` (T-07). | `TELEMETRY_ALERT` |
| `aios.<tenant>.telemetry.alert.escalated` | `aios.telemetry.alert.escalated` | Escalonamento por falta de reconhecimento (T-08). | `TELEMETRY_ALERT` |
| `aios.<tenant>.telemetry.alert.nodata` | `aios.telemetry.alert.nodata` | Alerta → `Expired` por ausência de dado (T-09). | `TELEMETRY_ALERT` |
| `aios.<tenant>.telemetry.slo.breached` | `aios.telemetry.slo.breached` | SLO violado na janela de conformidade. | `TELEMETRY_SLO` |
| `aios.<tenant>.telemetry.budget.exhausted` | `aios.telemetry.budget.exhausted` | Error budget consumido acima do limiar da política. | `TELEMETRY_SLO` |
| `aios.<tenant>.telemetry.cardinality.exceeded` | `aios.telemetry.cardinality.exceeded` | Orçamento de séries estourado; sinal quarentenado. | `TELEMETRY_PLATFORM` |
| `aios._platform.telemetry.signal.registered` | `aios.telemetry.signal.registered` | Novo `SignalDescriptor` admitido. | `TELEMETRY_PLATFORM` |
| `aios._platform.telemetry.signal.quarantined` | `aios.telemetry.signal.quarantined` | Sinal quarentenado (cardinalidade ou convenção). | `TELEMETRY_PLATFORM` |
| `aios.<tenant>.telemetry.ingest.degraded` | `aios.telemetry.ingest.degraded` | Descarte de telemetria acima do limiar tolerado. | `TELEMETRY_PLATFORM` |
| `aios._platform.telemetry.dashboard.published` | `aios.telemetry.dashboard.published` | Dashboard publicado/versionado. | `TELEMETRY_PLATFORM` |

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do módulo |
|-------------------|----------|----------------|
| `aios.<tenant>.policy.bundle.updated` | `022-Policy` | Invalida cache do `PolicyClient`; anota a mudança nas linhas do tempo dos dashboards. |
| `aios.<tenant>.policy.decision.updated` | `022-Policy` | Invalidação granular de decisão de consulta. |
| `aios._platform.deployment.rollout.started` | `028-Deployment` | Anota o rollout nos dashboards e ativa janela de supressão de alertas esperados. |
| `aios._platform.cluster.node.decommissioned` | `027-Cluster` | Remove o alvo de coleta; evita alerta `nodata` espúrio. |
| `aios.<tenant>.security.rtbf.requested` | `021-Security` | Dispara expurgo/pseudonimização da telemetria do titular (§12.3). |
| `aios._platform.security.jwks.rotated` | `021-Security` | Recarrega material de verificação usado no `QueryGateway`. |
| `aios.<tenant>.cost.budget.exhausted` | `026-Cost-Optimizer` | Correlaciona custo e carga; anota o evento nos dashboards. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Ingerir métricas, traces e logs por OTLP (gRPC e HTTP) sob mTLS. | Must | Lote válido de cada tipo aceito e visível na consulta em ≤ 10 s. |
| FR-002 | Nunca aplicar backpressure ao emissor: sob saturação, descartar e contabilizar. | Must | Teste de saturação: latência do emissor inalterada; `aios_obs_dropped_total` reflete o descarte. |
| FR-003 | Admitir apenas sinais registrados; quarentenar o desconhecido sem derrubar o lote. | Must | Sinal não registrado → quarentena + `signal.quarantined`; demais sinais do lote persistem. |
| FR-004 | Validar convenção de nome, tipo, unidade e labels no registro. | Must | Nome fora de `aios_<subsistema>_<nome>_<unidade>` → `AIOS-OBS-0004`. |
| FR-005 | Impor orçamento de cardinalidade por tenant e por serviço. | Must | Estouro → ação configurada (`quarantine`/`drop_labels`/`reject`) + `cardinality.exceeded`. |
| FR-006 | Rejeitar label de identidade (`*_urn`, `*_id` de entidade) em métrica. | Must | `agent_urn` como label → `AIOS-OBS-0004` no registro; descartado em runtime. |
| FR-007 | Redigir PII na borda, antes de qualquer persistência. | Must | Varredura de armazenamento não encontra PII em sinal `operational`. |
| FR-008 | Amostrar traces com política *tail-based*, retendo 100% dos erros e das caudas. | Must | Trace com `status=ERROR` ou acima do SLO sempre presente; demais na taxa configurada. |
| FR-009 | Enriquecer todo sinal com atributos de recurso canônicos. | Must | 100% dos sinais com `service.name`, `aios.tenant`, `deployment.environment`. |
| FR-010 | Calcular SLI e manter o ledger de error budget por SLO e janela. | Must | Ledger reproduz o SLI a partir dos sinais registrados; divergência = defeito. |
| FR-011 | Avaliar alertas com *multi-window multi-burn-rate* e histerese. | Must | Queima rápida dispara em ≤ 2 min; oscilação abaixo do limiar não dispara. |
| FR-012 | Gerir o ciclo de vida do alerta (§4) com deduplicação por `fingerprint`. | Must | Nunca mais de uma instância aberta por `fingerprint` (invariante I1). |
| FR-013 | Exigir runbook válido para toda regra de alerta. | Must | Publicação sem `runbook_urn` → `AIOS-OBS-0008`. |
| FR-014 | Suportar silêncios com prazo, justificativa e autor obrigatórios. | Must | Silêncio sem `expires_at` ou sem `reason` → `AIOS-OBS-0013`. |
| FR-015 | Continuar avaliando alerta silenciado (silêncio suprime notificação, não observação). | Must | Alerta silenciado que resolve registra `Resolved` normalmente (invariante I3). |
| FR-016 | Tratar ausência de dado como `Expired`, nunca como `Resolved`. | Must | Alvo removido sem `node.decommissioned` → `alert.nodata` (invariante I6). |
| FR-017 | Servir consulta unificada de métricas, traces e logs com isolamento por tenant. | Must | Consulta cross-tenant → `AIOS-OBS-0003`. |
| FR-018 | Atuar como PEP nas consultas e nas operações administrativas. | Must | Sem capability → negado pelo PDP; decisão auditada em `../025-Audit/`. |
| FR-019 | Aplicar retenção em camadas com *downsampling* programado. | Must | Métrica raw expurgada em 15 d; agregada de 5 min disponível por 90 d. |
| FR-020 | Publicar dashboards e runbooks como código versionado. | Should | Dashboard referenciando sinal não registrado é rejeitado. |
| FR-021 | Expor a saúde e a perda do próprio pipeline por caminho independente. | Must | Com o pipeline saturado, `SelfTelemetry` continua respondendo. |
| FR-022 | Emitir todos os eventos de §6 via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. |
| FR-023 | Suportar o direito ao esquecimento sobre a telemetria. | Must | `security.rtbf.requested` → expurgo/pseudonimização comprovada e correlacionada em `../025-Audit/`. |
| FR-024 | Anotar rollouts e publicações de política nas linhas do tempo dos dashboards. | Should | Evento de rollout visível no dashboard em ≤ 30 s. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (ingestão) | p99 de aceitação de lote OTLP ≤ **50 ms**. | `aios_obs_ingest_latency_ms`. |
| NFR-002 | Overhead no observado | ≤ **3%** de CPU e ≤ **1 ms** adicionados ao p99 do serviço instrumentado. | Benchmark comparativo com e sem instrumentação. |
| NFR-003 | Throughput | ≥ **2.000.000** *datapoints*/s e ≥ **200.000** spans/s por réplica de coletor. | Benchmark `Benchmark.md`. |
| NFR-004 | Disponibilidade (ingestão) | ≥ **99,9%**. | Uptime probe; *error budget* mensal. |
| NFR-005 | Disponibilidade (alerta) | ≥ **99,95%** — o caminho de alerta é mais crítico que o de consulta. | Probe sintético de alerta ponta a ponta. |
| NFR-006 | Perda de telemetria | ≤ **0,1%** em operação normal; **100%** do descarte contabilizado sob saturação. | `aios_obs_dropped_total` / `aios_obs_received_total`. |
| NFR-007 | Frescor | Sinal visível na consulta em ≤ **10 s** (p99) após a emissão. | Probe sintético fim a fim. |
| NFR-008 | Tempo de detecção | Alerta de queima rápida dispara em ≤ **2 min** do início da degradação. | Injeção controlada de falha. |
| NFR-009 | Cardinalidade | ≤ **10⁶** séries ativas por tenant; **0** labels de identidade em produção. | `aios_obs_active_series`; varredura do registro. |
| NFR-010 | Escalabilidade | ≥ **10⁶** agentes e ≥ **500** serviços emissores sem crescimento super-linear de custo. | Teste de escala `Scalability.md`. |
| NFR-011 | Recuperação | **RTO ≤ 15 min** (plano de controle), **RPO ≤ 5 min** para SLOs/regras/alertas; para telemetria bruta, perda máxima de **1 min** (buffer em disco do coletor). | DR drill. |
| NFR-012 | Retenção | Métricas: raw 15 d, 5 min/90 d, 1 h/400 d. Traces: 7 d. Logs: 14 d. **100%** de aderência. | Auditoria de expurgo. |
| NFR-013 | Latência de consulta | p95 de consulta de painel ≤ **2 s**; consulta acima do orçamento é abortada. | `aios_obs_query_latency_ms`. |
| NFR-014 | Privacidade | **0** ocorrências de PII em sinal classificado `operational`. | Varredura automatizada contínua (FR-007). |
| NFR-015 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key`. |
| NFR-016 | Cobertura de runbook | **100%** das regras de alerta ativas com runbook válido e revisado nos últimos **180 dias**. | Consulta ao registro; alerta de runbook envelhecido. |
| NFR-017 | Isolamento de tenant | **0** consultas que atravessem a fronteira de tenant. | RLS + validação estrutural; `AIOS-OBS-0003` monitorado. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global`, `tenant`, `service`, `signal`. Recarregável indica *hot reload*.
> Prefixo de env var: **`AIOS_OBS_`**.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `obs.ingest.max_batch_mb` | 4 | 1–32 | global | sim | Tamanho máximo do lote OTLP (`AIOS-OBS-0012`). |
| `obs.ingest.queue_size` | 100000 | 1000–1000000 | global | sim | Fila em memória antes do descarte. |
| `obs.ingest.disk_buffer_mb` | 2048 | 0–65536 | global | **não** | Buffer em disco do coletor (sustenta o RPO de 1 min do NFR-011). |
| `obs.ingest.drop_alert_ratio` | 0.01 | 0.0–1.0 | global | sim | Fração de descarte que dispara `ingest.degraded`. |
| `obs.ingest.fail_mode` | `open` | {open,closed} | global | **não** | **`open` é o desenho** (ADR-0243); `closed` só para diagnóstico em laboratório. |
| `obs.signal.registration_required` | `true` | {true,false} | global | **não** | Exige registro prévio do sinal (ADR-0241). |
| `obs.signal.unknown_action` | `quarantine` | {quarantine,drop} | global | sim | Ação para sinal não registrado. |
| `obs.cardinality.max_series_per_tenant` | 1000000 | 1000–50000000 | tenant | sim | Teto de séries ativas (NFR-009). |
| `obs.cardinality.max_series_per_signal` | 10000 | 10–1000000 | signal | sim | Teto por sinal. |
| `obs.cardinality.soft_threshold` | 0.8 | 0.1–1.0 | tenant | sim | Fração que dispara alerta antes do teto. |
| `obs.trace.head_sample_ratio` | 0.1 | 0.0–1.0 | tenant | sim | Amostragem barata no SDK. |
| `obs.trace.tail_keep_errors` | `true` | {true,false} | global | **não** | Retém 100% dos traces com erro (ADR-0244). |
| `obs.trace.tail_keep_slow_ms` | 1000 | 1–60000 | tenant | sim | Retém traces acima deste limiar. |
| `obs.pii.redaction_enabled` | `true` | {true,false} | global | **não** | Redação na borda (ADR-0249). |
| `obs.pii.redaction_mode` | `tokenize` | {tokenize,drop} | tenant | sim | Tokenizar ou remover o atributo pessoal. |
| `obs.retention.metrics_raw_days` | 15 | 1–90 | global | sim | Retenção de métrica bruta. |
| `obs.retention.metrics_5m_days` | 90 | 7–400 | global | sim | Retenção da agregação de 5 min. |
| `obs.retention.metrics_1h_days` | 400 | 30–1825 | global | sim | Retenção da agregação de 1 h. |
| `obs.retention.traces_days` | 7 | 1–90 | global | sim | Retenção de traces amostrados. |
| `obs.retention.logs_days` | 14 | 1–365 | tenant | sim | Retenção de logs. |
| `obs.slo.evaluation_interval_s` | 60 | 10–3600 | global | sim | Intervalo de avaliação de SLO. |
| `obs.slo.default_window_days` | 30 | 1–90 | tenant | sim | Janela de conformidade default. |
| `obs.alert.default_for_s` | 300 | 0–86400 | tenant | sim | Histerese default (`for_duration`). |
| `obs.alert.no_data_timeout_s` | 600 | 60–86400 | tenant | sim | Ausência de dado até `Expired` (T-09). |
| `obs.alert.escalation_after_s` | 900 | 60–86400 | tenant | sim | Escalonamento sem reconhecimento (T-08). |
| `obs.alert.max_silence_h` | 72 | 1–720 | tenant | sim | Prazo máximo de silêncio (`AIOS-OBS-0013`). |
| `obs.alert.runbook_required` | `true` | {true,false} | global | **não** | Exige runbook por regra (ADR-0247). |
| `obs.query.max_duration_s` | 30 | 1–300 | tenant | sim | Orçamento de tempo por consulta (`AIOS-OBS-0010`). |
| `obs.query.max_series_scanned` | 5000000 | 1000–100000000 | tenant | sim | Orçamento de varredura por consulta. |
| `obs.policy.fail_mode` | `closed` | {closed,open} | global | sim | PEP de **consulta/administração** — fail-closed, ao contrário da ingestão. |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| Saturação do pipeline de ingestão | Fila acima do limiar; `dropped_total` | **Descarte contabilizado**; emissor não é bloqueado (ADR-0243) | Escala horizontal do coletor; drena buffer de disco | Reenvio pelo SDK é opcional e limitado |
| Explosão de cardinalidade | `CardinalityGuard` | Quarentena **do sinal infrator**, não do tenant inteiro | Dono corrige o descritor; reabilita (`Quarantined → Registered`) | Determinístico |
| Prometheus indisponível | Health/scrape | Métricas represadas no buffer; traces e logs seguem | Failover; drenagem do buffer | Escrita idempotente por timestamp |
| Tempo/Jaeger indisponível | Health | Traces descartados após buffer; métricas e logs seguem | Failover; retomada | Dedupe por `trace_id`+`span_id` |
| Seq indisponível | Health | Logs represados; métricas e traces seguem | Failover; drenagem | Dedupe por `event.id` |
| PostgreSQL (`telemetry`) indisponível | Health | **Ingestão continua** com o registro em cache; administração e SLO param | Failover do `005`; reconciliação | Idempotente |
| Alertmanager indisponível | Health/probe | Alertas avaliados e persistidos, **notificação atrasada** | Retomada; entrega dos pendentes | Dedupe por `fingerprint` |
| Perda de dado sem alerta (silêncio permanente) | Varredura de `expires_at`; alerta de silêncio longo | — | Expiração automática; revisão semanal | Determinístico |
| Alerta *flapping* | Contagem de transições por hora | Histerese (`for_duration`) e confirmação na resolução | Ajuste da regra; revisão do limiar | — |
| Ausência de dado interpretada como saúde | `Expired` (T-09) e `alert.nodata` | Alerta próprio de ausência | Investigar alvo de coleta antes de assumir normalidade | — |
| Consulta abusiva | `max_series_scanned`, `max_duration_s` | Aborto da consulta (`AIOS-OBS-0010`), não do serviço | Ajuste do painel; orçamento por tenant | Retriable |
| PDP (022) indisponível | Timeout/CB no `PolicyClient` | **Consulta e administração** negadas (`fail_mode=closed`); **ingestão continua** | Retry após meia-abertura | Retriable |
| Falha do próprio observador | `SelfTelemetry` por caminho independente | Meta-monitoramento não depende do pipeline observado | Redeploy do coletor | — |

**Metas de recuperação:** **RTO ≤ 15 min** e **RPO ≤ 5 min** para o **plano de
controle** (SLOs, regras, alertas, silêncios, dashboards). Para **telemetria bruta**, o
RPO é de até **1 minuto**, correspondente ao buffer em disco do coletor — e essa
diferença é deliberada: reter todo dado bruto com RPO de 5 min exigiria um pipeline
mais caro e mais frágil do que o sistema que ele observa.

Degradação graciosa em ordem: sacrifica-se primeiro **traces amostrados**, depois
**logs de nível baixo**, depois **métricas de baixa prioridade**, preservando por
último **métricas de SLO e o caminho de alerta** — porque um sistema sem traces é um
sistema difícil de depurar; um sistema sem alerta é um sistema em que ninguém sabe que
há o que depurar.

> **Quem observa o observador.** O `SelfTelemetry` expõe saúde, fila e descarte por um
> endpoint *scrapeado* diretamente pelo Prometheus, **sem passar pelo pipeline de
> ingestão**. Se a meta-observabilidade dependesse do pipeline, a única falha que ela
> não conseguiria reportar seria exatamente a falha do pipeline.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Coletores | *Stateless* e horizontais, atrás de balanceador; escala por CPU e por profundidade de fila. |
| Agente vs. gateway | Coleta em duas camadas: agente local por nó (baixa latência, buffer em disco) → gateway central (agregação, *tail sampling*, roteamento). |
| *Tail-based sampling* | Exige que todos os spans de um trace cheguem ao mesmo coletor: roteamento consistente por `hash(trace_id)`. |
| Cardinalidade | O limite real não é bytes: é **séries ativas**. Orçamento por tenant e por sinal, com quarentena automática (ADR-0242). |
| Métricas de longo prazo | *Downsampling* em camadas (raw → 5 min → 1 h) com objeto em MinIO; consultas longas leem a camada agregada. |
| Consulta | Orçamento de tempo e de séries varridas por consulta; painéis pesados usam *recording rules* pré-calculadas em vez de agregação ao vivo. |
| Isolamento multi-tenant | Rótulo de tenant obrigatório e roteamento por `TenantRouter`; tenant ruidoso não degrada os demais. |
| Concorrência do registro | OCC por `version` no `SignalDescriptor`; descritor imutável após `Registered` elimina disputa. |
| Alertas | Avaliação particionada por regra; deduplicação por `fingerprint` evita tempestade de notificação. |
| Rumo a milhões | 10⁶ agentes ⇒ dezenas de milhões de *datapoints*/s se cada agente virar série. Sustentado por: **agentes nunca são label** (métricas agregadas por tenant/serviço/shard), *tail sampling* de traces, logs amostrados por nível, e o registro de sinais como ponto único de contenção. |

```
   ❌ 1 agente = 1 série                 ✅ agente vive em trace/log, não em label
      10⁶ agentes ⇒ 10⁶+ séries             métricas agregam por tenant/serviço/shard
      (o observador custa mais que o         cardinalidade cresce com a TOPOLOGIA,
       observado)                            não com a POPULAÇÃO
```

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada: `ADR-0240`..`ADR-0249`** (regra 024×10). Registrar em
> `../002-ADR/`. **ADR-0010** (observabilidade e auditoria por construção) é herdada e
> é a decisão-mãe deste módulo.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0240 | Domínio de erro `OBS` e registro do domínio de eventos `telemetry`. |
| ADR-0241 | Registro obrigatório de sinal: telemetria como contrato declarado, não emergente. |
| ADR-0242 | Orçamento de cardinalidade por tenant e proibição de labels de identidade. |
| ADR-0243 | Fail-open da telemetria: descarte contabilizado em vez de backpressure no observado. |
| ADR-0244 | *Tail-based sampling* com retenção obrigatória de erros e caudas de latência. |
| ADR-0245 | SLO como dado versionado e error budget como sinal de primeira classe. |
| ADR-0246 | Fronteira telemetria × auditoria (024 × 025): amostrável vs. imutável. |
| ADR-0247 | Dashboards, regras de alerta e runbooks como código versionado; alerta sem runbook é inválido. |
| ADR-0248 | Retenção em camadas com *downsampling* em vez de retenção única. |
| ADR-0249 | Redação de PII na borda de ingestão, não no armazenamento. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida) — §5.6 fixa a correlação W3C; §5.8 exige telemetria OTel em todo serviço; §7 fixa minimização. |
| RFC-0240 | AIOS Telemetry Signal & Semantic Conventions (nomes, tipos, unidades, labels permitidos, atributos de recurso, buckets). | A propor por este módulo. |
| RFC-0241 | AIOS SLO & Alerting Contract (definição de SLI/SLO, ledger de error budget, *multi-burn-rate*, ciclo de vida e roteamento de alerta). | A propor por este módulo. |

---

## 12. Decisões de Segurança

> A telemetria é um alvo atraente por um motivo simples: ela **atravessa todos os
> módulos**. Um atacante com acesso irrestrito à consulta de logs e traces obtém um
> retrato do sistema inteiro sem precisar comprometer nenhum serviço específico.

### 12.1 AuthN / AuthZ

- **AuthN de emissores**: **mTLS** com certificado de workload emitido pelo
  `021-Security` após atestação (RFC-0001 §6). O atributo `aios.tenant` do recurso
  **DEVE** ser coerente com a identidade do certificado (`AIOS-OBS-0003`).
- **AuthN de humanos**: OIDC validado no Gateway (`004-API`), com MFA exigido pelo
  `021`.
- **AuthZ**: o 024 é **PEP**. Consulta e administração exigem capability avaliada pelo
  **PDP** do `022-Policy`, com ***default deny***.
- **Assimetria deliberada de `fail_mode`**: o plano de **ingestão** é ***fail-open***
  (ADR-0243) — perder telemetria é preferível a derrubar o sistema observado; o plano
  de **consulta e administração** é ***fail-closed*** (`obs.policy.fail_mode=closed`) —
  ninguém lê a telemetria de ninguém quando não é possível autorizar.
- **Isolamento de tenant**: RLS nas tabelas de controle e rótulo de tenant obrigatório
  nos dados de série; consulta cruzada é negada estruturalmente (NFR-017).

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no 024-Observability | Mitigação |
|-----------------|-----------------------------|-----------|
| **S**poofing | Emissor falso injeta telemetria de outro serviço/tenant, poluindo SLOs e mascarando um incidente real. | mTLS com certificado atestado (`021`); `aios.tenant` verificado contra a identidade do certificado (`AIOS-OBS-0003`); sinal só é aceito do `owner_module` declarado no descritor. |
| **T**ampering | Alteração de regra de alerta ou de SLO para esconder degradação; silêncio permanente sobre um alerta crítico. | Toda mutação sob PEP+OCC e auditada em `../025-Audit/`; silêncio com `expires_at` obrigatório e teto de 72 h; alteração de SLO cria **nova versão**, preservando o histórico. |
| **R**epudiation | Negar ter silenciado, reconhecido ou alterado uma regra durante um incidente. | `created_by`, `acknowledged_by`, `reason` obrigatórios; eventos `alert.*` na trilha imutável do `025`. |
| **I**nformation disclosure | Log ou span carregando prompt, resposta de LLM, item de memória ou PII; consulta cruzada entre tenants; inferência de comportamento alheio por métricas. | `PiiRedactor` na **borda** (ADR-0249); `data_class` por sinal; proibição de conteúdo de negócio em log (NR-11); RLS + `TenantRouter`; labels de identidade proibidos. |
| **D**enial of service | Inundação de telemetria de um tenant; explosão de cardinalidade; consulta que varre o armazenamento inteiro. | Limite de ingestão por tenant; orçamento de cardinalidade com quarentena **do sinal**, não do tenant; orçamento de consulta (`AIOS-OBS-0010`); fail-open garante que o alvo do DoS seja o observador, nunca o observado. |
| **E**levation of privilege | Uso da consulta de logs como porta lateral para dados que a API de negócio não exporia; alteração de retenção para reter dado que deveria ser expurgado. | Consulta sob PEP com capability por tipo de sinal; retenção sob `obs:retention:manage` e validada contra `data_class` (`AIOS-OBS-0015`); nenhum caminho de consulta ignora o `TenantRouter`. |

### 12.3 LGPD / GDPR

- **Minimização**: telemetria é dado **operacional**. Conteúdo de negócio (prompt,
  resposta, item de memória) **NÃO DEVE** entrar em log, span ou métrica (NR-11); o
  que se registra é o identificador e a medida.
- **Redação na borda**: atributos classificados `pii` são tokenizados ou removidos
  **antes** de qualquer persistência (ADR-0249) — redigir no armazenamento significaria
  que o dado já esteve em claro em disco.
- **Base legal e retenção**: cada camada de retenção declara `legal_basis` quando
  aplicável (CV-08); telemetria `operational` tem retenção curta por desenho (§NFR-012).
- **Direito ao esquecimento**: ao consumir `security.rtbf.requested`, o módulo
  pseudonimiza ou expurga a telemetria associada ao titular nas três lojas (métricas,
  traces, logs), preservando agregados sem identificação, e emite comprovante
  correlacionado em `../025-Audit/`.
- **Segregação**: rótulo de tenant obrigatório; consulta cruzada negada; retenção
  configurável por tenant para atender exigências contratuais distintas.
- **Transferência internacional**: o armazenamento de telemetria segue a região do
  tenant definida em `../027-Cluster/`; o módulo não replica sinal entre regiões sem
  política explícita.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md` (§4 stack de observabilidade)
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§5.6 correlação,
  §5.8 telemetria obrigatória, §7 privacidade)
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../004-API/`, `../005-Database/`, `../006-Kernel/`,
  `../007-Agent-Runtime/`, `../020-Communication/`, `../021-Security/`,
  `../022-Policy/`, `../025-Audit/`, `../026-Cost-Optimizer/`, `../027-Cluster/`,
  `../028-Deployment/`, `../029-Operations/`.

*Fim do Design Brief interno do módulo 024-Observability. Este documento governa a
geração dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e
este brief é um defeito do documento gerado.*
