---
Documento: ADR.md
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010, ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: 001-Architecture, 022-Policy, 009-Scheduler, 008-Lifecycle, 002-ADR
---

# 006-Kernel — Índice de ADRs

> **Escopo deste documento.** Este documento **NÃO** decide nada por si — é um
> **índice navegável** das *Architecture Decision Records* (ADR) que afetam o
> módulo 006-Kernel. As decisões em si residem em `../002-ADR/`, no formato
> definido por `../002-ADR/TEMPLATE.md` (Contexto · Problema · Alternativas ·
> Análise · Escolha · Consequências · Riscos · Trade-offs). Este arquivo **DEVE**
> ser mantido sincronizado com `../_DESIGN_BRIEF.md` §11 e com `../002-ADR/README.md`;
> qualquer divergência é um defeito de documentação.

> **Convenção de numeração.** Conforme `../_DESIGN_BRIEF.md` §11, a faixa
> `ADR-0060`..`ADR-0069` está **reservada** ao módulo 006-Kernel (regra
> `módulo × 10`, evitando colisão com faixas de outros módulos — ex.: `009-Scheduler`
> usaria `ADR-0090`..`ADR-0099`). Nenhuma outra numeração PODE ser usada por este
> módulo sem atualizar este índice e `../002-ADR/README.md`.

---

## 1. ADRs Globais que Restringem o Kernel

Estas ADRs foram decididas na fundação canônica do AIOS (`../002-ADR/`) e **DEVEM**
ser respeitadas por qualquer ADR específica do Kernel — elas não são redecididas
aqui, apenas referenciadas.

| ADR | Título | Status | Como restringe o Kernel |
|-----|--------|--------|--------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O Kernel é o **núcleo cognitivo** de um SO próprio — não é uma camada de conveniência sobre um framework de orquestração de terceiros; a ABI de syscalls (§5 do brief) é a materialização direta desta decisão. |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | O Kernel vive inteiramente no **control plane** (.NET 10); nunca executa loop de raciocínio (isso é `007-Agent-Runtime`, no data plane). Fronteira de confiança formalizada no PEP. |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | O Kernel Service é implementado em **.NET 10**; comunica-se com o `007-Agent-Runtime` (Python) exclusivamente via a ABI de syscalls (REST/gRPC), nunca por acoplamento de processo. |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | Todo evento de ciclo de vida e de decisão do Kernel (§6 do brief) é publicado via **NATS/JetStream**, nunca via um bus alternativo. |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | O `AcbStore` usa **PostgreSQL** como fonte da verdade do ACB, cotas, `syscall_log` e Outbox. O Kernel não usa pgvector/AGE diretamente (isso é domínio de `010-Memory`), mas herda a decisão de unificação de armazenamento relacional. |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e locks | Accepted | Projeção quente do ACB, token-buckets de cota e locks distribuídos de transição de FSM usam **Redis**, nunca um cache alternativo. |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | Accepted | Fundamenta diretamente o papel do Kernel como **PEP**: toda syscall privilegiada consulta o PDP (`022-Policy`) com postura *default deny*. Ver ADR-0063 (específica deste módulo) para a fronteira PEP/PDP em detalhe. |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | Todo o `KernelTelemetry` (spans OTel, métricas `aios_kernel_*`, logs Serilog, auditoria via `025-Audit`) implementa esta ADR — observabilidade não é opcional nem adicionada a posteriori. |

---

## 2. ADRs Específicas do Módulo 006-Kernel (Propostas)

> **Estado de redação.** As ADRs abaixo estão **reservadas** na faixa `ADR-0060`–
> `ADR-0069` e seu conteúdo (Contexto/Problema/Alternativas/Análise/Escolha/
> Consequências/Riscos/Trade-offs) está **descrito no `../_DESIGN_BRIEF.md` §11**
> deste módulo, que é a fonte única de verdade até que cada ADR seja escrita como
> arquivo próprio em `../002-ADR/` e tenha seu `Status` promovido de `Proposed`
> para `Accepted`. Nenhum destes arquivos existe fisicamente em `../002-ADR/` no
> momento desta versão do índice; este documento resume o conteúdo pretendido para
> orientar quem for redigi-las.

