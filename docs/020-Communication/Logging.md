---
Documento: Logging
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0010, ADR-0202, ADR-0205, ADR-0207
RFCs relacionados: RFC-0001
Depende de: 024-Observability, 025-Audit, Security.md, Metrics.md
---

# 020-Communication — Log Estruturado

Stack: **Serilog → Seq**, com exportação OTel (`../024-Observability/`). Todo log é
estruturado (JSON). Os campos de correlação são os da RFC-0001 §5.6 e **não são
redefinidos aqui**.

> **Regra fundamental deste módulo:** o barramento **NÃO DEVE** logar o conteúdo das
> mensagens que transporta. Ele loga *sobre* as mensagens — subject, tamanho,
> resultado, latência — nunca o `data`. Um barramento que loga payload transforma o
> sistema de logs em cópia integral e não governada de todos os dados do AIOS.

---

## 1. Campos Obrigatórios

| Campo | Origem | Obrigatório | Observação |
|-------|--------|-------------|------------|
| `timestamp` | serviço | sim | RFC 3339, UTC. |
| `level` | serviço | sim | `Verbose`…`Fatal`. |
| `message_template` | Serilog | sim | Template, não a string interpolada. |
| `trace_id` / `span_id` | `traceparent` | sim | Correlação com traces OTel. |
| `tenant_id` | conta NATS / requisição | sim em operação com tenant | `_platform` em operação de plataforma. |
| `service` | config | sim | `aios-comm-svc`. |
| `component` | código | sim | `PublishGateway`, `A2AGateway`, … (PascalCase, igual a `./ClassDiagrams.md`). |
| `operation` | código | sim | `PublishMessage`, `OpenSession`, … |
| `subject` | mensagem | quando aplicável | Subject **de tipo** (nunca contém identificador de instância, por construção). |
| `stream` / `consumer` | contexto | quando aplicável | Para diagnóstico de entrega. |
| `event_id` | envelope | quando aplicável | ULID do evento — permite correlacionar com a DLQ. |
| `error_code` | catálogo | em falha | `AIOS-BUS-*` / `AIOS-A2A-*`. |
| `duration_ms` | código | em operação concluída | Correlaciona com as métricas. |
| `outcome` | código | sim | `ok` \| `rejected` \| `denied` \| `failed`. |

---

## 2. Níveis — critério de uso

| Nível | Quando usar | Exemplos neste módulo |
|-------|-------------|-----------------------|
| `Verbose` | Diagnóstico fino; **desligado** em produção. | Passo a passo da validação de envelope. |
| `Debug` | Fluxo interno útil em investigação. | Escolha de stream para um subject; cache de schema. |
| `Information` | Fato relevante e esperado. | Subject registrado; stream criado; sessão A2A estabelecida/encerrada. |
| `Warning` | Anomalia que ainda não é falha. | Cota aplicada; consumidor lento; nó fora com quórum mantido. |
| `Error` | Operação falhou e exige atenção. | Mensagem quarentenada; falha de publicação por quórum; sessão A2A falhada. |
| `Fatal` | Serviço não pode continuar. | Impossível conectar ao NATS na inicialização. |

**Amostragem obrigatória:** rejeições de publicação em regime de saturação podem ocorrer
milhares de vezes por segundo. `PublishRejected` **DEVE** ser amostrado (1:100 por
`reason`+`domain`) com um contador exato em `aios_bus_publish_rejected_total`. Logar
cada rejeição individualmente transformaria uma tempestade de tráfego em uma tempestade
de logs — e o segundo problema costuma ser pior que o primeiro.

---

## 3. Eventos de Log Canônicos

