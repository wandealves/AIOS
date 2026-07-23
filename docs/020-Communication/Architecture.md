---
Documento: Architecture
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0001, ADR-0004, ADR-0200, ADR-0201, ADR-0202, ADR-0203, ADR-0204, ADR-0205, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: 001-Architecture, 005-Database, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster, 028-Deployment
---

# 020-Communication — Arquitetura do Módulo

Este documento deriva de `./_DESIGN_BRIEF.md` §0, §2, §3 e §10. Onde houver
divergência, o brief prevalece e este documento é o defeito.

---

## 1. Visão C4 — Nível 1: Contexto

```
   ┌──────────────┐   ┌───────────────┐   ┌──────────────┐   ┌─────────────────┐
   │ 006 Kernel   │   │ 009 Scheduler │   │ 010 Memory   │   │ 025 Audit       │
   │ 007 Runtime  │   │ 014 Workflow  │   │ 017 Router   │   │ 026 CostOpt     │
   │ 008 Lifecycle│   │ 022 Policy    │   │ 004 API      │   │ …               │
   └──────┬───────┘   └───────┬───────┘   └──────┬───────┘   └────────┬────────┘
          │ publish/subscribe │ request/reply    │ streams            │ consume
          └───────────────────┴─────────┬────────┴────────────────────┘
                                        ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │              020-COMMUNICATION — Barramento (control + data)             │
   │  aios-comm-svc (controle: registro, streams, A2A, DLQ, contas)           │
   │  NATS cluster (dados: pub/sub, request/reply, JetStream)                 │
   └───┬──────────┬───────────┬────────────┬───────────┬──────────┬──────────┘
       │          │           │            │           │          │
       ▼          ▼           ▼            ▼           ▼          ▼
   021-Security 022-Policy 024-Observ.  025-Audit  027-Cluster 005-Database
   (NKey/JWT,   (PDP)      (OTel)       (trilha)   (topologia) (schema comm)
    mTLS)

   ┌──────────────────────────────────────────────────────────────────────────┐
   │ AGENTES (via 007-Agent-Runtime) — canais de grupo e sessões A2A          │
   │ PARES A2A EXTERNOS — federação sob mTLS, quando habilitada por tenant    │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Leitura:** há **dois planos** distintos, e confundi-los é o erro de arquitetura mais
provável neste módulo. O **plano de controle** (`aios-comm-svc`) registra subjects,
declara streams, autoriza sessões A2A e opera a DLQ. O **plano de dados** é o próprio
**NATS**: produtores e consumidores falam com o broker **diretamente**. Colocar o
serviço 020 no caminho de cada mensagem anularia a razão de escolher NATS (ADR-0004) —
seriam dois saltos de rede e um gargalo central onde hoje há um broker distribuído.

---

## 2. Visão C4 — Nível 2: Contêineres

| Contêiner | Tecnologia | Papel | Réplicas |
|-----------|-----------|-------|----------|
| `aios-comm-svc` | .NET 10 | Plano de controle: registro de subjects, declaração de streams/consumidores, A2A, DLQ, contas, replay. **Stateless**. | 2 |
| `aios-nats` | NATS 2.x + JetStream | Plano de dados: roteamento, streams duráveis (R=3), contas por tenant. | 3 (quórum) |
| `aios-nats-leaf` | NATS leaf node | Extensão do barramento a outra região/borda, quando habilitado por `027`. | 0..N |
| PostgreSQL (`comm`) | PG16 (via `005`) | Declaração de subjects/streams/consumidores, sessões A2A, DLQ. | externo |
| Redis | Redis 7 (via `028`) | Contadores de cota do `FlowController`. | externo |

```
   produtores/consumidores ─────── NATS client (NKey/JWT + TLS) ────┐
                                                                     ▼
                              ┌──────────────────────────────────────────────┐
                              │        aios-nats × 3  (JetStream R=3)        │
                              │  conta acme │ conta globex │ conta _platform │
                              └───────┬──────────────────────────┬───────────┘
                                      │ admin (declarações)      │ leaf/gateway
   ┌──────────────────┐  gRPC/REST    │                          ▼
   │  aios-comm-svc×2 │───────────────┘                    aios-nats-leaf
   │  (plano controle)│──▶ PostgreSQL (schema comm)              (027)
   └──────────────────┘──▶ Redis (contadores de cota)
