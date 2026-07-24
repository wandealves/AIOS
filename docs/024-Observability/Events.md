---
Documento: Events
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0004, ADR-0010, ADR-0240, ADR-0242, ADR-0245
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §6, 020-Communication, 004-API (registro de domínios), 029-Operations
---

# 024-Observability — Catálogo de Eventos

> Envelope **CloudEvents** e convenção de subjects conforme
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 — **referenciados, não
> redefinidos**. Domínio `<dominio>` = **`telemetry`**, a ser registrado no registro de
> domínios mantido por `../004-API/` conforme RFC-0001 §8 (ratificação em ADR-0240).

## 1. Regra de conteúdo

**Nenhum evento deste módulo transporta amostra de telemetria, valor de atributo
classificado `pii`/`sensitive`, nem conteúdo de log.** Eventos carregam
identificadores, nomes de sinal, severidade, números agregados e horários.

A tentação de violar essa regra é maior em `telemetry.alert.firing`: seria conveniente
anexar as últimas linhas de log que motivaram o alerta. Seria também a forma mais
rápida de vazar, por um stream com 12 consumidores, exatamente o dado que a redação de
PII na borda existiu para proteger.

## 2. Eventos emitidos

| Subject | `type` | Quando | Stream | Consumidores conhecidos |
|---------|--------|--------|--------|-------------------------|
| `aios.<tenant>.telemetry.alert.firing` | `aios.telemetry.alert.firing` | Alerta → `Firing` (T-02). | `TELEMETRY_ALERT` | `029`, `025` |
| `aios.<tenant>.telemetry.alert.acknowledged` | `aios.telemetry.alert.acknowledged` | Alerta → `Acknowledged` (T-04). | `TELEMETRY_ALERT` | `029`, `025` |
| `aios.<tenant>.telemetry.alert.resolved` | `aios.telemetry.alert.resolved` | Alerta → `Resolved` (T-07). | `TELEMETRY_ALERT` | `029`, `025` |
| `aios.<tenant>.telemetry.alert.escalated` | `aios.telemetry.alert.escalated` | Escalonamento sem reconhecimento (T-08). | `TELEMETRY_ALERT` | `029` |
| `aios.<tenant>.telemetry.alert.nodata` | `aios.telemetry.alert.nodata` | Alerta → `Expired` por ausência de dado (T-09). | `TELEMETRY_ALERT` | `029`, `027` |
| `aios.<tenant>.telemetry.slo.breached` | `aios.telemetry.slo.breached` | SLO violado na janela de conformidade. | `TELEMETRY_SLO` | `025`, `029`, `032` |
| `aios.<tenant>.telemetry.budget.exhausted` | `aios.telemetry.budget.exhausted` | Error budget acima do limiar da política. | `TELEMETRY_SLO` | `028`, `022`, `029` |
| `aios.<tenant>.telemetry.cardinality.exceeded` | `aios.telemetry.cardinality.exceeded` | Orçamento de séries estourado; sinal quarentenado. | `TELEMETRY_PLATFORM` | `029`, módulo dono |
| `aios._platform.telemetry.signal.registered` | `aios.telemetry.signal.registered` | Novo `SignalDescriptor` admitido. | `TELEMETRY_PLATFORM` | `032`, `033` |
| `aios._platform.telemetry.signal.quarantined` | `aios.telemetry.signal.quarantined` | Sinal quarentenado (cardinalidade ou convenção). | `TELEMETRY_PLATFORM` | `029`, módulo dono |
| `aios.<tenant>.telemetry.ingest.degraded` | `aios.telemetry.ingest.degraded` | Descarte acima de `obs.ingest.drop_alert_ratio`. | `TELEMETRY_PLATFORM` | `029` |
| `aios._platform.telemetry.dashboard.published` | `aios.telemetry.dashboard.published` | Dashboard publicado/versionado. | `TELEMETRY_PLATFORM` | `032` |

## 3. Schemas (payload `data`)

Envelope completo conforme RFC-0001 §5.2; abaixo apenas o campo `data`.

### 3.1 `aios.telemetry.alert.firing`

