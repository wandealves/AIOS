---
Documento: FailureRecovery
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (Admission control com reserva atômica), ADR-0094 (Preempção compensável), ADR-0095 (Backpressure em três níveis), ADR-0097 (Outbox transacional)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — FailureRecovery

> **Escopo.** Este documento detalha, no padrão FMEA (*Failure Mode and
> Effects Analysis*), os modos de falha do Scheduler, sua detecção, isolamento,
> estratégia de recuperação, o papel central da idempotência, a política de
> retries/backoff, o tratamento de mensagens não processáveis (DLQ) e as metas
> de RTO/RPO. Reproduz e aprofunda `_DESIGN_BRIEF.md` §9 sem contradizê-lo; não
> redefine os contratos centrais de idempotência/correlação de RFC-0001.

---

## Índice

1. Princípios de Recuperação
2. FMEA — Catálogo de Modos de Falha
3. Isolamento de Falha (Bulkhead e Circuit Breaker)
4. Idempotência como Mecanismo Central de Recuperação
5. Retries e Backoff
6. Dead-Letter Queue (DLQ)
7. RTO / RPO e Metas de Recuperação
8. Degradação Graciosa por Dependência
9. Detecção — Health Checks, Circuit Breakers e Alertas
10. Runbooks de Incidente
11. Testes de Caos e Validação Contínua
12. Referências

---

## 1. Princípios de Recuperação

O Scheduler adota quatro princípios inegociáveis de recuperação de falha,
todos derivados de `_DESIGN_BRIEF.md` §9 e das invariantes de §3/§4:

| Princípio | Descrição |
|-----------|-----------|
| **Nunca overcommit** | Sob qualquer falha, o Scheduler **DEVE** preferir rejeitar/deferir admissão a arriscar exceder `max_concurrency` (NFR-005). Overcommit é considerado pior falha que indisponibilidade parcial. |
| **Fail-safe, não fail-open** | Dependências de segurança (PDP) falham fechadas (nega); dependências de estado (Redis/PostgreSQL) falham para o modo mais conservador disponível, nunca para uma decisão otimista sem garantia. |
| **Idempotência universal em mutação** | Toda operação de mutação (`Submit`, `Cancel`, `Preempt`, `ReportRuntimeSignal`) **DEVE** ser segura sob repetição — pré-requisito para que qualquer estratégia de retry/reconciliação funcione sem efeito colateral duplicado (NFR-011). |
| **Nenhum trabalho admitido é silenciosamente perdido** | Uma tarefa que já foi `ADMITTED` **DEVE**, sob falha subsequente (lease órfão, réplica perdida, partição), ser reconciliada de volta a `QUEUED` — nunca simplesmente desaparecer do sistema sem gerar evento e sem oportunidade de nova tentativa. |

---

## 2. FMEA — Catálogo de Modos de Falha

> Tabela normativa; caso um modo de falha aqui não corresponda ao mapeado no
> `_DESIGN_BRIEF.md` §9, o brief prevalece.