| ADR | Título | Status | Resumo da decisão (1 linha) | Módulos afetados |
|-----|--------|--------|------------------------------|-------------------|
| ADR-0060 | ABI de Syscalls Cognitivas: 11 verbos, versionamento e compatibilidade | Proposed | Fixar os 11 verbos (`spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`, `invoke_tool`, `route_model`, `get_quota`, `checkpoint`) como superfície estável da ABI, com política de versionamento semântico e compatibilidade retroativa. | 006, 007, 010, 011, 012, 015, 017 |
| ADR-0061 | Modelo canônico do ACB e escolha de OCC sobre locks pessimistas | Proposed | Adotar **concorrência otimista** (campo `version`) como mecanismo primário de controle de concorrência do ACB, evitando locks pessimistas no caminho quente. | 006, 008, 009 |
| ADR-0062 | Token-bucket atômico em Redis para enforcement de cotas com reconciliação em PostgreSQL | Proposed | Enforcement de cota (tokens, cpu-ms, custo, ferramentas, syscalls/s) via *token-bucket* atômico (script Lua) em Redis, com PostgreSQL guardando limites e reconciliação periódica. | 006, 026 |
| ADR-0063 | PEP no Kernel vs. PDP no 022-Policy: fronteira, cache de decisões e fail-closed | Proposed | Kernel é exclusivamente **PEP**; a decisão de autorização é sempre delegada ao PDP (`022-Policy`); cache de decisão com TTL curto; comportamento `fail_mode=closed` por padrão quando o PDP está indisponível. | 006, 021, 022 |
| ADR-0064 | Coordenação de ciclo de vida como saga (Kernel↔Scheduler↔Lifecycle) com compensação | Proposed | Operações de ciclo de vida (`spawn`, `kill`, hibernação) são implementadas como **sagas orquestradas** pelo `LifecycleCoordinator`, com compensação explícita em caso de falha parcial. | 006, 008, 009 |
| ADR-0065 | Estratégia de sharding do ACB (`hash(tenant,agent) mod N`) e política de reparticionamento | Proposed | Particionamento determinístico do ACB por `shard = hash(tenant_id, agent_id) mod N`, com `N` configurável apenas via migração controlada (não recarregável a quente). | 006 |
| ADR-0066 | Outbox transacional + JetStream para entrega exactly-once efetiva de eventos de kernel | Proposed | Todo evento de kernel é publicado via **padrão Outbox** (tabela `kernel.outbox` + relay) para garantir atomicidade entre mutação de estado e publicação, com deduplicação por `event.id` no lado do consumidor. | 006, 020 |
| ADR-0067 | Modelo de hibernação/cold-agent e contrato de checkpoint durável (MinIO) | Proposed | Agentes ociosos migram para `Hibernated` (estado durável em MinIO/PostgreSQL, zero RAM); `checkpoint` produz um `checkpoint_ref` restaurável com meta de *resume* p99 < 250 ms. | 006, 008 |
| ADR-0068 | Brokering governado de recursos (remember/recall/plan/invoke_tool/route_model) vs. acesso direto | Proposed | O Kernel **encaminha** (broker) syscalls de recurso aos módulos donos após capability+cota, nunca executa a lógica de negócio desses módulos diretamente — preserva a fronteira de responsabilidade única. | 006, 010, 011, 012, 015, 017 |
| ADR-0069 | Domínios de código de erro reservados ao Kernel (KERNEL, CAP, SYSCALL) e faixa de QUOTA | Proposed | Reservar os domínios de erro `KERNEL` (0001–0099), `CAP` (0001–0099) e `SYSCALL` (0001–0099) ao Kernel, e a subfaixa `AIOS-QUOTA-0001..0020` (compartilhada com `026-Cost-Optimizer`) para erros de cota emitidos pelo Kernel. | 006, 026 |

### 2.1 Justificativa de cada ADR proposta (contexto resumido)

