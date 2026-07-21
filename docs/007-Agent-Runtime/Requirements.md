---
Documento: Requirements
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `../001-Architecture/Architecture.md`, `../006-Kernel/`, `../008-Lifecycle/`, `../009-Scheduler/`, `../010-Memory/`, `../011-Context/`, `../012-Planning/`, `../015-Tool-Manager/`, `../017-Model-Router/`, `../020-Communication/`, `../021-Security/`, `../022-Policy/`, `../024-Observability/`, `../025-Audit/`
---

# 007-Agent-Runtime — Requirements

> Este documento é o **índice consolidado** de requisitos do módulo Agent
> Runtime. Ele não redefine requisitos — apenas organiza, agrupa e rastreia os
> requisitos detalhados em `FunctionalRequirements.md` e
> `NonFunctionalRequirements.md` até os casos de uso (`UseCases.md`) e os
> testes que os verificam (`Testing.md`, `Benchmark.md`). Toda afirmação
> normativa DEVE ser consistente com o `_DESIGN_BRIEF.md` deste módulo, que é
> a fonte única de verdade interna.

## 1. Propósito e Escopo

### 1.1 Propósito

Consolidar, numerar e tornar rastreável o conjunto completo de requisitos do
Agent Runtime — o **plano de dados** do AIOS, escrito em **Python**, que
materializa um agente a partir de um `AgentSpec`, executa seu **loop cognitivo
ReAct** (`perceber → pensar → agir → observar → repetir`) isolado em
**sandbox**, hospeda o **MCP host** para ferramentas, invoca ferramentas
exclusivamente via `015-Tool-Manager` e modelos exclusivamente via
`017-Model-Router`, e reporta ciclo de vida/telemetria ao **Runtime
Supervisor** (`.NET 10`, plano de controle) e ao barramento NATS/JetStream.

### 1.2 Escopo (in)

- Requisitos funcionais do processo runtime Python: bootstrap de agente,
  loop cognitivo ReAct, hospedagem do MCP host, invocação governada de tools
  e modelos, recuperação/persistência de memória e contexto via clientes
  dedicados, checkpoint/hibernação (*cold agent*), cotas locais, PEP local
  (*default deny*), emissão de eventos/telemetria e canal de controle com o
  Runtime Supervisor.
- Requisitos não-funcionais de desempenho (cold start, overhead de sandbox,
  latência de passo), escalabilidade (sessões concorrentes, agentes *cold*),
  disponibilidade, recuperabilidade (RTO/RPO de sessão), throughput de
  eventos, segurança (isolamento de sandbox, PEP), observabilidade, custo de
  memória ociosa, manutenibilidade e isolamento multi-tenant.
- Casos de uso que materializam esses requisitos em fluxos observáveis
  (felizes e de falha), cobrindo a máquina de estados `AgentExecState` e
  `RuntimeState` definidas no `_DESIGN_BRIEF.md` §4.

### 1.3 Escopo (out)

- Requisitos de **onde**/**quando**/**com que prioridade** um agente executa
  (admissão, preempção, *placement*) — pertence a `009-Scheduler`.
- Requisitos de **gerenciamento do pool** de runtimes (criar/monitorar/matar
  réplicas, autoscaling, cotas de pool) — pertence ao Runtime Supervisor
  (`008-Lifecycle`).
- Requisitos de **escolha/roteamento de modelo** por custo/qualidade/latência
  — pertence a `017-Model-Router`.
- Requisitos de **registro/versionamento de ferramentas** e implementação de
  *drivers* de tool — pertence a `015-Tool-Manager`.
- Requisitos de **armazenamento/recuperação de memória**, **compressão de
  contexto** e **planejamento/replanejamento** em si — o runtime apenas
  consome esses serviços via `MemoryClient`, `ContextClient` e
  `PlanningClient` (pertencem a `010`, `011`, `012`, respectivamente).
- Requisitos de **definição de políticas** RBAC/ABAC — o runtime é PEP, não
  PDP (pertence a `022-Policy`).
- Requisitos de **orquestração de workflows determinísticos/sagas** de
  múltiplos agentes — pertence a `014-Workflow`.

