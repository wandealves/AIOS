---
Documento: Testing
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (FSM canônica), ADR-0081 (Hibernação), ADR-0082 (Codec de checkpoint), ADR-0083 (WarmPool), ADR-0084 (Lease/fencing), ADR-0085 (Migração/saga), ADR-0086 (Event sourcing/Outbox), ADR-0087 (Reconciliação)
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (a propor — Cold Agent Hibernation & Checkpoint/Restore Protocol)
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 027-Cluster
---

# 008-Agent-Lifecycle — Testing

> Este documento define a **estratégia de testes** do módulo 008: pirâmide de
> testes, critérios de cobertura, fixtures/ambientes, e os cenários
> obrigatórios derivados diretamente da FSM (§4), da superfície de API (§5),
> do modelo de hibernação/checkpoint (§3, §9) e dos modos de falha (§9) do
> `_DESIGN_BRIEF.md`. Testes que não existem aqui **NÃO DEVEM** ser
> considerados cobertos apenas por inspeção manual — o gate de qualidade de
> CI é definido na §7.

## 1. Objetivo e Princípios

1. Todo comportamento normativo do brief (DEVE/NÃO DEVE) DEVE ter pelo menos
   um teste automatizado que o verifique — rastreabilidade requisito→teste.
2. A pirâmide de testes prioriza a base (unit > integration > contract > e2e
   > chaos > load), mas o módulo 008, por ser o **coordenador de uma FSM
   crítica com sagas de checkpoint/migração e concorrência serializada por
   lease**, exige investimento acima da média em testes de **concorrência**
   (fencing token, OCC) e de **integridade de dados** (hash de checkpoint).
3. Testes de falha (caminho infeliz) são tão obrigatórios quanto testes de
   caminho feliz — cada transição `T1`..`T11` da FSM (§4.3 do brief) tem um
   teste de sucesso e, onde aplicável, um teste da guarda que a bloqueia.
4. Todo teste de mutação DEVE verificar simultaneamente: (a) o efeito no
   `AcbStore`/`LifecycleTransition`; (b) o evento emitido via Outbox; (c) o
   registro em auditoria (quando aplicável); (d) o código de erro RFC-7807,
   quando aplicável.
5. Testes de hibernação/checkpoint DEVEM verificar **round-trip determinístico**:
   `hibernate` seguido de `wake` DEVE produzir um ACB e working memory
   bit-a-bit equivalentes ao estado pré-hibernação (mesmo `content_hash`
   lógico), exceto pelos campos que legitimamente mudam (`generation`,
   `wake_count`, timestamps).

## 2. Pirâmide de Testes

```
                         ┌─────────────┐
                         │    Load     │  poucos, alto custo, ambiente dedicado
                         │  (k6/NBomber)│  → ./Benchmark.md
                        ┌┴─────────────┴┐
                        │     Chaos      │  falhas injetadas (rede, storage, crash)
                        │ (Litmus/Toxi.) │
                       ┌┴────────────────┴┐
                       │       E2E         │  fluxo completo via Gateway (YARP)
                       │  (poucos cenários) │
                      ┌┴────────────────────┴┐
                      │      Contract         │  OpenAPI/proto ↔ implementação
                      │  (Pact/Buf breaking)  │
                     ┌┴───────────────────────┴┐
                     │       Integration         │  módulo 008 + PostgreSQL + Redis +
                     │   (Testcontainers)         │  MinIO + NATS reais (containers efêmeros)
                    ┌┴────────────────────────────┴┐
                    │            Unit                │  FSM, lease, checkpoint,
                    │     (xUnit + FluentAssertions)  │  reconciliação (mock deps)
                    └─────────────────────────────────┘
```

| Camada | Ferramentas (.NET 10) | Dependências reais? | Meta de execução |
|--------|-------------------------|-----------------------|---------------------|
| Unit | xUnit, FluentAssertions, NSubstitute (mocks) | Não (tudo mockado/in-memory) | < 5 min no pipeline de PR |
| Integration | xUnit + Testcontainers (PostgreSQL 16, Redis, MinIO, NATS/JetStream) | Sim, containers efêmeros | < 15 min |
| Contract | Pact (consumer-driven, contra `006`/`007`/`009`/`022`/`027`), `buf breaking` (proto `aios.lifecycle.v1`), Spectral (OpenAPI lint) | Simulado (mocks de contrato) | < 10 min |
| E2E | Playwright/HTTP client via Gateway YARP, ambiente `staging` isolado | Sim, stack completa | Execução noturna + pré-release |
| Chaos | Toxiproxy (latência/partição de rede), scripts de `kill -9` no coordenador/relay | Sim, ambiente dedicado | Execução semanal |
| Load | k6 (REST) / NBomber ou ghz (gRPC) | Sim, ambiente de benchmark | Execução por release, ver `./Benchmark.md` |