| # | Modo de falha | Detecção | Efeito sem mitigação | Estratégia de recuperação | Severidade |
|---|----------------|----------|--------------------------|-------------------------------|------------|
| FM-01 | Redis indisponível (filas/cotas/leases) | Health probe / timeout de conexão | Sem garantia atômica de reserva → risco de overcommit; filas inacessíveis. | Modo conservador: novas admissões rejeitadas com `AIOS-SCHED-0003` (retriável); reservas em voo abortadas com rollback; sem overcommit (invariante preservada mesmo sob falha). | Crítica |
| FM-02 | PostgreSQL indisponível (`DecisionJournal`/`IdempotencyRecord`) | Timeout de conexão/transação | Decisões não podem ser persistidas de forma durável (perda de proveniência). | Buffer local temporário + retry com backoff; se persistir além do limiar, degrada para modo *read-only* de decisões (admissões que exigem journaling são negadas com `AIOS-SCHED-0012`, retriável); alerta imediato. | Crítica |
| FM-03 | NATS/JetStream indisponível | Timeout de ACK / lag de publicação crescente | Eventos de decisão não chegam a consumidores (025/023/024). | Outbox transacional (`DecisionJournal`) retém o evento; publicação re-tentada com backoff exponencial; consumidores idempotentes toleram atraso/duplicação subsequente (at-least-once). | Alta |
| FM-04 | Runtime Supervisor (007) não responde/heartbeat expira | TTL de heartbeat (`scheduler.capacity.heartbeat_ttl_ms`) | Tarefa fica presa em `SCHEDULED` sem confirmação de `RUNNING`; slot não é liberado. | `ReconciliationWorker` expira o lease de dispatch, transiciona `SCHEDULED→QUEUED` (re-schedule), libera o slot reservado; idempotência por `request_id` evita execução duplicada se o Runtime na verdade tiver iniciado. | Alta |
| FM-05 | Cost-Optimizer (026) indisponível | Timeout de `BudgetClient`/`CostEstimator` | Sem estimativa de custo, admissão não pode avaliar orçamento com precisão. | Usa última estimativa cacheada + margem conservadora; se não houver dado cacheado, aplica `AIOS-SCHED-0002` apenas a orçamentos no limite (degradação graciosa, não bloqueio total). | Média |
| FM-06 | Policy Engine — PDP (022) indisponível | Timeout de `PolicyClient` / circuit breaker aberto | Sem decisão de autorização, risco de admitir/preemptar sem PDP. | *Default deny*: admissão/preempção que exijam PDP são negadas (`AIOS-SCHED-0004`), preservando segurança (RFC-0001 §5.8); leituras não privilegiadas continuam. | Crítica (segurança) |
| FM-07 | Decisão duplicada (retry do cliente / entrega at-least-once) | `IdempotencyStore` detecta chave já processada | Sem controle, geraria dupla admissão/reserva. | Retorna o `response_snapshot` da primeira decisão; nenhuma reserva/admissão nova é criada (NFR-011); payload divergente para a mesma chave é rejeitado (`AIOS-SCHED-0006`). | Alta (integridade) |
| FM-08 | Overcommit por corrida entre réplicas concorrentes | Contador atômico Redis (script Lua) detecta tentativa acima do limite | Sem CAS atômico, duas réplicas poderiam reservar o mesmo slot simultaneamente. | `QuotaLedger.TryReserve` usa reserva atômica (CAS/Lua); falha de reserva por concorrência resulta em rollback automático da tentativa perdedora; invariante NFR-005 preservada por construção, não por detecção posterior. | Crítica |
| FM-09 | Starvation de tenant sob carga adversarial | Métrica de fairness por tenant abaixo de `min_share` | Um tenant de alto volume monopoliza despachos, outros tenants não progridem. | Aging força boost de prioridade até `scheduler.aging.max_boost`; garante vazão mínima (`scheduler.fairness.min_share`) em janela de observação (default 60 s), ver `Scalability.md` §11. | Média |
| FM-10 | Lease órfão (réplica/worker morre durante o dispatch) | TTL de lease de dispatch (`scheduler.lease.ttl_ms`) expira sem confirmação | Entrada de fila fica "presa" com lease ativo, invisível a novo dispatch. | `ReconciliationWorker` detecta lease expirado, libera a entrada para novo `Dequeue`; idempotência por `request_id` evita dupla execução caso o dispatch original na verdade tenha se completado. | Alta |
| FM-11 | Loop de preempção (*thrashing*) | Taxa de preempções/s por shard/tarefa acima do esperado | Tarefas alternam repetidamente entre `RUNNING`/`PREEMPTED` sem progresso líquido. | Histerese: cooldown por tarefa recém-agendada (não pode ser vítima na janela de cooldown) + anti-preempção cíclica (ADR-0094); métrica de preempção monitorada e alarmável. | Média |
| FM-12 | Partição de rede entre réplicas e Redis/PostgreSQL (split-brain parcial) | Health probe de dependência crítica falha em subconjunto de réplicas | Réplicas isoladas podem tomar decisões inconsistentes se continuarem servindo tráfego. | `readyz` retorna `503` para réplicas sem conectividade a dependências críticas — removidas do pool de roteamento (`Deployment.md` §7); nenhuma decisão de admissão é tomada sem acesso à reserva atômica (Redis). | Crítica |
| FM-13 | Rebalanceamento de shard (mudança de `N`) falha a meio caminho | Divergência entre mapa de shard esperado e observado (`ReconciliationWorker`) | Tarefas podem ficar associadas a um shard que já não tem *owner* claro. | Rebalanceamento é transacional por lote (hashing consistente, ADR-0092); em caso de falha parcial, o `ReconciliationWorker` reconcilia contra o último mapa shard→nó publicado por `027-Cluster` e reemite `capacity.changed` até convergência. | Média |
| FM-14 | Evento consumido malformado/schema divergente | Falha de deserialização/validação de `dataschema` | Consumidor trava ou descarta silenciosamente eventos válidos subsequentes. | Evento malformado é roteado à stream DLQ do JetStream (§6) após `scheduler.retry.max_attempts`; consumo dos demais eventos prossegue sem bloqueio (isolamento por mensagem). | Baixa |

