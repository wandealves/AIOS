---
Documento: UseCases
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `FunctionalRequirements.md`, `NonFunctionalRequirements.md`, `../009-Scheduler/`, `../008-Agent-Lifecycle/`, `../022-Policy/`
---

# 006-Kernel — Use Cases

> Convenção: cada caso de uso é numerado `UC-NNN`, descreve **ator**,
> **pré-condições**, **fluxo principal**, **fluxos alternativos**,
> **exceções** e **pós-condições**. Os códigos de erro citados (`AIOS-<DOMÍNIO>-<NNNN>`)
> seguem o catálogo do `_DESIGN_BRIEF.md` §5.2 e o envelope RFC 7807 da
> RFC-0001 §5.4. A rastreabilidade `UC → FR/NFR → Teste` está em
> `Requirements.md` §5.

## Índice

| UC | Título | Verbo(s) envolvido(s) |
|----|--------|-------------------------|
| UC-001 | Spawn de agente (fluxo feliz) | `spawn` |
| UC-002 | Spawn com rejeição/timeout de admissão | `spawn` |
| UC-003 | Falha de materialização do runtime (boot timeout) | `spawn` |
| UC-004 | Suspensão voluntária de agente | `suspend` |
| UC-005 | Preempção (suspensão involuntária pelo Scheduler) | `suspend` |
| UC-006 | Retomada de agente suspenso (resume quente) | `resume` |
| UC-007 | Hibernação por ociosidade | (interno, sem syscall externa) |
| UC-008 | Retomada de agente hibernado (cold resume) | `resume` |
| UC-009 | Encerramento de agente (kill) | `kill` |
| UC-010 | Checkpoint durável | `checkpoint` |
| UC-011 | Consulta de cota | `get_quota` |
| UC-012 | Excesso de cota (hard) e alerta de limiar (soft) | qualquer syscall consumidora |
| UC-013 | Negação de capability pelo PEP | qualquer syscall privilegiada |
| UC-014 | Isolamento de tenant (tenant mismatch) | qualquer syscall |
| UC-015 | Brokering de syscall de recurso | `remember`/`recall`/`plan`/`invoke_tool`/`route_model` |
| UC-016 | Repetição idempotente de syscall | qualquer syscall mutável |
| UC-017 | Conflito de concorrência otimista (OCC) | qualquer syscall mutável no ACB |
| UC-018 | Indisponibilidade do PDP (fail-closed) | qualquer syscall privilegiada |
| UC-019 | Atualização de policy bundle (invalidação de cache) | (interno, consumo de evento) |
| UC-020 | Árvore de agentes (spawn de sub-agente) | `spawn` |

---

## UC-001 — Spawn de agente (fluxo feliz)

**Ator primário:** Agent Runtime (`007`), em nome de um agente autenticado (ou operador via API).

**Atores secundários:** `009-Scheduler`, `008-Agent-Lifecycle`, `022-Policy` (PDP).

**Pré-condições:**
- P1. O chamador possui claims OIDC válidas com `tenant` e capability `agent:spawn`.
- P2. A cota `agent_count` do tenant possui saldo disponível.
- P3. O chamador fornece `Idempotency-Key` única e cabeçalhos de correlação (RFC-0001 §5.6).

**Fluxo principal:**
1. O chamador envia `POST /v1/kernel/agents` (ou `Spawn` gRPC) com especificação do agente (política, prioridade, labels).
2. O `SyscallGateway` valida schema/versão, extrai `traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`; verifica que a chave não foi usada antes (`IdempotencyStore`).
3. O `CapabilityEnforcer` monta o `DecisionRequest` e consulta o PDP via `PolicyClient`; decisão = `allow`.
4. O `LifecycleCoordinator` cria o ACB em estado `Pending` (T-01) dentro de uma transação que também grava a entrada no `Outbox`.
5. O `LifecycleCoordinator` solicita admissão ao `009-Scheduler` via `SchedulerClient`.
6. O Scheduler responde com slot aprovado; o ACB transita para `Admitted` (T-02).
7. O `LifecycleCoordinator` solicita materialização ao `008-Agent-Lifecycle` via `LifecycleClient`.
8. O Lifecycle confirma runtime iniciado (`runtime_ref` atribuído); o ACB transita para `Running` (T-04).
9. O `EventEmitter` publica, via relay do Outbox, os eventos `agent.lifecycle.spawned`, `agent.lifecycle.admitted` e `agent.lifecycle.running` em sequência, cada um após o commit correspondente.
10. O Kernel retorna `201 Created` com o ACB serializado (URN, estado, `runtime_ref`).
11. O `KernelTelemetry` registra span, métricas (`aios_kernel_spawn_latency_ms`) e entrada em `syscall_log` com `decision=allow`, `status=ok`.

**Fluxos alternativos:**
- FA1 (spawn de sub-agente): se a especificação inclui `parent_urn`, ver **UC-020**.
- FA2 (prioridade custom): chamador define `priority` (0–9); Scheduler usa esse valor na decisão de admissão (fora do escopo do Kernel).

