---
Documento: Security
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100 (7 camadas → backends físicos), ADR-0101 (embeddings/HNSW), ADR-0102 (consolidação versionada), ADR-0103 (políticas de esquecimento), ADR-0105 (cotas/backpressure), ADR-0106 (blobs MinIO content-addressed), ADR-0107 (fronteira Runtime↔Memory), ADR-0109 (retenção legal/RTBF); ADR-0007, ADR-0008 (Governança por Política/Default Deny), ADR-0010 (Observabilidade/Auditoria por Construção) — herdadas
RFCs relacionados: RFC-0001 (baseline); RFC-0100 (a propor)
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 017-Model-Router, 021-Security, 022-Policy, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Security

> **Escopo.** Este documento especifica a postura de segurança do
> `MemoryService`: como ele autentica chamadores, como aplica autorização como
> **PEP (Policy Enforcement Point)** via `MemoryPep`, sua superfície de
> ataque, o *threat model* STRIDE detalhado por componente, a gestão de
> segredos, TLS/mTLS, o isolamento entre dependências e a conformidade
> LGPD/GDPR (incluindo o direito ao esquecimento — RTBF). Este documento
> **NÃO redefine** os contratos centrais de segurança (mTLS interno,
> validação OIDC no Gateway, envelope de erro RFC 7807) — eles são normativos
> em `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7 e detalhados em
> `../021-Security/`. Aqui, especifica-se **como o `MemoryService` os
> consome e os aplica** em sua fronteira de confiança específica, conforme
> `_DESIGN_BRIEF.md` §12. Palavras normativas conforme RFC 2119/8174.

---

## Índice

1. Postura de Segurança e Princípios
2. Autenticação (AuthN)
3. Autorização (AuthZ) — o `MemoryPep` como PEP
4. Superfície de Ataque
5. Threat Model STRIDE (detalhado)
6. Gestão de Segredos
7. TLS/mTLS — Fronteiras de Confiança
8. Isolamento entre Dependências (Bulkhead)
9. Hardening de Infraestrutura
10. Segurança de Dados em Repouso e em Trânsito
11. LGPD / GDPR e Direito ao Esquecimento (RTBF)
12. Auditoria e Não-Repúdio
13. Matriz de Controles (resumo executivo)
14. Referências

---

## 1. Postura de Segurança e Princípios

O `MemoryService` (010) é o guardião de **todo o conteúdo cognitivo
persistido** de um agente ou tenant — fatos, episódios, habilidades, grafos de
conhecimento e conteúdo bruto (via `BlobStoreAdapter`). Comprometer o
`MemoryService` equivale a comprometer a confidencialidade e a integridade de
tudo que o AIOS "sabe" sobre um tenant. Sua postura de segurança se apoia em
cinco princípios, todos rastreáveis a R10/R11 e N1/N5/N6 do
`_DESIGN_BRIEF.md`:

| Princípio | Descrição | Consequência de design |
|-----------|-----------|-------------------------|
| **Default Deny** | Nenhuma leitura (`recall`/`getItem`) ou mutação (`remember`/`consolidate`/`forget`) é efetivada sem decisão explícita de `allow` do PDP (`022-Policy`). | `MemoryPep` bloqueia por padrão; ausência de decisão = negação (`AIOS-MEM-0004`). |
| **PEP, não PDP** | O `MemoryService` **NÃO DEVE** decidir política de autorização localmente; ele apenas a aplica. | Nenhuma lógica de RBAC/ABAC é *hardcoded*; regras de acesso por camada/tenant vivem no `022-Policy`. |
| **Isolamento de Tenant Absoluto** | Nenhum item, embedding, aresta de grafo, blob ou evento atravessa a fronteira de `tenant_id` sem autorização explícita. | RLS obrigatória em toda tabela; namespace Redis/NATS/MinIO por tenant; divergência de tenant → `AIOS-MEM-0005`. |
| **Minimização e Rastreabilidade Legal** | Todo item carrega `legal_basis`, `retention_class` e a flag `pii`, mesmo quando o conteúdo é gerado internamente pelo agente. | `MemoryItem` não é persistido sem esses três campos (violação = `AIOS-MEM-0001`); base para RTBF (§11). |
| **Nunca chamar LLM diretamente** | O `MemoryService` **NÃO DEVE** ter credenciais para invocar provedores de modelo; embeddings são sempre obtidos via `EmbeddingClient` → `017-Model-Router`. | Reduz a superfície de exfiltração de dados sensíveis para fora da fronteira do control plane (N1 do brief). |

Este documento assume que o leitor já conhece os contratos centrais da
RFC-0001 (URN, envelope de evento, envelope de erro RFC 7807,
`Idempotency-Key`, `traceparent`/`X-AIOS-Tenant`) e **não os repete**, exceto
quando referenciados para explicar um controle específico do
`MemoryService`.

---

## 2. Autenticação (AuthN)

O `MemoryService` **NÃO DEVE** emitir, validar ou renovar tokens de
identidade — essa responsabilidade pertence ao Gateway (YARP) em conjunto com
`021-Security`. O `MemoryService` consome apenas **claims já validadas** e
propagadas por canal autenticado.

### 2.1 Fluxo de autenticação do chamador

| Etapa | Ator | Ação |
|-------|------|------|
| 1 | Agent Runtime (007, via Kernel 006) / Context Manager (011) / Learning (023) / Web Console (032) / CLI (030) | Envia requisição (`remember`/`recall`/`consolidate`/`forget`) com identidade já estabelecida. |
| 2 | Gateway (YARP) — tráfego REST | Valida JWT (OIDC) contra `021-Security`; extrai claims (`sub`, `tenant`, `roles`) e as propaga como cabeçalhos internos assinados via mTLS. |
| 3 | `MemoryApiFacade` (gRPC interno) | Para tráfego gRPC interno (Runtime/Context/Learning→Memory), valida identidade de workload via mTLS mútuo (SPIFFE/SVID); para tráfego via Gateway, valida que os cabeçalhos internos chegaram por canal mTLS autenticado (§2.3). |
| 4 | `MemoryPep` | Extrai `tenant`, `agent` (`X-AIOS-Agent`, se presente), `traceparent`; valida `Idempotency-Key` obrigatória em toda mutação (RFC-0001 §5.5). |

### 2.2 Autenticação interna serviço-a-serviço (mTLS)

Toda comunicação do `MemoryService` com dependências internas **DEVE** usar
**mTLS** com certificados de curta duração emitidos e rotacionados pela
infraestrutura de PKI de `021-Security` (RFC-0001 §6).

| Canal | Protocolo | Certificado | Rotação |
|-------|-----------|-------------|---------|
| Agent Runtime (007, via Kernel) → `MemoryApiFacade` | gRPC + mTLS | SPIFFE/SVID por workload | ≤ 24h |
| Context Manager (011) → `MemoryApiFacade` (recall seletivo) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Learning (023) → `MemoryApiFacade` (consolidação dirigida) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| `MemoryPep` → `022-Policy` (PDP) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| `EmbeddingClient` → `017-Model-Router` | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| `MemoryService` → `026-Cost-Optimizer` (reporte de consumo) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| `MemoryService` → PostgreSQL (fonte da verdade, `pgvector`/AGE) | TLS 1.3 + client cert | Certificado de serviço | 90 dias, renovação automática |
| `MemoryService` → Redis (`WorkingMemoryStore`, `IdempotencyStore`, `QuotaManager`) | TLS 1.3 + AUTH | Certificado de serviço + senha rotacionada | 90 dias |
| `MemoryService` → MinIO (`BlobStoreAdapter`) | TLS 1.3 + chave de acesso S3 | Chave de serviço | 90 dias |
| `MemoryService` → NATS/JetStream | TLS 1.3 + NKey/JWT de conta | NATS account JWT | 30 dias |

### 2.3 Validação de cabeçalhos internos assinados

Como o `MemoryService` confia nas claims propagadas pelo Gateway (para
tráfego REST administrativo) sem revalidar o JWT original, ele **DEVE**
validar que os cabeçalhos internos (`X-AIOS-Tenant`, `X-AIOS-Subject`,
`X-AIOS-Roles`) chegaram por um canal mTLS autenticado cujo certificado de
cliente pertence ao pool de identidades confiáveis do Gateway (*workload
identity allowlist*). Requisições sem essa identidade de workload
autenticada **DEVEM** ser rejeitadas com `401` antes de qualquer
processamento — isso impede que um serviço comprometido da malha injete
claims forjadas diretamente no `MemoryService`, contornando o Gateway. Para
o caminho gRPC interno (Runtime/Context/Learning → Memory), a identidade do
chamador é o próprio certificado mTLS do workload; o `MemoryApiFacade`
**DEVE** verificar que o SPIFFE ID do chamador pertence à *allowlist* de
serviços autorizados a chamar `Remember`/`Recall`/`Consolidate`/`Forget`.

---

## 3. Autorização (AuthZ) — o `MemoryPep` como PEP

### 3.1 Modelo RBAC + ABAC (decidido externamente)

O `MemoryService` implementa exclusivamente o **ponto de aplicação** (PEP)
via `MemoryPep`; o modelo de papéis (RBAC) e atributos (ABAC) é definido e
avaliado pelo PDP em `022-Policy`. Toda operação — inclusive leituras —
**DEVE** passar por uma consulta ao PDP antes de ser efetivada, pois memória
é um recurso cuja simples leitura já é sensível (fatos, episódios e
habilidades de um agente/tenant).

| Operação | Rota | Consulta PDP obrigatória | Recurso avaliado (ABAC) |
|----------|------|----------------------------|--------------------------|
| `remember(item, layer)` | `POST /v1/memory/items` / `Remember` | Sim (`Authorize(subject, "memory.remember", layer, ctx)`) | `layer`, `tenant`, `agent_id`, `pii` |
| `recall(query, filtros)` | `POST /v1/memory/recall` / `Recall` | Sim (`Authorize(subject, "memory.recall", layers[], ctx)`) | `layer[]`, `tenant`, `agent_id`, `mode` |
| `consolidate()` | `POST /v1/memory/consolidate` / `Consolidate` | Sim, sempre | `from_layer`, `to_layer`, `tenant`, `agent_id` |
| `forget(policy)` | `POST /v1/memory/forget` / `Forget` | Sim, sempre | `scope` (`tenant\|agent\|layer`), `strategy` |
| `PurgeItem` (RTBF) | `DELETE /v1/memory/items/{id}` | Sim, sempre (requer papel de titular de dados ou operador de conformidade) | `item_id`, `legal_hold` |
| `RollbackConsolidation` | `POST /v1/memory/consolidate/{jobId}/rollback` | Sim, sempre; requer papel `memory-admin` | `version_id`, `tenant`, `agent_id` |
| `getItem`/`getStats`/`listLayers`/`getJob` | Leitura de metadados/config | Verificação de escopo de leitura (isolamento de tenant, §3.4); PDP consultado quando a política exigir escopo restrito | `tenant`, `agent_id` |

O `DecisionRequest` montado pelo `MemoryPep` **DEVE** conter, no mínimo:

| Campo | Origem | Descrição |
|-------|--------|-----------|
| `subject` | Claims propagadas / SPIFFE ID do workload chamador | Quem solicita (agente pai, service account, operador humano, titular de dados). |
| `action` | Operação da tabela acima (`memory.remember`, `memory.recall`, `memory.forget`, `memory.consolidate.rollback`, ...) | Ação sobre a qual se decide. |
| `resource` | URN `urn:aios:<tenant>:memory:<id>` do item alvo, ou `urn:aios:<tenant>:memory:layer:<layer>` para operações de escopo de camada | Sobre o que a ação incide. |
| `environment` | `tenant`, `agent_id`, `pii`, `legal_basis`, `retention_class`, hora, `mode` de recall | Contexto para regras ABAC (ex.: negar `recall` *cross-agent* de itens marcados `pii=true` sem consentimento explícito). |

### 3.2 Ciclo de decisão e cache

```
   MemoryApiFacade          MemoryPep                    PolicyClient            022-Policy (PDP)
        │                        │                            │                        │
        │  RememberRequest       │                             │                        │
        │───────────────────────▶│                             │                        │
        │                        │  cache hit? (TTL curto)     │                        │
        │                        │──────┐                      │                        │
        │                        │◀─────┘ sim → allow/deny      │                        │
        │                        │  (pula chamada ao PDP)       │                        │
        │                        │                             │                        │
        │                        │  cache miss                 │                        │
        │                        │  DecisionRequest             │                        │
        │                        │────────────────────────────▶│  Evaluate()            │
        │                        │                             │───────────────────────▶│
        │                        │                             │◀── allow|deny + ttl ───│
        │                        │◀────────────────────────────│                        │
        │◀── allow → prossegue ──│                             │                        │
        │◀── deny → 403 MEM-0004─│                             │                        │
