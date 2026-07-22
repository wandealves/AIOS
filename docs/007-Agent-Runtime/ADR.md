---
Documento: ADR.md
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: ../001-Architecture/, ../002-ADR/, ../008-Agent-Lifecycle/, ../021-Security/, ../022-Policy/
---

# 007-Agent-Runtime — Índice de ADRs

> **Escopo deste documento.** Este documento **NÃO** decide nada por si — é
> um **índice navegável** das *Architecture Decision Records* (ADR) que
> afetam o módulo 007-Agent-Runtime. As decisões em si residem em
> `../002-ADR/`, no formato definido por `../002-ADR/TEMPLATE.md` (Contexto ·
> Problema · Alternativas · Análise · Escolha · Consequências · Riscos ·
> Trade-offs). Este arquivo **DEVE** ser mantido sincronizado com
> `../_DESIGN_BRIEF.md` §11 e com `../002-ADR/README.md`; qualquer
> divergência é um defeito de documentação.

> **Convenção de numeração.** Conforme `./_DESIGN_BRIEF.md` §11, a faixa
> `ADR-0070`..`ADR-0079` está **reservada** ao módulo 007-Agent-Runtime
> (regra `módulo × 10`, evitando colisão com faixas de outros módulos —
> ex.: `006-Kernel` usa `ADR-0060`..`ADR-0069`, `009-Scheduler` usaria
> `ADR-0090`..`ADR-0099`). Nenhuma outra numeração PODE ser usada por este
> módulo sem atualizar este índice e `../002-ADR/README.md`.

---

## 1. ADRs Globais que Restringem o Agent Runtime

Estas ADRs foram decididas na fundação canônica do AIOS (`../002-ADR/`) e
**DEVEM** ser respeitadas por qualquer ADR específica do Agent Runtime —
elas não são redecididas aqui, apenas referenciadas.

| ADR | Título | Status | Como restringe o Agent Runtime |
|-----|--------|--------|-------------------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O Agent Runtime é o **espaço de execução de processo cognitivo** de um SO próprio — analogia direta a "processo do sistema operacional" (`./Vision.md` §1). |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | O Agent Runtime vive inteiramente no **plano de dados**; nunca decide governança, scheduling ou ciclo de vida de pool — isso é do control plane (`006`, `008`, `009`, `022`). |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | O Agent Runtime é implementado em **Python 3.12+/asyncio**; comunica-se com o Runtime Supervisor (.NET 10) exclusivamente via `aios.runtime.v1` (gRPC/NATS), nunca por acoplamento de processo. |

---

## 2. ADRs Específicas do Módulo 007-Agent-Runtime (Propostas)

> **Estado de redação.** As ADRs abaixo estão **reservadas** na faixa
> `ADR-0070`–`ADR-0079` e seu conteúdo (Contexto/Problema/Alternativas/
> Análise/Escolha/Consequências/Riscos/Trade-offs) está **descrito no
> `./_DESIGN_BRIEF.md` §11** deste módulo, que é a fonte única de verdade
> até que cada ADR seja escrita como arquivo próprio em `../002-ADR/` e
> tenha seu `Status` promovido de `Proposed` para `Accepted`. Nenhum destes
> arquivos existe fisicamente em `../002-ADR/` no momento desta versão do
> índice; este documento resume o conteúdo pretendido para orientar quem
> for redigi-las.

