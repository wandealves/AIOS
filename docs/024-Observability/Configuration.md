---
Documento: Configuration
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0242, ADR-0243, ADR-0244, ADR-0247, ADR-0248, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §8, NonFunctionalRequirements.md, Deployment.md
---

# 024-Observability — Configuração

Prefixo de variável de ambiente: **`AIOS_OBS_`**. A chave `obs.ingest.queue_size`
corresponde a `AIOS_OBS_INGEST_QUEUE_SIZE`.

Escopos: `global` (instalação), `tenant`, `service`, `signal`. Quando uma chave existe
em mais de um escopo, vale a **mais específica**, limitada pelo teto do escopo mais
geral — um tenant **NÃO DEVE** conseguir configurar-se para além do que a plataforma
permite.

## 1. Ingestão

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.ingest.max_batch_mb` | int | 4 | 1–32 | global | sim | Tamanho máximo do lote OTLP (`AIOS-OBS-0012`). |
| `obs.ingest.queue_size` | int | 100000 | 1000–1000000 | global | sim | Fila em memória antes do descarte. |
| `obs.ingest.disk_buffer_mb` | int | 2048 | 0–65536 | global | **não** | Buffer em disco do `otel-agent`; sustenta o RPO de 1 min (NFR-011). |
| `obs.ingest.drop_alert_ratio` | decimal | 0.01 | 0.0–1.0 | global | sim | Fração de descarte que dispara `telemetry.ingest.degraded`. |
| `obs.ingest.fail_mode` | enum | `open` | {open,closed} | global | **não** | **`open` é o desenho** (ADR-0243). |

> `obs.ingest.fail_mode` é **não-recarregável** e default `open` por decisão
> arquitetural, não por conveniência. `closed` existe apenas para diagnóstico em
> laboratório: em produção, ele transformaria uma saturação do coletor em
> indisponibilidade de todos os serviços instrumentados.

## 2. Registro de sinais e cardinalidade

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.signal.registration_required` | bool | `true` | {true,false} | global | **não** | Exige registro prévio do sinal (ADR-0241). |
| `obs.signal.unknown_action` | enum | `quarantine` | {quarantine,drop} | global | sim | Ação para sinal não registrado. |
| `obs.cardinality.max_series_per_tenant` | int | 1000000 | 1000–50000000 | tenant | sim | Teto de séries ativas (NFR-009). |
| `obs.cardinality.max_series_per_signal` | int | 10000 | 10–1000000 | signal | sim | Teto por sinal. |
| `obs.cardinality.soft_threshold` | decimal | 0.8 | 0.1–1.0 | tenant | sim | Fração que dispara alerta preventivo. |

`obs.signal.registration_required = false` desativaria o único ponto do sistema capaz
de impedir a explosão de cardinalidade **antes** dela acontecer — por isso é
não-recarregável.

## 3. Amostragem de traces

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.trace.head_sample_ratio` | decimal | 0.1 | 0.0–1.0 | tenant | sim | Amostragem barata no SDK. |
| `obs.trace.tail_keep_errors` | bool | `true` | {true,false} | global | **não** | Retém 100% dos traces com erro (ADR-0244). |
| `obs.trace.tail_keep_slow_ms` | int | 1000 | 1–60000 | tenant | sim | Retém traces acima deste limiar. |

## 4. Privacidade

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.pii.redaction_enabled` | bool | `true` | {true,false} | global | **não** | Redação na borda (ADR-0249). |
| `obs.pii.redaction_mode` | enum | `tokenize` | {tokenize,drop} | tenant | sim | Tokenizar (permite correlação) ou remover. |