## 2. Stakeholders

| Stakeholder | Papel/Interesse | Documentos primários de interesse |
|-------------|------------------|------------------------------------|
| Arquitetura-Chefe do AIOS | Garantir conformidade com `001-Architecture` e RFC-0001; aprovar ADRs `ADR-0070..0079`. | `Architecture.md`, `ADR.md`, `RFC.md` |
| Equipe do Agent Runtime (dono do módulo 007) | Implementar, operar e evoluir o processo Python; responder por SLOs de cold start e loop. | Todos os 26 documentos |
| Equipe do Runtime Supervisor (`008-Lifecycle`) | Contrato de controle `RuntimeControl` (boot/suspend/resume/kill/drain), *placement*, pool. | `API.md`, `SequenceDiagrams.md`, `StateMachine.md` |
| Equipe do 009-Scheduler | Consumidora de eventos de ciclo de vida do runtime para admissão/preempção. | `Events.md`, `API.md` |
| Equipe do 015-Tool-Manager | Contrato de invocação de ferramenta (autorização, métricas, versão) consumido pelo `ToolInvoker`/`McpHost`. | `API.md`, `SequenceDiagrams.md`, `Events.md` |
| Equipe do 017-Model-Router | Contrato de inferência consumido pelo `ModelRouterClient` (streaming, fallback). | `API.md`, `SequenceDiagrams.md` |
| Equipe do 010-Memory / 011-Context | Contratos de recuperação/persistência de memória e montagem/compressão de contexto. | `API.md`, `ClassDiagrams.md` |
| Equipe do 012-Planning | Contrato de plano/replanejamento consumido pelo `PlanningClient`. | `API.md`, `StateMachine.md` |
| Equipe do 021-Security | Perfis de sandbox (`seccomp-bpf`, `cgroups v2`), mTLS, segredos. | `Security.md` |
| Equipe do 022-Policy | PDP consultado pelo `CapabilityEnforcer` (PEP local) em decisões dinâmicas. | `Security.md`, `API.md` |
| Equipe do 024-Observability | Padrões de telemetria OTel e dashboards. | `Metrics.md`, `Monitoring.md`, `Logging.md` |
| Equipe do 025-Audit | Consumidora de eventos de violação de sandbox e ações privilegiadas. | `Events.md`, `Security.md` |
| Tenants (clientes do AIOS) | Consomem agentes executados pelo runtime; sujeitos a cotas locais e sandbox. | `API.md`, `Examples.md` |
| Operações/SRE (`029-Operations`) | Opera pools de runtime em produção; monitora cold start, throughput, sandbox violations. | `Monitoring.md`, `FailureRecovery.md`, `Deployment.md` |
| Compliance/DPO | Garante conformidade LGPD/GDPR sobre payloads de evento/log/checkpoint. | `Security.md`, `NonFunctionalRequirements.md` |

## 3. Sumário de Requisitos Funcionais

> Detalhamento completo em `FunctionalRequirements.md`. Os FRs seguem a
> numeração e o texto normativo já fixados no `_DESIGN_BRIEF.md` §7.1
> (`FR-001`..`FR-018`), agrupados aqui por categoria funcional para
> navegação.

| Categoria | Faixa de IDs | Resumo |
|-----------|-------------|--------|
| A. Materialização e Ciclo de Vida do Agente | FR-001, FR-009, FR-018 | Boot a partir de `AgentSpec`, checkpoint/hibernação (*cold agent*), heartbeat/health ao Supervisor. |
| B. Loop Cognitivo ReAct | FR-002, FR-006, FR-016, FR-017 | Sequenciamento `perceive→think→act→observe`, replanejamento, watchdog anti-*stuck*, limites de passos/profundidade/fan-out. |
| C. Integração com Model Router e Tool Manager | FR-003, FR-004, FR-014 | Inferência exclusivamente via 017, ferramentas exclusivamente via 015/MCP host, *circuit breaker*/retry. |
| D. Memória e Contexto | FR-005 | Recuperação/persistência via 010, montagem de contexto via 011. |
| E. Isolamento, Sandbox e Segurança | FR-007, FR-008, FR-015 | `seccomp-bpf`/`cgroups v2`/netns/FS, PEP local *default deny*, redação de PII. |
| F. Cotas e Governança de Recursos Locais | FR-010 | Tokens, USD, wall-clock, passos; abortar/suspender ao esgotar. |
| G. Eventos, Idempotência e Streaming | FR-011, FR-012, FR-013 | Eventos NATS/telemetria OTel via outbox, idempotência de submissão, *streaming* de passos ao control plane. |

