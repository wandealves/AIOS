---
Documento: Requirements
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010, ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `../001-Architecture/Architecture.md`, `../022-Policy/`, `../009-Scheduler/`, `../008-Agent-Lifecycle/`, `../020-Communication/`, `../021-Security/`, `../024-Observability/`, `../025-Audit/`
---

# 006-Kernel — Requirements

> Este documento é o **índice consolidado** de requisitos do módulo Kernel. Ele
> não redefine requisitos — apenas organiza, agrupa e rastreia os requisitos
> detalhados em `FunctionalRequirements.md` e `NonFunctionalRequirements.md` até
> os casos de uso (`UseCases.md`) e os testes que os verificam (`Testing.md`,
> `Benchmark.md`). Toda afirmação normativa DEVE ser consistente com o
> `_DESIGN_BRIEF.md` deste módulo, que é a fonte única de verdade interna.

## 1. Propósito e Escopo

### 1.1 Propósito

Consolidar, numerar e tornar rastreável o conjunto completo de requisitos do
Kernel — o núcleo cognitivo do AIOS responsável pela ABI de syscalls
cognitivas, pelo Agent Control Block (ACB), pelo enforcement de cotas de
recurso, pelo capability enforcement (PEP) e pela coordenação do ciclo de
vida do agente (delegando admissão ao `009-Scheduler` e materialização ao
`008-Agent-Lifecycle`).

### 1.2 Escopo (in)

- Requisitos funcionais da superfície de syscalls (`spawn`, `kill`, `suspend`,
  `resume`, `remember`, `recall`, `plan`, `invoke_tool`, `route_model`,
  `get_quota`, `checkpoint`) e da administração do ACB (`GetAcb`).
- Requisitos funcionais de governança (PEP↔PDP), enforcement de cotas,
  idempotência, emissão de eventos e brokering de recursos.
- Requisitos não-funcionais de desempenho, escalabilidade, disponibilidade,
  durabilidade, consistência, segurança, observabilidade, manutenibilidade e
  conformidade regulatória aplicáveis ao Kernel.
- Casos de uso que materializam esses requisitos em fluxos observáveis
  (felizes e de falha).

### 1.3 Escopo (out)

- Requisitos de **como** o PDP decide política (pertence a `022-Policy`).
- Requisitos de **como** o Scheduler faz placement/preempção (pertence a
  `009-Scheduler`).
- Requisitos de **como** o runtime do agente executa o loop ReAct (pertence a
  `007-Agent-Runtime`).
- Requisitos de armazenamento/recuperação de memória, compressão de contexto,
  planejamento, invocação de ferramentas ou roteamento de modelo em si — o
  Kernel apenas encaminha essas syscalls (pertencem a `010`, `011`, `012`,
  `015`, `017`, respectivamente).

## 2. Stakeholders

| Stakeholder | Papel/Interesse | Documentos primários de interesse |
|-------------|------------------|------------------------------------|
| Arquitetura-Chefe do AIOS | Garantir conformidade com `001-Architecture` e RFC-0001; aprovar ADRs `ADR-0060..0069`. | `Architecture.md`, `ADR.md`, `RFC.md` |
| Equipe do Kernel (dono do módulo 006) | Implementar, operar e evoluir o serviço; responder por SLOs. | Todos os 26 documentos |
| Equipe do 009-Scheduler | Contrato de admissão/preempção consumido pelo `LifecycleCoordinator`. | `API.md`, `SequenceDiagrams.md`, `Events.md` |
| Equipe do 008-Agent-Lifecycle | Contrato de materialização/hibernação/checkpoint. | `API.md`, `StateMachine.md`, `SequenceDiagrams.md` |
| Equipe do 022-Policy (PDP) | Contrato de decisão de capability consumido pelo `PolicyClient`. | `API.md`, `Security.md` |
| Equipe do 021-Security | AuthN/mTLS/claims consumidas pelo `SyscallGateway`. | `Security.md` |
| Equipe do 026-Cost-Optimizer | Fonte de orçamento consumida pelo `ResourceQuotaManager`. | `Events.md`, `Database.md` |
| Equipe do 024-Observability | Padrões de telemetria e dashboards. | `Metrics.md`, `Monitoring.md`, `Logging.md` |
| Equipe do 025-Audit | Consumidora dos registros de auditoria emitidos pelo Kernel. | `Events.md`, `Security.md` |
| Tenants (clientes do AIOS) | Consomem a ABI de syscalls via agentes; sujeitos a cotas e políticas. | `API.md`, `Examples.md` |
| Operações/SRE (029) | Opera o serviço em produção; monitora SLOs e executa runbooks. | `Monitoring.md`, `FailureRecovery.md`, `Deployment.md` |
| Compliance/DPO | Garante conformidade LGPD/GDPR sobre dados tratados pelo Kernel. | `Security.md`, `NonFunctionalRequirements.md` |

