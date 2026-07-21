---
Documento: Security
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (Máquina de estados canônica), ADR-0082 (Codec e cifragem de checkpoint), ADR-0084 (Lease + fencing token), ADR-0086 (Event sourcing + Outbox), ADR-0088 (Fronteira 008×006×009×007), ADR-0089 (Retenção/tombstone/LGPD)
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol — a propor)
Depende de: 006-Kernel, 021-Security, 022-Policy, 025-Audit, 020-Communication, 027-Cluster
---

# 008-Agent-Lifecycle — Security

> **Escopo.** Este documento especifica a postura de segurança do módulo
> Agent Lifecycle: como ele autentica chamadores, como aplica autorização como
> **PEP (Policy Enforcement Point)** para toda mutação de ciclo de vida, sua
> superfície de ataque, o *threat model* STRIDE aplicado aos seus componentes
> internos (§2 do `_DESIGN_BRIEF.md`), a gestão de segredos que protegem
> checkpoints (dados potencialmente sensíveis de working memory), TLS/mTLS, e a
> conformidade LGPD/GDPR sobre snapshots e tombstones. Este documento **NÃO
> redefine** os contratos centrais de segurança (mTLS interno, validação OIDC
> no Gateway, envelope de erro RFC 7807) — eles são normativos em
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7 e detalhados em
> `../021-Security/`. Aqui se especifica **como o módulo 008 os consome e os
> aplica** em sua fronteira de confiança específica: o Agent Control Block
> (ACB), o histórico de transições e os checkpoints/snapshots.

---

## 1. Postura de Segurança e Princípios

O módulo 008 é a **única autoridade** sobre a persistência de transições de
estado do ACB (R-01/R-02 do brief) e sobre a serialização/cifragem do estado
de agentes hibernados (R-05/R-06). Toda a sua postura de segurança deriva de
quatro princípios, rastreáveis ao brief:

| Princípio | Descrição | Consequência de design |
|-----------|-----------|-------------------------|
| **Default Deny** | Nenhuma mutação de ciclo de vida (`spawn`, `suspend`, `resume`, `hibernate`, `wake`, `migrate`, `checkpoint`, `restore`, `terminate`) é aplicada sem decisão explícita de `allow` do PDP (`022-Policy`). | `LifecyclePolicyEnforcer` bloqueia por padrão; ausência de decisão = negação (`AIOS-LIFECYCLE-0012`). |
| **PEP, não PDP** | O módulo 008 **NÃO DEVE** decidir política localmente (N5 do brief). | Nenhuma lógica de RBAC/ABAC é hardcoded; `policy_ref` do ACB é referência, não autoridade de decisão. |
| **Confidencialidade em repouso do estado hibernado** | Todo `Checkpoint` contém potencialmente dados pessoais da working memory do agente. | Cifragem obrigatória `AES-256-GCM` com chave por tenant (§3.3 do brief); hash de integridade `sha-256`. |
| **Serialização de transição = controle de segurança** | Mais de um coordenador atuando sobre o mesmo agente simultaneamente é uma condição de corrida com potencial de corrupção/rogue write. | `LeaseManager` com `fencing_token` monotônico (R-10 do brief) invalida escritas de coordenadores obsoletos — tratado também como controle STRIDE (Tampering/Elevation of Privilege, §5). |

Este documento assume que o leitor já conhece os contratos centrais da RFC-0001
(URN, envelope de evento, envelope de erro RFC 7807, `Idempotency-Key`,
`traceparent`/`X-AIOS-Tenant`) e **não os repete**, exceto quando referenciados
para explicar um controle específico do módulo 008.

---

## 2. Autenticação (AuthN)

O módulo 008 **NÃO DEVE** emitir, validar ou renovar tokens de identidade —
essa responsabilidade pertence ao Gateway (YARP) em conjunto com
`021-Security`. O módulo consome apenas **claims já validadas**.

### 2.1 Fluxo de autenticação do chamador externo

| Etapa | Ator | Ação |
|-------|------|------|
| 1 | Cliente (SDK/console/agente pai) | Envia requisição REST (`/v1/agents/**`) com `Authorization: Bearer <JWT>` ao Gateway. |
| 2 | Gateway (YARP) | Valida assinatura/expiração do JWT contra o IdP (OIDC) via `021-Security`; rejeita com `401` se inválido. |
| 3 | Gateway (YARP) | Extrai claims (`sub`, `tenant`, `roles`, `scopes`) e as propaga como cabeçalhos internos assinados (`X-AIOS-Tenant`, `X-AIOS-Subject`, `X-AIOS-Roles`) via mTLS. |
| 4 | `LifecycleApiSurface` | Recebe a requisição já autenticada; **NÃO DEVE** revalidar o JWT; **DEVE** validar que os cabeçalhos internos chegaram por um canal mTLS de identidade de workload confiável (§2.3). |
| 5 | `LifecycleApiSurface` | Extrai `tenant`, `agent_id` (da URN do recurso alvo) e `traceparent` (RFC-0001 §5.6) para o contexto do comando de ciclo de vida. |