```json
{
  "specversion": "1.0",
  "id": "01J9ZF00000000000000000000",
  "source": "urn:aios:acme:service:observability",
  "type": "aios.telemetry.alert.firing",
  "subject": "urn:aios:acme:event:01J9ZG00000000000000000000",
  "time": "2026-07-23T10:20:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/telemetry.alert.firing/1",
  "data": {
    "alertId": "01J9ZG00000000000000000000",
    "ruleKey": "KernelSyscallLatencyHigh",
    "severity": "critical",
    "fingerprint": "sha256:7a1c…",
    "labels": { "service": "kernel", "shard": "12" },
    "value": 84.2,
    "sloKey": "kernel.syscall.latency",
    "runbook": "runbooks/RB-K-03.md",
    "startedAt": "2026-07-23T10:15:00.000Z"
  }
}
```

`runbook` é campo obrigatório do payload: a notificação que chega ao plantonista
carrega o procedimento, não apenas o problema (ADR-0247).

### 3.2 `aios.telemetry.alert.nodata`

```json
{ "data": {
    "alertId": "01J9ZH00000000000000000000",
    "ruleKey": "KernelSyscallLatencyHigh",
    "fingerprint": "sha256:7a1c…",
    "lastSeenAt": "2026-07-23T10:05:00.000Z",
    "noDataForSeconds": 600,
    "previousState": "Firing" } }
```

`previousState` importa: um alerta que estava `Firing` e parou de reportar é
**suspeito**, não resolvido — o alvo que caiu deixou de emitir exatamente a métrica
que provaria a queda (invariante I6).

### 3.3 `aios.telemetry.budget.exhausted`

```json
{ "data": {
    "sloKey": "kernel.syscall.latency", "sloVersion": 3,
    "ownerModule": "006-Kernel",
    "objective": 0.999, "sliValue": 0.9962,
    "budgetConsumed": 0.62, "burnRate1h": 8.1, "burnRate6h": 4.2,
    "windowDays": 30,
    "policyAction": "freeze_non_urgent_releases",
    "evaluatedAt": "2026-07-23T10:20:00.000Z" } }
```

Consumido por `../028-Deployment/` (congela releases não urgentes) e por
`../022-Policy/` (adia publicações de bundle não urgentes). É o único ponto do AIOS em
que uma medição, e não uma decisão humana, altera o ritmo de mudança do sistema.

### 3.4 `aios.telemetry.cardinality.exceeded` e `aios.telemetry.signal.quarantined`

```json
{ "data": {
    "signalName": "aios_kernel_syscall_latency_ms",
    "ownerModule": "006-Kernel",
    "scope": "tenant", "scopeRef": "acme",
    "currentSeries": 1020431, "maxSeries": 1000000,
    "offendingLabels": ["agent_urn"],
    "action": "quarantine",
    "detectedAt": "2026-07-23T10:20:00.000Z" } }
```

`offendingLabels` é o dado que transforma o evento em ação: o dono do sinal sabe
**qual label** removeu do descritor, não apenas que "estourou".

### 3.5 `aios.telemetry.ingest.degraded`

```json
{ "data": {
    "receivedPerSecond": 2140000, "droppedPerSecond": 41200,
    "dropRatio": 0.0192, "threshold": 0.01,
    "reason": "queue_full",
    "affectedSignalKinds": ["metric","span"],
    "observedFrom": "2026-07-23T10:18:00.000Z" } }
```

> Este evento existe porque o *fail-open* (ADR-0243) tem um custo: a perda é
> silenciosa por construção. `ingest.degraded` é o que a torna audível — sem ele, uma
> decisão seria tomada sobre telemetria incompleta sem ninguém saber que estava
> incompleta.

### 3.6 `aios.telemetry.signal.registered`

```json
{ "data": {
    "signalName": "aios_kernel_syscall_latency_ms",
    "descriptorVersion": 1, "signalKind": "metric", "metricType": "histogram",
    "unit": "ms", "ownerModule": "006-Kernel",
    "allowedLabels": ["verb","component","tenant"],
    "retentionTier": "raw", "dataClass": "operational" } }
```

## 4. Eventos consumidos

