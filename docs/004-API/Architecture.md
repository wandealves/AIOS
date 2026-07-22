---
Documento: Architecture
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010 (globais); ADR-0040, ADR-0041, ADR-0044, ADR-0045, ADR-0046, ADR-0047 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0040 (API Gateway Contract & Versioning Policy, a propor); RFC-0041 (Central Contract Registry Schema, a propor)
Depende de: 001-Architecture, 021-Security, 022-Policy, 020-Communication, 024-Observability, 025-Audit, 027-Cluster, 028-Deployment
---

# 004-API — Arquitetura

> Este documento detalha a arquitetura interna do API Gateway em notação C4
> adaptada para ASCII. Ele **não redefine** contratos centrais (URN, envelope
> de evento, envelope de erro, idempotência, correlação, subjects) — estes são
> consumidos de `../003-RFC/RFC-0001-Architecture-Baseline.md`. Este documento
> é derivado de `./_DESIGN_BRIEF.md` (fonte única de verdade do módulo) e não
> pode contradizê-lo.

## Índice

1. Papel do Gateway na arquitetura global
2. C4 Nível 1 — Contexto do Sistema
3. C4 Nível 2 — Contêineres
4. C4 Nível 3 — Componentes
5. Tabela de responsabilidades por componente
6. Fronteiras e regras de dependência
7. Padrões arquiteturais adotados
8. Tecnologias e justificativas
9. Modelo de camadas e travessia de planos
10. Riscos arquiteturais e trade-offs
11. Alternativas descartadas
12. Referências e ADRs

---

## 1. Papel do Gateway na Arquitetura Global

O API Gateway (`004`) é o **único ponto de entrada norte-sul** do AIOS,
correspondendo à camada **L3 BORDA** do Modelo de Camadas
(`../001-Architecture/Architecture.md` §6). Ele é implementado sobre **YARP
(.NET 10)** e cumpre dois papéis simultâneos: (1) **proxy reverso
governado**, terminando REST/gRPC no *edge* e aplicando AuthN, AuthZ grossa,
rate limit, idempotência, validação de schema, versionamento e tradução de
erro antes de rotear a um upstream; e (2) **guardião dos registros centrais**
de RFC-0001 §8 (tipos de recurso URN, domínios de subject, catálogo de erros,
schemas de evento).

Diferente de módulos de domínio, o Gateway é **majoritariamente stateless**:
não persiste estado de negócio, apenas seus próprios registros de contrato e
o estado efêmero de borda (rate limit, idempotência). Isso o torna
horizontalmente escalável sem afinidade de réplica — propriedade central para
sustentar picos de tráfego externo sem se tornar o gargalo do sistema.

---

## 2. C4 — Nível 1: Contexto do Sistema

```
        CLI(030) / SDK(031) / WebConsole(032) / integrações A2A externas
                          │  HTTPS (REST) / gRPC-Web / SSE
                          ▼
        ┌──────────────────────────────────────────────────┐
        │              004-API (API GATEWAY)                │
        │        YARP · .NET 10 · fronteira norte-sul        │
        └───────┬──────────┬───────────┬──────────┬─────────┘
                │ mTLS     │ mTLS      │ mTLS      │ mTLS
       validação│ JWKS     │ decisão   │ eventos   │ REST/gRPC roteado
                ▼          ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐
        │021-Secur.│ │022-Policy│ │020-Comm. │ │ 006-019, 021-027    │
        │ (JWKS)   │ │  (PDP)   │ │ (NATS)   │ │ (upstreams roteados)│
        └──────────┘ └──────────┘ └──────────┘ └────────────────────┘
                          │
                          ▼ telemetria/audit
                ┌──────────────────────┐
                │ 024-Observability /   │
                │ 025-Audit             │
                └──────────────────────┘
```

**Atores e sistemas em fronteira**

