---
Documento: FailureRecovery
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0081 (Cold Agent Hibernation), ADR-0082 (Codec/cifragem de checkpoint), ADR-0084 (Lease + fencing token), ADR-0085 (Migração como saga), ADR-0086 (Event sourcing + Outbox), ADR-0087 (Reconciliação estado-desejado × observado)
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol — a propor)
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 020-Communication, 022-Policy, 025-Audit, 027-Cluster
---

# 008-Agent-Lifecycle — Failure Recovery

> **Escopo.** Este documento especifica, em profundidade, os modos de falha
> conhecidos do módulo Agent Lifecycle (análise estilo FMEA), como cada um é
> detectado, isolado, recuperado, e com quais garantias de idempotência/
> retry/DLQ. Consolida e aprofunda `_DESIGN_BRIEF.md` §9, sem contradizê-lo, e
> referencia — sem redefinir — os contratos de idempotência e correlação de
> RFC-0001 §5.5–§5.6.

---

## 1. Princípios de Recuperação

| Princípio | Aplicação no módulo 008 |
|-----------|------------------------|
| **Fail-Closed sobre Fail-Open** | Sob dúvida (PDP indisponível, storage de snapshot indisponível), o módulo nega/adia a mutação em vez de arriscar um estado inconsistente (`Security.md` §3.3). |
| **Idempotência em toda mutação** | Toda mutação de ciclo de vida aceita `Idempotency-Key` (RFC-0001 §5.5); repetição produz o mesmo efeito, nunca duplo efeito (NFR-013). |
| **Atomicidade estado+evento (Outbox)** | Nenhuma transição de ACB é confirmada sem garantia de que o evento correspondente será eventualmente publicado (R-08 do brief, INV2). |
| **Compensação, não travamento** | Falhas em sagas multi-etapa (`spawn`, `migrate`) são revertidas por compensação explícita, nunca deixam o agente em estado ambíguo. |
| **Isolamento por dependência** | Falha de uma dependência (ex.: MinIO) não DEVE degradar transições que não dependem dela (bulkhead, `Security.md` §9). |
| **Serialização por agente, não lock global** | `LeaseManager` + `fencing_token` garantem que apenas um coordenador aplique efeitos por vez, sem serializar agentes entre si (INV1). |
| **Degradação observável, nunca silenciosa** | Toda negação/degradação/compensação DEVE emitir métrica, log correlacionado e, quando aplicável, evento de auditoria. |

---

## 2. FMEA — Modos de Falha, Efeitos e Análise

> Convenção de severidade: **Alta** (impede operação crítica de ciclo de vida
> ou risco de inconsistência do ACB/checkpoint), **Média** (degrada
> subconjunto de transições, sem risco de inconsistência), **Baixa**
> (degradação cosmética/observável sem impacto funcional).