```

- Decisões de `allow` **PODEM** ser cacheadas por até um TTL curto (segundos,
  compatível com o SLO de `remember`/`recall` — NFR-001/NFR-003), **nunca**
  superior ao TTL retornado pelo PDP.
- Decisões de `deny` **NÃO DEVEM** ser cacheadas com TTL maior que o de
  `allow`, para permitir revogação rápida de acesso.
- Uma atualização de *policy bundle* (evento
  `aios.<tenant>.policy.decision.updated`, consumido pelo `MemoryService` —
  brief §6.2) **DEVE** invalidar o cache local de decisão do `PolicyClient`
  para o tenant afetado, independentemente do TTL restante.
- O cache de decisão de política é **estritamente distinto** do
  `IdempotencyStore` e do cache de cotas do `QuotaManager` — cada um tem sua
  própria política de invalidação.

### 3.3 Fail-Closed sob indisponibilidade do PDP

| Comportamento sob timeout/erro do PDP | Consequência |
|------------------------------------------|----------------|
| **Default** (produção) | Nega a operação (`AIOS-MEM-0004`, HTTP 403, `retriable=false` — a negação em si não é retriável, mas o chamador PODE tentar novamente após o PDP se recuperar). |
| Circuit breaker aberto no `PolicyClient` | Toda nova operação que exija PDP é imediatamente negada sem nova tentativa de rede, evitando amplificar carga sobre um PDP já degradado. |

Não há modo "*fail-open*" configurável para operações de memória em
produção — dado que memória concentra o conteúdo cognitivo mais sensível do
sistema, uma admissão indevida de leitura ou escrita tem impacto de
confidencialidade irreversível (um item lido/vazado não pode ser
"des-lido").

### 3.4 Isolamento multi-tenant como controle de autorização

Antes mesmo de consultar o PDP, o `MemoryApiFacade` **DEVE** validar que o
`tenant` presente na URN do recurso alvo (`urn:aios:<tenant>:memory:<id>`) é
idêntico ao `tenant` do contexto autenticado (`X-AIOS-Tenant` ou tenant
derivado da identidade de workload mTLS). Divergência **DEVE** ser rejeitada
com `AIOS-MEM-0005` (400) **antes** de qualquer chamada ao PDP — verificação
estrutural, barata e determinística, e o controle primário contra vazamento
*cross-tenant* (mapeado à ameaça *Information Disclosure* no STRIDE, §5).
Adicionalmente:

- Toda tabela do modelo de dados (`MemoryItem`, `MemoryLayerConfig`,
  `ConsolidationJob`, `ConsolidationVersion`, `ForgettingPolicy`,
  `MemoryQuota`, `OutboxEvent`, `IdempotencyRecord`) **DEVE** ter Row-Level
  Security (RLS) por `tenant_id` — ver `Database.md`.
- Namespace Redis é particionado por `mem:{tenant}:{layer}:...` — não há
  estrutura de dados compartilhada entre tenants.
- Namespace NATS (`aios.<tenant>.memory.*`) garante que eventos de um tenant
  não sejam entregues a consumidores de outro.
- Buckets/prefixos MinIO são particionados por `{tenant}/{layer}/{hash}` — o
  `BlobStoreAdapter` **NÃO DEVE** aceitar `content_ref` cujo prefixo de
  tenant diverja do contexto autenticado.
- Uma consulta `getItem`/`recall` para um recurso de outro tenant retorna
  `AIOS-MEM-0002` (não encontrado), nunca vazamento de existência.
- Acesso *cross-agent* dentro do mesmo tenant (um agente lendo memória de
  outro agente do mesmo tenant) **DEVE** ser mediado por capability
  explícita avaliada pelo PDP (syscall cognitiva, `006-Kernel`/`021`), não
  por *default allow* dentro do tenant.

---

## 4. Superfície de Ataque

### 4.1 Inventário de pontos de entrada

| Superfície | Protocolo | Exposição | Controle primário |
|------------|-----------|-----------|---------------------|
| `MemoryApiFacade` gRPC (`aios.memory.v1.MemoryService`) | gRPC + mTLS | Interna (Agent Runtime 007 via Kernel, Context Manager 011, Learning 023) | mTLS workload identity + PEP + `Idempotency-Key` |
| `MemoryApiFacade` REST (`/v1/memory/*`) | HTTPS via Gateway/YARP | Externa/administrativa (Web Console 032, CLI 030, operadores, titulares de dados via portal de privacidade) | AuthN no Gateway (OIDC) + PEP + rate-limit |
| Eventos consumidos (`aios.<tenant>.learning.consolidation.requested`, `learning.policy.rolledback`, `context.recall.requested`, `agent.lifecycle.*`, `policy.decision.updated`, `security.rtbf.requested`) | NATS/JetStream (TLS) | Interna | Conta NATS por serviço; validação de `dataschema`; deduplicação por `event.id` |
| Conexão PostgreSQL (`MemoryItem`, `pgvector`, Apache AGE) | TLS + client cert | Interna | RLS por `tenant_id`; credenciais de menor privilégio; sem acesso direto do Runtime (ADR-0107) |
| Conexão Redis (`WorkingMemoryStore`, `IdempotencyStore`, `QuotaManager`) | TLS + AUTH | Interna | Namespace de chave por tenant; ACL Redis restrita a comandos necessários (sem `FLUSHALL`/`KEYS *`) |
| Conexão MinIO (`BlobStoreAdapter`) | TLS + chave de acesso S3 | Interna | *Bucket policy* por tenant; content-addressed (hash) impede sobrescrita silenciosa |
| Endpoints de observabilidade (`/metrics`, `/healthz`, `/readyz`) | HTTP interno | Interna (scrape Prometheus/probe) | Sem dados sensíveis; acesso restrito à rede de observabilidade (`Deployment.md`) |

### 4.2 Vetores de ataque considerados e descartados

| Vetor | Considerado? | Justificativa |
|-------|---------------|----------------|
| Injeção via `content`/`metadata` (jsonb) de `MemoryItem` em consulta SQL | Sim, mitigado | Campos tratados como dados opacos (nunca interpolados em query SQL); acesso via ORM/queries parametrizadas. Travessias openCypher (Apache AGE) usam parâmetros vinculados, nunca concatenação de string. |
| Forjamento de `tenant` em URN de `MemoryItem` | Sim, mitigado | Validação estrutural §3.4 antes de qualquer processamento; RLS como segunda camada (*defense in depth*). |
| Replay de `remember`/`consolidate`/`forget`/`PurgeItem` | Sim, mitigado | `Idempotency-Key` obrigatório (RFC-0001 §5.5) + `IdempotencyStore`; reprodução retorna o mesmo resultado, sem duplo efeito (`AIOS-MEM-0003` em caso de conflito). |
| Exfiltração de embeddings para reconstrução aproximada do conteúdo original (*embedding inversion*) | Sim, mitigado | Embeddings **NÃO DEVEM** ser expostos em bruto por API pública sem contexto de autorização equivalente ao do item original; `RecallResult` retorna scores/ranking, não o vetor cru, salvo para consumidores internos autorizados (`RecallEngine`↔`Context` 011). |
| Manipulação direta de contador de cota (`QuotaManager`) para burlar `memory.quota.max_items/max_bytes` | Sim, mitigado | Reserva/contagem via operação atômica no Redis; ACL Redis restringe comandos a um *allowlist* que não inclui escrita arbitrária de chave. |
| Escalonamento de camada não autorizado (item classificado indevidamente como `Semantic`/`KnowledgeGraph` para escapar de TTL de camadas voláteis) | Sim, mitigado | `LayerRouter` valida `kind`×`layer` (rejeita incompatibilidade com `AIOS-MEM-0020`); mudança de camada só ocorre via `ConsolidationEngine`, sob PEP e versionada. |
| Expurgo indevido de item sob `legal_hold` para destruir evidência | Sim, mitigado | `ForgettingEngine` bloqueia com `AIOS-MEM-0050` (423) salvo RTBF autorizado explicitamente e auditado (025); nenhuma rota alternativa de expurgo existe fora de `Forget`/`PurgeItem`. |
| Exaustão de armazenamento via *flood* de `remember` (DoS) | Sim, mitigado | Cotas por (tenant, agente, camada) via `QuotaManager` + `memory.backpressure.max_inflight` + rate-limit de borda no Gateway (*defense in depth*). |
| Rollback de consolidação usado para reverter esquecimento legítimo (ex.: RTBF) e "ressuscitar" dado apagado | Sim, mitigado | `RollbackConsolidation` opera sobre `ConsolidationVersion`, não sobre tombstones de `FORGOTTEN`/`PURGED`; itens `PURGED` são fisicamente removidos e não fazem parte de nenhum *snapshot* de versão retido (ADR-0102, ADR-0109). |
| Execução de código arbitrário no processo do `MemoryService` | Não aplicável | O `MemoryService` **não** executa código de agente/ferramenta — é um serviço de persistência/recuperação; execução ocorre em `007-Agent-Runtime` (sandbox, fora do escopo deste módulo). |
| Vazamento *cross-tenant* via `RecallResult` de busca híbrida (grafo/ANN) | Sim, mitigado | Todo índice HNSW e toda travessia AGE são particionados por tenant/partição (`Scalability.md` §3); `RecallEngine` nunca consulta índice de outro tenant. |

---

## 5. Threat Model STRIDE (detalhado)

> A versão resumida está no `_DESIGN_BRIEF.md` §12.2; esta seção a detalha
> por componente interno e aponta o controle técnico exato.

| Ameaça | Componente afetado | Cenário concreto | Controle técnico | Detecção |
|--------|----------------------|-------------------|---------------------|----------|
| **Spoofing** | `MemoryApiFacade` | Serviço comprometido injeta `X-AIOS-Tenant` diretamente, contornando o Gateway. | mTLS workload identity allowlist (§2.3); rejeição sem identidade válida. | Alerta em taxa anômala de `401` (`aios_memory_authn_rejected_total`). |
| **Spoofing** | Eventos consumidos | Publicador não autorizado emite `aios.<tenant>.learning.consolidation.requested` forjado, disparando consolidação indevida. | Conta NATS por serviço com permissão de publish restrita ao subject próprio; validação de `source` no envelope CloudEvents. | Auditoria de publishers por subject (025). |
| **Tampering** | `MemoryItem` (PostgreSQL) | Alteração direta de `salience`/`decay_score`/`embedding` fora do fluxo do `ConsolidationEngine`. | Acesso exclusivo via API do `MemoryService` (nenhum outro serviço tem credencial de escrita direta na tabela — ADR-0107); mutação sempre via camada de domínio versionada. | Reconciliação periódica detecta divergência entre `updated_at`/versão esperada e observada. |
| **Tampering** | `content_hash` / `BlobStoreAdapter` | Substituição de blob em MinIO sem atualizar o hash referenciado. | Content-addressed: `content_ref` inclui `content_hash` (sha256); leitura valida hash antes de servir conteúdo. | Alerta de divergência de hash na leitura (`aios_memory_blob_hash_mismatch_total`). |
| **Repudiation** | Toda operação privilegiada | Chamador nega ter solicitado `forget`/`PurgeItem`/`consolidate`. | Eventos de domínio (§6 do brief) + trilha imutável em `025-Audit`, com `subject`, URN do item, `trace_id`, `decision_id` do PDP. | Consulta de auditoria por `trace_id`/URN/`job_id`. |
| **Information Disclosure** | `MemoryApiFacade` (erros) | Mensagem de erro vaza detalhe interno (query SQL, conteúdo do item, string de conexão). | Envelope RFC 7807 padronizado (RFC-0001 §5.4); `detail` sanitizado, sem PII/segredos/conteúdo de memória. | Revisão de amostras de log de erro em CI (regra de *lint* de log). |
| **Information Disclosure** | Cross-tenant | Tenant A consulta `getItem`/`recall` de recurso do tenant B. | Validação estrutural de tenant (§3.4) + RLS de banco + filtro de namespace Redis/NATS/MinIO como camadas adicionais (*defense in depth*). | `AIOS-MEM-0002`/`0005` monitorados; alerta em taxa anômala por tenant. |
| **Information Disclosure** | Logs/eventos de domínio | Payload de evento (`item.stored`, `item.recalled`) carrega conteúdo sensível/PII inline. | Eventos carregam apenas URN + metadados mínimos (RFC-0001 §7); conteúdo de negócio permanece em PG/MinIO sob RLS, nunca no envelope de evento; `pii=true` aciona redação adicional em logs (Serilog). | Varredura de *schema* de amostras de evento sem campos de conteúdo de negócio. |
| **Information Disclosure** | `embedding` (pgvector) | Vetor bruto exposto permite ataque de *embedding inversion* para reconstruir conteúdo aproximado. | Vetores não são expostos por API pública em bruto; acesso restrito a componentes internos autorizados (§4.2). | Varredura de contrato de API garantindo ausência de campo `embedding` em `RecallResult` público. |
| **Denial of Service** | `MemoryApiFacade` | *Flood* de `remember` de um único tenant satura camadas/índices compartilhados. | Cotas por (tenant, agente, camada) via `QuotaManager` + `memory.backpressure.max_inflight` + rate-limit de borda no Gateway. | Métrica de rejeição por motivo (`aios_memory_quota_exceeded_total`, `aios_memory_backpressure_active`). |
| **Denial of Service** | `EmbeddingClient`/`017-Model-Router` | Model Router lento degrada latência de todo `remember` com embedding. | Circuit breaker por dependência (bulkhead, §8); item persiste como `INGESTED` sem embedding, com *backfill* assíncrono (F3 de `FailureRecovery.md`). | `aios_memory_remember_duration_ms` p99 acima da meta (NFR-001) dispara alerta. |
| **Elevation of Privilege** | `ConsolidationEngine`/`ForgettingEngine` | Bypass do PEP por chamada direta a um *handler* interno de consolidação/esquecimento. | Nenhum *handler* é alcançável sem passar pelo pipeline `MemoryApiFacade → MemoryPep`; garantido por design de camadas (sem rota alternativa registrada). | Cobertura de teste de contrato (`Testing.md`) valida ausência de rota sem PEP; auditado por *pentest*. |
| **Elevation of Privilege** | Mutação de `MemoryLayerConfig` | Alteração não autorizada de TTL/cota/`decay_half_life` para favorecer um tenant ou evadir esquecimento. | Requer papel `memory-admin` + consulta PDP obrigatória (§3.1); toda alteração gera evento auditável. | Auditoria de mudança de configuração (025); alerta de mudança fora de janela de manutenção. |
| **Elevation of Privilege** | `RollbackConsolidation` | Reversão de consolidação usada para restaurar dado que deveria permanecer esquecido. | `RollbackConsolidation` requer papel `memory-admin`; não opera sobre tombstones/itens `PURGED` (ver §4.2). | Auditoria dedicada de rollback (`aios_memory_consolidation_rollback_total`, evento `consolidation.rolledback`). |

---

## 6. Gestão de Segredos

| Segredo | Tipo | Armazenamento | Rotação | Acesso |
|---------|------|-----------------|---------|--------|
| Certificados mTLS (SPIFFE/SVID) do `MemoryService` | Certificado X.509 de curta duração | Emitido em runtime pela infraestrutura de PKI (sidecar/SDS), nunca em disco persistente | ≤ 24h automática | Somente o processo do `MemoryService` via socket local do agente de identidade |
| Credencial PostgreSQL (fonte da verdade) | Usuário/senha + client cert | Cofre de segredos (Vault/KMS gerenciado por `021`), injetado como variável de ambiente/arquivo montado em `tmpfs` | 90 dias ou sob incidente | Processo do `MemoryService` apenas |
| Credencial Redis (AUTH) | Senha | Cofre de segredos | 90 dias ou sob incidente | Processo do `MemoryService` apenas |
| Chave de acesso MinIO (`BlobStoreAdapter`) | Access key/secret key S3 | Cofre de segredos | 90 dias ou sob incidente | Processo do `MemoryService` apenas |
| NATS account JWT/NKey | JWT assinado | Cofre de segredos | 30 dias | Processo do `MemoryService` apenas |
| Chaves de assinatura de cabeçalho interno (Gateway↔Memory) | Chave simétrica/assimétrica | Cofre de segredos, sincronizada entre Gateway e `MemoryService` | 30 dias | Gateway (assina) e `MemoryService` (verifica) |

**Regras normativas:**

- Nenhum segredo **DEVE** ser escrito em log, trace, métrica ou payload de
  evento (RFC-0001 §7 — minimização).
- Nenhum segredo **DEVE** ser embutido em imagem de container ou em
  código-fonte; todo segredo é injetado em tempo de execução (ver
  `Deployment.md` §3).
- Segredos **DEVEM** ser lidos apenas na inicialização ou via *hot reload*
  seguro (sem persistir cópia em disco além do necessário para o processo).
- Chaves de configuração de domínio `memory.*` (`Configuration.md`) **NÃO
  DEVEM** carregar segredos — apenas parâmetros comportamentais (TTL, cotas,
  limiares); DSNs/credenciais residem exclusivamente no cofre.
- Rotação de segredo **NÃO DEVE** exigir *downtime*: o `MemoryService`
  **DEVE** suportar recarregamento de credencial sem reinício (pools de
  conexão a Redis/PostgreSQL/MinIO renovados de forma gradual).

---

## 7. TLS/mTLS — Fronteiras de Confiança

```
        Zona Não-Confiável           Zona de Borda (DMZ)              Zona Interna (Malha mTLS)
   ┌────────────────────┐      ┌──────────────────────┐      ┌───────────────────────────────────┐
   │  Operador/Console   │      │   Gateway (YARP)     │      │        MEMORY SERVICE (010)        │
   │  (032/030) / titular│─TLS─▶│  - Valida OIDC/JWT   │─mTLS─▶│  MemoryApiFacade (PEP: MemoryPep) │
   │  de dados (RTBF)    │ 1.3  │  - Assina cabeçalhos │      │  LayerRouter · RecallEngine         │
   └────────────────────┘      │    internos           │      │  ConsolidationEngine · ForgettingEngine│
                                └──────────────────────┘      └───────────────┬───────────────────┘
   ┌────────────────────┐                                                     │
   │ Agent Runtime (007) │────────── mTLS direto (gRPC interno, via 006) ────▶│
   │ Context Mgr (011)   │                                                    │ mTLS (todas as setas)
   │ Learning (023)      │                       ┌──────────┬────────┬───────┼─────────┬────────────┐
   └────────────────────┘                          ▼          ▼        ▼       ▼         ▼            ▼
                                              022-Policy  017-Model  026-Cost PostgreSQL Redis/MinIO  NATS
                                                (PDP)      -Router    -Opt.   (+pgvector+AGE)
```

- **Fora → Borda**: TLS 1.3 padrão de mercado (SDK/browser/CLI/portal de
  privacidade → Gateway).
- **Runtime/Context/Learning → Memory**: mTLS interno direto (gRPC, sem
  passar pelo Gateway externo — tráfego *control-plane→control-plane*).
- **Borda → Interna** e **Interna → Interna**: mTLS obrigatório em toda
  chamada (RFC-0001 §6); nenhuma comunicação de serviço a serviço **DEVE**
  ocorrer em texto plano ou apenas TLS unilateral.
- Cifras **DEVEM** seguir o perfil moderno definido por `021-Security` (TLS
  1.3 preferencial; TLS 1.2 apenas com conjuntos de cifra aprovados como
  *fallback* transitório).

---

## 8. Isolamento entre Dependências (Bulkhead)

O `MemoryService` **não hospeda** sandbox de execução de agente (isso é
`007-Agent-Runtime`, N7 do brief). Seus controles de isolamento são de
**bulkhead** e **circuit breaker** entre dependências, para que a falha ou
degradação de uma dependência não se propague e não force o serviço a
comprometer garantias de segurança (ex.: aceitar item sem validação de
cota/PDP por indisponibilidade parcial):

| Dependência | Padrão de isolamento | Efeito de falha isolado |
|-------------|------------------------|----------------------------|
| PDP (`022-Policy`) | Bulkhead dedicado no `PolicyClient` + circuit breaker | Falha do PDP nega toda operação (`AIOS-MEM-0004`); não há *fail-open* (§3.3). |
| Model Router (`017`) | Bulkhead no `EmbeddingClient` | Falha degrada para item `INGESTED` sem embedding (F3, `FailureRecovery.md`); não bloqueia escrita nem compromete PEP. |
| Cost-Optimizer (`026`) | Bulkhead no reporte de consumo | Falha não bloqueia `remember`/`recall` — apenas atrasa a contabilização de custo, reportada de forma assíncrona/eventual. |
| PostgreSQL (fonte da verdade) | Circuit breaker de infraestrutura | Indisponibilidade força rejeição de mutações (`AIOS-MEM-0031`), nunca escrita silenciosa sem RLS/durabilidade. |
| Redis (`WorkingMemoryStore`, `IdempotencyStore`, `QuotaManager`) | Circuit breaker de infraestrutura | Indisponibilidade degrada caminho quente (F1, `FailureRecovery.md`); `IdempotencyStore` indisponível **NÃO DEVE** ser contornado — mutação sem chave validada é rejeitada, nunca aplicada sem *dedup*. |
| Apache AGE / MinIO | Circuit breaker por *backend* | Isola apenas o caminho de grafo/blob (F4/F5); não compromete PEP nem RLS das demais camadas. |

Cada *bulkhead* **DEVE** ter *pool* de conexões/threads dedicado e limite
próprio, de forma que a saturação de uma dependência não esgote recursos
compartilhados usados por chamadas a outras dependências, e — crucialmente —
**nenhuma degradação de dependência DEVE relaxar um controle de segurança**
(PEP, RLS, idempotência permanecem obrigatórios mesmo em modo degradado; ver
`FailureRecovery.md` §6 para a matriz de degradação graciosa funcional).

---

## 9. Hardening de Infraestrutura

| Controle | Especificação |
|----------|-----------------|
| Usuário do processo | Container do `MemoryService` **DEVE** executar como usuário não-root (UID dedicado, sem `CAP_SYS_ADMIN` ou capacidades Linux desnecessárias). |
| Sistema de arquivos | Sistema de arquivos raiz **DEVERIA** ser montado somente-leitura; diretórios graváveis restritos a `/tmp` efêmero e diretório de segredos em `tmpfs`. |
| Rede | Políticas de rede (NetworkPolicy/equivalente) **DEVEM** restringir *egress* do `MemoryService` apenas às dependências listadas em §4.1; nenhum *egress* arbitrário à internet (em particular, nenhum acesso direto a provedores de LLM externos). |
| Imagem | Imagem base mínima (distroless ou equivalente), sem *shell* interativo em produção; *scan* de vulnerabilidade (SCA) obrigatório no pipeline de CI. |
| Segredos em imagem | **NÃO DEVE** haver segredo, chave privada ou credencial embutida em camada de imagem (verificado por *secret scanning* em CI). |
| Dependências | *Software Composition Analysis* (SCA) contínuo sobre pacotes .NET 10; CVEs críticas **DEVEM** ser corrigidas dentro do SLA de patch de `021-Security`. |
| ACL Redis | Comandos permitidos ao `MemoryService` restritos a um *allowlist* (`SET`/`GET`/`EXPIRE`/`ZADD`/`INCR`, scripts Lua conhecidos, etc.); comandos administrativos (`FLUSHALL`, `CONFIG SET`, `KEYS *`) **NÃO DEVEM** estar acessíveis à credencial de serviço. |
| *Bucket policy* MinIO | Políticas de *bucket* restritas por prefixo de tenant; versão de objeto imutável para *snapshots* de `ConsolidationVersion` (proteção adicional contra *tampering*). |

---

## 10. Segurança de Dados em Repouso e em Trânsito

| Dado | Em trânsito | Em repouso |
|------|--------------|-------------|
| `content` (inline, jsonb) | TLS 1.3 (cliente→Gateway→Memory→PostgreSQL) | Criptografia de disco no nível de armazenamento gerenciado pela infraestrutura (`028-Deployment`); RLS por `tenant_id`. |
| `content_ref` (blob externalizado, MinIO) | TLS 1.3 (Memory↔MinIO) | Criptografia *server-side* do MinIO (SSE); *bucket policy* por tenant; *content-addressed* (hash íntegro). |
| `embedding` (vector, pgvector) | TLS 1.3 (Memory↔PostgreSQL) | RLS por `tenant_id`; não exposto em bruto por API pública (§4.2, §5). |
| `ConsolidationVersion.snapshot_ref` (MinIO) | TLS 1.3 | Objeto imutável versionado; retenção conforme `memory.consolidation.version.retention`. |
| `IdempotencyRecord`/`OutboxEvent` | TLS 1.3 (Memory↔Redis/PostgreSQL) | TTL/retenção conforme RFC-0001 §5.5; sem conteúdo de negócio de item, apenas referências e resultado memoizado. |

- Campos marcados `pii=true` **DEVEM** ser tratados como sensíveis em toda a
  cadeia: redação/tokenização em logs (Serilog→Seq), ausência em payload de
  evento (§5), e escopo de leitura restrito por ABAC no PDP.
- O `MemoryService` **NÃO DEVE** logar `content`/`embedding` em nível
  `Debug`/`Trace` em produção — apenas metadados estruturais (URN, `layer`,
  `state`, tamanhos) são elegíveis para log (ver `Logging.md`).

---

## 11. LGPD / GDPR e Direito ao Esquecimento (RTBF)

- **Base legal obrigatória**: todo `MemoryItem` **DEVE** carregar
  `legal_basis` (`consent|contract|legitimate_interest|legal_obligation`) e
  `retention_class` (`ephemeral|standard|extended|legal_hold`) — RFC-0001
  §7, brief §3.1. Ausência desses campos **DEVE** ser rejeitada na escrita
  (`AIOS-MEM-0001`).
- **Minimização de dados**: eventos/logs carregam o mínimo necessário
  (URN, `layer`, `state`, timestamps); conteúdo sensível permanece em
  PostgreSQL/MinIO sob RLS, nunca no envelope de evento (§5, RFC-0001 §7).
- **Direito ao esquecimento (RTBF)**: operação de expurgo rastreável
  (`DELETE /v1/memory/items/{id}` → `PurgeItem`, ou `forget(policy=rtbf)`),
  assíncrona, aceita com `AIOS-MEM-0051` (202) e agendada pelo
  `ForgettingEngine`. O fluxo completo:

```
 Titular/DPO ─▶ security.rtbf.requested (021/025)  ou  DELETE /v1/memory/items/{id}
        │
   MemoryPep autoriza (PDP 022) ──▶ ForgettingEngine
        │  legal_hold? ── sim & não-RTBF ─▶ 423 AIOS-MEM-0050
        │  autorizado
        ▼
   FORGET_PENDING ─(confirma)▶ FORGOTTEN (tombstone + evento item.forgotten)
        │  ⌛ memory.forget.grace_period
        ▼
   PURGED: remove vetor (pgvector) + blob (MinIO) + arestas (AGE)
        ─▶ evento item.purged ─▶ Audit (025)
```

  1. Requisição de RTBF (direta ou via evento
     `aios.<tenant>.security.rtbf.requested` consumido do 021/025) é
     autorizada pelo PDP (papel de titular de dados ou operador de
     conformidade).
  2. Item transita `*→FORGET_PENDING` (mesmo estando em `legal_hold`, pois
     RTBF autorizado é a exceção explícita à invariante — brief §4.1).
  3. Após `memory.forget.grace_period`, o item transita a `FORGOTTEN`
     (tombstone) e, ao final, a `PURGED` (terminal): remoção física de
     conteúdo inline, blob MinIO, vetor `pgvector` e arestas de grafo (AGE)
     associadas, mantendo apenas o tombstone mínimo para auditoria.
  4. Evento `item.purged` é emitido para consumo por `025-Audit` — a
     evidência de que o expurgo ocorreu é auditável mesmo após a remoção do
     conteúdo em si.
- **`legal_hold` não é absoluto**: um item em `legal_hold` **NÃO DEVE** ser
  purgado por poda automática (TTL/decay/LRU/cota), mas **DEVE** ser
  purgável sob RTBF explicitamente autorizado — a autorização de RTBF é o
  único caminho que sobrepõe `legal_hold` (invariante do brief §4.1,
  ADR-0109).
- **Retenção por classe**: `ephemeral`/`standard`/`extended` governam TTL e
  elegibilidade a poda automática; `legal_hold` suspende poda automática até
  liberação explícita ou RTBF autorizado.
- **Portabilidade/acesso do titular**: consultas de titular de dados (via
  Web Console/portal de privacidade → Gateway → `getItem`/`recall`
  filtrado) **DEVEM** passar pelo mesmo PEP/PDP que qualquer outra leitura,
  garantindo que o titular só veja itens de sua própria titularidade
  (avaliação ABAC pelo PDP, não uma rota especial sem autorização).

---

## 12. Auditoria e Não-Repúdio

Toda operação privilegiada (`remember`, `recall`, `consolidate`, `forget`,
`PurgeItem`, `RollbackConsolidation`, mutação de `MemoryLayerConfig`)
**DEVE** gerar:

1. Um evento de domínio correspondente (brief §6.1), publicado via padrão
   Outbox, contendo `subject`, URN do recurso, `trace_id`, `tenant`.
2. Quando aplicável, um registro de decisão do PDP correlacionável
   (`decision_id`) para reconstrução de "quem autorizou o quê".
3. Consumo pelo `025-Audit`, que mantém a trilha imutável — o
   `MemoryService` **NÃO DEVE** implementar sua própria trilha de auditoria
   paralela (N6 do brief); apenas garante que todo evento relevante seja
   emitido de forma confiável (Outbox *at-least-once*).

Essa trilha **DEVE** ser suficiente para responder, para qualquer operação
sobre memória: *quem* solicitou, *o quê*, *sobre qual item/camada/agente*,
*quando*, *qual foi a decisão do PDP* e *qual o resultado* — sem exigir
correlação manual entre sistemas (NFR-013: 100% das operações com trace
OTel + `tenant_id`/`trace_id`).

---

## 13. Matriz de Controles (resumo executivo)

| ID | Controle | Categoria | Verificação |
|----|----------|-----------|--------------|
| SEC-M-01 | Default deny em toda operação (inclusive leitura) | AuthZ | Teste de contrato: `remember`/`recall`/`consolidate`/`forget` sem decisão PDP `allow` → `AIOS-MEM-0004`. |
| SEC-M-02 | Fail-closed sob indisponibilidade do PDP (sem *fallback fail-open* em produção) | AuthZ/Resiliência | *Chaos test*: derrubar PDP → toda operação negada, nunca admitida por *default*. |
| SEC-M-03 | Validação estrutural de `tenant` antes do PDP | Isolamento | Teste: `tenant` divergente em URN → `AIOS-MEM-0005` sem chamada ao PDP. |
| SEC-M-04 | mTLS obrigatório em toda comunicação interna | Transporte | *Scan* de configuração + teste de conexão sem certificado válido é rejeitada. |
| SEC-M-05 | RLS por `tenant_id` em toda tabela do modelo de dados | Isolamento | Teste automatizado de RLS (query *cross-tenant* retorna vazio). |
| SEC-M-06 | Nenhum segredo em imagem/log/evento | Segredos | *Secret scanning* em CI + regra de *lint* de log. |
| SEC-M-07 | `Idempotency-Key` obrigatório em toda mutação | Integridade | Teste de *replay*: mesma chave → mesmo resultado, sem duplo efeito. |
| SEC-M-08 | Auditoria imutável de toda operação privilegiada | Repúdio | Cobertura auditada (NFR-013): 100% das operações com evento de proveniência. |
| SEC-M-09 | Bulkhead/circuit breaker por dependência externa, sem relaxar PEP/RLS/idempotência sob degradação | Isolamento de falha | *Chaos test* por dependência isolada (§8). |
| SEC-M-10 | Embeddings nunca expostos em bruto por API pública | Confidencialidade | Varredura de contrato de API (`Testing.md`) garantindo ausência do campo `embedding` em resposta pública. |
| SEC-M-11 | Nenhuma chamada direta a provedor de LLM/embedding fora do `017-Model-Router` | Confidencialidade/Superfície | *Scan* de *egress* de rede + revisão de credenciais do processo (§9). |
| SEC-M-12 | `legal_hold` bloqueia poda automática, exceto RTBF autorizado e auditado | LGPD/GDPR | Teste: expurgo automático de item `legal_hold` → `AIOS-MEM-0050`; RTBF autorizado → `AIOS-MEM-0051` e purge efetivo. |
| SEC-M-13 | `legal_basis`/`retention_class`/`pii` obrigatórios em todo `MemoryItem` | LGPD/GDPR | Validação de escrita: ausência de campo → `AIOS-MEM-0001`. |
| SEC-M-14 | ACL Redis e *bucket policy* MinIO restritas a comandos/prefixos necessários | Hardening | *Scan* de configuração de ACL/*bucket policy* em CI/CD. |

---

## 14. Referências

- Contratos centrais de segurança: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7.
- Modelo de política (PDP): `../022-Policy/`.
- Segurança transversal (PKI, OIDC, sandbox de runtime): `../021-Security/`.
- Auditoria imutável: `../025-Audit/`.
- Model Router (embeddings): `../017-Model-Router/`.
- Glossário: `../040-Glossary/Glossary.md` (termos: Default Deny, PEP/PDP, mTLS, STRIDE, RLS, LGPD/GDPR, Embedding).
- Brief interno do módulo: `./_DESIGN_BRIEF.md` §1.2 (N1, N5, N6), §12.
- Arquitetura e componentes: `./Architecture.md`.
- Deployment (hardening de infraestrutura): `./Deployment.md`.
- Escalabilidade (isolamento de *noisy neighbor*, sharding): `./Scalability.md`.
- Recuperação de falhas (fail-closed sob indisponibilidade de PDP/dependências, degradação graciosa): `./FailureRecovery.md`.
- Máquina de estado (`legal_hold`, `FORGET_PENDING`, `PURGED`): `./StateMachine.md`.
- Catálogo de erros: `_DESIGN_BRIEF.md` §5.2 (`AIOS-MEM-0001`–`0060`).
