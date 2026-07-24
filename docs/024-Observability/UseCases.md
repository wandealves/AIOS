---
Documento: UseCases
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0243, ADR-0244, ADR-0245, ADR-0247, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: FunctionalRequirements.md, StateMachine.md, API.md, Events.md
---

# 024-Observability — Casos de Uso

> Formato: ator · pré-condições · fluxo principal · fluxos alternativos · exceções ·
> pós-condições. Os IDs `UC-NNN` são referenciados por `./Requirements.md` §5 e §6 e
> por `./Testing.md`.

## UC-001 — Ingerir telemetria por OTLP

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Qualquer módulo emissor (SDK OTel via `otel-agent` local) |
| **Atores secundários** | `../021-Security/` (certificado de workload), `../027-Cluster/` (atributos de nó) |
| **Pré-condições** | Emissor com certificado mTLS válido; `aios.tenant` presente nos atributos de recurso |
| **Requisitos** | FR-001, FR-002, FR-009, FR-021; NFR-001..NFR-003, NFR-006 |

**Fluxo principal**

1. O SDK do emissor exporta um lote OTLP para o `otel-agent` do nó (latência local).
2. O agente aplica *head sampling* e persiste em **buffer de disco** antes de
   encaminhar ao `otel-gateway`.
3. O `TelemetryIngestGateway` valida o envelope, autentica por mTLS e confere que
   `aios.tenant` é coerente com a identidade do certificado.
4. O `SignalAdmission` consulta o `SignalRegistry` (cache local) para cada sinal do
   lote.
5. O `CardinalityGuard` contabiliza séries novas contra o orçamento do tenant.
6. O `PiiRedactor` redige atributos classificados como pessoais.
7. O `ResourceEnricher` injeta `service.name`, `service.version`,
   `deployment.environment`, `aios.tenant`, `aios.replica`, `aios.shard`.
8. O lote é **enfileirado** para os pipelines e o gateway responde sucesso — o ACK
   ocorre **após o enfileiramento**, nunca após a escrita no armazenamento.
9. `MetricPipeline`, `TracePipeline` e `LogPipeline` escrevem em Prometheus,
   Tempo/Jaeger e Seq; o sinal fica visível na consulta em ≤ 10 s (NFR-007).

**Fluxos alternativos**

- **A1 — Saturação:** fila acima de `obs.ingest.queue_size` ⇒ o gateway **descarta** e
  incrementa `aios_obs_dropped_total`, **sem** aplicar backpressure (ADR-0243). Acima
  de `obs.ingest.drop_alert_ratio`, emite `telemetry.ingest.degraded`.
- **A2 — Sinal desconhecido no lote:** apenas aquele sinal é quarentenado (UC-003); os
  demais persistem.
- **A3 — Gateway indisponível:** o `otel-agent` retém no buffer de disco até
  `obs.ingest.disk_buffer_mb`; a perda máxima é de 1 min (NFR-011).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | `aios.tenant` divergente do certificado | Lote rejeitado; `AIOS-OBS-0003`; evento de segurança para `../021-Security/` |
| E2 | Lote acima de `obs.ingest.max_batch_mb` | `AIOS-OBS-0012` |
| E3 | Armazenamento de destino indisponível | Lote permanece na fila; descarte conforme A1; alerta interno |
| E4 | Pipeline saturado **e** `SelfTelemetry` também inacessível | Incidente P1: o observador está cego sobre si mesmo (`./FailureRecovery.md` §6) |

**Pós-condições**

- Telemetria visível na consulta; nenhum emissor foi bloqueado; todo descarte está
  contabilizado.

---

## UC-002 — Registrar um novo sinal

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Módulo dono do sinal (pipeline de CI do módulo) |
| **Pré-condições** | Capability `obs:signal:register`; nome conforme RFC-0240 |
| **Requisitos** | FR-003, FR-004, FR-006; NFR-015 |

**Fluxo principal**

1. O módulo dono submete `RegisterSignal` com `name`, `signal_kind`, `metric_type`,
   `unit`, `allowed_labels`, `bucket_boundaries`, `data_class` e `retention_tier`.
2. O `SignalRegistry` valida o nome contra `aios_<subsistema>_<nome>_<unidade>`, a
   coerência tipo×unidade e a ausência de labels de identidade.
3. Valida que os buckets do histograma alinham-se aos limiares de SLO declarados.
4. Descritor passa a `Registered` (`./StateMachine.md` §7) e emite
   `telemetry.signal.registered`.