| # | Modo de falha | Efeito se não mitigado | Severidade | Detecção | Isolamento |
|---|-----------------|----------------------------|--------------|----------|-------------|
| FM-01 | Runtime (`007`) morre em `Running` | Agente pode estar travado consumindo recursos sem produzir progresso; ACB fica preso em estado incoerente com a realidade. | Alta | Ausência de heartbeat > `reconcile.runtime_dead_after_ms` (default 15000). | `ReconciliationController` isolado por agente; demais agentes não afetados. |
| FM-02 | Crash do coordenador (`LifecycleCoordinator`) durante transição | Lease expira; transição fica incompleta, ACB potencialmente em estado intermediário. | Alta | Lease não renovada dentro de `lease.ttl_ms`; `ReconciliationController` detecta transição órfã. | `fencing_token` invalida escritas do coordenador antigo; novo coordenador relê `LifecycleTransition` + saga log. |
| FM-03 | Checkpoint corrompido | Restore/wake reconstrói working memory incorreta ou falha silenciosamente. | Alta | `content_hash` (sha-256) divergente no momento do `restore`/`wake`. | Rejeita imediatamente (`AIOS-LIFECYCLE-0007`); nenhum estado parcialmente restaurado é exposto. |
| FM-04 | MinIO/PostgreSQL (storage de snapshot) indisponível | `hibernate`/`checkpoint`/`restore`/`migrate` bloqueados; risco de reter agentes em RAM além do necessário. | Alta | Erro de I/O no `SnapshotStore`. | Hibernação/migração **adiadas** (agente mantido quente); `AIOS-LIFECYCLE-0014`; demais transições (suspend/resume/terminate de agentes já quentes) não afetadas. |
| FM-05 | Migração travada (nó/shard destino cai durante a saga) | Agente pode ficar "preso" entre origem e destino, potencialmente inacessível em ambos. | Alta | `migration.saga_timeout_ms` (default 120000) excedido. | Compensação: invalida destino, reativa origem — agente nunca perde trabalho aceito; `MigrationJob.status = compensated`. |
| FM-06 | Split-brain — dois coordenadores atuando sobre o mesmo agente | Escritas concorrentes inconsistentes sobre o mesmo ACB/checkpoint. | Alta | `fencing_token` divergente na tentativa de escrita. | Escrita com token obsoleto rejeitada (`AIOS-LIFECYCLE-0013`); apenas o token mais novo persiste. |
| FM-07 | Evento duplicado (entrega at-least-once) | Reprocessamento indevido de uma transição já aplicada. | Média | `event.id` já presente no registro de deduplicação do consumidor. | Consumidor deduplica (RFC-0001 §5.5); no-op idempotente. |
| FM-08 | Storm de hibernação/wake sob pressão de RAM do cluster | Muitas transições simultâneas saturam `CheckpointService`/`SnapshotStore` ou o `WarmPoolManager`. | Média | `hibernation.ram_pressure_threshold_pct` cruzado; fila de materialização crescendo. | `HibernationController` prioriza por `priority_class`/idle; backpressure na admissão (`009-Scheduler`); `migration.max_concurrent_per_node` limita paralelismo de materialização. |
| FM-09 | Outbox não publica (NATS indisponível) | Consumidores de `agent.lifecycle.*` não recebem eventos; sistemas reativos (007/009/014) atrasam reação. | Média | Lag de publicação; linhas `LifecycleOutbox` com `published=false` crescente. | Retry do publicador; eventos persistem no outbox (durável); nenhuma mutação de ACB é perdida (at-least-once). |
| FM-10 | Conflito de `generation` obsoleta em restauração/materialização concorrente | Duas materializações do mesmo agente frio competem, risco de estado duplicado. | Média | Escrita com `generation` desatualizada rejeitada pela camada de persistência. | `AIOS-LIFECYCLE-0013` (mesmo mecanismo de fencing); apenas a materialização com token/generation corrente prossegue. |
| FM-11 | `Idempotency-Key` reutilizada com payload divergente | Ambiguidade sobre qual efeito é o "correto" para um comando repetido. | Média | Comparação de hash do payload associado à chave. | Rejeitado explicitamente (erro de contrato análogo ao padrão transversal, 409), sem executar. |
| FM-12 | Runtime perdido além do limite de recuperação | Agente não pode ser reconciliado nem recuperado a partir do último checkpoint válido. | Alta | `ReconciliationController` esgota tentativas de recuperação dentro de `recovery.rto_budget_s` (default 900). | Transição para `Failed` (T11); slot/cota liberados; evento `agent.lifecycle.failed` emitido; nenhuma retentativa automática indefinida. |
| FM-13 | Storage de checkpoint retorna sucesso mas blob nunca é gravado de fato (falha silenciosa de storage) | `Checkpoint` referencia um `storage_uri` inexistente; futura tentativa de `restore`/`wake` falha tarde demais. | Alta | Verificação de existência/hash pós-escrita antes de confirmar o checkpoint como `consistent`. | `CheckpointService` **NÃO DEVE** marcar a transição `Suspended → Hibernated` como concluída antes de confirmar a gravação (INV4); falha na confirmação mantém o agente em `Suspended`. |

---

## 3. Detecção