**Exceções:**
- E1. Capability negada → ver **UC-013**.
- E2. Tenant divergente → ver **UC-014**.
- E3. Admissão rejeitada/timeout → ver **UC-002**.
- E4. Falha de materialização → ver **UC-003**.
- E5. Payload inválido → `AIOS-KERNEL-0010` (HTTP 400), nenhum ACB criado.
- E6. `Idempotency-Key` já usada com payload idêntico → retorna a resposta original (HTTP 201, mesmo corpo), sem novo efeito colateral (ver **UC-016**).

**Pós-condições:**
- Q1. ACB existe em estado `Running`, com `runtime_ref` não-nulo.
- Q2. Cota `agent_count` do tenant reduzida em 1.
- Q3. Três eventos de lifecycle publicados e auditados.
- Q4. `syscall_log` contém uma linha para a invocação de `spawn`.

---

## UC-002 — Spawn com rejeição/timeout de admissão

**Ator primário:** Agent Runtime (`007`).

**Atores secundários:** `009-Scheduler`.

**Pré-condições:** Mesmas de UC-001 até o passo 4 (ACB criado em `Pending`).

**Fluxo principal (falha):**
1–4. Idênticos a UC-001.
5. O `LifecycleCoordinator` solicita admissão ao Scheduler.
6. O Scheduler responde com rejeição explícita (capacidade insuficiente) **ou** não responde dentro de `kernel.spawn.admission_timeout_ms`.
7. O `LifecycleCoordinator` executa a compensação: nenhuma cota de recurso adicional é consumida além da já contabilizada para o ACB `Pending`; a reserva de `agent_count`, se feita antecipadamente, é revertida.
8. O ACB transita de `Pending` para `Failed` (T-03).
9. O `EventEmitter` publica `agent.lifecycle.failed` com motivo `admission_rejected` ou `admission_timeout`.
10. O Kernel retorna `AIOS-KERNEL-0003` (HTTP 503, `retriable=true`) ao chamador original (se ainda aguardando de forma síncrona) ou disponibiliza o estado via `GetAcb` para chamadas assíncronas.

**Fluxos alternativos:**
- FA1. Chamador reenvia `spawn` com a mesma `Idempotency-Key`: como o resultado original (falha) já foi persistido, o Kernel retorna o mesmo erro sem nova tentativa de admissão (idempotência aplica-se também a falhas terminais).
- FA2. Chamador reenvia `spawn` com **nova** `Idempotency-Key`: tratado como novo `spawn` (UC-001), sujeito às mesmas cotas.

**Exceções:**
- E1. Scheduler retorna erro transitório distinto de rejeição/timeout (ex.: 5xx inesperado) → mapeado para `AIOS-KERNEL-0003` também, preservando `retriable=true`.

**Pós-condições:**
- Q1. ACB em estado terminal `Failed`, com `terminated_at` preenchido.
- Q2. Nenhuma cota de agente permanece reservada indevidamente.
- Q3. Evento `agent.lifecycle.failed` publicado e auditado.

---

## UC-003 — Falha de materialização do runtime (boot timeout)

**Ator primário:** Agent Runtime (`007`), indiretamente via `008-Agent-Lifecycle`.

**Atores secundários:** `008-Agent-Lifecycle`.

**Pré-condições:** ACB em estado `Admitted` (slot do Scheduler já reservado).

**Fluxo principal (falha):**
1. O `LifecycleCoordinator` solicita materialização ao `008-Agent-Lifecycle` via `LifecycleClient`.
2. O Lifecycle não confirma o início do runtime dentro de `kernel.spawn.boot_timeout_ms`.
3. O `LifecycleCoordinator` executa a saga de compensação: solicita ao Scheduler a liberação do slot reservado; reverte a reserva de cota de agente.
4. O ACB transita de `Admitted` para `Failed` (T-05).
5. O `EventEmitter` publica `agent.lifecycle.failed` com motivo `boot_timeout`.
6. O Kernel retorna `AIOS-KERNEL-0004` (HTTP 504, `retriable=true`).

**Fluxos alternativos:**
- FA1. O Lifecycle responde com falha explícita (não apenas timeout) antes do deadline: mesmo tratamento, motivo `boot_failed`.

**Exceções:**
- E1. Falha ao liberar o slot do Scheduler durante a compensação: registrada como incidente operacional (`Monitoring.md`); o ACB ainda transita para `Failed` e o *reconciliation job* do Scheduler eventualmente expira o slot órfão.

**Pós-condições:**
- Q1. ACB em `Failed`, `runtime_ref` permanece nulo (invariante I1).
- Q2. Slot do Scheduler liberado (ou marcado para expiração).
- Q3. Evento `agent.lifecycle.failed` publicado e auditado.

---

## UC-004 — Suspensão voluntária de agente

**Ator primário:** Agent Runtime (`007`), em nome do agente ou de seu operador.

**Pré-condições:**
- P1. ACB em estado `Running`.
- P2. Chamador possui capability `agent:suspend`.
- P3. Agente não está em uma seção crítica não-idempotente (guarda da T-06).

