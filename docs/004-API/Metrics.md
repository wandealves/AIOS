---
Documento: Metrics
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0010 (global)
RFCs relacionados: RFC-0001 (§5.6)
Depende de: ../024-Observability/README.md, ./NonFunctionalRequirements.md, ./Monitoring.md
---

# 004-API — Catálogo de Métricas

> Todas as métricas seguem a convenção global `aios_<subsistema>_<nome>_<unidade>`
> (subsistema = `api`), instrumentadas via OpenTelemetry e expostas em
> formato Prometheus. Valores-alvo são **idênticos** aos de
> `./NonFunctionalRequirements.md` e `./Monitoring.md`.

## 1. Métricas de latência e tráfego

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_api_gateway_overhead_latency_ms` | histogram | `route_id`, `method`, `api_major` | ms | Latência do pipeline do Gateway, excluindo o tempo de resposta do upstream. Meta: p99 ≤ 5 ms (NFR-001). |
| `aios_api_requests_total` | counter | `route_id`, `method`, `status_code`, `tenant_id` | requisições | Total de requisições processadas pelo Gateway. |
| `aios_api_upstream_latency_ms` | histogram | `upstream_cluster`, `route_id` | ms | Latência observada da chamada Gateway→upstream (não confundir com NFR-001). |
| `aios_api_request_body_bytes` | histogram | `route_id` | bytes | Tamanho do corpo de requisição, para dimensionar `max_body_size_bytes`. |

## 2. Métricas de AuthN/AuthZ

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_api_authn_failures_total` | counter | `reason` (expired\|invalid_signature\|missing) | falhas | Total de falhas de validação de JWT (base de `AIOS-API-0002`). |
| `aios_api_authz_decisions_total` | counter | `decision` (allow\|deny), `route_id` | decisões | Total de decisões do `RouteAuthorizer`. |
| `aios_api_policy_decision_cache_hit_ratio` | gauge | — | razão (0–1) | Fração de decisões servidas do cache do `PolicyClient`. |
| `aios_api_jwks_cache_age_s` | gauge | — | s | Idade do cache JWKS local em relação ao último `refresh`. |

## 3. Métricas de rate limit e idempotência

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_api_ratelimit_rejections_total` | counter | `tenant_id`, `route_id`, `enforcement` | rejeições | Total de requisições rejeitadas por `AIOS-API-0004`. |
| `aios_api_ratelimit_overshoot_ratio` | gauge | `tenant_id` | razão (0–1) | Fração de excesso observado sobre o limite `hard`. Meta: ≤ 1% (NFR-007). |
| `aios_api_idempotency_cache_hit_total` | counter | `route_id` | acertos | Repetições servidas do cache de idempotência sem chamar o upstream. |
| `aios_api_idempotency_conflicts_total` | counter | `route_id` | conflitos | Total de `AIOS-API-0012` (reuse com payload divergente). |

## 4. Métricas de contrato e versionamento

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_api_contract_sync_lag_s` | gauge | `module_owner` | s | Tempo desde a última publicação de contrato upstream até a reagregação. Meta: ≤ 60 s (NFR-011). |
| `aios_api_version_state_total` | gauge | `module_owner`, `state` | contagem | Nº de majors por estado da FSM (`Draft`/`Published`/`Deprecated`/`Retired`). |
| `aios_api_deprecated_requests_total` | counter | `module_owner`, `major` | requisições | Requisições servidas a uma major `Deprecated` (base para decidir T-05). |

## 5. Métricas de resiliência

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_api_circuit_breaker_state` | gauge | `upstream_cluster` | enum (0=closed,1=half-open,2=open) | Estado corrente do *breaker* por upstream. |
| `aios_api_upstream_errors_total` | counter | `upstream_cluster`, `error_code` | erros | Total de erros por upstream, usado para calcular `error_threshold`. |
| `aios_api_bulkhead_rejections_total` | counter | `upstream_cluster` | rejeições | Requisições rejeitadas por saturação de *bulkhead* (antes mesmo do *breaker* abrir). |

## 6. Métricas de streaming (SSE)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_api_sse_active_streams` | gauge | `tenant_id` | streams | Nº de conexões SSE ativas na réplica. Meta: ≤ `api.sse.max_streams_per_replica` (NFR-004). |
| `aios_api_sse_delivery_latency_ms` | histogram | — | ms | Latência entre publicação NATS e entrega SSE ao cliente. Meta: p99 ≤ 2000 ms (FR-012). |

## 7. Exemplo de consulta PromQL

```promql
# p99 de overhead do Gateway na última 5 min, por rota
histogram_quantile(0.99,
  sum(rate(aios_api_gateway_overhead_latency_ms_bucket[5m])) by (le, route_id)
)

# Overshoot médio de rate limit acima da meta de 1%
avg(aios_api_ratelimit_overshoot_ratio) > 0.01
```

*Fim de `Metrics.md`.*