| Subject assinado | Produtor | Ação do módulo |
|-------------------|----------|----------------|
| `aios.<tenant>.policy.bundle.updated` | `../022-Policy/` | Invalida cache do `PolicyClient`; **anota** a mudança nas linhas do tempo dos dashboards (FR-024). |
| `aios.<tenant>.policy.decision.updated` | `../022-Policy/` | Invalidação granular de decisão de consulta. |
| `aios._platform.deployment.rollout.started` | `../028-Deployment/` | Anota o rollout nos dashboards e ativa janela de supressão de alertas esperados. |
| `aios._platform.cluster.node.decommissioned` | `../027-Cluster/` | Remove o alvo de coleta; evita alerta `nodata` espúrio (UC-011 A2). |
| `aios.<tenant>.security.rtbf.requested` | `../021-Security/` | Dispara expurgo/pseudonimização da telemetria do titular (UC-015). |
| `aios._platform.security.jwks.rotated` | `../021-Security/` | Recarrega material de verificação usado no `QueryGateway`. |
| `aios.<tenant>.cost.budget.exhausted` | `../026-Cost-Optimizer/` | Correlaciona custo e carga; anota o evento nos dashboards. |

Todo consumo é **idempotente** por `event.id` (RFC-0001 §5.5). Um
`node.decommissioned` reprocessado não pode reintroduzir um alvo já removido.

## 5. Semântica de entrega e streams

| Stream | Subjects | Retenção | Réplicas | Semântica |
|--------|----------|----------|----------|-----------|
| `TELEMETRY_ALERT` | `aios.*.telemetry.alert.*` | 400 dias | 3 | at-least-once; base da análise de recorrência |
| `TELEMETRY_SLO` | `aios.*.telemetry.slo.*`, `aios.*.telemetry.budget.*` | 400 dias | 3 | at-least-once; conformidade anual |
| `TELEMETRY_PLATFORM` | `aios.*.telemetry.{signal,cardinality,ingest,dashboard}.*` | 90 dias | 3 | at-least-once |

- Publicação via **Outbox transacional** (`telemetry.outbox`, `./Database.md` §9): a
  transição de alerta e o evento são gravados na mesma transação (invariante I5) e
  publicados pelo `observability-outbox-relay`.
- Consumidores **DEVEM** deduplicar por `event.id` e **DEVEM** tolerar campos
  desconhecidos (evolução aditiva, RFC-0001 §5.7).
- A **telemetria em si não trafega por NATS**: métricas, traces e logs usam OTLP
  direto ao coletor. Publicar telemetria bruta no barramento faria o volume de
  observabilidade competir com o tráfego de negócio no mesmo recurso.

## 6. Ordenação e reprocessamento

```
   alert.firing ──▶ alert.acknowledged ──▶ alert.resolved
        │                                        │
        └──── ordem por subject dentro do stream TELEMETRY_ALERT ────┘

   Consumidor que processe fora de ordem DEVE usar `alertId` + `startedAt`
   como chave de estado, e IGNORAR evento de instância já encerrada.
   Uma recorrência gera NOVA instância (novo alertId) — nunca reabre a anterior.
```

> A confusão perigosa aqui é tratar `alert.firing` de uma **nova** instância como
> reabertura da anterior. O `alertId` distinto é o que preserva o histórico de
> recorrência — que é, na análise pós-incidente, mais informativo que a duração de
> qualquer ocorrência isolada (invariante I4).

## 7. Versionamento de schema

| `dataschema` | Versão | Mudanças permitidas |
|--------------|--------|---------------------|
| `aios://schemas/telemetry.alert.firing/1` | 1 | Aditivas (novos campos opcionais) |
| `aios://schemas/telemetry.alert.nodata/1` | 1 | Aditivas |
| `aios://schemas/telemetry.budget.exhausted/1` | 1 | Novos `policyAction` (aditivos) |
| `aios://schemas/telemetry.cardinality.exceeded/1` | 1 | Aditivas |
| `aios://schemas/telemetry.ingest.degraded/1` | 1 | Novos `reason` (aditivos) |
| `aios://schemas/telemetry.signal.registered/1` | 1 | Aditivas |

Mudança incompatível exige nova versão de schema e coexistência (RFC-0001 §5.7);
registro em `../004-API/Events.md` (RFC-0001 §8).

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §6
- Envelope e subjects: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3
- Barramento e streams: `../020-Communication/`
- Registro de domínios e schemas: `../004-API/Events.md`
- Outbox: `./Database.md` §9 · FSM: `./StateMachine.md`

*Fim de `Events.md`.*
