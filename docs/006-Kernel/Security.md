---
Documento: Security
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0008 (Governança por Política/Default Deny), ADR-0010 (Observabilidade/Auditoria por Construção), ADR-0060 (ABI de Syscalls), ADR-0061 (ACB & OCC), ADR-0063 (PEP no Kernel vs. PDP no 022-Policy), ADR-0069 (Domínios de código de erro do Kernel)
RFCs relacionados: RFC-0001 (baseline), RFC-0006 (Cognitive Syscall ABI, a propor), RFC-0007 (ACB & Resource Quota Model, a propor)
Depende de: 021-Security, 022-Policy, 025-Audit, 020-Communication, 001-Architecture
---

# 006-Kernel — Security

> **Escopo.** Este documento especifica a postura de segurança do Kernel: como ele
> autentica chamadores, como aplica autorização como **PEP (Policy Enforcement
> Point)**, sua superfície de ataque, o *threat model* STRIDE, a gestão de
> segredos e TLS/mTLS, os controles de isolamento (sandbox/bulkhead) e a
> conformidade LGPD/GDPR. Este documento **NÃO redefine** os contratos centrais de
> segurança (mTLS interno, validação OIDC no Gateway, envelope de erro) — eles são
> normativos em `../003-RFC/RFC-0001-Architecture-Baseline.md` §6 e detalhados em
> `../021-Security/`. Aqui, apenas se especifica **como o Kernel os consome e os
> aplica** em sua fronteira de confiança específica.

---

## 1. Postura de Segurança e Princípios

O Kernel é a **fronteira de confiança** entre o plano de dados (loop de raciocínio
do agente, hospedado por `007-Agent-Runtime`) e os serviços governados do plano de
controle. Toda a postura de segurança do Kernel deriva de quatro princípios,
todos rastreáveis à Responsabilidade R-03/R-10 e Não-Responsabilidade NR-04/NR-07
do `_DESIGN_BRIEF.md`:

| Princípio | Descrição | Consequência de design |
|-----------|-----------|-------------------------|
| **Default Deny** | Nenhuma syscall privilegiada é executada sem decisão explícita de `allow` do PDP (`022-Policy`). | `CapabilityEnforcer` bloqueia por padrão; ausência de decisão = negação. |
| **PEP, não PDP** | O Kernel **NÃO DEVE** decidir política localmente; ele apenas a aplica. | Nenhuma lógica de RBAC/ABAC é hardcoded no Kernel; `capability_set` do ACB é *cache*, não autoridade. |
| **Fail-Closed** | Sob indisponibilidade de dependência de segurança (PDP), o Kernel nega em vez de permitir. | `kernel.pep.fail_mode=closed` (default); ver §5. |
| **Isolamento de Tenant Absoluto** | Nenhum dado, decisão ou recurso atravessa a fronteira de `tenant_id` sem autorização explícita. | RLS em toda tabela; contas NATS por tenant; validação de `tenant` em toda syscall. |

Este documento assume que o leitor já conhece os contratos centrais da RFC-0001
(URN, envelope de evento, envelope de erro RFC 7807, `Idempotency-Key`,
`traceparent`/`X-AIOS-Tenant`) e **não os repete**, exceto quando referenciados
para explicar um controle específico do Kernel.

---

## 2. Autenticação (AuthN)

O Kernel **NÃO DEVE** emitir, validar ou renovar tokens de identidade — essa
responsabilidade pertence ao Gateway (YARP) em conjunto com `021-Security`
(NR-07 do brief). O Kernel consome apenas **claims já validadas**.

### 2.1 Fluxo de autenticação do chamador externo

| Etapa | Ator | Ação |
|-------|------|------|
| 1 | Cliente (SDK/agente externo) | Envia requisição REST com `Authorization: Bearer <JWT>` ao Gateway. |
| 2 | Gateway (YARP) | Valida assinatura/expiração do JWT contra o IdP (OIDC) via `021-Security`; rejeita com `401` se inválido. |
| 3 | Gateway (YARP) | Extrai claims (`sub`, `tenant`, `roles`, `scopes`) e as propaga como cabeçalhos internos assinados (`X-AIOS-Tenant`, `X-AIOS-Subject`, `X-AIOS-Roles`) ao Kernel via mTLS. |
| 4 | `SyscallGateway` (Kernel) | Recebe a requisição já autenticada; **NÃO DEVE** revalidar o JWT; **DEVE** validar que os cabeçalhos internos foram assinados pelo Gateway (verificação de assinatura de cabeçalho interno, ver §2.3). |
| 5 | `SyscallGateway` (Kernel) | Extrai `tenant`, `agent_urn` (quando presente em `X-AIOS-Agent`) e `traceparent` (RFC-0001 §5.6) para o contexto da syscall. |

