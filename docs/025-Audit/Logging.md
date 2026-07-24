---
Documento: Logging
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0246, ADR-0255, ADR-0257
RFCs relacionados: RFC-0001, RFC-0250
Depende de: Metrics.md, Database.md, 021-Security, 024-Observability
---

# 025-Audit — Logging

Log estruturado com **Serilog → Seq** (`../024-Observability/`), correlacionado por
`trace_id`/`span_id` (W3C Trace Context, RFC-0001 §5.6).

## 1. O log deste módulo **não** é a trilha

| | Log de aplicação | Trilha de auditoria |
|---|------------------|---------------------|
| **O que é** | Diagnóstico do serviço 025 | O produto do serviço 025 |
| **Onde vive** | Seq | `audit.record` (PostgreSQL) + WORM |
| **Retenção** | 14 dias | Retenção legal por classe (default 1825 d) |
| **Mutável?** | Sim (rotação, amostragem) | Não (append-only, encadeado, selado) |
| **Prova?** | Não | Sim |

Esta distinção é a mais importante deste documento. Um operador que procure evidência
de uma ação no **log** do 025 está no lugar errado: o log diz que o registro foi
gravado; a **trilha** é o registro. E o log tem 14 dias — a prova, cinco anos.

> A mesma fronteira que o `../024-Observability/` estabelece entre telemetria e
> auditoria (ADR-0246) vale **dentro** deste módulo: o 025 também emite telemetria, e
> ela também é perecível.

## 2. Campos obrigatórios em todo log

| Campo | Origem | Exemplo |
|-------|--------|---------|
| `timestamp` | RFC 3339 UTC | `2026-07-23T11:02:00.184Z` |
| `level` | Serilog | `Error` |
| `trace_id` / `span_id` | OTel | `4bf92f3577b34da6a3ce929d0e0e4736` |
| `tenant_id` | `X-AIOS-Tenant` ou `aios.tenant` | `acme` |
| `service` | Fixo | `025-audit` |
| `component` | Componente emissor (PascalCase) | `ChainAppender` |
| `event` | Chave estável do evento de log (§3) | `aud.chain.break_detected` |
| `partition_key` | Quando aplicável | `acme:07` |
| `record_class` | Quando aplicável | `policy.decision` |

## 3. Catálogo de eventos de log

| Componente | `event` | Nível | Quando | Campos adicionais |
|------------|---------|-------|--------|-------------------|
| `AuditIngestGateway` | `aud.ingest.durable_ack_failed` | **Error** | Escrita recusada (`AIOS-AUD-0005`) | `producer_module`, `reason` |
| `AuditIngestGateway` | `aud.ingest.tenant_mismatch` | **Error** | Tenant divergente do certificado | `claimed_tenant`, `cert_subject` |
| `AuditIngestGateway` | `aud.ingest.unregistered_class` | Warning | `record_class` não registrada | `record_class`, `producer_module` |
| `AuditIngestGateway` | `aud.ingest.duplicate` | **Debug** | Reingestão idempotente | `event_id`, `existing_seq` |
| `RecordNormalizer` | `aud.normalize.invalid_envelope` | Warning | Envelope malformado | `error_code`, `missing_field` |
| `ChainAppender` | `aud.chain.appended` | **Debug** | Cada registro gravado | `seq`, `record_hash` (prefixo) |
| `ChainAppender` | `aud.chain.prev_mismatch` | **Error** | `prev_hash` divergente na escrita | `partition_key`, `expected`, `observed` |
| `SealService` | `aud.seal.created` | Information | Selo produzido | `seq_from`, `seq_to`, `record_count` |
| `SealService` | `aud.seal.signing_failed` | **Error** | KMS indisponível ou assinatura recusada | `key_ref`, `attempt` |
| `SealService` | `aud.seal.overdue` | Warning | Registros não selados além do intervalo | `unsealed_count`, `oldest_age_s` |
| `IntegrityVerifier` | `aud.verify.completed` | Information | Ciclo de verificação concluído | `mode`, `records_checked`, `duration_ms` |
| `IntegrityVerifier` | `aud.chain.break_detected` | **Error** | Divergência de hash | `partition_key`, `seq`, `expected_hash`, `observed_hash` |
| `IntegrityVerifier` | `aud.worm.mismatch` | **Error** | Banco diverge da cópia arquivada | `seal_urn`, `worm_object` |
| `CompletenessMonitor` | `aud.completeness.gap` | **Error** | Lacuna de sequência | `partition_key`, `expected_seq`, `observed_seq` |
| `CompletenessMonitor` | `aud.completeness.class_silent` | Warning | Volume abaixo do esperado | `record_class`, `producer_module`, `ratio` |
| `LegalHoldManager` | `aud.hold.applied` | Information | *Hold* ativado | `hold_urn`, `case_ref`, `requested_by`, `approved_by`, `records_held` |
| `LegalHoldManager` | `aud.hold.released` | Information | *Hold* liberado | `hold_urn`, `released_by`, `reason` |
| `LegalHoldManager` | `aud.hold.blocked_operation` | Warning | Expurgo/apagamento recusado por *hold* | `hold_urn`, `operation`, `requested_by` |
| `RetentionEnforcer` | `aud.retention.purged` | Information | Payloads expurgados | `record_class`, `purged_count`, `chain_intact` |
| `CryptoShredder` | `aud.erasure.completed` | Information | Apagamento executado | `receipt_hash`, `records_affected`, `latency_h` |
| `CryptoShredder` | `aud.erasure.verification_failed` | **Error** | Payload ainda decifrável após destruição | `subject_ref_hash`, `key_ref` |
| `WormArchiver` | `aud.worm.archived` | Information | Intervalo arquivado | `seal_urn`, `object`, `bytes` |
| `QueryService` | `aud.query.executed` | Information | Consulta atendida | `by`, `records_returned`, `duration_ms` |
| `QueryService` | `aud.query.denied` | Warning | Consulta negada pelo PDP | `capability`, `principal_kind` |
| `ExportService` | `aud.export.completed` | Information | Pacote pronto | `export_urn`, `record_count`, `package_hash` |
| `AuditIngestGateway` | `aud.record.mutation_attempt` | **Error** | Tentativa de alteração (`AIOS-AUD-0016`) | `principal`, `record_id` |