## 3. Sumário de Requisitos Funcionais

> Detalhamento completo em `FunctionalRequirements.md`. Os FRs estão agrupados
> por categoria funcional do Kernel; a numeração é global e sequencial
> (`FR-001`..`FR-026`).

| Categoria | Faixa de IDs | Resumo |
|-----------|-------------|--------|
| A. ABI de Syscalls Cognitivas | FR-001..FR-003 | Exposição, versionamento e validação da superfície de 11 verbos + `GetAcb`. |
| B. Governança (PEP ↔ PDP) | FR-004..FR-007 | Enforcement de capability, *default deny*, cache de decisão, isolamento de tenant. |
| C. ACB / Ciclo de Vida | FR-008..FR-014 | FSM do ACB, sagas de `spawn`/`suspend`/`resume`/`kill`/`checkpoint`, hibernação, árvore de agentes. |
| D. Cotas de Recurso | FR-015..FR-018 | Reserva/consumo/liberação atômicos, `get_quota`, enforcement hard/soft, alertas de limiar. |
| E. Idempotência e Eventos | FR-019..FR-021 | `Idempotency-Key`, Outbox transacional, rejeição de reuso divergente. |
| F. Brokering de Recursos | FR-022..FR-023 | Encaminhamento governado a `010/011/012/015/017` com resiliência. |
| G. Auditoria e Observabilidade | FR-024..FR-025 | Registro de syscall, telemetria OTel. |
| H. Administração | FR-026 | Leitura administrativa do ACB. |

## 4. Sumário de Requisitos Não-Funcionais

> Detalhamento completo, com metas numéricas (SLO/SLI) e método de verificação,
> em `NonFunctionalRequirements.md` (`NFR-001`..`NFR-018`).

| Atributo de qualidade | Faixa de IDs | Resumo da meta |
|------------------------|-------------|-----------------|
| Desempenho | NFR-001..NFR-003 | p99 `spawn` ≤ 250 ms; p99 syscall de controle ≤ 20 ms; p99 decisão PEP ≤ 8/25 ms. |
| Capacidade/Throughput | NFR-004 | ≥ 20.000 syscalls/s por réplica. |
| Disponibilidade | NFR-005 | ≥ 99,95% do serviço Kernel. |
| Escalabilidade | NFR-006 | ≥ 10⁶ ACBs com crescimento sub-linear de custo. |
| Durabilidade | NFR-007 | Perda de evento = 0 sob crash. |
| Recuperação (DR) | NFR-008 | RTO ≤ 15 min, RPO ≤ 5 min. |
| Consistência | NFR-009 | Overshoot de cota `hard` ≤ 1%. |
| Segurança | NFR-010 | 100% das ações privilegiadas via PEP; 0 bypass. |
| Observabilidade | NFR-011 | 100% das syscalls com trace OTel correlacionado. |
| Idempotência | NFR-012 | Repetições produzem efeito único em 100% das mutações. |
| Manutenibilidade | NFR-013 | Acoplamento e complexidade limitados por componente. |
| Compatibilidade/Portabilidade | NFR-014 | Coexistência ≥ 2 majors de ABI. |
| Testabilidade | NFR-015 | 100% dos verbos cobertos por contract test. |
| Conformidade regulatória | NFR-016 | LGPD/GDPR: minimização e direito ao esquecimento. |
| Eficiência de custo operacional | NFR-017 | Densidade de agentes `Hibernated` por GiB de RAM. |
| Resiliência | NFR-018 | MTTR de circuit breaker e recuperação automática. |

## 5. Matriz de Rastreabilidade — Requisito → Caso de Uso → Teste

> `UC-NNN` referem-se a `UseCases.md`. `Teste` referencia a estratégia descrita
> em `Testing.md` (unit/integration/contract/e2e/chaos/load) e `Benchmark.md`
> onde aplicável. Esta matriz DEVE ser atualizada a cada novo requisito ou
> caso de uso.