| `operation` | Nível | Campos específicos | Correspondência |
|-------------|-------|--------------------|-----------------|
| `RegisterSubject` | Information | `domain`, `entity`, `action`, `producer_module` | UC-001 |
| `DeprecateSubject` | Warning | `subject`, `reason` | UC-001 |
| `PutStream` | Information | `stream`, `replicas`, `max_age`, `change_kind` (create\|compatible\|migrate) | UC-002 |
| `DrainStream` | Warning | `stream`, `pending_at_drain` | UC-002 A4 |
| `PutConsumer` | Information | `stream`, `consumer`, `max_deliver`, `ack_wait` | UC-003 |
| `PublishRejected` | Warning (amostrado) | `subject`, `reason`, `error_code`, `producer_module` | UC-004 E1..E5 |
| `MessageDeadLettered` | Error | `stream`, `consumer`, `event_id`, `delivery_count`, `last_error` | UC-006 |
| `DeadLetterReplayed` | Warning | `dlq_id`, `event_id`, `approved_by` | UC-007 |
| `DeadLetterDiscarded` | Warning | `dlq_id`, `reason`, `approved_by` | UC-007 A1 |
| `StreamReplayed` | Warning | `stream`, `from`, `to`, `messages` | UC-008 |
| `ProvisionAccount` | Information | `tenant_id`, `limits` | UC-009 |
| `PutAccountExport` | **Warning** | `subject`, `from_tenant`, `to_tenant`, `justification`, `approved_by` | UC-010 |
| `OpenSession` | Information | `session_urn`, `peer_kind`, `capabilities`, `handshake_ms` | UC-011 |
| `SessionSuspended` | Warning | `session_urn`, `cause` (quota\|policy) | T-06 |
| `CloseSession` | Information | `session_urn`, `close_reason`, `message_count`, `bytes_total` | T-09 |
| `SessionFailed` | Error | `session_urn`, `close_reason`, `error_code` | T-10 |
| `ConsumerStalled` | Warning | `stream`, `consumer`, `pending`, `stalled_for_ms` | UC-014 |
| `QuotaThrottled` | Warning (amostrado) | `scope`, `limit_kind`, `observed`, `limit` | UC-014 |
| `ClusterDegraded` | Warning/Error | `nodes_healthy`, `jetstream_quorum` | §7 de `./SequenceDiagrams.md` |
| `PolicyDecision` | Information/Warning | `capability`, `decision`, `deny_reason` | FR-017 |

`PutAccountExport` é registrado em **Warning** deliberadamente, embora seja uma operação
bem-sucedida: exportar subject entre contas é o único caminho legítimo de tráfego entre
tenants, e merece destaque em qualquer revisão de log.

---

## 4. Exemplos

### 4.1 Sessão A2A estabelecida

```json
{
  "timestamp": "2026-07-22T15:10:00.412Z",
  "level": "Information",
  "message_template": "Sessão A2A {SessionUrn} estabelecida com {PeerUrn} em {HandshakeMs}ms",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "service": "aios-comm-svc",
  "component": "A2AGateway",
  "operation": "OpenSession",
  "session_urn": "urn:aios:acme:a2asession:01J9ZC8U1W3Y5A7C9E1G3J5L7N",
  "peer_kind": "internal",
  "capabilities": ["task.delegate", "memory.share"],
  "handshake_ms": 118,
  "duration_ms": 214,
  "outcome": "ok"
}
```

### 4.2 Publicação rejeitada (amostrada)

```json
{
  "timestamp": "2026-07-22T15:12:33.907Z",
  "level": "Warning",
  "message_template": "Publicação rejeitada em {Subject}: {Reason}",
  "trace_id": "9c1a2b3c4d5e6f708192a3b4c5d6e7f8",
  "tenant_id": "acme",
  "component": "PublishGateway",
  "operation": "PublishRejected",
  "subject": "aios.acme.tool.usage.spike",
  "reason": "unregistered",
  "error_code": "AIOS-BUS-0004",
  "producer_module": "015-Tool-Manager",
  "sampled": "1:100",
  "outcome": "rejected"
}
```

### 4.3 Mensagem quarentenada