### 2.2 Autenticação interna serviço-a-serviço (mTLS)

Toda comunicação do Kernel com dependências internas (`022-Policy`, `009-Scheduler`,
`008-Lifecycle`, `010`/`011`/`012`/`015`/`017`, PostgreSQL, Redis, NATS) **DEVE**
usar **mTLS** com certificados de curta duração emitidos e rotacionados pela
infraestrutura de PKI de `021-Security` (RFC-0001 §6).

| Canal | Protocolo | Certificado | Rotação |
|-------|-----------|-------------|---------|
| Kernel → PolicyClient (022) | gRPC + mTLS | SPIFFE/SVID por workload | ≤ 24h (curta duração) |
| Kernel → SchedulerClient (009) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Kernel → LifecycleClient (008) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Kernel → ResourceBrokerRouter (010/011/012/015/017) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| Kernel → PostgreSQL | TLS 1.3 + client cert | Certificado de serviço | 90 dias, renovação automática |
| Kernel → Redis | TLS 1.3 + AUTH | Certificado de serviço + senha rotacionada | 90 dias |
| Kernel → NATS/JetStream | TLS 1.3 + NKey/JWT de conta | NATS account JWT | 30 dias |

### 2.3 Validação de cabeçalhos internos assinados

Como o Kernel confia nas claims propagadas pelo Gateway (e não revalida o JWT
original), ele **DEVE** validar que os cabeçalhos internos (`X-AIOS-Tenant`,
`X-AIOS-Subject`, `X-AIOS-Roles`) chegaram por um canal mTLS autenticado cujo
certificado de cliente pertence ao pool de identidades confiáveis do Gateway
(*workload identity allowlist*). Requisições que cheguem ao Kernel **sem** essa
identidade de workload autenticada **DEVEM** ser rejeitadas com `401` antes de
qualquer processamento de syscall — isso impede que um serviço comprometido
qualquer da malha injete claims forjadas diretamente no Kernel, contornando o
Gateway.

---

## 3. Autorização (AuthZ) — o Kernel como PEP

### 3.1 Modelo RBAC + ABAC (decidido externamente)

O Kernel implementa exclusivamente o **ponto de aplicação** (PEP); o modelo de
papéis (RBAC) e atributos (ABAC) é definido e avaliado pelo PDP em `022-Policy`.
O `CapabilityEnforcer` **DEVE**, para toda syscall privilegiada (todos os 11
verbos exceto leituras não sensíveis, ver Nota abaixo), montar um
`DecisionRequest` com no mínimo:

| Campo do `DecisionRequest` | Origem | Descrição |
|------------------------------|--------|-----------|
| `subject` | Claims propagadas (`X-AIOS-Subject`, `X-AIOS-Roles`) | Quem solicita (usuário, agente pai, service account). |
| `action` | Verbo da syscall (`spawn`, `kill`, `remember`, ...) | Ação cognitiva solicitada. |
| `resource` | URN do ACB alvo / recurso (`urn:aios:<tenant>:agent:<id>`) | Sobre o que a ação incide. |
| `environment` | `tenant`, `priority`, `quota_ref`, hora, IP de origem | Contexto para regras ABAC. |

> **Nota:** `get_quota` e leituras equivalentes (`GetAcb`) **PODEM** ter política
> de decisão mais permissiva (ex.: escopo "leitura do próprio agente"), mas ainda
> **DEVEM** passar pelo PEP — não há syscall que ignore o `CapabilityEnforcer" (NFR-010).

### 3.2 Ciclo de decisão e cache

```
   SyscallGateway         CapabilityEnforcer            PolicyClient            022-Policy (PDP)
        │                        │                            │                        │
        │  syscall(verb, urn)    │                             │                        │
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
        │◀── deny → 403 CAP-0001─│                             │                        │