## 4. Sumário de Requisitos Não-Funcionais

> Detalhamento completo, com metas numéricas (SLO/SLI) e método de
> verificação, em `NonFunctionalRequirements.md` (`NFR-001`..`NFR-016`).

| Atributo de qualidade | Faixa de IDs | Resumo da meta |
|------------------------|-------------|-----------------|
| Desempenho — cold start | NFR-001 | p99 boot→ready ≤ 250 ms (pool quente); ≤ 900 ms (sem pool quente). |
| Desempenho — overhead de sandbox | NFR-002 | Overhead de CPU ≤ 5%; overhead de latência por passo ≤ 8 ms. |
| Desempenho — passo do loop | NFR-003 | p95 de latência de passo (excl. LLM/tool) ≤ 15 ms. |
| Escalabilidade | NFR-004 | ≥ 500 sessões concorrentes por nó; 10⁶+ agentes majoritariamente *cold*. |
| Disponibilidade | NFR-005 | Pool disponível ≥ 99,9%; falha de instância não perde trabalho aceito. |
| Recuperabilidade | NFR-006 | RTO ≤ 30 s para retomar sessão suspensa; RPO ≤ 1 passo. |
| Throughput de eventos | NFR-007 | ≥ 5.000 msg/s de eventos de execução por nó sem *backpressure*. |
| Segurança | NFR-008 | 100% das ações privilegiadas via PEP; 0 escapes de sandbox em *pentest*. |
| Observabilidade | NFR-009 | 100% dos passos com trace/span e correlação `traceparent`/`tenant`. |
| Custo | NFR-010 | Overhead de memória por sessão ociosa ≤ 30 MiB; sessão *cold* consome 0 RAM. |
| Manutenibilidade | NFR-011 | Cobertura de testes ≥ 85%; contratos gRPC/eventos testados por *contract tests*. |
| Isolamento | NFR-012 | Nenhum vazamento cross-tenant; `tenant_id` validado contra contexto autenticado. |
| Conformidade regulatória (LGPD/GDPR) | NFR-013 | Redação de PII 100% aplicada; expurgo rastreável de checkpoints/working memory. |
| Compatibilidade/Portabilidade | NFR-014 | ABI gRPC `aios.runtime.v1` coexiste ≥ 2 majors; MCP host tolera evolução aditiva de catálogo. |
| Testabilidade | NFR-015 | 100% dos métodos `RuntimeControl`/`AgentExecution` cobertos por contract test. |
| Resiliência | NFR-016 | *Circuit breaker* de tool/LLM abre em ≤ 1 janela de avaliação e fecha em ≤ 5 min após recuperação. |

## 5. Matriz de Rastreabilidade — Requisito → Caso de Uso → Teste

> `UC-NNN` referem-se a `UseCases.md`. `Teste` referencia a estratégia
> descrita em `Testing.md` (unit/integration/contract/e2e/chaos/load) e
> `Benchmark.md` onde aplicável. Esta matriz DEVE ser atualizada a cada novo
> requisito ou caso de uso.