---

## 3. Isolamento de Falha (Bulkhead e Circuit Breaker)

Cada dependência externa do Scheduler é isolada por um **bulkhead** dedicado
(pool de conexão/thread próprio) e, quando aplicável, um **circuit breaker**,
de modo que a degradação de uma dependência não consuma recursos
compartilhados usados por chamadas a outras dependências nem bloqueie
indefinidamente o caminho de decisão:

```
                         ┌─────────────────────────┐
                         │   AdmissionController     │
                         └───────────┬───────────────┘
              ┌────────────┬─────────┼─────────┬────────────┐
              ▼            ▼         ▼         ▼            ▼
        ┌──────────┐ ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
        │ Bulkhead  │ │ Bulkhead ││ Bulkhead ││ Bulkhead ││ Bulkhead │
        │ Redis     │ │ PDP(022) ││Budget(026)││Capacity ││ Runtime  │
        │ (quota/   │ │ +breaker ││+ cache    ││(027)     ││Supervisor│
        │  filas)   │ │          ││ fallback  ││+ TTL      ││(007)     │
        └──────────┘ └──────────┘└──────────┘└──────────┘└──────────┘
```

| Dependência | Padrão | Efeito de falha isolado |
|-------------|--------|----------------------------|
| Redis | Bulkhead de conexão dedicado | Falha aqui é a única que força modo conservador global de admissão (é a dependência de reserva atômica — não há fallback seguro para ela). |
| PDP (022) | Bulkhead + circuit breaker | Falha nega apenas operações privilegiadas; leituras não privilegiadas continuam. |
| Cost-Optimizer (026) | Bulkhead + fallback de cache | Falha degrada precisão de orçamento, não bloqueia admissão fora dos casos de limite. |
| CapacityProvider/Cluster (027) | Bulkhead + TTL de frescor | Falha zera capacidade apenas do shard afetado; demais shards continuam operando normalmente. |
| Runtime Supervisor (007) | Bulkhead dedicado | Falha afeta apenas confirmação de dispatch/heartbeat; admissão/enfileiramento continuam. |

Circuit breakers **DEVEM** abrir com limiar configurável (análogo ao padrão
já adotado no Kernel, `kernel.broker.circuit_breaker_threshold`) — ao abrir,
novas chamadas à dependência degradada são recusadas imediatamente
(fail-fast), sem tentar rede, evitando amplificar carga sobre uma dependência
já comprometida.