### 2.2 Autenticação interna serviço-a-serviço (mTLS)

Toda comunicação do módulo 008 com dependências internas (`022-Policy`,
`009-Scheduler`, `007-Agent-Runtime`, `006-Kernel`, `010-Memory`, PostgreSQL,
Redis, MinIO, NATS) **DEVE** usar **mTLS** com certificados de curta duração
emitidos e rotacionados pela infraestrutura de PKI de `021-Security`
(RFC-0001 §6).

| Canal | Protocolo | Certificado | Rotação |
|-------|-----------|-------------|---------|
| 008 → PolicyClient (022) | gRPC + mTLS | SPIFFE/SVID por workload | ≤ 24h |
| 008 → Scheduler (009) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| 008 → Runtime Supervisor (007) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| 008 → Kernel (006) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| 008 → Memory (010, via `working_memory_ref`) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| 008 → Cluster (027, decisões de placement em migração) | gRPC + mTLS | SPIFFE/SVID | ≤ 24h |
| 008 → PostgreSQL (`AcbStore`, outbox, índice de snapshot) | TLS 1.3 + client cert | Certificado de serviço | 90 dias |
| 008 → Redis (`LeaseManager`, ACB quente) | TLS 1.3 + AUTH | Certificado de serviço + senha rotacionada | 90 dias |
| 008 → MinIO (`SnapshotStore`, blobs de checkpoint) | TLS 1.3 + credencial S3 assinada | Chave de acesso por tenant/bucket | 30 dias |
| 008 → NATS/JetStream | TLS 1.3 + NKey/JWT de conta | NATS account JWT | 30 dias |

### 2.3 Validação de cabeçalhos internos assinados

Como o `LifecycleApiSurface` confia nas claims propagadas pelo Gateway (e não
revalida o JWT original), ele **DEVE** validar que os cabeçalhos internos
(`X-AIOS-Tenant`, `X-AIOS-Subject`, `X-AIOS-Roles`) chegaram por um canal mTLS
autenticado cujo certificado de cliente pertence ao pool de identidades
confiáveis do Gateway (*workload identity allowlist*). Requisições que
cheguem sem essa identidade de workload autenticada **DEVEM** ser rejeitadas
com `401` antes de qualquer processamento de comando de ciclo de vida — isso
impede que um serviço comprometido da malha injete claims forjadas
diretamente, contornando o Gateway.

---

## 3. Autorização (AuthZ) — o módulo 008 como PEP

### 3.1 Modelo RBAC + ABAC (decidido externamente)

O `LifecyclePolicyEnforcer` implementa exclusivamente o **ponto de
aplicação**; o modelo de papéis (RBAC) e atributos (ABAC) é definido e
avaliado pelo PDP em `022-Policy`. Para toda mutação (todas as ações listadas
em §12.1 do brief), ele **DEVE** montar um `DecisionRequest` com no mínimo:

| Campo do `DecisionRequest` | Origem | Descrição |
|------------------------------|--------|-----------|
| `subject` | Claims propagadas (`X-AIOS-Subject`, `X-AIOS-Roles`) | Quem solicita (usuário, agente pai, service account, ou o próprio `ReconciliationController` atuando como *system principal*). |
| `action` | Permissão mapeada (`lifecycle:spawn`, `:suspend`, `:resume`, `:hibernate`, `:wake`, `:migrate`, `:checkpoint`, `:restore`, `:terminate`, `:read`) | Ação de ciclo de vida solicitada. |
| `resource` | URN do agente alvo (`urn:aios:<tenant>:agent:<id>`) | Sobre o que a ação incide. |
| `environment` | `tenant`, `priority_class`, `quota_ref`, `state` atual, hora, IP de origem | Contexto para regras ABAC (ex.: negar `terminate` em `priority_class=system` sem papel elevado). |

