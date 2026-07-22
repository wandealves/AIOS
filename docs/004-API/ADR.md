---
Documento: ADR
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0042, ADR-0043, ADR-0044, ADR-0045, ADR-0046, ADR-0047, ADR-0048, ADR-0049
RFCs relacionados: RFC-0001
Depende de: ../002-ADR/README.md
---

# 004-API — Índice de ADRs

> Registro imutável de decisões arquiteturais que afetam este módulo. ADRs
> globais são consumidas (não redefinidas); ADRs do módulo estão na faixa
> reservada **`ADR-0040`–`ADR-0049`** (regra `004×10`, evitando colisão
> entre módulos — ver `./_DESIGN_BRIEF.md` §11). Todas as ADRs do módulo
> estão em estado **a propor**: nenhuma foi ainda registrada em
> `../002-ADR/`; este índice antecipa a numeração reservada para que os 26
> documentos possam referenciá-las de forma consistente.

## 1. ADRs globais consumidas por este módulo

| ADR | Título | Status | Relação com o 004-API |
|-----|--------|--------|---------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O Gateway é a fronteira de "syscall externa" do SO — consistente com a analogia condutora. |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + *split* control/data plane | Accepted | O Gateway pertence exclusivamente ao *control plane*, camada L3 BORDA. |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no *runtime* | Accepted | Justifica .NET 10/YARP como *stack* do Gateway. |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | Base do transporte de eventos de registro/segurança e do relay SSE. |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | Base do `ContractRegistry` (sem uso de pgvector/AGE neste módulo). |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e *locks* | Accepted | Base do `RateLimiter`/`IdempotencyRelay`/cache de decisão e JWKS. |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, *default deny* | Accepted | Base normativa do `RouteAuthorizer` como PEP de borda. |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | Base do `GatewayTelemetry` e dos sinais de segurança emitidos a `025-Audit`. |

## 2. ADRs do módulo (a propor, faixa `ADR-0040`–`ADR-0049`)

| ADR | Título proposto | Status | Decisão (1 linha) |
|-----|-------------------|--------|-----------------------|
| ADR-0040 | Escolha de gateway único (YARP) como entrada norte-sul vs. gateways por domínio/BFF | A propor | Um único API Gateway (.NET 10/YARP) centraliza *cross-cutting concerns* de borda em vez de N *gateways* por domínio. |
| ADR-0041 | Modelo de autorização em duas camadas: AuthZ grossa de rota no Gateway (PDP) vs. AuthZ fina de capacidade nos serviços de destino | A propor | O Gateway é PEP grosso de rota; a autorização fina permanece no PEP local de cada módulo. |
| ADR-0042 | Ciclo de vida de `ApiVersion` (`Draft`/`Published`/`Deprecated`/`Retired`) e regra de coexistência ≥ 2 majors | A propor | FSM formal de versionamento operacionalizando RFC-0001 §5.7. |
| ADR-0043 | Registros globais de contrato (`RouteDefinition`, `ApiVersion`, catálogo de erros, *schemas* de evento, tipos de URN) como metadados de plataforma sem `tenant_id`/RLS | A propor | Registros de RFC-0001 §8 são globais, não multi-tenant; escrita restrita ao papel `api-registry-admin`. |
| ADR-0044 | Estratégia de *rate limiting* distribuído (*token-bucket* Redis) por tenant/rota e política `hard`/`soft` | A propor | *Token-bucket* atômico via *script* Lua no Redis, com dois modos de aplicação. |
| ADR-0045 | Agregação de contrato OpenAPI/proto dos módulos upstream em um contrato único versionado | A propor | `OpenApiAggregator` compõe um único documento de contrato consumido por SDK/CLI/parceiros. |
| ADR-0046 | Transcodificação REST↔gRPC (gRPC-Web) para o Web Console | A propor | `GrpcWebBridge` permite ao `032-WebConsole` consumir serviços gRPC internos sem cliente nativo. |
| ADR-0047 | *Circuit breaker*/*bulkhead* por serviço upstream e política de degradação graciosa | A propor | Isolamento de falha por upstream via `CircuitBreakerManager`, garantindo NFR-012. |
| ADR-0048 | Convenção de tenant reservado `_platform` para eventos e registros não escopados por tenant | A propor | Eventos de registro de contrato usam `tenant=_platform` em vez de omitir o campo. |
| ADR-0049 | Domínio de erro `AIOS-API-*` reservado e regras de mapeamento de erros upstream não conformes | A propor | Reserva formal do domínio `API` (faixa `0001`-`0099`) e da regra de tradução de erro não conforme (`AIOS-API-0014`). |

## 3. Como as ADRs deste módulo serão registradas

Seguindo `../002-ADR/README.md` §"Como escrever uma ADR": cada ADR do
módulo, quando formalmente registrada em `../002-ADR/`, **DEVE** conter
Contexto · Problema · Alternativas · Análise · Escolha · Consequências ·
Riscos · Trade-offs, e **DEVE** referenciar de volta o
`./_DESIGN_BRIEF.md` e os documentos deste módulo que a motivaram (em
particular `./Architecture.md` §11 "Alternativas Descartadas"). Este
`ADR.md` **DEVE** ser atualizado para apontar ao arquivo real assim que
cada ADR for aceita (`Status: Accepted`).

*Fim de `ADR.md`.*