---

## 4. Idempotência como Mecanismo Central de Recuperação

A idempotência (RFC-0001 §5.5) não é apenas um requisito de API — é o que
torna **toda** estratégia de recuperação abaixo segura de executar
repetidamente sem efeito colateral:

| Operação | Chave de idempotência | Comportamento sob repetição |
|----------|---------------------------|----------------------------------|
| `Submit` | `Idempotency-Key` = `request_id` | Retorna `response_snapshot` da primeira decisão; nenhuma nova reserva de slot (FM-07). |
| `Cancel` | `Idempotency-Key` | Repetir sobre tarefa já cancelada é no-op idempotente (mesmo resultado). |
| `Preempt` | `Idempotency-Key` | Repetir sobre vítima já preemptada retorna o mesmo `PreemptResult`, sem nova suspensão. |
| `ReportRuntimeSignal` | ID do sinal (`signal_id`) | Sinal duplicado (heartbeat/started/completed repetido) é deduplicado; não reabre uma tarefa já `COMPLETED`/`FAILED`. |
| Reenfileiramento por `ReconciliationWorker` | `request_id` original preservado | Requeue nunca cria uma nova `SchedulingRequest`/admissão — apenas retoma o ciclo de vida da mesma requisição (INV-03: no máximo uma `SCHEDULED` ativa simultânea). |
| Consumo de eventos (Cost-Optimizer/Cluster/Policy) | `event.id` (CloudEvents) | Consumidor deduplica por `event.id`; reprocessamento do mesmo evento não corrompe `CapacityProvider`/cache de orçamento. |

**Regra normativa central:** nenhuma estratégia de recuperação descrita neste
documento **DEVE** ser implementada de forma que viole a idempotência acima —
retries, reconciliação e failover de réplica confiam nesta propriedade como
pré-condição de segurança.

---

## 5. Retries e Backoff

| Contexto | Política | Parâmetro |
|----------|-----------|-----------|
| Cliente externo repetindo `Submit`/`Cancel`/`Preempt` após erro retriável | Backoff exponencial com jitter, respeitando `retryAfterMs` quando presente (ex.: `AIOS-SCHED-0003`). | Recomendação ao cliente; não configurável no servidor além do `retryAfterMs` sinalizado. |
| Retry interno de tarefa `FAILED` retriável (transição `FAILED→QUEUED`) | Backoff exponencial com jitter, até `scheduler.retry.max_attempts` (default 3). | `scheduler.retry.backoff_base_ms` (default 500 ms) como base do exponencial. |
| Retry de publicação de evento (outbox → JetStream) | Backoff exponencial com jitter; outbox retém o evento até sucesso ou até ser roteado à DLQ. | Configuração de relay de outbox (implementação); nunca descarta silenciosamente um evento pendente de publicação. |
| Retry de consumo de evento externo malformado | Até `scheduler.retry.max_attempts`; após esgotado, roteado à DLQ (§6). | Isolado por mensagem — não bloqueia o consumo de mensagens subsequentes válidas. |
| Retry de chamada a dependência com circuit breaker aberto | Nenhum retry imediato — falha rápida até o breaker permitir nova tentativa (half-open). | Evita tempestade de retries sobre dependência já degradada. |

**Regra normativa:** todo retry, interno ou externo, **DEVE** operar sobre uma
operação idempotente (§4) — nunca sobre uma mutação que possa duplicar efeito
sob repetição.

---

## 6. Dead-Letter Queue (DLQ)

- Eventos consumidos que falham em ser processados após
  `scheduler.retry.max_attempts` **DEVERIAM** ser roteados para uma stream DLQ
  dedicada do JetStream (ex.: `aios.<tenant>.scheduler.dlq.<dominio-origem>`),
  preservando o envelope CloudEvents original e anexando metadados de falha
  (`attempt_count`, `last_error`, `first_failed_at`).
