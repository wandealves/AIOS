---
Documento: ClassDiagrams
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0201, ADR-0202, ADR-0203, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: _DESIGN_BRIEF.md §2, Architecture.md, StateMachine.md, Database.md
---

# 020-Communication — Estruturas, Interfaces e Contratos

Diagramas ASCII das estruturas internas do `aios-comm-svc` (.NET 10). Os nomes de
componente são exatamente os de `./_DESIGN_BRIEF.md` §2.1 e `./Architecture.md` §3.

---

## 1. Visão de Pacotes

```
   Aios.Comm.Api          → controllers REST + serviços gRPC (aios.comm.v1)
   Aios.Comm.Registry     → SubjectRegistry, EnvelopeValidator
   Aios.Comm.Streaming    → StreamManager, ConsumerManager, ReplayService
   Aios.Comm.Delivery     → PublishGateway, SubscriptionBroker, RequestReplyBroker,
                            DeadLetterManager
   Aios.Comm.Fabric       → TenantAccountManager, BridgeConnector, FlowController
   Aios.Comm.A2A          → A2AGateway, SessionStore, CapabilityNegotiator
   Aios.Comm.Group        → SelectiveCommunicationRouter
   Aios.Comm.Platform     → PolicyClient, EventEmitter, BusTelemetry
```

Regra de dependência: `Api` → (`Registry`, `Streaming`, `Delivery`, `Fabric`, `A2A`,
`Group`) → `Platform`. **Nenhum** pacote de domínio depende de `Api`, e `Platform`
**NÃO DEVE** depender de outro pacote (evita ciclo).

---

## 2. Diagrama de Componentes e Relações

```
  ┌────────────────────────────┐        ┌──────────────────────────────┐
  │      SubjectRegistry       │◀──────▶│      EnvelopeValidator       │
  │────────────────────────────│        │──────────────────────────────│
  │ + Register(def): SubjectDef│        │ + Validate(env, subj): Report│
  │ + Resolve(subject): Def?   │        │ - rules: IReadOnlyList<IRule>│
  │ + IsProducer(mod, subj)    │        └──────────────┬───────────────┘
  │ + Deprecate(urn)           │                       △ (implementa)
  └──────────┬─────────────────┘        ┌──────────────┴──────────────┐
             │                          │ RequiredFieldsRule          │
             │                          │ TypeMatchesSubjectRule      │
             │                          │ TenantMatchesAccountRule    │
             │                          │ DataschemaRegisteredRule    │
             │                          │ PayloadSizeRule             │
             │                          │ TraceparentPresentRule      │
             │                          └─────────────────────────────┘
             ▼
  ┌────────────────────┐  cota  ┌──────────────────┐   ┌─────────────────────┐
  │  PublishGateway    │───────▶│  FlowController  │──▶│ TenantAccountManager│
  │ + PublishAsync(env)│        │ + Charge(t,a,n)  │   │ + Provision(tenant) │
  └─────────┬──────────┘        │ + IsStalled(cons)│   │ + PutExport(...)    │
            │                   └──────────────────┘   │ + Revoke(cred)      │
            ▼                                          └─────────────────────┘
  ┌────────────────────┐        ┌──────────────────────┐
  │   StreamManager    │───────▶│   ConsumerManager    │
  │ + Put(def): Stream │        │ + Put(def): Consumer │
  │ + Drain(name)      │        │ + Status(name)       │
  │ + Migrate(old,new) │        └──────────┬───────────┘
  └─────────┬──────────┘                   │
            │                              ▼
            │                   ┌──────────────────────┐
            ├──────────────────▶│  DeadLetterManager   │
            │                   │ + Quarantine(msg)    │
            ▼                   │ + Replay(id)         │
  ┌────────────────────┐        │ + Discard(id, why)   │
  │   ReplayService    │        └──────────────────────┘
  │ + ReplayStream(...)│
  └────────────────────┘

  ┌────────────────────┐        ┌──────────────────────┐   ┌───────────────────┐
  │ SubscriptionBroker │───────▶│ RequestReplyBroker   │   │ SelectiveComm.    │
  │ + Subscribe(subj)  │        │ + Request(subj,body) │   │ Router            │
  │ + Unsubscribe(id)  │        │ + Respond(replyTo)   │   │ + Broadcast(grp,m)│
  └────────────────────┘        └──────────────────────┘   │ + Resolve(grp)    │
                                                            └───────────────────┘
  ┌──────────────────────────────────────────┐
  │              A2AGateway                  │  ┌──────────────────┐
  │──────────────────────────────────────────│─▶│ CapabilityNegot. │
  │ + OpenAsync(req, key)   : A2ASession     │  │ + Negotiate(...) │
  │ + CloseAsync(urn, why)  : A2ASession     │  └──────────────────┘
  │ + RelayAsync(urn, msg)  : void           │  ┌──────────────────┐
  │ - fsm: A2ASessionStateMachine (§4)       │─▶│  SessionStore    │
  └──────────────────┬───────────────────────┘  └──────────────────┘
                     ▼
  ┌───────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
  │ PolicyClient  │   │ EventEmitter │   │ BusTelemetry │   │ BridgeConnector  │
  │ + Decide(req) │   │ + Enqueue(e) │   │ + Span(name) │   │ + Configure(topo)│
  └───────────────┘   └──────────────┘   └──────────────┘   └──────────────────┘
```

