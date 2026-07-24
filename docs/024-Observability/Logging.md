---
Documento: Logging
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0243, ADR-0246, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240
Depende de: Metrics.md, Database.md, 021-Security, 025-Audit
---

# 024-Observability — Logging

Log estruturado com **Serilog → Seq**, correlacionado por `trace_id`/`span_id` (W3C
Trace Context, RFC-0001 §5.6).

## 1. O módulo tem dois papéis de log — não os confunda

| Papel | O que é | Onde vive |
|-------|---------|-----------|
| **Operador do `LogPipeline`** | Recebe, redige, roteia e armazena os logs de **todos os outros módulos**. | Seq (multi-tenant) |
| **Emissor de log próprio** | Registra a operação do **próprio** pipeline: admissões, quarentenas, descartes, transições de alerta. | Seq (tenant `_platform`) |

O segundo papel tem uma armadilha: registrar em log cada lote processado produziria
mais log do que todo o resto do AIOS junto. Por isso a operação de rotina do pipeline
vive em **métricas** (`./Metrics.md`), e o log próprio registra apenas **eventos
discretos e acionáveis** (§3).

## 2. Três destinos distintos

| Destino | O que registra | Retenção | Fonte da verdade para |
|---------|----------------|----------|-----------------------|
| **Log de aplicação** (Serilog→Seq) | Operação do serviço: erros, transições, degradações. | 14 dias | Diagnóstico técnico |
| **Métricas** (`aios_obs_*`) | Volumes, taxas e latências do pipeline. | camadas de `./Database.md` §11.2 | Tendência e alerta |
| **Trilha imutável** (`../025-Audit/`) | Fatos de governança: silêncio criado, SLO alterado, retenção modificada, expurgo executado. | Definida pelo `025` | Prova legal e auditoria |

> A fronteira com o `../025-Audit/` (ADR-0246) é a mesma que este módulo impõe aos
> outros: **telemetria é amostrável e perecível; auditoria não é.** Um silêncio criado
> às 3h da manhã durante um incidente é fato de governança — vai para a trilha, não
> apenas para o log de 14 dias.

## 3. Campos obrigatórios em todo log

| Campo | Origem | Exemplo |
|-------|--------|---------|
| `timestamp` | RFC 3339 UTC | `2026-07-23T10:15:00.123Z` |
| `level` | Serilog | `Warning` |
| `trace_id` / `span_id` | OTel | `4bf92f3577b34da6a3ce929d0e0e4736` |
| `tenant_id` | `aios.tenant` ou `_platform` | `acme` |
| `service` | Fixo | `024-observability` |
| `component` | Componente emissor (PascalCase) | `CardinalityGuard` |
| `event` | Chave estável do evento de log (§4) | `obs.signal.quarantined` |
| `signal_name` | Quando aplicável | `aios_kernel_syscall_latency_ms` |

## 4. Catálogo de eventos de log

| Componente | `event` | Nível | Quando | Campos adicionais |
|------------|---------|-------|--------|-------------------|
| `TelemetryIngestGateway` | `obs.ingest.tenant_mismatch` | **Error** | `aios.tenant` ≠ identidade do certificado | `claimed_tenant`, `cert_subject` |
| `TelemetryIngestGateway` | `obs.ingest.batch_rejected` | Warning | Lote acima do tamanho máximo | `batch_bytes`, `limit` |
| `TelemetryIngestGateway` | `obs.ingest.dropped` | Warning | Descarte por fila cheia (**agregado por janela**, não por lote) | `dropped_count`, `reason`, `window_s` |
| `TelemetryIngestGateway` | `obs.ingest.degraded` | **Error** | Taxa de descarte acima de `drop_alert_ratio` | `drop_ratio`, `threshold` |
| `SignalAdmission` | `obs.signal.unregistered` | Warning | Sinal desconhecido admitido/quarentenado | `owner_module` (inferido), `action` |
| `SignalRegistry` | `obs.signal.registered` | Information | Novo descritor admitido | `descriptor_version`, `owner_module` |
| `SignalRegistry` | `obs.signal.rejected` | Warning | Registro reprovado na convenção | `error_code`, `violation` |
| `CardinalityGuard` | `obs.cardinality.soft_threshold` | Warning | Orçamento acima de `soft_threshold` | `current_series`, `max_series`, `ratio` |
| `CardinalityGuard` | `obs.signal.quarantined` | **Error** | Sinal quarentenado por cardinalidade | `offending_labels`, `owner_module` |
| `PiiRedactor` | `obs.pii.undeclared` | **Error** | PII detectada em sinal não declarado | `owner_module`, `attribute_kind` (**nunca o valor**) |
| `TraceSampler` | `obs.trace.fragmented` | Warning | Trace montado incompleto | `trace_id`, `span_count` |
| `SloEngine` | `obs.slo.evaluated` | **Debug** | Cada avaliação de SLO | `slo_key`, `sli_value`, `burn_rate_1h` |
| `SloEngine` | `obs.slo.breached` | Information | SLO violado na janela | `slo_key`, `objective`, `sli_value` |
| `AlertEvaluator` | `obs.alert.transition` | Information | Transição da FSM | `rule_key`, `from`, `to`, `alert_id` |
| `AlertEvaluator` | `obs.alert.nodata` | Warning | Instância → `Expired` | `rule_key`, `no_data_for_s`, `previous_state` |
| `AlertRouter` | `obs.alert.notify_failed` | **Error** | Falha de entrega de notificação | `channel`, `attempt`, `alert_id` |
| `AlertRouter` | `obs.alert.silenced` | Information | Instância silenciada | `silence_urn`, `created_by`, `expires_at` |
| `QueryGateway` | `obs.query.denied` | Warning | Consulta negada pelo PDP | `capability`, `principal` |
| `QueryGateway` | `obs.query.aborted` | Warning | Consulta abortada por orçamento | `store`, `series_scanned`, `duration_ms` |
| `RetentionManager` | `obs.retention.purged` | Information | Expurgo executado | `tier`, `signal_kind`, `purged_count` |
| `RetentionManager` | `obs.erasure.completed` | Information | Direito ao esquecimento executado | `receipt_id`, `stores_affected` |
| `SelfTelemetry` | `obs.self.degraded` | **Error** | O próprio pipeline em degradação | `queue_depth`, `drop_ratio` |