```

---

## 3. Visão C4 — Nível 3: Componentes

Os componentes internos e suas responsabilidades são normativos no
`./_DESIGN_BRIEF.md` §2.1. Resumo com a fronteira de cada um:

| Componente | Responsabilidade sintética | Fronteira (o que NÃO faz) |
|------------|---------------------------|---------------------------|
| `SubjectRegistry` | Catálogo de subjects, produtor declarado, `dataschema`. | Não define a semântica do evento. |
| `EnvelopeValidator` | Valida CloudEvents no publish. | Não interpreta o `data`. |
| `PublishGateway` | Caminho de publicação validado, com cota e correlação. | Não é proxy do tráfego normal — expõe o contrato que o relay do produtor usa. |
| `StreamManager` | Ciclo de vida declarativo de streams. | Não decide retenção de negócio (o dono declara). |
| `ConsumerManager` | Consumidores duráveis, ack, backoff, `max_deliver`. | Não processa a mensagem. |
| `SubscriptionBroker` | Assinaturas e curingas dentro do escopo autorizado. | Não filtra por conteúdo de negócio. |
| `RequestReplyBroker` | Request/reply com timeout e correlação. | Não implementa a lógica do respondente. |
| `A2AGateway` | Sessões A2A: handshake, negociação, FSM, trilha. | Não decide se a colaboração faz sentido (é do agente). |
| `SelectiveCommunicationRouter` | Canais de grupo, fan-out, escopo. | Não define a composição do *Agent Group* (`008`/`009`). |
| `FlowController` | Cotas, backpressure, *slow consumer*. | Não define orçamento (é do `026`). |
| `DeadLetterManager` | Quarentena, inspeção, replay controlado. | Não corrige a mensagem. |
| `ReplayService` | Replay por tempo/sequência em consumidor isolado. | Não reescreve o stream. |
| `TenantAccountManager` | Contas NATS, credenciais, exports/imports. | Não emite credencial (é do `021`). |
| `BridgeConnector` | Leaf nodes e gateways entre clusters. | Não decide topologia (é do `027`). |
| `PolicyClient` | PEP → PDP, *fail-closed*. | Não decide (é PDP do `022`). |
| `BusTelemetry` | OTel + auditoria. | Não armazena a trilha (é do `025`). |

O diagrama completo é o do `./_DESIGN_BRIEF.md` §2.2, reproduzido com interfaces em
`./ClassDiagrams.md` §2.

---

## 4. Padrões Arquiteturais Adotados

| Padrão | Onde | Por quê | ADR |
|--------|------|---------|-----|
| **Barramento pub/sub com subjects hierárquicos** | Namespace inteiro | Filtro no broker (não no consumidor); assinatura no nível exato de interesse. | ADR-0004 |
| **Registro de subjects** | `comm.subject_registry` | Torna o catálogo do barramento verdadeiro e impede evento órfão ou forjado. | ADR-0200 |
| **Conta NATS por tenant** | Isolamento | Fronteira aplicada pelo roteamento, não por convenção de nome. | ADR-0201 |
| **Validação de envelope no publish** | `EnvelopeValidator` | Mensagem malformada nunca entra no stream nem contamina consumidores. | ADR-0202 |
| **Streams declarativos versionados** | `StreamManager` | Configuração é código revisado; mudança incompatível migra sem perda. | ADR-0203 |
| **Um stream por domínio de fluxo** | Organização de streams | Mantém o número de streams em O(domínios), não O(tenants). | ADR-0204 |
| **DLQ com replay explícito** | `DeadLetterManager` | Nada some em silêncio; replay automático repetiria o defeito. | ADR-0205 |
| **Comunicação seletiva por grupo** | `SelectiveCommunicationRouter` | Evita tráfego O(N²) com 10⁶ agentes. | ADR-0206 |
| **Sessão A2A com FSM** | `A2AGateway` | Colaboração entre agentes vira recurso governado, autorizado e cotado. | ADR-0207 |
| **PEP → PDP com *fail-closed* no controle** | `PolicyClient` | *Default deny* (RFC-0001 §5.8) sem derrubar o tráfego já autorizado. | ADR-0209 |
| **Outbox no produtor** | Contrato de publicação | Atomicidade entre mudança de estado e publicação. | herdado de ADR-0006 |

---

## 5. Decisões Tecnológicas e Alternativas Descartadas

| Escolha | Alternativas consideradas | Por que a escolha | Trade-off aceito |
|---------|---------------------------|-------------------|------------------|
| **NATS + JetStream** | Kafka; RabbitMQ; Pulsar | Latência de micro/milissegundos no core pub/sub, request/reply nativo, subjects hierárquicos com curinga, operação leve (binário único), contas multi-tenant de primeira classe. O AIOS é dominado por **tráfego de controle pequeno e sensível a latência** — o perfil oposto ao que justifica Kafka. | Menor ecossistema de conectores e menor tradição em retenção de longuíssimo prazo; mitigado porque retenção longa é de `025`/`005`. |
| **Conta por tenant** | Prefixo de subject + autorização por permissão | Isolamento entra no roteamento: outra conta é invisível, não apenas proibida. | Provisionamento de conta vira passo obrigatório do onboarding de tenant. |
| **Um stream por domínio** | Um stream por tenant | 10⁴ tenants × N domínios seria inoperável (limites de arquivo, memória e observabilidade). | Retenção é por domínio, não por cliente; expurgo por tenant depende de `005`. |
| **Validação no publish** | Validação apenas no consumidor | Falha aparece no autor do defeito, não em N consumidores; o stream permanece confiável. | Custo de CPU no caminho de publicação (mitigado por cache de schema). |
| **Replay explícito** | `auto_replay` ligado | Replay cego reprocessa a mesma mensagem envenenada e reabre o mesmo incidente. | Exige intervenção humana ou automação deliberada. |
| **Limite de payload (1 MiB)** | Payload livre | Mensagens grandes destroem a latência de todos os demais e incham streams. | Produtores precisam do padrão "referência a objeto" (ADR-0208). |

---

## 6. Fluxo Arquitetural Principal — Publicação e Entrega

```
 Produtor(006)  Outbox(PG)  Relay  NATS/JetStream  Consumidor(025)  DLQ
     │             │          │          │               │           │
     ├─ BEGIN; muda estado; INSERT outbox; COMMIT ▶      │           │
     │             │          │          │               │           │
     │             │◀─ poll ──┤          │               │           │
     │             │          ├─ publish (envelope CloudEvents) ─────▶│
     │             │          │   [EnvelopeValidator: campos, type, dataschema]
     │             │          │   [SubjectRegistry: produtor autorizado?]
     │             │          │   [FlowController: cota do tenant?]
     │             │          │◀── ack (seq) ─┤          │           │
     │             │◀─ marca published=true ──┤          │           │
     │             │          │          ├─ deliver ────▶│           │
     │             │          │          │◀── ack ───────┤           │
     │             │          │          │               │           │
     │             │          │          │  (sem ack em ack_wait → reentrega c/ backoff)
     │             │          │          ├─ tentativa 5 ─▶│ (falha)   │
     │             │          │          ├──────────── max_deliver esgotado ──▶│
     │             │          │          │            evento comm.delivery.deadlettered