## 5. Retenção

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.retention.metrics_raw_days` | int | 15 | 1–90 | global | sim | Retenção de métrica bruta. |
| `obs.retention.metrics_5m_days` | int | 90 | 7–400 | global | sim | Retenção da agregação de 5 min. |
| `obs.retention.metrics_1h_days` | int | 400 | 30–1825 | global | sim | Retenção da agregação de 1 h. |
| `obs.retention.traces_days` | int | 7 | 1–90 | global | sim | Retenção de traces amostrados. |
| `obs.retention.logs_days` | int | 14 | 1–365 | tenant | sim | Retenção de logs. |

## 6. SLO e alerta

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.slo.evaluation_interval_s` | int | 60 | 10–3600 | global | sim | Intervalo de avaliação de SLO. |
| `obs.slo.default_window_days` | int | 30 | 1–90 | tenant | sim | Janela de conformidade default. |
| `obs.alert.default_for_s` | int | 300 | 0–86400 | tenant | sim | Histerese default (`for_duration`). |
| `obs.alert.no_data_timeout_s` | int | 600 | 60–86400 | tenant | sim | Ausência de dado até `Expired` (T-09). |
| `obs.alert.escalation_after_s` | int | 900 | 60–86400 | tenant | sim | Escalonamento sem reconhecimento (T-08). |
| `obs.alert.max_silence_h` | int | 72 | 1–720 | tenant | sim | Prazo máximo de silêncio (`AIOS-OBS-0013`). |
| `obs.alert.runbook_required` | bool | `true` | {true,false} | global | **não** | Exige runbook por regra (ADR-0247). |

> `obs.alert.runbook_required` é não-recarregável pela mesma razão que
> `pol.test.required_before_publish` é no `../022-Policy/`: é um controle de qualidade
> que, se pudesse ser desligado em runtime, seria desligado exatamente no momento de
> pressa em que mais importa.

## 7. Consulta e dependências

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `obs.query.max_duration_s` | int | 30 | 1–300 | tenant | sim | Orçamento de tempo por consulta (`AIOS-OBS-0010`). |
| `obs.query.max_series_scanned` | int | 5000000 | 1000–100000000 | tenant | sim | Orçamento de varredura por consulta. |
| `obs.policy.fail_mode` | enum | `closed` | {closed,open} | global | sim | PEP de **consulta/administração** — fail-closed, ao contrário da ingestão. |
| `obs.db.connection_string` | secret | — | — | global | **não** | PostgreSQL (schema `telemetry`); credencial dinâmica do `../021-Security/`. |
| `obs.redis.connection_string` | secret | — | — | global | **não** | Contadores de cardinalidade e limites. |
| `obs.nats.url` | string | — | — | global | **não** | Barramento (`../020-Communication/`). |
| `obs.prometheus.remote_write_url` | string | — | — | global | **não** | Destino das métricas. |
| `obs.trace_store.endpoint` | string | — | — | global | **não** | Tempo/Jaeger. |
| `obs.log_store.endpoint` | string | — | — | global | **não** | Seq. |
| `obs.alertmanager.url` | string | — | — | global | **não** | Entrega de notificações. |
| `obs.outbox.poll_interval_ms` | int | 200 | 50–5000 | global | sim | Intervalo do `observability-outbox-relay`. |

Segredos **NÃO DEVEM** vir de arquivo em imagem: são injetados em runtime pelo
`../021-Security/` (`./Deployment.md` §5).

## 8. A assimetria de `fail_mode`

| Plano | Chave | Default | Racional |
|-------|-------|---------|----------|
| **Ingestão** | `obs.ingest.fail_mode` | **`open`** | Perder telemetria degrada a visibilidade; bloquear o emissor derruba o produto. |
| **Consulta/administração** | `obs.policy.fail_mode` | **`closed`** | Sem poder autorizar, ninguém lê a telemetria de ninguém. |

Essa assimetria é deliberada e é o oposto do `../022-Policy/`, que é fail-closed em
**todo** o seu caminho quente. A diferença tem uma explicação simples: negar uma
autorização é seguro; negar uma métrica não protege nada e ainda quebra quem a emitia.

## 9. Precedência e validação

```
   default do produto
        ↓ sobrescrito por
   configuração global da instalação
        ↓ sobrescrito por
   configuração do tenant        (limitada pelo teto global)
        ↓ sobrescrito por
   configuração de service / signal
```

Validações na carga (falham o *startup*, não o primeiro lote):

