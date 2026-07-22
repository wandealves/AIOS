---
Documento: FailureRecovery
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0043, ADR-0044, ADR-0047
RFCs relacionados: RFC-0001 (§5.5)
Depende de: ./_DESIGN_BRIEF.md §9, ./NonFunctionalRequirements.md, ./Deployment.md
---

# 004-API — Modos de Falha e Recuperação

> Reproduz e detalha `./_DESIGN_BRIEF.md` §9 no formato FMEA (*Failure Mode
> and Effects Analysis*). Metas de RTO/RPO são **idênticas** a NFR-006.

## 1. FMEA — Modos de falha, detecção, isolamento e recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| `021-Security` (JWKS) indisponível | Timeout no `AuthNFilter`. | Bulkhead de AuthN. | Usa cache JWKS até `api.auth.jwks_cache_ttl_s`; expirado → `api.auth.fail_mode=closed` nega (`AIOS-API-0002`). | Retry idempotente após religar. |
| `022-Policy` (PDP) indisponível | Timeout/CB no `PolicyClient`. | Bulkhead do `RouteAuthorizer`. | `fail_mode=closed` → nega (`AIOS-API-0003`); alerta operacional. | Retry idempotente pós meia-abertura do CB. |
| Upstream (qualquer módulo) indisponível | Circuit breaker por serviço. | Isolamento por `CircuitBreakerManager` (bulkhead por *cluster*). | `AIOS-API-0005`; outras rotas não afetadas (NFR-012). | Retry idempotente com *backoff* no cliente. |
| Redis (rate limit/idempotência) indisponível | *Health probe*. | Modo *fail-safe* conservador. | Modo degradado: rejeita rajadas acima do *default* estático; idempotência cai para PostgreSQL (mais lento, sem perda). | Sem duplo efeito (verificação dupla Redis+PG). |
| PostgreSQL (`ContractRegistry`) indisponível | Timeout. | Rotas seguem servidas do **snapshot em memória** (último carregado). | Leitura de rotas continua; mutações de registro ficam bloqueadas até religar. | N/A (somente leitura afetada). |
| NATS/JetStream indisponível | *Health* do `EventPublisher`/Outbox relay. | Outbox retém eventos de registro. | Backlog drenado ao religar; SSE degrada para *long-poll*. | Ordenado por *stream*; *dedupe* por `event.id`. |
| Crash de réplica do Gateway | *Health*/*readiness probe*. | Réplica *stateless* removida do balanceador. | Nova réplica sobe e recarrega *snapshot* em < 1 min. | N/A (sem estado local a recuperar). |
| Contrato agregado desatualizado | `aios_api_contract_sync_lag_s` acima do limiar. | N/A (degradação, não falha total). | Serve última versão válida com aviso; força re-*sync*. | N/A. |
| Loop de rejeição por PDP flutuante | Métrica de taxa de *deny*. | Cache de decisão evita martelar o PDP. | Cache com TTL curto e *jitter* evita *thundering herd*. | Idempotente por natureza. |

## 2. Metas de recuperação (RTO/RPO)

| Componente | RTO | RPO |
|------------|-----|-----|
| Réplica do Gateway (stateless) | < **1 min** | N/A (sem estado local) |
| `ContractRegistry` (PostgreSQL) | ≤ **10 min** | ≤ **5 min** |
| Consistência global do módulo | ≤ **5 min** (agregado, NFR-006) | ≤ **5 min** |

Estes valores são **idênticos** a NFR-006 de `./NonFunctionalRequirements.md`
e ao V-R10 da Visão Global (`../000-Vision/Vision.md`).

## 3. Degradação graciosa — princípio normativo

Sob indisponibilidade de `021-Security`/`022-Policy`, o Gateway **DEVE**
priorizar **negar com segurança** (*default deny*, `fail_mode=closed`) em
vez de rotear sem autenticação/autorização. Esta é uma decisão de segurança
estrutural, não uma degradação de conveniência — reverter para
`fail_mode=open` **NÃO DEVE** ocorrer automaticamente em nenhuma
circunstância (ver `./Security.md` §1.1).

## 4. Retry e *backoff*

- Retries do **cliente** sobre erros `retriable=true` (`AIOS-API-0004`,
  `AIOS-API-0005`, `AIOS-API-0006`, `AIOS-API-0014`) **DEVEM** usar
  *backoff* exponencial com *jitter*, respeitando `Retry-After` quando
  presente.
- Retries **internos** do Gateway para dependências (`021`, `022`,
  upstreams) **DEVEM** ser limitados por *circuit breaker* — nunca retry
  ilimitado que amplifique uma falha em cascata.

## 5. DLQ (Dead-Letter Queue)

Eventos de registro que falham repetidamente na publicação (após esgotar
retries do relay do Outbox) **DEVEM** ser movidos a uma fila de
*dead-letter* dedicada (`API_REGISTRY_DLQ`, gerida via `../020-Communication/`)
para investigação manual, nunca descartados silenciosamente — preserva
NFR-005 (perda de evento de registro = 0).

## 6. DR Drill (validação periódica)

O `ContractRegistry` (PostgreSQL) **DEVE** ser incluído no calendário de
*disaster recovery drills* da plataforma (`../028-Deployment/`), validando
periodicamente que o RTO ≤ 10 min / RPO ≤ 5 min de §2 é alcançável na
prática, não apenas na especificação.

*Fim de `FailureRecovery.md`.*
