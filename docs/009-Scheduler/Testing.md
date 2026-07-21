---
Documento: Testing
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (admission control), ADR-0091 (score multiobjetivo), ADR-0092 (sharding), ADR-0093 (filas ZSET/aging), ADR-0094 (preempção compensável), ADR-0095 (backpressure), ADR-0096 (`ISchedulingPolicy`), ADR-0097 (outbox transacional)
RFCs relacionados: RFC-0001 (baseline); RFC-0090 (Scheduling Decision Contract), RFC-0091 (Learned Scheduling Policy)
Depende de: 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Testing

> Este documento define a **estratégia de testes** do Scheduler: pirâmide de
> testes, critérios de cobertura, fixtures/ambientes, e os cenários
> obrigatórios derivados diretamente da máquina de estados (`StateMachine.md`),
> da API (`API.md`), do modelo de admissão/prioridade/placement/preempção
> (`_DESIGN_BRIEF.md` §1–§4) e dos modos de falha (`_DESIGN_BRIEF.md` §9).
> Comportamento normativo (DEVE/NÃO DEVE) sem teste associado **NÃO DEVE** ser
> considerado coberto apenas por inspeção manual — o gate de qualidade de CI é
> definido na §7.

## 1. Objetivo e Princípios

1. Todo requisito funcional/não-funcional (`FunctionalRequirements.md`,
   `NonFunctionalRequirements.md`) DEVE ter pelo menos um teste automatizado
   que o verifique — rastreabilidade requisito→teste.
2. A pirâmide de testes prioriza a base (unit > integration > contract > e2e >
   chaos > load), mas o Scheduler, por ser um **broker de decisão sob pressão
   de latência com estado externalizado concorrente** (Redis ZSET, contadores
   atômicos, leases), exige investimento acima da média em testes de
   **concorrência** (reserva de slot, dispatch, ownership de shard) e de
   **contrato** (estabilidade de `ISchedulingPolicy` v1→v2, FR-013).
3. Testes de falha (caminho infeliz) são tão obrigatórios quanto testes de
   caminho feliz — cada transição da `StateMachine.md` §4 tem um teste de
   sucesso e, onde aplicável, um teste da guarda que a bloqueia.
4. Todo teste de mutação (admissão, preempção, cancelamento) DEVE verificar
   simultaneamente: (a) o efeito em `PriorityQueueStore`/`QuotaLedger`
   (Redis); (b) o registro em `DecisionJournal` (PostgreSQL); (c) o evento
   emitido via `EventPublisher` (NATS/JetStream); (d) o código de erro RFC 7807,
   quando aplicável.
5. Testes de troca de política (`heuristic`↔`rl`, ADR-0096) DEVEM comprovar
   que a superfície de API/eventos permanece idêntica (NFR-013) — este é um
   requisito de teste de contrato, não apenas de unidade.

## 2. Pirâmide de Testes

```
                         ┌─────────────┐
                         │    Load     │  poucos, alto custo, ambiente dedicado
                         │  (k6/ghz)   │  → ./Benchmark.md
                        ┌┴─────────────┴┐
                        │     Chaos      │  falhas injetadas (rede, deps, crash)
                        │ (Litmus/Toxi.) │
                       ┌┴────────────────┴┐
                       │       E2E         │  fluxo completo via Kernel(006)/Gateway
                       │  (poucos cenários) │
                      ┌┴────────────────────┴┐
                      │      Contract         │  OpenAPI/proto ↔ implementação;
                      │  (Pact/Buf breaking)  │  ISchedulingPolicy v1↔v2
                     ┌┴───────────────────────┴┐
                     │       Integration         │  Scheduler + PostgreSQL + Redis +
                     │   (Testcontainers)         │  NATS reais (containers efêmeros)
                    ┌┴────────────────────────────┴┐
                    │            Unit                │  AdmissionController, Priority-
                    │     (xUnit + FluentAssertions)  │  CostEvaluator, PlacementEngine,
                    └─────────────────────────────────┘  PreemptionManager (mock deps)
```

