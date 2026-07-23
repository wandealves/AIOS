---
Documento: Metrics
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0205, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001
Depende de: 024-Observability, NonFunctionalRequirements.md, Monitoring.md
---

# 020-Communication — Catálogo de Métricas

Convenção de nome: `aios_<subsistema>_<nome>_<unidade>`, subsistema = **`bus`**.
Exportação **OpenTelemetry → Prometheus** (`../024-Observability/`). Labels comuns:
`service`, `instance`, `env`.

> **Regra de cardinalidade.** `tenant_id`, `agent_urn`, `event_id` e `subject` completo
> **NÃO DEVEM** ser labels — com 10⁴ tenants e 10⁵ subjects, isso multiplicaria as
> séries temporais até inviabilizar o Prometheus. Use `domain` e `stream` como
> agregação, e traces/logs para o recorte por tenant (`./Logging.md`).

---

## 1. Tráfego e Latência

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_messages_total` | counter | `domain`, `direction` (in\|out), `durable` | 1 | Mensagens publicadas/entregues. | NFR-004 |
| `aios_bus_bytes_total` | counter | `domain`, `direction` | bytes | Volume trafegado. | NFR-004 |
| `aios_bus_publish_latency_ms` | histogram | `durable` (true\|false), `domain` | ms | Latência de publicação (com/sem ack). | NFR-001, NFR-002 |
| `aios_bus_delivery_latency_ms` | histogram | `stream` | ms | Intervalo publish→deliver. | NFR-008 |
| `aios_bus_request_latency_ms` | histogram | `domain` | ms | Latência de request/reply. | NFR-003 |
| `aios_bus_request_timeouts_total` | counter | `domain` | 1 | Timeouts (`AIOS-BUS-0012`). | NFR-003 |

Buckets de `aios_bus_publish_latency_ms`: `[0.25, 0.5, 1, 2, 5, 10, 15, 25, 50, 100,
500]` — o bucket **2 ms** é a meta de NFR-001 e **15 ms** a de NFR-002.
Buckets de `aios_bus_delivery_latency_ms`: `[1, 2, 5, 10, 20, 50, 100, 500, 1000,
5000]`, com **20 ms** como alvo (NFR-008).

---

## 2. Validação e Governança de Namespace

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_publish_rejected_total` | counter | `reason` (unregistered\|envelope\|producer\|payload\|quota), `domain` | 1 | Publicações recusadas por motivo. | FR-002..FR-004 |
| `aios_bus_subjects_registered` | gauge | `domain` | 1 | Subjects ativos no registro. | NFR-007 |
| `aios_bus_subjects_deprecated` | gauge | `domain` | 1 | Subjects em período de coexistência. | — |
| `aios_bus_schema_cache_misses_total` | counter | — | 1 | Consultas ao registro de `dataschema` do `004-API`. | NFR-001 |

`aios_bus_publish_rejected_total{reason="producer"}` merece atenção especial: ele
sinaliza tentativa de publicar em subject de outro módulo — pode ser erro de
configuração, mas também é o sinal de uma tentativa de forjar evento (`./Security.md` §4).

---

## 3. Streams e Consumidores

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_stream_messages` | gauge | `stream` | 1 | Mensagens armazenadas no stream. | NFR-007 |
| `aios_bus_stream_bytes` | gauge | `stream` | bytes | Tamanho do stream (alerta perto de `max_bytes`). | — |
| `aios_bus_stream_limit_ratio` | gauge | `stream` | ratio | `bytes ÷ max_bytes`. | — |
| `aios_bus_consumer_pending` | gauge | `stream`, `consumer` | 1 | Mensagens aguardando entrega (lag). | NFR-008 |
| `aios_bus_consumer_ack_pending` | gauge | `stream`, `consumer` | 1 | Mensagens em voo, aguardando ack. | NFR-011 |
| `aios_bus_consumer_redelivered_total` | counter | `stream`, `consumer` | 1 | Reentregas realizadas. | NFR-012 |
| `aios_bus_consumers_stalled` | gauge | `stream` | 1 | Consumidores declarados parados. | NFR-011 |
| `aios_bus_ack_latency_ms` | histogram | `stream`, `consumer` | ms | Tempo entre entrega e ack (saúde do consumidor). | NFR-011 |

---

## 4. DLQ

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_deadletter_total` | counter | `stream`, `consumer` | 1 | Mensagens quarentenadas. | NFR-012 |
| `aios_bus_deadletter_pending` | gauge | `stream` | 1 | Entradas em `quarantined` aguardando decisão. | NFR-012 |
| `aios_bus_deadletter_replayed_total` | counter | `stream` | 1 | Replays autorizados. | FR-008 |
| `aios_bus_deadletter_discarded_total` | counter | `stream` | 1 | Descartes deliberados. | FR-008 |

---