## 3. Cobertura-Alvo e Critérios de Qualidade de Gate

| Métrica de qualidade | Meta | Gate de CI |
|------------------------|------|------------|
| Cobertura de linha (unit + integration) | ≥ 85% no core (`LifecycleCoordinator`, `StateMachineEngine`, `AcbStore`, `LeaseManager`, `CheckpointService`, `MigrationOrchestrator`) | Bloqueia merge se abaixo |
| Cobertura de branch da FSM (§4.3 do brief) | 100% das transições `T1`..`T11` exercitadas (sucesso e guarda violada, quando aplicável) | Bloqueia merge |
| Cobertura de códigos de erro (`AIOS-LIFECYCLE-0001`..`0015`) | 100% dos códigos do catálogo (`API.md`) têm teste que os produz | Bloqueia merge |
| Contract tests vs. dependências (`006`,`007`,`009`,`022`,`027`) | 100% dos clients (`LifecycleEventConsumer`, `LifecyclePolicyEnforcer`, `MigrationOrchestrator.ClusterClient`) com contrato verificado | Bloqueia merge se contrato quebrado |
| Mutação (mutation testing, Stryker.NET) | Mutation score ≥ 70% nos componentes core | Relatório obrigatório, não bloqueante na v0.1 (bloqueante a partir de `Stable`) |
| Testes de idempotência | 100% das operações mutantes (`spawn`,`suspend`,`resume`,`hibernate`,`wake`,`migrate`,`checkpoint`,`restore`,`terminate`) com teste de replay de `Idempotency-Key` | Bloqueia merge |
| Testes de concorrência (lease/fencing) | Cenário de duas transições concorrentes no mesmo `agent_id` com verificação de serialização e fencing determinístico | Bloqueia merge |
| Testes de integridade de checkpoint | Corrupção sintética de blob → `restore` DEVE detectar e rejeitar (nunca aceitar silenciosamente) | Bloqueia merge |
| Vulnerabilidades (SCA) | 0 críticas/altas não mitigadas (`dotnet list package --vulnerable`) | Bloqueia merge |

## 4. Cenários Obrigatórios por Categoria

### 4.1 Unit — FSM Canônica (`StateMachineEngine` / `LifecycleCoordinator`)

| Cenário | Verifica |
|---------|----------|
| `∅ → Created` ao consumir `agent.control.created` (006) | T1, guarda G0 |
| `Created → Ready` após admissão do Scheduler | T2, guarda G1 |
| `Ready → Running` com slot alocado e `generation` incrementado | T3, guarda G3 |
| `Running → Suspended` com lease válido | T4, guarda G4 |
| `Running → Suspended` rejeitada durante operação crítica não-preemptível | T4, guarda G4 negada |
| `Suspended → Running` com working memory íntegra | T5, guarda G5 |
| `Suspended → Hibernated` após `idle_ttl` excedido e checkpoint `consistent` | T6, guarda G6 |
| `Suspended → Hibernated` rejeitada se checkpoint falhar | T6, guarda G6 negada, invariante INV4 |
| `Hibernated → Ready/Running` (`wake`) com checkpoint íntegro | T7, guarda G5/G7 |
| `Hibernated → Ready/Running` rejeitada com `content_hash` divergente | T7, `AIOS-LIFECYCLE-0007` |
| `Running/Suspended → Migrating` com decisão de placement e destino saudável | T8, guarda G7 |
| `Migrating → Ready/Running/Hibernated` (`cutover`) após materialização confirmada no destino | T9, guarda G8 |
| Qualquer não-terminal → `Terminated` com efeitos colaterais drenados | T10, guarda G10 |
| Qualquer não-terminal → `Failed` por erro irrecuperável | T11, guarda G9 |
| Transição fora da tabela (ex.: `Terminated → Running`) rejeitada | invariante INV3, `AIOS-LIFECYCLE-0002` |
| Escrita com `generation`/`fencing_token` desatualizado rejeitada | invariante geral, `AIOS-LIFECYCLE-0013` |
| Toda transição grava `LifecycleTransition` e `LifecycleOutbox` na mesma transação | invariante INV2 |
| Estado terminal é absorvente (nenhuma transição de saída aceita) | invariante INV3 |

### 4.2 Unit — LeaseManager (concorrência/fencing)