| Mecanismo | Onde se aplica | Sinal |
|-----------|------------------|-------|
| Timeout configurável por dependência | `SpawnManager`, `MigrationOrchestrator`, `CheckpointService`, `LifecyclePolicyEnforcer` | Excede `spawn.timeout_ms`, `migration.saga_timeout_ms` ou timeout equivalente (`_DESIGN_BRIEF.md` §8). |
| Ausência de heartbeat de runtime | `ReconciliationController` | Excede `reconcile.runtime_dead_after_ms` desde o último `agent.runtime.heartbeat` consumido. |
| Verificação de `content_hash` | `CheckpointService` (em todo `restore`/`wake`) | Hash calculado diverge do metadado persistido em `Checkpoint`. |
| Conflito de `fencing_token`/`generation` | `AcbStore` (camada de persistência do ACB) | Escrita com token/geração obsoleta rejeitada na transação. |
| Expiração de lease sem renovação | `LeaseManager` (Redis) | TTL da chave `aios:{tenant}:lc:lease:{agent_id}` expira sem renovação — coordenador presumido morto. |
| Varredura do Outbox | `LifecycleEventPublisher` (relay) | Linhas em `LifecycleOutbox` com `published=false` além do intervalo esperado. |
| Health probe de dependência | `readyz` do módulo 008 (`Deployment.md` §7) | PostgreSQL/Redis/MinIO/NATS inacessíveis. |
| Loop de reconciliação periódico | `ReconciliationController` | Compara `desired_state` × `state` observado do ACB a cada `reconcile.interval_ms` (default 2000); deriva detectada quando divergem além de um limiar de tempo. |

Todo evento de detecção **DEVE** ser correlacionado com `trace_id`/
`tenant_id` (RFC-0001 §5.6) e refletido em métrica (`Metrics.md`) para
permitir alerta e análise de causa raiz.

---

## 4. Isolamento

O isolamento segue o padrão **bulkhead + circuit breaker por dependência**,
detalhado em `Security.md` §9: cada dependência externa (PDP, Scheduler,
Runtime, storage de snapshot) possui *pool* de conexões/circuito próprio, de
modo que a saturação ou falha de uma não consome a capacidade disponível
para as demais. A tabela abaixo resume o comportamento de isolamento por
modo de falha da FMEA:

| Modo de falha (FMEA) | Isolamento aplicado |
|-------------------------|-------------------------|
| FM-01 (runtime morto) | Apenas o agente afetado é reconciliado; demais agentes `Running` não são impactados. |
| FM-02 (crash do coordenador) | Lease por agente garante que apenas a transição daquele agente específico fica pendente; outros agentes continuam sendo coordenados normalmente por outras réplicas. |
| FM-04 (storage indisponível) | Apenas transições que dependem de storage (`hibernate`/`checkpoint`/`restore`/`migrate`) são bloqueadas; `suspend`/`resume`/`terminate` de agentes já quentes continuam operando via PostgreSQL/Redis. |
| FM-05 (migração travada) | Apenas o `MigrationJob` específico é afetado; demais migrações em curso (até `migration.max_concurrent_per_node`) não são impactadas. |
| FM-08 (storm de hibernação/wake) | Backpressure limita paralelismo por nó/shard; agentes já `Running` não são forçados a hibernar além da política configurada. |
| FM-09 (Outbox/NATS) | Persistência de estado no PostgreSQL continua normalmente; apenas a publicação de eventos é retida, sem bloquear novas transições. |

---

## 5. Recuperação — por Modo de Falha

### 5.1 FM-01 — Runtime morto em `Running`

1. `LifecycleEventConsumer` monitora `agent.runtime.heartbeat`
   (produtor: `007-Agent-Runtime`) por agente.
2. Ausência de heartbeat além de `reconcile.runtime_dead_after_ms` dispara o
   `ReconciliationController`.
3. O controlador tenta materializar o agente a partir do **último
   checkpoint válido** (RPO ≤ 5 min, NFR-006) → `Ready`/`Running` (trigger
   `wake`/`reconcile`).
4. Se a recuperação falhar repetidamente além de `recovery.rto_budget_s`
   (default 900) → T11: ACB → `Failed`, com `failure_code =
   AIOS-LIFECYCLE-0015`.

### 5.2 FM-02 — Crash do coordenador durante transição

1. A lease do agente (Redis) não é renovada dentro de `lease.ttl_ms`
   (default 5000) e expira.
2. Qualquer réplica saudável do módulo 008 **PODE** adquirir a lease
   liberada, obtendo um novo `fencing_token` (monotonicamente maior).