| Ator/Sistema | Papel na interação com o Gateway | Protocolo |
|--------------|-------------------------------------|-----------|
| CLI (`030`) / SDK (`031`) / Web Console (`032`) | Clientes primários; toda chamada externa entra pelo Gateway. | HTTPS (REST), gRPC-Web, SSE |
| Integrações A2A externas | Agentes de terceiros que consomem a API pública versionada. | HTTPS (REST) |
| `021-Security` | Fonte de chaves JWKS para validação de token; fonte de eventos de rotação/revogação. | HTTPS (JWKS), NATS |
| `022-Policy` (PDP) | Autoridade de decisão consultada pelo `RouteAuthorizer` para toda rota `auth_required`. | gRPC |
| `020-Communication` (NATS/JetStream) | Transporte de eventos de registro de contrato e de streams SSE. | NATS/JetStream |
| `006`–`019`, `021`–`027` (upstreams) | Serviços do plano de controle donos do recurso; destino do roteamento. | gRPC/REST interno, mTLS |
| `024-Observability` / `025-Audit` | Destino de telemetria OTel e de sinais de segurança. | OTLP / gRPC |
| `027-Cluster` / `028-Deployment` | Topologia de réplicas, autoscaling, anti-affinity — operam o Gateway. | N/A (infraestrutura) |

---

## 3. C4 — Nível 2: Contêineres