| Camada | Ferramentas (.NET 10) | Dependências reais? | Meta de execução |
|--------|-------------------------|-----------------------|---------------------|
| Unit | xUnit, FluentAssertions, NSubstitute (mocks) | Não (tudo mockado/in-memory) | < 5 min no pipeline de PR |
| Integration | xUnit + Testcontainers (PostgreSQL 16, Redis 7, NATS/JetStream) | Sim, containers efêmeros | < 15 min |
| Contract | Pact (consumer-driven, contra `006`/`022`/`026`/`027`/`007`), `buf breaking` (proto `aios.scheduler.v1`), Spectral (OpenAPI lint), suíte dupla `heuristic`/`rl` de `ISchedulingPolicy` | Simulado (mocks de contrato) | < 10 min |
| E2E | Cliente gRPC/HTTP via Kernel (006) simulado ou real, ambiente `staging` isolado | Sim, stack completa | Execução noturna + pré-release |
| Chaos | Toxiproxy (latência/partição de rede), scripts de `kill -9` no `EventPublisher`/`DispatchLoop` | Sim, ambiente dedicado | Execução semanal |
| Load | k6 (REST) / ghz ou NBomber (gRPC) | Sim, ambiente de benchmark | Execução por release, ver `./Benchmark.md` |

## 3. Cobertura-Alvo e Critérios de Qualidade de Gate

| Métrica de qualidade | Meta | Gate de CI |
|------------------------|------|------------|
| Cobertura de linha (unit + integration) | ≥ 85% no core (`SchedulingGateway`, `AdmissionController`, `PriorityCostEvaluator`, `PlacementEngine`, `PreemptionManager`, `PriorityQueueStore`, `DispatchLoop`, `QuotaLedger`) | Bloqueia merge se abaixo |
| Cobertura de branch da máquina de estados (`StateMachine.md` §4) | 100% das transições exercitadas (sucesso e guarda violada, quando aplicável) | Bloqueia merge |
| Cobertura de códigos de erro (`AIOS-SCHED-0001`..`0012`) | 100% dos códigos do catálogo (`API.md` §5) têm teste que os produz | Bloqueia merge |
| Contract tests vs. dependências (`006`,`022`,`026`,`027`,`007`) | 100% dos clients (`PolicyClient`, `BudgetClient`, `CapacityProvider`, `RuntimeSupervisorClient`) com contrato verificado | Bloqueia merge se contrato quebrado |
| Contract tests de `ISchedulingPolicy` (v1/v2) | 100% de paridade de schema de request/response/evento entre `HeuristicPolicy` e `LearnedPolicy` | Bloqueia merge (NFR-013) |
| Mutação (mutation testing, Stryker.NET) | Mutation score ≥ 70% nos componentes core | Relatório obrigatório, não bloqueante na v0.1 (bloqueante a partir de `Stable`) |
| Testes de idempotência | 100% das operações mutantes (`Submit`,`Cancel`,`Preempt`,`ReportRuntimeSignal`) com teste de replay de `Idempotency-Key`/`signal_id` | Bloqueia merge |
| Testes de concorrência (reserva atômica de slot) | Cenário de `Submit` concorrente no limite exato de `max_concurrency` com verificação de 0 overcommit | Bloqueia merge |
| Testes de fairness | Cenário multi-tenant adversarial validando `min_share` (NFR-006) | Bloqueia merge |
| Vulnerabilidades (SCA) | 0 críticas/altas não mitigadas (`dotnet list package --vulnerable`) | Bloqueia merge |

## 4. Cenários Obrigatórios por Categoria

### 4.1 Unit — Admissão (`SchedulingGateway`, `AdmissionController`)

| Cenário | Verifica |
|---------|----------|
| `Submit` com cota, orçamento, capacidade e PDP ok → `RECEIVED → ADMITTED` | FR-001, transição de admissão feliz |
| `Submit` com cota de concorrência esgotada → `REJECTED`, `AIOS-SCHED-0001` | FR-001, guarda de cota |
| `Submit` com orçamento insuficiente → `REJECTED`, `AIOS-SCHED-0002` | FR-001, guarda de orçamento |
| `Submit` sob backpressure em nível `reject` → `REJECTED`, `AIOS-SCHED-0003`, `retryAfterMs` presente | FR-005, guarda de backpressure |
| `Submit` com PDP negando → `REJECTED`, `AIOS-SCHED-0004` | FR-001/FR-020, *default deny* |
| `Submit` com `SchedulingRequest` malformada (URN inválida) → `AIOS-SCHED-0005` | FR-001, validação de envelope |
| `X-AIOS-Tenant` divergente do claim autenticado → `AIOS-SCHED-0004` | FR-019, isolamento multi-tenant |
| `deadline_at` já vencido na submissão → nasce `EXPIRED`, `AIOS-SCHED-0008` | FR-012, guarda de deadline |
| Admissão feliz reserva exatamente um slot (nunca zero, nunca dois) | invariante SM-02, NFR-005 |

