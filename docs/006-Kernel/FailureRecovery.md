---
Documento: FailureRecovery
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0061 (ACB & OCC), ADR-0062 (Token-bucket atômico em Redis), ADR-0063 (PEP no Kernel vs. PDP), ADR-0064 (Saga de ciclo de vida), ADR-0066 (Outbox + JetStream)
RFCs relacionados: RFC-0001 (baseline), RFC-0007 (ACB & Resource Quota Model, a propor)
Depende de: 009-Scheduler, 008-Lifecycle, 022-Policy, 020-Communication, 027-HA-DR, 025-Audit
---

# 006-Kernel — Failure Recovery

> **Escopo.** Este documento especifica, em profundidade, os modos de falha
> conhecidos do Kernel (análise estilo FMEA), como cada um é detectado, isolado,
> recuperado, e com quais garantias de idempotência/retry/DLQ. Consolida e
> aprofunda `_DESIGN_BRIEF.md` §9, sem contradizê-lo, e referencia — sem
> redefinir — os contratos de idempotência e correlação de RFC-0001 §5.5–§5.6.

---

## 1. Princípios de Recuperação

| Princípio | Aplicação no Kernel |
|-----------|------------------------|
| **Fail-Closed sobre Fail-Open** | Sob dúvida (dependência de segurança/cota indisponível), o Kernel nega em vez de permitir (`Security.md` §3.3). |
| **Idempotência em toda mutação** | Toda syscall mutante aceita `Idempotency-Key` (RFC-0001 §5.5); repetição produz o mesmo efeito, nunca duplo efeito. |
| **Atomicidade estado+evento (Outbox)** | Nenhuma mutação de ACB é confirmada sem garantia de que o evento correspondente será eventualmente publicado (R-07 do brief). |
| **Compensação, não travamento** | Falhas em sagas multi-etapa (`spawn`) são revertidas por compensação explícita, nunca deixam o ACB em estado ambíguo. |
| **Isolamento por dependência** | Falha de uma dependência (ex.: `015-Tool-Manager`) não DEVE degradar syscalls que não dependem dela (bulkhead, `Security.md` §8). |
| **Degradação observável, nunca silenciosa** | Toda negação/degradação DEVE emitir métrica, log correlacionado e, quando aplicável, evento de auditoria. |

---

## 2. FMEA — Modos de Falha, Efeitos e Análise

> Convenção de severidade: **Alta** (impede operação crítica do Kernel ou risco
> de inconsistência de dado autoritativo), **Média** (degrada subconjunto de
> syscalls, sem risco de inconsistência), **Baixa** (degradação cosmética/observável
> sem impacto funcional).

