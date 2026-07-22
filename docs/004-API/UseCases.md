---
Documento: UseCases
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0042, ADR-0044, ADR-0046, ADR-0047
RFCs relacionados: RFC-0001, RFC-0040
Depende de: ./FunctionalRequirements.md, ./StateMachine.md, ./API.md
---

# 004-API — Casos de Uso

> Cada caso de uso referencia os `FR-NNN` que satisfaz (`./FunctionalRequirements.md`)
> e é a base para os diagramas de `./SequenceDiagrams.md` e os cenários de
> `./Testing.md`. Ator "Cliente" cobre CLI (`030`), SDK (`031`), Web Console
> (`032`) e integrações A2A externas, salvo indicação contrária.

## UC-001 — Rotear requisição REST autenticada e autorizada (caminho feliz)

- **Ator:** Cliente autenticado.
- **Requisitos relacionados:** FR-001, FR-005.
- **Pré-condições:** Existe `RouteDefinition` `status=active` para o caminho/método/versão; token válido e não expirado.
- **Fluxo principal:**
  1. Cliente envia `POST /v1/kernel/agents` com `Authorization: Bearer <jwt>`, `Idempotency-Key` e `traceparent` opcional.
  2. `EdgeRouter` resolve a `RouteDefinition` ativa para `api_major` correto.
  3. `AuthNFilter` valida o JWT contra JWKS cacheado e popula claims.
  4. `RouteAuthorizer` consulta o cache de decisão (ou o PDP) e obtém `allow`.
  5. `RateLimiter` confirma que o tenant/rota está dentro da cota.
  6. `SchemaValidator` valida o corpo contra o `schema_ref` da rota.
  7. `CorrelationPropagator` garante/gera `traceparent`, propaga `X-AIOS-Tenant`.
  8. `EdgeRouter` encaminha ao `upstream_cluster` (mTLS interno).
  9. Resposta do upstream é repassada ao cliente; `GatewayTelemetry` registra trace/métrica.
- **Fluxos alternativos:** nenhum (ver UC-002..UC-008 para desvios).
- **Exceções:** timeout de upstream → `AIOS-API-0006` (ver `./FailureRecovery.md`).
- **Pós-condições:** requisição completada; span OTel e métrica `aios_api_gateway_overhead_latency_ms` registrados.

## UC-002 — Rejeitar requisição com AuthN de borda inválida

- **Ator:** Cliente com token ausente, expirado, malformado ou com `X-AIOS-Tenant` divergente.
- **Requisitos relacionados:** FR-002, FR-014, FR-008.
- **Pré-condições:** Rota `auth_required=true`.
- **Fluxo principal:**
  1. `AuthNFilter` tenta validar o JWT; falha de assinatura/expiração/issuer.
  2. `ErrorTranslator` monta o envelope RFC-0001 §5.4 com `code=AIOS-API-0002`, `status=401`.
  3. Requisição é rejeitada **antes** de alcançar `RouteAuthorizer`/upstream.
  4. `GatewayTelemetry` emite `aios.<tenant>.api.request.rejected` (amostrado) e sinal a `025-Audit`.
- **Fluxo alternativo (tenant divergente):** token válido, mas `X-AIOS-Tenant` do cabeçalho ≠ `tenant` da claim → `AIOS-API-0016`, `status=403`.
- **Pós-condições:** nenhuma chamada ao upstream ocorreu (NFR-008: 0 *bypass*).

## UC-003 — Negar requisição por decisão do PDP (autorização de rota)

- **Ator:** Cliente autenticado sem permissão de rota.
- **Requisitos relacionados:** FR-003, FR-008.
- **Pré-condições:** Token válido; rota `auth_required=true` com `required_roles` definidos.
- **Fluxo principal:**
  1. `AuthNFilter` valida o token com sucesso.
  2. `RouteAuthorizer` monta `DecisionRequest` (tenant, role, rota) e consulta `PolicyClient`.
  3. PDP (`022-Policy`) retorna `deny`.
  4. `ErrorTranslator` retorna `AIOS-API-0003`, `status=403`.
  5. Decisão é auditada via `025-Audit`.