| Requisito | Caso(s) de Uso | Tipo de teste |
|-----------|-----------------|----------------|
| FR-001, FR-002, FR-003 | UC-001, UC-016 | Contract test (OpenAPI/proto), unit |
| FR-004, FR-005, FR-006 | UC-013, UC-018, UC-019 | Integration (mock PDP), chaos (PDP indisponível) |
| FR-007 | UC-014 | Integration (claims cruzadas), security test |
| FR-008 | UC-001, UC-017 | Unit (FSM), integration (OCC concorrente) |
| FR-009 | UC-001, UC-002, UC-003 | Integration (saga), chaos (falha do Scheduler/Lifecycle) |
| FR-010 | UC-004, UC-005, UC-006 | Integration, e2e |
| FR-011 | UC-007, UC-008 | Integration (cold resume), load (latência de materialização) |
| FR-012 | UC-009 | Integration (drenagem), e2e |
| FR-013 | UC-010 | Integration (checkpoint→MinIO), e2e |
| FR-014 | UC-020 | Integration (árvore de agentes), teste de limite de fan-out |
| FR-015, FR-017 | UC-012 | Integration (concorrência de cota), load |
| FR-016 | UC-011 | Unit, integration |
| FR-018 | UC-012 | Integration (evento de limiar) |
| FR-019, FR-021 | UC-016 | Integration (replay/duplicação) |
| FR-020 | UC-001 (todos os fluxos que emitem evento) | Chaos (crash entre commit/publish), integration |
| FR-022, FR-023 | UC-015 | Integration (mock de broker), chaos (broker indisponível) |
| FR-024, FR-025 | Todos os UCs | Inspeção de spans/logs, auditoria de cobertura |
| FR-026 | UC-011 (leitura administrativa correlata) | Unit, integration |
| NFR-001..NFR-004 | UC-001, UC-004, UC-006, UC-011 | Load test (`Benchmark.md`) |
| NFR-005, NFR-007, NFR-008 | UC-002, UC-003, UC-009 | Chaos test, DR drill |
| NFR-006, NFR-017 | UC-007, UC-008 | Teste de escala (`Scalability.md`) |
| NFR-009 | UC-012 | Teste concorrente de token-bucket |
| NFR-010 | UC-013, UC-014, UC-018 | Pentest, auditoria de cobertura de PEP |
| NFR-011 | Todos os UCs | Inspeção de spans/campos de correlação |
| NFR-012 | UC-016 | Teste de replay/duplicação |
| NFR-013, NFR-015 | Todos os UCs | Revisão de arquitetura, cobertura de contract test |
| NFR-014 | UC-001, UC-016 | Teste de compatibilidade entre versões de ABI |
| NFR-016 | UC-009 | Auditoria de expurgo/retenção |
| NFR-018 | UC-018 | Chaos test (circuit breaker) |

## 6. Glossário Local

> Este módulo **reutiliza** as definições de `../040-Glossary/Glossary.md` sem
> redefini-las. Os termos abaixo são os mais relevantes para o Kernel e estão
> listados apenas como referência de navegação (definição completa no
> glossário global):

| Termo | Ver definição em |
|-------|-------------------|
| ACB (Agent Control Block) | `../040-Glossary/Glossary.md#a` |
| Syscall Cognitiva | `../040-Glossary/Glossary.md#s` |
| Capability | `../040-Glossary/Glossary.md#c` |
| PEP / PDP | `../040-Glossary/Glossary.md#p` |
| Quota (Cota) | `../040-Glossary/Glossary.md#q` |
| Cold Agent | `../040-Glossary/Glossary.md#c` |
| Idempotency (Idempotência) | `../040-Glossary/Glossary.md#i` |
| Outbox | `../040-Glossary/Glossary.md#o` |
| Saga | `../040-Glossary/Glossary.md#s` |
| Shard | `../040-Glossary/Glossary.md#s` |
| RLS (Row-Level Security) | `../040-Glossary/Glossary.md#r` |
| Default Deny | `../040-Glossary/Glossary.md#d` |
| Bulkhead / Circuit Breaker | `../040-Glossary/Glossary.md#b`, `#c` |
| Tenant | `../040-Glossary/Glossary.md#t` |

## 7. Referências

- Fonte de verdade do módulo: `_DESIGN_BRIEF.md`
- Requisitos funcionais detalhados: `FunctionalRequirements.md`
- Requisitos não-funcionais detalhados: `NonFunctionalRequirements.md`
- Casos de uso: `UseCases.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Módulo de política: `../022-Policy/`
- Módulo de escalonamento: `../009-Scheduler/`
- Módulo de ciclo de vida físico: `../008-Agent-Lifecycle/`
