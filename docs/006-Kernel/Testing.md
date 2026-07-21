---
Documento: Testing
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060 (ABI de Syscalls), ADR-0061 (ACB/OCC), ADR-0062 (Cotas), ADR-0063 (PEP/PDP), ADR-0064 (Saga), ADR-0066 (Outbox)
RFCs relacionados: RFC-0001 (baseline), RFC-0006 (a propor — Cognitive Syscall ABI), RFC-0007 (a propor — ACB & Resource Quota Model)
Depende de: 022-Policy, 009-Scheduler, 008-Agent-Lifecycle, 020-Communication
---

# 006-Kernel — Testing

> Este documento define a **estratégia de testes** do Kernel: pirâmide de testes,
> critérios de cobertura, fixtures/ambientes, e os cenários obrigatórios
> derivados diretamente da FSM (§4), da ABI (§5), do modelo de cotas (§3.2) e
> dos modos de falha (§9) do `_DESIGN_BRIEF.md`. Testes que não existem aqui
> **NÃO DEVEM** ser considerados cobertos apenas por inspeção manual — o gate de
> qualidade de CI é definido na §7.

## 1. Objetivo e Princípios

1. Todo comportamento normativo do brief (DEVE/NÃO DEVE) DEVE ter pelo menos um
   teste automatizado que o verifique — rastreabilidade requisito→teste.
2. A pirâmide de testes prioriza a base (unit > integration > contract > e2e >
   chaos > load), mas o Kernel, por ser um **broker governado com FSM crítica**,
   exige investimento acima da média em testes de **contrato** e de
   **concorrência** (OCC, cotas).
3. Testes de falha (caminho infeliz) são tão obrigatórios quanto testes de
   caminho feliz — cada transição `T-0x` da FSM tem um teste de sucesso e,
   onde aplicável, um teste da guarda que a bloqueia.
4. Todo teste de mutação DEVE verificar simultaneamente: (a) o efeito no
   `AcbStore`; (b) o evento emitido via Outbox; (c) o registro em
   `syscall_log`; (d) o código de erro RFC-7807, quando aplicável.

## 2. Pirâmide de Testes

```
                         ┌─────────────┐
                         │    Load     │  poucos, alto custo, ambiente dedicado
                         │  (k6/NBomber)│  → ./Benchmark.md
                        ┌┴─────────────┴┐
                        │     Chaos      │  falhas injetadas (rede, deps, crash)
                        │ (Litmus/Toxi.) │
                       ┌┴────────────────┴┐
                       │       E2E         │  fluxo completo via Gateway (YARP)
                       │  (poucos cenários) │
                      ┌┴────────────────────┴┐
                      │      Contract         │  OpenAPI/proto ↔ implementação
                      │  (Pact/Buf breaking)  │
                     ┌┴───────────────────────┴┐
                     │       Integration         │  Kernel + PostgreSQL + Redis +
                     │   (Testcontainers)         │  NATS reais (containers efêmeros)
                    ┌┴────────────────────────────┴┐
                    │            Unit                │  handlers, FSM, PEP,
                    │     (xUnit + FluentAssertions)  │  ResourceQuotaManager (mock deps)
                    └─────────────────────────────────┘
```

| Camada | Ferramentas (.NET 10) | Dependências reais? | Meta de execução |
|--------|-------------------------|-----------------------|---------------------|
| Unit | xUnit, FluentAssertions, NSubstitute (mocks) | Não (tudo mockado/in-memory) | < 5 min no pipeline de PR |
| Integration | xUnit + Testcontainers (PostgreSQL 16, Redis, NATS/JetStream) | Sim, containers efêmeros | < 15 min |
| Contract | Pact (consumer-driven, contra `009`/`022`/`010`/`011`/`012`/`015`/`017`), `buf breaking` (proto), Spectral (OpenAPI lint) | Simulado (mocks de contrato) | < 10 min |
| E2E | Playwright/HTTP client via Gateway YARP, ambiente `staging` isolado | Sim, stack completa | Execução noturna + pré-release |
| Chaos | Toxiproxy (latência/partição de rede), scripts de `kill -9` no relay do Outbox | Sim, ambiente dedicado | Execução semanal |
| Load | k6 (REST) / NBomber ou ghz (gRPC) | Sim, ambiente de benchmark | Execução por release, ver `./Benchmark.md` |

## 3. Cobertura-Alvo e Critérios de Qualidade de Gate

