---
Documento: Examples
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0244, ADR-0245, ADR-0247
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: API.md, UseCases.md, Configuration.md
---

# 024-Observability — Exemplos

> Exemplos usam o tenant `acme` e os endpoints reais de `./API.md`. O CLI é o do
> `../030-CLI/`; o SDK, o do `../031-SDK/`.

## 1. Hello world — instrumentar um serviço

```csharp
// Serviço .NET do control plane (ex.: 006-Kernel)
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r
        .AddService(serviceName: "kernel", serviceVersion: "1.4.2")
        .AddAttributes(new Dictionary<string, object> {
            ["aios.tenant"] = tenantId,                    // OBRIGATÓRIO (API.md §2.2)
            ["deployment.environment"] = "production" }))
    .WithMetrics(m => m
        .AddMeter("Aios.Kernel")
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://otel-agent:4317")))
    .WithTracing(t => t
        .AddSource("Aios.Kernel")
        .SetSampler(new TraceIdRatioBasedSampler(0.1))     // head sampling (10%)
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://otel-agent:4317")));
```

O export vai ao **`otel-agent` local do nó** (~1 ms), nunca direto ao gateway central:
é o que mantém o overhead no observado dentro de NFR-002.

## 2. Registrar o sinal antes de emiti-lo

Emitir uma métrica não registrada resulta em quarentena (ADR-0241). O registro é parte
do pipeline de CI do módulo dono:

```bash
aios obs signal register \
  --name aios_kernel_syscall_latency_ms \
  --kind metric --metric-type histogram --unit ms \
  --owner 006-Kernel \
  --allowed-labels verb,component,tenant \
  --forbidden-labels agent_urn,task_urn \
  --buckets 5,20,50,250,1000 \
  --data-class operational --retention-tier raw
# → urn:aios:_platform:event:01J9ZE00000000000000000000  state=Registered
```

Três decisões visíveis no comando:

- `allowed-labels` é **lista fechada**: `agent_urn` não está lá, e por isso 10⁶ agentes
  não viram 10⁶ séries.
- `buckets` alinham-se aos limiares de SLO do Kernel; buckets automáticos não caem nas
  fronteiras do objetivo e `histogram_quantile` sobre eles devolve um número que
  **parece** um p99.
- `retention-tier raw` significa 15 dias em resolução nativa, depois agregação.

## 3. O que acontece com um sinal desconhecido

```bash
# Um serviço emite uma métrica de depuração esquecida no código
curl -s localhost:8888/metrics | grep kernel_temp_debug
# kernel_temp_debug_counter 42

aios obs signal list --state Quarantined
```

```
NAME                        OWNER        REASON        SINCE
kernel_temp_debug_counter   006-Kernel   unregistered  2026-07-23T10:14:00Z
```

O lote que continha essa métrica **não foi rejeitado** — as métricas registradas do
mesmo serviço seguiram normalmente (FR-003). Rejeitar o lote inteiro faria uma métrica
esquecida apagar toda a telemetria daquele serviço.

## 4. Explosão de cardinalidade, na prática

```bash
aios obs cardinality status --tenant acme
```

```
SCOPE    REF                            SERIES      MAX         RATIO
tenant   *                              1.021.430   1.000.000   1.02  ⚠ HARD
signal   aios_kernel_syscall_latency_ms   998.211      10.000   99.82 ⚠ HARD
signal   aios_pol_decisions_total           4.120      10.000    0.41
```

```bash
aios obs signal describe aios_kernel_syscall_latency_ms --show-offending
# offendingLabels: ["agent_urn"]   ← introduzido no deploy de 2026-07-23T09:40Z
```

O sinal já está quarentenado automaticamente. A correção é nova versão do descritor —
não elevar o teto:

```bash
aios obs signal register --name aios_kernel_syscall_latency_ms \
  --descriptor-version 2 --allowed-labels verb,component,tenant   # sem agent_urn
aios obs signal reinstate aios_kernel_syscall_latency_ms
```

> Elevar `max_series_per_tenant` ao receber o alerta adia o problema e fixa o custo
> mais alto permanentemente (`./Monitoring.md` RB-O-03).

## 5. Definir um SLO e observar o error budget

```bash
aios obs slo put \
  --key kernel.syscall.latency --owner 006-Kernel \
  --sli 'sum(rate(aios_kernel_syscall_latency_ms_bucket{le="50"}[5m])) / sum(rate(aios_kernel_syscall_latency_ms_count[5m]))' \
  --objective 0.999 --window 30d \
  --budget-policy '{"at":0.5,"action":"freeze_non_urgent_releases"}'
```

```bash
aios obs slo status kernel.syscall.latency
```

```json
{ "sloKey": "kernel.syscall.latency", "sloVersion": 3,
  "objective": 0.999, "sliValue": 0.9962,
  "budgetConsumed": 0.62, "burnRate1h": 8.1, "burnRate6h": 4.2,
  "breached": false, "windowDays": 30,
  "evaluatedAt": "2026-07-23T10:20:00Z" }
```

Com `budgetConsumed = 0.62` e a política em `0.5`, o evento
`telemetry.budget.exhausted` já foi emitido: `../028-Deployment/` congelou releases
não urgentes e `../022-Policy/` adiou publicações de bundle não urgentes. É o único
ponto do AIOS em que uma **medição**, e não uma decisão humana, altera o ritmo de
mudança do sistema.

## 6. Regra de alerta — com runbook obrigatório