### 4.2 Unit — Prioridade/Custo (`PriorityCostEvaluator`, `FeatureExtractor`, `CostEstimator`, `ISchedulingPolicy`)

| Cenário | Verifica |
|---------|----------|
| Cálculo de `C = α·L + β·$ + γ·E` com pesos normalizados (`alpha+beta+gamma≈1`) | FR-002 |
| Pesos não normalizados são normalizados na leitura (`_DESIGN_BRIEF.md` §8) | FR-002 |
| `quality_floor` inatingível sob restrições correntes → `REJECTED`, `AIOS-SCHED-0007` | FR-002 |
| Prioridade efetiva é monotônica em relação a `C` (menor `C` → maior urgência) | FR-002 |
| Duas tarefas com deadlines diferentes (EDF habilitado) são ordenadas por deadline mais próximo | FR-012 |
| Troca de `scheduler.policy.engine` de `heuristic` para `rl` não altera o schema de `SchedulingDecision` | FR-013, NFR-013 |
| `feature_vector` persistido é determinístico e reproduzível a partir dos mesmos insumos | FR-010, auditabilidade |

### 4.3 Unit — Placement (`PlacementEngine`, `ShardRouter`)

| Cenário | Verifica |
|---------|----------|
| `shard = hash(tenant_id, agent_id) mod N` é determinístico (mesma entrada → mesmo shard sempre) | FR-003 |
| Nenhum shard/nó com capacidade → `REJECTED`, `AIOS-SCHED-0010` | FR-003 |
| Anti-afinidade respeitada quando há alternativa com capacidade | FR-003 |
| Anti-afinidade ignorada (fallback logado) quando não há alternativa | FR-003, comportamento documentado |
| Mudança de `N` (rebalanceamento) remapeia fração próxima de `1/N` das chaves | FR-023, NFR-014, ADR-0092 |

### 4.4 Unit — Preempção (`PreemptionManager`)

| Cenário | Verifica |
|---------|----------|
| Preempção só ocorre após PDP retornar `allow` | FR-004 |
| Vítima selecionada é sempre de menor prioridade efetiva que o iniciador | FR-004 |
| Evento `preempted` emitido com `{taskUrn, byTaskUrn, gracePeriodMs}` antes da suspensão confirmada | FR-004 |
| Vítima recém-agendada (dentro do cooldown) NÃO é escolhida (anti-*thrashing*) | invariante SM-06, ADR-0094 |
| Overhead de decisão de preempção ≤ 50 ms em cenário sintético | NFR-009 |
| `PREEMPTED → QUEUED` preserva aging acumulado (posição relativa) | invariante SM-05, FR-009 |

### 4.5 Unit — Filas, Dispatch e Backpressure (`PriorityQueueStore`, `DispatchLoop`, `BackpressureController`)

| Cenário | Verifica |
|---------|----------|
| `enqueue` insere na ZSET `sched:{tenant}:{class}:{shard}` com score = prioridade efetiva | FR-006 |
| `DispatchLoop` sempre retira o membro de menor score elegível (capacidade disponível) | FR-006 |
| Tarefa nunca aparece em mais de uma ZSET simultaneamente | invariante de unicidade, FR-006 |
| Pressão ≥ `reject_threshold` (0.95) → novas admissões negadas com `AIOS-SCHED-0003` | FR-005 |
| Pressão entre `defer_threshold` (0.80) e `reject_threshold` → tarefa aceita, potencialmente `DEFERRED` | FR-005 |
| Aging aplicado incrementa score até `scheduler.aging.max_boost`, nunca além | FR-009 |
| Fairness: tenant minoritário sob carga adversarial de tenant dominante ainda recebe ≥ `min_share` na janela | FR-009, NFR-006 |

### 4.6 Unit — Cotas, Idempotência e Outbox (`QuotaLedger`, `IdempotencyStore`, `DecisionJournal`, `EventPublisher`)