> O Gateway é publicado como **um único serviço de borda** (stateless,
> escalado horizontalmente atrás de um balanceador L7), acompanhado dos
> armazenamentos que sustentam rate limit, idempotência e registros de
> contrato. Não há múltiplos contêineres de aplicação — a subdivisão interna é
> em **componentes** (Nível 3).

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        004-API — CONTÊINERES                                 │
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │      API Gateway Service (.NET 10 / YARP, stateless, N réplicas)   │      │
│  │  Expõe: REST /v1/... (externo) · gRPC aios.api.v1 (interno/console)│      │
│  │        · SSE /v1/.../events · gRPC-Web (Web Console)               │      │
│  └───────────┬───────────────────┬───────────────────┬────────────────┘      │
│              │                   │                   │                       │
│   ┌──────────▼─────────┐ ┌──────▼──────────┐ ┌───────▼──────────┐            │
│   │   PostgreSQL 16      │ │     Redis       │ │  NATS/JetStream   │            │
│   │  schema `api`         │ │ (rate limit     │ │ (API_REGISTRY,     │            │
│   │  (route, version,     │ │  token-bucket,  │ │  API_SECURITY       │            │
│   │  error_code,           │ │  idempotency    │ │  streams; SSE       │            │
│   │  event_schema,         │ │  quente,        │ │  relay)              │            │
│   │  resource_type,        │ │  decision cache │ │                      │            │
│   │  rate_limit_policy,    │ │  do PEP grosso) │ │                      │            │
│   │  idempotency_record)   │ └─────────────────┘ └──────────────────────┘            │
│   └────────────────────────┘                                                          │
│                                                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐    │
│   │  Sidecar de Telemetria: OTel Collector (traces/metrics) → 024;       │    │
│   │  Serilog → Seq (024/017-Logging); ambos fora do caminho crítico.     │    │
│   └─────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
```

| Contêiner | Responsabilidade | Persistência | Escala |
|-----------|-------------------|---------------|--------|
| API Gateway Service | Hospeda todos os componentes internos (§4); expõe REST/gRPC/SSE; stateless. | Nenhuma local (tudo externalizado). | Horizontal por CPU/conexões; sem afinidade obrigatória (§9 do brief). |
| PostgreSQL (`api` schema) | Fonte da verdade dos registros de contrato (`RouteDefinition`, `ApiVersion`, `ErrorCodeRegistryEntry`, `EventSchemaRegistryEntry`, `ResourceTypeRegistryEntry`) e do estado por tenant (`RateLimitPolicy`, `IdempotencyRecord`). | Durável; registros globais sem `tenant_id`; estado por tenant com RLS. | Réplicas de leitura + streaming replication (HA). |
| Redis | Token-bucket de rate limit, cache de idempotência quente, cache de decisão do PEP grosso, cache JWKS local. | Volátil (reconstruível a partir do PostgreSQL/`021`). | Cluster com réplicas. |
| NATS/JetStream | Transporte de eventos de registro (`API_REGISTRY`) e de segurança (`API_SECURITY`); backend de multiplexação SSE. | Streams duráveis `API_REGISTRY`, `API_SECURITY`. | Cluster NATS (`020`). |

---

## 4. C4 — Nível 3: Componentes

> Reproduzido e detalhado a partir de `./_DESIGN_BRIEF.md` §2.2. Este é o
> diagrama de componentes autoritativo do Gateway.

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

## 5. Tabela de Responsabilidades por Componente

| Componente | Responsabilidade primária | Colaboradores diretos | Dependências externas |
|------------|---------------------------|------------------------|-------------------------|
| **EdgeRouter** | Núcleo YARP: resolve `RouteDefinition` ativa por caminho/método/versão e encaminha ao *cluster* upstream correto; *front door* físico do Gateway. | VersionNegotiator, RouteRegistry (interno ao ContractRegistry), CircuitBreakerManager | Upstreams gRPC/REST |
| **AuthNFilter** | Valida JWT (assinatura, expiração, issuer) contra JWKS cacheado; popula claims (`tenant`, `sub`, `roles`) no contexto da requisição. | JwksCache (interno) | `021-Security` (JWKS) |
| **RouteAuthorizer** | PEP grosso de rota: monta `DecisionRequest` (tenant, role, rota) e consulta o PDP; cacheia decisões com TTL curto; *default deny*. | PolicyClient | `022-Policy` |
| **PolicyClient** | Cliente resiliente do PDP: gRPC com circuit breaker, cache de decisões, `fail_mode` configurável. | RouteAuthorizer | `022-Policy` |
| **RateLimiter** | Token-bucket atômico (Redis/Lua) por (tenant, rota); aplica `enforcement` hard/soft; sinaliza `AIOS-API-0004`. | IdempotencyRelay | Redis |
| **IdempotencyRelay** | Verifica/reserva `Idempotency-Key` na borda; devolve resultado cacheado em repetição; detecta reuse com payload divergente. | RateLimiter, SchemaValidator | Redis (quente) + PostgreSQL (durável) |
| **SchemaValidator** | Valida payload REST contra OpenAPI (JSON Schema) e mensagens gRPC contra descritor proto do contrato ativo. | ContractRegistry | — |
| **VersionNegotiator** | Resolve `/vN` + `X-AIOS-Api-Version` para o `major` correto conforme estado de `ApiVersion` (`StateMachine.md`); aplica `Deprecation`/`Sunset` headers. | ContractRegistry (ApiVersionStore) | — |
| **ErrorTranslator** | Normaliza qualquer erro (interno ou de upstream) no envelope RFC-0001 §5.4; mapeia `google.rpc.Status` (gRPC) para o mesmo `code`. | ContractRegistry (ErrorCodeRegistry) | — |
| **CorrelationPropagator** | Garante/gera `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` em toda chamada (RFC-0001 §5.6). | GatewayTelemetry | — |
| **GrpcWebBridge** | Transcodifica REST↔gRPC (gRPC-Web) para consumidores web (`032-WebConsole`), preservando o contrato proto. | EdgeRouter | — |
| **SseStreamRelay** | Mantém streams SSE/long-poll por cliente, multiplexando a assinatura de subjects NATS relevantes por tenant/tarefa. | GatewayTelemetry | NATS/JetStream |
| **ContractRegistry** | Mantém os registros de RFC-0001 §8: tipos de recurso URN, domínios de subject, catálogo de erros, versões de `dataschema`; expõe API de consulta/registro governada. | OpenApiAggregator | PostgreSQL |
| **OpenApiAggregator** | Agrega specs OpenAPI/proto publicadas pelos módulos upstream em um contrato único versionado, servido em `/v1/api/openapi.json`. | ContractRegistry | Consome `<module>.contract.published` |
| **CircuitBreakerManager** | Circuit breaker + bulkhead por serviço upstream; isola falhas, evita cascata; expõe estado a `EdgeRouter`. | EdgeRouter | Upstreams |
| **GatewayTelemetry** | Instrumentação OTel transversal (spans por request, métricas `aios_api_*`, logs Serilog→Seq) e emissão de sinais de segurança a `025-Audit`. | Todos os componentes acima (cross-cutting) | `024-Observability`, `025-Audit` |

---

## 6. Fronteiras e Regras de Dependência

- O Gateway **DEVE** tratar `021-Security` e `022-Policy` como **serviços
  externos consultados**, nunca como bibliotecas internas — reforça NR-03 do
  brief (o Gateway consome JWKS e decisões, não emite tokens nem decide
  política).
- O Gateway **NÃO DEVE** implementar lógica de negócio de qualquer módulo
  upstream — o `EdgeRouter` **DEVE** apenas resolver a `RouteDefinition` e
  encaminhar (NR-01, R-01).
- O Gateway **NÃO DEVE** decidir autorização fina de capacidade/recurso; essa
  decisão pertence ao PEP local do serviço de destino (NR-02, R-03).
- Toda dependência externa do Gateway (`021`, `022`, upstreams,
  `020-Communication`) **DEVE** estar isolada por *circuit breaker* e
  *bulkhead* dedicados, para que a falha de uma dependência não degrade as
  demais (Padrão Bulkhead, §7; NFR-012).
- O Gateway **DEVE** ser *stateless* ao nível do processo: todo estado que
  sobrevive a um restart de réplica **DEVE** residir em PostgreSQL/Redis/NATS,
  nunca em memória de processo não externalizada.
- Upstreams internos **NÃO DEVEM** ser alcançáveis diretamente da rede
  pública — apenas via mTLS a partir do Gateway/malha interna (segmentação de
  rede, `../028-Deployment/`), servindo de *backstop* contra *bypass* do PEP
  de borda.

---

## 7. Padrões Arquiteturais Adotados

| Padrão | Onde é aplicado | Motivação |
|--------|-------------------|-----------|
| **API Gateway / Edge Proxy** | `EdgeRouter` (YARP). | Ponto único de entrada norte-sul; centraliza *cross-cutting concerns* de borda (ADR-0040). |
| **PEP/PDP (Policy Enforcement/Decision Point)** | `RouteAuthorizer` (PEP de rota) consulta `022-Policy` (PDP) via `PolicyClient`. | Separação entre aplicação e decisão de política (ADR-0008, ADR-0041); *default deny* uniforme. |
| **Autorização em duas camadas** | `RouteAuthorizer` (grossa, borda) + PEP fino no serviço de destino. | Defesa em profundidade sem duplicar a lógica de capability nos módulos (ADR-0041). |
| **Circuit Breaker** | `CircuitBreakerManager`, `PolicyClient`. | Isola falha de dependência externa; evita cascata (ADR-0047). |
| **Bulkhead** | Pools/limites dedicados por serviço upstream dentro do `CircuitBreakerManager`. | Um upstream lento não esgota recursos usados por outro (NFR-012). |
| **Token-bucket atômico (Lua/Redis)** | `RateLimiter`. | Enforcement de rate limit sob alta concorrência sem *round-trip* ao PostgreSQL a cada requisição (ADR-0044). |
| **Idempotency-Key + cache de resultado** | `IdempotencyRelay`. | Efeito único sobre repetição de mutação de cliente (RFC-0001 §5.5). |
| **Registro central de contrato (Contract Registry)** | `ContractRegistry`, `OpenApiAggregator`. | Consistência de URN/subjects/erros/schemas entre todos os módulos (RFC-0001 §8, ADR-0045). |
| **Máquina de estados versionada (Semantic Versioning por FSM)** | `VersionNegotiator`, `ApiVersion` (ver `StateMachine.md`). | Coexistência governada de majors com depreciação previsível (ADR-0042). |
| **Transcodificação de protocolo (gRPC-Web)** | `GrpcWebBridge`. | Web Console consome serviços gRPC internos sem cliente gRPC nativo no browser (ADR-0046). |
| **Multiplexação de streaming (SSE relay)** | `SseStreamRelay`. | Entrega de eventos em tempo real ao cliente sem exigir cliente NATS nativo (R-12). |

---

## 8. Tecnologias e Justificativas

| Tecnologia | Uso no Gateway | Justificativa | Alternativa descartada |
|------------|-----------------|----------------|--------------------------|
| **.NET 10 / YARP** | Runtime e núcleo de proxy reverso do Gateway. | *Reverse proxy* de alto desempenho nativo do ecossistema .NET, extensível via middlewares tipados; alinhado ao control plane do AIOS (ADR-0003, ADR-0040). | NGINX/Envoy puro (menor integração tipada com o restante do control plane .NET); Kong (introduziria um segundo *plugin runtime* fora do padrão AIOS). |
| **gRPC** (`aios.api.v1`) | Superfície de administração/registro interna e comunicação Gateway→upstream. | Contrato fortemente tipado, baixa latência — essencial ao caminho quente (NFR-001/002). | REST interno (overhead de serialização/latência maior). |
| **REST** (`/v1/api/...`) | Superfície de administração/registro externa e *pass-through* para clientes REST. | Interoperabilidade universal com SDKs/CLIs/parceiros; RFC-0001 §5.7 exige versionamento por caminho. | GraphQL (complexidade desnecessária para uma superfície majoritariamente *pass-through*). |
| **PostgreSQL 16** | Fonte da verdade dos registros de contrato e do estado por tenant. | Transações ACID, RLS nativo por `tenant_id` para o estado por tenant, maturidade operacional (ADR-0005). | MongoDB (sem RLS nativo equivalente). |
| **Redis** | Token-bucket de rate limit, cache de idempotência quente, cache de decisão do PEP, cache JWKS. | Latência sub-ms, scripts Lua atômicos, TTL nativo (ADR-0006). | Memcached (sem scripts atômicos, sem estruturas ricas). |
| **NATS/JetStream** | Transporte de eventos de registro/segurança; backend de multiplexação SSE. | Baixa latência, *streams* duráveis, alinhado ao barramento único do AIOS (ADR-0004). | Kafka (operação mais pesada; NATS já é o barramento padrão). |
| **OpenTelemetry + Prometheus + Grafana + Serilog/Seq** | `GatewayTelemetry`: spans por request, métricas `aios_api_*`, logs correlacionados. | Padrão transversal do AIOS (ADR-0010); correlação `trace_id`/`tenant_id` obrigatória (RFC-0001 §5.6). | APM proprietário (vendor lock-in). |

---

## 9. Modelo de Camadas e Travessia de Planos

```
   MUNDO EXTERNO                  L3 BORDA (004-API)                  L2 PLANO DE CONTROLE
 ┌───────────────────┐  REST/gRPC/SSE ┌───────────────────────────┐  gRPC/REST mTLS  ┌─────────────────────┐
 │ CLI/SDK/Console/   │───────────────▶│         004-API             │─────────────────▶│ 006-Kernel, 009-Sched,│
 │ integrações A2A    │◀───────────────│ (AuthN+AuthZ grossa+       │◀─────────────────│ 021-Security,        │
 │  externas           │  resultado/    │  rate limit+idempotência+  │   resultado        │ 022-Policy, ...       │
 └───────────────────┘  erro          │  versionamento+tradução)   │                    └─────────────────────┘
                                        └──────────┬────────────────┘
                                                    │ registros de contrato
                                                    ▼
                                        ┌─────────────────────┐
                                        │ PostgreSQL / Redis /  │
                                        │ NATS (armazenamento   │
                                        │ de borda)              │
                                        └─────────────────────┘