**Fluxos alternativos**

- **A1 — Nova versão:** alterar um descritor `Registered` exige nova
  `descriptor_version`; a anterior vai a `Deprecated` com janela de migração.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Nome fora da convenção ou unidade ausente | `AIOS-OBS-0004`; estado `Rejected` |
| E2 | Label de identidade em `allowed_labels` (`agent_urn`, `task_id`) | `AIOS-OBS-0004` |
| E3 | Tentativa de editar descritor `Registered` | `AIOS-OBS-0014` |
| E4 | `data_class = pii` sem base legal na retenção | `AIOS-OBS-0015` |

**Pós-condições:** sinal admitido na ingestão; disponível para SLO e dashboards.

---

## UC-003 — Quarentenar sinal por explosão de cardinalidade

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `CardinalityGuard` (automático) |
| **Atores secundários** | Módulo dono do sinal; SRE |
| **Pré-condições** | Orçamento definido em `telemetry.cardinality_budget` |
| **Requisitos** | FR-003, FR-005, FR-006; NFR-009 |

**Fluxo principal**

1. Um deploy introduz um label novo com valores por agente; as séries do sinal
   disparam.
2. O `CardinalityGuard` detecta `current_series` acima de `soft_threshold` (80%) e
   dispara alerta preventivo.
3. Ao ultrapassar `max_series`, aplica a `hard_action` configurada (default
   `quarantine`).
4. O descritor vai a `Quarantined`; a ingestão daquele sinal passa a ser descartada.
5. Emite `telemetry.cardinality.exceeded` e `telemetry.signal.quarantined`,
   notificando o `owner_module`.
6. O dono corrige o descritor (remove o label ofensor) em nova versão e reabilita.

**Fluxos alternativos**

- **A1 — `hard_action = drop_labels`:** apenas os labels não declarados são removidos;
  o sinal continua sendo coletado em forma degradada.
- **A2 — `hard_action = reject`:** o sinal é recusado na admissão, sem quarentena
  persistida (usado para sinais de baixa importância).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Alertas dependem do sinal quarentenado | Instâncias afetadas vão a `Expired`, **não** a `Resolved` (invariante I6/S3) |
| E2 | Orçamento do tenant inteiro estourado | Quarentena dos sinais de maior crescimento, **nunca** bloqueio do tenant inteiro |

**Pós-condições:** cardinalidade contida; dono notificado; visibilidade do sinal
temporariamente perdida e **explicitamente sinalizada**.

---

## UC-004 — Redigir PII na borda

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `PiiRedactor` (automático) |
| **Pré-condições** | `obs.pii.redaction_enabled = true`; `data_class` declarado no descritor |
| **Requisitos** | FR-007; NFR-014 |

**Fluxo principal**

1. Um log estruturado chega com um atributo classificado `pii` (ex.: `user.email`).
2. O `PiiRedactor` aplica `obs.pii.redaction_mode`: `tokenize` (substitui por um
   token estável) ou `drop` (remove o atributo).
3. O sinal segue para o pipeline **já redigido** — nenhuma cópia em claro é
   persistida.
4. O contador `aios_obs_pii_redacted_total` é incrementado por tipo de atributo.

**Fluxos alternativos**

- **A1 — Tokenização estável:** o token permite correlacionar ocorrências do mesmo
  titular sem revelar a identidade; a chave de tokenização é do `../021-Security/`.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Atributo pessoal não declarado no descritor | Detecção heurística → redação + alerta ao dono; o descritor **DEVE** ser corrigido |
| E2 | `redaction_enabled = false` | Configuração **não-recarregável**; alteração exige exceção documentada em ADR |

**Pós-condições:** nenhum dado pessoal em claro no armazenamento de telemetria
(NFR-014).

---

## UC-005 — Amostrar traces (*tail-based*)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `TraceSampler` (automático) |
| **Pré-condições** | Todos os spans do trace roteados ao mesmo gateway (`hash(trace_id)`) |
| **Requisitos** | FR-008 |

**Fluxo principal**

1. O SDK aplica *head sampling* barato (`obs.trace.head_sample_ratio`, 10%) e propaga
   a decisão em `traceparent`.
2. O gateway acumula os spans do trace até o fim da janela de montagem.
3. Ao fechar o trace, o `TraceSampler` decide:
   - `status = ERROR` em qualquer span ⇒ **retém 100%**;
   - duração total acima de `obs.trace.tail_keep_slow_ms` ⇒ **retém 100%**;
   - caso contrário ⇒ retém conforme a taxa configurada.