| Métrica de qualidade | Meta | Gate de CI |
|------------------------|------|------------|
| Cobertura de linha (unit + integration) | ≥ 85% no core (`SyscallGateway`, `CapabilityEnforcer`, `AcbStore`, `ResourceQuotaManager`, `LifecycleCoordinator`) | Bloqueia merge se abaixo |
| Cobertura de branch da FSM (§4 do brief) | 100% das transições `T-01`..`T-13` exercitadas (sucesso e guarda violada, quando aplicável) | Bloqueia merge |
| Cobertura de códigos de erro (`AIOS-KERNEL-*`, `AIOS-CAP-*`, `AIOS-QUOTA-*`, `AIOS-SYSCALL-*`) | 100% dos códigos do catálogo (`API.md`) têm teste que os produz | Bloqueia merge |
| Contract tests vs. dependências (`009`,`022`,`010`,`011`,`012`,`015`,`017`) | 100% dos clients (`SchedulerClient`, `PolicyClient`, `LifecycleClient`, `ResourceBrokerRouter`) com contrato verificado | Bloqueia merge se contrato quebrado |
| Mutação (mutation testing, Stryker.NET) | Mutation score ≥ 70% nos componentes core | Relatório obrigatório, não bloqueante na v0.1 (bloqueante a partir de `Stable`) |
| Testes de idempotência | 100% das syscalls mutantes (`spawn`,`kill`,`suspend`,`resume`,`remember`,`plan`,`invoke_tool`,`route_model`,`checkpoint`) com teste de replay de `Idempotency-Key` | Bloqueia merge |
| Testes de concorrência (OCC) | Cenário de escrita concorrente no mesmo `urn` com verificação de conflito determinístico | Bloqueia merge |
| Vulnerabilidades (SCA) | 0 críticas/altas não mitigadas (`dotnet list package --vulnerable`) | Bloqueia merge |

## 4. Cenários Obrigatórios por Categoria

### 4.1 Unit — FSM do ACB (`AcbStore` / `LifecycleCoordinator`)

| Cenário | Verifica |
|---------|----------|
| `spawn` com capability e cota disponíveis → `Pending` | T-01 caminho feliz |
| `spawn` sem capability `agent:spawn` → permanece ausente, `AIOS-CAP-0001` | T-01 guarda negada |
| `Pending` → `Admitted` após resposta do Scheduler | T-02 |
| `Pending` → `Failed` em timeout de admissão | T-03 |
| `Admitted` → `Running` após confirmação do Lifecycle | T-04 |
| `Admitted` → `Failed` em `boot_timeout` | T-05 |
| `Running` → `Suspended` via `suspend` com capability | T-06 |
| `Suspended` → `Running` via `resume` | T-07 |
| `Suspended` → `Hibernated` por ociosidade (`idle_ttl` excedido) | T-08 |
| `Hibernated` → `Admitted` via `resume` com checkpoint válido | T-09 |
| Qualquer estado não-terminal → `Terminating` via `kill` autorizado | T-10 |
| `Terminating` → `Terminated` após drenagem e liberação de cotas | T-11 |
| `Running` → `Failed` por heartbeat perdido além do deadline | T-12 |
| `checkpoint` em `Running`/`Suspended` atualiza `checkpoint_ref` sem mudar estado | T-13 |
| Transição fora da tabela (ex.: `Terminated` → `Running`) rejeitada | `AIOS-KERNEL-0002` |
| Escrita com `version` desatualizado rejeitada e retry até `optimistic_retry_max` | `AIOS-KERNEL-0009`, invariante I3 |
| Todo estado terminal preenche `terminated_at` e libera `quota_ref` | invariante I2 |
| `runtime_ref` só não-nulo em `Running`/`Suspended` | invariante I1 |

### 4.2 Unit — CapabilityEnforcer (PEP)

| Cenário | Verifica |
|---------|----------|
| Syscall privilegiada sem decisão prévia consulta o PDP e aplica o resultado | FR-002 |
| Decisão em cache (TTL não expirado) não reconsulta o PDP | performance NFR-003 |
| PDP indisponível com `fail_mode=closed` → nega (`AIOS-CAP-0003`) | *fail-closed* |
| PDP indisponível com `fail_mode=open` (config explícita) → permite com alerta | modo degradado documentado |
| `tenant` do payload diverge do contexto autenticado → nega (`AIOS-CAP-0002`) | FR-011, isolamento multi-tenant |
| Toda decisão (allow/deny) gera evento de auditoria e métrica `aios_kernel_pep_decisions_total` | FR-012, NFR-011 |

### 4.3 Unit — ResourceQuotaManager

| Cenário | Verifica |
|---------|----------|
| Reserva atômica de tokens dentro do limite → sucesso | FR-004 |
| Reserva que excede limite `hard` → rejeitada, `AIOS-QUOTA-0001`, evento `agent.quota.exceeded` | FR-004 |
| Reserva que excede limite `soft` → aceita com marcação e evento `agent.quota.warning` | enforcement `soft` |
| N reservas concorrentes no mesmo `subject_urn` não ultrapassam o limite em mais de 1% | NFR-009 |
| Liberação de cota após `kill`/`Terminated` zera a reserva do agente | invariante I2 |
| `get_quota` retorna saldo coerente com os contadores Redis (±1 janela) | FR-010 |

### 4.4 Unit — IdempotencyStore / EventEmitter

| Cenário | Verifica |
|---------|----------|
| Repetição da mesma `Idempotency-Key` e mesmo payload retorna resultado idêntico | FR-006, NFR-012 |
| Mesma `Idempotency-Key`, payload divergente → `AIOS-SYSCALL-0002` | FR-006 |
| Toda mutação persistida grava exatamente um registro no Outbox na mesma transação | invariante I4 |
| Relay publica eventos pendentes e marca `published=true` | comportamento do EventEmitter |
| Consumidor duplicado (mesmo `event.id`) processado apenas uma vez (dedupe) | RFC-0001 §5.5 |

