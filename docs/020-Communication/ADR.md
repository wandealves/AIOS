---
Documento: ADR
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0006, ADR-0008, ADR-0010; ADR-0200..ADR-0209
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: 002-ADR, _DESIGN_BRIEF.md §11
---

# 020-Communication — Índice de ADRs

Faixa reservada a este módulo: **`ADR-0200`..`ADR-0209`** (regra `NNN × 10`). As ADRs
são registradas em `../002-ADR/` — este documento é o índice com o impacto de cada
decisão sobre o módulo.

---

## 1. ADRs Globais Herdadas

| ADR | Decisão | Impacto no 020-Communication | Status |
|-----|---------|------------------------------|--------|
| ADR-0001 | AIOS é um SO, não um framework | O barramento é subsistema de IPC do SO, não biblioteca de mensageria de aplicação. | Accepted |
| ADR-0002 | Microserviços + split control/data plane | Comunicação leste-oeste entre serviços é responsabilidade deste módulo. | Accepted |
| **ADR-0004** | **NATS + JetStream como barramento primário** | **Decisão fundadora deste módulo**: define o broker, o modelo de subjects, o request/reply nativo e as contas multi-tenant. | Accepted |
| ADR-0006 | Redis para estado quente | Contadores de cota do `FlowController` vivem em Redis. | Accepted |
| ADR-0008 | Governança por política, *default deny* | Operações administrativas e sessões A2A passam pelo PDP. | Accepted |
| ADR-0010 | Observabilidade e auditoria por construção | `traceparent` em toda mensagem; métricas `aios_bus_*`; trilha de sessões A2A. | Accepted |

---

## 2. ADRs a Propor por Este Módulo

| ADR | Título | Escopo da decisão | Documentos afetados | Status |
|-----|--------|-------------------|---------------------|--------|
| ADR-0200 | Registro obrigatório de subjects e criação do domínio de eventos `comm` | Catálogo `comm.subject_registry` como fonte da verdade do namespace; produtor declarado por subject; registro do domínio `comm` conforme RFC-0001 §8. | `Events.md`, `Database.md`, `Security.md` | Proposed |
| ADR-0201 | Conta NATS por tenant como fronteira de isolamento | Conta por tenant vs. prefixo de subject com permissões; implicações de provisionamento e de *export/import*. | `Security.md`, `Deployment.md`, `Scalability.md` | Proposed |
| ADR-0202 | Validação do envelope CloudEvents no *publish* | Validar na entrada vs. no consumidor; custo de CPU vs. confiabilidade do stream. | `Architecture.md`, `Events.md`, `API.md` | Proposed |
| ADR-0203 | Streams declarativos versionados e migração sem perda | Declaração como código; mudança compatível em lugar; incompatível por migração espelhada. | `StateMachine.md`, `Deployment.md`, `Testing.md` | Proposed |
| ADR-0204 | Um stream por domínio de fluxo, não por tenant | 8 streams vs. 80.000; onde reside efetivamente o isolamento. | `Scalability.md`, `Database.md` | Proposed |
| ADR-0205 | DLQ com replay explícito e `auto_replay` desligado | Nada some em silêncio; replay cego reproduz o defeito. | `FailureRecovery.md`, `UseCases.md`, `Configuration.md` | Proposed |
| ADR-0206 | Comunicação seletiva por canais de *Agent Group* com limite de fan-out | Difusão a custo O(1) para o produtor; contenção do tráfego quadrático. | `Scalability.md`, `Benchmark.md`, `Metrics.md` | Proposed |
| ADR-0207 | Perfil A2A do AIOS: sessões com FSM e negociação de capacidades | Colaboração entre agentes como recurso governado, cotado e auditado; suporte a pares externos. | `StateMachine.md`, `Security.md`, `API.md` | Proposed |
| ADR-0208 | Limite de payload e padrão "referência a objeto" | 1 MiB como teto; blobs em MinIO com ponteiro na mensagem. | `Configuration.md`, `Benchmark.md`, `FAQ.md` | Proposed |
| ADR-0209 | Domínios de erro (`BUS`, `A2A`) e política de *fail-closed* seletivo | *Fail-closed* para operações novas; tráfego autorizado continua sob PDP indisponível. | `API.md`, `Security.md`, `FailureRecovery.md` | Proposed |

---

## 3. Decisões e Alternativas Descartadas (resumo)

| ADR | Escolha | Alternativas descartadas | Trade-off aceito |
|-----|---------|--------------------------|------------------|
| ADR-0004 (herdada) | NATS + JetStream | Kafka; RabbitMQ; Pulsar | Ecossistema de conectores menor; não é plataforma de retenção longa. |
| ADR-0201 | Conta por tenant | Prefixo de subject com permissões | Provisionamento de conta vira passo obrigatório do onboarding. |
| ADR-0202 | Validar no publish | Validar no consumidor; não validar | Custo de CPU no caminho de publicação (mitigado por cache de schema). |
| ADR-0204 | Stream por domínio | Stream por tenant | Retenção uniforme por domínio; expurgo por titular fica com `../005-Database/`. |
| ADR-0205 | Replay explícito | `auto_replay` ligado | Exige intervenção humana ou automação deliberada. |
| ADR-0206 | Canais de grupo | Iteração de destinatários no produtor | Composição do grupo depende de `008`/`009`; o barramento só resolve a comunicação. |
| ADR-0207 | Sessão A2A com FSM | Canal livre entre agentes | Overhead de handshake e de negociação por capacidade. |
| ADR-0208 | Teto de 1 MiB | Payload livre | Casos com dados grandes exigem padrão de referência a objeto. |
| ADR-0209 | *Fail-closed* seletivo | *Fail-closed* total; *fail-open* | Tráfego autorizado segue mesmo com PDP fora — decisão consciente de disponibilidade. |

---

## 4. Processo

1. Uma decisão arquitetural do módulo **NÃO DEVE** ser tomada dentro de um dos 26
   documentos: ela é registrada como ADR em `../002-ADR/` e apenas **referenciada** aqui.
2. Novas ADRs deste módulo usam a próxima numeração livre dentro de
   `ADR-0200`..`ADR-0209`. Esgotada a faixa, a extensão é decidida com a arquitetura-chefe.
3. Uma ADR que altere contrato central exige também **RFC** (ver `./RFC.md`).
4. Ao mudar o status de uma ADR, atualize esta tabela e o índice em
   `../002-ADR/README.md`.

---

## 5. Referências

- Índice global de decisões: `../002-ADR/README.md`
- Brief (origem das propostas): `./_DESIGN_BRIEF.md` §11
- RFCs do módulo: `./RFC.md`