- **Fluxo alternativo (PDP indisponível):** `PolicyClient` falha/timeout → `api.auth.fail_mode=closed` (default) → mesma resposta `AIOS-API-0003` (*default deny* estrutural).
- **Pós-condições:** nenhuma chamada ao upstream ocorreu.

## UC-004 — Rejeitar requisição por rate limit excedido

- **Ator:** Cliente/tenant que excede a cota configurada.
- **Requisitos relacionados:** FR-004.
- **Pré-condições:** `RateLimitPolicy` ativa (default ou específica de rota) para o tenant.
- **Fluxo principal:**
  1. Requisição passa por AuthN/AuthZ com sucesso.
  2. `RateLimiter` executa script Lua de token-bucket no Redis; bucket vazio.
  3. Resposta `429` com `code=AIOS-API-0004`, cabeçalho `Retry-After`.
  4. Sob limiar sustentado, evento `aios.<tenant>.api.ratelimit.breached` é emitido.
- **Fluxo alternativo (`enforcement=soft`):** requisição é permitida, mas marcada e alertada (não bloqueada).
- **Pós-condições:** cliente recebe orientação de *backoff* via `Retry-After`.

## UC-005 — Repetir mutação com `Idempotency-Key` (idempotência de borda)

- **Ator:** Cliente que repete uma chamada de mutação (retry de rede, timeout do cliente).
- **Requisitos relacionados:** FR-006.
- **Pré-condições:** Primeira chamada já foi processada e registrada em `IdempotencyRecord`.
- **Fluxo principal:**
  1. Cliente reenvia a mesma requisição com o mesmo `Idempotency-Key`.
  2. `IdempotencyRelay` encontra registro existente com `request_hash` idêntico.
  3. Resposta cacheada (`response_snapshot`, `status_code`) é devolvida sem chamar o upstream novamente.
- **Fluxo alternativo (reuse divergente):** mesma chave, `request_hash` diferente → `AIOS-API-0012`, `status=409`.
- **Pós-condições:** efeito de mutação ocorre exatamente uma vez no upstream (NFR-010).

## UC-006 — Rejeitar payload não conforme ao schema registrado

- **Ator:** Cliente que envia payload inválido.
- **Requisitos relacionados:** FR-007.
- **Pré-condições:** Rota com `schema_ref` definido e `api.schema_validation.enabled=true`.
- **Fluxo principal:**
  1. `SchemaValidator` valida o corpo contra o OpenAPI (REST) ou descritor proto (gRPC) do contrato ativo.
  2. Payload não conforme → `AIOS-API-0007`, `status=400`, sem chamar o upstream.
- **Pós-condições:** upstream nunca recebe payload inválido.

## UC-007 — Consumir rota de versão `Deprecated`

- **Ator:** Cliente que ainda usa uma major antiga.
- **Requisitos relacionados:** FR-009.
- **Pré-condições:** `ApiVersion` do módulo/major está em estado `Deprecated` (ver `./StateMachine.md`).
- **Fluxo principal:**
  1. `VersionNegotiator` resolve a major solicitada e identifica `state=Deprecated`.
  2. Requisição é roteada normalmente ao upstream.
  3. Resposta inclui `Deprecation: true` e `Sunset: <data estimada>` (invariante I3 da FSM).
- **Pós-condições:** cliente é avisado sem interrupção de serviço.

## UC-008 — Rejeitar requisição para versão `Retired`

- **Ator:** Cliente que usa uma major já retirada.
- **Requisitos relacionados:** FR-009.
- **Pré-condições:** `ApiVersion` em estado terminal `Retired`.
- **Fluxo principal:**
  1. `VersionNegotiator` identifica `state=Retired` para a major solicitada.
  2. Resposta `410 Gone`, `code=AIOS-API-0011`, sem chamar o upstream (invariante I2).
- **Pós-condições:** nenhuma chamada ao upstream ocorreu.

## UC-009 — Registrar nova `RouteDefinition` (administração)