| # | Modo de falha | Efeito se não mitigado | Severidade | Detecção | Isolamento |
|---|-----------------|----------------------------|--------------|----------|-------------|
| FM-01 | PDP (`022-Policy`) indisponível ou lento | Syscalls privilegiadas travam ou (pior) são permitidas sem decisão. | Alta | Timeout no `PolicyClient`; circuit breaker mede taxa de erro. | Bulkhead dedicado no `PolicyClient` (`Security.md` §8). |
| FM-02 | Scheduler (`009`) indisponível | `spawn`/`resume` não avançam; ACB preso em `Pending`. | Alta | Timeout de admissão (`kernel.spawn.admission_timeout_ms`). | Saga aborta isoladamente; demais syscalls não afetadas. |
| FM-03 | Falha de materialização do runtime (`008-Lifecycle`) | Agente nunca chega a `Running`; recursos reservados ficam presos. | Alta | `kernel.spawn.boot_timeout_ms` excedido. | Compensação libera slot/cota; ACB → `Failed`. |
| FM-04 | Crash do processo Kernel entre commit da transação e publicação do evento | Evento de lifecycle nunca é emitido apesar do estado já mutado. | Alta | Relay do Outbox varre `kernel.outbox` por `published=false`. | Estado em PostgreSQL permanece íntegro (commit atômico com a escrita do outbox). |
| FM-05 | Conflito de concorrência otimista (OCC) | Duas mutações concorrentes competem pelo mesmo ACB. | Média | `UPDATE` afeta 0 linhas (`version` divergente). | Transação isolada; nenhuma outra syscall é afetada. |
| FM-06 | Falha de um broker de recurso (`010`/`011`/`012`/`015`/`017`) | Syscall de recurso específica falha; risco de propagação a outras dependências. | Média | Circuit breaker por dependência no `ResourceBrokerRouter`. | Bulkhead isola por dependência; demais brokers não afetados. |
| FM-07 | Overshoot de cota sob corrida de alta concorrência | Consumo além do limite `hard` antes da rejeição surtir efeito. | Média | Reconciliação periódica token-bucket (Redis) vs. limites (PostgreSQL). | Contador atômico via script Lua (`Scalability.md` §5.1) minimiza a janela. |
| FM-08 | Partição de rede entre Kernel e NATS/JetStream | Eventos não chegam a consumidores; backlog cresce. | Média | Health-check de conexão do `EventEmitter`; alerta de backlog do Outbox. | Outbox retém eventos até reconexão; syscalls continuam sendo aceitas. |
| FM-09 | Perda/indisponibilidade de Redis (contadores quentes / ACB hot) | Enforcement de cota e leitura rápida de ACB degradam. | Alta | Probe de saúde do cliente Redis. | Fallback conservador aos limites do PostgreSQL (nega dúvidas, §7). |
| FM-10 | Réplica lenta de leitura do PostgreSQL usada indevidamente para decisão de mutação | Decisão de FSM baseada em dado obsoleto (*stale read*). | Alta | Verificação de `version` na própria transação de escrita (OCC cobre isso estruturalmente). | Escrita sempre valida `version` no momento do `UPDATE`, não confia em leitura anterior isolada. |
| FM-11 | `Idempotency-Key` reutilizada com payload divergente | Ambiguidade sobre qual efeito é o "correto". | Média | Comparação de hash do payload associado à chave no `IdempotencyStore`. | Rejeitado explicitamente com `AIOS-SYSCALL-0002` (409), sem executar. |
| FM-12 | Heartbeat do runtime perdido (agente `Running` sem sinal de vida) | Agente pode estar travado consumindo recursos sem produzir progresso. | Alta | `kernel.runtime.heartbeat_deadline_ms` excedido. | ACB → `Failed` (T-12); cota/slot liberados; evento `agent.lifecycle.failed`. |
| FM-13 | Falha ao restaurar checkpoint durante `resume` de agente `Hibernated` | Agente não retoma; risco de perda de progresso cognitivo. | Alta | Erro/timeout do `LifecycleClient` na chamada de restore. | ACB permanece `Hibernated` (não avança para `Admitted` sem confirmação); `checkpoint_ref` anterior preservado para nova tentativa. |
| FM-14 | Auditoria (`025-Audit`) indisponível | Ação privilegiada executa sem trilha imutável correspondente. | Alta (conformidade) | Falha/timeout na emissão do evento de auditoria. | Syscall **não é bloqueada** (auditoria é assíncrona), mas gera alerta de "gap de auditoria" de alta prioridade. |

---

## 3. Detecção

| Mecanismo | Onde se aplica | Sinal |
|-----------|------------------|-------|
| Timeout configurável por dependência | `PolicyClient`, `SchedulerClient`, `LifecycleClient`, `ResourceBrokerRouter` | Excede `*_timeout_ms` correspondente (`_DESIGN_BRIEF.md` §8). |
| Circuit breaker (taxa de erro) | Todo cliente de dependência externa | Taxa de erro > `kernel.broker.circuit_breaker_threshold` (default 0,5) abre o circuito. |
| Heartbeat de runtime | `LifecycleCoordinator` | Ausência de sinal de vida além de `kernel.runtime.heartbeat_deadline_ms`. |
| Varredura do Outbox | `EventEmitter` (relay) | Linhas em `kernel.outbox` com `published=false` além de `kernel.outbox.relay_interval_ms`. |
| Reconciliação de cota | `ResourceQuotaManager` (job periódico) | Divergência entre contador Redis e saldo esperado em PostgreSQL. |
| Health probe de dependência | `readyz` do Kernel (`Deployment.md` §7) | PostgreSQL/Redis/NATS inacessíveis. |
| Conflito de OCC | Camada de persistência do `AcbStore` | `UPDATE` retorna 0 linhas afetadas. |

Todo evento de detecção **DEVE** ser correlacionado com `trace_id`/`tenant_id`
(RFC-0001 §5.6) e refletido em métrica (`Metrics.md`) para permitir alerta e
análise de causa raiz.

---

## 4. Isolamento

O isolamento segue o padrão **bulkhead + circuit breaker por dependência**,
detalhado em `Security.md` §8: cada dependência externa (PDP, Scheduler,
Lifecycle, e cada broker de recurso individualmente) possui *pool* de
conexões/threads e circuito próprios, de modo que a saturação ou falha de uma
não consome a capacidade disponível para as demais. A tabela abaixo resume o
comportamento de isolamento por modo de falha da FMEA:

| Modo de falha (FMEA) | Isolamento aplicado |
|-------------------------|-------------------------|
| FM-01 (PDP) | Syscalls não-privilegiadas (leituras simples) continuam operando; apenas syscalls privilegiadas são negadas. |
| FM-02/FM-03 (Scheduler/Lifecycle) | Apenas `spawn`/`resume` são afetados; `suspend`/`kill`/`get_quota` de agentes já materializados continuam. |
| FM-06 (broker de recurso) | Apenas a syscall daquele recurso específico (`remember`, `plan`, etc.) falha; as demais syscalls de recurso não são afetadas. |
| FM-08 (partição NATS) | Persistência de estado no PostgreSQL continua normalmente; apenas a publicação de eventos é retida. |
| FM-09 (Redis) | Enforcement de cota degrada para modo conservador; FSM do ACB continua via PostgreSQL (mais lenta, porém correta). |

---

## 5. Recuperação — por Modo de Falha

### 5.1 FM-01 — PDP indisponível

1. `PolicyClient` detecta timeout/erro e incrementa contador de falha do
   circuito.
2. Ao ultrapassar `kernel.broker.circuit_breaker_threshold`, o circuito abre:
   novas syscalls privilegiadas são negadas **imediatamente** (sem nova
   tentativa de rede) com `AIOS-CAP-0003` (503, `retriable=true`).
3. Após o intervalo de "meia-abertura" do circuito, uma amostra de tráfego é
   permitida como sonda; sucesso fecha o circuito, falha reabre.
4. Cliente (agente/SDK) **DEVE** respeitar `retryAfterMs` do envelope de erro
   (RFC-0001 §5.4) antes de repetir — a repetição com o mesmo
   `Idempotency-Key` é segura.

### 5.2 FM-02/FM-03 — Scheduler/Lifecycle indisponível durante `spawn`

1. `LifecycleCoordinator` inicia a saga de `spawn`: cria ACB em `Pending`
   (T-01), aguarda resposta do `SchedulerClient` dentro de
   `kernel.spawn.admission_timeout_ms`.
2. Timeout ou rejeição → T-03: ACB → `Failed`; toda reserva de cota feita para
   o `spawn` é **compensada** (liberada) antes de o estado terminal ser
   persistido, garantindo que `Failed` nunca retenha recurso preso.
3. Análogo para falha em `kernel.spawn.boot_timeout_ms` durante a
   materialização (T-05).
4. Cliente pode reenviar `spawn` com o **mesmo** `Idempotency-Key`: como o ACB
   original já terminou em `Failed`, o `IdempotencyStore` reconhece a repetição
   e ou (a) retorna o resultado já conhecido (`Failed`) se o payload for
   idêntico, ou (b) — se a intenção for uma nova tentativa lógica — o cliente
   **DEVE** usar uma nova `Idempotency-Key`, pois `spawn` com a mesma chave é
   por definição a mesma operação, não um retry semântico de "tentar de novo".

### 5.3 FM-04 — Crash entre commit e publicação (garantia Outbox)

```
   Transação DB (atômica)                    Processo separado (relay)
   ┌─────────────────────────┐               ┌─────────────────────────┐
   │ BEGIN                    │               │  a cada                 │
   │  UPDATE kernel.acb ...   │               │  kernel.outbox.relay_   │
   │  INSERT INTO kernel.     │               │  interval_ms:           │
   │    outbox (published=    │               │   SELECT * FROM outbox  │
   │    false) ...            │               │   WHERE published=false│
   │ COMMIT                   │◀── crash aqui │   → publish no NATS     │
   │                          │    é seguro:   │   → UPDATE published=  │
   │                          │    outbox já   │     true                │
   │                          │    persistido  │                        │
   └─────────────────────────┘               └─────────────────────────┘
```

- Como a mutação do ACB e a inserção no Outbox ocorrem na **mesma transação
  PostgreSQL**, um crash a qualquer momento — inclusive imediatamente após o
  `COMMIT` e antes do processo de relay rodar — deixa o outbox com a linha
  pendente (`published=false`) já persistida de forma durável.
- Ao reiniciar (ou em uma réplica saudável), o relay do Outbox varre
  `published=false` e republica.
- Consumidores **DEVEM** deduplicar por `event.id` (RFC-0001 §5.2/§5.5) — isso
  cobre o caso raro em que o relay publica e o crash ocorre antes de marcar
  `published=true`, causando uma segunda publicação do mesmo evento.