```

O Gateway é a **única travessia legítima** entre o mundo externo e o plano de
controle governado. Essa travessia única é o que torna o *AuthN de borda*, o
*rate limiting* e a *tradução de erro* propriedades universais e auditáveis
(R-02, R-04, R-08).

---

## 10. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Mitigação |
|-------------------|-----------|-----------|
| Gateway como ponto único de travessia norte-sul | Toda requisição externa passa pelo Gateway — risco de gargalo de throughput e alvo de DoS. | *Stateless* + escala horizontal + caminho quente sem I/O bloqueante desnecessário (NFR-002: ≥ 150.000 req/s por réplica). |
| Latência adicional do PEP grosso em cada rota `auth_required` | Consulta ao PDP soma latência ao caminho crítico. | Cache de decisões com TTL curto (`api.gateway.*`); meta p99 de overhead ≤ 5 ms (NFR-001). |
| Acoplamento à disponibilidade de `021`/`022` no caminho de AuthN/AuthZ | Falha externa pode bloquear todo o tráfego externo. | `fail_mode=closed` configurável + cache JWKS/decisão + circuit breaker (§9 do brief). |
| Overshoot de rate limit sob alta concorrência | Token-bucket em Redis pode, sob corrida extrema, permitir pequeno excesso. | Meta de overshoot ≤ 1% (NFR-007); reconciliação periódica. |
| Contrato agregado (`OpenApiAggregator`) desatualizado em relação aos módulos | Cliente vê contrato obsoleto por até `contract_sync_interval_s`. | Meta `aios_api_contract_sync_lag_s` ≤ 60 s (NFR-011); re-sync sob evento `<module>.contract.published`. |

---

## 11. Alternativas Descartadas

| Alternativa | Por que foi descartada |
|-------------|--------------------------|
| Gateway por domínio/BFF (um gateway por módulo ou por consumidor) | Fragmentaria AuthN/AuthZ de borda e os registros centrais de contrato; multiplicaria a superfície de ataque norte-sul (ver §2.1 de `Vision.md`, ADR-0040). |
| Gateway decidir autorização fina de capacidade/recurso localmente | Duplicaria lógica de PEP fino já existente nos módulos de destino; violaria a separação de responsabilidades (NR-02, ADR-0041). |
| Kafka como barramento de eventos do Gateway | Introduziria um segundo barramento além do NATS já padronizado (ADR-0004), sem ganho proporcional. |
| Gateway com estado local (afinidade de réplica para SSE/rate limit) | Impediria escala horizontal simples e complicaria o *failover*; substituído por estado externalizado em Redis/NATS (§10 do brief). |
| REST-only (sem gRPC interno) | Aumentaria a latência de comunicação Gateway→upstream no caminho quente e perderia tipagem forte do contrato interno. |

---

## 12. Referências e ADRs

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- ADRs globais consumidas: `../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md`,
  `../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md`,
  `../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md`,
  `../002-ADR/ADR-0004-NATS-como-Barramento.md`,
  `../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md`,
  `../002-ADR/ADR-0006-Redis-Estado-Quente.md`,
  `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`,
  `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- ADRs do módulo (a propor, faixa `ADR-0040`–`ADR-0049`): ver `./ADR.md` para o
  índice detalhado.
- RFCs do módulo (a propor): ver `./RFC.md`.
- Componentes acoplados: `../021-Security/`, `../022-Policy/`,
  `../020-Communication/`, `../024-Observability/`, `../025-Audit/`,
  `../027-Cluster/`, `../028-Deployment/`, `../030-CLI/`, `../031-SDK/`,
  `../032-WebConsole/`.

*Fim do documento `Architecture.md` do módulo 004-API.*