| Cenário | Verifica |
|---------|----------|
| Reserva atômica de slot dentro do limite → sucesso; no limite exato → sucesso; acima → rejeição | FR-001, NFR-005 |
| N reservas concorrentes no mesmo `(tenant, service_class)` não ultrapassam `max_concurrency` | NFR-005 |
| Repetição da mesma `Idempotency-Key` e mesmo payload retorna resultado idêntico, sem nova reserva | FR-008, NFR-011 |
| Mesma `Idempotency-Key`, payload divergente → `AIOS-SCHED-0006` | FR-008 |
| Toda `SchedulingDecision` persistida grava exatamente um evento no outbox na mesma transação | invariante SM-04, ADR-0097 |
| Relay publica eventos pendentes e marca `published=true` | comportamento do `EventPublisher` |
| Consumidor duplicado (mesmo `event.id`) processado apenas uma vez (dedupe) | RFC-0001 §5.5 |

### 4.7 Integration (Testcontainers: PostgreSQL, Redis, NATS/JetStream)

| Cenário | Verifica |
|---------|----------|
| Ciclo completo `RECEIVED→ADMITTED→QUEUED→SCHEDULED→RUNNING→COMPLETED` persiste corretamente em PostgreSQL e Redis | máquina de estados completa persistida |
| Crash simulado do processo Scheduler entre gravação da decisão e publicação do evento (mata o processo antes do relay) | NFR-007: outbox reenvia após restart, 0 perda |
| RLS por `tenant_id`: query com `tenant_id` de outro tenant não retorna linhas | isolamento multi-tenant (`Database.md`) |
| Sharding: distribuição de tarefas entre shards conforme `hash(tenant,agent) mod N` | ADR-0092 |
| Lease de dispatch expirado é detectado e a tarefa é re-enfileirada pelo `ReconciliationWorker`, sem duplicar execução | FR-011, FR-014 |
| `FAILED → QUEUED` (retry) respeita `max_attempts` e `backoff_base_ms` | FR-011, `StateMachine.md` §9.4 |

### 4.8 Contract Tests

| Par (consumidor→provedor) | Ferramenta | Verifica |
|----------------------------|------------|----------|
| Scheduler → `022-Policy` (`PolicyClient`) | Pact | Schema de `DecisionRequest`/`DecisionResponse` de admissão/preempção estável |
| Scheduler → `026-Cost-Optimizer` (`BudgetClient`) | Pact | Schema de orçamento/estimativa de custo estável |
| Scheduler → `027-Cluster` (`CapacityProvider`) | Pact | Schema de inventário de capacidade/topologia estável |
| Scheduler → `007-Agent-Runtime` (`RuntimeSupervisorClient`) | Pact | Schema de `SchedulingDecision`/`RuntimeSignal` estável |
| Kernel (`006`) → Scheduler (gRPC `Submit`) | Pact | Schema de `SchedulingRequest`/`SchedulingDecision` estável |
| Cliente externo → Scheduler (REST) | Spectral (lint OpenAPI) + Dredd/Schemathesis | Conformidade do `API.md` (OpenAPI) |
| Cliente interno → Scheduler (gRPC) | `buf breaking` | Nenhuma mudança incompatível no pacote `aios.scheduler.v1` sem bump major |
| `HeuristicPolicy` (v1) ↔ `LearnedPolicy` (v2) | Suíte de contrato interna (xUnit parametrizado) | Ambas implementações de `ISchedulingPolicy` produzem `SchedulingDecision` com o mesmo schema (NFR-013) |

### 4.9 E2E (via Kernel/Gateway, ambiente `staging`)

| Cenário | Verifica |
|---------|----------|
| Kernel solicita slot (`spawn`) → Scheduler admite, prioriza, posiciona e despacha → Runtime Supervisor confirma `RUNNING` → conclusão | fluxo completo do brief §0 |
| Tarefa interativa de alta prioridade preempta tarefa batch em execução, que retoma após liberação | FR-004, T de preempção/resume |
| Requisição repetida com mesma `Idempotency-Key` via Kernel produz efeito único (nenhum novo slot/evento) | FR-008 |
| Cancelamento de tarefa em `QUEUED` via `Cancel` libera slot e emite `rejected` (`reason=cancelled`) | FR-015 |
| Atualização de política via `PUT /v1/scheduler/policy` reflete em novas decisões em ≤ 5s | FR-018, NFR-017 |

### 4.10 Chaos Engineering

