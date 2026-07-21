---
Documento: Security
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (Admission Control com reserva atômica), ADR-0094 (Preempção compensável), ADR-0095 (Backpressure em três níveis), ADR-0097 (Outbox transacional de decisão); ADR-0008 (Governança por Política/Default Deny), ADR-0010 (Observabilidade/Auditoria por Construção) — herdadas
RFCs relacionados: RFC-0001 (baseline); RFC-0090, RFC-0091 (a propor)
Depende de: 021-Security, 022-Policy, 025-Audit, 026-Cost-Optimizer, 027-Cluster, 001-Architecture
---

# 009-Scheduler — Security

> **Escopo.** Este documento especifica a postura de segurança do Scheduler
> cognitivo: como ele autentica chamadores, como aplica autorização como **PEP
> (Policy Enforcement Point)** para admissão/preempção/mutação de política, sua
> superfície de ataque, o *threat model* STRIDE, a gestão de segredos e
> TLS/mTLS, os controles de isolamento entre dependências e a conformidade
> LGPD/GDPR. Este documento **NÃO redefine** os contratos centrais de segurança
> (mTLS interno, validação OIDC no Gateway, envelope de erro RFC 7807) — eles
> são normativos em `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7 e
> detalhados em `../021-Security/`. Aqui, especifica-se **como o Scheduler os
> consome e os aplica** em sua fronteira de confiança específica, conforme
> `_DESIGN_BRIEF.md` §12.

---

## Índice

1. Postura de Segurança e Princípios
2. Autenticação (AuthN)
3. Autorização (AuthZ) — o Scheduler como PEP
4. Superfície de Ataque
5. Threat Model STRIDE (detalhado)
6. Gestão de Segredos
7. TLS/mTLS — Fronteiras de Confiança
8. Isolamento entre Dependências (Bulkhead)
9. Hardening de Infraestrutura
10. LGPD / GDPR
11. Auditoria e Não-Repúdio
12. Matriz de Controles (resumo executivo)
13. Referências

---

## 1. Postura de Segurança e Princípios

O Scheduler é o **guardião da admissão** do plano de controle: toda tarefa que
consome recurso computacional/cognitivo no AIOS passa, direta ou
indiretamente, por sua decisão. Comprometer o Scheduler equivale a comprometer
o controle de recursos de todo o sistema — daí a postura de segurança ser
particularmente rígida em torno de quatro princípios, todos rastreáveis às
Responsabilidades R-01/R-04/R-08 e Não-Responsabilidades N-03/N-09 do
`_DESIGN_BRIEF.md`:

| Princípio | Descrição | Consequência de design |
|-----------|-----------|-------------------------|
| **Default Deny** | Nenhuma admissão ou preempção é efetivada sem decisão explícita de `allow` do PDP (`022-Policy`). | `AdmissionController` e `PreemptionManager` bloqueiam por padrão; ausência de decisão = negação (`AIOS-SCHED-0004`). |
| **PEP, não PDP** | O Scheduler **NÃO DEVE** decidir política de autorização localmente; ele apenas a aplica. | Nenhuma lógica de RBAC/ABAC é hardcoded no Scheduler — pesos/cotas/thresholds são parâmetros de *scheduling*, não decisões de acesso. |
| **Fail-Closed sob dependência crítica** | Sob indisponibilidade do PDP, o Scheduler nega em vez de permitir; sob indisponibilidade de Redis, entra em modo conservador de rejeição (sem overcommit). | `AIOS-SCHED-0004` (PDP indisponível) e `AIOS-SCHED-0003` (backpressure/Redis indisponível) — ver `FailureRecovery.md`. |
| **Isolamento de Tenant Absoluto** | Nenhuma decisão, fila, cota ou proveniência atravessa a fronteira de `tenant_id` sem autorização explícita. | RLS em `SchedulingDecision`; namespace Redis (`sched:{tenant}:...`) e NATS (`aios.<tenant>.*`) por tenant. |

Este documento assume que o leitor já conhece os contratos centrais da RFC-0001
(URN, envelope de evento, envelope de erro RFC 7807, `Idempotency-Key`,
`traceparent`/`X-AIOS-Tenant`) e **não os repete**, exceto quando referenciados
para explicar um controle específico do Scheduler.

---

## 2. Autenticação (AuthN)

O Scheduler **NÃO DEVE** emitir, validar ou renovar tokens de identidade — essa
responsabilidade pertence ao Gateway (YARP) em conjunto com `021-Security`. O
Scheduler consome apenas **claims já validadas** e propagadas por canal
autenticado.

### 2.1 Fluxo de autenticação do chamador

| Etapa | Ator | Ação |
|-------|------|------|
| 1 | Kernel (006) / Web Console (032) / CLI (030) | Envia `SchedulingRequest` (gRPC interno ou REST via Gateway) com identidade já estabelecida. |
| 2 | Gateway (YARP) — apenas para tráfego REST/admin | Valida JWT (OIDC) contra `021-Security`; extrai claims (`sub`, `tenant`, `roles`) e as propaga como cabeçalhos internos assinados via mTLS. |
| 3 | `SchedulingGateway` (Scheduler, PEP) | Para tráfego gRPC interno (Kernel→Scheduler), valida identidade de workload via mTLS mútuo (SPIFFE/SVID); para tráfego via Gateway, valida que os cabeçalhos internos chegaram por canal mTLS autenticado (ver §2.3). |
| 4 | `SchedulingGateway` | Extrai `tenant`, `traceparent` e (quando presente) `X-AIOS-Agent` para o contexto da decisão; valida `Idempotency-Key` obrigatória em toda mutação (RFC-0001 §5.5). |

### 2.2 Autenticação interna serviço-a-serviço (mTLS)

Toda comunicação do Scheduler com dependências internas **DEVE** usar **mTLS**
com certificados de curta duração emitidos e rotacionados pela infraestrutura
de PKI de `021-Security` (RFC-0001 §6).

| Canal | Protocolo | Certificado | Rotação |
|-------|-----------|-------------|---------|
| Kernel (006) → SchedulingGateway | gRPC + mTLS | SPIFFE/SVID por workload | ≤ 24h |
| Scheduler → `PolicyClient` (022) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Scheduler → `BudgetClient` (026) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Scheduler → `RuntimeSupervisorClient` (007) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Scheduler → `CapacityProvider`/inventário (027) | gRPC/eventos + mTLS | SPIFFE/SVID | ≤ 24h |
| Scheduler → PostgreSQL (`DecisionJournal`) | TLS 1.3 + client cert | Certificado de serviço | 90 dias, renovação automática |
| Scheduler → Redis (`PriorityQueueStore`, `QuotaLedger`) | TLS 1.3 + AUTH | Certificado de serviço + senha rotacionada | 90 dias |
| Scheduler → NATS/JetStream | TLS 1.3 + NKey/JWT de conta | NATS account JWT | 30 dias |

### 2.3 Validação de cabeçalhos internos assinados

Como o Scheduler confia nas claims propagadas pelo Gateway (para tráfego
administrativo REST) sem revalidar o JWT original, ele **DEVE** validar que os
cabeçalhos internos (`X-AIOS-Tenant`, `X-AIOS-Subject`, `X-AIOS-Roles`)
chegaram por um canal mTLS autenticado cujo certificado de cliente pertence ao
pool de identidades confiáveis do Gateway (*workload identity allowlist*).
Requisições sem essa identidade de workload autenticada **DEVEM** ser
rejeitadas com `401` antes de qualquer processamento de `SchedulingRequest` —
isso impede que um serviço comprometido da malha injete claims forjadas
diretamente no Scheduler, contornando o Gateway. Para o caminho gRPC
interno (Kernel→Scheduler), a identidade do chamador é o próprio certificado
mTLS do workload — não há cabeçalho a validar, mas o `SchedulingGateway`
**DEVE** verificar que o SPIFFE ID do chamador pertence à allowlist de
serviços autorizados a chamar `Submit`/`Cancel`/`Preempt`.

---

## 3. Autorização (AuthZ) — o Scheduler como PEP

### 3.1 Modelo RBAC + ABAC (decidido externamente)

O Scheduler implementa exclusivamente o **ponto de aplicação** (PEP); o modelo
de papéis (RBAC) e atributos (ABAC) é definido e avaliado pelo PDP em
`022-Policy`. Toda operação privilegiada — admissão, preempção e mutação de
política de pesos/cotas — **DEVE** passar por uma consulta ao PDP via
`PolicyClient` antes de ser efetivada.

| Operação | Verbo/rota | Consulta PDP obrigatória | Roles esperadas |
|----------|-----------|----------------------------|-------------------|
| `Submit` (admissão) | gRPC `Submit` / `POST /v1/scheduler/requests` | Sim (`Authorize(subject, "schedule.submit", task_urn, ctx)`) | `scheduler-submit` |
| `Cancel` | gRPC `Cancel` / `DELETE /v1/scheduler/requests/{id}` | Sim (verifica que o chamador é dono do `request_id` ou possui `scheduler-admin`) | `scheduler-submit` (próprio) ou `scheduler-admin` |
| `Preempt` (dirigida) | gRPC `Preempt` / `POST /v1/scheduler/preemptions` | Sim, sempre | `scheduler-admin` |
| `PUT /v1/scheduler/policy` (pesos α/β/γ, cotas) | REST admin | Sim, sempre | `scheduler-admin` |
| `GET /v1/scheduler/policy`, `/queues`, `/capacity`, `GetDecision`, `StreamQueueState` | Leitura | Verificação de escopo de leitura (isolamento de tenant, §3.4); PDP consultado apenas se a política exigir escopo restrito de observabilidade. | `scheduler-submit` ou `scheduler-admin` (leitura própria) |

O `DecisionRequest` montado pelo `PolicyClient` **DEVE** conter, no mínimo:

| Campo | Origem | Descrição |
|-------|--------|-----------|
| `subject` | Claims propagadas / SPIFFE ID do workload chamador | Quem solicita (agente pai, service account, operador humano). |
| `action` | Operação da tabela acima (`schedule.submit`, `schedule.preempt`, `schedule.policy.update`, ...) | Ação sobre a qual se decide. |
| `resource` | URN do `task_urn`/`agent_urn` alvo, ou `urn:aios:<tenant>:scheduler:policy` para mutação de pesos | Sobre o que a ação incide. |
| `environment` | `tenant`, `service_class`, `priority_hint`, hora, contexto de orçamento | Contexto para regras ABAC (ex.: negar preempção de tarefas `SYSTEM` fora de janela de manutenção). |

### 3.2 Ciclo de decisão e cache

```
   SchedulingGateway        AdmissionController          PolicyClient            022-Policy (PDP)
        │                        │                            │                        │
        │  SchedulingRequest     │                             │                        │
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
        │◀── deny → 403 SCHED-0004─│                           │                        │