```json
{
  "timestamp": "2026-07-22T15:05:41.220Z",
  "level": "Error",
  "message_template": "Mensagem {EventId} quarentenada após {DeliveryCount} tentativas",
  "trace_id": "1a2b3c4d5e6f708192a3b4c5d6e7f809",
  "tenant_id": "acme",
  "component": "DeadLetterManager",
  "operation": "MessageDeadLettered",
  "stream": "MEMORY_EVENTS",
  "consumer": "learning-consolidation",
  "event_id": "01J9ZCAW3Y5A7C9E1G3J5L7N9Q",
  "subject": "aios.acme.memory.item.consolidated",
  "delivery_count": 5,
  "last_error": "ValidationException: campo 'layer' ausente",
  "outcome": "failed"
}
```

O `event_id` no log é a ponte para `comm.dlq_entry`, onde o envelope completo está
preservado sob RLS — o log dá o rastro, a DLQ dá o conteúdo, e o acesso a cada um tem
controle próprio.

---

## 5. Regras de Privacidade no Log

| Regra | Detalhe |
|-------|---------|
| **Sem payload** | O campo `data` do envelope **NÃO DEVE** ser logado, em nenhum nível, nem truncado. Para diagnóstico, use `event_id` + a DLQ. |
| **Sem PII** | Identificação é sempre por URN opaco (RFC-0001 §7). |
| **Sem credenciais** | NKey, JWT, seeds e certificados **NÃO DEVEM** aparecer, nem mascarados parcialmente. |
| **`last_error` saneado** | A mensagem de erro do consumidor é registrada como **classe + descrição técnica**; se contiver eco de payload, é truncada no `DeadLetterManager` antes de persistir. |
| **Subjects são seguros por construção** | Como subjects identificam tipos e nunca instâncias (regra RN-04), logá-los não revela dado de negócio. |

Violações dessas regras são incidente de segurança, não defeito de formatação.

---

## 6. Retenção e Amostragem

| Categoria | Nível | Retenção em Seq | Amostragem |
|-----------|-------|-----------------|------------|
| Operações administrativas e A2A | Information+ | 1 ano | 100% (nunca amostrado) |
| Segurança (`PublishRejected{reason=producer}`, `PutAccountExport`) | Warning+ | 1 ano | 100% |
| Rejeições por cota e por validação | Warning | 90 dias | 1:100 por `reason`+`domain` |
| DLQ e falhas de entrega | Error | 1 ano | 100% |
| Fluxo interno | Debug | 7 dias | 100% em incidente, 1% em regime |
| Diagnóstico fino | Verbose | desligado em produção | — |

Log **não substitui auditoria**: a trilha imutável de sessões A2A, exports e operações
privilegiadas vive em `../025-Audit/` (`./Security.md` §9). O log é para diagnóstico; a
auditoria é para prova.

---

## 7. Correlação Ponta a Ponta

O `traceparent` viaja **dentro do envelope** (RFC-0001 §5.2), não apenas no cabeçalho
de transporte — é o que permite que um trace atravesse a fronteira assíncrona:

```
   006-Kernel                    NATS                     025-Audit
   span "Spawn"                                           span "AuditIngest"
      │ traceparent: 00-4bf9…-00f0…-01                          ▲
      ├─ span "Outbox.Enqueue"                                  │
      ├─ span "Bus.Publish" ──── envelope.traceparent ──────────┤
      │                          (mesmo trace_id,               │
      │                           novo span_id no consumidor)   │
      ▼                                                          │
   log(trace_id) → Seq        métrica(exemplar) → Prometheus   log(trace_id) → Seq
                                    │
                                    └── auditoria(trace_id) → 025-Audit
```

Sem essa propagação, um evento que atravessa seis módulos viraria seis incidentes
independentes em vez de um trace — que é exatamente o que NFR-013 existe para impedir.

---

## 8. Referências

- Métricas: `./Metrics.md` · Alertas: `./Monitoring.md`
- Segurança e auditoria: `./Security.md` · `../025-Audit/`
- Correlação (contrato): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