| ADR | Título | Status | Resumo da decisão (1 linha) | Módulos afetados |
|-----|--------|--------|--------------------------------|----------------------|
| ADR-0070 | Runtime como processo Python por agente vs. threads/*workers* compartilhados | Proposed | Adotar **isolamento forte de processo** (um agente por processo) em vez de threads/*workers* compartilhados, priorizando contenção de falha sobre economia de recurso. | 007, 008, 009 |
| ADR-0071 | Mecanismo de sandbox: `seccomp-bpf` + `cgroups v2` + namespaces vs. gVisor/microVM (Firecracker) | Proposed | Adotar namespaces+`seccomp-bpf`+`cgroups v2` como perfil-**default** de isolamento, mantendo microVM como opção por tenant regulado. | 007, 021 |
| ADR-0072 | Protocolo de controle Runtime↔Supervisor: gRPC (comando) + NATS (heartbeat/eventos) | Proposed | Fixar `aios.runtime.v1` (`RuntimeControl`/`AgentExecution`) sobre gRPC para comando e NATS para heartbeat/eventos, com paridade de contrato entre Supervisor e control plane. | 007, 008, 020 |
| ADR-0073 | Estratégia de cold start: pool quente pré-forkado + *copy-on-write* de intérprete | Proposed | Manter um **pool quente** (`warm_size`) de processos pré-forkados com intérprete *copy-on-write* para atingir p99 ≤ 250 ms de boot. | 007, 008 |
| ADR-0074 | Modelo de checkpoint/hibernação: serialização de working memory + cursor em MinIO | Proposed | Serializar working memory + `step_cursor` + plano em blob MinIO (via control plane) com checksum sha256, habilitando *cold agent* verdadeiro. | 007, 010 |
| ADR-0075 | MCP host embarcado no sandbox vs. *sidecar* externo | Proposed | Embarcar o `McpHost` **dentro** do sandbox do processo do agente, evitando travessia de rede/IPC adicional por chamada de ferramenta. | 007, 015 |
| ADR-0076 | Outbox transacional local (SQLite/WAL efêmero) para atomicidade estado↔evento | Proposed | Usar outbox SQLite/WAL **local ao processo** (não um banco durável compartilhado) para garantir que nenhum evento seja publicado sem a transição de estado correspondente. | 007, 020 |
| ADR-0077 | Loop ReAct assíncrono (`asyncio`) e limites de passos/profundidade/fan-out | Proposed | Adotar `asyncio` não-bloqueante para o `CognitiveLoopEngine`, com limites explícitos de passos/profundidade/fan-out como controle estrutural anti-loop-improdutivo. | 007, 012 |
| ADR-0078 | PEP local com cache de *capabilities* e invalidação por evento de política | Proposed | `CapabilityEnforcer` cacheia decisões com TTL curto e invalida via `policy.decision.updated`, equilibrando latência do caminho quente com atualidade de política. | 007, 022 |
| ADR-0079 | *Lease* distribuído por sessão (Redis) para exatamente-um-ativo em resume | Proposed | Usar *lease* Redis por `session_id`, adquirido obrigatoriamente antes de `Resume`, para eliminar dupla execução em cenários de *split-brain*. | 007, 008 |

### 2.1 Justificativa de Cada ADR Proposta (Contexto Resumido)

| ADR | Por que uma ADR (e não um detalhe de implementação)? |
|-----|------------------------------------------------------|
| ADR-0070 | Define a **unidade de isolamento** de todo o sistema de execução de agentes — decisão estrutural com impacto direto em segurança, escala e custo, não um detalhe de código. |
| ADR-0071 | Escolha de mecanismo de sandbox afeta diretamente NFR-001 (cold start) e NFR-008 (segurança) — *trade-off* explícito entre desempenho e isolamento máximo. |
| ADR-0072 | O protocolo Runtime↔Supervisor é um **contrato entre dois planos** (dados/controle) mantidos por equipes potencialmente distintas — exige processo de RFC formal (RFC-0070), não apenas ADR. |
| ADR-0073 | A estratégia de cold start determina se a meta agressiva de NFR-001 é sequer atingível — decisão de plataforma, não de código de aplicação. |
| ADR-0074 | O modelo de checkpoint é o pré-requisito estrutural da meta de escala V-R1 (10⁶+ agentes) — mudar o formato de serialização depois de adotado tem custo de migração alto. |
| ADR-0075 | Embarcar vs. sidecar MCP afeta a superfície de ataque e a latência de toda invocação de ferramenta — decisão de fronteira de segurança. |
| ADR-0076 | A escolha de outbox local vs. alternativas (fila em memória, acesso direto a banco durável) define a garantia de RPO (NFR-006) sob crash. |
| ADR-0077 | Limites de passos/profundidade/fan-out são a defesa estrutural contra P2 (planejamento frágil) da Visão Global — decisão de governança cognitiva, não só de engenharia. |
| ADR-0078 | O ponto de equilíbrio entre cache local (latência) e atualidade de política (segurança) é uma decisão de *trade-off* explícito revisável. |
| ADR-0079 | Prevenção de dupla execução é uma garantia de correção crítica (RSK-A05) — a escolha de mecanismo (lease vs. lock distribuído vs. consenso) tem impacto de disponibilidade. |

---

## 3. Rastreabilidade ADR → Componente → Requisito

| ADR | Componente(s) afetado(s) (ver `./Architecture.md` §5) | Requisito(s) relacionado(s) |
|-----|-----------------------------------------------------------|----------------------------------|
| ADR-0070 | `RuntimeBootstrapper`, `ExecutionStateMachine` | FR-001, NFR-008, NFR-012 |
| ADR-0071 | `SandboxManager` | FR-007, NFR-002, NFR-008 |
| ADR-0072 | `SupervisorChannel` | FR-018, NFR-014 |
| ADR-0073 | `RuntimeBootstrapper` | FR-001, NFR-001 |
| ADR-0074 | `CheckpointManager` | FR-009, NFR-006, NFR-010 |
| ADR-0075 | `McpHost`, `ToolInvoker` | FR-004 |
| ADR-0076 | `EventPublisher` | FR-011, NFR-007 |
| ADR-0077 | `CognitiveLoopEngine` | FR-002, FR-017 |
| ADR-0078 | `CapabilityEnforcer` | FR-008, NFR-008 |
| ADR-0079 | `CheckpointManager`, `SupervisorChannel` | FR-009, NFR-005, NFR-006 |

---

## 4. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `ADR-00NN-<slug>.md` em `../002-ADR/` usando
   `../002-ADR/TEMPLATE.md`, preenchendo todas as seções obrigatórias
   (Contexto, Problema, Alternativas, Análise, Escolha, Consequências,
   Riscos, Trade-offs).
2. O conteúdo **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste módulo;
   caso a análise revele necessidade de divergência, o brief DEVE ser
   atualizado primeiro (o brief é a fonte única de verdade).
3. Revisão pelo Arquiteto-Chefe e pelo(s) módulo(s) afetado(s) listados na
   tabela da §3.
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da ADR e neste
   índice; o `../002-ADR/README.md` global é atualizado com a nova linha.
5. ADRs aceitas **NÃO SÃO** editadas — mudanças de rumo geram uma nova ADR
   que marca a anterior como `Superseded by ADR-XXXX`.

---

## 5. Referências

- Índice global de ADRs: `../002-ADR/README.md`
- Template de ADR: `../002-ADR/TEMPLATE.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §11
- Arquitetura do módulo: `./Architecture.md`
- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim do índice de ADRs do módulo 007-Agent-Runtime.*
