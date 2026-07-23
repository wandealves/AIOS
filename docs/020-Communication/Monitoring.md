---
Documento: Monitoring
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0205, ADR-0206
RFCs relacionados: RFC-0001
Depende de: Metrics.md, NonFunctionalRequirements.md, 024-Observability, 029-Operations
---

# 020-Communication — Monitoramento

Dashboards e alertas do barramento. Todos os limiares derivam dos SLO de
`./NonFunctionalRequirements.md` — divergência é defeito.

---

## 1. Golden Signals do Módulo

| Sinal | Métrica principal | Meta |
|-------|-------------------|------|
| **Latência** | `aios_bus_publish_latency_ms`, `aios_bus_delivery_latency_ms` | p99 ≤ 2 ms (core) · ≤ 15 ms (durável) · ≤ 20 ms (entrega) |
| **Tráfego** | `aios_bus_messages_total` | ≥ 1M msg/s core sustentáveis |
| **Erros** | `aios_bus_publish_rejected_total`, `aios_bus_deadletter_total` | DLQ ≤ 0,01% das entregas |
| **Saturação** | `aios_bus_consumer_ack_pending`, `aios_bus_stream_limit_ratio` | em voo < `max_ack_pending` · stream < 80% do teto |

Sinais adicionais, específicos deste módulo por serem **obrigações** e não desempenho:
**quórum de JetStream**, **amplificação de fan-out** e **rejeições por produtor não
autorizado**.

---

## 2. Dashboards

### 2.1 `BUS / Visão Geral`
- Mensagens/s por domínio e direção; bytes/s.
- p99 de publish (core × durável) e de entrega, com linhas tracejadas nas metas.
- Rejeições de publicação por motivo (`unregistered`, `envelope`, `producer`,
  `payload`, `quota`) — um painel que responde "por que a mensagem não passou?".
- Conexões e assinaturas ativas por tipo (serviço × agente).

### 2.2 `BUS / Streams e Consumidores`
- `pending` e `ack_pending` por consumidor, ordenados por lag.
- `aios_bus_stream_limit_ratio` por stream, com alerta em 80%.
- Reentregas por consumidor — identifica quem está falhando antes de virar DLQ.
- p99 de `ack_latency_ms` por consumidor (saúde do processamento, não do barramento).

### 2.3 `BUS / DLQ`
- Entradas quarentenadas por stream/consumidor nas últimas 24 h.
- Top padrões de `last_error` (base para decidir replay, UC-007).
- Replays e descartes com autor e justificativa.

### 2.4 `BUS / Cluster`
- Nós saudáveis, quórum de JetStream, réplicas em dia por stream.
- Leaf nodes conectados por região.
- Latência entre nós (indicador precoce de degradação de replicação).

### 2.5 `BUS / A2A e Grupos`
- Sessões ativas por tipo de par; p99 de handshake.
- Sessões por desfecho (`closed`/`rejected`/`failed`) — muitas `rejected` indicam
  política desalinhada com o que os agentes tentam fazer.
- Amplificação de fan-out (deve ser exatamente 1) e histograma de tamanho de difusão.

### 2.6 `BUS / Cotas`
- Rejeições por cota, por escopo e tipo de limite.
- Distribuição de uso de cota (faixas), identificando *hot tenants* sem expor
  cardinalidade por tenant.

---

## 3. Regras de Alerta (Prometheus)