```

- Decisões de `allow` **PODEM** ser cacheadas por até um TTL curto (segundos,
  ordem de grandeza compatível com o caminho quente de `Submit`, p99 ≤ 20 ms —
  NFR-001), **nunca** superior ao TTL retornado pelo PDP.
- Decisões de `deny` **NÃO DEVEM** ser cacheadas com TTL maior que o de
  `allow`, para permitir revogação rápida de acesso.
- Uma atualização de *policy bundle* (evento
  `aios.<tenant>.policy.decision.updated`, consumido pelo Scheduler — brief
  §6.2) **DEVE** invalidar o cache local de decisão do `PolicyClient` para o
  tenant afetado, independentemente do TTL restante.
- O cache de decisão de política é **estritamente distinto** do cache de
  orçamento (`BudgetClient`) e de capacidade (`CapacityProvider`) — cada um tem
  sua própria política de invalidação (ver `_DESIGN_BRIEF.md` §9).

### 3.3 Fail-Closed sob indisponibilidade do PDP

| Comportamento sob timeout/erro do PDP | Consequência |
|------------------------------------------|----------------|
| **Default** (produção) | Nega a operação privilegiada (`AIOS-SCHED-0004`, HTTP 403, `retriable=false` — a negação em si não é retriável, mas o chamador PODE tentar novamente após o PDP se recuperar). |
| Circuit breaker aberto no `PolicyClient` | Toda nova operação que exija PDP é imediatamente negada sem nova tentativa de rede, evitando amplificar carga sobre um PDP já degradado. |

Não há modo "fail-open" configurável para operações de admissão/preempção no
Scheduler — ao contrário do Kernel (que PODE permitir *fail-open* apenas em
ambientes não produtivos e sob exceção documentada), o Scheduler **NÃO DEVE**
jamais admitir ou preemptar sem decisão explícita do PDP, dado o impacto
sistêmico de uma admissão indevida (consumo de orçamento/capacidade de todo o
tenant/sistema).

### 3.4 Isolamento multi-tenant como controle de autorização

Antes mesmo de consultar o PDP, o `SchedulingGateway` **DEVE** validar que o
`tenant` presente nas URNs de `task_urn`/`agent_urn` da `SchedulingRequest` é
idêntico ao `tenant` do contexto autenticado (`X-AIOS-Tenant` ou tenant
derivado da identidade de workload mTLS). Divergência **DEVE** ser rejeitada
com `AIOS-SCHED-0005` (requisição inválida, 400) **antes** de qualquer chamada
ao PDP — verificação estrutural, barata e determinística, e o controle
primário contra vazamento cross-tenant (mapeado à ameaça *Information
Disclosure* no STRIDE, §5). Adicionalmente:

- Filas Redis são particionadas por `sched:{tenant}:{class}:{shard}` — não há
  estrutura de dados compartilhada entre tenants que permita um tenant influir
  na fila de outro.
- `SchedulingDecision` carrega `tenant_id` com RLS ativo — uma consulta
  `GetDecision` para um `request_id` de outro tenant retorna
  `AIOS-SCHED-0009` (não encontrado), nunca vazamento de existência.
- Namespace NATS (`aios.<tenant>.*`) garante que eventos de decisão de um
  tenant não sejam entregues a consumidores de outro.

---

## 4. Superfície de Ataque

### 4.1 Inventário de pontos de entrada

| Superfície | Protocolo | Exposição | Controle primário |
|------------|-----------|-----------|---------------------|
| `SchedulingGateway` gRPC (`aios.scheduler.v1.SchedulerService`) | gRPC + mTLS | Interna (Kernel 006, Runtime Supervisor 007, malha de serviços) | mTLS workload identity + PEP + `Idempotency-Key` |
| `SchedulingGateway` REST (`/v1/scheduler/*`) | HTTPS via Gateway/YARP | Externa/administrativa (Web Console 032, CLI 030, operadores) | AuthN no Gateway (OIDC) + PEP + rate-limit |
| Eventos consumidos (`aios.<tenant>.agent.lifecycle.*`, `cost.budget.updated`, `cluster.capacity.changed`, `policy.decision.updated`) | NATS/JetStream (TLS) | Interna | Conta NATS por serviço; validação de `dataschema`; deduplicação por `event.id` |
| Conexão PostgreSQL (`DecisionJournal`, `IdempotencyRecord`) | TLS + client cert | Interna | RLS por `tenant_id`; credenciais de menor privilégio |
| Conexão Redis (`PriorityQueueStore`, `QuotaLedger`, leases) | TLS + AUTH | Interna | Namespace de chave por tenant; ACL Redis restrita a comandos necessários (scripts Lua, sem `FLUSHALL`/`KEYS *`) |
| Endpoints de observabilidade (`/metrics`, `/healthz`, `/readyz`) | HTTP interno | Interna (scrape Prometheus/probe) | Sem dados sensíveis; acesso restrito à rede de observabilidade (ver `Deployment.md`) |

### 4.2 Vetores de ataque considerados e descartados

| Vetor | Considerado? | Justificativa |
|-------|---------------|----------------|
| Injeção via `affinity`/`payload_ref` (jsonb) de `SchedulingRequest` | Sim, mitigado | Campos tratados como dados opacos (nunca interpolados em query SQL ou comando Redis); acesso via ORM/queries parametrizadas e scripts Lua fixos. |
| Forjamento de `tenant` em `task_urn`/`agent_urn` | Sim, mitigado | Validação estrutural §3.4 antes de qualquer processamento; RLS como segunda camada (*defense in depth*). |
| Replay de `Submit`/`Cancel`/`Preempt` | Sim, mitigado | `Idempotency-Key` obrigatório (RFC-0001 §5.5) + `IdempotencyStore`; reprodução retorna o mesmo resultado (`response_snapshot`), sem dupla admissão (NFR-011). |
| Manipulação direta de contador de cota (`QuotaLedger`) para burlar `max_concurrency` | Sim, mitigado | Reserva atômica via script Lua no Redis; ACL Redis restringe comandos a um *allowlist* que não inclui escrita arbitrária de chave. |
| Preempção não autorizada (agente comum tentando preemptar outro tenant/tarefa) | Sim, mitigado | `Preempt` dirigida exige role `scheduler-admin` + consulta PDP obrigatória (§3.1, §3.3). |
| Exaustão de filas via *flood* de `Submit` (DoS) | Sim, mitigado | Cota de concorrência por tenant/classe (`QuotaLedger`) + `BackpressureController` (`AIOS-SCHED-0003`) + rate-limit de borda no Gateway (*defense in depth*). |
| Envenenamento de `feature_vector`/proveniência para manipular treino de política RL (v2) | Sim, mitigado | `feature_vector` é derivado por `FeatureExtractor` a partir de sinais internos (carga, fila, custo estimado) — o chamador externo **não** fornece o vetor diretamente; `priority_hint` é apenas uma sugestão não vinculante (brief §3.1), nunca usada como score final sem passar por `ISchedulingPolicy`. |
| Execução de código arbitrário no processo do Scheduler | Não aplicável | O Scheduler **não** executa código de agente/ferramenta — é um serviço de decisão puro; execução ocorre em `007-Agent-Runtime` (sandbox, fora do escopo deste módulo). |
| Exfiltração de proveniência de decisão de outro tenant via `GetDecision`/`StreamQueueState` | Sim, mitigado | RLS + validação de tenant no `SchedulingGateway`; `StreamQueueState` filtra por `X-AIOS-Tenant` do chamador, nunca expõe filas de outro tenant. |

---

## 5. Threat Model STRIDE (detalhado)

> A versão resumida está no `_DESIGN_BRIEF.md` §12.2; esta seção a detalha por
> componente interno e aponta o controle técnico exato.

| Ameaça | Componente afetado | Cenário concreto | Controle técnico | Detecção |
|--------|----------------------|-------------------|---------------------|----------|
| **Spoofing** | `SchedulingGateway` | Serviço comprometido injeta `X-AIOS-Tenant` diretamente, contornando o Gateway. | mTLS workload identity allowlist (§2.3); rejeição sem identidade válida. | Alerta em taxa anômala de `401` (`aios_scheduler_authn_rejected_total`). |
| **Spoofing** | Eventos consumidos | Publicador não autorizado emite `aios.<tenant>.cost.budget.updated` forjado, inflando orçamento aparente. | Conta NATS por serviço com permissão de publish restrita ao subject próprio; validação de `source` no envelope CloudEvents. | Auditoria de publishers por subject (025). |
| **Tampering** | `PriorityQueueStore` (Redis ZSET) | Alteração direta de `score` de uma entrada para burlar prioridade. | Acesso exclusivo via API do Scheduler (nenhum outro serviço tem credencial de escrita nas chaves `sched:*`); ACL Redis restrita a scripts Lua conhecidos. | Reconciliação periódica (`ReconciliationWorker`) detecta score fora do padrão esperado para a idade da entrada. |
| **Tampering** | `SchedulingDecision` (PostgreSQL) | Tentativa de `UPDATE`/`DELETE` em registro de decisão (event sourcing violado). | `DecisionJournal` é a única via de escrita; tabela é append-only por design de aplicação e permissão de banco (usuário de serviço sem privilégio `UPDATE`/`DELETE` na tabela). | Alerta de tentativa de escrita não permitida (log de banco). |
| **Repudiation** | Toda operação privilegiada | Chamador nega ter submetido/cancelado/preemptado. | `DecisionJournal` append-only + evento `scheduler.decision.recorded` → 025-Audit (imutável), com `subject`, `request_id`, `decision_id`, `trace_id`. | Consulta de auditoria por `trace_id`/`request_id`/`decision_id`. |
| **Information Disclosure** | `SchedulingGateway` (erros) | Mensagem de erro vaza detalhe interno (query SQL, string de conexão Redis). | Envelope RFC 7807 padronizado (RFC-0001 §5.4); `detail` sanitizado, sem PII/segredos/dados internos. | Revisão de amostras de log de erro em CI (regra de lint de log). |
| **Information Disclosure** | Cross-tenant | Tenant A consulta `GetDecision`/`StreamQueueState` de recurso do tenant B. | Validação estrutural de tenant (§3.4) + RLS de banco + filtro de namespace Redis/NATS como segunda e terceira camadas (*defense in depth*). | `AIOS-SCHED-0009`/`0005` monitorados; alerta em taxa anômala por tenant. |
| **Information Disclosure** | `SchedulingRequest.payload_ref` | Payload de tarefa (potencialmente sensível) trafega inline em vez de por referência. | Campo `payload_ref` é a única forma normativa de referenciar conteúdo de negócio; `SchedulingRequest` **NÃO DEVE** carregar payload inline (brief §12.3); validação de contrato em CI. | Varredura de schema de amostras de `SchedulingRequest` sem campos de payload de negócio. |
| **Denial of Service** | `SchedulingGateway` | *Flood* de `Submit` de um único tenant satura filas/capacidade de todos. | Cota de concorrência por tenant/classe (`QuotaLedger`) + `BackpressureController` em três níveis (`AIOS-SCHED-0001`/`0003`) + rate-limit de borda no Gateway. | Métrica de rejeição por motivo (`aios_scheduler_admission_rejected_total{reason=...}`). |
| **Denial of Service** | `PolicyClient`/`BudgetClient` | PDP ou Cost-Optimizer lentos degradam latência de toda admissão. | Circuit breaker por dependência (bulkhead, §8); `fail_mode` conservador isola o impacto a negação/estimativa cacheada, não a bloqueio total. | `aios_scheduler_decision_latency_ms` p99 acima da meta (NFR-001) dispara alerta. |
| **Elevation of Privilege** | `PreemptionManager` | Bypass do PEP por chamada direta a um handler interno de preempção. | Nenhum handler de preempção é alcançável sem passar pelo pipeline `SchedulingGateway → PolicyClient`; garantido por design de camadas (sem rota alternativa registrada). | Cobertura de teste de contrato (`Testing.md`) valida ausência de rota sem PEP; auditado por pentest. |
| **Elevation of Privilege** | `PUT /v1/scheduler/policy` | Alteração não autorizada de pesos `alpha`/`beta`/`gamma` ou cotas para favorecer um tenant. | Requer role `scheduler-admin` + consulta PDP obrigatória (§3.1); toda alteração gera evento auditável. | Auditoria de mudança de política (025); alerta de mudança fora de janela de manutenção. |

---

## 6. Gestão de Segredos

| Segredo | Tipo | Armazenamento | Rotação | Acesso |
|---------|------|-----------------|---------|--------|
| Certificados mTLS (SPIFFE/SVID) do Scheduler | Certificado X.509 de curta duração | Emitido em runtime pela infraestrutura de PKI (sidecar/SDS), nunca em disco persistente | ≤ 24h automática | Somente o processo do Scheduler via socket local do agente de identidade |
| Credencial PostgreSQL (`DecisionJournal`) | Usuário/senha + client cert | Cofre de segredos (Vault/KMS gerenciado por `021`), injetado como variável de ambiente/arquivo montado em `tmpfs` | 90 dias ou sob incidente | Processo do Scheduler apenas |
| Credencial Redis (AUTH) | Senha | Cofre de segredos | 90 dias ou sob incidente | Processo do Scheduler apenas |
| NATS account JWT/NKey | JWT assinado | Cofre de segredos | 30 dias | Processo do Scheduler apenas |
| Chaves de assinatura de cabeçalho interno (Gateway↔Scheduler) | Chave simétrica/assimétrica | Cofre de segredos, sincronizada entre Gateway e Scheduler | 30 dias | Gateway (assina) e Scheduler (verifica) |

**Regras normativas:**

- Nenhum segredo **DEVE** ser escrito em log, trace, métrica ou payload de
  evento (RFC-0001 §7 — minimização).
- Nenhum segredo **DEVE** ser embutido em imagem de container ou em
  código-fonte; todo segredo é injetado em tempo de execução (ver
  `Deployment.md` §3).
- Segredos **DEVEM** ser lidos apenas na inicialização ou via *hot reload*
  seguro (sem persistir cópia em disco além do necessário para o processo).
- Rotação de segredo **NÃO DEVE** exigir *downtime*: o Scheduler **DEVE**
  suportar recarregamento de credencial sem reinício (pools de conexão a
  Redis/PostgreSQL renovados de forma gradual).

---

## 7. TLS/mTLS — Fronteiras de Confiança

```
        Zona Não-Confiável           Zona de Borda (DMZ)              Zona Interna (Malha mTLS)
   ┌────────────────────┐      ┌──────────────────────┐      ┌───────────────────────────────────┐
   │  Operador/Console   │      │   Gateway (YARP)     │      │        SCHEDULER (009)             │
   │  (032/030) externo  │─TLS─▶│  - Valida OIDC/JWT   │─mTLS─▶│  SchedulingGateway (PEP)           │
   │                     │ 1.3  │  - Assina cabeçalhos │      │  AdmissionController                │
   └────────────────────┘      │    internos           │      │  PriorityCostEvaluator · Placement  │
                                └──────────────────────┘      │  PreemptionManager · DispatchLoop   │
   ┌────────────────────┐                                     └───────────────┬───────────────────┘
   │   Kernel (006)      │──────────────── mTLS direto (gRPC interno) ────────▶│
   └────────────────────┘                                                      │ mTLS (todas as setas)
                                                        ┌──────────────┬────────┼─────────┬────────────┬───────────┐
                                                        ▼              ▼        ▼         ▼            ▼           ▼
                                                  022-Policy   026-Cost-Opt  027-Cluster 007-RTSup  PostgreSQL   Redis/NATS
```

- **Fora → Borda**: TLS 1.3 padrão de mercado (SDK/browser/CLI → Gateway).
- **Kernel → Scheduler**: mTLS interno direto (gRPC, sem passar pelo Gateway
  externo — tráfego control-plane→control-plane).
- **Borda → Interna** e **Interna → Interna**: mTLS obrigatório em toda
  chamada (RFC-0001 §6); nenhuma comunicação de serviço a serviço **DEVE**
  ocorrer em texto plano ou apenas TLS unilateral.
- Cifras **DEVEM** seguir o perfil moderno definido por `021-Security` (TLS 1.3
  preferencial; TLS 1.2 apenas com conjuntos de cifra aprovados como fallback
  transitório).

---

## 8. Isolamento entre Dependências (Bulkhead)

O Scheduler **não hospeda** sandbox de execução de agente (isso é
`007-Agent-Runtime`, N-01/N-02 do brief). Seus controles de isolamento são de
**bulkhead** e **circuit breaker** entre dependências, para que a falha ou
degradação de uma dependência não se propague e não force o Scheduler a
overcommitar recurso por indisponibilidade parcial:

| Dependência | Padrão de isolamento | Efeito de falha isolado |
|-------------|------------------------|----------------------------|
| PDP (`022-Policy`) | Bulkhead dedicado no `PolicyClient` + circuit breaker | Falha do PDP nega admissão/preempção (`AIOS-SCHED-0004`); não afeta leitura de estado de fila (`GetDecision`/`StreamQueueState`). |
| Cost-Optimizer (`026`) | Bulkhead no `BudgetClient`/`CostEstimator` | Falha degrada para última estimativa cacheada + margem conservadora; apenas orçamentos no limite são rejeitados (`AIOS-SCHED-0002`), sem bloquear todo o caminho de admissão. |
| Cluster (`027`)/inventário | Bulkhead no `CapacityProvider` | Snapshot expirado é tratado como capacidade zero para o shard afetado (fail-safe), sem afetar shards com dado fresco. |
| Runtime Supervisor (`007`) | Bulkhead no `RuntimeSupervisorClient` | Falha de heartbeat/dispatch não corrompe `PriorityQueueStore`; `ReconciliationWorker` expira lease e reenfileira. |
| Redis (`PriorityQueueStore`/`QuotaLedger`) | Circuit breaker de infraestrutura | Indisponibilidade força modo conservador de rejeição (`AIOS-SCHED-0003`) em vez de decisão sem garantia de não-overcommit. |
| PostgreSQL (`DecisionJournal`) | Circuit breaker de infraestrutura | Indisponibilidade degrada para buffer local + retry; persistência de decisão nunca é pulada silenciosamente (ver `FailureRecovery.md`). |

Cada bulkhead **DEVE** ter *pool* de conexões/threads dedicado e limite
próprio, de forma que a saturação de uma dependência não esgote recursos
compartilhados usados por chamadas a outras dependências (isolamento de
"vizinho barulhento" em nível de dependência, complementando o isolamento por
tenant descrito em `Scalability.md`).

---

## 9. Hardening de Infraestrutura

| Controle | Especificação |
|----------|-----------------|
| Usuário do processo | Container do Scheduler **DEVE** executar como usuário não-root (UID dedicado, sem `CAP_SYS_ADMIN` ou capacidades Linux desnecessárias). |
| Sistema de arquivos | Sistema de arquivos raiz **DEVERIA** ser montado somente-leitura; diretórios graváveis restritos a `/tmp` efêmero e diretório de segredos em `tmpfs`. |
| Rede | Políticas de rede (NetworkPolicy/equivalente) **DEVEM** restringir egress do Scheduler apenas às dependências listadas em §4.1; nenhum egress arbitrário à internet. |
| Imagem | Imagem base mínima (distroless ou equivalente), sem *shell* interativo em produção; scan de vulnerabilidade (SCA) obrigatório no pipeline de CI. |
| Segredos em imagem | **NÃO DEVE** haver segredo, chave privada ou credencial embutida em camada de imagem (verificado por *secret scanning* em CI). |
| Dependências | *Software Composition Analysis* (SCA) contínuo sobre pacotes .NET 10; CVEs críticas **DEVEM** ser corrigidas dentro do SLA de patch de `021-Security`. |
| ACL Redis | Comandos permitidos ao Scheduler restritos a um *allowlist* (`ZADD`, `ZRANGEBYSCORE`, `EVALSHA` de scripts conhecidos, etc.); comandos administrativos (`FLUSHALL`, `CONFIG SET`, `KEYS *`) **NÃO DEVEM** estar acessíveis à credencial de serviço. |

---

## 10. LGPD / GDPR

- **Minimização de dados**: o Scheduler **NÃO DEVE** persistir payload de
  tarefa; usa `payload_ref` (URN/URI para MinIO) conforme `SchedulingRequest`
  (brief §3.1). O `feature_vector` de `SchedulingDecision` **NÃO DEVE** conter
  PII bruta — apenas sinais numéricos derivados (carga, custo estimado, folga
  de SLA, histórico agregado).
- **Retenção**: `IdempotencyRecord` tem TTL configurável
  (`scheduler.idempotency.ttl_hours`, default 24h, mínimo 24h por RFC-0001
  §5.5); `SchedulingDecision` segue política de retenção do
  `DecisionJournal` definida em `Database.md`, com expurgo rastreável.
- **Direito ao esquecimento**: uma operação de expurgo por
  `tenant_id`/`agent_urn` **DEVE** ser propagada e auditada (coordenada com
  `025-Audit`), removendo entradas de fila ativas, `IdempotencyRecord` e, para
  `SchedulingDecision`, mantendo apenas os metadados mínimos exigidos para
  auditoria/conformidade legal (identificadores, timestamps, `outcome`,
  `policy_id`) e removendo qualquer campo de negócio residual.
- **Segregação**: RLS + namespaces NATS/Redis garantem que dados de um tenant
  não vazem para decisões de outro (§3.4).
- **Base legal**: o Scheduler não é o titular de base legal de dados pessoais
  (isso é responsabilidade dos módulos de memória/contexto, `010`/`011`); ele
  apenas processa metadados de controle (URNs, scores, timestamps) necessários
  à decisão de escalonamento.

---

## 11. Auditoria e Não-Repúdio

Toda operação privilegiada (admissão, rejeição, preempção, mutação de
política) **DEVE** gerar:

1. Um registro em `SchedulingDecision` (`Database.md`) com `outcome`,
   `reason_code` (se rejeição/negação), `policy_id`/`policy_version`,
   `decision_id`, `request_id`.
2. Um evento de auditoria imutável enviado a `025-Audit`, contendo os mesmos
   campos de correlação (RFC-0001 §5.6) — `aios.<tenant>.scheduler.decision.recorded`
   (brief §6.1).
3. Em caso de negação por política, o evento correspondente
   (`aios.<tenant>.task.execution.rejected` com `reasonCode=AIOS-SCHED-0004`)
   publicado no NATS para consumo por sistemas de alerta de segurança.

Essa trilha **DEVE** ser suficiente para responder, para qualquer decisão de
escalonamento, às perguntas: *quem* solicitou, *o quê*, *sobre qual tarefa/agente*,
*quando*, *qual foi a decisão do PDP*, *qual política/versão decidiu* e *qual o
resultado* — sem exigir correlação manual entre sistemas (NFR-010: 100% das
decisões com trace/span e evento de proveniência).

---

## 12. Matriz de Controles (resumo executivo)

| ID | Controle | Categoria | Verificação |
|----|----------|-----------|--------------|
| SEC-S-01 | Default deny em toda admissão/preempção | AuthZ | Teste de contrato: `Submit`/`Preempt` sem decisão PDP `allow` → `AIOS-SCHED-0004`. |
| SEC-S-02 | Fail-closed sob indisponibilidade do PDP (sem fallback fail-open em produção) | AuthZ/Resiliência | Chaos test: derrubar PDP → operações privilegiadas negadas, nunca admitidas por default. |
| SEC-S-03 | Validação estrutural de `tenant` antes do PDP | Isolamento | Teste: `tenant` divergente em URN → `AIOS-SCHED-0005` sem chamada ao PDP. |
| SEC-S-04 | mTLS obrigatório em toda comunicação interna | Transporte | Scan de configuração + teste de conexão sem cert válido é rejeitada. |
| SEC-S-05 | RLS por `tenant_id` em `SchedulingDecision`/`SchedulingRequest` | Isolamento | Teste automatizado de RLS (query cross-tenant retorna vazio). |
| SEC-S-06 | Nenhum segredo em imagem/log/evento | Segredos | Secret scanning em CI + regra de lint de log. |
| SEC-S-07 | `Idempotency-Key` obrigatório em toda mutação | Integridade | Teste de replay: mesma chave → mesmo `response_snapshot`, sem dupla admissão (NFR-011). |
| SEC-S-08 | Auditoria imutável de toda decisão de escalonamento | Repúdio | Cobertura auditada (NFR-010): 100% das decisões com evento de proveniência. |
| SEC-S-09 | Bulkhead/circuit breaker por dependência externa | Isolamento de falha | Chaos test por dependência isolada (§8). |
| SEC-S-10 | Preempção sempre autorizada pelo PDP, sem exceção | AuthZ | Teste: nenhuma vítima suspensa sem `Authorize()` retornando `allow` prévio. |
| SEC-S-11 | Expurgo rastreável de decisões/filas sob pedido de esquecimento | LGPD/GDPR | Teste de expurgo: dado de negócio removido, metadado mínimo de auditoria retido. |
| SEC-S-12 | ACL Redis restrita a comandos necessários (sem `FLUSHALL`/`KEYS *`) | Hardening | Scan de configuração de ACL do Redis em CI/CD. |

---

## 13. Referências

- Contratos centrais de segurança: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7.
- Modelo de política (PDP): `../022-Policy/`.
- Segurança transversal (PKI, OIDC, sandbox de runtime): `../021-Security/`.
- Auditoria imutável: `../025-Audit/`.
- Glossário: `../040-Glossary/Glossary.md` (termos: Capability, Default Deny, PEP/PDP, mTLS, STRIDE, RLS, LGPD/GDPR, Preemption, Backpressure).
- Brief interno do módulo: `./_DESIGN_BRIEF.md` §1.2 (N-03, N-09), §12.
- Arquitetura e componentes: `./Architecture.md`.
- Deployment (hardening de infraestrutura): `./Deployment.md`.
- Escalabilidade (isolamento de noisy neighbor, sharding): `./Scalability.md`.
- Recuperação de falhas (fail-closed sob indisponibilidade de PDP/dependências): `./FailureRecovery.md`.
- Catálogo de erros: `_DESIGN_BRIEF.md` §5.4 (`AIOS-SCHED-0001`–`0012`).