3. O novo coordenador relê `LifecycleTransition` (log de transições) e o
   estado do `MigrationJob`/`Checkpoint` em andamento, determinando se a
   transição deve ser **retomada** (efeito ainda não confirmado) ou
   **compensada** (efeito parcialmente aplicado e não seguro de continuar).
4. Qualquer escrita tardia do coordenador antigo (que eventualmente
   "acorda" após um GC pause, por exemplo) é rejeitada pela validação de
   `fencing_token`, prevenindo corrupção por split-brain (FM-06).

### 5.3 FM-03 — Checkpoint corrompido

1. `CheckpointService` recalcula `content_hash` (sha-256) do blob lido de
   MinIO antes de aceitar seu conteúdo em qualquer `restore`/`wake`.
2. Divergência → rejeita imediatamente com `AIOS-LIFECYCLE-0007` (422),
   **sem** aplicar qualquer estado parcial ao agente.
3. O `CheckpointService` **DEVERIA** tentar o checkpoint **anterior** válido
   do mesmo agente (se existir e estiver dentro da política de retenção),
   aceitando a perda de progresso entre os dois pontos (dentro do orçamento
   de RPO ≤ 5 min, NFR-006).
4. Se nenhum checkpoint válido existir → T11: ACB → `Failed`, com evento
   auditável detalhando a corrupção detectada (`025-Audit`).

### 5.4 FM-04 — Storage de snapshot indisponível

```
   Transição solicitada: Suspended → Hibernated (T6)
        │
        ▼
   CheckpointService tenta gravar blob em MinIO
        │
        ├── sucesso ──▶ hash confirmado ──▶ Hibernated (RAM liberada)
        │
        └── falha (I/O) ──▶ AIOS-LIFECYCLE-0014 (503, retriable)
                              │
                              ▼
                    Agente permanece em Suspended
                    (RAM NÃO é liberada até checkpoint confirmado — INV4)
```

1. Erro de I/O no `SnapshotStore` (MinIO ou índice PostgreSQL) é detectado
   pelo `CheckpointService`.
2. A transição **NÃO avança** — o agente permanece no estado anterior
   seguro (`Suspended`, não `Hibernated`), preservando a invariante INV4
   (RAM só é liberada após checkpoint `consistent` confirmado).
3. Retry com backoff exponencial + jitter (§7); se persistente, alerta de
   alta prioridade — hibernações/migrações ficam **adiadas** globalmente
   até recuperação do storage (degradação graciosa, §10), mas agentes
   continuam operando normalmente enquanto quentes.

### 5.5 FM-05 — Migração travada

1. `MigrationOrchestrator` monitora a saga (`quiesce → checkpoint →
   transfer → materialize → cutover → cleanup`) contra
   `migration.saga_timeout_ms` (default 120000).
2. Timeout em qualquer fase antes de `cutover` confirmado dispara
   compensação: o destino é invalidado (runtime/slot reservado é liberado),
   e a origem é reativada a partir do estado anterior à migração (que nunca
   foi invalidada antes do `cutover` confirmado — G8 do brief).
3. `MigrationJob.status = compensated`; evento correspondente emitido;
   agente permanece acessível na origem durante todo o processo — **nenhum
   trabalho aceito é perdido**.
4. Cliente pode reenviar `migrate` (nova tentativa, tipicamente com novo
   destino escolhido por `009`/`027`) — operação idempotente por natureza
   de saga compensável.

### 5.6 FM-06 — Split-brain (fencing token)

1. Dois processos (coordenador antigo "zumbi" + novo coordenador) tentam
   escrever efeitos para o mesmo agente.
2. A escrita que carrega o `fencing_token` **mais recente** (obtido pela
   aquisição de lease mais nova) é aceita; a escrita com token obsoleto é
   rejeitada com `AIOS-LIFECYCLE-0013` (409, não retriable para aquele
   token específico — o chamador deve readquirir a lease).
3. Nenhuma reconciliação manual é necessária: o mecanismo é auto-corretivo
   por construção (a cada tentativa de escrita, o token é revalidado).

### 5.7 FM-07 — Evento duplicado

1. Consumidor de `agent.lifecycle.*` (ou de eventos consumidos pelo módulo
   008: `scheduler.placement.decided`, `agent.runtime.exited`, etc.)
   verifica `event.id` contra seu registro de deduplicação antes de
   processar.