- A DLQ **DEVE** ser inspecionável por operadores (via ferramenta
  administrativa/CLI) e **DEVE** suportar *replay* manual após correção da
  causa raiz (ex.: schema corrigido, dependência restaurada).
- Mensagens na DLQ **NÃO DEVEM** bloquear o consumo da stream principal — o
  isolamento é por mensagem individual, não por stream inteira (FM-14).
- Volume crescente na DLQ **DEVE** gerar alerta — é sinal de uma classe de
  falha sistemática (ex.: mudança de schema não coordenada em um módulo
  produtor) que requer investigação, não apenas replay mecânico.

---

## 7. RTO / RPO e Metas de Recuperação

| Métrica | Meta | Escopo |
|---------|------|--------|
| **RPO** (Recovery Point Objective) | ≤ 5 min | Decisões críticas (`SchedulingDecision`) são persistidas via outbox durável (JetStream + PostgreSQL) antes de serem consideradas "confirmadas" ao chamador — perda de dado limitada à janela entre commit local e replicação/backup, nunca à perda de uma decisão já confirmada (NFR-007). |
| **RTO** (Recovery Time Objective) | ≤ 15 min | Tempo entre falha de uma réplica (ou de uma dependência crítica) e retomada plena de capacidade de atendimento, sem perda de trabalho já admitido (NFR-008). |

### 7.1 Cenários de recuperação e tempo esperado

| Cenário | Ação de recuperação | Tempo esperado |
|---------|-------------------------|--------------------|
| Perda de 1 réplica do Scheduler | Orquestrador de containers reinicia/substitui a réplica; leases de shard órfãos expiram (`scheduler.lease.ttl_ms`, default 10 s) e são assumidos por réplicas remanescentes. | Segundos a poucos minutos (bem abaixo do RTO de 15 min). |
| Falha do Redis primário (cluster gerenciado) | Failover para réplica Redis (responsabilidade de `027-Cluster`); Scheduler opera em modo conservador (`AIOS-SCHED-0003`) durante a janela de failover. | Minutos, dependente do SLA de failover de `027-Cluster`; dentro do RTO de 15 min. |
| Falha do PostgreSQL primário | Failover para *standby* via *streaming replication* (responsabilidade de `027-Cluster`); Scheduler degrada para modo *read-only* de decisões durante a janela. | Minutos; dentro do RTO de 15 min; RPO ≤ 5 min garantido por replicação síncrona/semi-síncrona do PostgreSQL. |
| Partição de rede entre Scheduler e NATS | Outbox retém eventos localmente (PostgreSQL); republicação automática ao restabelecer conectividade. | Sem perda de decisão; atraso de entrega de evento limitado à duração da partição. |
| Perda total do datacenter/AZ do Scheduler | Réplicas remanescentes em outras AZs (≥ 2 AZ, `Deployment.md` §5) assumem 100% do tráfego; nenhuma tarefa em `QUEUED` é perdida (estado em Redis/PostgreSQL replicados entre AZs, responsabilidade de `027-Cluster`). | Dentro do RTO de 15 min, validado por *drill* de failover (§11). |

### 7.2 Degradação graciosa como alternativa a downtime total

Em nenhum cenário acima o Scheduler **DEVE** simplesmente parar de responder
sem sinalização — a filosofia de recuperação prioriza **degradação graciosa**
(recusar novas admissões com erro retriável e `Retry-After`) sobre
indisponibilidade completa, sempre que a dependência crítica específica
permitir esse modo (ver §8).

---

## 8. Degradação Graciosa por Dependência

