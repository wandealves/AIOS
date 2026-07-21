---
Documento: Security
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0111, ADR-0112, ADR-0117, ADR-0119
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 003-RFC (RFC-0001), 005-Database, 021-Security, 022-Policy, 025-Audit, 040-Glossary
---

# 011-Context — Security

> Decisões de segurança do `ContextService`. **Deriva** da §12 (e §1.2, §3, §9)
> do `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**. Os contratos de identidade,
> correlação, mTLS e privacidade são **reutilizados** de
> [RFC-0001 §5.6/§6/§7](../003-RFC/RFC-0001-Architecture-Baseline.md) — aqui
> apenas os aplicamos ao domínio de janela de contexto e cache semântico.
> Palavras normativas conforme RFC 2119/8174.

## 1. AuthN / AuthZ

### 1.1 Autenticação (AuthN)

- Tokens **OAuth2/OIDC** DEVEM ser validados no `ContextApiGateway`/Gateway YARP
  ([021-Security](../021-Security/Security.md)); as claims assinadas (incluindo
  `tenant` e `agent`) são propagadas ao `ContextService` (RFC-0001 §6).
- A comunicação **interna** entre serviços (`ModelRouterClient`→`017`,
  `MemoryClient`→`010`, `ContextPolicyGuard`→`022`, `EventPublisher`→NATS) DEVE
  usar **mTLS** (RFC-0001 §6); o `ContextService` NÃO DEVE aceitar tráfego não
  autenticado no plano de controle.
- Cabeçalhos de correlação/identidade obrigatórios em toda chamada externa:
  `Authorization`, `X-AIOS-Tenant`, `traceparent`, `Idempotency-Key` (mutações) —
  ver [API.md](./API.md) e RFC-0001 §5.6.

### 1.2 Autorização (AuthZ) — PEP/PDP

- O `ContextPolicyGuard` (Policy Enforcement Point) DEVE consultar o **PDP** de
  [022-Policy](../022-Policy/API.md) para toda operação **privilegiada**: montar
  contexto (`assemble`) para outro agente, criar/atualizar `BudgetProfile`
  (`PUT /v1/context/budgets/{scope}`) e invalidação em massa de cache
  (`POST /v1/context/cache/invalidate`) — sob **default deny** (RFC-0001 §5.8;
  brief §12.1).
- Operações de leitura do próprio escopo (`assemble` para o próprio agente,
  `GET /v1/context/bundles/{bundle_id}` do próprio bundle,
  `POST /v1/context/tokens/estimate`) PODEM ser autorizadas por *fast path* de
  claim (agente == dono do recurso) sem round-trip síncrono ao PDP a cada
  chamada, desde que a decisão de política aplicável esteja em cache local com
  invalidação por `aios.<tenant>.policy.decision.updated` (brief §6.2) — a
  decisão em si continua sendo do PDP, nunca do `011`.
- Toda operação privilegiada DEVE emitir registro de auditoria imutável para
  [025-Audit](../025-Audit/Events.md).
- Autorização negada retorna `AIOS-CTX-0011` (403, `TenantMismatch`) quando o
  motivo é divergência de tenant, ou o código de negação equivalente do PDP
  (`022`) propagado como 403 nos demais casos.

```
 Requisição autenticada (claims: tenant, agent, roles)
        │
   ┌────▼──────────────┐   decision request   ┌──────────────┐
   │ ContextPolicyGuard │─────────────────────▶│  PDP (022)   │  RBAC/ABAC · default deny
   │ (PEP)              │◀─────────────────────│              │
   └────┬────────────────┘  allow/deny + obligs └──────────────┘
        │ allow           │ deny → 403 (código do PDP)
        │ tenant guard (token.tenant == X-AIOS-Tenant) → mismatch: AIOS-CTX-0011
        ▼
   comando validado → ContextAssembler   ── auditoria (025) em toda operação privilegiada