2. `event.id` já visto → no-op (idempotência RFC-0001 §5.5), sem reexecutar
   efeitos colaterais.

### 5.8 FM-08 — Storm de hibernação/wake

1. `HibernationController` detecta `hibernation.ram_pressure_threshold_pct`
   cruzado (default 85%) e prioriza candidatos a `hibernate` por
   `priority_class`/tempo de idle, em vez de processar em ordem arbitrária.
2. `MigrationOrchestrator`/`SpawnManager` aplicam
   `migration.max_concurrent_per_node` (default 8) como teto de
   paralelismo de materialização por nó de destino — excesso é enfileirado
   (JetStream aplica pressão nos consumidores), não descartado nem
   processado de forma a saturar o nó.
3. Nenhum agente é forçado a hibernar/acordar fora da política configurada
   apenas para aliviar a fila — a fila absorve o pico, respeitando os SLOs
   de latência dos agentes já em `Running`.

### 5.9 FM-09 — Outbox não publica

1. `LifecycleEventPublisher` detecta falha de publicação (conexão NATS
   indisponível).
2. Eventos permanecem em `LifecycleOutbox` com `published=false` (já
   persistidos de forma durável na mesma transação da mutação do ACB —
   garantia estrutural idêntica ao padrão Outbox de `../001-Architecture/Architecture.md` §13).
3. Relay retry com backoff até reconexão, então drena o backlog **na ordem
   de inserção** por agente/stream, preservando a ordenação exigida
   (RFC-0001 §5.3).
4. Consumidores deduplicam por `event.id`; nenhuma ação corretiva adicional
   é necessária do lado do consumidor.

### 5.10 FM-12 — Runtime perdido além do limite de recuperação

1. `ReconciliationController` esgota as tentativas de materialização a
   partir do último checkpoint válido dentro do orçamento
   `recovery.rto_budget_s` (default 900s, RTO ≤ 15 min — NFR-005).
2. T11 é acionada: ACB → `Failed`; cota/slot são liberados (compensação);
   evento `agent.lifecycle.failed` é emitido com `failure_code =
   AIOS-LIFECYCLE-0015`.
3. Nenhuma tentativa automática adicional de `wake`/`resume` é feita pelo
   módulo 008 — cabe ao chamador (ou a uma política de auto-restart em
   `009`/aplicação, fora do escopo do 008) decidir se um novo `spawn` é
   apropriado.

### 5.11 FM-13 — Falha silenciosa de gravação de checkpoint

1. `CheckpointService` **DEVE** confirmar a existência e o hash do blob
   recém-gravado (leitura de confirmação ou verificação de resposta de
   commit do storage) antes de marcar o `Checkpoint` como `consistent` e
   antes de permitir a transição `Suspended → Hibernated`.
2. Falha na confirmação mantém o agente em `Suspended` (RAM não liberada) e
   retorna `AIOS-LIFECYCLE-0011`/`AIOS-LIFECYCLE-0014` conforme a causa
   (serialização vs. storage), preservando INV4.

---

## 6. Idempotência (aplicação no contexto de falha)

Toda a estratégia de recuperação acima depende da garantia de idempotência
de RFC-0001 §5.5, aplicada pelo módulo 008 via `LifecycleApiSurface` +
`AcbStore`:

| Garantia | Mecanismo |
|----------|-----------|
| Repetição de uma mutação com a mesma `Idempotency-Key` retorna o mesmo resultado. | Resultado persistido por chave por, no mínimo, 24h (RFC-0001 §5.5). |
| Reuso da mesma chave com payload **diferente** é um erro do cliente, não reexecutado. | Comparação de hash do payload; divergência → erro 409 do catálogo transversal. |
| Consumidores de eventos não duplicam efeito sob entrega repetida. | Deduplicação por `event.id` (RFC-0001 §5.2), responsabilidade do consumidor. |
| Retentativas de rede (timeout do cliente sem confirmação de resposta) são seguras. | Mesma chave reenviada localiza o resultado já processado, mesmo que a primeira resposta tenha se perdido em trânsito. |
| Transições de FSM já concluídas não são reaplicadas por retry. | `StateMachineEngine` valida o estado atual antes de aplicar; uma transição já efetivada para o estado-alvo é tratada como sucesso idempotente, não como erro. |