- **Ator:** Operador com papel `api-registry-admin`.
- **Requisitos relacionados:** FR-011, FR-011a.
- **Pré-condições:** Ator autenticado com papel `api-registry-admin`; `api_major` referenciado já existe em `ApiVersion`.
- **Fluxo principal:**
  1. `POST /v1/api/routes` com `path_pattern`, `methods`, `protocol`, `upstream_cluster`, `api_major`.
  2. `RouteAuthorizer` confirma o papel (via PDP).
  3. `ContractRegistry` valida unicidade (`path_pattern`+`methods`+`api_major`) e persiste com OCC (`version=0`).
  4. Evento `aios._platform.api.route.registered` é publicado via Outbox.
- **Fluxo alternativo (conflito):** duplicidade → `AIOS-API-0015`, `status=409`.
- **Pós-condições:** rota disponível para roteamento imediatamente (invalidação de *snapshot* em memória via evento).

## UC-010 — Publicar nova `ApiVersion` (transição `Draft` → `Published`)

- **Ator:** Operador com papel `api-registry-admin` (tipicamente automatizado por *pipeline* de CI/CD do módulo dono).
- **Requisitos relacionados:** FR-011, FR-011b.
- **Pré-condições:** `ApiVersion` em `Draft`; contrato OpenAPI/proto publicado; contract tests aprovados; ≥ 1 `RouteDefinition` ativa referenciando a major (guarda T-02, `./StateMachine.md`).
- **Fluxo principal:**
  1. `POST /v1/api/versions/{module}/{major}:publish` (idempotente por `Idempotency-Key`).
  2. `ContractRegistry` valida as guardas de T-02.
  3. Transição `Draft → Published`; `published_at` preenchido; OCC incrementado.
  4. Evento `aios._platform.api.version.published` publicado.
- **Pós-condições:** rotas da major tornam-se roteáveis externamente sob o contrato publicado.

## UC-011 — Depreciar `ApiVersion` automaticamente ao publicar `M+1`

- **Ator:** Sistema (automático, disparado por UC-010 de uma nova major).
- **Requisitos relacionados:** FR-011, FR-011b.
- **Pré-condições:** Nova major `M+1` do mesmo módulo atinge `Published`; major `M` está em `Published`.
- **Fluxo principal:**
  1. Ao concluir T-02 para `M+1`, `LifecycleCoordinator` da FSM (interno ao `ContractRegistry`) verifica a invariante I1.
  2. Transição `M: Published → Deprecated` (T-04); `deprecated_at` preenchido.
  3. Evento `aios._platform.api.version.deprecated` publicado.
- **Pós-condições:** requisições a `M` passam a incluir `Deprecation`/`Sunset` (UC-007).

## UC-012 — Retirar `ApiVersion` (transição `Deprecated` → `Retired`)

- **Ator:** Operador com papel `api-registry-admin`.
- **Requisitos relacionados:** FR-011, FR-011b.
- **Pré-condições:** `now - deprecated_at ≥ deprecation_window_days`; tráfego residual abaixo de `api.version.retirement_traffic_threshold`; ≥ 2 majors mais novas em `Published`/`Deprecated` (guarda T-05).
- **Fluxo principal:**
  1. `POST /v1/api/versions/{module}/{major}:retire`.
  2. `ContractRegistry` valida as guardas de T-05 e a invariante I1.
  3. Transição `Deprecated → Retired` (terminal); `retired_at` preenchido.
  4. Evento `aios._platform.api.version.retired` publicado.
- **Fluxo alternativo (guarda falha):** requisição rejeitada com detalhe da guarda não satisfeita (não é um `AIOS-API-*` de rota; é erro de validação de administração).
- **Pós-condições:** requisições subsequentes à major retirada seguem UC-008.

## UC-013 — Consumir stream de eventos de tarefa via SSE