4. Traces retidos vão ao `TraceStore`; os demais são descartados com contabilização.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Spans do mesmo trace em gateways diferentes | Trace parcial; métrica `aios_obs_trace_fragmented_total`; roteamento revisado |
| E2 | Janela de montagem estourada | Trace decidido com os spans disponíveis; marcado como incompleto |

**Pós-condições:** 100% dos traces que explicam falha estão disponíveis; o volume
armazenado é uma fração do emitido.

---

## UC-006 — Definir e publicar um SLO

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Dono do serviço |
| **Pré-condições** | Capability `obs:slo:write`; sinais da `sli_query` registrados |
| **Requisitos** | FR-010; NFR-015 |

**Fluxo principal**

1. O dono submete `PutSlo` com `slo_key`, `sli_query` (PromQL), `objective`, `window`
   e `budget_policy`.
2. O `SloEngine` valida que todos os sinais referenciados estão `Registered` e que
   `objective ∈ (0,1)`.
3. Publica com `status = active` e nova `slo_version`.
4. As regras de alerta derivadas (*multi-burn-rate*) passam a ser avaliadas.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | `sli_query` referencia sinal não registrado ou quarentenado | `AIOS-OBS-0009` |
| E2 | `objective` fora de (0,1) | `AIOS-OBS-0009` |
| E3 | Alteração de SLO ativo | Cria **nova versão**; a anterior fica `deprecated`, preservando o histórico do ledger |

**Pós-condições:** SLI avaliado a cada `obs.slo.evaluation_interval_s`; ledger de error
budget alimentado.

---

## UC-007 — Avaliar error budget e sinalizar exaustão

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `SloEngine` (automático) |
| **Atores secundários** | `../028-Deployment/`, `../022-Policy/` |
| **Pré-condições** | SLO ativo; sinais disponíveis na janela |
| **Requisitos** | FR-010, FR-011; NFR-008 |

**Fluxo principal**

1. A cada intervalo, o `SloEngine` calcula o SLI da janela e
   `budget_consumed = (1 - SLI) / (1 - objective)`.
2. Grava o lançamento no `telemetry.error_budget_ledger` com `burn_rate_1h`,
   `burn_rate_6h` e `breached`.
3. Aplica os limiares *multi-burn-rate* (`./StateMachine.md` §6) e alimenta o
   `AlertEvaluator`.
4. Ao ultrapassar o limiar da `budget_policy`, emite
   `telemetry.budget.exhausted`.
5. `../028-Deployment/` congela releases não urgentes; `../022-Policy/` adia
   publicações de bundle não urgentes.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Sinal indisponível na janela | SLI não é calculado; alerta de dado ausente (T-09), **não** SLI de 100% |
| E2 | SLO violado na janela de conformidade | `telemetry.slo.breached` + registro em `../025-Audit/` |

**Pós-condições:** consumo de budget é dado, não opinião; decisões de release e de
política têm um sinal objetivo.

---

## UC-008 — Disparar, notificar e escalonar alerta

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `AlertEvaluator` (automático) |
| **Atores secundários** | `AlertRouter`, Alertmanager, `../029-Operations/` |
| **Pré-condições** | Regra `active` com `runbook_urn` válido |
| **Requisitos** | FR-011, FR-012, FR-022; NFR-005, NFR-008 |

**Fluxo principal**

1. A condição da regra torna-se verdadeira; instância criada em `Pending` (T-01) com
   `fingerprint` calculado a partir dos labels.
2. Decorrida `for_duration` com a condição contínua, transita a `Firing` (T-02).
3. Em **uma transação**: estado gravado + evento `telemetry.alert.firing` no outbox
   (invariante I5).
4. O `AlertRouter` agrupa, deduplica e entrega a notificação nos `notify_channels`,
   **incluindo o link do runbook**.
5. Sem reconhecimento em `obs.alert.escalation_after_s`, escalona (T-08) e emite
   `telemetry.alert.escalated`.

**Fluxos alternativos**

- **A1 — Condição cessa antes de `for_duration`:** vai a `Resolved` (T-03) sem jamais
  ter notificado um humano.
- **A2 — Janela de rollout:** com `deployment.rollout.started` ativo, alertas
  esperados são suprimidos pela janela configurada.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Regra sem runbook válido | Não transita a `Firing`; `AIOS-OBS-0008` na publicação da regra |