- **Garantia resultante:** perda de evento = 0 (NFR-007); duplicação é possível
  mas inofensiva graças à deduplicação por `event.id` (semântica *at-least-once*
  + idempotência = *exactly-once* efetivo, RFC-0001 §5.2).

### 5.4 FM-05 — Conflito de OCC

1. `UPDATE ... WHERE version = $esperado` afeta 0 linhas.
2. O `LifecycleCoordinator` (ou o handler de syscall correspondente) relê o ACB
   atual, reavalia se a transição desejada ainda é válida dado o novo estado, e
   tenta novamente.
3. Repetido até `kernel.acb.optimistic_retry_max` (default 5); esgotado, retorna
   `AIOS-KERNEL-0009` (409, `retriable=true`) ao chamador, que **PODE**
   reenviar a syscall (idempotente por natureza da transição de FSM).

### 5.5 FM-06 — Falha de broker de recurso

1. Circuit breaker específico da dependência (ex.: `015-Tool-Manager`) abre
   após taxa de erro configurada.
2. `ResourceBrokerRouter` retorna `AIOS-KERNEL-0005` (502, `retriable=true`)
   apenas para syscalls daquele recurso; demais brokers operam normalmente.
3. Cliente aplica retry com backoff (§7) — a operação de recurso já foi
   validada por capability+cota antes do encaminhamento, então o retry reutiliza
   o mesmo `Idempotency-Key` sem reexecutar verificações redundantes de forma
   não seguras (a validação em si é idempotente).

### 5.6 FM-07 — Overshoot de cota sob corrida

1. Script Lua atômico (`Scalability.md` §5.1) minimiza a janela de corrida a
   praticamente zero para o caminho de decisão em si.
2. Um job de reconciliação periódico compara o contador Redis por
   `subject_urn` contra o limite vigente em PostgreSQL; divergência acima de um
   limiar aciona alerta e, se necessário, correção conservadora do contador
   (nunca "perdoa" consumo já ocorrido, apenas ajusta o saldo restante para
   baixo em caso de dúvida).
3. Meta: overshoot ≤ 1% mesmo sob concorrência extrema (NFR-009).

### 5.7 FM-08 — Partição do NATS

1. `EventEmitter` detecta perda de conexão com o cluster NATS/JetStream.
2. Eventos continuam sendo **inseridos no Outbox** normalmente (a mutação de
   estado no PostgreSQL nunca depende da disponibilidade do NATS).
3. O relay do Outbox entra em modo de retry com backoff até a conexão ser
   restabelecida, então drena o backlog **na ordem de inserção** por stream,
   preservando a ordenação exigida (RFC-0001 §5.3).
4. Consumidores deduplicam por `event.id`; nenhuma ação corretiva adicional é
   necessária do lado do consumidor.

### 5.8 FM-09 — Perda de Redis

1. Probe de saúde do cliente Redis detecta indisponibilidade.
2. `ResourceQuotaManager` e `AcbStore` alternam para **modo de fallback
   conservador**:
   - Leituras de ACB passam a ir diretamemte ao PostgreSQL (mais lentas, porém
     corretas — viola temporariamente NFR-002 de forma conhecida e alertada,
     nunca a correção do dado).
   - Decisões de cota sob incerteza de contador **DEVEM** negar (fail-closed
     também se aplica a cotas, não apenas a capability) em vez de assumir saldo
     disponível.
3. Ao Redis se recuperar, contadores são reconstruídos a partir dos limites e
   do consumo reconciliado de PostgreSQL (nunca a partir de um estado "zerado"
   que permitiria duplo consumo).

### 5.9 FM-12 — Heartbeat perdido

1. `LifecycleCoordinator` monitora o último heartbeat reportado pelo runtime
   materializado (via `008-Lifecycle`).
2. Ultrapassado `kernel.runtime.heartbeat_deadline_ms` sem sinal, dispara T-12:
   ACB `Running` → `Failed`.
3. Cota e slot de execução são liberados (compensação); evento
   `agent.lifecycle.failed` é emitido.
4. Nenhuma tentativa automática de `resume` é feita pelo Kernel — cabe ao
   chamador (ou a uma política de auto-restart em `009`/`008`, fora do escopo
   do Kernel) decidir se um novo `spawn` é apropriado.

### 5.10 FM-13 — Falha ao restaurar checkpoint

1. `LifecycleClient` reporta erro/timeout na restauração durante `resume` de
   agente `Hibernated`.