**Fluxo principal:**
1. Chamador envia `POST /v1/kernel/agents/{urn}:suspend` com `Idempotency-Key`.
2. `SyscallGateway` valida e despacha; `CapabilityEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` verifica a guarda de suspensibilidade; transita o ACB para `Suspended` (T-06) com OCC.
4. `EventEmitter` publica `agent.lifecycle.suspended`.
5. Kernel retorna `200 OK` com o ACB atualizado.

**Fluxos alternativos:**
- FA1. Agente em seção crítica não-idempotente: Kernel aguarda até um limite curto configurável ou retorna erro de conflito de estado (`AIOS-KERNEL-0002`) se a guarda não puder ser satisfeita a tempo.

**Exceções:**
- E1. ACB não está em `Running` (ex.: já `Suspended` ou `Terminated`) → `AIOS-KERNEL-0002` (transição inválida) — exceto se for uma repetição idempotente exata (ver UC-016).
- E2. Capability negada → UC-013.
- E3. Conflito de OCC → UC-017.

**Pós-condições:**
- Q1. ACB em `Suspended`; `runtime_ref` preservado (invariante I1 ainda válida — `Suspended` mantém `runtime_ref`).
- Q2. Evento `agent.lifecycle.suspended` publicado e auditado.

---

## UC-005 — Preempção (suspensão involuntária pelo Scheduler)

**Ator primário:** `009-Scheduler` (via evento).

**Pré-condições:** ACB em `Running`; Scheduler decidiu preemptar em favor de agente de maior prioridade.

**Fluxo principal:**
1. O Kernel consome `aios.<tenant>.scheduler.preemption.requested` publicado pelo Scheduler.
2. O `LifecycleCoordinator` trata o evento como um gatilho equivalente a `suspend` (T-06), sem exigir nova consulta de capability do agente afetado (a decisão de preempção já é uma decisão de política de escalonamento autorizada do Scheduler; o Kernel apenas executa).
3. O ACB transita para `Suspended`.
4. O `EventEmitter` publica `agent.lifecycle.suspended` com `labels.reason=preemption`.
5. O `syscall_log` registra a suspensão com origem `scheduler` (não uma syscall externa do agente).

**Fluxos alternativos:**
- FA1. Agente já `Suspended`/`Terminated` quando o evento de preempção chega: evento é ignorado de forma idempotente (dedupe por `event.id`), sem erro.

**Exceções:**
- E1. Falha ao processar o evento (ex.: ACB não encontrado): registrada em log de erro; evento é reencaminhado para DLQ conforme política de `020-Communication`.

**Pós-condições:**
- Q1. ACB em `Suspended` com motivo `preemption` registrado em `labels`.
- Q2. Evento de lifecycle publicado e auditado.

---

## UC-006 — Retomada de agente suspenso (resume quente)

**Ator primário:** Agent Runtime (`007`) ou o próprio Scheduler (ao liberar capacidade).

**Pré-condições:**
- P1. ACB em `Suspended`.
- P2. Chamador possui capability `agent:resume`.

**Fluxo principal:**
1. Chamador envia `POST /v1/kernel/agents/{urn}:resume` com `Idempotency-Key`.
2. `CapabilityEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` solicita ao `009-Scheduler` a re-reserva de slot via `SchedulerClient`.
4. Scheduler confirma slot; ACB transita para `Running` (T-07).
5. `EventEmitter` publica `agent.lifecycle.running` (reentrada).
6. Kernel retorna `200 OK`.

**Fluxos alternativos:**
- FA1. Slot não disponível imediatamente: Kernel aplica a mesma política de timeout de admissão (`kernel.spawn.admission_timeout_ms`); se esgotado, retorna `AIOS-KERNEL-0003` mantendo o ACB em `Suspended` (não falha o agente, apenas a tentativa de resume).

**Exceções:**
- E1. ACB não está em `Suspended` (ex.: já `Running`) → tratado como idempotente se repetição exata (UC-016), senão `AIOS-KERNEL-0002`.
- E2. Capability negada → UC-013.

**Pós-condições:**
- Q1. ACB em `Running` com `runtime_ref` restaurado/preservado.
- Q2. Latência de resume medida por `aios_kernel_syscall_latency_ms{verb="resume"}` (NFR-002).

---

## UC-007 — Hibernação por ociosidade

**Ator primário:** Processo interno do Kernel (varredura periódica / *idle sweep*), sem chamador externo.

**Pré-condições:**
- P1. ACB em `Suspended`.
- P2. `now - last_active_at > kernel.hibernation.idle_ttl_ms`.

**Fluxo principal:**
1. O processo de varredura de ociosidade (baseado no índice `btree(tenant_id, last_active_at)`) identifica ACBs elegíveis.
2. O `LifecycleCoordinator` invoca o `CheckpointManager` para garantir um checkpoint durável recente (cria um novo se necessário).
3. O `LifecycleCoordinator` solicita ao `008-Agent-Lifecycle` a liberação de recursos quentes (RAM) do agente.
4. O ACB transita para `Hibernated` (T-08); `runtime_ref` é limpo (invariante I1).
5. `EventEmitter` publica `agent.lifecycle.hibernated`.