### 4.5 Integration (Testcontainers: PostgreSQL, Redis, NATS/JetStream)

| Cenário | Verifica |
|---------|----------|
| Ciclo completo `spawn`→`Running`→`checkpoint`→`suspend`→`resume`→`kill`→`Terminated` persiste corretamente em PostgreSQL e Redis | FSM completa persistida |
| Crash simulado do processo Kernel entre commit da mutação e publicação do evento (mata o processo antes do relay) | NFR-007: relay reenvia após restart, 0 perda |
| RLS por `tenant_id`: query com `tenant_id` de outro tenant não retorna linhas | isolamento multi-tenant (Database.md) |
| Sharding: distribuição de ACBs entre `shard_key` conforme `hash(tenant,agent) mod N` | ADR-0065 |
| Reconciliação periódica de cota (Redis↔PostgreSQL) corrige *drift* | `kernel.quota` reconciliação |

### 4.6 Contract Tests

| Par (consumidor→provedor) | Ferramenta | Verifica |
|----------------------------|------------|----------|
| Kernel → `009-Scheduler` (`SchedulerClient`) | Pact | Schema de admissão/decisão de preempção estável |
| Kernel → `022-Policy` (`PolicyClient`) | Pact | Schema de `DecisionRequest`/`DecisionResponse` estável |
| Kernel → `008-Agent-Lifecycle` (`LifecycleClient`) | Pact | Schema de materialização/hibernação/checkpoint estável |
| Kernel → `010/011/012/015/017` (`ResourceBrokerRouter`) | Pact | Schema de encaminhamento de `remember/recall/plan/invoke_tool/route_model` |
| Cliente externo → Kernel (REST) | Spectral (lint OpenAPI) + Dredd/Schemathesis | Conformidade do `API.md` (OpenAPI) |
| Cliente interno → Kernel (gRPC) | `buf breaking` | Nenhuma mudança incompatível no pacote `aios.kernel.v1` sem bump major |

### 4.7 E2E (via API Gateway YARP, ambiente `staging`)

| Cenário | Verifica |
|---------|----------|
| Agente é criado (`spawn`), executa `remember`/`recall`/`plan`/`invoke_tool`/`route_model` e é encerrado (`kill`) — fluxo completo | fluxo de ponta a ponta do brief §2.2 |
| Agente ocioso hiberna e é retomado sob demanda (`resume` de `Hibernated`) dentro de 250 ms | FR-009, T-09 |
| Syscall sem capability retorna 403 com envelope RFC-7807 e evento de auditoria correspondente | FR-002, FR-012 |
| Requisição repetida com mesma `Idempotency-Key` via API pública produz efeito único | FR-006 |

### 4.8 Chaos Engineering

| Experimento | Falha injetada | Resultado esperado |
|-------------|-------------------|-----------------------|
| Latência de rede para `022-Policy` | Toxiproxy adiciona 2s de latência | PEP degrada para `fail_mode` configurado sem travar outras syscalls (bulkhead) |
| Partição do NATS/JetStream | Bloqueio de porta do broker | Outbox acumula (`aios_kernel_outbox_pending` cresce) sem perder mensagens; drena ao restaurar |
| `kill -9` no processo Kernel entre commit e publish | Sinal ao processo em ponto aleatório | 0 evento perdido após restart (NFR-007) |
| Queda do Redis (contadores quentes de cota) | Container Redis parado | Fallback conservador: nega syscalls marginais em vez de permitir sem controle (ver `FailureRecovery.md`) |
| Scheduler indisponível durante `spawn` | Mock retorna 503 | ACB atinge `Failed` com compensação (libera cota reservada), não fica preso em `Pending` |

## 5. Fixtures e Dados de Teste

| Fixture | Descrição |
|---------|-----------|
| `AcbFixtureFactory` | Gera ACBs em qualquer estado válido da FSM, com `quota_ref`, `memory_ptr`, `context_ptr` sintéticos, para testes de unidade/integração. |
| `TenantFixture` | Conjunto de tenants sintéticos (`acme`, `globex`, `initech`) com `Quota` distintas (hard/soft) para testes de isolamento e enforcement. |
| `PolicyStub` | Dublê do PDP (`022-Policy`) configurável para retornar `allow`/`deny`/timeout determinístico em testes de unidade e integração. |
| `SchedulerStub` / `LifecycleStub` | Dublês dos clientes de `009`/`008` com cenários de sucesso, timeout e rejeição parametrizáveis. |
| `IdempotencyKeyGenerator` | Gera ULIDs válidos para `Idempotency-Key`, incluindo casos de reuso intencional (teste de conflito). |
| Golden files de eventos | Payloads CloudEvents de referência (um por subject do §6 do brief) usados em testes de *snapshot* de serialização. |

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