```

### 1.3 Isolamento de tenant

- O `tenant` do token autenticado DEVE coincidir com `X-AIOS-Tenant`; divergência
  ⇒ `AIOS-CTX-0011` (403) — invariante da máquina de estado `RECEIVED→REJECTED`
  (brief §4.1).
- **Row-Level Security (RLS)** é OBRIGATÓRIA em toda tabela do módulo
  (`ContextBundle`, `ContextFragment`, `SemanticCacheEntry`, `BudgetProfile`,
  `SummaryNode` — brief §3; [005-Database](../005-Database/Security.md));
  `tenant_id` é a fronteira de isolamento, inclusive no **namespace de cache**
  (chaves Redis e entradas `pgvector` sempre escopadas por `tenant_id`).
- O namespace NATS é isolado por tenant (`aios.<tenant>.context.*`, RFC-0001
  §5.3).
- Acesso cruzado agente↔agente ao contexto de outro agente exige capability
  explícita, resolvida via `ContextPolicyGuard`/PDP — nunca decidida
  localmente pelo `011` (brief §1.3 N-06).

### 1.4 Modelo de permissões (referência)

| Papel/atributo | assemble (próprio) | assemble (outro agente) | compress | cache lookup/store | cache invalidate | budget upsert |
|----------------|:-------------------:|:------------------------:|:--------:|:-------------------:|:-----------------:|:-------------:|
| agent (próprio escopo) | ✔ | — | ✔ | ✔ | — | — |
| service `007-Agent-Runtime`/`006-Kernel` (em nome do agente) | ✔ | — | ✔ | ✔ | — | — |
| service `012-Planning` | ✔ (para replan) | — | ✔ | ✔ | — | — |
| operador tenant-admin | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| controlador de privacidade (RTBF/expurgo LGPD) | — | — | — | — | ✔ (por `source_urn`) | — |

> A matriz acima é **indicativa**; a decisão efetiva é sempre do PDP (`022`) —
> o `011` não decide autorização localmente (RFC-0001 §5.8; brief §1.3 N-06).

## 2. Threat Model — STRIDE

Derivado literalmente da §12.2 do brief; nenhuma mitigação nova é introduzida
aqui além do que o brief prevê.

| Ameaça | Vetor no `011` | Mitigação | Controle rastreável |
|--------|-----------------|-----------|----------------------|
| **S**poofing | Requisição com `tenant` forjado no payload/claim. | mTLS + validação de claims no Gateway; check `tenant == X-AIOS-Tenant`. | `AIOS-CTX-0011`; auditoria (025). |
| **T**ampering | Alteração de `ContextBundle`/`SemanticCacheEntry` em trânsito ou em repouso. | mTLS interno; imutabilidade de eventos (outbox); `ContextFragment.embedding` e `content_ref` protegidos por RLS. | Hash de payload no envelope de evento; `trace_id`. |
| **R**epudiation | Negar ter feito `assemble`/invalidação de cache. | Eventos auditáveis (`025`) com `trace_id`, `actor`, `event.id`; imutabilidade do outbox. | `context.window.assembled`/`cache.invalidated` (ver [Events.md](./Events.md)). |
| **I**nformation disclosure | Vazamento de contexto/resposta cacheada entre tenants via `SemanticCacheEntry`. | RLS por `tenant_id` + namespace de cache por tenant (Redis/pgvector) + `similarity_threshold` estrito; teste de fuzz de tenant. | NFR-014 (vazamento = 0); NFR-011 (falso-hit ≤ 0,5%). |
| **D**enial of service | Flood de `assemble`/`cache lookup`; explosão de cache-fill concorrente na mesma chave. | Rate-limit por tenant (`context.ratelimit.assemble_rps_per_tenant`), *single-flight* de cache-fill (lock Redis `SET NX PX`), *bulkhead*/circuit breaker em `ModelRouterClient`/`MemoryClient`. | `ADR-0111`; NFR-009 (throughput ≥ 500 req/s/réplica). |
| **E**levation of privilege | Uso de `assemble` para outro agente, upsert de `BudgetProfile` ou invalidação em massa sem permissão. | PEP/PDP *default deny*; capabilities via `021`/`022`. | `AIOS-CTX-0011` / código de negação do PDP. |

**Superfície de ataque** do módulo: endpoints REST/gRPC (`/v1/context/*`, via
Gateway), assinaturas NATS consumidas (brief §6.2:
`memory.item.consolidated/deleted`, `model.registry.updated`,
`knowledge.node.updated`, `agent.lifecycle.suspended/terminated`,
`policy.decision.updated`), e os backends de dados (Redis/PostgreSQL/MinIO),
acessíveis **apenas** internamente por mTLS/RLS — nunca diretamente pelo Agent
Runtime ou por outro módulo (brief §1.2, §2.1).

## 3. Segredos, TLS/mTLS e criptografia

| Segredo | Uso | Gestão |
|---------|-----|--------|
| DSN PostgreSQL | fonte da verdade de `ContextBundle`/`ContextFragment`/`SemanticCacheEntry`/`BudgetProfile`/`SummaryNode` | cofre de segredos (021); injetado por `secretRef` (ver [Deployment.md](./Deployment.md)); nunca em chave `context.*` de domínio. |
| Credenciais Redis | cache L1, locks de *single-flight*, `IdempotencyStore` | idem cofre; rotação periódica. |
| Credenciais MinIO | *offload* de fragmentos/blobs grandes (`content_ref`) | idem cofre; políticas de bucket restritas ao serviço. |
| Credenciais NATS | eventos `context.window.*`/`context.cache.*`/`context.budget.*` | conta NATS por tenant; token/nkey no cofre. |
| Certificados mTLS | comunicação interna (`010`, `017`, `022`, `025`) | emitidos/rotacionados pela PKI de `021`. |

Regras:
- Segredos NÃO DEVEM ser passados por variáveis `AIOS_CONTEXT_*` de domínio nem
  registrados em log (ver [Configuration.md](./Configuration.md)).
- **TLS externo** terminado no Gateway (YARP); **mTLS** obrigatório entre
  serviços internos, incluindo chamadas a `017-Model-Router` (embedding/
  sumarização) e `010-Memory` (recall seletivo).
- **Criptografia em repouso**: dados duráveis (PostgreSQL/MinIO) DEVEM usar
  criptografia de volume/bucket conforme [005-Database](../005-Database/Security.md)
  e `027-Cluster`; `embedding` (`vector(1536)`) e `content_ref`/`content_inline`
  seguem a mesma política de repouso do restante do dado do módulo.
- Envelopes de erro NÃO DEVEM vazar dados sensíveis em `detail` (RFC-0001 §6) —
  fragmentos/prompts nunca aparecem em texto claro no envelope de erro
  `AIOS-CTX-<NNNN>`.

## 4. LGPD / GDPR

Derivado da §12.3 do brief e RFC-0001 §7.

- **Minimização de dados:** payloads de eventos (`context.window.*`,
  `context.cache.*`) e logs carregam apenas identificadores/URN, contadores de
  tokens e métricas de compressão; conteúdo textual sensível trafega apenas por
  `content_ref` (MinIO, acesso controlado) ou `content_inline` (sob RLS), **nunca**
  em `detail` de erro ou em `data` de evento (RFC-0001 §6/§7).
- **Herança de base legal:** fragmentos derivados de memória (`source_kind =
  memory`) herdam os metadados de base legal e classe de retenção definidos pelo
  `010-Memory`; o `011-Context` não define base legal própria — apenas propaga.
- **Efemeridade por padrão:** `ContextBundle` e `SummaryNode` são efêmeros
  (`expires_at`), consistente com a fronteira do brief §1.2 (a sumarização feita
  aqui é *efêmera e por chamada*; sumários reutilizáveis de longo prazo
  pertencem ao `010-Memory`, nunca são promovidos a memória canônica pelo `011`).
- **Direito ao esquecimento:** o expurgo rastreável (FR-012) é disparado por
  `aios.<tenant>.memory.item.deleted` (invalidação/expurgo automático de cache +
  `SummaryNode` derivados) ou por `POST /v1/context/cache/invalidate` (invocação
  manual por `scope`/`source_urn`). Toda invalidação emite
  `context.cache.invalidated`/`context.cache.evicted` + auditoria (`025`).
- **PII no cache semântico:** entradas de `SemanticCacheEntry` com origem em
  fragmentos marcados `pii` (herdado do `010`) DEVEM usar `ttl_seconds` reduzido
  e permanecer expurgáveis; o `prompt_embedding` é tratado como dado pessoal
  derivado — não é reversível a texto claro, mas é isolado por tenant (RLS) e
  purgado junto com a entrada.

Fluxo de invalidação/expurgo por evento (sequência ASCII):

```
 010-Memory ─▶ aios.<tenant>.memory.item.deleted (LGPD/RTBF a montante)
        │
   EventPublisher (consumidor) ──▶ SemanticCacheManager.invalidate(source_urn)
        │
   INDEXED/SERVING ──▶ INVALIDATED ──▶ EvictionManager ──▶ EVICTED
        │
   evento aios.<tenant>.context.cache.invalidated + context.cache.evicted
        │
   Auditoria (025): actor=system, cause=upstream_deletion, source_urn, trace_id
```

## 5. Controles e verificação

| Controle | Requisito | Verificação |
|----------|-----------|-------------|
| Default deny | Toda operação privilegiada passa por `ContextPolicyGuard`/PDP. | Teste de autorização negada (403). |
| Isolamento de tenant | RLS + tenant guard; zero vazamento de cache entre tenants. | NFR-014: teste de RLS + fuzz de `tenant`. |
| Falso-hit de cache controlado | `similarity ≥ similarity_threshold` estrito antes de servir. | NFR-011: amostragem/eval offline (falso-hit ≤ 0,5%). |
| Auditoria imutável | Toda operação privilegiada e toda invalidação auditada. | Amostragem de trilha (025). |
| Idempotência de mutação | RFC-0001 §5.5 aplicada a `assemble`/`cache store`/`budget upsert`. | Repetição com mesma `Idempotency-Key` (`AIOS-CTX-0012` em divergência de payload). |
| Minimização/redação de PII | PII fora de eventos/logs; apenas em `content_ref`/`content_inline` sob RLS. | Scan de payloads/logs. |
| Expurgo rastreável | Invalidação por `source_urn`/tenant emite evento + auditoria. | Teste E2E de expurgo (FR-012). |
| mTLS interno | Serviço recusa tráfego interno não-mTLS. | Teste de conexão sem certificado. |

## 6. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-------------------|--------------------------|
| Cache semântico serve resposta de tarefa/tenant divergente (*cache poisoning*/falso-hit). | `similarity_threshold` conservador (default `0.92`, faixa `0.80–0.99`) + auditoria de amostragem (NFR-011) + namespace por tenant; decisão em `ADR-0112`/`ADR-0119` (FM-08 do brief §9). |
| Embedding de prompt/fragmento reconstruível a partir do vetor. | Vetor tratado como dado pessoal derivado quando a origem é PII; expurgo remove o vetor junto com a entrada de cache; acesso sob RLS/PDP. |
| Invalidação assíncrona deixa janela de exposição pós-`memory.item.deleted`. | Meta FR-008: invalidação ≤ 5 s após o evento; `EvictionManager` prioriza filas de invalidação sobre TTL/LRU. |
| Segredo em log por engano (ex.: DSN/URL em `content_ref`). | Proibição normativa + redação estruturada (Serilog→Seq); revisão de logging (`024`). |
| `017-Model-Router` recebe conteúdo sensível para embedding/sumarização. | Minimização a montante; contrato `017` trata payload como transitório, sem persistência lá (brief §1.3 N-01). |
| Bypass de política via *fast path* de leitura do próprio escopo (§1.2). | *Fast path* aplica apenas a leitura do próprio recurso e usa decisão de política cacheada com invalidação por evento — nunca substitui a decisão do PDP, apenas evita round-trip redundante. |

Decisões governadas por `ADR-0111` (cache em dois níveis), `ADR-0112`
(limiar de similaridade), `ADR-0117` (offload MinIO vs. inline) e `ADR-0119`
(isolamento de cache por tenant e defesa contra *cache poisoning*/falso-hit),
além de `RFC-0011` (proposta) — ver [ADR.md](./ADR.md) e [RFC.md](./RFC.md).