**Fluxos alternativos:**
- FA1. Checkpoint já existente e válido (criado há pouco tempo): reaproveitado, sem novo snapshot.

**Exceções:**
- E1. Falha ao criar checkpoint: hibernação é adiada (ACB permanece `Suspended`); incidente registrado para nova tentativa na próxima varredura.

**Pós-condições:**
- Q1. ACB em `Hibernated`, zero RAM ativa (NFR-017), `checkpoint_ref` válido.
- Q2. Evento `agent.lifecycle.hibernated` publicado e auditado.

---

## UC-008 — Retomada de agente hibernado (cold resume)

**Ator primário:** Agent Runtime (`007`), disparado por nova demanda de trabalho para o agente.

**Pré-condições:**
- P1. ACB em `Hibernated` com `checkpoint_ref` válido.
- P2. Chamador possui capability `agent:resume`.

**Fluxo principal:**
1. Chamador envia `resume`.
2. `CapabilityEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` solicita admissão ao `009-Scheduler` (materialização sob demanda equivale a um novo *placement*); ACB transita para `Admitted` (T-09).
4. `LifecycleCoordinator` solicita ao `008-Agent-Lifecycle` a restauração do runtime a partir de `checkpoint_ref` (MinIO).
5. Runtime materializado; ponteiros `memory_ptr`/`context_ptr` restaurados; ACB transita para `Running`.
6. `EventEmitter` publica `agent.lifecycle.admitted` e `agent.lifecycle.running`.
7. Kernel retorna `200 OK`.

**Fluxos alternativos:**
- FA1. Nenhum, a restauração é sempre a partir do último `checkpoint_ref` válido.

**Exceções:**
- E1. Checkpoint corrompido/inacessível no MinIO: `AIOS-KERNEL-0005` (falha de broker/dependência de recurso); ACB permanece `Hibernated`; incidente registrado para investigação.
- E2. Admissão negada/timeout: mesmo tratamento de UC-002, porém a partir de `Hibernated` (ACB não vai a `Failed` automaticamente — permanece `Hibernated` para nova tentativa, já que o agente não estava em execução ativa).

**Pós-condições:**
- Q1. ACB em `Running` com estado cognitivo equivalente ao momento do checkpoint.
- Q2. Latência de materialização sob demanda dentro da meta de NFR-001 (p99 ≤ 250 ms).

---

## UC-009 — Encerramento de agente (kill)

**Ator primário:** Agent Runtime (`007`), operador humano (via `030-CLI`/`032-WebConsole`), ou o próprio Kernel (encerramento administrativo).

**Pré-condições:**
- P1. ACB em `Running`, `Suspended` ou `Hibernated`.
- P2. Chamador é o *owner* do agente ou possui *role* administrativa com capability `agent:kill`.

**Fluxo principal:**
1. Chamador envia `DELETE /v1/kernel/agents/{urn}` com `Idempotency-Key`.
2. `CapabilityEnforcer` consulta o PDP → `allow`.
3. ACB transita para `Terminating` (T-10).
4. `LifecycleCoordinator` orquestra drenagem: solicita ao `008-Agent-Lifecycle` o encerramento gracioso do runtime (se `Running`/`Suspended`) ou marca diretamente para expurgo (se `Hibernated`); executa compensações pendentes (ex.: cancelamento de operações em curso nos brokers).
5. `ResourceQuotaManager` libera as cotas reservadas de agente.
6. ACB transita para `Terminated` (T-11); `terminated_at` preenchido.
7. `EventEmitter` publica `agent.lifecycle.terminated`.
8. Kernel retorna `202 Accepted` (encerramento assíncrono) ou `200 OK` quando a drenagem é síncrona e rápida.

**Fluxos alternativos:**
- FA1. Encerramento administrativo forçado (operador com *role* admin, agente não é o *owner*): mesma sequência, capability distinta (`agent:kill:admin`, avaliada pelo PDP).
- FA2. `kill` de agente com sub-agentes (`parent_urn` apontando para ele): política de propagação (cascata ou órfão) é decidida por configuração/política, não pelo Kernel diretamente — o Kernel apenas expõe o estado da árvore (ver UC-020).

**Exceções:**
- E1. Capability negada → UC-013.
- E2. ACB já em `Terminated`/`Terminating`: tratado como idempotente (UC-016) se houver `Idempotency-Key` correspondente; caso contrário, retorna o estado atual sem erro (kill é *naturally idempotent* em direção ao estado terminal).
- E3. Falha durante drenagem (broker indisponível): registrada; drenagem é retentada com backoff; ACB permanece em `Terminating` até sucesso ou intervenção operacional.

**Pós-condições:**
- Q1. ACB em `Terminated` (terminal), retido para auditoria.
- Q2. Todas as cotas de agente liberadas (invariante I2).
- Q3. Evento `agent.lifecycle.terminated` publicado exatamente uma vez.

---

## UC-010 — Checkpoint durável

**Ator primário:** Agent Runtime (`007`) ou processo interno de hibernação (UC-007).

**Pré-condições:**
- P1. ACB em `Running` ou `Suspended`.
- P2. Chamador possui capability `agent:checkpoint`.

