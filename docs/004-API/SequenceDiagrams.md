---
Documento: SequenceDiagrams
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0044, ADR-0047
RFCs relacionados: RFC-0001
Depende de: ./UseCases.md, ./Architecture.md, ./API.md
---

# 004-API — Diagramas de Sequência

> Diagramas ASCII dos fluxos críticos (caminho feliz e de falha),
> correlacionados aos casos de uso de `./UseCases.md`. Timeouts e correlação
> seguem RFC-0001 §5.5–§5.6.

## 1. UC-001 — Roteamento REST autenticado e autorizado (caminho feliz)

```
Cliente     EdgeRouter   AuthNFilter  RouteAuthorizer  RateLimiter  SchemaValidator  Upstream
  │              │             │              │              │             │            │
  │ POST /v1/... │             │              │              │             │            │
  │─────────────▶│             │              │              │             │            │
  │              │ resolve RouteDefinition    │              │             │            │
  │              │─┐           │              │              │             │            │
  │              │◀┘           │              │              │             │            │
  │              │  Authenticate(token)       │              │             │            │
  │              │────────────▶│              │              │             │            │
  │              │             │ (JWKS cache) │              │             │            │
  │              │◀─claims─────│              │              │             │            │
  │              │  Decide(tenant,role,route) │              │             │            │
  │              │─────────────────────────────▶│             │            │
  │              │                              │ (cache hit) │            │
  │              │◀───────────allow─────────────│             │            │
  │              │  TryConsume(tenant,route)                  │            │
  │              │─────────────────────────────────────────▶│             │            │
  │              │◀────────────────────────────────────────ok│             │            │
  │              │  Validate(body, schema_ref)                             │            │
  │              │─────────────────────────────────────────────────────▶│            │
  │              │◀───────────────────────────────────────────────────ok│            │
  │              │  propagate traceparent/X-AIOS-Tenant/Idempotency-Key                │
  │              │───────────────────────────────────────────────────────────────────▶│
  │              │◀──────────────────────────────────────────────────────────200 OK───│
  │◀─────200─────│             │              │              │             │            │
```

Timeout do trecho Gateway→Upstream: `api.gateway.upstream_timeout_ms`
(default 5000 ms). Overhead total do Gateway (excluindo o upstream) DEVE
respeitar NFR-001 (p99 ≤ 5 ms).

## 2. UC-002 — Rejeição por AuthN inválida (caminho de falha, curto-circuito precoce)

```
Cliente     EdgeRouter   AuthNFilter   ErrorTranslator   025-Audit
  │              │             │              │              │
  │ token inválido/expirado    │              │              │
  │─────────────▶│────────────▶│              │              │
  │              │◀─── null claims (falha) ───│              │
  │              │  Translate(AuthNError)     │              │
  │              │────────────────────────────▶│              │
  │              │◀── envelope AIOS-API-0002 ──│              │
  │              │  emit api.request.rejected  │              │
  │              │───────────────────────────────────────────▶│
  │◀───401───────│             │              │              │
```

Nenhuma chamada a `RouteAuthorizer`, `RateLimiter` ou upstream ocorre — a
rejeição de AuthN é o primeiro filtro do pipeline (§1 de `./StateMachine.md`).

## 3. UC-003 — Negação por PDP (com fallback `fail_mode=closed`)

```
Cliente   AuthNFilter  RouteAuthorizer   PolicyClient    022-Policy(PDP)   ErrorTranslator
  │            │              │                │               │                │
  │ token OK   │              │                │               │                │
  │───────────▶│─claims──────▶│                │               │                │
  │            │              │ Decide(...)     │               │                │
  │            │              │───────────────▶│               │                │
  │            │              │                 │  gRPC call    │                │
  │            │              │                 │──────────────▶│                │
  │            │              │                 │◀──deny─────────│                │
  │            │              │◀────deny────────│               │                │
  │            │              │  Translate(AIOS-API-0003)                        │
  │            │              │──────────────────────────────────────────────────▶│
  │◀──────────────────────────403 (AIOS-API-0003)────────────────────────────────│
```