| Requisito | Caso(s) de Uso | Tipo de teste |
|-----------|-----------------|----------------|
| FR-001 | UC-001, UC-002 | Contract test (gRPC `Boot`), integration (sandbox real) |
| FR-002 | UC-003, UC-004 | Unit (máquina de estados), integration (loop completo) |
| FR-003 | UC-003, UC-012 | Contract test (proibição de chamada direta a provedor), chaos (Model Router indisponível) |
| FR-004 | UC-003, UC-011, UC-013 | Contract test (proibição de invocação direta), chaos (Tool Manager indisponível) |
| FR-005 | UC-003 | Integration (mock 010/011), e2e |
| FR-006 | UC-005 | Integration (replanejamento), chaos (falha recuperável simulada) |
| FR-007 | UC-010 | *Pentest*/security test (escape de sandbox), integration (seccomp/cgroups) |
| FR-008 | UC-011 | Integration (capability ausente), security test |
| FR-009 | UC-007, UC-008 | Integration (checkpoint→resume), chaos (crash entre checkpoint) |
| FR-010 | UC-007 | Integration (esgotamento de cota), load |
| FR-011 | Todos os UCs | Inspeção de spans/eventos, teste de outbox (crash injection) |
| FR-012 | UC-014 | Integration (replay de `SubmitTask`) |
| FR-013 | UC-016 | Integration (stream de passos), e2e |
| FR-014 | UC-012, UC-013 | Chaos test (circuit breaker), integration (retry/backoff) |
| FR-015 | UC-003 (verificação transversal) | Auditoria de amostragem de payloads |
| FR-016 | UC-015 | Chaos test (loop travado/deadlock simulado) |
| FR-017 | UC-006 | Integration (limite de passos/profundidade/fan-out excedido) |
| FR-018 | UC-001, UC-009 | Integration (heartbeat), *chaos* (heartbeat perdido) |
| NFR-001, NFR-002, NFR-003 | UC-001, UC-003 | Load test (`Benchmark.md`) |
| NFR-004, NFR-010 | UC-007, UC-008 | Teste de escala (`Scalability.md`) |
| NFR-005, NFR-006 | UC-007, UC-008, UC-009 | Chaos test, DR drill |
| NFR-007 | Todos os UCs que publicam evento | Load test de outbox/publish |
| NFR-008 | UC-010, UC-011 | Pentest, auditoria de cobertura de PEP |
| NFR-009 | Todos os UCs | Inspeção de spans/campos de correlação |
| NFR-011, NFR-015 | Todos os UCs | Cobertura de contract test, revisão de arquitetura |
| NFR-012 | UC-017 (isolamento de tenant, transversal) | Teste de isolamento, RLS no control plane |
| NFR-013 | UC-003 (redação transversal) | Auditoria de amostragem de PII |
| NFR-014 | UC-001, UC-018 | Teste de compatibilidade entre versões de ABI/catálogo MCP |
| NFR-016 | UC-012, UC-013 | Chaos test (circuit breaker) |

## 6. Glossário Local

> Este módulo **reutiliza** as definições de `../040-Glossary/Glossary.md`
> sem redefini-las. Os termos abaixo são os mais relevantes para o Agent
> Runtime e estão listados apenas como referência de navegação (definição
> completa no glossário global):

| Termo | Ver definição em |
|-------|-------------------|
| Agent Runtime | `../040-Glossary/Glossary.md#a` |
| ReAct | `../040-Glossary/Glossary.md#r` |
| MCP (Model Context Protocol) | `../040-Glossary/Glossary.md#m` |
| Sandbox | `../040-Glossary/Glossary.md#s` |
| Cold Agent | `../040-Glossary/Glossary.md#c` |
| Capability | `../040-Glossary/Glossary.md#c` |
| PEP / PDP | `../040-Glossary/Glossary.md#p` |
| Default Deny | `../040-Glossary/Glossary.md#d` |
| Circuit Breaker | `../040-Glossary/Glossary.md#c` |
| Outbox | `../040-Glossary/Glossary.md#o` |
| Idempotency (Idempotência) | `../040-Glossary/Glossary.md#i` |
| Working Memory | `../040-Glossary/Glossary.md#w` |
| Data Plane (Plano de Dados) | `../040-Glossary/Glossary.md#d` |
| Runtime Supervisor | `../040-Glossary/Glossary.md#r` |
| Shard | `../040-Glossary/Glossary.md#s` |
| Bulkhead | `../040-Glossary/Glossary.md#b` |
| Reflexion | `../040-Glossary/Glossary.md#r` |

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
- Módulo de ciclo de vida físico (Runtime Supervisor): `../008-Lifecycle/`
- Módulo de ferramentas: `../015-Tool-Manager/`
- Módulo de roteamento de modelo: `../017-Model-Router/`