**Fluxo principal:**
1. Chamador envia `POST /v1/kernel/agents/{urn}:checkpoint` com `Idempotency-Key`.
2. `CapabilityEnforcer` consulta o PDP → `allow`.
3. `CheckpointManager` coleta os ponteiros atuais (`memory_ptr`, `context_ptr`) do ACB.
4. `CheckpointManager` solicita ao `008-Agent-Lifecycle` a criação de um snapshot durável em MinIO.
5. Ao concluir, `CheckpointManager` atualiza `checkpoint_ref` no ACB (self-loop na FSM, T-13).
6. `EventEmitter` publica `agent.checkpoint.created`.
7. Kernel retorna `200 OK` com o `checkpoint_ref`.

**Fluxos alternativos:**
- FA1. Checkpoint disparado internamente pela hibernação (UC-007): mesmo fluxo, sem chamador externo; auditado com `source=internal`.

**Exceções:**
- E1. Falha ao persistir snapshot em MinIO: `AIOS-KERNEL-0005`; `checkpoint_ref` anterior permanece válido (não há atualização parcial — atomicidade da atualização do ponteiro).
- E2. Capability negada → UC-013.

**Pós-condições:**
- Q1. `checkpoint_ref` do ACB aponta para um snapshot íntegro e recuperável.
- Q2. Evento `agent.checkpoint.created` publicado e auditado.

---

## UC-011 — Consulta de cota

**Ator primário:** Agent Runtime (`007`) ou o próprio agente (introspecção de saldo antes de operação custosa).

**Pré-condições:**
- P1. ACB existe (qualquer estado não-terminal, ou terminal para fins de auditoria histórica).
- P2. Chamador possui capability de leitura sobre o próprio ACB (ou capability administrativa para ACB de terceiros).

**Fluxo principal:**
1. Chamador envia `GET /v1/kernel/agents/{urn}/quota`.
2. `CapabilityEnforcer` avalia (leitura de baixo risco pode usar decisão em cache — FR-006).
3. `ResourceQuotaManager` consulta os contadores quentes no Redis (token-bucket) e os limites correntes no PostgreSQL.
4. Kernel retorna `200 OK` com saldo de tokens, cpu-ms, memória, custo USD, ferramentas e syscalls/s (consumido/limite/restante).

**Fluxos alternativos:**
- FA1. Consulta em lote (tenant inteiro, via `GET` administrativo agregando múltiplos `subject_urn`) — variação administrativa fora do verbo canônico, sujeita a capability distinta.

**Exceções:**
- E1. ACB não encontrado → `AIOS-KERNEL-0001` (HTTP 404).
- E2. Tenant divergente → UC-014.

**Pós-condições:**
- Q1. Nenhuma mutação de estado (`get_quota` é somente leitura).
- Q2. Resposta reflete o saldo real com desvio ≤ 1 janela de reconciliação (NFR-009).

---

## UC-012 — Excesso de cota (hard) e alerta de limiar (soft)

**Ator primário:** Qualquer syscall consumidora de cota (`spawn`, `remember`, `invoke_tool`, `route_model`, etc.), disparada pelo Agent Runtime.

**Pré-condições:** ACB ativo; `Quota` associada com `enforcement` definido (`hard` ou `soft`).

**Fluxo principal (limiar soft):**
1. A syscall consumidora chega ao `ResourceQuotaManager` para reserva/consumo.
2. O consumo projetado ultrapassa o limiar configurado (ex.: 80%) mas não o limite.
3. A syscall é executada normalmente.
4. `EventEmitter` publica `agent.quota.warning` com o percentual de consumo.

**Fluxo principal (excesso hard):**
1. A syscall consumidora chega ao `ResourceQuotaManager`.
2. O consumo projetado ultrapassaria o limite `hard` configurado.
3. O `ResourceQuotaManager` rejeita a reserva atomicamente (script Lua no Redis, sem *double-spend*).
4. Kernel retorna o código apropriado: `AIOS-QUOTA-0001` (tokens/custo), `AIOS-QUOTA-0002` (rate-limit de syscalls) ou `AIOS-QUOTA-0003` (ferramentas/inferências), todos HTTP 429 com `retriable=true` e `retryAfterMs`.
5. `EventEmitter` publica `agent.quota.exceeded`.

**Fluxos alternativos:**
- FA1. Cota com `enforcement=soft` mesmo em excesso: syscall é executada, evento `agent.quota.warning` emitido com `enforcement=soft`, sem bloqueio.

**Exceções:**
- E1. Falha do Redis durante a verificação: Kernel adota postura conservadora — nega a syscall (fail-closed de cota) conforme §9 do brief, para evitar *double-spend* sob degradação.

**Pós-condições:**
- Q1. (Caso hard) Nenhum consumo além do limite é persistido; evento `agent.quota.exceeded` auditado.
- Q2. (Caso soft) Consumo registrado além do limiar; evento `agent.quota.warning` auditado; overshoot rastreável para cobrança/alerta.

---

## UC-013 — Negação de capability pelo PEP

**Ator primário:** Qualquer chamador de syscall privilegiada sem a capability necessária.