| Cenário | Verifica |
|---------|----------|
| Aquisição de lease livre → sucesso, `fencing_token` monotônico emitido | FR-010 |
| Aquisição concorrente do mesmo `agent_id` → uma sucede, outra recebe `AIOS-LIFECYCLE-0003` | invariante INV1, FR-010 |
| Renovação de lease antes do TTL mantém posse | `lease.renew_ms` |
| Expiração de lease sem renovação libera para outro coordenador | `lease.ttl_ms` |
| Escrita no `AcbStore` com `fencing_token` obsoleto (coordenador antigo) rejeitada | `AIOS-LIFECYCLE-0013`, mitigação de split-brain |

### 4.3 Unit — CheckpointService / SnapshotStore

| Cenário | Verifica |
|---------|----------|
| Criação de checkpoint `consistent` calcula `content_hash` (sha-256) corretamente | FR-006 |
| Cifragem AES-256-GCM aplicada por tenant | ADR-0082, §12.1 do brief |
| `restore` com hash íntegro reconstrói ACB e working memory idênticos ao momento do checkpoint | FR-006, round-trip determinístico |
| `restore` com hash divergente rejeitado, `AIOS-LIFECYCLE-0007`, tenta checkpoint anterior | modo de falha §9 do brief |
| Checkpoint sob demanda (`POST .../checkpoint`) não altera o estado do agente | contrato da API |
| Working memory acima de `checkpoint.max_working_memory_mib` rejeitada com erro apropriado | limite de configuração |

### 4.4 Unit — HibernationController / WarmPoolManager / SpawnManager

| Cenário | Verifica |
|---------|----------|
| Agente ocioso além de `hibernation.idle_ttl_s` é candidato a hibernação | FR-004 |
| Pressão de RAM acima de `hibernation.ram_pressure_threshold_pct` prioriza hibernação por `priority_class` | §9, storm de hibernação |
| `spawn` com runtime pré-aquecido (WarmPool hit) atinge `Running` sem boot completo | FR-014, NFR-001 (p50) |
| `spawn` sem runtime pré-aquecido (WarmPool miss) ainda atinge `p99 ≤ 250 ms` no ambiente de referência | NFR-001 (p99) |
| `wake` de agente frio restaura `generation+1` e emite `woken` | FR-005 |

### 4.5 Unit — MigrationOrchestrator (saga)

| Cenário | Verifica |
|---------|----------|
| Saga completa `quiesce→checkpoint→transfer→materialize→cutover→cleanup` sem perda de trabalho aceito | FR-007 |
| Falha na fase `materialize` (destino indisponível) compensa e reativa origem | modo de falha "migração travada", `MigrationJob=compensated` |
| `saga_timeout_ms` excedido aciona compensação automática | §9 do brief |
| Migração em lote respeita `migration.max_concurrent_per_node` | §10 do brief |

### 4.6 Unit — TombstoneManager / ReconciliationController

| Cenário | Verifica |
|---------|----------|
| Expurgo de agente `Terminated` após `retention.terminated_ttl_days` remove snapshots/checkpoints | FR-011 |
| ID de agente expurgado não é reutilizado (anti-resurrection) | §12.3 do brief |
| Runtime morto sem heartbeat além de `reconcile.runtime_dead_after_ms` é reconciliado para materialização/`Failed` | FR-009 |
| Checkpoint órfão (sem ACB correspondente) é detectado e tratado | reconciliação, FR-009 |

### 4.7 Integration (Testcontainers: PostgreSQL, Redis, MinIO, NATS/JetStream)

| Cenário | Verifica |
|---------|----------|
| Ciclo completo `Created→Ready→Running→Suspended→Hibernated→Ready→Running→Terminated` persiste corretamente em PostgreSQL, Redis e MinIO | FSM completa persistida |
| Crash simulado do coordenador entre commit da mutação e publicação do evento (mata o processo antes do relay) | FR-008: relay reenvia após restart, 0 perda |
| RLS por `tenant_id` em `AgentControlBlock`, `LifecycleTransition`, `Checkpoint`, `MigrationJob`: query com `tenant_id` de outro tenant não retorna linhas | isolamento multi-tenant (`Database.md`) |
| Sharding: distribuição de ACBs entre `placement_shard` conforme `hash(tenant,agent) mod N` | §10 do brief |
| Migração real entre dois "nós" simulados (containers distintos) completa `cutover` | FR-007, teste de integração de rede |

### 4.8 Contract Tests