`obs.slo.evaluated` é **Debug** por desenho: com N SLOs avaliados a cada 60 s,
registrá-lo em `Information` produziria ruído sem informação — a série já está em
`aios_obs_slo_sli_ratio`.

`obs.ingest.dropped` é **agregado por janela**, não emitido por lote descartado. Um log
por descarte, sob saturação, faria o sistema de log competir com o pipeline saturado
que ele tenta reportar — a forma mais confiável de transformar degradação em queda.

## 5. Correlação ponta a ponta

```
   Serviço observado (006-Kernel)        024-Observability            Investigador
        │ traceparent: 00-4bf9…-00f0…-01
        ├─ span: kernel.syscall.spawn
        │        └─ log: kernel.syscall.denied  trace_id=4bf9…
        │
        └──▶ OTLP ──▶ obs.ingest.batch (span próprio, trace do 024)
                          │
                          └─ log obs.signal.quarantined  signal_name=…
                                                          │
   ════════════════════════════════════════════════════════│════════════════
   O investigador consulta o log do KERNEL por trace_id ───┘
   e pivota para o trace completo em um clique (RFC-0001 §5.6).

   O trace do 024 é OUTRO trace: o pipeline não herda o traceparent do
   sinal que transporta. Herdá-lo faria a telemetria de um agente aparecer
   como filha do span daquele agente — e um trace de negócio ganharia
   milhares de spans de infraestrutura.
```

## 6. Níveis e o que NÃO logar

| Nível | Uso |
|-------|-----|
| `Error` | Rompimento de garantia: tenant divergente, PII não declarada, quarentena, degradação de ingestão, falha de notificação. |
| `Warning` | Descarte agregado, sinal desconhecido, pressão de cardinalidade, consulta negada/abortada, `nodata`. |
| `Information` | Registro de sinal, transições de alerta, silêncios, expurgos, violação de SLO. |
| `Debug` | Avaliação individual de SLO, decisão de amostragem por trace. |
| `Verbose` | Não usado em produção. |

**Nunca em log:**

- Valores de atributo classificados `pii` ou `sensitive` — apenas o **tipo**
  (`attribute_kind`).
- Conteúdo de log de outros módulos (o `LogPipeline` **transporta**, não **registra**).
- Amostras de datapoint ou de span transportados.
- `connection string`, chave de tokenização, material de certificado.

> A regra mais sutil: o módulo que opera o pipeline de log **não deve logar o conteúdo
> do log alheio**. Um erro de processamento deve registrar `signal_name`,
> `owner_module` e o código do erro — jamais a linha que causou o problema, que pode
> conter exatamente o dado que a redação deveria ter protegido.

## 7. Retenção e expurgo

| Destino | Retenção | Expurgo |
|---------|----------|---------|
| Seq — log do próprio módulo (`_platform`) | 14 dias | Automático |
| Seq — logs dos módulos observados | `obs.retention.logs_days` (14 d, por tenant) | `RetentionManager` |
| Métricas próprias | camadas de `./Database.md` §11.2 | `RetentionManager` |
| Trilha de governança | Definida pelo `../025-Audit/` | Fora deste módulo |

Direito ao esquecimento: o expurgo de logs por titular é executado pelo
`RetentionManager` (UC-015) sobre **todas** as lojas, inclusive o log do próprio
módulo, com comprovante em `../025-Audit/`.

## 8. Referências

- Métricas: `./Metrics.md` · Alertas e runbooks: `./Monitoring.md`
- Modelo físico: `./Database.md` · Privacidade: `./Security.md` §4 e §6
- Trilha imutável: `../025-Audit/` · Correlação: RFC-0001 §5.6

*Fim de `Logging.md`.*
