---
Documento: Monitoring
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0010 (global)
RFCs relacionados: RFC-0001 (§5.6)
Depende de: ../024-Observability/README.md, ./Metrics.md, ./NonFunctionalRequirements.md
---

# 004-API — Monitoramento

> Dashboards, alertas e *golden signals* de borda, derivados dos NFRs de
> `./NonFunctionalRequirements.md`. Metas numéricas são **idênticas** às do
> brief e de `./NonFunctionalRequirements.md` — qualquer divergência é
> defeito deste documento.

## 1. Golden Signals do Gateway

| Sinal | Métrica primária | Meta | NFR |
|-------|---------------------|------|-----|
| Latência | `aios_api_gateway_overhead_latency_ms` (p50/p99) | p99 ≤ 5 ms, p50 ≤ 1,5 ms | NFR-001 |
| Tráfego | `aios_api_requests_total` (taxa) | ≥ 150.000 req/s por réplica sob carga sustentada | NFR-002 |
| Erros | `aios_api_requests_total{code=~"5.."}` / total | Taxa de erro 5xx de borda tratada como incidente acima de 0,1% sustentado | NFR-003, NFR-012 |
| Saturação | `aios_api_ratelimit_rejections_total`, `aios_api_sse_active_streams` | Saturação de rate limit/SSE gera alerta antes do limite hard | NFR-004, NFR-007 |

## 2. Dashboards

### 2.1 Dashboard "Borda — Visão Geral"

- Taxa de requisições por rota/tenant (top-N).
- Latência de overhead do Gateway (p50/p95/p99) — `aios_api_gateway_overhead_latency_ms`.
- Distribuição de códigos de status HTTP e `AIOS-API-*`.
- Disponibilidade do endpoint de borda (uptime probe multi-região).

### 2.2 Dashboard "AuthN/AuthZ de Borda"

- Taxa de rejeição por `AIOS-API-0002` (AuthN) e `AIOS-API-0003` (AuthZ).
- Latência do `PolicyClient` (P50/p99) e taxa de *cache hit* de decisão.
- Estado do *circuit breaker* de `022-Policy`.
- TTL restante do cache JWKS por réplica.

### 2.3 Dashboard "Rate Limit e Idempotência"

- `aios_api_ratelimit_rejections_total` por tenant/rota.
- Taxa de *hit* de `IdempotencyRelay` (repetições servidas do cache vs. novas).
- Conflitos de reuse divergente (`AIOS-API-0012`) — indicador de possível bug de cliente.

### 2.4 Dashboard "Upstreams e Circuit Breaker"

- Estado do *circuit breaker* por *cluster* upstream (fechado/meio-aberto/aberto).
- Latência de upstream por serviço (`006`–`019`, `021`–`027`).
- Impacto de isolamento (p99 de rotas não afetadas durante falha de um upstream — valida NFR-012).

### 2.5 Dashboard "Registro de Contrato"

- `aios_api_contract_sync_lag_s` (deve permanecer ≤ 60 s — NFR-011).
- Transições de `ApiVersion` (contagem por estado, por módulo).
- Idade das versões `Deprecated` em relação a `deprecation_window_days`.

## 3. Alertas (regras Prometheus, resumo semântico)

| Alerta | Condição | Severidade | Ação/Runbook |
|--------|----------|-------------|----------------|
| `ApiGatewayHighOverheadLatency` | `aios_api_gateway_overhead_latency_ms{quantile="0.99"} > 5` por 5 min | Warning | Verificar cache de decisão do PDP e saúde do `RouteAuthorizer` — runbook `./Monitoring.md#runbooks`. |
| `ApiGatewayAvailabilityBreach` | Uptime probe < 99,99% na janela de 1h | Critical | Escalar SRE; verificar réplicas/AZ; ver `./FailureRecovery.md`. |
| `ApiGatewayAuthFailClosed` | `api.auth.fail_mode=closed` ativo E taxa de `AIOS-API-0002`/`0003` > baseline por 10 min | Critical | Verificar disponibilidade de `021-Security`/`022-Policy`. |
| `ApiGatewayRateLimitOvershoot` | Overshoot medido de token-bucket `hard` > 1% | Warning | Investigar corrida de concorrência no Redis Lua script — viola NFR-007. |
| `ApiGatewayContractSyncLagHigh` | `aios_api_contract_sync_lag_s > 60` por 5 min | Warning | Verificar `OpenApiAggregator` e consumo de `<module>.contract.published`. |
| `ApiGatewayCircuitBreakerOpen` | `CircuitBreakerManager` aberto para um *cluster* upstream por > 5 min | Warning | Investigar saúde do upstream específico; confirmar isolamento (NFR-012). |
| `ApiGatewaySseNearCapacity` | `aios_api_sse_active_streams` > 90% de `api.sse.max_streams_per_replica` | Warning | Avaliar autoscaling horizontal — `./Scalability.md`. |
| `ApiGatewayContractViolationSpike` | Taxa de `api.contract.violation` acima do baseline por 15 min | Warning | Investigar cliente com contrato desatualizado ou tentativa de abuso. |

## 4. SLO/SLA

| SLO | Definição | Janela | Error budget |
|-----|-----------|--------|-----------------|
| Disponibilidade de borda | ≥ 99,99% de requisições completadas sem erro 5xx atribuível ao Gateway | Mensal | ≈ 4,3 min/mês (NFR-003) |
| Latência de overhead | p99 ≤ 5 ms | Contínuo (janela deslizante de 5 min) | N/A (meta técnica) |
| Consistência de contrato agregado | Lag ≤ 60 s | Contínuo | N/A (meta técnica, NFR-011) |

O **SLA** externo (se publicado a parceiros) DEVE ser um subconjunto mais
conservador do SLO interno, nunca o contrário — decisão de produto fora do
escopo técnico deste documento.

## 5. Runbooks associados

| Cenário | Runbook (referência) |
|---------|--------------------------|
| Alta latência de overhead | `./FailureRecovery.md` §"Modo de falha: 022-Policy indisponível" |
| Disponibilidade abaixo do SLO | `./FailureRecovery.md` §"Crash de réplica do Gateway" |
| Rate limit *overshoot* | `./Testing.md` §"Teste concorrente de token-bucket" |
| *Circuit breaker* preso aberto | `./FailureRecovery.md` §"Upstream indisponível" |

*Fim de `Monitoring.md`.*