---

## 7. Retries e Backoff

| Operação/dependência | Máximo de tentativas | Estratégia de backoff | Jitter |
|--------------------------|--------------------------|----------------------------|--------|
| Conflito de `fencing_token`/`generation` (`AcbStore`) | Configurável internamente (curto) | Backoff exponencial curto (ex.: 5ms, 10ms, 20ms, ...) | Sim, para evitar *retry storm* sob conflito simultâneo. |
| Gravação/leitura de checkpoint (MinIO/PostgreSQL) | Limitado, com fallback ao checkpoint anterior (§5.3) | Backoff exponencial + jitter | Sim. |
| Chamada ao PDP (`LifecyclePolicyEnforcer`) | Governado pelo circuit breaker, não por contagem fixa | Circuit breaker com meia-abertura periódica | Implícito no intervalo de sonda. |
| Chamada ao Scheduler (`009`)/Runtime (`007`) durante `spawn`/`migrate` | 1 tentativa síncrona dentro do timeout da saga; retry é responsabilidade do **cliente externo** reenviando o comando | N/A no lado do módulo 008 (a saga não reintenta indefinidamente — falha explícita, ACB permanece em estado seguro ou vai a `Failed`) | N/A |
| Relay do Outbox | Contínuo (job recorrente) até sucesso | Retry indefinido com backoff limitado (o evento não é descartado) | Sim. |
| Materialização a partir de checkpoint (`ReconciliationController`) | Limitado por `recovery.rto_budget_s` | Backoff exponencial + jitter | Sim. |

**Regra normativa:** o módulo 008 **NÃO DEVE** reintentar automaticamente e
de forma indefinida uma mutação privilegiada em nome do chamador quando a
falha é de **negação** (política/cota) — apenas falhas **transitórias de
infraestrutura** (timeout de dependência, conflito de fencing) justificam
retry interno limitado; negações são definitivas até nova decisão do PDP.

---

## 8. Dead-Letter Handling

O módulo 008 não opera uma fila de mensagens própria (delegada ao
NATS/JetStream, `../020-Communication/`), mas define comportamento de
"poison message" nos dois pontos onde lida com falhas repetidas de
processamento:

| Fonte | Condição de "poison" | Tratamento |
|-------|----------------------|--------------|
| `LifecycleOutbox` | Evento falha em ser publicado após um número elevado de tentativas do relay (ex.: payload corrompido, subject inválido). | Marcado para inspeção manual (flag/tabela de exceção); **NÃO É** descartado silenciosamente — alerta de alta prioridade, pois representa risco de perda de evento. |
| Eventos consumidos (`scheduler.placement.decided`, `agent.runtime.exited`, `task.execution.completed`, `cluster.node.draining`) | Handler falha repetidamente ao processar um evento (ex.: referência a `agent_id` inexistente). | DLQ nativa do JetStream (`../020-Communication/`); o módulo 008 **DEVE** logar e emitir métrica de "evento não processável", nunca travar o consumo dos demais eventos do stream. |

---

## 9. RTO / RPO

| Meta | Valor | Escopo |
|------|-------|--------|
| **RTO** (Recovery Time Objective) | `≤ 15 minutos` (`recovery.rto_budget_s` = 900) | Tempo para restaurar um agente após falha de nó/runtime (NFR-005). |
| **RPO** (Recovery Point Objective) | `≤ 5 minutos` | Perda máxima tolerável de progresso de working memory, garantida por checkpoint periódico (`checkpoint.interval_s`, default 120) + checkpoint obrigatório em `suspend`/`hibernate` (NFR-006). |

**Como as metas são atingidas:**

- **RTO**: o `ReconciliationController` detecta runtime morto dentro de
  `reconcile.runtime_dead_after_ms` (default 15s) e inicia materialização a
  partir do último checkpoint válido; o processo inteiro (detecção +
  materialização) **DEVE** concluir dentro do orçamento de
  `recovery.rto_budget_s`. PostgreSQL em modo HA com *streaming
  replication* (`027-Cluster`) e réplicas stateless do módulo 008 permitem
  reconexão rápida a uma nova topologia de banco.