| Dependência indisponível | O que continua funcionando | O que degrada/para |
|---------------------------|--------------------------------|---------------------------|
| Redis | Leitura de proveniência já persistida em PostgreSQL (`GetDecision`). | Toda admissão nova (`AIOS-SCHED-0003`); dispatch de novas tarefas. |
| PostgreSQL | Admissão/enfileiramento em Redis pode continuar por curto prazo (buffer). | Confirmação durável da decisão; publicação de evento (outbox bloqueado); eventualmente, se persistir, novas admissões também degradam para evitar acumular decisões não duráveis. |
| NATS/JetStream | Todo o caminho de decisão (admissão, dispatch, preempção). | Apenas a notificação a consumidores externos (025/023/024) é adiada — não bloqueante para o caminho crítico. |
| PDP (022) | Leituras não privilegiadas (`GetDecision`, `StreamQueueState`). | Toda admissão/preempção nova (`AIOS-SCHED-0004`, fail-closed). |
| Cost-Optimizer (026) | Admissão com estimativa cacheada + margem conservadora. | Apenas admissões cujo orçamento esteja no limite exato podem ser rejeitadas por excesso de cautela (`AIOS-SCHED-0002`). |
| Cluster/Capacity (027) | Admissão em shards com `CapacitySnapshot` ainda fresco. | Placement em shards com snapshot expirado (`AIOS-SCHED-0010`, capacidade tratada como zero). |
| Runtime Supervisor (007) | Admissão e enfileiramento continuam normalmente. | Confirmação de `SCHEDULED→RUNNING`; tarefas acumulam em `SCHEDULED` até TTL de lease, então retornam a `QUEUED` (FM-04). |

---

## 9. Detecção — Health Checks, Circuit Breakers e Alertas

| Mecanismo | O que detecta | Ação disparada |
|-----------|-------------------|--------------------|
| `readyz` (por réplica) | Conectividade com Redis/PostgreSQL/NATS | Remoção da réplica do pool de roteamento (`Deployment.md` §7). |
| Circuit breaker por dependência (`PolicyClient`, `BudgetClient`, `CapacityProvider`, `RuntimeSupervisorClient`) | Taxa de erro/timeout acima de limiar | Abertura do circuito → fail-fast local, sem nova tentativa de rede até half-open. |
| TTL de lease (dispatch e ownership de shard) | Ausência de renovação (worker/réplica morta) | Liberação do lease para nova aquisição (`ReconciliationWorker`/outra réplica). |
| Métrica de profundidade de fila por shard | Crescimento sem dispatch correspondente | Alerta operacional; possível indicativo de FM-04/FM-10/FM-12. |
| Métrica de latência de decisão (p99) | Degradação de dependência no caminho quente | Alerta; correlacionado com dependência específica via trace (OTel). |
| Métrica de taxa de rejeição por `reason_code` | Aumento anômalo de `AIOS-SCHED-000X` específico | Alerta direcionado à dependência/causa correspondente (cota vs. orçamento vs. backpressure vs. política). |
| Volume de mensagens na DLQ | Falha sistemática de consumo de evento | Alerta de investigação de causa raiz (schema, produtor). |

Detalhamento completo de regras de alerta e dashboards: `Monitoring.md`
(documento de outro batch).

---

## 10. Runbooks de Incidente