| E2 | Instância já aberta com o mesmo `fingerprint` | Deduplicação: nenhuma nova instância (invariante I1) |
| E3 | Alertmanager indisponível | Estado persiste; notificação entregue ao restabelecer (`./FailureRecovery.md` §2) |

**Pós-condições:** o plantão foi notificado uma única vez, com procedimento em mãos.

---

## UC-009 — Reconhecer alerta

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | SRE de plantão |
| **Pré-condições** | Instância em `Firing`; capability `obs:alert:ack` |
| **Requisitos** | FR-012 |

**Fluxo principal**

1. O operador chama `AcknowledgeAlert` com o id da instância.
2. Estado transita a `Acknowledged` (T-04); `acknowledged_by`/`acknowledged_at`
   gravados; escalonamento cancelado.
3. Emite `telemetry.alert.acknowledged`; o fato entra na trilha do `../025-Audit/`.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Instância já `Resolved` | `AIOS-OBS-0007` (transição inválida) |
| E2 | Sem capability | Negado pelo PDP; `AIOS-OBS-0003` |

**Pós-condições:** o alerta tem dono identificado; o escalonamento não acorda mais
ninguém.

---

## UC-010 — Silenciar alerta

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | SRE ou dono do serviço |
| **Pré-condições** | Capability `obs:silence:write`; justificativa preenchida |
| **Requisitos** | FR-014, FR-015 |

**Fluxo principal**

1. O operador cria um silêncio com `matcher` de labels, `reason` e `expires_at`.
2. O `ExceptionManager` do módulo limita o prazo a `obs.alert.max_silence_h` (72 h).
3. Instâncias que casam vão a `Silenced` (T-05); a **notificação** é suprimida.
4. A **avaliação continua** (invariante I3): se a condição cessar, a instância vai a
   `Resolved` normalmente.
5. Ao expirar, se a condição persistir, volta a `Firing` (T-06).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Silêncio sem `expires_at` ou sem `reason` | `AIOS-OBS-0013` |
| E2 | Prazo acima do máximo | `AIOS-OBS-0013` com o teto no `detail` |
| E3 | Silêncios repetidos sobre o mesmo alerta | Alerta `ObservabilitySilenceChurnHigh` (`./Monitoring.md` §5) |

**Pós-condições:** ruído suprimido por tempo determinado, com autor e motivo
registrados — nunca permanentemente.

---

## UC-011 — Resolver alerta e tratar ausência de dado

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `AlertEvaluator` (automático) |
| **Pré-condições** | Instância aberta |
| **Requisitos** | FR-012, FR-015, FR-016 |

**Fluxo principal**

1. A condição deixa de ser verdadeira e a reavaliação confirma em ≥ 1 janela
   (anti-*flapping*).
2. Instância transita a `Resolved` (T-07); `resolved_at` gravado.
3. Emite `telemetry.alert.resolved`; a notificação de resolução é entregue.

**Fluxos alternativos**

- **A1 — Ausência de dado:** sem dado por mais de `obs.alert.no_data_timeout_s`, a
  instância vai a `Expired` (T-09) e emite `telemetry.alert.nodata` — **nunca**
  `Resolved`.
- **A2 — Alvo legitimamente removido:** ao consumir `cluster.node.decommissioned`, o
  alvo é retirado e o alerta `nodata` correspondente é suprimido.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Condição oscilando (*flapping*) | Confirmação em janela adicional; contador `aios_obs_alert_flapping_total` |
| E2 | Recorrência após resolução | **Nova** instância com novo `started_at` (invariante I4) |

**Pós-condições:** o histórico distingue "resolveu" de "parei de ver" — distinção que
o pós-incidente depende.

---

## UC-012 — Consultar telemetria

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | SRE, dono do serviço, engenheiro depurando |
| **Atores secundários** | `../022-Policy/` (PDP) |
| **Pré-condições** | Capability `obs:query:*`; tenant coerente |
| **Requisitos** | FR-017, FR-018; NFR-007, NFR-013, NFR-017 |

**Fluxo principal**

1. O usuário consulta métricas (PromQL), traces (por `trace_id` ou filtro) ou logs.
2. O `QueryGateway` monta o `DecisionRequest` e consulta o PDP do `../022-Policy/`.
3. Autorizado, o `TenantRouter` restringe a consulta ao tenant do solicitante.
4. Orçamento de consulta aplicado (`obs.query.max_duration_s`,
   `obs.query.max_series_scanned`).
5. Resultado devolvido; a consulta é registrada para auditoria.