```yaml
groups:
- name: aios-bus-critical
  rules:
  - alert: BusJetStreamQuorumLost
    expr: aios_bus_jetstream_quorum == 0
    for: 30s
    labels: { severity: P1, module: "020-Communication" }
    annotations:
      summary: "Quórum de JetStream perdido — publicação durável falhando (AIOS-BUS-0011)"
      description: "Produtores retêm no Outbox; nenhuma perda esperada, mas o fluxo parou."
      runbook: "../029-Operations/#bus-quorum-lost"

  - alert: BusClusterDegraded
    expr: aios_bus_cluster_nodes{state="healthy"} < 3
    for: 2m
    labels: { severity: P1 }
    annotations:
      summary: "Menos de 3 nós NATS saudáveis"
      runbook: "../029-Operations/#bus-node-down"

  - alert: BusStreamNearLimit
    expr: aios_bus_stream_limit_ratio > 0.9
    for: 5m
    labels: { severity: P1 }
    annotations:
      summary: "Stream acima de 90% do max_bytes — risco de descarte (AIOS-BUS-0013)"
      runbook: "../029-Operations/#bus-stream-full"

  - alert: BusUnauthorizedProducer
    expr: increase(aios_bus_publish_rejected_total{reason="producer"}[15m]) > 0
    for: 1m
    labels: { severity: P1 }
    annotations:
      summary: "Tentativa de publicar em subject de outro módulo (AIOS-BUS-0010)"
      description: "Pode ser erro de configuração — ou tentativa de forjar evento."
      runbook: "../029-Operations/#bus-unauthorized-producer"

- name: aios-bus-warning
  rules:
  - alert: BusPublishLatencyHigh
    expr: histogram_quantile(0.99, sum by (le) (rate(aios_bus_publish_latency_ms_bucket{durable="false"}[10m]))) > 2
    for: 10m
    labels: { severity: P2 }
    annotations: { summary: "p99 de publish core acima de 2 ms (NFR-001)" }

  - alert: BusDeliveryLatencyHigh
    expr: histogram_quantile(0.99, sum by (le) (rate(aios_bus_delivery_latency_ms_bucket[10m]))) > 20
    for: 10m
    labels: { severity: P2 }
    annotations: { summary: "p99 de entrega acima de 20 ms (NFR-008)" }

  - alert: BusDeadLetterRateHigh
    expr: sum(rate(aios_bus_deadletter_total[1h])) / sum(rate(aios_bus_messages_total[1h])) > 0.0001
    for: 15m
    labels: { severity: P2 }
    annotations: { summary: "Taxa de DLQ acima de 0,01% (NFR-012)" }

  - alert: BusConsumerStalled
    expr: aios_bus_consumers_stalled > 0
    for: 5m
    labels: { severity: P2 }
    annotations:
      summary: "Consumidor parado — backlog acumulando"
      runbook: "../029-Operations/#bus-consumer-stalled"

  - alert: BusConsumerLagGrowing
    expr: deriv(aios_bus_consumer_pending[30m]) > 0 and aios_bus_consumer_pending > 10000
    for: 30m
    labels: { severity: P2 }
    annotations: { summary: "Lag crescente: consumidor não drena o backlog" }

  - alert: BusFanoutAmplification
    expr: sum(rate(aios_bus_group_publish_total[10m])) / sum(rate(aios_bus_group_broadcast_total[10m])) > 1.1
    for: 15m
    labels: { severity: P2 }
    annotations:
      summary: "Amplificação de fan-out > 1 (NFR-016)"
      description: "Alguém está iterando destinatários na aplicação em vez de usar canal de grupo."

  - alert: BusQuotaThrottlingSustained
    expr: sum(rate(aios_bus_quota_throttled_total[15m])) > 100
    for: 15m
    labels: { severity: P2 }
    annotations: { summary: "Rejeição por cota sustentada — revisar produtor ou orçamento (026)" }

  - alert: BusA2ARejectionRateHigh
    expr: sum(rate(aios_bus_sessions_rejected[1h])) / sum(rate(aios_bus_a2a_sessions_total[1h])) > 0.2
    for: 30m
    labels: { severity: P3 }
    annotations: { summary: "Mais de 20% das sessões A2A recusadas — política desalinhada?" }

  - alert: BusOutboxBacklog
    expr: aios_bus_outbox_pending > 1000
    for: 5m
    labels: { severity: P2 }
    annotations: { summary: "Eventos de plataforma do próprio barramento não publicados" }
```