| Experimento | Falha injetada | Resultado esperado |
|-------------|-------------------|-----------------------|
| Latência de rede para `022-Policy` | Toxiproxy adiciona 2s de latência | Admissão degrada para *default deny* sem travar outras requisições (circuit breaker + bulkhead) |
| Partição do NATS/JetStream | Bloqueio de porta do broker | Outbox acumula (`aios_scheduler_outbox_pending` cresce) sem perder mensagens; drena ao restaurar |
| `kill -9` no processo Scheduler entre gravação e publicação | Sinal ao processo em ponto aleatório | 0 evento perdido após restart (NFR-007) |
| Queda do Redis (filas/cotas quentes) | Container Redis parado | Fallback conservador: nega novas admissões (`AIOS-SCHED-0003`) em vez de arriscar overcommit |
| `026-Cost-Optimizer` indisponível durante pico de submissões | Mock retorna 503 | `CostEstimator` usa última estimativa cacheada com margem conservadora; degradação graciosa sem travar o caminho quente |
| Mudança de `scheduler.shards.count` sob carga | Reconfiguração de `N` em produção simulada | Rebalanceamento com hashing consistente; 0 downtime de admissão (NFR-014) |
| Preempção sob contenção extrema (múltiplos tenants competindo) | Carga sintética adversarial | Sem *thrashing* sustentado (cooldown/histerese contêm o ciclo, ADR-0094) |

## 5. Fixtures e Dados de Teste

| Fixture | Descrição |
|---------|-----------|
| `SchedulingRequestFixtureFactory` | Gera `SchedulingRequest` sintéticas com combinações de `service_class`, `deadline_at`, `quality_floor`, `affinity`, para testes de unidade/integração. |
| `TenantFixture` | Conjunto de tenants sintéticos (`acme`, `globex`, `initech`) com cotas/orçamentos distintos, para testes de isolamento, fairness e enforcement. |
| `PolicyStub` | Dublê do PDP (`022-Policy`) configurável para retornar `allow`/`deny`/timeout determinístico. |
| `BudgetStub` | Dublê do `Cost-Optimizer` (`026`) com orçamento/estimativa configuráveis, incluindo cenário de indisponibilidade. |
| `CapacityStub` | Dublê do `Cluster` (`027`)/`RuntimeSupervisorClient` (`007`) com inventário de capacidade e heartbeats configuráveis, incluindo staleness simulada. |
| `IdempotencyKeyGenerator` | Gera ULIDs válidos para `Idempotency-Key`, incluindo casos de reuso intencional (teste de conflito). |
| Golden files de eventos | Payloads CloudEvents de referência (um por subject do brief §6.1/§6.2) usados em testes de *snapshot* de serialização, ver `Events.md`. |
| `LoadGeneratorSeeds` | Seeds determinísticos para geração de mix de tenants/agentes usados em testes de fairness e chaos de rebalanceamento. |

## 6. Ambientes de Teste

| Ambiente | Uso | Infraestrutura |
|----------|-----|-----------------|
| `local/dev` | Unit + integration no laptop do desenvolvedor | Testcontainers efêmeros (Docker) |
| `ci` | Todo PR: unit, integration, contract | Runners efêmeros, containers descartáveis |
| `staging` | E2E, chaos semanal, benchmark de release | Stack completa espelhando `prod`, dados sintéticos |
| `prod` (somente leitura/observação) | Nenhum teste ativo executa aqui; apenas monitoramento contínuo (`Monitoring.md`) valida comportamento real | — |

## 7. Gate de Qualidade do Pipeline de CI

```
PR aberto
   │
   ▼
[1] Lint + build (dotnet format, analyzers) ──── falha → bloqueia
   │
   ▼
[2] Unit tests + cobertura ≥ 85% (core) ──────── falha → bloqueia
   │
   ▼
[3] Integration tests (Testcontainers) ───────── falha → bloqueia
   │
   ▼
[4] Contract tests (Pact broker, buf breaking, suíte heuristic/rl) ─ falha → bloqueia
   │
   ▼
[5] SCA / vulnerabilidades críticas/altas ─────── falha → bloqueia
   │
   ▼
[6] Mutation testing (relatório, não bloqueante v0.1)
   │
   ▼
merge liberado → pipeline noturno roda E2E + chaos semanal + benchmark por release
```

## 8. Referências

- Máquina de estados fonte: `./StateMachine.md`, `_DESIGN_BRIEF.md` §4
- Superfície de API e catálogo de erros: `./API.md`
- Requisitos rastreados: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Catálogo de eventos usado nos testes de snapshot: `./Events.md`, `_DESIGN_BRIEF.md` §6
- Metodologia de carga (complementar): `./Benchmark.md`
- Observabilidade usada para verificar resultados de teste: `./Monitoring.md`, `./Metrics.md`
- Modos de falha (base dos experimentos de chaos): `_DESIGN_BRIEF.md` §9