**Variante — PDP indisponível (timeout/CB aberto):**
```
                 │                 │  gRPC timeout │                │
                 │                 │──────────────▶│                │
                 │                 │◀── timeout ────│                │
                 │  fail_mode=closed → trata como deny              │
                 │────────────────────────────────────────────────▶│
                 │              403 (AIOS-API-0003) + alerta        │
```

## 4. UC-004 — Rate limit excedido

```
Cliente   RateLimiter (Redis Lua)   ErrorTranslator
  │              │                       │
  │  N+1 req/s   │                       │
  │─────────────▶│ bucket vazio          │
  │              │──────────────────────▶│
  │              │◀── AIOS-API-0004 ─────│
  │◀───429 + Retry-After─────────────────│
```

## 5. UC-005 — Repetição idempotente de mutação

```
Cliente        IdempotencyRelay (Redis+PG)    Upstream
  │  Idempotency-Key=K (1ª chamada)                  │
  │─────────────▶│ TryReserve(K, hash) → reservado    │
  │              │────────────────────────────────────▶│
  │              │◀───────────────────200 (result)─────│
  │              │ store response_snapshot              │
  │◀────200──────│                                       │

  │  Idempotency-Key=K (repetição, mesmo payload)        │
  │─────────────▶│ GetCached(K) → hit                     │
  │◀────200 (cached, upstream NÃO chamado)──────────────│

  │  Idempotency-Key=K (repetição, payload diferente)    │
  │─────────────▶│ hash diferente → conflito              │
  │◀────409 (AIOS-API-0012)──────────────────────────────│
```

## 6. UC-008 — Requisição para versão `Retired`

```
Cliente     VersionNegotiator     ErrorTranslator
  │              │                     │
  │ X-AIOS-Api-Version: 1 (Retired)    │
  │─────────────▶│ lookup ApiVersion(module,1) → state=Retired
  │              │────────────────────▶│
  │              │◀── AIOS-API-0011 ───│
  │◀───410 Gone──│                     │
```

Nenhuma chamada ao upstream ocorre (invariante I2 da FSM, `./StateMachine.md`).

## 7. UC-013 — Streaming SSE de eventos de tarefa

```
Cliente        SseStreamRelay        NATS/JetStream
  │  GET .../events (SSE)                  │
  │─────────────▶│ subscribe aios.<tenant>.task.execution.*
  │              │────────────────────────▶│
  │              │◀── evento (task.execution.progressed) ──│
  │◀── event: task.execution.progressed ───│
  │              │◀── heartbeat (a cada heartbeat_interval_s) ──
  │◀── : heartbeat ─────────────────────────│
  │              │◀── evento (task.execution.completed) ────│
  │◀── event: task.execution.completed ────│
  │  (cliente fecha conexão)                │
  │─────────────▶│ unsubscribe              │
```

## 8. UC-014 — Circuit breaker isolando upstream indisponível

```
Cliente A    EdgeRouter   CircuitBreakerMgr   Upstream-X (falho)   Upstream-Y (saudável)
  │              │               │                    │                    │
  │ req rota X   │  IsOpen(X)?   │                    │                    │
  │─────────────▶│──────────────▶│ (aberto: falhas > threshold)            │
  │              │◀── open ──────│                    │                    │
  │◀──503 (AIOS-API-0005), sem chamar Upstream-X───────│                    │

Cliente B                                                                    │
  │ req rota Y   │  IsOpen(Y)? → fechado               │                    │
  │─────────────▶│──────────────────────────────────────────────────────────▶│
  │◀──200────────────────────────────────────────────────────────────────────│
```

O isolamento garante que a falha de `Upstream-X` não aumenta o p99 de rotas
servidas por `Upstream-Y` em mais de 5% (NFR-012).

*Fim de `SequenceDiagrams.md`.*