`aud.chain.appended` e `aud.ingest.duplicate` são **Debug** por desenho: a 50.000
registros/s, registrá-los em `Information` produziria mais log que todo o resto do
AIOS — e a informação já está na trilha e nas métricas.

`aud.record.mutation_attempt` é **Error** mesmo quando a tentativa é rejeitada com
sucesso: tentar alterar um registro de auditoria não é um erro de aplicação, é um
sinal.

## 4. O que a **trilha** registra que o log não registra

O 025 grava fatos sobre si mesmo **na própria trilha**, não apenas no log:

| Fato | Classe na trilha | Por que na trilha e não só no log |
|------|------------------|-----------------------------------|
| Consulta à trilha | `audit.access` | Quem consultou é evidência (ADR-0257) |
| Aplicação/liberação de *hold* | `audit.governance` | Prova de decisão jurídica |
| Apagamento executado | `privacy.erasure_receipt` | O comprovante precisa ser imutável |
| Exportação realizada | `audit.export` | Quem levou a trilha para fora |
| Recusa de operação por *hold* | `audit.governance` | A tentativa é parte do histórico do caso |
| Registro de nova classe auditável | `audit.governance` | Mudança do escopo do que é auditado |

Log de 14 dias não sustenta nenhum desses. É por isso que este módulo é **cliente de
si mesmo**.

## 5. Correlação ponta a ponta

```
   022-Policy                     025-Audit                    Investigador
        │ traceparent: 00-4bf9…-00f0…-01
        ├─ span: policy.evaluate
        │        └─ log 022: policy.decision.denied  decision_id=01J9ZB…
        │
        └─▶ evento ─▶ span: aud.chain.append (MESMO traceparent do fato)
                          │
                          └─ registro na trilha: decision_ref=01J9ZB…  seq=418291
                                    │
   ═══════════════════════════════════════════════════════════════════════
   Investigação: consulta a trilha por decision_ref  ─────────────────────▶
                 → obtém actor, action, resource, outcome, occurred_at
                 → pivota para o trace completo por trace_id
                 → a CONSULTA gera novo registro (audit.access)
```

O `traceparent` do fato original é **preservado** no registro de auditoria — ao
contrário do `../024-Observability/`, cujo pipeline usa trace próprio. A razão é a
finalidade: telemetria mede o transporte; auditoria precisa apontar para o **fato
auditado**.

## 6. Níveis e o que NÃO logar

| Nível | Uso |
|-------|-----|
| `Error` | Rompimento de garantia: quebra de cadeia, lacuna, divergência WORM, recusa de durabilidade, tentativa de alteração, falha de apagamento. |
| `Warning` | Classe silenciosa, selo atrasado, operação bloqueada por *hold*, consulta negada, envelope inválido. |
| `Information` | Selos, *holds*, expurgos, apagamentos, exportações, consultas atendidas. |
| `Debug` | Append individual, duplicata idempotente. |
| `Verbose` | Não usado em produção. |

**Nunca em log:**

- **Payload do registro auditado** (em claro ou cifrado) — ele vive na trilha, cifrado.
- `subject_ref` em claro — apenas o hash, quando indispensável ao diagnóstico.
- `record_hash` completo — apenas prefixo, para correlação sem replicar a prova.
- Material de chave, `connection string`, conteúdo de pacote de exportação.

> A regra mais sutil: **o log do módulo de auditoria não deve conter a auditoria**.
> Um log com o payload dos registros seria uma cópia não cifrada, não encadeada, não
> selada e com retenção diferente — exatamente o que a trilha existe para não ser.

## 7. Retenção e expurgo

| Destino | Retenção | Expurgo |
|---------|----------|---------|
| Seq — log de aplicação do 025 | 14 dias | Automático |
| `audit.record` (trilha) | Retenção legal por classe (`./Database.md` §11) | `RetentionEnforcer`, respeitando *holds* |
| Métricas `aios_aud_*` | Camadas do `../024-Observability/` | `RetentionManager` do 024 |
| Cópia WORM | `aud.worm.object_lock_days` (1825 d) | *Object lock* — não expurgável antes do prazo |

Direito ao esquecimento: o expurgo alcança **também** o log de aplicação, quando este
contiver `subject_ref` hasheado correlacionável. O comprovante correspondente é
registrado na trilha (UC-010).

## 8. Referências

- Métricas: `./Metrics.md` · Alertas e runbooks: `./Monitoring.md`
- Modelo físico: `./Database.md` · Privacidade: `./Security.md` §5 e §7
- Fronteira telemetria × auditoria: `../024-Observability/` (ADR-0246)
- Correlação: RFC-0001 §5.6

*Fim de `Logging.md`.*