Relações: linha cheia com `▶` = dependência de uso; `△` = implementação de interface.

---

## 3. Contratos (interfaces públicas do serviço)

```
  interface ISubjectRegistry
  ────────────────────────────────────────────────────────────────────────────
  + RegisterAsync(SubjectDefinition, IdempotencyKey) : SubjectDefinition
  + ResolveAsync(subject)                            : SubjectDefinition?
  + IsProducerAuthorized(module, subject)            : bool
  + DeprecateAsync(Urn, Reason)                      : SubjectDefinition
  Invariantes: subject único por (domain, entity, action); um produtor por subject;
               forma conforme RFC-0001 §5.3 (nunca subject por instância)

  interface IStreamManager
  ────────────────────────────────────────────────────────────────────────────
  + PutAsync(StreamDefinition)      : StreamDefinition   // idempotente
  + DrainAsync(name)                : StreamDefinition   // active → draining
  + ListAsync(filter)               : StreamDefinition[]
  Invariante: mudança incompatível → migração espelhada, nunca recriação destrutiva

  interface IConsumerManager
  ────────────────────────────────────────────────────────────────────────────
  + PutAsync(ConsumerDefinition)    : ConsumerDefinition
  + StatusAsync(stream, durable)    : ConsumerStatus     // pending, redelivered, lag
  Invariante: ack_policy = explicit em tráfego de controle; backoff crescente

  interface IDeadLetterManager
  ────────────────────────────────────────────────────────────────────────────
  + ListAsync(filter)               : DeadLetterEntry[]
  + ReplayAsync(id, IdempotencyKey) : ReplayOutcome      // exige capability
  + DiscardAsync(id, Reason)        : DeadLetterEntry    // justificativa obrigatória
  Invariante: nenhuma transição automática quarantined → replayed (ADR-0205)

  interface IA2AGateway
  ────────────────────────────────────────────────────────────────────────────
  + OpenAsync(OpenSessionRequest, IdempotencyKey) : A2ASession   // T-01
  + GetAsync(Urn)                                 : A2ASession
  + CloseAsync(Urn, Reason)                       : A2ASession   // T-08 → T-09
  + RelayAsync(Urn, Message)                      : void
  Invariantes: I1 (canal exclusivo), I2 (só trafega em Established),
               I4 (capacidades imutáveis após Established)

  interface ITenantAccountManager
  ────────────────────────────────────────────────────────────────────────────
  + ProvisionAsync(tenant, IdempotencyKey) : AccountInfo
  + PutExportAsync(ExportSpec)             : ExportSpec   // alta sensibilidade
  + RevokeAsync(credentialRef)             : void
  Invariante: nenhuma conta é criada sem credencial gerenciada pelo 021-Security

  interface IPolicyClient    // PEP → PDP (022), default deny
  ────────────────────────────────────────────────────────────────────────────
  + DecideAsync(DecisionRequest) : Decision   // Allow | Deny(code)
  Falha do PDP ⇒ Deny para operações NOVAS; tráfego já autorizado continua (P-09)
```

---

## 4. Estruturas de Dados (projeção das entidades)