- **Ator:** Cliente (tipicamente Web Console) inscrito em eventos de uma tarefa/tenant.
- **Requisitos relacionados:** FR-012, FR-005, FR-015 (quando via gRPC-Web/console).
- **Pré-condições:** Cliente autenticado; tenant autorizado a assinar o subject correspondente.
- **Fluxo principal:**
  1. Cliente abre `GET /v1/tasks/{urn}/events` com `Accept: text/event-stream`.
  2. AuthN/AuthZ de borda aplicados como em UC-001.
  3. `SseStreamRelay` assina o subject NATS `aios.<tenant>.task.execution.*` relevante.
  4. Eventos são retransmitidos ao cliente como `event:`/`data:` SSE, com heartbeat a cada `api.sse.heartbeat_interval_s`.
- **Fluxo alternativo (limite de streams atingido):** réplica no limite de `api.sse.max_streams_per_replica` → `503`.
- **Pós-condições:** cliente recebe eventos em ≤ 2 s de latência de entrega (FR-012).

## UC-014 — Isolar upstream indisponível (circuit breaker/bulkhead)

- **Ator:** Sistema (automático).
- **Requisitos relacionados:** FR-013, FR-008.
- **Pré-condições:** Upstream de um `RouteDefinition` está falhando acima do limiar configurado.
- **Fluxo principal:**
  1. `CircuitBreakerManager` observa taxa de erro acima de `api.gateway.circuit_breaker.error_threshold` sobre `min_requests` amostras.
  2. Circuito abre para aquele *cluster* upstream especificamente.
  3. Novas requisições para rotas daquele upstream retornam `AIOS-API-0005` imediatamente, sem tentar a chamada de rede.
  4. Rotas de outros upstreams continuam operando normalmente (bulkhead — NFR-012).
  5. Após período de *half-open*, o circuito testa novamente e fecha se saudável.
- **Pós-condições:** degradação contida a um único upstream; demais rotas não afetadas em mais de 5% de p99 (NFR-012).

## UC-015 — Agregar contrato OpenAPI/proto de múltiplos módulos

- **Ator:** Sistema (automático, disparado por evento de publicação de contrato de módulo).
- **Requisitos relacionados:** FR-010, NFR-011.
- **Pré-condições:** Um módulo upstream publica `aios._platform.<module>.contract.published`.
- **Fluxo principal:**
  1. `OpenApiAggregator` consome o evento e busca o novo *bundle* de contrato (via `ContractRegistry`/MinIO).
  2. Recompõe o documento agregado `openapi.json`/descritor proto unificado.
  3. Documento agregado é servido em `GET /v1/api/openapi.json` / `GET /v1/api/proto`.
- **Fluxo alternativo (agregação periódica):** mesmo sem evento, `api.registry.contract_sync_interval_s` dispara reagregação de segurança.
- **Pós-condições:** `aios_api_contract_sync_lag_s` permanece ≤ 60 s (NFR-011).

## UC-016 — Consultar catálogo público de contratos (sem AuthN)

- **Ator:** Cliente anônimo (CLI 030, SDK 031, WebConsole 032 ou ferramenta de terceiros), sem token.
- **Requisitos relacionados:** FR-011c.
- **Pré-condições:** Nenhuma — o catálogo de metadados de contrato é público por definição (não expõe dados de tenant).
- **Fluxo principal:**
  1. Cliente emite `GET /v1/api/errors`, `GET /v1/api/resource-types` ou `GET /v1/api/events/schemas` **sem** cabeçalho `Authorization`.
  2. `EdgeRouter` reconhece a rota como pública (marcada `auth: none` na `RouteDefinition`) e **não** aciona `AuthNFilter`/`RouteAuthorizer`.
  3. `ContractRegistry` responde `200` com o catálogo correspondente (códigos `AIOS-API-*`, tipos de recurso do URN, schemas de evento e respectivos `dataschema`).
- **Fluxo alternativo (versão inválida):** consulta a `dataschema`/versão inexistente retorna `404`/`AIOS-API-0001`.
- **Pós-condições:** Resposta idêntica para todos os tenants (metadado de plataforma, sem `tenant_id`); nenhum evento por requisição é emitido.

*Fim de `UseCases.md`.*