> **Nota:** `GET /v1/agents/{id}` e `GET /v1/agents/{id}/lifecycle` (leituras)
> **PODEM** ter política de decisão mais permissiva (ex.: escopo "leitura do
> próprio agente" ou "leitura por tenant"), mas ainda **DEVEM** passar pelo
> `LifecyclePolicyEnforcer` — nenhuma rota do `LifecycleApiSurface` ignora o
> PEP (FR-013).

### 3.2 Ciclo de decisão e cache

```
   LifecycleApiSurface   LifecyclePolicyEnforcer         PolicyClient          022-Policy (PDP)
        │                        │                            │                        │
        │  comando(ação, urn)    │                             │                        │
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
        │◀── allow → coordinator │                             │                        │
        │◀── deny → 403 LC-0012 ─│                             │                        │
```

- Decisões de `allow` **PODEM** ser cacheadas pelo `LifecyclePolicyEnforcer**
  por um TTL curto, nunca superior ao TTL retornado pelo PDP.
- Decisões de `deny` **NÃO DEVEM** ser cacheadas com TTL maior que o de
  `allow`, para permitir revogação rápida de acesso.
- Uma atualização de *policy bundle* consumida do domínio de política **DEVE**
  invalidar todo o cache local do `LifecyclePolicyEnforcer` para o tenant
  afetado, independentemente do TTL restante.

### 3.3 Fail-Closed sob indisponibilidade do PDP

Consistente com R-05/R-06 do brief e com o princípio Default Deny, sob
timeout/erro do PDP o módulo 008 **DEVE** negar a mutação de ciclo de vida
com `AIOS-LIFECYCLE-0012` (403) ou, quando a causa é explicitamente
infraestrutural (timeout de rede ao PDP), com o código de disponibilidade
mais apropriado do catálogo transversal de `022-Policy`, sempre
`retriable=true`. O `LifecyclePolicyEnforcer` **DEVE** implementar *circuit
breaker* — ao abrir, novas mutações são imediatamente negadas sem nova
tentativa de rede, evitando amplificação de carga sobre um PDP já degradado.
Leituras não sensíveis **PODEM** continuar operando (ver `FailureRecovery.md`
§4).

### 3.4 Isolamento multi-tenant como controle de autorização

Antes mesmo de consultar o PDP, o `LifecycleApiSurface` **DEVE** validar que o
`tenant` presente na URN do recurso alvo (`urn:aios:<tenant>:agent:<id>`) é
idêntico ao `tenant` do contexto autenticado (`X-AIOS-Tenant`). Divergência
**DEVE** ser rejeitada com `AIOS-LIFECYCLE-0001` (404 — o agente
"não existe" do ponto de vista do tenant chamador, evitando confirmar
existência cross-tenant) **antes** de qualquer chamada ao PDP. Este é o
controle primário contra vazamento cross-tenant, mapeado à ameaça
*Information Disclosure* no STRIDE (§5).

---

## 4. Superfície de Ataque

### 4.1 Inventário de pontos de entrada

| Superfície | Protocolo | Exposição | Controle primário |
|------------|-----------|-----------|---------------------|
| `LifecycleApiSurface` REST (`/v1/agents/**`) | HTTPS via Gateway/YARP | Externa (agentes/SDKs autenticados) | AuthN no Gateway + PEP + `Idempotency-Key` |
| `LifecycleApiSurface` gRPC (`aios.lifecycle.v1.LifecycleService`) | gRPC + mTLS | Interna (malha de serviços: 006, 007, 009, 027) | mTLS workload identity + PEP |
| Eventos consumidos (`agent.control.created`, `scheduler.placement.decided`, `scheduler.preemption.requested`, `agent.runtime.heartbeat`, `agent.runtime.exited`, `task.execution.completed`, `cluster.node.draining`) | NATS/JetStream (TLS) | Interna | Conta NATS por serviço; validação de schema/`dataschema`; deduplicação por `event.id` |
| Conexão PostgreSQL (`AcbStore`, `LifecycleTransition`, `LifecycleOutbox`, índice de `Checkpoint`) | TLS + client cert | Interna | RLS por `tenant_id`; credenciais de menor privilégio |
| Conexão Redis (`LeaseManager`, ACB quente) | TLS + AUTH | Interna | Namespace de chave por tenant; ACL Redis restrita a comandos necessários |
| Conexão MinIO (`SnapshotStore`, blobs de checkpoint) | TLS + credencial S3 | Interna | Bucket por tenant; política de acesso restrita ao serviço; objetos cifrados (§7) |
| Endpoints de observabilidade (`/metrics`, `/healthz`, `/readyz`) | HTTP interno | Interna (scrape Prometheus/probe) | Sem dados sensíveis; acesso restrito à rede de observabilidade (ver `Deployment.md`) |

### 4.2 Vetores de ataque considerados e descartados

| Vetor | Considerado? | Justificativa |
|-------|---------------|----------------|
| Injeção SQL no `AcbStore`/`SnapshotStore` | Sim, mitigado | Acesso exclusivo via queries parametrizadas (.NET 10); nenhuma concatenação de string com entrada do agente/rótulo. |
| Escalonamento via transição de FSM forjada (ex.: pular `Ready` e ir direto a `Running` sem `spawn`) | Sim, mitigado | `StateMachineEngine` valida toda transição contra a tabela de guardas (§4.3 do brief); transições fora do grafo são rejeitadas com `AIOS-LIFECYCLE-0002`, independentemente da intenção do chamador. |
| Replay de comando mutante (`hibernate`, `terminate`) | Sim, mitigado | `Idempotency-Key` obrigatório (RFC-0001 §5.5); reprodução retorna o mesmo resultado, sem duplo efeito. |
| Forjamento de `tenant` em payload/URN | Sim, mitigado | Validação estrutural §3.4 antes de qualquer processamento. |
| Exfiltração de working memory via checkpoint mal cifrado | Sim, mitigado | `CheckpointService` **DEVE** cifrar todo blob com `AES-256-GCM` e chave por tenant antes de persistir em MinIO (§7); nenhum checkpoint em texto claro sai do processo do `CheckpointService`. |
| Corrupção de checkpoint para forçar restore malicioso (`Tampering`) | Sim, mitigado | `content_hash` (sha-256) verificado antes de qualquer `restore`; divergência rejeita com `AIOS-LIFECYCLE-0007`. |
| Coordenador "fantasma" (rogue) continuando a agir após perder a liderança de um agente | Sim, mitigado | `fencing_token` monotônico do `LeaseManager`; escrita com token obsoleto rejeitada com `AIOS-LIFECYCLE-0013`. |
| Exaustão de RAM via storm de `wake`/`spawn` (DoS) | Sim, mitigado | Backpressure via `migration.max_concurrent_per_node`, admissão do Scheduler (009) e prioridade de `HibernationController` sob pressão de RAM (`hibernation.ram_pressure_threshold_pct`). |
| Execução de código arbitrário no processo do módulo 008 | Não aplicável | O módulo 008 **não** executa o loop cognitivo do agente (N1 do brief) — ele apenas inicia/pausa/encerra o processo de runtime, que roda isolado em `007-Agent-Runtime` (sandbox). |
| Ressurreição de agente terminado (reuso de `agent_id`) | Sim, mitigado | `TombstoneManager` garante *anti-resurrection*: IDs terminados **NÃO DEVEM** ser reutilizados (invariante INV3 + RFC-0001 §5.1). |

---

## 5. Threat Model STRIDE (detalhado)

> A versão resumida está no `_DESIGN_BRIEF.md` §12.2; esta seção a detalha por
> componente interno e aponta o controle técnico exato.

| Ameaça | Componente afetado | Cenário concreto | Controle técnico | Detecção |
|--------|----------------------|-------------------|---------------------|----------|
| **Spoofing** | `LifecycleApiSurface` | Serviço comprometido injeta `X-AIOS-Tenant` diretamente, contornando o Gateway. | mTLS workload identity allowlist (§2.3); rejeição sem identidade válida. | Alerta em métrica de rejeição de AuthN anômala. |
| **Spoofing** | `LifecycleEventConsumer` | Publicador não autorizado emite `aios.<tenant>.scheduler.placement.decided` forjado para forçar admissão indevida. | Conta NATS por serviço com permissão de publish restrita ao subject próprio; validação de `source` no envelope CloudEvents. | Auditoria de publishers por subject. |
| **Tampering** | `AcbStore` | Escrita concorrente altera `state`/`placement_shard` sem trilha. | `fencing_token` (OCC de lease), RLS, `LifecycleTransition` event-sourced imutável. | Conflito de `fencing_token` gera `AIOS-LIFECYCLE-0013`; auditado. |
| **Tampering** | `SnapshotStore` (blobs de checkpoint em MinIO) | Alteração direta de um objeto de checkpoint fora do fluxo do `CheckpointService`. | `content_hash` (sha-256) validado em todo `restore`/`wake`; política de bucket restringe escrita ao serviço 008. | `AIOS-LIFECYCLE-0007` monitorado; alerta de integridade. |
| **Repudiation** | Toda mutação de ciclo de vida | Chamador nega ter solicitado `hibernate`/`terminate` de um agente. | `LifecycleTransition` event-sourced (`actor`, `trigger`, `correlation`) + auditoria imutável (`025-Audit`). | Consulta de auditoria por `trace_id`/`agent_id`. |
| **Information Disclosure** | `LifecycleApiSurface` (erros) | Mensagem de erro vaza detalhe interno (stack trace, URI de storage). | Envelope RFC 7807 padronizado (RFC-0001 §5.4); `detail` sanitizado, sem PII/segredos/`storage_uri` bruto. | Revisão de amostras de log de erro em CI. |
| **Information Disclosure** | Cross-tenant | Agente do tenant A consulta ACB/checkpoints do tenant B via `GetState`. | Validação estrutural de tenant (§3.4) + RLS de banco + escopo de bucket MinIO por tenant (*defense in depth*). | `AIOS-LIFECYCLE-0001` monitorado; alerta de taxa anômala. |
| **Information Disclosure** | `LifecycleEventPublisher` | Payload de evento `agent.lifecycle.*` vaza conteúdo de working memory. | Payload mínimo normativo (§6.1 do brief): apenas metadados de estado, nunca conteúdo de memória (RFC-0001 §7 — minimização). | Revisão de schema de evento em CI (`dataschema` versionado). |
| **Denial of Service** | `SpawnManager`/`HibernationController` | Storm de `spawn`/`wake` esgota RAM do cluster. | Backpressure (`migration.max_concurrent_per_node`), WarmPool limitado (`warmpool.max_per_shard`), admissão do Scheduler (009). | Métrica de rejeição por capacidade/quota acima do baseline. |
| **Denial of Service** | `LifecyclePolicyEnforcer` | PDP lento degrada latência de toda mutação. | Circuit breaker + fail-closed isola o impacto a negação rápida, não a bloqueio de todo o serviço. | Latência de decisão de PEP acima do SLO dispara alerta. |
| **Elevation of Privilege** | `StateMachineEngine` | Bypass da FSM por chamada direta a um efeito colateral (ex.: acionar `CheckpointService` sem transição válida). | Nenhum efeito colateral (spawn/checkpoint/migrate) é alcançável sem passar pelo `LifecycleCoordinator → StateMachineEngine`; garantido por design de camadas, sem rota alternativa registrada. | Cobertura de teste de contrato valida ausência de rota sem FSM. |
| **Elevation of Privilege** | `LeaseManager` | Coordenador antigo (rogue) continua aplicando efeitos após perder a lease (ex.: após um failover). | `fencing_token` monotônico persistido no ACB; toda escrita de efeito colateral externo valida o token corrente antes de aplicar. | Alerta de rejeição por token obsoleto (`AIOS-LIFECYCLE-0013`) acima do baseline. |

---

## 6. Gestão de Segredos

| Segredo | Tipo | Armazenamento | Rotação | Acesso |
|---------|------|-----------------|---------|--------|
| Certificados mTLS (SPIFFE/SVID) do módulo 008 | Certificado X.509 de curta duração | Emitido em runtime pela infraestrutura de PKI (sidecar/SDS), nunca em disco persistente | ≤ 24h automática | Somente o processo do módulo via socket local do agente de identidade |
| Credencial PostgreSQL | Usuário/senha + client cert | Cofre de segredos (`021-Security`), injetado como variável de ambiente/arquivo em `tmpfs` | 90 dias ou sob incidente | Processo do módulo 008 apenas |
| Credencial Redis (AUTH) | Senha | Cofre de segredos | 90 dias ou sob incidente | Processo do módulo 008 apenas |
| Credencial de acesso MinIO (por tenant/bucket) | Chave de acesso S3 | Cofre de segredos, escopada por bucket de tenant | 30 dias | `CheckpointService`/`SnapshotStore` apenas |
| **Chave de cifragem de checkpoint** (`AES-256-GCM`, por tenant) | Chave simétrica gerenciada por KMS | KMS gerenciado por `021-Security`; nunca embutida no blob nem no ACB | Rotação de chave-mestra periódica (envelope encryption); chaves de dados por checkpoint são efêmeras, cifradas pela chave-mestra do tenant | `CheckpointService` apenas, via chamada ao KMS no momento de cifrar/decifrar |
| NATS account JWT/NKey | JWT assinado | Cofre de segredos | 30 dias | Processo do módulo 008 apenas |

**Regras normativas:**

- Nenhum segredo **DEVE** ser escrito em log, trace, métrica ou payload de
  evento (RFC-0001 §7 — minimização).
- Nenhum segredo **DEVE** ser embutido em imagem de container ou em
  código-fonte; todo segredo é injetado em tempo de execução (ver
  `Deployment.md` §6).
- A **chave de cifragem por tenant** **NÃO DEVE** ser cacheada em disco pelo
  `CheckpointService` além do tempo estritamente necessário para cifrar/
  decifrar um checkpoint em memória; ela é obtida do KMS por operação (ou por
  um envelope de chave de dados de vida curta).
- Rotação de segredo **NÃO DEVE** exigir *downtime*: o módulo 008 **DEVE**
  suportar recarregamento de credencial sem reinício (conexões *pool*
  renovadas de forma gradual).
- Rotação da chave-mestra de tenant **NÃO DEVE** invalidar checkpoints já
  cifrados com uma chave-mestra anterior — o `CheckpointService` **DEVE**
  suportar decifragem com chaves históricas (ainda válidas no KMS) enquanto
  cifra novos checkpoints com a chave corrente (padrão *key rotation with
  grace period*).

---

## 7. Cifragem de Checkpoints e Snapshots

O checkpoint (`Checkpoint`, §3.3 do brief) é o artefato de maior sensibilidade
do módulo, pois contém `working_memory_ref` e, transitivamente, o payload
serializado da working memory do agente no momento da captura — que **PODE**
conter dados pessoais ou informação sensível do tenant.

| Controle | Especificação |
|----------|-----------------|
| Algoritmo | `AES-256-GCM` (autenticado, protege confidencialidade e integridade do blob). Campo `encryption` do `Checkpoint` fixado em `aes-256-gcm` (não configurável por tenant — decisão de ADR-0082). |
| Escopo de chave | Uma chave-mestra por `tenant_id`, gerenciada por KMS (`021-Security`); nenhum tenant compartilha material de chave com outro. |
| Integridade | `content_hash` (sha-256) calculado sobre o blob **cifrado** e armazenado no metadado do `Checkpoint`; todo `restore`/`wake` **DEVE** recalcular e comparar antes de aceitar o conteúdo (`AIOS-LIFECYCLE-0007` em caso de divergência). |
| Local de cifragem | Em memória do processo `CheckpointService`, **antes** de qualquer escrita em MinIO — o blob **NUNCA** trafega ou repousa em texto claro fora do processo que o produziu. |
| Compressão vs. cifragem | Compressão (`zstd`) ocorre **antes** da cifragem (comprimir dado já cifrado não produz ganho e pode vazar padrões); ordem normativa: serializar → comprimir → cifrar → hash → persistir. |
| Consistência (`consistent`/`crash`) | Checkpoints marcados `crash` (capturados sob falha, não quiesced) seguem os **mesmos** controles de cifragem/integridade — a diferença é semântica de consistência do conteúdo, não de segurança do transporte/repouso. |
| Buckets MinIO | Um bucket (ou prefixo isolado com política dedicada) por `tenant_id`; política de acesso restringe leitura/escrita ao serviço 008, nunca a outros módulos diretamente. |

---

## 8. TLS/mTLS — Fronteiras de Confiança

```
        Zona Não-Confiável           Zona de Borda (DMZ)              Zona Interna (Malha mTLS)
   ┌────────────────────┐      ┌──────────────────────┐      ┌───────────────────────────────────┐
   │  Cliente/Agente     │      │   Gateway (YARP)     │      │     AGENT LIFECYCLE (008)          │
   │  externo            │─TLS─▶│  - Valida OIDC/JWT   │─mTLS─▶│  LifecycleApiSurface               │
   │                     │ 1.3  │  - Assina cabeçalhos │      │  LifecyclePolicyEnforcer            │
   └────────────────────┘      │    internos           │      │  LifecycleCoordinator + Managers   │
                                └──────────────────────┘      └───────────────┬───────────────────┘
                                                                                │ mTLS (todas as setas)
                                                        ┌──────────────┬────────┼─────────┬────────────┬───────────┐
                                                        ▼              ▼        ▼         ▼            ▼           ▼
                                                  022-Policy    009-Scheduler 007-Runtime 006-Kernel  010-Memory  027-Cluster
                                                                                                                        │
                                                                                          PostgreSQL/Redis/MinIO/NATS ◀┘
```

- **Fora → Borda**: TLS 1.3 padrão de mercado (SDK/browser/CLI → Gateway).
- **Borda → Interna** e **Interna → Interna**: mTLS obrigatório em toda
  chamada (RFC-0001 §6); nenhuma comunicação de serviço a serviço **DEVE**
  ocorrer em texto plano ou apenas TLS unilateral.
- Cifras **DEVEM** seguir o perfil moderno definido por `021-Security` (TLS
  1.3 preferencial; TLS 1.2 apenas com conjuntos de cifra aprovados como
  fallback transitório).
- A conexão do `SnapshotStore` ao MinIO **DEVE** usar TLS 1.3 mesmo quando o
  MinIO reside na mesma rede interna — checkpoints cifrados em repouso não
  dispensam TLS em trânsito (defesa em profundidade).

---

## 9. Isolamento e Sandbox

O módulo 008 **não hospeda** sandbox de execução de agente (isso é
`007-Agent-Runtime`, N1 do brief). Seus controles de isolamento são de
**bulkhead** e **circuit breaker** entre dependências, para que a falha ou
comprometimento de uma não se propague:

| Dependência | Padrão de isolamento | Efeito de falha isolado |
|-------------|------------------------|----------------------------|
| PDP (`022-Policy`) | Bulkhead dedicado + circuit breaker no `LifecyclePolicyEnforcer` | Falha do PDP não bloqueia leitura de ACB (`GetState`), apenas mutações. |
| Scheduler (`009`) | Bulkhead no `SpawnManager`/`MigrationOrchestrator` | Falha do Scheduler afeta apenas `spawn`/`wake`/`migrate`; `suspend`/`terminate` de agentes já materializados continuam. |
| Runtime Supervisor (`007`) | Bulkhead no `SpawnManager`/`WarmPoolManager` | Falha em provisionar runtime não corrompe o `AcbStore` (saga compensa, ACB retorna a estado seguro). |
| Storage de snapshot (MinIO/PostgreSQL) | Bulkhead + circuit breaker no `SnapshotStore` | Falha de storage adia **novas** hibernações/migrações, mantendo agentes quentes (degradação graciosa, ver `FailureRecovery.md` §10). |

Cada bulkhead **DEVE** ter *pool* de conexões/threads dedicado e limite
próprio, de forma que a saturação de uma dependência não esgote recursos
compartilhados usados por chamadas a outras dependências.

---

## 10. Hardening de Infraestrutura

| Controle | Especificação |
|----------|-----------------|
| Usuário do processo | Container do módulo 008 **DEVE** executar como usuário não-root (UID dedicado, sem `CAP_SYS_ADMIN` ou capacidades Linux desnecessárias). |
| Sistema de arquivos | Sistema de arquivos raiz **DEVERIA** ser montado somente-leitura; diretórios graváveis restritos a `/tmp` efêmero e diretório de segredos em `tmpfs`. |
| Rede | Políticas de rede (NetworkPolicy/equivalente) **DEVEM** restringir egress do módulo apenas às dependências listadas em §4.1; nenhum egress arbitrário à internet. |
| Imagem | Imagem base mínima (distroless ou equivalente), sem *shell* interativo em produção; scan de vulnerabilidade (SCA) obrigatório no pipeline de CI. |
| Segredos em imagem | **NÃO DEVE** haver segredo, chave privada ou credencial embutida em camada de imagem (verificado por *secret scanning* em CI). |
| Dependências | *Software Composition Analysis* (SCA) contínuo sobre pacotes .NET 10; CVEs críticas **DEVEM** ser corrigidas dentro do SLA de patch de `021-Security`. |

---

## 11. LGPD / GDPR

- **Minimização de dados**: eventos `agent.lifecycle.*` **NÃO DEVEM**
  transportar conteúdo de working memory — apenas metadados de estado
  (`agentId`, `tenantId`, `generation`, `fromState`, `toState`, `trigger`,
  `checkpointId`) (RFC-0001 §7; §6.1 do brief). O payload de checkpoint
  **NÃO DEVE** ser logado pelo módulo 008; ele é opaco ao Kernel/Scheduler,
  cifrado ponta a ponta pelo `CheckpointService`.
- **Direito ao esquecimento**: o `TombstoneManager` **DEVE** executar expurgo
  rastreável de ACB, `LifecycleTransition`, `Checkpoint` e blobs de snapshot
  ao término da retenção (`retention.terminated_ttl_days`,
  `retention.checkpoint_ttl_days`) ou sob solicitação explícita do titular,
  emitindo evento auditável (`025-Audit`) — conforme R-12 do brief.
- **Não-reutilização de identidade**: `agent_id` de agentes `Terminated`
  **NÃO DEVEM** ser reatribuídos a um novo ACB (RFC-0001 §5.1; invariante
  estrutural do brief) — o expurgo remove o conteúdo, não a reserva do
  identificador, prevenindo *identity collision* e ambiguidade de auditoria
  histórica.
- **Base legal e retenção diferenciada**: metadados de controle de ciclo de
  vida (estado, timestamps, `actor`, decisões de guarda) **PODEM** ser
  retidos por período de auditoria mais longo que o conteúdo de checkpoint
  em si, desde que a base legal de cada retenção seja distinta e documentada
  (metadados de controle vs. conteúdo de working memory) — coordenado com
  `010-Memory` e `025-Audit` para a política de base legal do conteúdo.
- **Segregação por tenant**: RLS por `tenant_id` em todas as tabelas do
  módulo 008 + bucket/prefixo MinIO isolado por tenant garantem que um
  pedido de exclusão/portabilidade de um tenant não afete outro.
- **Portabilidade de dados**: o histórico de transições (`GET
  …/lifecycle`) é exportável sob demanda do titular como metadado de
  controle; o conteúdo de checkpoint, por depender de decifragem, segue o
  fluxo de portabilidade coordenado com `010-Memory`/`021-Security`.

---

## 12. Auditoria e Não-Repúdio

Toda mutação de ciclo de vida e toda decisão de política (allow/deny)
**DEVEM** gerar:

1. Um registro append-only em `LifecycleTransition` (§3.2 do brief) com
   `from_state`, `to_state`, `trigger`, `actor`, `guard_result`, `generation`
   e `correlation` (`traceparent`, `X-AIOS-Tenant`, `idempotency_key`).
2. Um evento de auditoria imutável enviado a `025-Audit` (R-08/N8 do brief:
   o módulo 008 *emite*, a trilha imutável é de `025`), contendo os mesmos
   campos de correlação (RFC-0001 §5.6).
3. Em caso de negação de política, o registro de `guard_result` reflete a
   negação, e o comando **NÃO DEVE** avançar a FSM — nenhum efeito colateral
   (spawn/checkpoint/migração) é iniciado antes da confirmação de `allow`.

Essa trilha **DEVE** ser suficiente para responder, para qualquer mutação de
ciclo de vida, às perguntas: *quem* solicitou, *o quê*, *sobre qual agente*,
*quando*, *qual foi a decisão do PDP* e *qual o resultado da transição* — sem
exigir correlação manual entre sistemas.

---

## 13. Matriz de Controles (resumo executivo)

| ID | Controle | Categoria | Verificação |
|----|----------|-----------|--------------|
| SEC-LC-01 | Default deny em toda mutação de ciclo de vida | AuthZ | Teste de contrato: mutação sem `allow` → `AIOS-LIFECYCLE-0012`. |
| SEC-LC-02 | Fail-closed do PEP sob indisponibilidade do PDP | AuthZ/Resiliência | Chaos test: derrubar PDP → mutações negadas, não permitidas. |
| SEC-LC-03 | Validação estrutural de `tenant` antes do PDP | Isolamento | Teste: `tenant` divergente → `AIOS-LIFECYCLE-0001` sem chamada ao PDP. |
| SEC-LC-04 | mTLS obrigatório em toda comunicação interna | Transporte | Scan de configuração + teste de conexão sem cert válido é rejeitada. |
| SEC-LC-05 | RLS por `tenant_id` em todas as tabelas do módulo | Isolamento | Teste automatizado de RLS (query cross-tenant retorna vazio). |
| SEC-LC-06 | Cifragem `AES-256-GCM` de todo checkpoint em repouso | Confidencialidade | Verificação de metadado `encryption`; teste de leitura direta do blob (deve ser opaco). |
| SEC-LC-07 | Verificação de `content_hash` em todo `restore`/`wake` | Integridade | Teste de corrupção deliberada de blob → `AIOS-LIFECYCLE-0007`. |
| SEC-LC-08 | Fencing token invalida coordenador obsoleto | Isolamento/Elevação | Teste de split-brain: coordenador antigo tenta escrever → `AIOS-LIFECYCLE-0013`. |
| SEC-LC-09 | Nenhum segredo/PII em log, evento ou imagem | Segredos/Minimização | Secret scanning em CI + regra de lint de log/evento. |
| SEC-LC-10 | `Idempotency-Key` obrigatório em toda mutação | Integridade | Teste de replay: mesma chave → mesmo resultado (NFR-013). |
| SEC-LC-11 | Auditoria imutável de toda transição | Repúdio | Cobertura auditada: 100% das transições com `traceparent` (NFR-012). |
| SEC-LC-12 | Anti-resurrection de `agent_id` após tombstone | LGPD/Integridade | Teste: novo `spawn` com `agent_id` terminado é rejeitado/impossível por construção (ULID novo sempre gerado). |
| SEC-LC-13 | Purge rastreável de checkpoints/snapshots sob retenção/pedido | LGPD/GDPR | Teste de expurgo: blob removido do MinIO, metadado de auditoria retido. |

---

## 14. Referências

- Contratos centrais de segurança: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7.
- Modelo de política (PDP): `../022-Policy/`.
- Segurança transversal (PKI, OIDC, KMS): `../021-Security/`.
- Auditoria imutável: `../025-Audit/`.
- Glossário: `../040-Glossary/Glossary.md` (termos: ACB, Cold Agent, Default Deny, PEP/PDP, mTLS, STRIDE, RLS, LGPD/GDPR).
- Brief interno do módulo: `./_DESIGN_BRIEF.md` §1.2 (N5), §12.
- Deployment (hardening de infraestrutura): `./Deployment.md`.
- Escalabilidade (isolamento de noisy neighbor): `./Scalability.md`.
- Recuperação de falhas (fail-closed sob indisponibilidade de PDP/dependências): `./FailureRecovery.md`.
- ADRs: `./ADR.md` (ADR-0080, ADR-0082, ADR-0084, ADR-0086, ADR-0088, ADR-0089).