```

- Decisões de `allow` **PODEM** ser cacheadas por até `kernel.pep.decision_cache_ttl_ms`
  (default 3000 ms) — nunca superior ao TTL retornado pelo PDP.
- Decisões de `deny` **NÃO DEVEM** ser cacheadas com TTL maior que o de `allow`, para
  permitir revogação rápida de acesso (ex.: revogação de capability em incidente).
- Uma atualização de *policy bundle* (evento `aios.<tenant>.policy.bundle.updated`,
  consumido pelo Kernel — brief §6.2) **DEVE** invalidar todo o cache local do
  `CapabilityEnforcer` para o tenant afetado, independentemente do TTL restante.

### 3.3 Fail-Closed sob indisponibilidade do PDP

| Configuração `kernel.pep.fail_mode` | Comportamento sob timeout/erro do PDP | Uso recomendado |
|--------------------------------------|----------------------------------------|-------------------|
| `closed` (**default**) | Nega a syscall (`AIOS-CAP-0003`, 503, `retriable=true`). | Produção. Alinhado a Default Deny. |
| `open` | Permite com decisão *stale* (última decisão em cache, se existir) ou nega se não houver cache. | Apenas ambientes de teste/homologação, **NUNCA** produção sem exceção documentada em ADR. |

O circuito (`PolicyClient`) **DEVE** implementar *circuit breaker* com limiar
configurável (`kernel.broker.circuit_breaker_threshold`, default 0,5) — ao abrir,
toda nova syscall privilegiada é imediatamente negada com `AIOS-CAP-0003` sem
tentar nova chamada de rede, evitando amplificação de carga sobre um PDP já
degradado.

### 3.4 Isolamento multi-tenant como controle de autorização

Antes mesmo de consultar o PDP, o `SyscallGateway` **DEVE** validar que o `tenant`
presente na URN do recurso alvo (`urn:aios:<tenant>:agent:<id>`) é idêntico ao
`tenant` do contexto autenticado (`X-AIOS-Tenant`). Divergência **DEVE** ser
rejeitada com `AIOS-CAP-0002` (403) **antes** de qualquer chamada ao PDP — é uma
verificação estrutural, não uma decisão de política, e portanto barata e
determinística. Este é o controle primário contra vazamento cross-tenant
(mapeado à ameaça *Information Disclosure* no STRIDE, §5).

---

## 4. Superfície de Ataque

### 4.1 Inventário de pontos de entrada

| Superfície | Protocolo | Exposição | Controle primário |
|------------|-----------|-----------|---------------------|
| `SyscallGateway` REST (`/v1/kernel/*`) | HTTPS via Gateway/YARP | Externa (agentes/SDKs autenticados) | AuthN no Gateway + PEP + rate-limit por agente |
| `SyscallGateway` gRPC (`aios.kernel.v1`) | gRPC + mTLS | Interna (apenas malha de serviços) | mTLS workload identity + PEP |
| Eventos consumidos (`aios.<tenant>.scheduler.*`, `lifecycle.*`, `policy.*`, `cost.*`) | NATS/JetStream (TLS) | Interna | Conta NATS por serviço; validação de schema/`dataschema`; deduplicação por `event.id` |
| Conexão PostgreSQL (`kernel.*`) | TLS + client cert | Interna | RLS por `tenant_id`; credenciais de menor privilégio por serviço |
| Conexão Redis (contadores/ACB quente) | TLS + AUTH | Interna | Namespace de chave por tenant; ACL Redis restrita a comandos necessários |
| Endpoints de observabilidade (`/metrics`, `/healthz`, `/readyz`) | HTTP interno | Interna (scrape Prometheus/probe) | Sem dados sensíveis; acesso restrito à rede de observabilidade (ver `Deployment.md`) |

### 4.2 Vetores de ataque considerados e descartados

| Vetor | Considerado? | Justificativa |
|-------|---------------|----------------|
| Injeção SQL no `AcbStore` | Sim, mitigado | Acesso exclusivo via ORM/queries parametrizadas (.NET 10); nenhuma concatenação de string com entrada do agente. |
| Escalonamento de privilégio via `capability_set` do ACB | Sim, mitigado | `capability_set` é somente *cache de leitura*; toda mutação de capability exige nova consulta ao PDP — o campo nunca é a fonte de decisão de `allow`. |
| Replay de syscall mutante | Sim, mitigado | `Idempotency-Key` obrigatório (RFC-0001 §5.5) + `IdempotencyStore`; reprodução retorna o mesmo resultado, sem duplo efeito. |
| Forjamento de `tenant` em payload | Sim, mitigado | Validação estrutural §3.4 antes de qualquer processamento. |
| Exaustão de recursos via `spawn` em massa (DoS) | Sim, mitigado | Cota `agent_count`/tenant + `syscalls_per_sec` + admissão do Scheduler (backpressure, ver `Scalability.md`). |
| Execução de código arbitrário no processo do Kernel | Não aplicável | O Kernel **não** executa código do agente/ferramenta — ele é broker; execução ocorre em `007-Agent-Runtime` (sandbox) e `015-Tool-Manager`. |
| Exfiltração de dados de memória via syscall `recall` sem capability | Sim, mitigado | `ResourceBrokerRouter` aplica capability+cota **antes** de encaminhar a `010-Memory`; o Kernel nunca expõe payload de memória de outro agente/tenant. |

---

## 5. Threat Model STRIDE (detalhado)

> A versão resumida está no `_DESIGN_BRIEF.md` §12.2; esta seção a detalha por
> componente interno e aponta o controle técnico exato.

| Ameaça | Componente afetado | Cenário concreto | Controle técnico | Detecção |
|--------|----------------------|-------------------|---------------------|----------|
| **Spoofing** | `SyscallGateway` | Serviço comprometido injeta cabeçalho `X-AIOS-Tenant` diretamente, contornando o Gateway. | mTLS workload identity allowlist (§2.3); rejeição sem identidade válida. | Alerta em `aios_kernel_authn_rejected_total` anômalo. |
| **Spoofing** | Eventos consumidos | Publicador não autorizado emite `aios.<tenant>.scheduler.admission.decided` forjado. | Conta NATS por serviço com permissão de publish restrita ao subject próprio; validação de `source` no envelope. | Auditoria de publishers por subject. |
| **Tampering** | `AcbStore` | Escrita concorrente altera `state`/`quota_ref` sem trilha. | OCC (`version`), RLS, escrita exclusiva via API do Kernel (sem acesso direto de outros serviços ao schema `kernel.*`). | Conflito de `version` gera `AIOS-KERNEL-0009`; auditado. |
| **Tampering** | `Quota` (contadores Redis) | Manipulação direta de contador para burlar cota. | Scripts Lua atômicos, sem exposição de comando `SET` livre; ACL Redis restrita por *command allowlist*. | Reconciliação periódica PG vs Redis detecta divergência. |
| **Repudiation** | Toda syscall privilegiada | Chamador nega ter executado `kill`/`spawn`. | `kernel.syscall_log` + auditoria imutável (`025`) com `trace_id`, `agent_urn`, `subject`, decisão do PEP. | Consulta de auditoria por `trace_id`/`agent_urn`. |
| **Information Disclosure** | `SyscallGateway` (erros) | Mensagem de erro vaza detalhe interno (stack trace, string de conexão). | Envelope RFC 7807 padronizado (RFC-0001 §5.4); `detail` sanitizado, sem PII/segredos. | Revisão de amostras de log de erro em CI (regra de lint de log). |
| **Information Disclosure** | Cross-tenant | Agente do tenant A consulta ACB do tenant B via `GetAcb`. | Validação estrutural de tenant (§3.4) + RLS de banco como segunda camada (*defense in depth*). | `AIOS-CAP-0002` monitorado; alerta em taxa anômala. |
| **Denial of Service** | `SyscallGateway` | Flood de `spawn` de um único tenant satura o Scheduler. | `syscalls_per_sec` por agente; cota `agent_count`/tenant; rejeição precoce `429 AIOS-QUOTA-0002`. | Métrica `aios_kernel_quota_rejected_total{reason="rate_limit"}`. |
| **Denial of Service** | `PolicyClient` | PDP lento degrada latência de toda syscall. | Circuit breaker + `fail_mode=closed` isola o impacto a negação rápida, não a bloqueio. | `aios_kernel_pep_decision_ms` p99 acima do SLO (NFR-003) dispara alerta. |
| **Elevation of Privilege** | `CapabilityEnforcer` | Bypass do PEP por chamada direta a um handler interno. | Nenhum handler de syscall mutante é alcançável sem passar pelo pipeline `SyscallGateway → CapabilityEnforcer`; garantido por design de camadas (sem rota alternativa registrada). | Cobertura de teste de contrato (`Testing.md`) valida ausência de rota sem PEP; auditado por pentest (NFR-010). |
| **Elevation of Privilege** | `ResourceBrokerRouter` | Encaminhar syscall de recurso sem reverificar capability específica do recurso (ex.: `invoke_tool` para ferramenta restrita). | Capability é avaliada por **ação + recurso específico**, não apenas por verbo genérico; `DecisionRequest.resource` inclui o identificador da ferramenta/modelo alvo. | Teste de contrato por ferramenta restrita. |

---

## 6. Gestão de Segredos

| Segredo | Tipo | Armazenamento | Rotação | Acesso |
|---------|------|-----------------|---------|--------|
| Certificados mTLS (SPIFFE/SVID) do Kernel | Certificado X.509 de curta duração | Emitido em runtime pela infraestrutura de PKI (sidecar/SDS), nunca em disco persistente | ≤ 24h automática | Somente o processo do Kernel via socket local do agente de identidade |
| Credencial PostgreSQL | Usuário/senha + client cert | Cofre de segredos (ex.: Vault/KMS gerenciado por `021`), injetado como variável de ambiente/arquivo montado em `tmpfs` | 90 dias ou sob incidente | Processo do Kernel apenas |
| Credencial Redis (AUTH) | Senha | Cofre de segredos | 90 dias ou sob incidente | Processo do Kernel apenas |
| NATS account JWT/NKey | JWT assinado | Cofre de segredos | 30 dias | Processo do Kernel apenas |
| Chaves de assinatura de cabeçalho interno (Gateway↔Kernel) | Chave simétrica/assimétrica | Cofre de segredos, sincronizada entre Gateway e Kernel | 30 dias | Gateway (assina) e Kernel (verifica) |

**Regras normativas:**

- Nenhum segredo **DEVE** ser escrito em log, trace, métrica ou payload de evento
  (RFC-0001 §7 — minimização).
- Nenhum segredo **DEVE** ser embutido em imagem de container ou em código-fonte;
  todo segredo é injetado em tempo de execução (ver `Deployment.md` §6).
- Segredos **DEVEM** ser lidos apenas na inicialização ou via *hot reload* seguro
  (sem persistir cópia em disco além do necessário para o processo).
- Rotação de segredo **NÃO DEVE** exigir *downtime*: o Kernel **DEVE** suportar
  recarregamento de credencial sem reinício (conexões *pool* renovadas de forma
  gradual).

---

## 7. TLS/mTLS — Fronteiras de Confiança

```
        Zona Não-Confiável           Zona de Borda (DMZ)              Zona Interna (Malha mTLS)
   ┌────────────────────┐      ┌──────────────────────┐      ┌───────────────────────────────────┐
   │  Cliente/Agente     │      │   Gateway (YARP)     │      │        KERNEL (006)               │
   │  externo            │─TLS─▶│  - Valida OIDC/JWT   │─mTLS─▶│  SyscallGateway                   │
   │                     │ 1.3  │  - Assina cabeçalhos │      │  CapabilityEnforcer                │
   └────────────────────┘      │    internos           │      │  AcbStore / QuotaMgr / ...        │
                                └──────────────────────┘      └───────────────┬───────────────────┘
                                                                                │ mTLS (todas as setas)
                                                        ┌──────────────┬────────┼─────────┬────────────┐
                                                        ▼              ▼        ▼         ▼            ▼
                                                  022-Policy    009-Scheduler 008-Lifecycle  PostgreSQL/Redis/NATS
```

- **Fora → Borda**: TLS 1.3 padrão de mercado (SDK/browser/CLI → Gateway).
- **Borda → Interna** e **Interna → Interna**: mTLS obrigatório em toda chamada
  (RFC-0001 §6); nenhuma comunicação de serviço a serviço **DEVE** ocorrer em
  texto plano ou apenas TLS unilateral.
- Cifras **DEVEM** seguir o perfil moderno definido por `021-Security` (TLS 1.3
  preferencial; TLS 1.2 apenas com conjuntos de cifra aprovados como fallback
  transitório).

---

## 8. Isolamento e Sandbox

O Kernel **não hospeda** sandbox de execução de agente (isso é `007-Agent-Runtime`,
NR-01). Seus controles de isolamento são de **bulkhead** e **circuit breaker**
entre dependências, para que a falha ou comprometimento de uma dependência não
se propague:

| Dependência | Padrão de isolamento | Efeito de falha isolado |
|-------------|------------------------|----------------------------|
| PDP (`022-Policy`) | Bulkhead dedicado no `PolicyClient` + circuit breaker | Falha do PDP não bloqueia leitura de ACB nem `get_quota`. |
| Scheduler (`009`) | Bulkhead no `SchedulerClient` | Falha do Scheduler afeta apenas `spawn`/`resume`; `kill`/`suspend` continuam operando. |
| Lifecycle (`008`) | Bulkhead no `LifecycleClient` | Falha de materialização não corrompe o `AcbStore` (saga compensa). |
| Brokers de recurso (`010/011/012/015/017`) | Bulkhead + circuit breaker por dependência no `ResourceBrokerRouter` | Falha em `015-Tool-Manager` não impede `remember`/`recall` em `010-Memory`. |

Cada bulkhead **DEVE** ter *pool* de conexões/threads dedicado e limite próprio,
de forma que a saturação de uma dependência não esgote recursos compartilhados
usados por chamadas a outras dependências (isolamento de "vizinho barulhento" em
nível de dependência, complementando o isolamento por tenant em `Scalability.md`).

---

## 9. Hardening de Infraestrutura

| Controle | Especificação |
|----------|-----------------|
| Usuário do processo | Container do Kernel **DEVE** executar como usuário não-root (UID dedicado, sem `CAP_SYS_ADMIN` ou capacidades Linux desnecessárias). |
| Sistema de arquivos | Sistema de arquivos raiz **DEVERIA** ser montado somente-leitura; diretórios graváveis restritos a `/tmp` efêmero e diretório de segredos em `tmpfs`. |
| Rede | Políticas de rede (NetworkPolicy/equivalente) **DEVEM** restringir egress do Kernel apenas às dependências listadas em §4.1; nenhum egress arbitrário à internet. |
| Imagem | Imagem base mínima (distroless ou equivalente), sem *shell* interativo em produção; scan de vulnerabilidade (SCA) obrigatório no pipeline de CI. |
| Segredos em imagem | **NÃO DEVE** haver segredo, chave privada ou credencial embutida em camada de imagem (verificado por *secret scanning* em CI). |
| Dependências | *Software Composition Analysis* (SCA) contínuo sobre pacotes .NET 10; CVEs críticas **DEVEM** ser corrigidas dentro do SLA de patch de `021-Security`. |

---

## 10. LGPD / GDPR

- **Minimização de dados**: eventos e logs do Kernel **NÃO DEVEM** transportar
  conteúdo de memória, prompts ou dados pessoais do usuário final — apenas
  identificadores opacos (URNs) e metadados de controle (RFC-0001 §7). Payload
  de `remember`/`recall` **NÃO DEVE** ser logado pelo Kernel; ele é opaco ao
  broker (responsabilidade de base legal e retenção é de `010-Memory`).
- **Direito ao esquecimento**: a syscall `kill` **DEVE** poder ser seguida de uma
  operação de expurgo rastreável do ACB, coordenada com `025-Audit`, que retém
  apenas os metadados de auditoria exigidos por lei (identificador, timestamps,
  decisões de política) e remove qualquer dado pessoal residual do próprio ACB
  (ex.: `labels` fornecidos pelo usuário).
- **Base legal e retenção**: `kernel.syscall_log` e `kernel.outbox` seguem
  política de retenção configurável (`kernel.idempotency.retention_h`, retenção
  de stream JetStream); nenhuma dessas tabelas armazena conteúdo de memória —
  isso pertence a `010-Memory`, que carrega sua própria base legal e retenção.
- **Segregação por tenant**: RLS por `tenant_id` em todas as tabelas `kernel.*` +
  contas NATS isoladas por tenant garantem que um pedido de exclusão/portabilidade
  de dados de um tenant não afete outro.
- **Portabilidade de dados**: o ACB, sendo metadado de controle (não conteúdo),
  é exportável via `GetAcb`/auditoria sob demanda do titular, sem necessidade de
  acesso direto ao schema do banco.

---

## 11. Auditoria e Não-Repúdio

Toda ação privilegiada e toda decisão de política (allow/deny) **DEVEM** gerar:

1. Um registro em `kernel.syscall_log` (§3.3 de `Database.md`, quando publicado)
   com `decision`, `deny_code` (se negado), `trace_id`, `agent_urn`, `verb`.
2. Um evento de auditoria imutável enviado a `025-Audit` (R-09 do brief),
   contendo os mesmos campos de correlação (RFC-0001 §5.6).
3. Em caso de negação, o evento `agent.syscall.denied` (catálogo em `Events.md`)
   publicado no NATS para consumo por sistemas de alerta de segurança.

Essa trilha **DEVE** ser suficiente para responder, para qualquer syscall
executada, às perguntas: *quem* solicitou, *o quê*, *sobre qual recurso*, *quando*,
*qual foi a decisão do PDP* e *qual o resultado de execução* — sem exigir
correlação manual entre sistemas.

---

## 12. Matriz de Controles (resumo executivo)

| ID | Controle | Categoria | Verificação |
|----|----------|-----------|--------------|
| SEC-K-01 | Default deny em toda syscall privilegiada | AuthZ | Teste de contrato: syscall sem capability → `AIOS-CAP-0001`. |
| SEC-K-02 | Fail-closed do PEP sob indisponibilidade do PDP | AuthZ/Resiliência | Chaos test: derrubar PDP → syscalls negadas, não permitidas. |
| SEC-K-03 | Validação estrutural de `tenant` antes do PDP | Isolamento | Teste: `tenant` divergente → `AIOS-CAP-0002` sem chamada ao PDP. |
| SEC-K-04 | mTLS obrigatório em toda comunicação interna | Transporte | Scan de configuração + teste de conexão sem cert válido é rejeitada. |
| SEC-K-05 | RLS por `tenant_id` em todas as tabelas `kernel.*` | Isolamento | Teste automatizado de RLS (query cross-tenant retorna vazio). |
| SEC-K-06 | Nenhum segredo em imagem/log/evento | Segredos | Secret scanning em CI + regra de lint de log. |
| SEC-K-07 | `Idempotency-Key` obrigatório em mutações | Integridade | Teste de replay: mesma chave → mesmo resultado. |
| SEC-K-08 | Auditoria imutável de toda ação privilegiada | Repúdio | Cobertura auditada (NFR-010): 100% das ações privilegiadas. |
| SEC-K-09 | Bulkhead/circuit breaker por dependência externa | Isolamento de falha | Chaos test por dependência isolada (§8). |
| SEC-K-10 | Purge rastreável do ACB sob pedido de esquecimento | LGPD/GDPR | Teste de expurgo: dado pessoal removido, metadado de auditoria retido. |

---

## 13. Referências

- Contratos centrais de segurança: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7.
- Modelo de política (PDP): `../022-Policy/` (a detalhar).
- Segurança transversal (PKI, OIDC, sandbox de runtime): `../021-Security/`.
- Auditoria imutável: `../025-Audit/`.
- Glossário: `../040-Glossary/Glossary.md` (termos: Capability, Default Deny, PEP/PDP, mTLS, STRIDE, RLS, LGPD/GDPR).
- Brief interno do módulo: `./_DESIGN_BRIEF.md` §1.3 (NR-04, NR-07), §12.
- Deployment (hardening de infraestrutura): `./Deployment.md`.
- Escalabilidade (isolamento de noisy neighbor): `./Scalability.md`.
- Recuperação de falhas (fail-closed sob indisponibilidade de PDP/dependências): `./FailureRecovery.md`.