```
  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ SubjectDefinition            │        │ StreamDefinition              │
  │──────────────────────────────│   *  1 │───────────────────────────────│
  │ Urn           : Urn          │───────▶│ Id            : Guid          │
  │ Domain        : string       │        │ Name          : string        │
  │ Entity        : string       │        │ Subjects      : string[]      │
  │ Action        : string       │        │ Retention     : {limits,      │
  │ EventType     : string       │        │                  interest,    │
  │ ProducerModule: string       │        │                  workqueue}   │
  │ Dataschema    : string       │        │ MaxAge        : TimeSpan      │
  │ TrafficClass  : {control,    │        │ MaxBytes      : long          │
  │                  telemetry,  │        │ Replicas      : byte (1..5)   │
  │                  bulk, a2a}  │        │ Discard       : {old, new}    │
  │ PlatformScoped: bool         │        │ DedupeWindow  : TimeSpan      │
  │ DeprecatedAt  : DateTime?    │        │ OwnerModule   : string        │
  │ Version       : long (OCC)   │        │ State         : StreamState   │
  └──────────────────────────────┘        │ Version       : long (OCC)    │
                                          └───────────┬───────────────────┘
                                                      │ 1
                                                      ▼ *
                                          ┌───────────────────────────────┐
                                          │ ConsumerDefinition            │
                                          │───────────────────────────────│
                                          │ Id            : Guid          │
                                          │ DurableName   : string        │
                                          │ ConsumerModule: string        │
                                          │ FilterSubject : string?       │
                                          │ AckPolicy     : {explicit,    │
                                          │                  none, all}   │
                                          │ AckWait       : TimeSpan      │
                                          │ MaxDeliver    : short         │
                                          │ Backoff       : TimeSpan[]    │
                                          │ MaxAckPending : int           │
                                          │ DlqSubject    : string        │
                                          └───────────┬───────────────────┘
                                                      │ esgotou MaxDeliver
                                                      ▼ *
                                          ┌───────────────────────────────┐
                                          │ DeadLetterEntry               │
                                          │───────────────────────────────│
                                          │ Id             : Ulid         │
                                          │ TenantId       : string (RLS) │
                                          │ OriginalSubject: string       │
                                          │ StreamName     : string       │
                                          │ ConsumerName   : string       │
                                          │ DeliveryCount  : short        │
                                          │ LastError      : string       │
                                          │ Envelope       : CloudEvent   │
                                          │ Status         : {quarantined,│
                                          │                   replayed,   │
                                          │                   discarded}  │
                                          └───────────────────────────────┘

  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ A2ASession                   │        │ GroupChannel                  │
  │──────────────────────────────│        │───────────────────────────────│
  │ Urn            : Urn         │        │ Urn           : Urn           │
  │ TenantId       : string (RLS)│        │ TenantId      : string (RLS)  │
  │ InitiatorUrn   : Urn         │        │ GroupRef      : string        │
  │ PeerUrn        : Urn         │        │ SubjectPrefix : string        │
  │ PeerKind       : {internal,  │        │ FanoutLimit   : int (256)     │
  │                   external}  │        │ Scope         : {group,       │
  │ State          : A2ASession  │        │                  hierarchy,   │
  │                   State (§4) │        │                  broadcast}   │
  │ ChannelSubject : string      │        │ RateLimitMsgS : int           │
  │ Capabilities   : string[]    │        └───────────────────────────────┘
  │ MessageCount   : long        │
  │ BytesTotal     : long        │        ┌───────────────────────────────┐
  │ CloseReason    : string?     │        │ CloudEvent (RFC-0001 §5.2)    │
  │ Version        : long (OCC)  │        │ — NÃO redefinido aqui —       │
  └──────────────────────────────┘        │ specversion, id, source, type,│
                                          │ subject, time, tenant,        │
                                          │ traceparent, dataschema, data │
                                          └───────────────────────────────┘
```

`Urn` e `Ulid` são *value objects* imutáveis; `Urn` valida o formato
`urn:aios:<tenant>:<tipo>:<ULID>` conforme RFC-0001 §5.1 — o módulo **não redefine** o
formato, apenas o valida. `CloudEvent` é a projeção do envelope da RFC-0001 §5.2 e
**não** é estendido por este módulo.

---

## 5. Invariantes de Estrutura

| # | Invariante | Onde é garantido |
|---|-----------|------------------|
| C-01 | Um subject tem **exatamente um** `ProducerModule`. | `ISubjectRegistry.RegisterAsync` → `AIOS-BUS-0010` no publish |
| C-02 | `Subjects` de dois streams **NÃO DEVEM** se sobrepor. | `IStreamManager.PutAsync` → `AIOS-BUS-0004` |
| C-03 | `Backoff.Length` compatível com `MaxDeliver` e estritamente crescente. | `IConsumerManager.PutAsync` |
| C-04 | `A2ASession.ChannelSubject` é único entre sessões não-terminais e nunca reutilizado. | `A2AGateway` (invariante I1) |
| C-05 | `A2ASession.Capabilities` é imutável após `Established`. | `A2AGateway` (invariante I4) → `AIOS-A2A-0007` |
| C-06 | Toda mutação de entidade incrementa `Version` (OCC); conflito → `AIOS-BUS-0009`. | Camada de persistência |
| C-07 | `DeadLetterEntry.Envelope` preserva o evento original **sem** enriquecimento nem PII adicional. | `DeadLetterManager` |

---

## 6. Referências

- Componentes e diagrama-fonte: `./_DESIGN_BRIEF.md` §2
- Arquitetura: `./Architecture.md` · FSM: `./StateMachine.md`
- Modelo físico: `./Database.md` · API: `./API.md` · Eventos: `./Events.md`