| ADR | Por que uma ADR (e não um detalhe de implementação)? |
|-----|-------------------------------------------------------|
| ADR-0060 | A ABI é um **contrato público** consumido por `007-Agent-Runtime` e por SDKs externos; mudar os verbos após adoção tem custo de migração alto — decisão arquitetural, não de código. |
| ADR-0061 | OCC vs. locks pessimistas afeta diretamente o throughput-alvo (NFR-004: ≥ 20.000 syscalls/s) e a filosofia de escalonamento horizontal do Kernel. |
| ADR-0062 | A escolha de Redis+Lua vs. alternativas (ex.: rate-limit no PostgreSQL, ou em um serviço dedicado) tem impacto direto em NFR-009 (overshoot ≤ 1%) e é compartilhada com `026-Cost-Optimizer`. |
| ADR-0063 | A fronteira PEP/PDP determina onde a latência de autorização é paga e qual é a postura de segurança sob falha do PDP — decisão de segurança de alto impacto (ver ADR-0008 global). |
| ADR-0064 | Modelar o ciclo de vida como saga (em vez de transação distribuída ou coreografia pura por eventos) é uma escolha estrutural que define como falhas parciais são tratadas. |
| ADR-0065 | Sharding determinístico vs. alternativas (hashing consistente com rebalanceamento automático) afeta a estratégia de escala a 10⁶+ ACBs (NFR-006). |
| ADR-0066 | Outbox vs. alternativas (2PC, transactional outbox de terceiros, CDC) define a garantia de entrega de eventos sob falha (NFR-007: perda de evento = 0). |
| ADR-0067 | O modelo de hibernação é central à economia de recursos do AIOS (agentes majoritariamente *cold*) e ao SLO de *resume* (T-09, p99 < 250 ms). |
| ADR-0068 | Preservar o Kernel como **broker governado** (não executor) é o que mantém a responsabilidade única dos módulos de recurso (010/011/012/015/017) — decisão de fronteira, não de código. |
| ADR-0069 | Reserva de domínios de erro evita colisão entre módulos no catálogo global de erros `AIOS-<DOMINIO>-<NNNN>` (RFC-0001 §5.4). |

---

## 3. Rastreabilidade ADR → Componente → Requisito

| ADR | Componente(s) afetado(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) |
|-----|-------------------------------------------------------|-------------------------------|
| ADR-0060 | `SyscallGateway` | FR-001 |
| ADR-0061 | `AcbStore`, `LifecycleCoordinator` | FR-003, NFR-004, NFR-009 |
| ADR-0062 | `ResourceQuotaManager` | FR-004, NFR-009 |
| ADR-0063 | `CapabilityEnforcer`, `PolicyClient` | FR-002, NFR-003, NFR-010 |
| ADR-0064 | `LifecycleCoordinator`, `SchedulerClient`, `LifecycleClient` | FR-005, NFR-001, NFR-008 |
| ADR-0065 | `AcbStore` | NFR-004, NFR-006 |
| ADR-0066 | `EventEmitter` | FR-007, NFR-007 |
| ADR-0067 | `CheckpointManager`, `LifecycleClient` | FR-009, NFR-001 |
| ADR-0068 | `ResourceBrokerRouter` | FR-008 |
| ADR-0069 | `SyscallGateway`, `CapabilityEnforcer`, `ResourceQuotaManager` | FR-002, FR-004, NFR-010 |

---

## 4. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `ADR-00NN-<slug>.md` em `../002-ADR/` usando
   `../002-ADR/TEMPLATE.md`, preenchendo todas as seções obrigatórias (Contexto,
   Problema, Alternativas, Análise, Escolha, Consequências, Riscos, Trade-offs).
2. O conteúdo **NÃO PODE** contradizer o `../_DESIGN_BRIEF.md` deste módulo; caso
   a análise revele necessidade de divergência, o brief DEVE ser atualizado
   primeiro (o brief é a fonte única de verdade).
3. Revisão pelo Arquiteto-Chefe e pelo(s) módulo(s) afetado(s) listados na
   tabela da §3.
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da ADR e neste
   índice; o `../002-ADR/README.md` global é atualizado com a nova linha.
5. ADRs aceitas **NÃO SÃO** editadas — mudanças de rumo geram uma nova ADR que
   marca a anterior como `Superseded by ADR-XXXX`.

---

## 5. Referências

- Índice global de ADRs: `../002-ADR/README.md`
- Template de ADR: `../002-ADR/TEMPLATE.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §11
- Arquitetura do módulo: `./Architecture.md`
- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim do índice de ADRs do módulo 006-Kernel.*