- **RPO**: checkpoints periódicos (`checkpoint.interval_s`) somados a
  checkpoint obrigatório em toda transição `Running → Suspended` e
  `Suspended → Hibernated` garantem que a janela de perda potencial nunca
  excede o intervalo configurado somado ao tempo do último checkpoint bem
  sucedido — auditável via `aios_lifecycle_checkpoint_duration_ms` e o
  timestamp `created_at` do último `Checkpoint`.
- Simulações de recuperação (*DR drills*) **DEVEM** ser executadas
  periodicamente, incluindo o cenário de "matar o runtime no meio de uma
  tarefa" e "matar o coordenador entre commit e publish", para validar
  empiricamente as garantias de RTO/RPO e de zero perda de evento
  (NFR-008 sob análise de outbox).

---

## 10. Degradação Graciosa

Quando uma dependência não-crítica está degradada, o módulo 008 **DEVERIA**
continuar atendendo o máximo de superfície possível, seguindo esta ordem de
prioridade (da mais para a menos protegida):

1. **Segurança e isolamento de tenant** — nunca degradado; sempre
   fail-closed (`Security.md` §3.3).
2. **Integridade do ACB (FSM/fencing)** — nunca degradado; nenhuma
   transição inconsistente é aceita mesmo sob pressão, e RAM de agente
   hibernado nunca é liberada sem checkpoint `consistent` confirmado
   (INV4).
3. **Continuidade de agentes já quentes** (`Running`/`Suspended`) — mantida
   o máximo possível; sob indisponibilidade de storage de snapshot, o
   módulo **suspende novas hibernações e migrações**, preservando agentes
   quentes em vez de arriscar perda de trabalho (§5.4).
4. **Latência de leitura** (`GetState`, histórico de transições) — pode
   degradar (maior latência via fallback direto ao PostgreSQL) antes de
   recusar a leitura por completo.
5. **Emissão de eventos/auditoria** — pode atrasar (retido no Outbox) sem
   bloquear a transição original, desde que a garantia de entrega eventual
   seja preservada.

Esta ordem reflete o princípio central do brief (`_DESIGN_BRIEF.md` §9):
*"sob storage indisponível, o módulo mantém agentes quentes e suspende
novas hibernações/migrações, preservando a FSM em memória + PG."*

---

## 11. Testes de Caos (referência)

A validação empírica dos comportamentos descritos acima (não apenas o
desenho) é responsabilidade de `./Testing.md`, que **DEVE** incluir, no
mínimo, os seguintes cenários de caos derivados desta FMEA:

| Cenário de caos | Modo de falha exercitado |
|---------------------|------------------------------|
| Matar o processo do runtime (007) durante execução ativa | FM-01 |
| Matar o coordenador (008) entre aquisição de lease e conclusão de efeito | FM-02 |
| Corromper deliberadamente um blob de checkpoint em MinIO | FM-03 |
| Derrubar MinIO/PostgreSQL durante rajada de `hibernate` | FM-04 |
| Derrubar o nó de destino no meio de uma saga de migração | FM-05 |
| Forçar dois coordenadores a disputar a mesma lease (fault injection) | FM-06 |
| Particionar a rede entre módulo 008 e NATS | FM-09 |
| Simular storm de `wake` em massa após reconexão de um tenant grande | FM-08 |
| Interromper heartbeat de um runtime materializado além do RTO budget | FM-12 |
| Injetar falha de confirmação pós-escrita no `SnapshotStore` | FM-13 |

---

## 12. Referências

- Modos de falha e metas de recuperação (fonte): `./_DESIGN_BRIEF.md` §9.
- Máquina de estados do ACB (transições T1..T11): `./_DESIGN_BRIEF.md` §4.
- Postura de segurança fail-closed: `./Security.md` §3.3, §9.
- Modelo de concorrência (lease, fencing token): `./Scalability.md` §4.
- Idempotência e correlação (contratos centrais): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.5–§5.6.
- Topologia de implantação e probes de saúde: `./Deployment.md` §7, §11.
- Cluster/HA/DR (transversal): `../027-Cluster/`.
- Auditoria imutável: `../025-Audit/`.
- Glossário: `../040-Glossary/Glossary.md` (FMEA, RTO/RPO, DLQ, Circuit Breaker, Bulkhead, Saga, Idempotency, Cold Agent).