2. O ACB **permanece** em `Hibernated` (a transição para `Admitted` só ocorre
   após confirmação positiva, T-09) — nenhum estado intermediário inconsistente
   é exposto.
3. `checkpoint_ref` anterior é preservado, permitindo nova tentativa de
   `resume` (idempotente) sem perda do último ponto de checkpoint válido.
4. Falhas repetidas além de um limite configurável **DEVEM** gerar alerta de
   possível corrupção de snapshot durável, para investigação por `008-Lifecycle`.

---

## 6. Idempotência (aplicação no contexto de falha)

Toda a estratégia de recuperação acima depende da garantia de idempotência de
RFC-0001 §5.5, que o Kernel implementa via `IdempotencyStore`:

| Garantia | Mecanismo |
|----------|-----------|
| Repetição de uma syscall mutante com a mesma `Idempotency-Key` retorna o mesmo resultado. | `IdempotencyStore` persiste o par (chave, resultado) por ≥ `kernel.idempotency.retention_h` (default 48h, mínimo 24h por RFC-0001). |
| Reuso da mesma chave com payload **diferente** é um erro do cliente, não reexecutado. | Comparação de hash do payload; divergência → `AIOS-SYSCALL-0002` (409). |
| Consumidores de eventos não duplicam efeito sob entrega repetida. | Deduplicação por `event.id` (RFC-0001 §5.2), responsabilidade do consumidor, não do Kernel. |
| Retentativas de rede (timeout do cliente sem confirmação de resposta) são seguras. | Mesma chave reenviada localiza o resultado já processado no `IdempotencyStore`, mesmo que a primeira resposta tenha se perdido em trânsito. |

---

## 7. Retries e Backoff

| Operação/dependência | Máximo de tentativas | Estratégia de backoff | Jitter |
|--------------------------|--------------------------|----------------------------|--------|
| Conflito de OCC (`AcbStore`) | `kernel.acb.optimistic_retry_max` (default 5) | Backoff exponencial curto (ex.: 5ms, 10ms, 20ms, ...) internamente ao Kernel | Sim, para evitar *retry storm* sob conflito simultâneo. |
| Chamada ao PDP (`PolicyClient`) | Governado pelo circuit breaker, não por contagem fixa | Circuit breaker com meia-abertura periódica | Implícito no intervalo de sonda. |
| Chamada ao Scheduler/Lifecycle | 1 tentativa síncrona dentro do timeout da saga; retry é responsabilidade do **cliente externo** reenviando a syscall | N/A no lado do Kernel (a saga não reintenta indefinidamente — falha explícita → `Failed`) | N/A |
| Chamada a broker de recurso | Governado pelo circuit breaker por dependência | Circuit breaker + retry do **cliente chamador** com backoff exponencial e jitter, respeitando `retryAfterMs` do erro | Sim, recomendado ao cliente. |
| Relay do Outbox | Contínuo (job recorrente a cada `kernel.outbox.relay_interval_ms`) até sucesso | Retry indefinido com backoff limitado (o evento não é descartado) | Sim. |

**Regra normativa:** o Kernel **NÃO DEVE** reintentar automaticamente e de
forma indefinida uma syscall privilegiada em nome do chamador quando a falha é
de **negação** (capability/cota) — apenas falhas **transitórias de
infraestrutura** (timeout de dependência, conflito de OCC) justificam retry
interno limitado; negações são definitivas até nova decisão do PDP ou novo
saldo de cota.

---

## 8. Dead-Letter Handling

O Kernel não opera uma fila de mensagens própria (delegada ao NATS/JetStream,
`../020-Communication/`), mas define comportamento de "poison message" nos dois
pontos onde lida com falhas repetidas de processamento:

| Fonte | Condição de "poison" | Tratamento |
|-------|----------------------|--------------|
| `kernel.outbox` | Evento falha em ser publicado após um número elevado de tentativas do relay (ex.: payload corrompido, subject inválido). | Marcado para inspeção manual (flag/tabela de exceção); **NÃO É** descartado silenciosamente — alerta de alta prioridade, pois representa risco de perda de evento (NFR-007). |
| Eventos consumidos (`aios.<tenant>.scheduler.*`, `lifecycle.*`, etc.) | Handler do Kernel falha repetidamente ao processar um evento consumido (ex.: `runtime.started` referenciando um `agent_urn` inexistente). | Ver DLQ nativa do JetStream (`../020-Communication/`); o Kernel **DEVE** logar e emitir métrica de "evento não processável", nunca travar o consumo dos demais eventos do stream. |