| Par (consumidor→provedor) | Ferramenta | Verifica |
|----------------------------|------------|----------|
| Módulo 008 → `006-Kernel` (consumo de `agent.control.created`) | Pact | Schema do evento de criação de ACB estável |
| Módulo 008 → `009-Scheduler` (`scheduler.placement.decided`, `preemption.requested`) | Pact | Schema de decisão de admissão/placement estável |
| Módulo 008 → `007-Agent-Runtime` (heartbeat/exit) | Pact | Schema de `agent.runtime.heartbeat`/`exited` estável |
| Módulo 008 → `027-Cluster` (`cluster.node.draining`) | Pact | Schema de drenagem de nó estável |
| Módulo 008 → `022-Policy` (PDP) | Pact | Schema de `DecisionRequest`/`DecisionResponse` estável |
| Cliente externo → módulo 008 (REST) | Spectral (lint OpenAPI) + Dredd/Schemathesis | Conformidade do `API.md` (OpenAPI) |
| Cliente interno → módulo 008 (gRPC) | `buf breaking` | Nenhuma mudança incompatível no pacote `aios.lifecycle.v1` sem bump major |

### 4.9 E2E (via API Gateway YARP, ambiente `staging`)

| Cenário | Verifica |
|---------|----------|
| Agente é criado (`spawn`), executa, é suspenso, hiberna por ociosidade e é retomado sob demanda dentro de 250 ms | fluxo de ponta a ponta do brief, FR-005 |
| Agente é migrado de um shard para outro sem interrupção perceptível do trabalho aceito | FR-007 |
| Operação sem autorização do PDP retorna 403 com envelope RFC-7807 e evento de auditoria correspondente | FR-013 |
| Requisição repetida com mesma `Idempotency-Key` via API pública produz efeito único | NFR-013 |
| Encerramento (`terminate`) seguido de retenção e expurgo automático via `TombstoneManager` | FR-011, FR-012 |

### 4.10 Chaos Engineering

| Experimento | Falha injetada | Resultado esperado |
|-------------|-------------------|-----------------------|
| Latência de rede para `009-Scheduler` | Toxiproxy adiciona 2s de latência | `spawn` degrada latência sem travar outras transições (bulkhead) |
| Partição do NATS/JetStream | Bloqueio de porta do broker | Outbox acumula (`aios_lifecycle_outbox_pending` cresce) sem perder mensagens; drena ao restaurar |
| `kill -9` no coordenador entre commit e publish | Sinal ao processo em ponto aleatório | 0 evento perdido após restart (FR-008) |
| Queda do MinIO durante hibernação | Container MinIO parado | Hibernação adiada (agente permanece quente); `AIOS-LIFECYCLE-0014`; alerta |
| Nó destino cai durante migração | Kill do container destino na fase `materialize` | Compensação reativa origem; agente nunca perde trabalho aceito |
| Dois coordenadores simultâneos (split-brain simulado) | Bypass proposital de lease em teste controlado | Fencing token rejeita escrita do coordenador obsoleto (`AIOS-LIFECYCLE-0013`) |

## 5. Fixtures e Dados de Teste

| Fixture | Descrição |
|---------|-----------|
| `AcbFixtureFactory` | Gera ACBs em qualquer estado válido da FSM, com `working_memory_ref`, `context_ref`, `policy_ref`, `quota_ref` sintéticos, para testes de unidade/integração. |
| `TenantFixture` | Conjunto de tenants sintéticos (`acme`, `globex`, `initech`) para testes de isolamento RLS e retenção diferenciada. |
| `PolicyStub` | Dublê do PDP (`022-Policy`) configurável para retornar `allow`/`deny`/timeout determinístico. |
| `SchedulerStub` / `RuntimeStub` / `ClusterStub` | Dublês dos clientes de `009`/`007`/`027` com cenários de sucesso, timeout e rejeição parametrizáveis. |
| `CheckpointCorruptionInjector` | Utilitário que corrompe deliberadamente bytes de um blob de checkpoint para testes de detecção de integridade. |
| `IdempotencyKeyGenerator` | Gera ULIDs válidos para `Idempotency-Key`, incluindo casos de reuso intencional (teste de conflito). |
| Golden files de eventos | Payloads CloudEvents de referência (um por subject do §6.1 do brief) usados em testes de *snapshot* de serialização. |

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
[4] Contract tests (Pact broker, buf breaking) ─ falha → bloqueia
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
- Modos de falha (base dos experimentos de chaos): `./FailureRecovery.md`
- Metodologia de carga (complementar): `./Benchmark.md`
- Observabilidade usada para verificar resultados de teste: `./Monitoring.md`, `./Metrics.md`