## 5. Cotas e Contenção

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_quota_throttled_total` | counter | `scope` (tenant\|agent), `limit_kind` | 1 | Rejeições por cota (`AIOS-BUS-0007`). | NFR-011 |
| `aios_bus_quota_usage_ratio` | gauge | `scope`, `bucket` (faixa de uso) | ratio | Distribuição de consumo de cota, sem cardinalidade por tenant. | NFR-004 |
| `aios_bus_subscriptions_active` | gauge | `kind` (service\|agent) | 1 | Assinaturas ativas. | NFR-007 |
| `aios_bus_connections_active` | gauge | `kind` | 1 | Conexões de cliente ativas. | NFR-007 |

---

## 6. Comunicação Seletiva

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_group_broadcast_total` | counter | `scope` | 1 | Difusões solicitadas por produtores. | NFR-016 |
| `aios_bus_group_publish_total` | counter | `scope` | 1 | Publicações efetivas geradas por essas difusões. | NFR-016 |
| `aios_bus_group_fanout_size` | histogram | `scope` | 1 | Destinatários por difusão. | NFR-016 |
| `aios_bus_group_fanout_rejected_total` | counter | — | 1 | Difusões acima de `fanout_limit` (`AIOS-BUS-0008`). | FR-012 |

A razão `aios_bus_group_publish_total ÷ aios_bus_group_broadcast_total` **DEVE** ser
**1**: uma difusão custa uma publicação do produtor, qualquer que seja o tamanho do
grupo (NFR-016). Valor acima de 1 indica que alguém está iterando destinatários na
aplicação — exatamente o antipadrão que este módulo existe para evitar.

---

## 7. Sessões A2A

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_a2a_sessions_active` | gauge | `peer_kind` | 1 | Sessões em `Established`/`Suspended`. | NFR-015 |
| `aios_bus_a2a_sessions_total` | counter | `outcome` (closed\|rejected\|failed), `peer_kind` | 1 | Sessões por desfecho. | NFR-015 |
| `aios_bus_a2a_handshake_ms` | histogram | `peer_kind` | ms | Duração do handshake (T-01→T-04). | NFR-015 |
| `aios_bus_a2a_messages_total` | counter | `peer_kind` | 1 | Mensagens em canais de sessão. | NFR-015 |
| `aios_bus_a2a_suspended_total` | counter | `cause` (quota\|policy) | 1 | Suspensões (T-06). | FR-013 |

**Recording rule** usada por `./Monitoring.md` §3, para manter a expressão do alerta legível:

```yaml
- record: aios_bus_sessions_rejected
  expr: sum(rate(aios_bus_a2a_sessions_total{outcome="rejected"}[5m]))
```

---

## 8. Cluster e Plataforma

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_bus_cluster_nodes` | gauge | `state` (healthy\|unhealthy) | 1 | Nós do cluster NATS. | NFR-005 |
| `aios_bus_jetstream_quorum` | gauge | — | 0/1 | Quórum de JetStream disponível. | NFR-006 |
| `aios_bus_stream_replicas_healthy` | gauge | `stream` | 1 | Réplicas em dia por stream. | NFR-006 |
| `aios_bus_leaf_connections` | gauge | `region` | 1 | Leaf nodes conectados. | NFR-007 |
| `aios_bus_policy_decisions_total` | counter | `capability`, `decision` | 1 | Decisões do PDP para operações do barramento. | FR-017 |
| `aios_bus_outbox_pending` | gauge | — | 1 | Eventos próprios aguardando publicação. | FR-018 |
| `aios_bus_config_pending_reload` | gauge | — | 1 | Chaves não recarregáveis aguardando reinício. | — |

---

## 9. Exemplos de Consulta

```promql
# NFR-001 — p99 de publish core
histogram_quantile(0.99, sum by (le) (rate(aios_bus_publish_latency_ms_bucket{durable="false"}[15m]))) > 2

# NFR-012 — taxa de DLQ acima do limite
sum(rate(aios_bus_deadletter_total[1h])) / sum(rate(aios_bus_messages_total[1h])) > 0.0001

# NFR-016 — amplificação de fan-out (deve permanecer em 1)
sum(rate(aios_bus_group_publish_total[10m])) / sum(rate(aios_bus_group_broadcast_total[10m])) > 1

# Quórum de JetStream perdido
aios_bus_jetstream_quorum == 0

# Consumidor com lag crescente (backlog não drena)
deriv(aios_bus_consumer_pending[30m]) > 0 and aios_bus_consumer_pending > 10000

# Tentativas de publicar em subject alheio (sinal de segurança)
increase(aios_bus_publish_rejected_total{reason="producer"}[15m]) > 0
```

---

## 10. Referências

- NFR e metas: `./NonFunctionalRequirements.md`
- Alertas e dashboards: `./Monitoring.md` · Logs: `./Logging.md`
- Plataforma de observabilidade: `../024-Observability/`