---

## 9. RTO / RPO

| Meta | Valor | Escopo |
|------|-------|--------|
| **RTO** (Recovery Time Objective) | ≤ **15 minutos** | Tempo para o serviço Kernel voltar a atender syscalls após uma falha grave (ex.: perda de uma AZ, corrupção detectada). |
| **RPO** (Recovery Point Objective) | ≤ **5 minutos** | Perda máxima tolerável de mutações de estado de controle (ACB, Quota, Outbox) em um desastre. |

**Como as metas são atingidas:**

- **RTO**: PostgreSQL em modo HA com *streaming replication* e promoção
  automática/assistida de réplica standby; réplicas do Kernel são stateless e
  reiniciam/reconectam rapidamente a uma nova topologia de banco (`Deployment.md`
  §11, `../027-HA-DR/`).
- **RPO**: replicação síncrona ou semi-síncrona do PostgreSQL para o RPO alvo;
  o Outbox garante que nenhuma mutação confirmada seja perdida mesmo que o
  evento correspondente ainda não tenha sido publicado (§5.3) — a perda
  potencial em um desastre é limitada à janela de replicação do banco, não à
  camada de eventos.
- Simulações de recuperação (*DR drills*) **DEVEM** ser executadas
  periodicamente, incluindo o cenário de "kill -9 entre commit e publish" para
  validar empiricamente a garantia de zero perda de evento (NFR-007).

---

## 10. Degradação Graciosa

Quando uma dependência não-crítica está degradada, o Kernel **DEVERIA**
continuar atendendo o máximo de superfície possível, seguindo esta ordem de
prioridade (da mais para a menos protegida):

1. **Segurança e isolamento de tenant** — nunca degradado; sempre fail-closed.
2. **Integridade do ACB (FSM/OCC)** — nunca degradado; nenhuma transição
   inconsistente é aceita mesmo sob pressão.
3. **Enforcement de cota** — degrada para modo conservador (nega dúvidas) antes
   de arriscar overshoot além da meta (NFR-009).
4. **Latência de leitura** (`get_quota`, `GetAcb`) — pode degradar (maior
   latência via fallback a PostgreSQL) antes de recusar a leitura por completo.
5. **Emissão de eventos/auditoria** — pode atrasar (retido no Outbox) sem
   bloquear a syscall original, desde que a garantia de entrega eventual seja
   preservada.

Esta ordem reflete o princípio central do brief (`_DESIGN_BRIEF.md` §9): *"sob
indisponibilidade de dependência, o Kernel prioriza negar com segurança em vez
de permitir sem governança."*

---

## 11. Testes de Caos (referência)

A validação empírica dos comportamentos descritos acima (não apenas o desenho)
é responsabilidade de `./Testing.md`, que **DEVE** incluir, no mínimo, os
seguintes cenários de caos derivados desta FMEA:

| Cenário de caos | Modo de falha exercitado |
|---------------------|------------------------------|
| Derrubar o PDP durante carga sustentada | FM-01 |
| Derrubar o Scheduler durante rajada de `spawn` | FM-02 |
| Matar o processo Kernel entre commit e publish (fault injection) | FM-04 |
| Forçar conflitos de escrita concorrente no mesmo ACB | FM-05 |
| Derrubar um broker de recurso específico sob tráfego misto | FM-06 |
| Particionar a rede entre Kernel e NATS | FM-08 |
| Derrubar o Redis sob carga de cota concorrente | FM-09 |
| Interromper heartbeat de um runtime materializado | FM-12 |
| Simular falha de restauração de checkpoint durante `resume` | FM-13 |

---

## 12. Referências

- Modos de falha e metas de recuperação (fonte): `./_DESIGN_BRIEF.md` §9.
- Máquina de estados do ACB (transições T-01..T-13): `./_DESIGN_BRIEF.md` §4.
- Postura de segurança fail-closed: `./Security.md` §3.3, §8.
- Modelo de concorrência (OCC, token-bucket): `./Scalability.md` §4–§5.
- Idempotência e correlação (contratos centrais): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.5–§5.6.
- Topologia de implantação e probes de saúde: `./Deployment.md` §7, §11.
- Alta disponibilidade e disaster recovery (transversal): `../027-HA-DR/`.
- Auditoria imutável: `../025-Audit/`.
- Glossário: `../040-Glossary/Glossary.md` (FMEA, RTO/RPO, DLQ, Circuit Breaker, Bulkhead, Saga, Idempotency).