```

Os fluxos de falha (subject não registrado, cota excedida, *slow consumer*, perda de
quórum) estão em `./SequenceDiagrams.md` §4–§7.

---

## 7. Fronteiras de Confiança

```
   ┌─── Zona externa ────────────────────────────────────────────────────┐
   │ Pares A2A externos (outra organização)                              │
   └──────────────────────────┬──────────────────────────────────────────┘
                              │ mTLS + identidade federada + PDP por capacidade
   ┌──────────────────────────▼──────────────────────────────────────────┐
   │ Zona de serviços (malha interna, mTLS)                              │
   │  aios-comm-svc · módulos produtores/consumidores                    │
   └──────────────────────────┬──────────────────────────────────────────┘
                              │ NKey/JWT por conta + TLS
   ┌──────────────────────────▼──────────────────────────────────────────┐
   │ NATS: conta acme │ conta globex │ conta _platform                   │
   │  ← fronteira de roteamento: uma conta não enxerga subjects da outra │
   └─────────────────────────────────────────────────────────────────────┘
```

Detalhamento em `./Security.md`.

---

## 8. Atributos de Qualidade Arquiteturalmente Significativos

| Atributo | Decisão arquitetural que o sustenta | NFR |
|----------|-------------------------------------|-----|
| Latência | Core pub/sub sem persistência no caminho quente; serviço de controle fora do caminho de mensagem. | NFR-001, NFR-003 |
| Durabilidade | JetStream R=3 com quórum; Outbox no produtor. | NFR-006, NFR-009 |
| Isolamento | Conta por tenant; *export* explícito e auditado. | NFR-010 |
| Escala de coordenação | Subjects hierárquicos + canais de grupo + limite de fan-out. | NFR-007, NFR-016 |
| Contenção | Cotas, `max_ack_pending`, `discard policy`, limite de payload. | NFR-011 |
| Confiabilidade operacional | DLQ inspecionável + replay controlado. | NFR-012 |
| Observabilidade | `traceparent` em toda mensagem; métricas `aios_bus_*`. | NFR-013 |

---

## 9. Riscos Arquiteturais

| Risco | Impacto | Mitigação |
|-------|---------|-----------|
| Perda de quórum JetStream | Publicação durável indisponível | R=3 em zonas distintas; core pub/sub segue operando; produtores retêm no Outbox. |
| Explosão de subjects (cardinalidade) | Degradação de roteamento e observabilidade | Registro obrigatório; subject por **tipo de evento**, nunca por instância (sem `…agent.<ULID>.status`). |
| Barramento usado como banco | Retenção crescente, custo, expectativa errada | `max_age` explícito por stream; FAQ e revisão de declaração de stream. |
| Sessão A2A como canal paralelo não governado | Fuga da governança do Kernel | Capacidades autorizadas item a item pelo PDP; sessões cotadas, auditadas e com timeout de ociosidade. |
| Dependência do PDP no caminho de sessão | Falha de política impede colaboração | *Fail-closed* apenas para **novas** sessões; tráfego autorizado continua (P-09). |
| Leaf nodes replicando tráfego local | Custo de rede entre regiões | Filtros de propagação no `BridgeConnector`, conforme `../027-Cluster/`. |

---

## 10. Referências

- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Brief do módulo: `./_DESIGN_BRIEF.md`
- Eventos: `./Events.md` · API de controle: `./API.md` · FSM: `./StateMachine.md`
- Escala: `./Scalability.md` · Falhas: `./FailureRecovery.md` · Segurança: `./Security.md`
- Decisões: `./ADR.md` · Especificações: `./RFC.md`