| Cenário | Passos de resposta imediata |
|---------|---------------------------------|
| Pico de `AIOS-SCHED-0003` (backpressure/reject) | 1) Confirmar se é saturação real (capacidade insuficiente) ou falha de `CapacityProvider` (snapshot expirado); 2) Verificar `Deployment.md` §9 para gatilho de escalonamento; 3) Comunicar produtores para respeitar `retryAfterMs`. |
| Pico de `AIOS-SCHED-0004` (negação PDP) | 1) Verificar saúde do PDP (022); 2) Confirmar se circuit breaker do `PolicyClient` está aberto; 3) Se PDP saudável mas negando legitimamente, investigar mudança de política recente (não é um bug do Scheduler). |
| Fila crescendo sem dispatch em um shard específico | 1) Verificar qual réplica é *owner* do shard (lease); 2) Verificar saúde daquela réplica; 3) Se réplica saudável mas travada, verificar dependência bloqueante (`RuntimeSupervisorClient`); 4) Considerar reinício controlado da réplica *owner* (lease será reassumido automaticamente). |
| Divergência entre filas e execução real (tarefas "fantasmas") | 1) Acionar execução manual/antecipada do `ReconciliationWorker` se disponível como operação administrativa; 2) Verificar métrica de leases órfãos expirados; 3) Validar que nenhuma tarefa em `SCHEDULED` ultrapassou `scheduler.lease.ttl_ms` sem ação corretiva. |
| Volume anômalo na DLQ | 1) Inspecionar amostra de mensagens (schema, origem); 2) Identificar módulo produtor e coordenar correção; 3) Após correção, executar replay controlado da DLQ (não em lote total sem validação). |
| Suspeita de overcommit (slots reservados > `max_concurrency` observado) | 1) Tratar como incidente de severidade crítica (viola NFR-005 por definição); 2) Auditar logs de `QuotaLedger.TryReserve` para identificar janela de corrida; 3) Reconciliar contadores Redis contra `SchedulingDecision` (PostgreSQL) como fonte da verdade; 4) Abrir post-mortem — este cenário não deveria ser possível sob operação normal do script Lua atômico. |

---

## 11. Testes de Caos e Validação Contínua

A estratégia de recuperação descrita neste documento **DEVE** ser validada
continuamente por testes de caos (detalhados em `Testing.md`, documento de
outro batch), cobrindo no mínimo:

- Derrubar Redis durante carga de admissão ativa → validar zero overcommit e
  transição correta para `AIOS-SCHED-0003`.
- Derrubar PostgreSQL durante decisões em voo → validar que nenhum evento é
  publicado sem a decisão correspondente persistida (atomicidade do outbox).
- Derrubar PDP (022) durante admissão/preempção → validar *fail-closed*
  (nenhuma admissão/preempção silenciosa).
- Matar a réplica *owner* de um shard durante dispatch ativo → validar
  reassociação de lease e ausência de execução duplicada.
- Introduzir latência artificial em `BudgetClient`/`CapacityProvider` →
  validar fallback de cache e ausência de bloqueio do caminho quente além do
  timeout configurado.
- Simular mudança de `N` (rebalanceamento) sob carga → validar ausência de
  overcommit/perda de tarefa durante a transição.
- Injetar evento malformado em subject consumido → validar isolamento (DLQ)
  sem impacto em mensagens subsequentes válidas.

Resultados desses testes **DEVEM** ser rastreados contra as metas de RTO/RPO
(§7) e as invariantes de correção (NFR-005, NFR-011) como critério de
aceite de release (*gate* de qualidade, ver `Testing.md`).

---

## 12. Referências

- Fonte de verdade dos modos de falha: `_DESIGN_BRIEF.md` §9.
- Arquitetura e componentes envolvidos: `./Architecture.md`, `./ClassDiagrams.md`.
- Máquina de estados (transições sob falha, retry, preempção): `./StateMachine.md`.
- Topologia de implantação e health/readiness: `./Deployment.md` §7.
- Modelo de escala e ownership de shard (contexto de lease/reconciliação): `./Scalability.md` §3, §9.
- Postura de segurança (fail-closed do PDP, bulkhead): `./Security.md` §3.3, §8.
- Requisitos não-funcionais associados: `./NonFunctionalRequirements.md` (NFR-005, NFR-007, NFR-008, NFR-011).
- Catálogo de erros: `_DESIGN_BRIEF.md` §5.4 (`AIOS-SCHED-0001`–`0012`).
- Contratos centrais de idempotência/correlação: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.5–§5.6.
- Glossário: `../040-Glossary/Glossary.md` (termos: Idempotency, DLQ, RTO/RPO, FMEA, Circuit Breaker, Bulkhead, Exactly-once/At-least-once).