**Pré-condições:** Chamador autenticado (claims válidas), mas sem a capability requerida pela ação.

**Fluxo principal:**
1. Chamador envia qualquer syscall privilegiada.
2. `SyscallGateway` despacha ao `CapabilityEnforcer`.
3. `CapabilityEnforcer` monta o `DecisionRequest` e consulta o PDP via `PolicyClient`.
4. PDP retorna `deny`.
5. Kernel **não executa** nenhuma lógica de negócio, não consome cota, não muda o ACB.
6. Kernel retorna `AIOS-CAP-0001` (HTTP 403).
7. `EventEmitter` publica `agent.syscall.denied` com o verbo, o sujeito e o motivo.
8. `syscall_log` registra `decision=deny`, `deny_code=AIOS-CAP-0001`.

**Fluxos alternativos:**
- FA1. Decisão vem do cache do `PolicyClient` (dentro do TTL) em vez de nova chamada ao PDP — comportamento observável idêntico.

**Exceções:**
- E1. Nenhuma (a negação **é** o caminho de exceção de outros casos de uso, mas é ela mesma bem-definida e não-excepcional do ponto de vista do PEP).

**Pós-condições:**
- Q1. Nenhum efeito colateral no ACB, cotas ou brokers a jusante.
- Q2. Evento e auditoria de negação registrados (NFR-010, NFR-011).

---

## UC-014 — Isolamento de tenant (tenant mismatch)

**Ator primário:** Qualquer chamador cujo payload/URN referencia um `tenant` diferente do contexto autenticado.

**Pré-condições:** Chamador autenticado com `X-AIOS-Tenant=A`, mas a URN-alvo ou o payload referencia `tenant=B`.

**Fluxo principal:**
1. Chamador envia syscall referenciando recurso de outro tenant (por engano, tentativa de escalonamento de privilégio, ou bug de cliente).
2. `SyscallGateway`/`CapabilityEnforcer` detecta a divergência entre `tenant` do contexto autenticado e o `tenant` embutido na URN/payload.
3. A syscall é rejeitada **antes** de qualquer consulta ao PDP para o recurso alheio (a verificação de tenant é uma guarda anterior à decisão de capability, pois nenhuma política deveria sequer ser avaliada cross-tenant).
4. Kernel retorna `AIOS-CAP-0002` (HTTP 403), sem revelar se o recurso do outro tenant existe.
5. `EventEmitter` publica `agent.syscall.denied` com motivo `tenant_mismatch`.

**Fluxos alternativos:** Nenhum — este é um caminho estritamente de rejeição.

**Exceções:** Nenhuma adicional.

**Pós-condições:**
- Q1. Nenhuma informação sobre o tenant alheio é exposta na resposta de erro (RFC-0001 §6).
- Q2. Evento de negação auditado com o `tenant` do chamador e o `tenant` alvo rejeitado (para investigação de segurança).

---

## UC-015 — Brokering de syscall de recurso

**Ator primário:** Agent Runtime (`007`), solicitando `remember`, `recall`, `plan`, `invoke_tool` ou `route_model`.

**Atores secundários:** Módulo dono do recurso (`010-Memory`, `012-Planning`, `015-Tool-Manager`, `017-Model-Router`).

**Pré-condições:**
- P1. ACB em `Running`.
- P2. Chamador possui a capability específica do recurso (ex.: `tool:invoke:<tool_id>`).
- P3. Cota do recurso correspondente possui saldo.