> `aios_bus_sessions_rejected` é derivada de
> `aios_bus_a2a_sessions_total{outcome="rejected"}` por *recording rule*, para manter a
> expressão do alerta legível.

---

## 4. SLO, SLI e Error Budget

| SLO | SLI | Janela | Budget |
|-----|-----|--------|--------|
| Disponibilidade ≥ 99,95% (NFR-005) | fração de minutos com publicação aceita | 30 dias | 21,9 min |
| p99 publish core ≤ 2 ms (NFR-001) | fração de janelas de 5 min dentro da meta | 30 dias | 1% das janelas |
| Perda zero de mensagem persistida (NFR-006) | contagem publicada × consumida em chaos test | contínuo | **0 violações** |
| Isolamento (NFR-010) | violações detectadas em pentest/auditoria | contínuo | **0 violações** |
| DLQ ≤ 0,01% (NFR-012) | razão DLQ/entregues | 30 dias | 0,01% |

Ao esgotar o budget de disponibilidade, mudanças de topologia (leaf nodes, novos
streams não críticos) são **congeladas** até a recomposição.

---

## 5. Runbooks Associados

| Alerta | Runbook (`../029-Operations/`) | Primeira ação |
|--------|-------------------------------|---------------|
| `BusJetStreamQuorumLost` | `#bus-quorum-lost` | Verificar quantos nós estão fora; restaurar o mínimo para quórum antes de qualquer outra ação. Produtores estão retendo no Outbox — não há perda, mas há atraso. |
| `BusClusterDegraded` | `#bus-node-down` | Confirmar com `../027-Cluster/`; reingressar o nó; verificar `stream_replicas_healthy` antes de qualquer manutenção adicional. |
| `BusStreamNearLimit` | `#bus-stream-full` | Verificar `discard`: com `old`, há perda silenciosa de histórico; com `new`, publicações falham. Aumentar `max_bytes` ou reduzir `max_age`, e investigar o produtor. |
| `BusUnauthorizedProducer` | `#bus-unauthorized-producer` | Identificar módulo e subject; se não for erro de configuração, tratar como incidente de segurança (`./Security.md` §4). |
| `BusConsumerStalled` | `#bus-consumer-stalled` | Identificar o consumidor; verificar se está travado em uma mensagem específica (candidata a `term()`); escalar réplicas se for volume. |
| `BusDeadLetterRateHigh` | `#bus-dlq-triage` | Agrupar por `last_error` (consulta em `./Database.md` §10); corrigir a causa **antes** de qualquer replay. |
| `BusFanoutAmplification` | `#bus-fanout` | Identificar o produtor; migrar de iteração de destinatários para canal de grupo (UC-013). |
| `BusQuotaThrottlingSustained` | `#bus-quota` | Distinguir abuso de crescimento legítimo; ajuste permanente vem do `../026-Cost-Optimizer/`, não de edição manual. |

---

## 6. Observação de Longo Prazo

Tendências que não disparam página, mas orientam capacidade e arquitetura:

- Crescimento de `aios_bus_subjects_registered` por domínio — explosão de cardinalidade
  costuma começar como "só mais um subject".
- Evolução de `aios_bus_stream_bytes` por stream (projeção de disco a 90 dias).
- Distribuição de `aios_bus_group_fanout_size` — grupos que crescem além de algumas
  centenas geralmente pedem subdivisão.
- Razão entre tráfego `control` e `telemetry`: telemetria dominando o barramento indica
  que ela deveria ir por caminho próprio, com amostragem.
- Sessões A2A por agente ao longo do tempo — crescimento sustentado pode indicar que
  uma colaboração recorrente deveria virar um canal de grupo.

---

## 7. Referências

- Métricas: `./Metrics.md` · SLO: `./NonFunctionalRequirements.md`
- Logs: `./Logging.md` · Falhas: `./FailureRecovery.md`
- Runbooks: `../029-Operations/` · Observabilidade: `../024-Observability/`
