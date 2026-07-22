---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0006, ADR-0008, ADR-0010 (globais); ADR-0040..ADR-0049 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — este módulo mantém os registros criados por RFC-0001 §8); RFC-0040, RFC-0041 (deste módulo, a propor)
Depende de: 021-Security (JWKS/mTLS), 022-Policy (PDP), 020-Communication (NATS), 024-Observability, 025-Audit, 027-Cluster, 028-Deployment (topologia YARP); roteia para todos os serviços do plano de controle (006-019, 021-027) como *upstreams*; consumido por 030-CLI, 031-SDK, 032-WebConsole
---

# 004-API — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 004-API. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, subjects, versionamento), este
> documento **referencia** a `../003-RFC/RFC-0001-Architecture-Baseline.md` e **não
> o redefine** — ele o **especializa e opera** no ponto de entrada do sistema.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

> **Domínio de erro/subject deste módulo:** `API` (erros `AIOS-API-<NNNN>`) e
> domínio de subject `api` (rotas/versões/registro de contrato), conforme §5–§6.

---

## 0. Posição no AIOS e contrato de fronteira

O módulo 004-API é o **API Gateway** do AIOS: o **único ponto de entrada
norte-sul** (north-south) para clientes externos e internos-de-borda — CLI
(`030`), SDK (`031`), Web Console (`032`), integrações de terceiros e agentes
externos via A2A. Ele corresponde à camada **L3 BORDA** do Modelo de Camadas
(`../001-Architecture/Architecture.md` §6) e é implementado sobre **YARP
(.NET 10)**, conforme `../001-Architecture/Architecture.md` §4 e ADR global de
escolha de gateway.

Duas responsabilidades distintas e complementares definem o módulo:

1. **Plano de execução (runtime do Gateway)**: terminar REST e gRPC no edge,
   autenticar, autorizar grosseiramente por rota, aplicar rate limiting,
   idempotência de borda, validação de schema, versionamento e tradução de erro
   — e então **rotear/agregar** para o serviço do plano de controle dono do
   recurso, sem executar a lógica de negócio dele.
2. **Registro de contratos (registry)**: manter, conforme **RFC-0001 §8**, os
   catálogos centrais e versionados que todo módulo consulta e para os quais
   contribui — tipos de recurso URN, domínios de subject NATS, catálogo de
   códigos de erro `AIOS-<DOMINIO>-<NNNN>` e versões de `dataschema` de eventos.

```
   CLI(030) / SDK(031) / WebConsole(032) / A2A externo
                    │  HTTPS (REST) / gRPC / SSE
                    ▼
        ┌───────────────────────────────┐
        │   API GATEWAY (004 · YARP,    │   ◀── registra/serve: rotas, versões,
        │        .NET 10)               │        catálogo de erros, schemas de
        └───────────────┬───────────────┘        evento, tipos de URN (RFC-0001 §8)
                         │ REST/gRPC internos (mTLS)
     ┌───────────────────┼──────────────────────────────────────┐
     ▼                   ▼                                      ▼
 006-Kernel         009-Scheduler   010/011/012/...    021-Security / 022-Policy
 (e demais serviços do plano de controle, como upstreams roteados)
```

O Gateway **decide se a chamada entra**; os serviços do plano de controle
**decidem o que fazer com ela**. O Gateway NÃO executa lógica de domínio, NÃO é
o PDP (consulta o `022`), e NÃO gerencia identidade federada (consulta o `021`).

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O API Gateway é a **fachada de contrato externo** do AIOS: o análogo da
interface de chamadas de sistema exposta ao mundo externo, e do *ambassador*
que traduz o exterior (HTTP/gRPC público, versionado, multi-tenant) para o
interior (gRPC interno, mTLS, plano de controle governado). É a **fronteira de
confiança norte-sul**: tudo que entra no AIOS por rede pública passa por aqui
antes de tocar qualquer serviço do plano de controle. Ele também é o **guardião
dos registros centrais** de contrato (URN, subjects, erros, schemas de evento)
criados pela RFC-0001 §8, garantindo que todo módulo publique e consuma
contratos de forma consistente e versionada.

### 1.2 Responsabilidades (o API Gateway DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | **Terminar** REST (externo) e gRPC (interno/console) no edge, roteando cada requisição para o serviço upstream dono do recurso, conforme `RouteDefinition` ativa. |
| R-02 | Atuar como **ponto de validação de AuthN de borda**: validar tokens OAuth2/OIDC (JWT) usando as chaves publicadas (JWKS) por `021-Security`; rejeitar tokens ausentes/inválidos/expirados antes de rotear. |
| R-03 | Atuar como **PEP grosso de rota**: para toda rota marcada `auth_required`, consultar o **PDP** do `022-Policy` para autorização em nível de rota/role (ex.: "este tenant/role pode chamar `POST /v1/kernel/agents`?"), com **default deny**. A autorização fina de capacidade/recurso permanece no serviço de destino (ex.: Kernel `006`). |
| R-04 | Aplicar **rate limiting e quotas de borda** por tenant e por rota (token-bucket), sinalizando `429`/`AIOS-API-0004` e `Retry-After` sob saturação. |
| R-05 | Garantir e **propagar** os cabeçalhos de correlação obrigatórios (`traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` — RFC-0001 §5.6), gerando `traceparent` quando ausente. |
| R-06 | **Relayar idempotência de borda**: para mutações externas, verificar `Idempotency-Key` e devolver o resultado cacheado em repetições, delegando a execução real ao serviço de destino apenas na primeira ocorrência. |
| R-07 | **Validar o schema** de requisição (OpenAPI para REST, descritor proto para gRPC) contra o contrato registrado antes de encaminhar, rejeitando payloads não conformes. |
| R-08 | **Normalizar todo erro** (próprio do Gateway ou originado no upstream) no envelope RFC-0001 §5.4, garantindo `code`, `traceId`, `retriable` consistentes mesmo quando o upstream falha de forma não conforme. |
| R-09 | Implementar o **versionamento de API** (`/vN` + `X-AIOS-Api-Version`), resolvendo a rota para o `major` correto e sustentando a coexistência de ≥ 2 majors (RFC-0001 §5.7) via a máquina de estados de `ApiVersion` (§4). |
| R-10 | **Agregar e publicar** o contrato unificado (OpenAPI + proto) de todos os módulos upstream, mantendo-o sincronizado com as versões ativas. |
| R-11 | Manter os **registros centrais** de RFC-0001 §8: tipos de recurso URN (`<tipo>`), domínios de subject NATS, catálogo de códigos de erro `AIOS-<DOMINIO>-<NNNN>` e versões de `dataschema` de evento — aceitando novas entradas apenas via processo de PR + RFC/ADR de módulo. |
| R-12 | Suportar **streaming de eventos** (SSE / long-poll) para o cliente, multiplexando o consumo do NATS/JetStream de forma segura e por tenant. |
| R-13 | **Isolar upstreams com falha** via circuit breaker/bulkhead por serviço de destino, evitando que a indisponibilidade de um módulo derrube o Gateway inteiro. |
| R-14 | Rejeitar qualquer `X-AIOS-Tenant` divergente do tenant contido no token autenticado (isolamento multi-tenant de borda). |
| R-15 | Produzir **telemetria OTel** (traces/metrics/logs) de todo request de borda e registrar sinais de segurança relevantes (auth rejeitada, violação de contrato) para `025-Audit`. |