**Fluxo principal:**
1. Chamador envia, por exemplo, `POST /v1/kernel/agents/{urn}/tools:invoke` com `Idempotency-Key`.
2. `CapabilityEnforcer` consulta o PDP → `allow` (capability específica da ferramenta/recurso solicitado).
3. `ResourceQuotaManager` reserva a cota do recurso (ex.: `tools_limit`).
4. `ResourceBrokerRouter` encaminha a requisição ao módulo dono (`015-Tool-Manager`), com *circuit breaker*/*bulkhead* dedicados a essa dependência.
5. O módulo dono executa a lógica de negócio (não replicada pelo Kernel) e retorna o resultado.
6. `ResourceQuotaManager` confirma o consumo definitivo da cota (ou libera a reserva parcial, se o custo real for menor que o estimado).
7. Kernel retorna o resultado ao chamador, envelopado conforme o contrato do verbo.
8. `syscall_log`/telemetria registram o verbo, o broker de destino e a latência.

**Fluxos alternativos:**
- FA1 (`recall`, leitura): reserva de cota pode ser dispensada ou mínima (syscall majoritariamente de leitura), mas capability ainda é obrigatória.
- FA2 (`route_model`): o Kernel encaminha ao `017-Model-Router`, que por sua vez pode consultar o `026-Cost-Optimizer` — fora do escopo do Kernel, que apenas aplica capability e cota antes do encaminhamento.

**Exceções:**
- E1. Capability negada → UC-013.
- E2. Cota insuficiente → UC-012.
- E3. Módulo dono indisponível (circuit breaker aberto): `AIOS-KERNEL-0005` (HTTP 502, `retriable=true`); reserva de cota é revertida (não há cobrança por trabalho não realizado).
- E4. Timeout do módulo dono: mesmo código `AIOS-KERNEL-0005`; reserva revertida.

**Pós-condições:**
- Q1. Nenhuma lógica de negócio do recurso é executada pelo Kernel — apenas encaminhamento governado (NR-05 do brief).
- Q2. Cota do recurso reflete exatamente o consumo real (nem mais, nem menos que o efetivamente utilizado).

---

## UC-016 — Repetição idempotente de syscall

**Ator primário:** Agent Runtime (`007`), reenviando uma requisição após timeout de rede ou retry automático de cliente.

**Pré-condições:**
- P1. Uma syscall de mutação foi previamente processada com uma `Idempotency-Key` específica, dentro da janela de retenção (`kernel.idempotency.retention_h`, ≥ 24h).

**Fluxo principal (payload idêntico):**
1. Chamador reenvia a mesma requisição (mesmo verbo, mesmo payload, mesma `Idempotency-Key`).
2. `SyscallGateway` consulta o `IdempotencyStore` **antes** de qualquer outro processamento.
3. Resultado previamente persistido é encontrado e retornado **byte-a-byte idêntico** (mesmo HTTP status, mesmo corpo).
4. Nenhuma nova consulta ao PDP, nenhum novo consumo de cota, nenhum novo evento é publicado.

**Fluxos alternativos:**
- FA1. Repetição de uma syscall que originalmente falhou (ex.: UC-002): o mesmo erro é retornado novamente, de forma consistente.

**Exceções:**
- E1. Mesma `Idempotency-Key`, payload **divergente** → `AIOS-SYSCALL-0002` (HTTP 409); resultado original não é alterado (ver FR-021).
- E2. `Idempotency-Key` expirada (fora da janela de retenção) e reenviada: tratada como nova invocação (novo processamento completo), pois o Kernel não possui mais o registro de deduplicação.

**Pós-condições:**
- Q1. Efeito único garantido: para qualquer número de repetições dentro da janela de retenção com payload idêntico, o estado do sistema é o mesmo que após a primeira execução bem-sucedida (NFR-012).

---

## UC-017 — Conflito de concorrência otimista (OCC)

**Ator primário:** Duas (ou mais) chamadas concorrentes de mutação sobre o **mesmo** ACB (ex.: `suspend` e `checkpoint` quase simultâneos).

**Pré-condições:** ACB com `version=N`; duas requisições leem o mesmo `version` antes de qualquer uma commitar.

**Fluxo principal:**
1. Requisição A lê o ACB com `version=N` e inicia sua transição.
2. Requisição B lê o ACB com o mesmo `version=N` e inicia sua transição.
3. Requisição A commita primeiro, incrementando para `version=N+1`.
4. Requisição B tenta commitar com a condição `WHERE version=N`; a atualização afeta 0 linhas (conflito detectado).
5. O `AcbStore` sinaliza conflito de OCC ao `LifecycleCoordinator`.
6. O `LifecycleCoordinator` reexecuta a operação de B a partir do estado atual (`version=N+1`), respeitando `kernel.acb.optimistic_retry_max`.
7. Se a operação de B ainda é válida após reler o novo estado (guarda da FSM satisfeita), ela é aplicada com sucesso sobre `version=N+1`.

**Fluxos alternativos:**
- FA1. Após reler o novo estado, a operação de B deixa de ser válida (ex.: A já moveu o ACB para um estado do qual a transição de B não é permitida): B falha com `AIOS-KERNEL-0002` (transição inválida), não mais com conflito de OCC.

**Exceções:**
- E1. Número de retries excede `kernel.acb.optimistic_retry_max`: Kernel retorna `AIOS-KERNEL-0009` (HTTP 409, `retriable=true`) ao chamador de B, que pode tentar novamente depois.

**Pós-condições:**
- Q1. O ACB nunca reflete uma escrita perdida (*lost update*) — toda mutação bem-sucedida incrementa `version` de forma estritamente sequencial.
- Q2. Overshoot de conflitos não resolvidos é observável via métrica de retries e taxa de `AIOS-KERNEL-0009`.

---

## UC-018 — Indisponibilidade do PDP (fail-closed)

**Ator primário:** Qualquer chamador de syscall privilegiada durante uma janela de indisponibilidade do `022-Policy` (PDP).

**Pré-condições:** `kernel.pep.fail_mode=closed` (default); PDP não responde dentro do timeout do `PolicyClient` ou o circuit breaker do `PolicyClient` está aberto.

**Fluxo principal:**
1. Chamador envia syscall privilegiada.
2. `CapabilityEnforcer` tenta consultar o PDP via `PolicyClient`.
3. `PolicyClient` detecta indisponibilidade (timeout ou circuit breaker aberto) e não obtém decisão.
4. Como `fail_mode=closed`, o `CapabilityEnforcer` trata a ausência de decisão como `deny`.
5. Kernel retorna `AIOS-CAP-0003` (HTTP 503, `retriable=true`).
6. `EventEmitter` publica `agent.syscall.denied` com motivo `pdp_unavailable`.
7. `KernelTelemetry` eleva alerta operacional (PDP fora do ar impacta 100% das syscalls privilegiadas).

**Fluxos alternativos:**
- FA1. `kernel.pep.fail_mode=open` (configuração explícita, não-default, tipicamente restrita a ambientes de teste): syscalls são permitidas mesmo sem decisão do PDP, com risco assumido e fortemente desencorajado em produção — DEVE ser acompanhado de alerta crítico permanente enquanto ativo.
- FA2. `PolicyClient` em meia-abertura (circuit breaker half-open): uma requisição de sondagem é permitida ao PDP; se bem-sucedida, o breaker fecha e o fluxo normal (UC-013/decisão real) é retomado.

**Exceções:**
- E1. Nenhuma adicional além da correta aplicação do `fail_mode`.

**Pós-condições:**
- Q1. Nenhuma syscall privilegiada é executada sem uma decisão de autorização válida (mesmo que essa decisão seja um `deny` implícito por indisponibilidade) — postura de segurança preservada (NFR-010).
- Q2. Recuperação automática ao religar o PDP dentro da meta de NFR-018.

---

## UC-019 — Atualização de policy bundle (invalidação de cache)

**Ator primário:** `022-Policy` (PDP), publicando evento de atualização de política.

**Pré-condições:** Um novo *policy bundle* foi publicado e versionado pelo `022-Policy`.

**Fluxo principal:**
1. O Kernel consome `aios.<tenant>.policy.bundle.updated`.
2. O `PolicyClient` invalida imediatamente todas as entradas de cache de decisão associadas ao `tenant` do evento.
3. A próxima syscall privilegiada para esse tenant força uma nova consulta ao PDP (cache miss), obtendo a decisão sob o bundle atualizado.

**Fluxos alternativos:**
- FA1. Evento sem escopo de `tenant` (atualização global de bundle base): invalida o cache de todos os tenants.

**Exceções:**
- E1. Falha ao processar o evento (ex.: Kernel temporariamente fora do ar durante a publicação): ao religar, o Kernel consome o evento pendente da assinatura durável do JetStream (at-least-once), garantindo invalidação eventual mesmo que atrasada.

**Pós-condições:**
- Q1. Nenhuma decisão de capability é servida a partir de um cache obsoleto além do tempo de propagação do evento (tipicamente sub-segundo).

---

## UC-020 — Árvore de agentes (spawn de sub-agente)

**Ator primário:** Agent Runtime (`007`), em nome de um agente `Running` que solicita a criação de um sub-agente.

**Pré-condições:**
- P1. O agente pai está em `Running` e possui capability `agent:spawn` (delegada/herdada conforme política do `022-Policy`).
- P2. O limite de *fan-out* configurado para a árvore do agente pai não foi atingido.

**Fluxo principal:**
1. O agente pai invoca `spawn` incluindo `parent_urn` = URN do próprio agente pai.
2. O fluxo segue UC-001 (capability, cota, admissão, materialização), com uma verificação adicional: o `LifecycleCoordinator` consulta a contagem atual de filhos diretos (ou da árvore, conforme política) do `parent_urn` antes de admitir.
3. Se dentro do limite, o ACB filho é criado com `parent_urn` preenchido e segue a FSM normalmente.
4. Se o limite de *fan-out* seria excedido, a criação é rejeitada antes de consumir cota de admissão.

**Fluxos alternativos:**
- FA1. Consulta administrativa da árvore de agentes (para observabilidade/depuração): consulta recursiva por `parent_urn`, exposta via `GetAcb` estendido ou endpoint administrativo — não altera estado.

**Exceções:**
- E1. Limite de fan-out excedido: rejeitado com o código de cota aplicável (`AIOS-QUOTA-0001` se modelado como cota de `agent_count` por sub-árvore, ou `AIOS-QUOTA-0003` se modelado como limite de operações) — o código exato é definido pela configuração de cota associada ao `parent_urn` (Should, ver FR-014).
- E2. Agente pai é encerrado (`kill`) enquanto sub-agentes ainda estão ativos: o Kernel não decide automaticamente a cascata; expõe o estado da árvore para que a política/operador decida (ver UC-009, FA2).

**Pós-condições:**
- Q1. ACB filho referencia corretamente `parent_urn`, permitindo reconstrução da árvore completa de spawn.
- Q2. O custo de coordenação da árvore permanece limitado pelo *fan-out* configurado, evitando crescimento O(N²) (Brief §10).

---

## Notas finais

- Todos os fluxos de exceção referenciam o catálogo de erros `AIOS-<DOMÍNIO>-<NNNN>`
  definido no `_DESIGN_BRIEF.md` §5.2, sem redefini-lo.
- Todo caso de uso que resulta em mutação de estado publica exatamente um
  evento correspondente do catálogo em `_DESIGN_BRIEF.md` §6.1, via Outbox
  transacional (invariante I4 da FSM).
- A tabela de rastreabilidade `UC → FR/NFR → Teste` está consolidada em
  `Requirements.md` §5 e DEVE ser mantida sincronizada com este documento.