| # | Validação | Erro |
|---|-----------|------|
| V-01 | `obs.retention.metrics_raw_days` ≤ `metrics_5m_days` ≤ `metrics_1h_days` | *startup* recusado |
| V-02 | `obs.cardinality.soft_threshold` ∈ (0,1] | *startup* recusado |
| V-03 | `obs.alert.no_data_timeout_s` > `obs.slo.evaluation_interval_s` | *startup* recusado (senão todo alerta expira antes de avaliar) |
| V-04 | `obs.trace.head_sample_ratio` ∈ [0,1] | *startup* recusado |
| V-05 | `obs.alert.max_silence_h` ≤ teto global | valor truncado ao teto + alerta |
| V-06 | Chave não-recarregável alterada em runtime | ignorada + log `Warning` + métrica |

## 10. Exemplo — `appsettings.Production.json`

```json
{
  "Observability": {
    "Ingest":      { "MaxBatchMb": 4, "QueueSize": 100000, "DiskBufferMb": 2048,
                     "DropAlertRatio": 0.01, "FailMode": "open" },
    "Signal":      { "RegistrationRequired": true, "UnknownAction": "quarantine" },
    "Cardinality": { "MaxSeriesPerTenant": 1000000, "MaxSeriesPerSignal": 10000,
                     "SoftThreshold": 0.8 },
    "Trace":       { "HeadSampleRatio": 0.1, "TailKeepErrors": true, "TailKeepSlowMs": 1000 },
    "Pii":         { "RedactionEnabled": true, "RedactionMode": "tokenize" },
    "Retention":   { "MetricsRawDays": 15, "Metrics5mDays": 90, "Metrics1hDays": 400,
                     "TracesDays": 7, "LogsDays": 14 },
    "Slo":         { "EvaluationIntervalS": 60, "DefaultWindowDays": 30 },
    "Alert":       { "DefaultForS": 300, "NoDataTimeoutS": 600, "EscalationAfterS": 900,
                     "MaxSilenceH": 72, "RunbookRequired": true },
    "Query":       { "MaxDurationS": 30, "MaxSeriesScanned": 5000000 },
    "Policy":      { "FailMode": "closed" }
  }
}
```

Equivalente por variável de ambiente:

```bash
AIOS_OBS_INGEST_FAIL_MODE=open
AIOS_OBS_SIGNAL_REGISTRATION_REQUIRED=true
AIOS_OBS_CARDINALITY_MAX_SERIES_PER_TENANT=1000000
AIOS_OBS_TRACE_TAIL_KEEP_ERRORS=true
AIOS_OBS_PII_REDACTION_ENABLED=true
AIOS_OBS_ALERT_RUNBOOK_REQUIRED=true
AIOS_OBS_POLICY_FAIL_MODE=closed
```

## 11. Perfis por ambiente

| Chave | Produção | Homologação | Desenvolvimento |
|-------|----------|-------------|-----------------|
| `obs.ingest.fail_mode` | `open` | `open` | `open` |
| `obs.signal.registration_required` | `true` | `true` | `false` |
| `obs.trace.head_sample_ratio` | 0.1 | 0.5 | 1.0 |
| `obs.retention.metrics_raw_days` | 15 | 7 | 1 |
| `obs.retention.logs_days` | 14 | 7 | 1 |
| `obs.alert.runbook_required` | `true` | `true` | `false` |
| `obs.policy.fail_mode` | `closed` | `closed` | `open` |
| `obs.pii.redaction_enabled` | `true` | `true` | `true` |

> `obs.pii.redaction_enabled` é `true` em **todos** os ambientes, inclusive
> desenvolvimento. Ambientes de desenvolvimento recebem dado real com mais frequência
> do que se admite, e a redação é barata o bastante para não justificar a exceção.

## 12. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §8
- Metas associadas: `./NonFunctionalRequirements.md`
- Erros: `./API.md` §6 · Implantação: `./Deployment.md`
- Segredos: `../021-Security/` · Operação: `./Monitoring.md`

*Fim de `Configuration.md`.*