```bash
# 1. O runbook PRIMEIRO — sem ele a regra é rejeitada (AIOS-OBS-0008)
aios obs runbook put --key RB-K-03 \
  --path runbooks/RB-K-03-kernel-syscall-latency.md \
  --owner 006-Kernel --steps 6

# 2. A regra, derivada do SLO, com multi-burn-rate
aios obs alert-rule put --key KernelSyscallLatencyHigh \
  --expr 'aios_obs_slo_burn_rate{slo_key="kernel.syscall.latency",window="1h"} > 14.4
          and aios_obs_slo_burn_rate{slo_key="kernel.syscall.latency",window="6h"} > 6' \
  --for 5m --severity critical \
  --slo kernel.syscall.latency \
  --runbook RB-K-03 \
  --notify pagerduty-sre,slack-kernel
```

Tentar publicar sem runbook:

```json
HTTP/1.1 422 Unprocessable Entity
{ "code": "AIOS-OBS-0008",
  "detail": "Regra de alerta exige runbook_urn válido.", "retriable": false }
```

## 7. Ciclo de vida de um alerta

```bash
aios obs alert list --state Firing
```

```
ID (abreviado)   RULE                        SEV       AGE   RUNBOOK
01J9ZG…5J4K      KernelSyscallLatencyHigh    critical  6m    RB-K-03
```

```bash
# Assumir (cancela o escalonamento)
aios obs alert ack 01J9ZG00000000000000000000

# Silenciar durante a manutenção — prazo e justificativa obrigatórios
aios obs silence create \
  --matcher 'rule_key="KernelSyscallLatencyHigh",shard="12"' \
  --ttl 4h \
  --reason "Manutenção planejada do shard 12, CHG-4821"
```

Durante o silêncio, a **avaliação continua** (invariante I3): se a condição cessar, o
alerta vai a `Resolved` normalmente. O silêncio suprime a notificação, não a
observação.

```bash
aios obs silence create --matcher 'rule_key="X"' --reason "investigando"
# → 403 AIOS-OBS-0013: silêncio exige expires_at (máximo 72 h)
```

## 8. Investigar: do alerta ao log ao trace

```bash
# 1. Consultar a métrica que disparou
aios obs query metrics \
  --query 'histogram_quantile(0.99, sum by (le,verb) (rate(aios_kernel_syscall_latency_ms_bucket[5m])))' \
  --from -30m
```

```bash
# 2. Achar os logs correlacionados do período
aios obs query logs \
  --filter "service = '006-kernel' and level = 'Warning'" \
  --from -30m --limit 20
```

```json
{ "entries": [ { "timestamp": "2026-07-23T10:12:03.441Z",
                 "service": "006-kernel", "event": "kernel.syscall.slow",
                 "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
                 "verb": "invoke_tool", "latencyMs": 842 } ] }
```

```bash
# 3. Pivotar para o trace completo — um salto, não uma correlação por horário
aios obs query trace 4bf92f3577b34da6a3ce929d0e0e4736
```

O `traceId` do log é o mesmo do trace (RFC-0001 §5.6). Essa é a razão de a correlação
ser obrigatória em todo sinal: sem ela, a investigação vira arqueologia por timestamp.

## 9. Publicar um dashboard como código

```bash
aios obs dashboard put --key kernel-golden-signals \
  --owner 006-Kernel \
  --signals aios_kernel_syscall_latency_ms,aios_kernel_syscalls_total \
  --file dashboards/kernel-golden-signals.json
```

```json
HTTP/1.1 422 Unprocessable Entity
{ "code": "AIOS-OBS-0009",
  "detail": "signals_used contém 'aios_kernel_debug_gauge' não registrado.",
  "retriable": false }
```

Dashboard que referencia sinal não registrado é rejeitado: um painel construído sobre
uma métrica que pode ser quarentenada a qualquer momento é um painel que vai ficar
vazio no pior momento possível.

## 10. Verificar a não-interferência (o teste que importa)

```bash
# Terminal 1 — carga nominal e medição do p99 do EMISSOR
k6 run --vus 200 emitter-latency.js
# p99 do emissor: 12,4 ms

# Terminal 2 — saturar o gateway com 5× a carga de telemetria
aios obs bench saturate --factor 5 --duration 2m

# Terminal 1 — medir novamente
# p99 do emissor: 12,5 ms   ← INALTERADO (dentro do ruído)

aios obs metrics get aios_obs_dropped_total
# aios_obs_dropped_total{signal_kind="metric",reason="queue_full"} 4.812.030
```

A perda é real, é medida e é anunciada por `telemetry.ingest.degraded` — mas é do
observador, nunca do observado (ADR-0243). Se o p99 do emissor subisse, o *fail-open*
não estaria funcionando (`./Benchmark.md` §4.2).

## 11. Direito ao esquecimento sobre telemetria

```bash
# Disparado por evento do 021-Security; consulta do comprovante:
aios obs erasure status --receipt 01J9ZJ00000000000000000000
```

```json
{ "receiptId": "01J9ZJ00000000000000000000",
  "subjectRef": "tok:9f2c…",
  "storesAffected": [
    { "store": "seq",        "action": "purged",        "count": 1842 },
    { "store": "tempo",      "action": "purged",        "count": 96 },
    { "store": "prometheus", "action": "no_occurrence", "count": 0 } ],
  "aggregatesPreserved": true,
  "auditRef": "urn:aios:acme:audit:01J9ZK00000000000000000000",
  "completedAt": "2026-07-23T11:02:00Z" }
```

`prometheus: no_occurrence` é o resultado esperado e correto: métricas agregadas não
contêm identificação, logo não há dado pessoal a esquecer nelas.

## 12. Referências

- Contratos: `./API.md` · Eventos: `./Events.md`
- Casos de uso: `./UseCases.md` · Configuração: `./Configuration.md`
- CLI e SDK: `../030-CLI/`, `../031-SDK/`
- FAQ: `./FAQ.md`

*Fim de `Examples.md`.*
