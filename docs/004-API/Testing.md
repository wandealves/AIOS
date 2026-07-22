---
Documento: Testing
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0044, ADR-0047
RFCs relacionados: RFC-0001
Depende de: ./FunctionalRequirements.md, ./NonFunctionalRequirements.md, ./UseCases.md
---

# 004-API — Estratégia de Testes

> Cada cenário (`TC-API-NNN`) referencia o `FR-NNN`/`NFR-NNN`/`UC-NNN` que
> valida, conforme a matriz de `./Requirements.md` §3.

## 1. Pirâmide de testes

```
                    ▲  E2E / Chaos (poucos, caros)
                   ╱ ╲    - fluxo completo cliente→Gateway→upstream real
                  ╱   ╲   - chaos: kill de réplica, upstream indisponível
                 ╱─────╲
                ╱ Contract╲  - contract tests OpenAPI/proto vs. upstream real
               ╱───────────╲ - garante FR-001, FR-010
              ╱  Integration ╲ - Gateway + Redis + PostgreSQL + NATS (via Testcontainers)
             ╱─────────────────╲
            ╱      Unit          ╲ - componentes isolados (RateLimiter, ErrorTranslator, VersionNegotiator)
           ╱───────────────────────╲
```

## 2. Cobertura-alvo por camada

| Camada | Cobertura-alvo | Ferramentas |
|--------|-------------------|----------------|
| Unit | ≥ 85% de linha nos componentes de `./ClassDiagrams.md` §2 | Testes unitários .NET, *mocks* de `IPolicyDecisionPoint`/`IIdempotencyStore`. |
| Integration | 100% dos fluxos de `./SequenceDiagrams.md` | Testcontainers (PostgreSQL, Redis, NATS). |
| Contract | 100% das rotas registradas (`RouteDefinition`) validadas contra o OpenAPI/proto do módulo upstream real | *Pact*-like contract testing ou verificação de schema automatizada. |
| E2E | Todos os `UC-NNN` de `./UseCases.md` | Ambiente de *staging* completo. |
| Chaos | Cenários de `./FailureRecovery.md` | Injeção de falha (kill de dependência, latência artificial). |
| Load | Metas de `./Benchmark.md` | k6/Gatling/vegeta. |

## 3. Cenários de teste (`TC-API-NNN`)

| ID | Cenário | Requisitos validados |
|----|---------|------------------------|
| TC-API-001 | Roteamento *happy path* REST resolve `RouteDefinition` correta e encaminha ao upstream esperado. | FR-001 |
| TC-API-002 | Token ausente/expirado/assinatura inválida retorna `AIOS-API-0002` sem alcançar upstream. | FR-002, NFR-008 |
| TC-API-003 | PDP retorna `deny` e, separadamente, PDP indisponível com `fail_mode=closed`, ambos retornam `AIOS-API-0003`. | FR-003 |
| TC-API-004 | Rate limit `hard` bloqueia acima do limite com `429`+`Retry-After`; `soft` permite e alerta. | FR-004, NFR-007 |
| TC-API-005 | `traceparent`/`X-AIOS-Tenant`/`X-AIOS-Agent` propagados/gerados em 100% das chamadas upstream. | FR-005, NFR-009 |
| TC-API-006 | Repetição com mesma `Idempotency-Key`+payload retorna resultado cacheado; payload divergente retorna `AIOS-API-0012`. | FR-006, NFR-010 |
| TC-API-007 | Payload não conforme ao schema é rejeitado com `AIOS-API-0007` sem chamar upstream. | FR-007 |
| TC-API-008 | Erro de upstream não conforme é normalizado para o envelope RFC-0001 §5.4 (`AIOS-API-0014`). | FR-008 |
| TC-API-009 | Major `Deprecated` inclui `Deprecation`/`Sunset`; major `Retired` retorna `410`/`AIOS-API-0011`. | FR-009 |
| TC-API-010 | `/v1/api/openapi.json` reflete novo contrato upstream em ≤ 60 s após evento `contract.published`. | FR-010, NFR-011 |
| TC-API-011 | Registro/atualização/desativação de rota e transições de `ApiVersion` respeitam guardas da FSM e papel `api-registry-admin`. | FR-011, FR-011a, FR-011b |
| TC-API-012 | Cliente SSE recebe eventos do seu tenant em ≤ 2 s; não recebe eventos de outro tenant. | FR-012 |
| TC-API-013 | Falha de um upstream não aumenta p99 de outras rotas em mais de 5%; *circuit breaker* abre/fecha corretamente. | FR-013, NFR-012 |
| TC-API-014 | `X-AIOS-Tenant` divergente do token é rejeitado com `AIOS-API-0016`. | FR-014 |
| TC-API-015 | Cliente gRPC-Web consegue invocar serviço gRPC interno via `GrpcWebBridge`. | FR-015 |
| TC-API-016 | `GET /v1/api/errors`, `/v1/api/resource-types` e `/v1/api/events/schemas` respondem `200` **sem** `Authorization`; `dataschema` inexistente retorna `404`/`AIOS-API-0001`. | FR-011c |

## 4. Fixtures e ambientes

- **Fixtures**: conjunto de `RouteDefinition`/`ApiVersion` sintéticas
  cobrindo `rest`, `grpc` e `sse`; tenants sintéticos (`acme`, `globex`) com
  `RateLimitPolicy` variadas (hard/soft, tenant/rota).
- **Ambientes**: `local` (Testcontainers), `staging` (topologia completa de
  `./Deployment.md` em escala reduzida), `load` (dimensionado para os
  volumes de `./Benchmark.md`).
- **Duplos de teste**: `022-Policy` e `021-Security` são substituídos por
  *stubs* controláveis em testes de integração (permitem simular `deny`,
  timeout, JWKS rotacionado) sem exigir os módulos reais.

## 5. Critérios de gate (bloqueiam merge/release)

- 100% dos `TC-API-NNN` acima passando em `staging`.
- Nenhuma regressão nas metas de `./NonFunctionalRequirements.md` (comparado
  ao *baseline* de `./Benchmark.md`).
- Contract tests 100% verdes contra os módulos upstream registrados em
  produção.
- Cobertura de teste unitário não regride abaixo do limiar definido em §2.
- Nenhum achado crítico/alto em *scan* de segurança (SAST/dependência) sem
  mitigação registrada.

*Fim de `Testing.md`.*