### 1.3 Não-Responsabilidades (o API Gateway NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Executar a lógica de negócio de qualquer módulo (planejar, escalonar, gerenciar memória, invocar ferramentas, etc.) — apenas roteia/agrega. | Cada módulo de destino (`006`, `009`, `010`...`019`, `021`-`027`) |
| NR-02 | **Decidir** autorização fina de capacidade/recurso (ex.: "este agente pode invocar esta ferramenta específica?"). O Gateway autoriza a **rota**; a decisão fina é do serviço de destino, que é o PEP de capacidade (ex.: Kernel `006` R-03). | Módulo de destino + `022-Policy` |
| NR-03 | Emitir, renovar ou revogar tokens OAuth2/OIDC; gerenciar chaves JWKS ou identidade federada. O Gateway apenas **valida** tokens com chaves publicadas por `021`. | `021-Security` |
| NR-04 | Persistir estado de negócio (ACB, memória, planos, execuções). Persiste apenas seus próprios registros de contrato/borda (§3). | Módulo dono do dado |
| NR-05 | Persistir a trilha de auditoria imutável (o Gateway emite sinais; o Audit persiste). | `025-Audit` |
| NR-06 | Fazer *scheduling*/*placement* de agentes ou tarefas. | `009-Scheduler` |
| NR-07 | Aplicar sandbox/isolamento de execução de agente. | `007-Agent-Runtime` / `021-Security` |
| NR-08 | Definir topologia de deployment, réplicas, autoscaling ou anti-affinity de zona. O Gateway é operado por essa topologia, não a define. | `028-Deployment` / `027-Cluster` |
| NR-09 | Calcular custo, orçamento ou faturamento de tenant. | `026-Cost-Optimizer` |
| NR-10 | Prover o broker de mensageria (NATS) em si; o Gateway apenas publica/consome subjects. | `020-Communication` |
| NR-11 | Ser a fonte da verdade de política RBAC/ABAC (é PEP de borda; a decisão é do PDP). | `022-Policy` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **EdgeRouter** | Núcleo YARP: resolve `RouteDefinition` ativa por caminho/método/versão e encaminha ao *cluster* upstream correto. É o *front door* físico do Gateway. | VersionNegotiator, RouteRegistry, CircuitBreakerManager; → upstreams gRPC/REST |
| **AuthNFilter** | Valida JWT (assinatura, expiração, issuer) contra JWKS cacheado; popula claims (`tenant`, `sub`, `roles`) no contexto da requisição. | JwksCache; → 021-Security (JWKS endpoint) |
| **RouteAuthorizer** | PEP grosso de rota: monta `DecisionRequest` (tenant, role, rota) e consulta o PDP; cacheia decisões com TTL curto; *default deny*. | PolicyClient; → 022-Policy |
| **PolicyClient** | Cliente resiliente do PDP (`022`): gRPC com circuit breaker, cache de decisões, `fail_mode` configurável. | → 022-Policy |
| **RateLimiter** | Token-bucket atômico (Redis/Lua) por (tenant, rota); aplica `enforcement` hard/soft; sinaliza `AIOS-API-0004`. | → Redis |
| **IdempotencyRelay** | Verifica/reserva `Idempotency-Key` na borda; devolve resultado cacheado em repetição; detecta reuse com payload divergente. | → Redis (quente) + PostgreSQL (durável) |
| **SchemaValidator** | Valida payload REST contra OpenAPI (JSON Schema) e mensagens gRPC contra descritor proto do contrato ativo. | ContractRegistry |
| **VersionNegotiator** | Resolve `/vN` + `X-AIOS-Api-Version` para o `major` correto conforme estado de `ApiVersion` (§4); aplica `Deprecation`/`Sunset` headers. | ApiVersionStore |
| **ErrorTranslator** | Normaliza qualquer erro (interno ou de upstream) no envelope RFC-0001 §5.4; mapeia `google.rpc.Status` (gRPC) para o mesmo `code`. | ErrorCodeRegistry |
| **CorrelationPropagator** | Garante/gera `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` em toda chamada (RFC-0001 §5.6). | GatewayTelemetry |
| **GrpcWebBridge** | Transcodifica REST↔gRPC (gRPC-Web) para consumidores web (`032-WebConsole`), preservando o contrato proto. | EdgeRouter |
| **SseStreamRelay** | Mantém streams SSE/long-poll por cliente, multiplexando a assinatura de subjects NATS relevantes por tenant/tarefa. | → NATS/JetStream |
| **ContractRegistry** | Mantém os registros de RFC-0001 §8: tipos de recurso URN, domínios de subject, catálogo de erros, versões de `dataschema`; expõe API de consulta/registro governada. | → PostgreSQL |
| **OpenApiAggregator** | Agrega specs OpenAPI/proto publicadas pelos módulos upstream em um contrato único versionado, servido em `/v1/api/openapi.json`. | ContractRegistry; consome `<module>.contract.published` |
| **CircuitBreakerManager** | Circuit breaker + bulkhead por serviço upstream; isola falhas, evita cascata; expõe estado a `EdgeRouter`. | → upstreams |
| **GatewayTelemetry** | Instrumentação OTel transversal (spans por request, métricas `aios_api_*`, logs Serilog→Seq) e emissão de sinais de segurança a `025-Audit`. | → 024-Observability, 025-Audit |

### 2.2 Diagrama de Componentes (ASCII)

```
                    HTTPS (REST) / gRPC / SSE  ·  clientes: CLI(030) SDK(031) WebConsole(032)
                                          │
   ┌──────────────────────────────────────▼──────────────────────────────────────┐
   │                      API GATEWAY SERVICE (004 · .NET 10 / YARP)              │
   │                                                                              │
   │   ┌───────────────┐        ┌──────────────────────┐                          │
   │   │  EdgeRouter   │───────▶│ CorrelationPropagator │                          │
   │   │  (YARP core)  │        └──────────┬───────────┘                          │
   │   └──────┬────────┘                   │                                      │
   │          │                            ▼                                      │
   │          │                   ┌───────────────────┐                           │
   │          ├──────────────────▶│   AuthNFilter      │──▶ 021-Security (JWKS)   │
   │          │                   └─────────┬──────────┘                          │
   │          │                             │ claims                              │
   │          │                             ▼                                     │
   │          │                   ┌───────────────────┐     ┌─────────────┐       │
   │          ├──────────────────▶│  RouteAuthorizer   │────▶│PolicyClient│──▶ 022 │
   │          │                   │      (PEP)         │     └─────────────┘       │
   │          │                   └─────────┬──────────┘                          │
   │          │ allow                        │                                     │
   │          ▼                              ▼                                     │
   │   ┌───────────────┐          ┌───────────────────┐        ┌────────────────┐ │
   │   │  RateLimiter  │◀────────▶│ IdempotencyRelay  │◀──────▶│ SchemaValidator│ │
   │   │  (Redis)      │          │  (Redis + PG)     │        └────────┬───────┘ │
   │   └───────┬───────┘          └───────────────────┘                 │         │
   │           │ ok                                                     ▼         │
   │           │                                              ┌────────────────┐  │
   │           │                                              │VersionNegotiator│  │
   │           │                                              └────────┬───────┘  │
   │           ▼                                                       ▼          │
   │   ┌────────────────────┐       ┌────────────────┐      ┌───────────────────┐ │
   │   │ CircuitBreakerMgr  │──────▶│  GrpcWebBridge │      │  SseStreamRelay   │ │
   │   └─────────┬──────────┘       └────────┬───────┘      └─────────┬─────────┘ │
   │             │ route/proxy               │                        │ subscribe │
   │             ▼                           ▼                        ▼           │
   │      upstreams gRPC/REST (006,009,010…021,022…)          NATS/JetStream       │
   │                                                                              │
   │   ┌──────────────────┐   ┌────────────────────┐   ┌──────────────────────┐  │
   │   │  ErrorTranslator │   │  ContractRegistry  │──▶│  OpenApiAggregator   │  │
   │   │  (RFC-0001 §5.4) │   │  (PostgreSQL)       │   │  /v1/api/openapi.json│  │
   │   └──────────────────┘   └────────────────────┘   └──────────────────────┘  │
   │                                                                              │
   │   ┌────────────────────────────────────────────────────────────────────┐    │
   │   │ GatewayTelemetry (OTel spans/metrics/logs + sinais de segurança)     │    │
   │   └────────────────────────────────────────────────────────────────────┘    │
   └──────────────┬───────────────┬───────────────┬──────────────────┬───────────┘
                  ▼               ▼               ▼                   ▼
        006-Kernel …   021-Security   022-Policy    NATS aios.<tenant>.api.*
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1. **Nuance deste módulo**: os registros de contrato
> (`RouteDefinition`, `ApiVersion`, `ErrorCodeRegistryEntry`,
> `EventSchemaRegistryEntry`, `ResourceTypeRegistryEntry`) são **metadados de
> plataforma**, não recursos de negócio de um tenant — por isso **NÃO carregam
> `tenant_id`/RLS**; sua escrita é restrita a um papel administrativo
> (`api-registry-admin`) via `RouteAuthorizer`, e sua leitura é pública
> (contrato do sistema). Já o estado de borda por tenant
> (`RateLimitPolicy`, `IdempotencyRecord`) **carrega `tenant_id` + RLS**, como
> exigido pela RFC-0001 §5.1 para todo recurso multi-tenant. Esta decisão de
> design (registro global vs. estado por tenant) DEVE ser validada no ADR-0043.

### 3.1 Entidade `RouteDefinition` — tabela `api.route`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | PK (ULID) | Identidade da rota. |
| `module_owner` | `text` | NOT NULL | Módulo dono, ex.: `006-Kernel`. |
| `path_pattern` | `text` | NOT NULL | Padrão de caminho REST, ex.: `/v1/kernel/agents/{urn}`. |
| `methods` | `text[]` | NULL (NULL = gRPC-only) | Verbos HTTP aceitos. |
| `protocol` | `text` | NOT NULL, CHECK ∈ {rest, grpc, sse} | Protocolo de borda. |
| `upstream_cluster` | `text` | NOT NULL | Identidade do cluster YARP de destino. |
| `api_major` | `int` | NOT NULL, FK→`api.version(major)` | Versão major à qual a rota pertence. |
| `auth_required` | `boolean` | NOT NULL default true | Exige AuthN/AuthZ de borda. |
| `required_roles` | `text[]` | NULL | Roles grossas aceitas (RouteAuthorizer). |
| `rate_limit_policy_id` | `uuid` | NULL, FK→`api.rate_limit_policy.id` | Política aplicável (default se NULL). |
| `schema_ref` | `text` | NULL | Referência ao OpenAPI `operationId`/proto `rpc`. |
| `status` | `text` | NOT NULL, CHECK ∈ {draft, active, disabled} | Estado operacional da rota. |
| `version` | `bigint` | NOT NULL default 0 | Concorrência otimista (OCC). |
| `created_at` | `timestamptz` | NOT NULL | Registro. |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(id)`; `UNIQUE(path_pattern, methods, api_major)`;
`btree(module_owner, status)`; `btree(api_major)`.

### 3.2 Entidade `ApiVersion` — tabela `api.version`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `module_owner` | `text` | PK (parte 1) | Módulo dono da versão (ex.: `006-Kernel`, ou `platform` para o Gateway). |
| `major` | `int` | PK (parte 2), ≥ 1 | Versão major. |
| `state` | `text` | NOT NULL, CHECK ∈ ApiVersionState | Estado da FSM (§4). |
| `contract_ref` | `text` | NOT NULL | URI do bundle OpenAPI/proto publicado (MinIO). |
| `deprecation_window_days` | `int` | NOT NULL default 180 | Janela mínima antes de `Retired`. |
| `published_at` | `timestamptz` | NULL | Quando entrou em `Published`. |
| `deprecated_at` | `timestamptz` | NULL | Quando entrou em `Deprecated`. |
| `retired_at` | `timestamptz` | NULL | Quando entrou em `Retired`. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(module_owner, major)`; `btree(module_owner, state)`.

### 3.3 Entidade `ErrorCodeRegistryEntry` — tabela `api.error_code`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `code` | `text` | PK | `AIOS-<DOMINIO>-<NNNN>` (RFC-0001 §5.4). |
| `domain` | `text` | NOT NULL | Domínio reservado (ex.: `API`, `KERNEL`, `SCHED`). |
| `http_status` | `smallint` | NOT NULL | Status HTTP associado. |
| `retriable` | `boolean` | NOT NULL | Indica se o cliente PODE repetir. |
| `title` | `text` | NOT NULL | Título curto (RFC 7807 `title`). |
| `owner_module` | `text` | NOT NULL | Módulo que registrou o código. |
| `registered_at` | `timestamptz` | NOT NULL | Data de registro. |

Índices: `PK(code)`; `btree(domain)`.

### 3.4 Entidade `EventSchemaRegistryEntry` — tabela `api.event_schema`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `dataschema` | `text` | PK | `aios://schemas/<type>/<version>` (RFC-0001 §5.2). |
| `event_type` | `text` | NOT NULL | `aios.<dominio>.<entidade>.<acao>`. |
| `json_schema` | `jsonb` | NOT NULL | Schema JSON do payload `data`. |
| `owner_module` | `text` | NOT NULL | Módulo produtor do evento. |
| `status` | `text` | NOT NULL, CHECK ∈ {draft, active, deprecated} | Estado do schema. |
| `registered_at` | `timestamptz` | NOT NULL | Data de registro. |

Índices: `PK(dataschema)`; `btree(event_type)`.

### 3.5 Entidade `ResourceTypeRegistryEntry` — tabela `api.resource_type`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `type` | `text` | PK | `<tipo>` do URN (RFC-0001 §5.1), ex.: `agent`, `task`. |
| `description` | `text` | NOT NULL | Descrição semântica do tipo. |
| `owner_module` | `text` | NOT NULL | Módulo dono do recurso deste tipo. |
| `introduced_in` | `text` | NULL | ADR/RFC que introduziu o tipo. |
| `registered_at` | `timestamptz` | NOT NULL | Data de registro. |

Índices: `PK(type)`.

### 3.6 Entidade `RateLimitPolicy` — tabela `api.rate_limit_policy`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `uuid` | PK | Identidade da política. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `scope` | `text` | NOT NULL, CHECK ∈ {tenant, route} | Nível de aplicação. |
| `route_id` | `char(26)` | NULL, FK→`api.route.id` | Rota alvo se `scope=route`. |
| `rps` | `int` | NOT NULL | Requisições por segundo permitidas. |
| `burst` | `int` | NOT NULL | Capacidade de rajada. |
| `window` | `interval` | NOT NULL default '1 second' | Janela de reset. |
| `enforcement` | `text` | NOT NULL, CHECK ∈ {hard, soft} | `hard`=rejeita; `soft`=marca e alerta. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(id)`; `btree(tenant_id, scope)`.

### 3.7 Entidade `IdempotencyRecord` (borda) — tabela `api.idempotency_record`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `idempotency_key` | `text` | PK (composta com `tenant_id`) | Chave fornecida pelo cliente (RFC-0001 §5.5). |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `route_id` | `char(26)` | NOT NULL, FK→`api.route.id` | Rota da mutação original. |
| `request_hash` | `text` | NOT NULL | Hash do payload, para detectar reuse divergente. |
| `response_snapshot` | `jsonb` | NOT NULL | Corpo/status a devolver em repetição. |
| `status_code` | `int` | NOT NULL | Status HTTP da primeira resposta. |
| `expires_at` | `timestamptz` | NOT NULL | ≥ 24h após criação (RFC-0001 §5.5). |

Índices: `PK(tenant_id, idempotency_key)`; `btree(expires_at)` (expurgo).

### 3.8 Relações (ASCII)

```
   ApiVersion(1)───<RouteDefinition(*)───1 RateLimitPolicy(scope=route)
        │  contract_ref                          │
        ▼                                         └───< IdempotencyRecord(*) (tenant, rota)
   OpenAPI/proto bundle (MinIO)

   ErrorCodeRegistry(*)   EventSchemaRegistry(*)   ResourceTypeRegistry(*)
   (registros globais de plataforma — referenciados, não possuídos, por todos os módulos)

   Tenant(1)───< RateLimitPolicy(scope=tenant)
        └───< IdempotencyRecord(*)
```

---

## 4. Máquina de Estados Canônica — Ciclo de Vida de `ApiVersion`

> O Gateway em si é majoritariamente um **pipeline de requisição sem estado**
> (Recebida → Autenticada → Autorizada → Limitada → Validada → Roteada →
> Traduzida), não uma FSM de entidade persistente — cada etapa é um filtro
> YARP idempotente e sem memória entre requisições. A **única entidade com
> máquina de estados canônica relevante** neste módulo é `ApiVersion`, que
> implementa a regra de versionamento da RFC-0001 §5.7 (coexistência de ≥ 2
> majors) e governa quando uma rota pode ser desativada com segurança.

### 4.1 Estados (`ApiVersionState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Draft` | Versão major em desenvolvimento; ainda não roteável para clientes externos. | não |
| `Published` | Versão ativa, roteável, coberta por SLO; contrato publicado e testado. | não |
| `Deprecated` | Versão ainda roteável, mas sinalizada como obsoleta (`Deprecation`/`Sunset` headers); nova major já `Published`. | não |
| `Retired` | Versão desativada; requisições retornam `410 Gone` (`AIOS-API-0011`). | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Draft` | registro de nova major (admin/PR) | Módulo dono aprovado (ADR/RFC do módulo referenciado). |
| T-02 | `Draft` → `Published` | `publish` | Contrato OpenAPI/proto publicado ∧ contract tests passam ∧ ≥ 1 `RouteDefinition` ativa referenciando a major. |
| T-03 | `Draft` → `Retired` | `cancel` | Nenhuma rota ativa referencia a major (versão abandonada antes do lançamento). |
| T-04 | `Published` → `Deprecated` | nova major `M+1` atinge `Published` | Versão `M` NÃO É a única `Published`/`Deprecated` do módulo (garante coexistência, invariante I1). |
| T-05 | `Deprecated` → `Retired` | `retire` | `now - deprecated_at ≥ deprecation_window_days` ∧ tráfego observado abaixo do limiar de retirada (`api.version.retirement_traffic_threshold`) ∧ ≥ 2 majors mais novas em `Published`/`Deprecated`. |
| T-06 | `Deprecated` → `Published` | `undeprecate` (excepcional) | Reversão administrativa antes de `Retired`, com justificativa auditada. |

**Invariantes:** (I1) uma versão major **NÃO DEVE** transicionar para `Retired`
se for a única versão em `Published`/`Deprecated` do módulo — garante a
coexistência mínima de 2 majors exigida pela RFC-0001 §5.7. (I2) toda
requisição para uma major em `Retired` retorna `410 Gone`
(`AIOS-API-0011`). (I3) toda requisição para uma major em `Deprecated`
inclui os cabeçalhos `Deprecation: true` e `Sunset: <data estimada>`. (I4) toda
transição é OCC (`version`) e emite exatamente um evento
`aios._platform.api.version.<acao>` (§6).

### 4.3 Diagrama de estados (ASCII)

```
                registrar (T-01)
      ∅ ──────────────────────────▶ ┌─────────┐
                                     │  Draft  │
                                     └────┬────┘
                 publish (T-02)           │  cancel (T-03, sem rotas ativas)
              ┌──────────────────────────┤────────────────────┐
              ▼                          │                    ▼
        ┌───────────┐                    │              ┌───────────┐
        │ Published │◀── undeprecate ────┼──────────────│  Retired  │ (terminal)
        └─────┬─────┘   (T-06, exc.)     │              └───────────┘
              │ nova major M+1 Published (T-04)                ▲
              ▼                                                 │ retire (T-05):
        ┌───────────┐   janela+tráfego OK + ≥2 majors novas ────┘  janela cumprida ∧
        │Deprecated │──────────────────────────────────────────────  tráfego residual baixo
        └───────────┘
```

---

## 5. Superfície de API (REST e gRPC)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7.
> Pacote gRPC: `aios.api.v1`. Base REST: `/v1/api` (endpoints de **administração
> e registro** do próprio Gateway). A **maioria** do tráfego do módulo é
> *pass-through* transparente: cada rota registrada (`RouteDefinition`)
> preserva o contrato REST/gRPC do módulo de destino — o Gateway aplica apenas
> os *cross-cutting concerns* desta seção (AuthN/AuthZ de rota, rate limit,
> idempotência, validação de schema, correlação, tradução de erro) e não
> redefine o contrato de negócio de cada módulo (ver `API.md` de cada módulo).

### 5.1 Operações de administração/registro (`aios.api.v1.RegistryService`)

| Operação | REST | gRPC (rpc) | Idempotente | Notas |
|----------|------|-----------|-------------|-------|
| Listar rotas | `GET /v1/api/routes` | `ListRoutes` | leitura | Filtra por `module_owner`, `status`, `api_major`. |
| Registrar/atualizar rota | `POST /v1/api/routes` | `RegisterRoute` | sim (key) | Papel `api-registry-admin`. |
| Desativar rota | `POST /v1/api/routes/{id}:disable` | `DisableRoute` | sim | → `status=disabled`. |
| Publicar contrato OpenAPI/proto agregado | `GET /v1/api/openapi.json` / `GET /v1/api/proto` | `GetAggregatedContract` | leitura | Servido por `OpenApiAggregator`. |
| Listar versões | `GET /v1/api/versions` | `ListVersions` | leitura | Estado da FSM (§4) por módulo. |
| Publicar versão | `POST /v1/api/versions` | `PublishVersion` | sim (key) | T-02. |
| Depreciar versão | `POST /v1/api/versions/{module}/{major}:deprecate` | `DeprecateVersion` | sim | T-04 (geralmente automática ao publicar `M+1`). |
| Retirar versão | `POST /v1/api/versions/{module}/{major}:retire` | `RetireVersion` | sim | T-05. |
| Consultar catálogo de erros | `GET /v1/api/errors` | `ListErrorCodes` | leitura | RFC-0001 §8. |
| Registrar código de erro | `POST /v1/api/errors` | `RegisterErrorCode` | sim (key) | Governado (ADR/RFC de módulo). |
| Consultar schemas de evento | `GET /v1/api/events/schemas` | `ListEventSchemas` | leitura | RFC-0001 §8. |
| Registrar schema de evento | `POST /v1/api/events/schemas` | `RegisterEventSchema` | sim (key) | Governado. |
| Consultar tipos de recurso (URN) | `GET /v1/api/resource-types` | `ListResourceTypes` | leitura | RFC-0001 §5.1/§8. |
| Saúde do Gateway | `GET /v1/healthz` / `GET /v1/readyz` | `HealthCheck` | leitura | Orquestração (`028`). |

### 5.2 Catálogo de códigos de erro (domínio `API`, faixa reservada 0001–0099)

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-API-0001` | 404 | não | Nenhuma `RouteDefinition` ativa corresponde ao caminho/método/versão. |
| `AIOS-API-0002` | 401 | não | Token OAuth2/OIDC ausente, inválido ou expirado. |
| `AIOS-API-0003` | 403 | não | Autorização de rota negada pelo PDP (*default deny*). |
| `AIOS-API-0004` | 429 | sim | Rate limit de borda excedido (tenant ou rota). |
| `AIOS-API-0005` | 503 | sim | Upstream indisponível / circuit breaker aberto. |
| `AIOS-API-0006` | 504 | sim | Timeout aguardando resposta do upstream. |
| `AIOS-API-0007` | 400 | não | Payload não conforme ao schema registrado (OpenAPI/proto). |
| `AIOS-API-0008` | 415 | não | `Content-Type`/codec não suportado. |
| `AIOS-API-0009` | 413 | não | Corpo da requisição excede `api.gateway.max_body_size_bytes`. |
| `AIOS-API-0010` | 400 | não | Versão de API solicitada inválida ou não registrada. |
| `AIOS-API-0011` | 410 | não | Versão de API em estado `Retired` (§4). |
| `AIOS-API-0012` | 409 | não | `Idempotency-Key` reusada com payload divergente (`request_hash` diferente). |
| `AIOS-API-0013` | 400 | não | Cabeçalho de correlação obrigatório ausente (`traceparent`/`X-AIOS-Tenant`). |
| `AIOS-API-0014` | 502 | sim | Upstream retornou erro fora do envelope RFC-0001 §5.4 (não conforme). |
| `AIOS-API-0015` | 409 | não | Conflito ao registrar rota (duplicidade de `path_pattern`+`methods`+`api_major`). |
| `AIOS-API-0016` | 403 | não | `X-AIOS-Tenant` divergente do tenant no token autenticado. |

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3
> (`aios.<tenant>.<dominio>.<entidade>.<acao>`). `<dominio>` = `api`. Entrega
> at-least-once via JetStream; consumidores DEVEM deduplicar por `event.id`.
>
> **Convenção especializada deste módulo**: eventos de **ciclo de vida do
> registro de contrato** (rota, versão, código de erro, schema de evento) são
> **metadados de plataforma**, não eventos de um tenant específico — eles usam
> o tenant reservado **`_platform`** (`aios._platform.api.*`), documentado aqui
> e a ser ratificado em ADR-0048. Eventos que refletem **atividade por tenant**
> (rate limit excedido, autenticação rejeitada, violação de contrato) usam o
> `<tenant>` real do chamador.

### 6.1 Eventos emitidos (produtor: API Gateway)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios._platform.api.route.registered` | `aios.api.route.registered` | Nova `RouteDefinition` criada. | API_REGISTRY (durável) |
| `aios._platform.api.route.updated` | `aios.api.route.updated` | Rota atualizada. | API_REGISTRY |
| `aios._platform.api.route.disabled` | `aios.api.route.disabled` | Rota desativada. | API_REGISTRY |
| `aios._platform.api.version.published` | `aios.api.version.published` | `ApiVersion` → `Published` (T-02). | API_REGISTRY |
| `aios._platform.api.version.deprecated` | `aios.api.version.deprecated` | `ApiVersion` → `Deprecated` (T-04). | API_REGISTRY |
| `aios._platform.api.version.retired` | `aios.api.version.retired` | `ApiVersion` → `Retired` (T-05). | API_REGISTRY |
| `aios._platform.api.errorcode.registered` | `aios.api.errorcode.registered` | Novo código `AIOS-<DOMINIO>-<NNNN>` registrado. | API_REGISTRY |
| `aios._platform.api.eventschema.registered` | `aios.api.eventschema.registered` | Novo `dataschema` registrado. | API_REGISTRY |
| `aios.<tenant>.api.request.rejected` | `aios.api.request.rejected` | AuthN/AuthZ/rate-limit/schema rejeitou (amostrado, sinal de segurança). | API_SECURITY |
| `aios.<tenant>.api.contract.violation` | `aios.api.contract.violation` | Payload não conforme ao schema registrado. | API_SECURITY |
| `aios.<tenant>.api.ratelimit.breached` | `aios.api.ratelimit.breached` | Limiar sustentado de rate-limit excedido (sinal de saturação/abuso). | API_SECURITY |

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do Gateway |
|-------------------|----------|------------------|
| `aios.<tenant>.policy.decision.updated` | `022-Policy` | Invalida cache de decisões do `RouteAuthorizer`. |
| `aios._platform.security.jwks.rotated` | `021-Security` | Recarrega `JwksCache` do `AuthNFilter`. |
| `aios.<tenant>.security.token.revoked` | `021-Security` | Invalida sessões/tokens ativos correspondentes na borda. |
| `aios._platform.cluster.capacity.changed` | `027-Cluster` | Atualiza saúde de upstream para `CircuitBreakerManager`. |
| `aios._platform.<module>.contract.published` | Cada módulo upstream | Dispara `OpenApiAggregator` a re-agregar o contrato unificado. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Terminar REST e gRPC no edge e rotear conforme `RouteDefinition` ativa. | Must | Toda rota registrada resolve ao cluster upstream correto; 100% cobertura em contract test. |
| FR-002 | Validar token OAuth2/OIDC via JWKS de `021` antes de rotear. | Must | Token inválido/expirado retorna `AIOS-API-0002`; nunca alcança upstream. |
| FR-003 | Consultar o PDP (`022`) para autorização de rota antes de encaminhar. | Must | Sem PDP=allow, requisição retorna `AIOS-API-0003`; auditoria registra `deny`. |
| FR-004 | Aplicar rate limiting/quota por tenant e por rota. | Must | Excesso retorna `429`/`AIOS-API-0004` com `Retry-After`. |
| FR-005 | Propagar/gerar cabeçalhos de correlação (RFC-0001 §5.6) em toda chamada. | Must | 100% das requisições upstream carregam `traceparent`+`X-AIOS-Tenant`. |
| FR-006 | Garantir idempotência de borda por `Idempotency-Key` (≥24h). | Must | Repetição retorna resultado idêntico; payload divergente → `AIOS-API-0012`. |
| FR-007 | Validar payload contra o schema registrado antes de encaminhar. | Should | Payload não conforme retorna `AIOS-API-0007` sem chamar upstream. |
| FR-008 | Normalizar todo erro (próprio ou upstream) no envelope RFC-0001 §5.4. | Must | 100% das respostas de erro conformes ao schema RFC 7807 do AIOS. |
| FR-009 | Resolver versão via `/vN`+header e sustentar coexistência de ≥2 majors. | Must | `Retired` retorna `410`; `Deprecated` inclui `Sunset`; nunca há <2 majors coexistentes sem plano de retirada aprovado. |
| FR-010 | Agregar e publicar o contrato unificado (OpenAPI+proto) dos módulos. | Must | `/v1/api/openapi.json` reflete contratos ativos em ≤ 60 s após publicação upstream (NFR-011). |
| FR-011 | Manter os registros de RFC-0001 §8 (tipos URN, subjects, erros, event schemas). | Must | Toda entrada nova exige PR + ADR/RFC referenciado; catálogo consultável via API. |
| FR-012 | Suportar streaming SSE de eventos de tarefa por tenant. | Should | Cliente recebe eventos `task.execution.*` do seu tenant em ≤ 2 s de latência de entrega. |
| FR-013 | Isolar upstreams com falha via circuit breaker por serviço. | Must | Falha de um módulo não degrada latência de rotas de outros módulos (bulkhead comprovado em chaos test). |
| FR-014 | Rejeitar `X-AIOS-Tenant` divergente do tenant autenticado. | Must | Retorna `AIOS-API-0016`; evento `api.request.rejected`. |
| FR-015 | Transcodificar REST↔gRPC (gRPC-Web) para o Web Console. | Should | Console consome serviços gRPC internos sem cliente gRPC nativo no browser. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Latência (overhead de borda) | p99 do overhead do Gateway ≤ **5 ms**, p50 ≤ **1,5 ms** (sem contar o tempo do upstream). | histograma `aios_api_gateway_overhead_latency_ms`. |
| NFR-002 | Throughput | ≥ **150.000 req/s** por réplica (proxy stateless, I/O-bound). | load test sustentado (`Benchmark.md`). |
| NFR-003 | Disponibilidade | ≥ **99,99%** do endpoint de borda (ponto único de entrada norte-sul). | uptime probe multi-região; error budget mensal. |
| NFR-004 | Escalabilidade de conexões | ≥ **100.000** streams SSE concorrentes por réplica; escala horizontal sem limite teórico (stateless). | teste de escala `Scalability.md`. |
| NFR-005 | Durabilidade de registros | Perda de evento de registro de contrato = **0** sob crash (Outbox + JetStream). | chaos test kill entre commit/publish. |
| NFR-006 | Recuperação | RTO ≤ **5 min** por réplica (stateless, recarrega snapshot); RTO ≤ **10 min** para o `ContractRegistry` (PG); RPO ≤ **5 min**. | DR drill; ver §9. |
| NFR-007 | Precisão de rate limit | Overshoot de limite `hard` ≤ **1%** sob concorrência. | teste concorrente com token-bucket. |
| NFR-008 | Segurança | **100%** dos tokens inválidos/expirados rejeitados antes de alcançar upstream; **0** bypass de PEP de rota. | pentest + auditoria de cobertura. |
| NFR-009 | Observabilidade | **100%** das requisições de borda com trace OTel + correlação completa. | inspeção de spans; cobertura de campos. |
| NFR-010 | Idempotência | Repetições de mutação produzem efeito único em **100%** dos casos. | teste de replay/duplicação. |
| NFR-011 | Consistência de contrato agregado | `/v1/api/openapi.json` reflete novo contrato upstream em ≤ **60 s**. | métrica `aios_api_contract_sync_lag_s`. |
| NFR-012 | Isolamento de falha (bulkhead) | Indisponibilidade de 1 upstream não aumenta p99 de outras rotas em mais de **5%**. | chaos test de falha isolada. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global` (cluster), `tenant`, `route`. Recarregável indica *hot
> reload*. Prefixo de env var: `AIOS_API_`.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `api.gateway.rate_limit.default_rps` | 100 | 1–100000 | tenant | sim | RPS default por tenant quando sem `RateLimitPolicy` explícita. |
| `api.gateway.rate_limit.default_burst` | 200 | 1–200000 | tenant | sim | Rajada default. |
| `api.gateway.max_body_size_bytes` | 10485760 | 1024–104857600 | global/route | sim | Tamanho máximo de corpo aceito. |
| `api.gateway.upstream_timeout_ms` | 5000 | 100–60000 | global/route | sim | Timeout aguardando resposta do upstream. |
| `api.gateway.circuit_breaker.error_threshold` | 0.5 | 0–1 | global | sim | Taxa de erro que abre o breaker por upstream. |
| `api.gateway.circuit_breaker.min_requests` | 20 | 1–1000 | global | sim | Amostra mínima antes de avaliar abertura. |
| `api.version.coexistence_majors_min` | 2 | 1–5 | global | sim | Nº mínimo de majors coexistentes (RFC-0001 §5.7). |
| `api.version.deprecation_window_days` | 180 | 30–720 | global | sim | Janela mínima entre `Deprecated` e `Retired`. |
| `api.version.retirement_traffic_threshold` | 0.01 | 0–1 | global | sim | Fração máxima de tráfego residual para permitir T-05. |
| `api.auth.jwks_cache_ttl_s` | 300 | 30–3600 | global | sim | TTL do cache de chaves JWKS. |
| `api.auth.fail_mode` | `closed` | {closed, open} | global | sim | Comportamento se `021`/`022` indisponíveis. `closed`=nega. |
| `api.idempotency.ttl_hours` | 24 | 24–720 | global | sim | Retenção de resultados idempotentes (≥24h RFC-0001). |
| `api.schema_validation.enabled` | true | bool | global/route | sim | Ativa/desativa `SchemaValidator` por rota. |
| `api.cors.allowed_origins` | `[]` | lista | tenant | sim | Origens CORS permitidas por tenant. |
| `api.sse.max_streams_per_replica` | 20000 | 100–200000 | global | sim | Limite de streams SSE concorrentes por réplica. |
| `api.sse.heartbeat_interval_s` | 15 | 5–120 | global | sim | Intervalo de heartbeat SSE. |
| `api.registry.contract_sync_interval_s` | 30 | 5–300 | global | sim | Frequência de re-agregação do `OpenApiAggregator`. |
| `api.registry.write_role` | `api-registry-admin` | string | global | **não** | Role exigida para mutar registros de contrato. |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| `021-Security` (JWKS) indisponível | Timeout no `AuthNFilter` | Bulkhead de AuthN | Usa cache JWKS até `jwks_cache_ttl_s`; expirado → `fail_mode=closed` nega (`AIOS-API-0002`) | Retry idempotente após religar |
| `022-Policy` (PDP) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do `RouteAuthorizer` | `fail_mode=closed` → nega (`AIOS-API-0003`); alerta | Retry idempotente pós meia-abertura do CB |
| Upstream (qualquer módulo) indisponível | CB por serviço | Isolamento por `CircuitBreakerManager` (bulkhead por cluster) | `AIOS-API-0005`; outras rotas não afetadas (NFR-012) | Retry idempotente com backoff no cliente |
| Redis (rate limit/idempotência) indisponível | health probe | Fail-safe conservador | Modo degradado: rejeita rajadas acima do default estático; idempotência cai para PostgreSQL (mais lento, sem perda) | Sem duplo efeito (verificação dupla Redis+PG) |
| PostgreSQL (`ContractRegistry`) indisponível | timeout | Rotas seguem servidas do **snapshot em memória** (último carregado) | Leitura de rotas continua; mutações de registro ficam bloqueadas até religar | N/A (somente leitura afetada) |
| NATS/JetStream indisponível | health do `EventPublisher` | Outbox retém eventos de registro | Backlog drenado ao religar; SSE degrada para *long-poll* | Ordenado por stream; dedupe por `event.id` |
| Crash de réplica do Gateway | health/readiness probe | Réplica stateless removida do LB | Nova réplica sobe e recarrega snapshot em < 1 min | N/A (sem estado local a recuperar) |
| Contrato agregado desatualizado | `contract_sync_lag_s` acima do limiar | N/A | Serve última versão válida com header de aviso; força re-sync | N/A |
| Loop de rejeição por PDP flutuante | métrica de taxa de deny | Cache de decisão evita martelar o PDP | Cache com TTL curto e *jitter* evita *thundering herd* | Idempotente por natureza |

**Metas de recuperação:** réplicas do Gateway (stateless) recuperam em
**< 1 min**; o `ContractRegistry` (PostgreSQL) tem **RTO ≤ 10 min, RPO ≤ 5 min**
(alinhado a V-R10 da Visão e §2 da Arquitetura). Degradação graciosa: sob
indisponibilidade de `021`/`022`, o Gateway prioriza **negar com segurança**
(default deny) em vez de rotear sem autenticação/autorização.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Particionamento | O Gateway **não fragmenta estado por shard**: é um proxy **stateless** por natureza; qualquer réplica atende qualquer requisição. Não há `hash(tenant,agent) mod N` aplicável a este módulo (nota de decisão: ver §11 ADR-0040). |
| Estado externalizado | Contadores de rate limit e cache de idempotência em **Redis**; registros de contrato em **PostgreSQL**; snapshot de rotas/versões carregado em memória por réplica com invalidação por evento (`aios._platform.api.*`). |
| Réplicas | Escala horizontal pura por réplica atrás do balanceador L7 (YARP/LB), com anti-affinity por zona (`../028-Deployment/`). Sem afinidade obrigatória. |
| Concorrência de registro | OCC (`version`) em `RouteDefinition`/`ApiVersion`; mutações de registro são de baixa frequência (administrativas), sem necessidade de locks distribuídos. |
| Backpressure | `RateLimiter` rejeita cedo (`429`) por tenant/rota; `CircuitBreakerManager` isola upstream saturado; `SseStreamRelay` limita streams por réplica (`api.sse.max_streams_per_replica`) e sinaliza `503` acima do limite. |
| Caminho quente | AuthN com cache JWKS, AuthZ com cache de decisão TTL curto, rate limit em Redis (sub-ms); sem I/O bloqueante síncrono desnecessário no caminho `EdgeRouter`→upstream. Alvo p99 de overhead ≤ 5 ms (NFR-001). |
| SSE / conexões longas | Cada réplica multiplexa suas próprias conexões SSE assinando os subjects NATS relevantes por tenant sob demanda; sem afinidade de cliente-réplica exigida (qualquer réplica pode assumir a assinatura). |
| Rumo a milhões (de agentes, indiretamente) | O volume de requisições do Gateway escala com o número de **operadores/integrações** (humanos, CLIs, dashboards, SDKs) e com picos de administração (ex.: `spawn` em lote), não linearmente com o número de agentes ativos — a carga de execução de agentes trafega majoritariamente **dentro** do plano de controle (gRPC interno, NATS), não através deste Gateway. Ver `../001-Architecture/Architecture.md` §11. |

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada a este módulo: `ADR-0040`..`ADR-0049`** (regra 004×10,
> evitando colisão entre módulos). Registrar em `../002-ADR/`.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0040 | Escolha de gateway único (YARP) como entrada norte-sul vs. gateways por domínio/BFF. |
| ADR-0041 | Modelo de autorização em duas camadas: AuthZ grossa de rota no Gateway (PDP) vs. AuthZ fina de capacidade nos serviços de destino. |
| ADR-0042 | Ciclo de vida de `ApiVersion` (Draft/Published/Deprecated/Retired) e regra de coexistência ≥2 majors (RFC-0001 §5.7). |
| ADR-0043 | Registros globais de contrato (`RouteDefinition`, `ApiVersion`, catálogo de erros, schemas de evento, tipos de URN) como metadados de plataforma sem `tenant_id`/RLS. |
| ADR-0044 | Estratégia de rate limiting distribuído (token-bucket Redis) por tenant/rota e política `hard`/`soft`. |
| ADR-0045 | Agregação de contrato OpenAPI/proto dos módulos upstream em um contrato único versionado. |
| ADR-0046 | Transcodificação REST↔gRPC (gRPC-Web) para o Web Console. |
| ADR-0047 | Circuit breaker/bulkhead por serviço upstream e política de degradação graciosa. |
| ADR-0048 | Convenção de tenant reservado `_platform` para eventos e registros não escopados por tenant. |
| ADR-0049 | Domínio de erro `AIOS-API-*` reservado e regras de mapeamento de erros upstream não conformes. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida; §8 é operacionalizada por este módulo). |
| RFC-0040 | API Gateway Contract & Versioning Policy (regras completas de `/vN`, coexistência, depreciação, agregação de contrato). | A propor por este módulo. |
| RFC-0041 | Central Contract Registry Schema (esquema formal dos registros de URN types, subjects, error codes, event schemas). | A propor por este módulo. |

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN**: o Gateway é o **ponto de validação de AuthN de borda** — valida
  tokens OAuth2/OIDC (JWT) contra as chaves publicadas (JWKS) por
  `021-Security`, que permanece a **fonte da verdade** de identidade
  (emissão, rotação de chaves, revogação). Comunicação Gateway→upstream via
  **mTLS** (RFC-0001 §6).
- **AuthZ em duas camadas (defesa em profundidade)**: (1) o Gateway é **PEP
  grosso de rota**, consultando o **PDP** do `022-Policy` para decidir se
  tenant/role pode sequer chamar a rota (*default deny*); (2) o serviço de
  destino (ex.: Kernel `006`) permanece o **PEP fino de capacidade/recurso**.
  O Gateway **NÃO substitui** a autorização fina downstream — é uma camada
  adicional, não uma alternativa.
- **Isolamento de tenant**: o Gateway **NÃO DEVE** aceitar `X-AIOS-Tenant`
  divergente do `tenant` contido no token autenticado (RFC-0001 §6) →
  `AIOS-API-0016`. Upstreams internos **NÃO DEVEM** ser alcançáveis
  diretamente da rede pública — apenas via mTLS a partir do Gateway/malha
  interna (segmentação de rede, `028-Deployment`), servindo de backstop contra
  bypass do PEP de borda.

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no API Gateway | Mitigação |
|-----------------|-----------------------|-----------|
| **S**poofing | Forjar tenant/identidade via token adulterado ou reaproveitado. | Validação de assinatura JWT contra JWKS rotacionado; `X-AIOS-Tenant` casado com claim do token (`AIOS-API-0016`); mTLS interno. |
| **T**ampering | Alterar registro de rota/versão para desviar tráfego ou reativar versão retirada. | OCC (`version`), papel `api-registry-admin` exigido, RLS no estado por tenant, auditoria imutável (`025`) de toda mutação de registro. |
| **R**epudiation | Negar ter feito uma chamada de borda ou uma mutação de registro. | Logs de acesso + `GatewayTelemetry` correlacionados por `trace_id`; eventos `api.route.*`/`api.version.*` auditados via `025`. |
| **I**nformation disclosure | Vazamento de PII/segredos em `detail` de erro, logs de acesso ou payload agregado. | Envelope RFC-0001 §5.4 sem PII em `detail`; redação de `Authorization`/tokens nos logs; minimização (RFC-0001 §7). |
| **D**enial of service | Flood na borda pública (alvo primário de DoS por ser o único ponto de entrada). | Rate limiting por tenant/rota, limite de tamanho de corpo, circuit breaker/bulkhead por upstream, autoscaling de réplicas (`028`), proteção L7/edge na camada de infraestrutura (`027`). |
| **E**levation of privilege | Contornar o PEP de rota chamando o upstream diretamente, ou escalar role via payload manipulado. | Segmentação de rede (upstreams só aceitam mTLS do Gateway/malha interna); roles derivadas de claims assinadas, nunca de payload do cliente; PEP fino redundante no upstream. |

### 12.3 LGPD / GDPR

- **Minimização**: logs de acesso e eventos de borda **NÃO DEVEM** conter
  corpo de requisição/resposta com PII por padrão; apenas metadados
  (rota, status, latência, `trace_id`, `tenant_id`) são logados
  estruturadamente (RFC-0001 §7). Cabeçalhos sensíveis (`Authorization`,
  cookies) DEVEM ser redigidos nos logs.
- **Retenção**: registros de idempotência de borda têm TTL configurável
  (`api.idempotency.ttl_hours`, mínimo 24h); logs de acesso seguem política de
  retenção definida em `024-Observability`/`025-Audit`, não neste módulo.
- **Direito ao esquecimento**: o Gateway **NÃO É** o dono de dados pessoais —
  não persiste payloads de negócio; expurgo de dados de agente/usuário é
  coordenado pelos módulos donos (`010`, `025`) e não requer ação neste módulo
  além de invalidar caches de sessão associados ao tenant/usuário.
- **Segregação**: `tenant_id` + RLS no estado por tenant (`RateLimitPolicy`,
  `IdempotencyRecord`); CORS e política de origem configuráveis por tenant
  (`api.cors.allowed_origins`) para suportar requisitos de residência/soberania
  de dados.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../021-Security/`, `../022-Policy/`, `../020-Communication/`,
  `../024-Observability/`, `../025-Audit/`, `../027-Cluster/`, `../028-Deployment/`,
  `../030-CLI/`, `../031-SDK/`, `../032-WebConsole/`; segundo exemplo do padrão:
  `../006-Kernel/_DESIGN_BRIEF.md`, `../009-Scheduler/_DESIGN_BRIEF.md`.

*Fim do Design Brief interno do módulo 004-API. Este documento governa a geração
dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e este
brief é um defeito do documento gerado.*