**Fluxos alternativos**

- **A1 — Pivô log→trace:** o log carrega `trace_id`; a consulta do trace completo é um
  clique (correlação RFC-0001 §5.6).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Consulta cross-tenant | `AIOS-OBS-0003` |
| E2 | Orçamento de consulta excedido | `AIOS-OBS-0010`; consulta abortada, serviço preservado |
| E3 | PDP indisponível | Consulta **negada** (`fail_mode = closed`) — a ingestão continua |
| E4 | Armazenamento indisponível | `AIOS-OBS-0011` |

**Pós-condições:** o investigador tem o dado; nenhum tenant viu o dado de outro.

---

## UC-013 — Publicar dashboard e runbook

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Dono do serviço |
| **Pré-condições** | Capabilities `obs:dashboard:write` e `obs:runbook:write` |
| **Requisitos** | FR-013, FR-018, FR-020, FR-024; NFR-016 |

**Fluxo principal**

1. O dono submete o runbook (`PutRunbook`) com `path` versionado e `steps_count > 0`.
2. Submete a regra de alerta referenciando `runbook_urn` — sem ele, a publicação é
   rejeitada.
3. Submete o dashboard (`PutDashboard`) declarando `signals_used`.
4. O `DashboardRegistry` valida que todos os sinais estão `Registered` e publica.
5. Eventos de rollout (`deployment.rollout.started`) e de política
   (`policy.bundle.updated`) passam a ser anotados na linha do tempo do painel.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Regra sem runbook | `AIOS-OBS-0008` |
| E2 | Dashboard referencia sinal não registrado/quarentenado | `AIOS-OBS-0009` |
| E3 | Runbook vazio (`steps_count = 0`) | `AIOS-OBS-0004` |
| E4 | Runbook sem revisão há mais de 180 dias | Publicação permitida, mas alerta `ObservabilityRunbookStale` dispara (NFR-016) |

**Pós-condições:** todo alerta ativo tem procedimento; todo painel usa sinal
governado.

---

## UC-014 — Aplicar retenção e *downsampling*

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `RetentionManager` (automático) |
| **Pré-condições** | Políticas definidas em `telemetry.retention_policy` |
| **Requisitos** | FR-019; NFR-011, NFR-012 |

**Fluxo principal**

1. O job periódico agrega métricas raw em resolução de 5 min e, depois, de 1 h.
2. Expurga a camada raw além de `obs.retention.metrics_raw_days` (15 d).
3. Expurga traces além de 7 d e logs além de 14 d (configurável por tenant).
4. Consultas de janela longa passam a ler automaticamente a camada agregada.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Sinal `pii` sem base legal declarada | Expurgo antecipado + `AIOS-OBS-0015` na próxima tentativa de configuração |
| E2 | *Downsampling* atrasado | Alerta interno; consultas longas caem para a camada disponível |
| E3 | Retenção configurada acima do permitido pela classificação | `AIOS-OBS-0015` |

**Pós-condições:** custo proporcional ao valor do dado ao longo do tempo; nenhuma
série cresce para sempre.

---

## UC-015 — Direito ao esquecimento sobre telemetria

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `RetentionManager`, disparado por evento |
| **Atores secundários** | `../021-Security/` (gatilho), `../025-Audit/` (comprovante) |
| **Pré-condições** | `security.rtbf.requested` recebido |
| **Requisitos** | FR-023; NFR-014 |

**Fluxo principal**

1. O módulo consome `aios.<tenant>.security.rtbf.requested` com o identificador do
   titular.
2. Localiza ocorrências nas três lojas (métricas agregadas, traces, logs).
3. Pseudonimiza ou expurga conforme a classificação do sinal, preservando **agregados
   sem identificação**.
4. Emite comprovante correlacionado para o `../025-Audit/`.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Dado já expurgado por retenção | Comprovante registra "sem ocorrências" — resultado válido |
| E2 | Ocorrência em backup de longo prazo | Marcada para expurgo na próxima rotação; prazo declarado no comprovante |
| E3 | Métrica agregada sem identificação | **Não** é expurgada: não há dado pessoal a esquecer |

**Pós-condições:** o titular não é mais identificável na telemetria; a capacidade de
medir o sistema permanece.

---

## Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Fluxos detalhados: `./SequenceDiagrams.md` · FSM: `./StateMachine.md`
- Contratos: `./API.md` · `./Events.md`
- Rastreabilidade: `./Requirements.md` §5 e §6

*Fim de `UseCases.md`.*
