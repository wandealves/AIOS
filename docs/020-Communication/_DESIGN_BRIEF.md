---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0006, ADR-0008, ADR-0010 (globais, herdados); ADR-0200..ADR-0209 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — não redefinida); RFC-0200 (AIOS Bus Contract), RFC-0201 (A2A Profile for AIOS), a propor por este módulo
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database (schema `comm` e contrato de outbox), 021-Security (contas/NKey/JWT, mTLS), 022-Policy (PDP), 024-Observability, 025-Audit, 027-Cluster (supercluster/leaf nodes), 028-Deployment (topologia); usado como barramento por 004, 006, 007, 008, 009, 010, 011, 012, 013, 014, 015, 016, 017, 018, 019, 022, 023, 025, 026
---

# 020-Communication — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 020-Communication. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, **convenção de subjects**), este
> documento **referencia** a `../003-RFC/RFC-0001-Architecture-Baseline.md` e
> **não o redefine**. Termos são os do `../040-Glossary/Glossary.md`.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 0. Posição no AIOS e contrato de fronteira

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  MÓDULOS PRODUTORES/CONSUMIDORES (006 Kernel · 009 Scheduler · 010 · 011  │
   │  · 014 Workflow · 017 · 022 · 025 Audit · 026 · 004 API · 007 Runtime)    │
   │  · definem o SIGNIFICADO de cada evento (payload, semântica, quando emitir)│
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ publish / subscribe / request-reply / A2A
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  020-COMMUNICATION — BARRAMENTO (este módulo)                            │
   │  · governa o NAMESPACE de subjects (RFC-0001 §5.3) e o registro de streams│
   │  · garante ENTREGA (at-least-once, ordem por stream, replay, DLQ)        │
   │  · isola TENANTS por conta NATS; aplica backpressure e cotas de tráfego   │
   │  · media A2A (agente↔agente, inclusive externo) e comunicação seletiva    │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ opera
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  NATS 2.x + JetStream (cluster de 3+ nós, supercluster/leaf via 027)      │
   │  (PostgreSQL e Redis NÃO pertencem a este módulo; ver 005 e 028)          │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Fronteira em uma frase:** *o módulo produtor decide **o que** dizer e a quem
interessa; o 020 decide **como** a mensagem trafega, é isolada, entregue, reentregue,
reproduzida e contida.*

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O 020-Communication é o **subsistema de IPC** do AIOS — o análogo do conjunto
*sockets + pipes + sinais* de um sistema operacional clássico, elevado à escala de uma
frota distribuída. Ele não conhece a semântica de um evento de ciclo de vida de agente
ou de uma decisão de política; conhece **subjects, streams, consumidores, acks,
reentregas, ordem, isolamento e pressão**.

Sua missão é oferecer a todos os módulos um barramento **de baixa latência, durável
quando necessário e multi-tenant por construção**, sobre **NATS + JetStream**
(ADR-0004), com três garantias que nenhum módulo consumidor precisa reimplementar:
**entrega at-least-once**, **ordem por stream** e **isolamento entre tenants**.

A fronteira de confiança do módulo é o **namespace de subjects**: `aios.<tenant>.…` é
uma fronteira de segurança, não apenas de organização. Um agente do tenant `acme`
**NÃO DEVE** conseguir assinar, publicar ou sequer descobrir tráfego de `globex`, e
essa garantia é imposta por **contas NATS distintas por tenant**, não por convenção de
nomenclatura.

O módulo é também o responsável por manter o **custo de coordenação sub-linear** à
medida que a frota cresce. Com 10⁶ agentes, difusão indiscriminada é a forma mais
rápida de derrubar o sistema: `N` agentes conversando entre si produzem `N²` caminhos.
Por isso a **comunicação seletiva** (canais por *Agent Group*, escopo de fan-out,
limites de difusão) é responsabilidade de primeira classe — não uma otimização futura.

### 1.2 Responsabilidades (o 020-Communication DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Operar o cluster **NATS + JetStream** como barramento primário do AIOS: pub/sub, request/reply e streams duráveis. |
| R-02 | Governar o **namespace de subjects** conforme RFC-0001 §5.3, mantendo o registro `comm.subject_registry` (domínio, entidade, ação, produtor, schema) e rejeitando publicação fora do registro. |
| R-03 | Gerenciar o ciclo de vida de **streams** e **consumidores duráveis** JetStream: retenção, réplicas, limites, política de ack, `max_deliver`, backoff e *idle heartbeat*. |
| R-04 | Validar, no momento da publicação, o **envelope CloudEvents** da RFC-0001 §5.2 (campos obrigatórios, `type` coerente com o subject, `dataschema` registrado). |
| R-05 | Garantir **at-least-once** com dedupe por `event.id` (RFC-0001 §5.5) e **ordem por stream**; oferecer **replay** por tempo ou sequência. |
| R-06 | Isolar **tenants** em **contas NATS** distintas, com credenciais NKey/JWT emitidas pelo `021-Security`; nenhum subject cruza a fronteira de conta sem *export/import* explícito e autorizado. |
| R-07 | Aplicar **backpressure e cotas de tráfego** por tenant e por agente (mensagens/s, bytes/s, conexões, assinaturas), detectando e contendo *slow consumers*. |
| R-08 | Prover **comunicação seletiva**: canais por *Agent Group*, escopo de difusão e limite de fan-out, mantendo o custo de coordenação sub-linear. |
| R-09 | Mediar **A2A (Agent-to-Agent)**: estabelecer, autorizar, observar e encerrar sessões entre agentes — internos e externos — traduzindo o protocolo para subjects governados. |
| R-10 | Operar a **DLQ**: mensagens que esgotam as tentativas vão para quarentena inspecionável, com replay controlado e sem perda silenciosa. |
| R-11 | Atuar como **PEP**: toda operação administrativa (criar stream, exportar subject entre contas, replay, encerrar sessão A2A) consulta o **PDP** do `022-Policy`. *Default deny*. |
| R-12 | Emitir **telemetria OTel** (`aios_bus_*`), propagar `traceparent` em todas as mensagens e registrar em `025-Audit` toda ação privilegiada e toda sessão A2A. |

### 1.3 Não-Responsabilidades (o 020-Communication NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Definir a **semântica** de um evento (o que significa `agent.lifecycle.spawned`, quando emiti-lo, o que vai no `data`). O 020 transporta e valida a forma. | Módulo produtor (`006`, `009`, `010`, …) |
| NR-02 | Redefinir o envelope de evento, a convenção de subjects, a idempotência ou a correlação — estes são contratos da RFC-0001. | `003-RFC` |
| NR-03 | Persistir a **trilha de auditoria imutável** (o 020 entrega os eventos; o Audit os torna prova). | `025-Audit` |
| NR-04 | **Decidir** política (ser PDP). O 020 é PEP. | `022-Policy` |
| NR-05 | Emitir ou custodiar identidades e credenciais (NKey/JWT de conta, certificados mTLS). O 020 **consome** o que o cofre emite. | `021-Security` |
| NR-06 | Ser fonte da verdade de estado de domínio. O barramento é **transporte**, não banco: quem precisa de estado autoritativo usa `005-Database`. | `005-Database` |
| NR-07 | Implementar o padrão **Outbox** dentro dos módulos produtores (o 020 define o contrato de publicação; a tabela e o relay pertencem ao produtor, sobre o contrato físico do `005`). | Módulo produtor + `005-Database` |
| NR-08 | Transportar **blobs grandes**: payloads acima do limite configurado são rejeitados; o produtor envia referência a MinIO. | `028-Deployment` (MinIO), produtor |
| NR-09 | Terminar tráfego **norte-sul** externo (REST/gRPC de clientes). O 020 é o barramento leste-oeste interno. | `004-API` |
| NR-10 | Hospedar ou mediar o protocolo **MCP** de ferramentas (isso ocorre dentro do sandbox do runtime). | `007-Agent-Runtime`, `015-Tool-Manager` |
| NR-11 | Decidir **topologia geográfica** e failover de cluster (supercluster, leaf nodes, DR entre regiões). O 020 configura o que o `027` decide. | `027-Cluster` |
| NR-12 | Definir **orçamento de custo** de tráfego por tenant (o 020 mede e aplica a cota; o orçamento vem do `026`). | `026-Cost-Optimizer` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **SubjectRegistry** | Catálogo autoritativo de subjects permitidos: domínio, entidade, ação, produtor declarado, `dataschema`, classe de tráfego. Fonte da verdade de "quem pode publicar o quê". | → PostgreSQL (`comm.*`), `004-API` (registro de domínios) |
| **EnvelopeValidator** | Valida no *publish* o envelope CloudEvents (RFC-0001 §5.2): campos obrigatórios, `type` coerente com o subject, `tenant` casado com a conta, `dataschema` registrado, tamanho do payload. | SubjectRegistry |
| **PublishGateway** | Caminho de publicação: valida, carimba correlação, aplica cota, escreve no stream correto e confirma. Expõe o contrato que o relay de Outbox de cada produtor consome. | EnvelopeValidator, FlowController; → NATS |
| **StreamManager** | Ciclo de vida de streams JetStream: criação declarativa, retenção, réplicas, limites, *discard policy*, migração de configuração sem perda. | SubjectRegistry, PolicyClient; → JetStream |
| **ConsumerManager** | Consumidores duráveis: política de ack, `ack_wait`, `max_deliver`, backoff exponencial, *idle heartbeat*, `max_ack_pending`; detecta consumidor parado. | StreamManager; → JetStream |
| **SubscriptionBroker** | Assinaturas efêmeras e por wildcard; aplica escopo de subject por identidade; rejeita curinga que ultrapasse a fronteira autorizada. | PolicyClient, TenantAccountManager |
| **RequestReplyBroker** | Padrão request/reply com `reply-to` gerado, timeout, correlação por `traceparent` e mapeamento de erro para o envelope da RFC-0001 §5.4. | SubscriptionBroker |
| **A2AGateway** | Sessões A2A: handshake, negociação de capacidades, autorização no PDP, tradução para subjects governados, encerramento e trilha. Suporta pares externos sob mTLS. | PolicyClient, SessionStore, SubjectRegistry; → 021 |
| **SelectiveCommunicationRouter** | Canais por *Agent Group*: resolve destinatários, aplica limite de fan-out e escopo de difusão, prevenindo tráfego O(N²). | SubscriptionBroker, FlowController |
| **FlowController** | Backpressure: cotas por tenant/agente (msg/s, bytes/s, conexões, assinaturas), detecção de *slow consumer*, sinalização de recuo ao produtor. | → Redis (contadores), 026 (limites) |
| **DeadLetterManager** | Quarentena de mensagens que esgotaram `max_deliver`; catálogo inspecionável, replay controlado e política de expiração. | ConsumerManager; → PostgreSQL |
| **ReplayService** | Reprodução de stream por tempo, sequência ou filtro de subject, em consumidor isolado, sem perturbar os consumidores de produção. | StreamManager, PolicyClient |
| **TenantAccountManager** | Contas NATS por tenant: provisionamento, credenciais NKey/JWT (emitidas pelo `021`), *exports/imports* explícitos entre contas, revogação. | PolicyClient; → 021-Security |
| **BridgeConnector** | Conectividade entre clusters: leaf nodes, gateways de supercluster, filtros de propagação — configurado conforme decisão do `027`. | StreamManager; → 027-Cluster |
| **PolicyClient** | Cliente resiliente do PDP (`022`) para operações administrativas e autorização de sessão A2A; circuit breaker, cache curto, *fail-closed*. | → 022-Policy |
| **BusTelemetry** | Instrumentação OTel: spans de publish/deliver/A2A, métricas `aios_bus_*`, logs Serilog correlacionados, emissão de auditoria para `025`. | → 024-Observability, 025-Audit |

### 2.2 Diagrama de Componentes (ASCII)

```
        publish / subscribe / request-reply          A2A (interno e externo, mTLS)
                        │                                        │
   ┌────────────────────▼────────────────────────────────────────▼──────────────┐
   │                 COMMUNICATION SERVICE (020 · .NET 10)                       │
   │                                                                             │
   │   ┌──────────────────┐        ┌────────────────────┐                        │
   │   │ SubjectRegistry  │◀──────▶│ EnvelopeValidator  │                        │
   │   │ (domínio/ação/   │        │ (CloudEvents §5.2) │                        │
   │   │  produtor/schema)│        └─────────┬──────────┘                        │
   │   └────────┬─────────┘                  │                                   │
   │            │                            ▼                                   │
   │            │                  ┌────────────────────┐   cota   ┌───────────┐ │
   │            │                  │  PublishGateway    │─────────▶│ FlowCtrl  │ │
   │            │                  └─────────┬──────────┘          │(backpress)│ │
   │            │                            │                     └─────┬─────┘ │
   │   ┌────────▼─────────┐   ┌──────────────▼───────┐   ┌───────────────▼─────┐ │
   │   │  StreamManager   │──▶│  ConsumerManager     │──▶│ DeadLetterManager   │ │
   │   │ (retenção/réplic)│   │ (ack/max_deliver)    │   │ (quarentena+replay) │ │
   │   └────────┬─────────┘   └──────────┬───────────┘   └─────────────────────┘ │
   │            │                        │                                       │
   │   ┌────────▼─────────┐   ┌──────────▼───────────┐   ┌─────────────────────┐ │
   │   │  ReplayService   │   │ SubscriptionBroker   │──▶│ RequestReplyBroker  │ │
   │   └──────────────────┘   └──────────┬───────────┘   └─────────────────────┘ │
   │                                     │                                       │
   │   ┌──────────────────┐   ┌──────────▼───────────┐   ┌─────────────────────┐ │
   │   │ TenantAccountMgr │──▶│ SelectiveCommunic.   │   │  A2AGateway         │ │
   │   │ (conta por tenant│   │ Router (agent groups)│◀──│  (sessões, FSM §4)  │ │
   │   │  NKey/JWT ← 021) │   └──────────────────────┘   └──────────┬──────────┘ │
   │   └──────────────────┘                                         │            │
   │   ┌──────────────────┐   ┌──────────────────────┐   ┌──────────▼──────────┐ │
   │   │ BridgeConnector  │   │ PolicyClient (PEP)   │──▶│ BusTelemetry        │ │
   │   │ (leaf/gateway)   │   │  → 022-Policy        │   │ (OTel + auditoria)  │ │
   │   └──────────────────┘   └──────────────────────┘   └─────────────────────┘ │
   └────────┬─────────────────────┬──────────────────────┬───────────────────────┘
            ▼                     ▼                      ▼
   NATS 2.x + JetStream    PostgreSQL (schema comm)   024-Observability · 025-Audit
   (contas por tenant)     Redis (contadores)         027-Cluster (topologia)
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN `urn:aios:<tenant>:<tipo>:<id>`, `<id>` = ULID) e às
> convenções físicas do `../005-Database/Database.md` §2: `tenant_id` + RLS em toda
> tabela multi-tenant, `version bigint` para OCC, `timestamptz` em UTC. Schema:
> **`comm`**. O **estado de runtime** (streams, consumidores, conexões) vive no NATS;
> o PostgreSQL guarda a **declaração** e o histórico auditável — nunca as mensagens.
>
> Entidades de plataforma usam o tenant reservado **`_platform`** (convenção
> estabelecida em `../004-API/_DESIGN_BRIEF.md` §6).

### 3.1 Entidade `SubjectDefinition` — tabela `comm.subject_registry`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:subjectdef:<ULID>`. |
| `domain` | `text` | NOT NULL | `<dominio>` conforme RFC-0001 §5.3 (`agent`, `task`, `memory`, …). |
| `entity` | `text` | NOT NULL | `<entidade>` (ex.: `lifecycle`, `execution`). |
| `action` | `text` | NOT NULL, UNIQUE(domain,entity,action) | `<acao>` (ex.: `spawned`, `completed`). |
| `event_type` | `text` | NOT NULL | `aios.<dominio>.<entidade>.<acao>` (RFC-0001 §5.2). |
| `producer_module` | `text` | NOT NULL | Único módulo autorizado a publicar. |
| `dataschema` | `text` | NOT NULL | `aios://schemas/<tipo>/<versão>` registrado em `../004-API/`. |
| `stream_ref` | `uuid` | NOT NULL, FK→comm.stream.id | Stream que captura o subject. |
| `traffic_class` | `text` | NOT NULL, CHECK ∈ {control, telemetry, bulk, a2a} | Classe de tráfego (define cota e prioridade). |
| `platform_scoped` | `boolean` | NOT NULL default false | Usa o tenant reservado `_platform`. |
| `deprecated_at` | `timestamptz` | NULL | Início da depreciação (coexistência ≥ 2 majors). |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(urn)`; `UNIQUE(domain, entity, action)`; `btree(producer_module)`;
`btree(traffic_class)`.

### 3.2 Entidade `StreamDefinition` — tabela `comm.stream`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `uuid` | PK | Identidade da declaração. |
| `name` | `text` | NOT NULL, UNIQUE | Nome do stream (ex.: `KERNEL_LIFECYCLE`). |
| `subjects` | `text[]` | NOT NULL | Padrões capturados (`aios.*.agent.lifecycle.>`). |
| `retention` | `text` | NOT NULL, CHECK ∈ {limits, interest, workqueue} | Política JetStream. |
| `max_age` | `interval` | NOT NULL | Retenção temporal. |
| `max_bytes` | `bigint` | NOT NULL | Teto de armazenamento do stream. |
| `replicas` | `smallint` | NOT NULL, CHECK BETWEEN 1 AND 5 | Réplicas JetStream (3 em produção). |
| `discard` | `text` | NOT NULL, CHECK ∈ {old, new} | Descarte ao atingir limite. |
| `dedupe_window` | `interval` | NOT NULL default '2 minutes' | Janela de dedupe por `event.id` (RFC-0001 §5.5). |
| `owner_module` | `text` | NOT NULL | Módulo dono do fluxo. |
| `state` | `text` | NOT NULL, CHECK ∈ {declared, active, draining, retired} | Situação da declaração. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(id)`; `UNIQUE(name)`; `btree(owner_module)`; `btree(state)`.

### 3.3 Entidade `ConsumerDefinition` — tabela `comm.consumer`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `uuid` | PK | Identidade da declaração. |
| `stream_id` | `uuid` | NOT NULL, FK→comm.stream.id | Stream de origem. |
| `durable_name` | `text` | NOT NULL, UNIQUE(stream_id,durable_name) | Nome do consumidor durável. |
| `consumer_module` | `text` | NOT NULL | Módulo consumidor. |
| `filter_subject` | `text` | NULL | Filtro adicional de subject. |
| `ack_policy` | `text` | NOT NULL, CHECK ∈ {explicit, none, all} | Política de ack (`explicit` é o default do AIOS). |
| `ack_wait` | `interval` | NOT NULL default '30 seconds' | Prazo de ack antes da reentrega. |
| `max_deliver` | `smallint` | NOT NULL default 5 | Tentativas antes da DLQ. |
| `backoff` | `interval[]` | NOT NULL | Backoff por tentativa (ex.: `{1s,5s,30s,2min}`). |
| `max_ack_pending` | `int` | NOT NULL default 1000 | Limite de mensagens em voo (backpressure). |
| `dlq_subject` | `text` | NOT NULL | Destino da quarentena. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(id)`; `UNIQUE(stream_id, durable_name)`; `btree(consumer_module)`.

### 3.4 Entidade `A2ASession` — tabela `comm.a2a_session` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:a2asession:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento. |
| `initiator_urn` | `text` | NOT NULL | Agente iniciador (`urn:aios:<t>:agent:<ULID>`). |
| `peer_urn` | `text` | NOT NULL | Par: agente interno ou identidade externa federada. |
| `peer_kind` | `text` | NOT NULL, CHECK ∈ {internal, external} | Interno ao AIOS ou externo (A2A federado). |
| `state` | `text` | NOT NULL, CHECK ∈ A2ASessionState | Estado da FSM (§4). |
| `channel_subject` | `text` | NOT NULL | Subject privado da sessão (`aios.<t>.a2a.session.<ULID>`). |
| `capabilities` | `text[]` | NOT NULL default '{}' | Capacidades negociadas e autorizadas pelo PDP. |
| `message_count` | `bigint` | NOT NULL default 0 | Mensagens trocadas (base de cota e custo). |
| `bytes_total` | `bigint` | NOT NULL default 0 | Volume trocado. |
| `idempotency_key` | `text` | NULL, UNIQUE(tenant_id,idempotency_key) | RFC-0001 §5.5. |
| `close_reason` | `text` | NULL | Motivo do encerramento (`completed`, `timeout`, `revoked`, `error`). |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Abertura. |
| `last_activity_at` | `timestamptz` | NOT NULL | Base do timeout de ociosidade. |
| `closed_at` | `timestamptz` | NULL | Encerramento. |

Índices: `PK(urn)`; `btree(tenant_id, state)`; `btree(tenant_id, initiator_urn)`;
`btree(tenant_id, last_activity_at)` (varredura de sessões ociosas).

### 3.5 Entidade `GroupChannel` — tabela `comm.group_channel` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:groupchannel:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `group_ref` | `text` | NOT NULL | *Agent Group* (a composição pertence a `008`/`009`; aqui é referência). |
| `subject_prefix` | `text` | NOT NULL, UNIQUE(tenant_id,subject_prefix) | Prefixo de subject do canal. |
| `fanout_limit` | `int` | NOT NULL default 256 | Máximo de destinatários por difusão. |
| `scope` | `text` | NOT NULL, CHECK ∈ {group, hierarchy, broadcast} | Escopo de propagação. |
| `rate_limit_msg_s` | `int` | NOT NULL default 100 | Cota de difusão do canal. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, subject_prefix)`; `btree(tenant_id, group_ref)`.

### 3.6 Entidade `DeadLetterEntry` — tabela `comm.dlq_entry` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | PK (ULID = `event.id` original) | Identidade da mensagem morta. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `original_subject` | `text` | NOT NULL | Subject de origem. |
| `stream_name` | `text` | NOT NULL | Stream de origem. |
| `consumer_name` | `text` | NOT NULL | Consumidor que esgotou as tentativas. |
| `delivery_count` | `smallint` | NOT NULL | Tentativas realizadas. |
| `last_error` | `text` | NOT NULL | Erro reportado na última tentativa (sem PII). |
| `envelope` | `jsonb` | NOT NULL | Envelope CloudEvents preservado. |
| `status` | `text` | NOT NULL, CHECK ∈ {quarantined, replayed, discarded} | Situação. |
| `replayed_at` | `timestamptz` | NULL | Momento do replay autorizado. |
| `created_at` | `timestamptz` | NOT NULL | Entrada na DLQ. |

Índices: `PK(id)`; `btree(tenant_id, status)`; `btree(stream_name, created_at DESC)`.
Particionamento: `range_time` por `created_at`; retenção 30 dias (`../005-Database/`).

### 3.7 Relações (ASCII)

```
   comm.stream(1) ───< comm.subject_registry(*)      (qual stream captura o subject)
        │
        └───< comm.consumer(*) ───> dlq_subject ───> comm.dlq_entry(*)

   comm.a2a_session(*) ──[channel_subject]──▶ subject privado da sessão
        │                                     (fora do registry público)
        └── initiator_urn / peer_urn ──▶ urn:aios:<t>:agent:<ULID>   (006/008)

   comm.group_channel(*) ──[group_ref]──▶ Agent Group (composição em 008/009)
        └── fanout_limit / scope ──▶ SelectiveCommunicationRouter

   <schema>.outbox (do módulo produtor, contrato de 005) ──relay──▶ PublishGateway
```

> **Nota de fronteira:** nenhuma tabela deste módulo armazena mensagens de tráfego
> normal. O único payload persistido é o da **DLQ** — e ele existe justamente porque
> a mensagem **não** foi entregue.

---

## 4. Máquina de Estados Canônica da `A2ASession`

> A entidade com ciclo de vida governado neste módulo é a **sessão A2A**. Streams e
> consumidores têm ciclo declarativo simples (`declared → active → draining → retired`),
> descrito em §4.4; as demais entidades são configurações versionadas por OCC.

### 4.1 Estados (`A2ASessionState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Requested` | Iniciador pediu sessão; par ainda não respondeu. | não |
| `Negotiating` | Par respondeu; capacidades sendo negociadas e submetidas ao PDP. | não |
| `Established` | Sessão autorizada e ativa; canal privado alocado. | não |
| `Suspended` | Ativa porém pausada (cota excedida, política revista, par indisponível). | não |
| `Closing` | Encerramento em curso (drenagem de mensagens em voo). | não |
| `Closed` | Encerrada normalmente; canal liberado; trilha registrada. | **sim** |
| `Rejected` | Recusada na negociação (PDP negou, par recusou, capacidade incompatível). | **sim** |
| `Failed` | Encerrada por erro irrecuperável (timeout de handshake, par sumiu, violação). | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Requested` | `OpenSession` | Capability `a2a:session:open` ∧ `tenant` do par compatível ∧ cota de sessões do agente disponível. |
| T-02 | `Requested` → `Negotiating` | par respondeu ao handshake | Resposta dentro de `bus.a2a.handshake_timeout_ms` ∧ identidade do par verificada (mTLS/JWT). |
| T-03 | `Requested` → `Failed` | timeout de handshake | Sem resposta dentro do prazo. |
| T-04 | `Negotiating` → `Established` | PDP autorizou o conjunto negociado | Decisão `allow` para todas as `capabilities` ∧ canal privado alocado. |
| T-05 | `Negotiating` → `Rejected` | PDP negou ∨ par recusou ∨ capacidades incompatíveis | Motivo registrado em `close_reason`. |
| T-06 | `Established` → `Suspended` | cota excedida ∨ `policy.bundle.updated` revoga capacidade | Cota de mensagens/bytes estourada ∨ nova decisão do PDP nega capacidade em uso. |
| T-07 | `Suspended` → `Established` | cota renovada ∨ reautorização | Nova janela de cota ∨ decisão `allow` do PDP. |
| T-08 | `Established`/`Suspended` → `Closing` | `CloseSession` ∨ ociosidade | Pedido explícito ∨ `now - last_activity_at > bus.a2a.idle_timeout_ms`. |
| T-09 | `Closing` → `Closed` | drenagem concluída | Mensagens em voo entregues ou expiradas ∧ canal liberado ∧ evento emitido. |
| T-10 | `Established`/`Suspended`/`Closing` → `Failed` | erro irrecuperável | Par desapareceu além do limite ∨ violação de protocolo ∨ revogação de identidade pelo `021`. |

**Invariantes:**
(I1) Uma sessão em estado não-terminal possui **exatamente um** `channel_subject`
exclusivo; o subject é liberado e **NÃO DEVE** ser reutilizado após o estado terminal.
(I2) Mensagens só trafegam no canal quando o estado é `Established`; em `Suspended` são
recusadas com `AIOS-A2A-0004`, não enfileiradas indefinidamente.
(I3) Toda transição para estado terminal emite exatamente um evento
`aios.<tenant>.comm.a2a.<acao>` e um registro de auditoria em `025`.
(I4) `capabilities` **NÃO DEVEM** ser ampliadas após `Established` sem nova negociação
(volta a `Negotiating` implicaria nova sessão — o desenho exige sessão nova).
(I5) Transição só ocorre com `version` esperado (OCC); conflito → `AIOS-BUS-0009`.

### 4.3 Diagrama de estados (ASCII)

```
        OpenSession (T-01)
   ∅ ───────────────────────▶ ┌───────────┐
                               │ Requested │
                               └─────┬─────┘
             handshake ok (T-02)     │     timeout (T-03)
          ┌─────────────────────────┤────────────────────────┐
          ▼                          │                        ▼
   ┌─────────────┐                   │                  ┌──────────┐
   │ Negotiating │                   │                  │  Failed  │◀── erro irrecuperável
   └──────┬──────┘                   │                  └──────────┘        (T-10)
          │ PDP allow (T-04)         │  PDP deny /            ▲
          │                          │  par recusa (T-05)     │
          │                          ▼                        │
          │                   ┌────────────┐                  │
          │                   │  Rejected  │ (terminal)       │
          │                   └────────────┘                  │
          ▼                                                    │
   ┌─────────────┐  cota/revogação (T-06)  ┌─────────────┐    │
   │ Established │────────────────────────▶│  Suspended  │────┘
   │             │◀────────────────────────│             │
   └──────┬──────┘  reautorização (T-07)   └──────┬──────┘
          │                                        │
          │ close / ociosidade (T-08)              │ close (T-08)
          ▼                                        ▼
   ┌─────────────┐        drenagem concluída (T-09)      ┌──────────┐
   │   Closing   │──────────────────────────────────────▶│  Closed  │ (terminal)
   └─────────────┘                                        └──────────┘
```

### 4.4 Ciclo declarativo de `StreamDefinition`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `declared` | Declaração registrada em `comm.stream`, ainda não materializada no NATS. | `active` |
| `active` | Stream existe no JetStream e aceita publicação. | `draining` |
| `draining` | Não aceita novas publicações; consumidores drenam o backlog. | `retired` |
| `retired` | Removido do NATS; declaração retida para histórico. | — (terminal) |

Alterações de configuração de um stream `active` (retenção, réplicas, limites) são
aplicadas **sem perda**: quando o NATS não suporta a mudança em lugar, o
`StreamManager` cria o novo, espelha e drena o antigo (ADR-0203).

---

## 5. Superfície de API (REST e gRPC)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.comm.v1`. Base REST: `/v1/comm`. Toda mutação DEVE aceitar
> `Idempotency-Key` e os cabeçalhos de correlação (RFC-0001 §5.6).
>
> **Esta é uma API de controle.** O caminho de mensagem não passa por ela: produtores e
> consumidores falam **NATS diretamente**, sob as contas, cotas e validações que este
> módulo governa. Colocar o serviço 020 no caminho de cada mensagem anularia a razão de
> escolher NATS (ADR-0004).

### 5.1 Operações

| Operação | REST | gRPC (rpc) | Idempotente | Notas |
|----------|------|-----------|-------------|-------|
| Registrar subject | `POST /v1/comm/subjects` | `RegisterSubject` | sim (key) | Valida contra RFC-0001 §5.3. |
| Listar/ler subjects | `GET /v1/comm/subjects[/{urn}]` | `ListSubjects`/`GetSubject` | sim (leitura) | Catálogo público do barramento. |
| Depreciar subject | `POST /v1/comm/subjects/{urn}:deprecate` | `DeprecateSubject` | sim | Coexistência ≥ 2 majors. |
| Declarar stream | `PUT /v1/comm/streams/{name}` | `PutStream` | sim | Cria/atualiza declaração. |
| Listar streams | `GET /v1/comm/streams` | `ListStreams` | sim (leitura) | Inclui estado e uso. |
| Drenar/aposentar stream | `POST /v1/comm/streams/{name}:drain` | `DrainStream` | sim | `active` → `draining` → `retired`. |
| Declarar consumidor | `PUT /v1/comm/streams/{name}/consumers/{durable}` | `PutConsumer` | sim | Ack, `max_deliver`, backoff. |
| Estado de consumidor | `GET /v1/comm/streams/{name}/consumers/{durable}` | `GetConsumer` | sim (leitura) | Lag, pendentes, redeliveries. |
| Listar DLQ | `GET /v1/comm/dlq` | `ListDeadLetters` | sim (leitura) | Filtro por stream/consumidor/período. |
| Replay de DLQ | `POST /v1/comm/dlq/{id}:replay` | `ReplayDeadLetter` | sim (key) | Reinjeta no subject original. |
| Descartar da DLQ | `POST /v1/comm/dlq/{id}:discard` | `DiscardDeadLetter` | sim | Requer justificativa; auditado. |
| Replay de stream | `POST /v1/comm/streams/{name}:replay` | `ReplayStream` | sim (key) | Por tempo/sequência, em consumidor isolado. |
| Abrir sessão A2A | `POST /v1/comm/a2a/sessions` | `OpenSession` | sim (key) | T-01. |
| Ler sessão A2A | `GET /v1/comm/a2a/sessions/{urn}` | `GetSession` | sim (leitura) | Estado, capacidades, volume. |
| Encerrar sessão A2A | `POST /v1/comm/a2a/sessions/{urn}:close` | `CloseSession` | sim | T-08. |
| Criar canal de grupo | `PUT /v1/comm/groups/{group}/channel` | `PutGroupChannel` | sim | Fan-out e escopo. |
| Provisionar conta de tenant | `POST /v1/comm/accounts` | `ProvisionAccount` | sim (key) | Conta NATS + credenciais (`021`). |
| Exportar subject entre contas | `POST /v1/comm/accounts/{tenant}/exports` | `PutAccountExport` | sim | **Alta sensibilidade**: cruza fronteira de tenant. |
| Status do barramento | `GET /v1/comm/health` | `GetBusHealth` | sim (leitura) | Nós, lag de replicação, slow consumers. |

### 5.2 Catálogo de códigos de erro

> Formato RFC-0001 §5.4: `AIOS-<DOMINIO>-<NNNN>`. Domínios reservados por este módulo:
> **`BUS`** (0001–0099) e **`A2A`** (0001–0099), registrados no catálogo mantido por
> `../004-API/` (RFC-0001 §8).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-BUS-0001` | 404 | não | Subject, stream, consumidor ou sessão não encontrado. |
| `AIOS-BUS-0002` | 403 | não | Operação negada pelo PDP (*default deny*). |
| `AIOS-BUS-0003` | 403 | não | Tenant divergente do contexto autenticado (fronteira de conta). |
| `AIOS-BUS-0004` | 422 | não | Subject fora do registro ou fora da convenção RFC-0001 §5.3. |
| `AIOS-BUS-0005` | 422 | não | Envelope CloudEvents inválido (campo obrigatório ausente ou `type` incoerente). |
| `AIOS-BUS-0006` | 413 | não | Payload acima de `bus.publish.max_payload_bytes`; use referência a objeto. |
| `AIOS-BUS-0007` | 429 | sim | Cota de tráfego excedida (mensagens/s ou bytes/s). |
| `AIOS-BUS-0008` | 429 | sim | Limite de fan-out do canal de grupo excedido. |
| `AIOS-BUS-0009` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-BUS-0010` | 409 | não | Produtor não autorizado para o subject (dono declarado é outro módulo). |
| `AIOS-BUS-0011` | 503 | sim | Cluster NATS indisponível ou sem quórum de JetStream. |
| `AIOS-BUS-0012` | 504 | sim | Timeout em request/reply. |
| `AIOS-BUS-0013` | 507 | não | Stream atingiu `max_bytes` com `discard=new`. |
| `AIOS-BUS-0014` | 409 | não | *Slow consumer*: `max_ack_pending` excedido; assinatura contida. |
| `AIOS-BUS-0015` | 422 | não | `dataschema` não registrado ou versão desconhecida. |
| `AIOS-A2A-0001` | 403 | não | Sessão negada pelo PDP na negociação (T-05). |
| `AIOS-A2A-0002` | 504 | sim | Timeout de handshake (T-03). |
| `AIOS-A2A-0003` | 409 | não | Transição de estado inválida (viola a FSM §4). |
| `AIOS-A2A-0004` | 409 | sim | Sessão em `Suspended`: mensagem recusada. |
| `AIOS-A2A-0005` | 403 | não | Identidade do par não verificada ou revogada. |
| `AIOS-A2A-0006` | 429 | sim | Cota de sessões simultâneas do agente excedida. |
| `AIOS-A2A-0007` | 422 | não | Capacidade negociada não suportada ou fora do perfil da RFC-0201. |

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3
> (`aios.<tenant>.<dominio>.<entidade>.<acao>`). `<dominio>` = **`comm`**, novo valor a
> ser registrado no registro de domínios mantido por `../004-API/` conforme
> RFC-0001 §8 (ratificação em **ADR-0200**). Entrega at-least-once via JetStream;
> consumidores DEVEM deduplicar por `event.id`.
>
> Este módulo é peculiar: ele **transporta os eventos de todos os outros**, mas emite
> poucos próprios — apenas os que descrevem o estado do próprio barramento.

### 6.1 Eventos emitidos (produtor: 020-Communication)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios._platform.comm.subject.registered` | `aios.comm.subject.registered` | Novo subject registrado. | `COMM_PLATFORM` (durável) |
| `aios._platform.comm.subject.deprecated` | `aios.comm.subject.deprecated` | Subject entrou em depreciação. | `COMM_PLATFORM` |
| `aios._platform.comm.stream.created` | `aios.comm.stream.created` | Stream materializado no JetStream. | `COMM_PLATFORM` |
| `aios._platform.comm.stream.retired` | `aios.comm.stream.retired` | Stream aposentado após drenagem. | `COMM_PLATFORM` |
| `aios._platform.comm.cluster.degraded` | `aios.comm.cluster.degraded` | Perda de nó ou de quórum JetStream. | `COMM_HEALTH` |
| `aios._platform.comm.account.provisioned` | `aios.comm.account.provisioned` | Conta NATS criada para um tenant. | `COMM_PLATFORM` |
| `aios.<tenant>.comm.a2a.established` | `aios.comm.a2a.established` | Sessão A2A → `Established` (T-04). | `COMM_A2A` |
| `aios.<tenant>.comm.a2a.rejected` | `aios.comm.a2a.rejected` | Sessão → `Rejected` (T-05). | `COMM_A2A` |
| `aios.<tenant>.comm.a2a.closed` | `aios.comm.a2a.closed` | Sessão → `Closed` (T-09). | `COMM_A2A` |
| `aios.<tenant>.comm.a2a.failed` | `aios.comm.a2a.failed` | Sessão → `Failed` (T-10). | `COMM_A2A` |
| `aios.<tenant>.comm.delivery.deadlettered` | `aios.comm.delivery.deadlettered` | Mensagem esgotou `max_deliver`. | `COMM_HEALTH` |
| `aios.<tenant>.comm.delivery.replayed` | `aios.comm.delivery.replayed` | Replay autorizado de DLQ ou stream. | `COMM_HEALTH` |
| `aios.<tenant>.comm.consumer.stalled` | `aios.comm.consumer.stalled` | Consumidor parado ou lag acima do limiar. | `COMM_HEALTH` |
| `aios.<tenant>.comm.quota.throttled` | `aios.comm.quota.throttled` | Cota de tráfego aplicada a um produtor. | `COMM_HEALTH` |

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do módulo |
|-------------------|----------|----------------|
| `aios.<tenant>.policy.bundle.updated` | `022-Policy` | Invalida cache do `PolicyClient`; reavalia sessões A2A ativas (pode disparar T-06). |
| `aios.<tenant>.security.token.revoked` | `021-Security` | Revoga credencial de conta/par; encerra sessões afetadas (T-10). |
| `aios._platform.cluster.topology.changed` | `027-Cluster` | Reconfigura leaf nodes/gateways via `BridgeConnector`. |
| `aios.<tenant>.cost.budget.updated` | `026-Cost-Optimizer` | Atualiza cotas de tráfego do `FlowController`. |
| `aios.<tenant>.agent.lifecycle.terminated` | `006-Kernel` | Encerra sessões A2A e assinaturas do agente extinto. |
| `aios._platform.api.eventschema.registered` | `004-API` | Atualiza o cache de `dataschema` do `EnvelopeValidator`. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Operar pub/sub, request/reply e streams duráveis sobre NATS + JetStream. | Must | Os três padrões exercitados por teste de integração com cluster real de 3 nós. |
| FR-002 | Rejeitar publicação em subject fora do registro ou fora da convenção RFC-0001 §5.3. | Must | Subject não registrado → `AIOS-BUS-0004`; teste cobre malformado e não registrado. |
| FR-003 | Validar o envelope CloudEvents no *publish*. | Must | Campo obrigatório ausente ou `type` incoerente com o subject → `AIOS-BUS-0005`. |
| FR-004 | Garantir que apenas o módulo produtor declarado publique em cada subject. | Must | Publicação por outro módulo → `AIOS-BUS-0010`. |
| FR-005 | Garantir entrega at-least-once com dedupe por `event.id` na janela configurada. | Must | Publicação duplicada dentro de `dedupe_window` resulta em uma única mensagem no stream. |
| FR-006 | Preservar ordem por stream. | Must | Sequência de 10⁵ mensagens chega em ordem no consumidor durável. |
| FR-007 | Reentregar mensagens não confirmadas com backoff e enviar à DLQ ao esgotar `max_deliver`. | Must | Consumidor que nunca dá ack gera entrada em `comm.dlq_entry` e evento `delivery.deadlettered`. |
| FR-008 | Permitir replay de DLQ e de stream (por tempo ou sequência) em consumidor isolado. | Must | Replay não perturba consumidores de produção; evento `delivery.replayed` emitido. |
| FR-009 | Isolar tenants em contas NATS distintas. | Must | Credencial de `acme` não publica nem assina em `aios.globex.>`; tentativa → `AIOS-BUS-0003`. |
| FR-010 | Aplicar cotas de tráfego por tenant e agente, com sinalização de recuo. | Must | Excesso → `AIOS-BUS-0007` e evento `quota.throttled`. |
| FR-011 | Detectar e conter *slow consumers*. | Must | `max_ack_pending` excedido → `AIOS-BUS-0014` e evento `consumer.stalled`; demais consumidores não degradam. |
| FR-012 | Prover canais de *Agent Group* com limite de fan-out e escopo de difusão. | Must | Difusão acima de `fanout_limit` → `AIOS-BUS-0008`. |
| FR-013 | Estabelecer, autorizar e encerrar sessões A2A conforme a FSM §4. | Must | Sessão sem capability → `AIOS-A2A-0001`; transição inválida → `AIOS-A2A-0003`. |
| FR-014 | Suportar par A2A **externo** com identidade verificada por mTLS/JWT federado. | Should | Par com identidade revogada → `AIOS-A2A-0005` e encerramento (T-10). |
| FR-015 | Rejeitar payload acima do limite e orientar uso de referência a objeto. | Must | Payload > `bus.publish.max_payload_bytes` → `AIOS-BUS-0006`. |
| FR-016 | Declarar streams e consumidores de forma idempotente e versionada. | Must | `PUT` repetido não altera estado; mudança incompatível migra sem perda (ADR-0203). |
| FR-017 | Consultar o PDP antes de toda operação administrativa e de toda abertura de sessão A2A. | Must | Sem capability → `AIOS-BUS-0002` / `AIOS-A2A-0001`; decisão auditada. |
| FR-018 | Propagar `traceparent` em 100% das mensagens e emitir telemetria OTel. | Must | Trace contínuo do produtor ao consumidor em teste ponta a ponta. |
| FR-019 | Exportar subject entre contas somente por declaração explícita e autorizada. | Must | Export sem capability → `AIOS-BUS-0002`; export registrado em `../025-Audit/`. |
| FR-020 | Encerrar automaticamente sessões A2A ociosas e assinaturas de agentes extintos. | Should | Ociosidade > `bus.a2a.idle_timeout_ms` → T-08; `agent.lifecycle.terminated` limpa recursos. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (publicação core) | p99 de publish **fire-and-forget** ≤ **2 ms** intra-cluster. | `aios_bus_publish_latency_ms{durable="false"}`. |
| NFR-002 | Desempenho (publicação durável) | p99 de publish com ack JetStream (3 réplicas) ≤ **15 ms**. | `aios_bus_publish_latency_ms{durable="true"}`. |
| NFR-003 | Desempenho (request/reply) | p99 ≤ **10 ms** intra-cluster, excluído o tempo de processamento do respondente. | `aios_bus_request_latency_ms`. |
| NFR-004 | Throughput | ≥ **1.000.000 mensagens/s** core e ≥ **200.000 mensagens/s** persistidas, em cluster de 3 nós. | `aios_bus_messages_total` (taxa); `./Benchmark.md`. |
| NFR-005 | Disponibilidade | ≥ **99,95%** do barramento (perda de 1 nó sem indisponibilidade percebida). | Uptime probe; *error budget* mensal. |
| NFR-006 | Entrega | Perda de mensagem persistida = **0** sob falha de 1 nó (R=3, quórum mantido). | Chaos test com contagem publicada × consumida. |
| NFR-007 | Escalabilidade | ≥ **10⁶** agentes conectados (via runtimes), ≥ **10⁵** subjects ativos, ≥ **10⁴** consumidores duráveis, com custo de coordenação **sub-linear**. | Teste de escala em `./Scalability.md`. |
| NFR-008 | Latência de entrega | p99 do intervalo publish→deliver ≤ **20 ms** para consumidor saudável. | `aios_bus_delivery_latency_ms`. |
| NFR-009 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min** para streams duráveis. | DR drill; restauração de snapshot JetStream. |
| NFR-010 | Isolamento multi-tenant | **0** vazamento entre contas; 100% dos tenants em conta própria. | Teste de penetração de subject; auditoria de contas. |
| NFR-011 | Contenção | Um *slow consumer* **NÃO DEVE** degradar o p99 dos demais em mais de **10%**. | Teste de contenção com consumidor artificialmente lento. |
| NFR-012 | Taxa de DLQ | ≤ **0,01%** das mensagens entregues em regime normal. | `aios_bus_deadletter_total` ÷ `aios_bus_messages_total`. |
| NFR-013 | Observabilidade | **100%** das mensagens com `traceparent` propagado e correlação preservada. | Amostragem de traces ponta a ponta. |
| NFR-014 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key`. |
| NFR-015 | Sessões A2A | Suportar ≥ **10⁴** sessões simultâneas por cluster; p99 de handshake ≤ **200 ms**. | `aios_bus_a2a_sessions_active`, `aios_bus_a2a_handshake_ms`. |
| NFR-016 | Eficiência de fan-out | Difusão para um *Agent Group* de 1.000 membros **NÃO DEVE** custar mais que **1** publicação do produtor. | Contagem de publicações × entregas em `./Benchmark.md`. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global` (cluster), `tenant`, `agent`, `stream`. Recarregável indica *hot
> reload*. Prefixo de env var: **`AIOS_BUS_`**.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `bus.publish.max_payload_bytes` | 1048576 | 4096–8388608 | global/tenant | sim | Teto de payload (`AIOS-BUS-0006`). Acima disso, use referência a objeto. |
| `bus.publish.validate_envelope` | `true` | {true,false} | global | sim | Validação do envelope CloudEvents no publish. |
| `bus.publish.require_registered_subject` | `true` | {true,false} | global | sim | Rejeita subject fora do registro (`AIOS-BUS-0004`). |
| `bus.stream.default_replicas` | 3 | 1–5 | global/stream | **não** | Réplicas JetStream; alterar exige migração de stream. |
| `bus.stream.default_max_age` | `7d` | 1h–3650d | stream | sim | Retenção temporal default. |
| `bus.stream.default_max_bytes` | 53687091200 | 1e9–1e13 | stream | sim | Teto de armazenamento por stream (50 GiB default). |
| `bus.stream.dedupe_window` | `2m` | 10s–1h | stream | sim | Janela de dedupe por `event.id` (RFC-0001 §5.5). |
| `bus.stream.discard_policy` | `old` | {old,new} | stream | sim | Descarte ao atingir o limite. |
| `bus.consumer.ack_wait_ms` | 30000 | 1000–600000 | stream/global | sim | Prazo de ack antes da reentrega. |
| `bus.consumer.max_deliver` | 5 | 1–100 | stream/global | sim | Tentativas antes da DLQ. |
| `bus.consumer.backoff_ms` | `[1000,5000,30000,120000]` | lista crescente | stream | sim | Backoff por tentativa. |
| `bus.consumer.max_ack_pending` | 1000 | 1–100000 | stream/global | sim | Mensagens em voo por consumidor (backpressure). |
| `bus.consumer.stall_threshold_ms` | 60000 | 5000–3600000 | global | sim | Silêncio até declarar consumidor parado. |
| `bus.quota.msgs_per_sec_tenant` | 50000 | 100–5000000 | tenant | sim | Cota de publicação por tenant (`AIOS-BUS-0007`). |
| `bus.quota.msgs_per_sec_agent` | 200 | 1–100000 | agent | sim | Cota de publicação por agente. |
| `bus.quota.bytes_per_sec_tenant` | 104857600 | 1e6–1e10 | tenant | sim | Cota de banda por tenant. |
| `bus.quota.max_subscriptions_agent` | 64 | 1–4096 | agent | sim | Assinaturas simultâneas por agente. |
| `bus.group.fanout_limit` | 256 | 1–10000 | tenant/global | sim | Destinatários por difusão de canal de grupo. |
| `bus.group.default_scope` | `group` | {group,hierarchy,broadcast} | tenant | sim | Escopo default de propagação. |
| `bus.request.timeout_ms` | 5000 | 100–120000 | global/tenant | sim | Timeout de request/reply (`AIOS-BUS-0012`). |
| `bus.a2a.handshake_timeout_ms` | 3000 | 500–30000 | global/tenant | sim | Prazo de handshake (T-03). |
| `bus.a2a.idle_timeout_ms` | 300000 | 10000–86400000 | tenant/agent | sim | Ociosidade até encerrar sessão (T-08). |
| `bus.a2a.max_sessions_agent` | 32 | 1–1024 | agent | sim | Sessões simultâneas por agente (`AIOS-A2A-0006`). |
| `bus.a2a.allow_external_peers` | `false` | {true,false} | tenant | sim | Habilita pares A2A externos (exige identidade federada). |
| `bus.dlq.retention_days` | 30 | 1–365 | global | sim | Retenção da quarentena. |
| `bus.dlq.auto_replay` | `false` | {true,false} | global | sim | Replay automático da DLQ. Default desligado: replay cego repete o defeito. |
| `bus.policy.fail_mode` | `closed` | {closed,open} | global | sim | Comportamento se o PDP estiver indisponível. |
| `bus.cluster.leaf_enabled` | `false` | {true,false} | global | **não** | Habilita leaf nodes (topologia decidida por `027`). |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| Perda de 1 nó NATS | Health do cluster + `aios_bus_cluster_nodes` | Quórum mantido (R=3); clientes reconectam a outro nó | Reingresso automático do nó; ressincronização de stream | Sem perda; publicação com ack é retentável |
| Perda de quórum JetStream | Falha de ack de publicação durável | Publicação durável rejeitada (`AIOS-BUS-0011`); core pub/sub PODE seguir | Restaurar nós até o quórum; snapshot se necessário | Produtor retém no Outbox e republica |
| *Slow consumer* | `max_ack_pending` atingido; `pending` crescente | Contenção da assinatura; demais consumidores intactos | Escalar consumidores ou aumentar paralelismo; evento `consumer.stalled` | Reentrega natural após ack |
| Consumidor parado | Sem ack por `stall_threshold_ms` | Consumidor marcado; backlog acumula no stream | Alerta + reinício do consumidor; DLQ após `max_deliver` | Backoff exponencial |
| Mensagem envenenada (*poison message*) | Reentregas repetidas com mesmo erro | Vai para DLQ ao esgotar `max_deliver` | Inspeção humana; correção do consumidor; replay controlado | Replay é explícito, nunca automático por default |
| Stream atinge `max_bytes` | `aios_bus_stream_bytes` no teto | `discard=old` descarta antigo; `discard=new` rejeita (`AIOS-BUS-0013`) | Aumentar limite ou reduzir retenção; investigar produtor | Produtor recua com backpressure |
| Produtor inundando (*noisy neighbor*) | Cota por tenant/agente | `AIOS-BUS-0007` com recuo; cota é por tenant, não global | Ajuste de cota via `026`; investigação do produtor | Retry com backoff |
| Fan-out explosivo em grupo | `fanout_limit` excedido | Difusão rejeitada (`AIOS-BUS-0008`) antes de executar | Reduzir escopo ou dividir o grupo | Determinístico |
| PDP (022) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do PEP | `fail_mode=closed`: operações administrativas e novas sessões A2A negadas; **tráfego já autorizado continua** | Retry após meia-abertura do CB |
| Par A2A desaparece | Sem atividade + heartbeat perdido | Sessão isolada | T-10 → `Failed`; recursos liberados; evento emitido | Nova sessão exige novo handshake |
| Credencial de conta revogada | Evento `security.token.revoked` | Conta desconectada | Sessões afetadas encerradas; nova credencial emitida pelo `021` | Reconexão idempotente |
| Partição entre clusters (leaf/gateway) | Health do `BridgeConnector` | Cada lado opera localmente | Reconciliação de streams espelhados ao religar; dedupe por `event.id` | At-least-once preservado |
| Perda de dados de stream | Falha de leitura/quórum | Stream isolado | Restauração de snapshot JetStream; produtores republicam do Outbox | Dedupe evita duplicidade efetiva |

**Metas de recuperação:** **RTO ≤ 15 min**, **RPO ≤ 5 min** para streams duráveis
(alinhado a V-R10 da Visão e §2 da Arquitetura). Degradação graciosa em ordem:
sacrifica-se primeiro **telemetria e tráfego bulk**, depois **replay e operações
administrativas**, preservando por último o **tráfego de controle** e a **entrega de
mensagens já aceitas**. O barramento prefere **rejeitar na entrada** (com sinal claro
de recuo) a aceitar e perder depois.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Namespace | Subjects hierárquicos `aios.<tenant>.<dominio>.<entidade>.<acao>` (RFC-0001 §5.3) permitem assinatura por wildcard no nível exato de interesse — o filtro acontece no **broker**, não no consumidor. |
| Isolamento de tenant | **Conta NATS por tenant**: o roteamento nem sequer considera subjects de outras contas, o que torna o isolamento também uma otimização de dispersão. |
| Comunicação seletiva | Canais por *Agent Group* com `fanout_limit` e `scope`: uma difusão custa **1 publicação** do produtor, independentemente do tamanho do grupo (NFR-016) — o fan-out é do broker, não do agente. |
| Contenção de N² | Agentes **NÃO DEVEM** manter assinaturas ponto a ponto com todos os pares: `max_subscriptions_agent` (64) força o uso de grupos e sessões A2A explícitas. |
| Streams por domínio | Um stream por família de fluxo (não um por tenant): mantém o número de streams em O(domínios), não O(tenants), preservando a operabilidade com 10⁴ tenants. |
| Consumidores duráveis | `max_ack_pending` limita o em-voo por consumidor; paralelismo se obtém com múltiplas instâncias no mesmo *durable*, não com janela maior. |
| Backpressure | Cotas por tenant/agente + `max_ack_pending` + `discard policy`: a pressão sobe até o produtor, que recua com `AIOS-BUS-0007`, em vez de o broker inchar. |
| Escala horizontal | Cluster NATS de 3+ nós com JetStream R=3; o serviço `aios-comm-svc` é **stateless** (a declaração está no PostgreSQL, o runtime no NATS). |
| Escala geográfica | Leaf nodes e gateways de supercluster (configurados conforme `027`), com filtros de propagação para não replicar tráfego local entre regiões. |
| Sessões A2A | Canal privado por sessão, encerrado por ociosidade; sessões são recurso cotado (`max_sessions_agent`), não um recurso livre. |
| Rumo a milhões | 10⁶ agentes só é viável se a maior parte do tráfego for **hierárquico e seletivo**: eventos de controle por domínio, difusão por grupo, ponto a ponto apenas em sessões A2A explícitas e cotadas. O custo de coordenação cresce com o número de **grupos e domínios**, não com o quadrado do número de agentes. |

```
   ❌ malha ponto a ponto            ✅ hierarquia + grupos
      N agentes → N² caminhos           N agentes → O(N) assinaturas

      A ─── B                        aios.acme.group.squad-7.>
      │ \ / │                              ▲   ▲   ▲
      │  X  │                              │   │   │
      C ─── D                            A   B   C   D   (1 publish → fan-out do broker)
```

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada a este módulo: `ADR-0200`..`ADR-0209`** (regra 020×10).
> Registrar em `../002-ADR/`. **ADR-0004** (NATS como barramento primário) é herdada
> da fundação e é a decisão fundadora deste módulo.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0200 | Registro obrigatório de subjects (`comm.subject_registry`) e criação do domínio de eventos `comm`. |
| ADR-0201 | Conta NATS por tenant como fronteira de isolamento (vs. prefixo de subject com autorização). |
| ADR-0202 | Validação do envelope CloudEvents no *publish* (vs. validação apenas no consumidor). |
| ADR-0203 | Streams declarativos versionados e migração sem perda de configuração incompatível. |
| ADR-0204 | Um stream por domínio de fluxo, não por tenant: implicações de retenção e operabilidade. |
| ADR-0205 | DLQ com replay explícito e `auto_replay` desligado por default. |
| ADR-0206 | Comunicação seletiva por canais de *Agent Group* com limite de fan-out. |
| ADR-0207 | Perfil A2A do AIOS: sessões com FSM, negociação de capacidades e pares externos sob mTLS. |
| ADR-0208 | Limite de payload e política de "referência a objeto" para cargas grandes. |
| ADR-0209 | Domínios de código de erro reservados (`BUS`, `A2A`) e política de *fail-closed* do PEP. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida). |
| RFC-0200 | AIOS Bus Contract (registro de subjects, declaração de streams/consumidores, garantias de entrega, DLQ, backpressure). | A propor por este módulo. |
| RFC-0201 | A2A Profile for AIOS (handshake, negociação de capacidades, FSM de sessão, federação externa). | A propor por este módulo. |

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN**: cada tenant possui uma **conta NATS** própria; clientes autenticam com
  **NKey/JWT** emitidos pelo cofre do `021-Security`, com rotação sem downtime.
  Comunicação de serviço e pares A2A externos usam **mTLS** (RFC-0001 §6). O 020
  **não emite** credenciais — apenas as consome e as revoga quando instruído.
- **AuthZ**: o 020 é **PEP**. Operações administrativas (declarar stream, exportar
  subject entre contas, replay, provisionar conta) e **toda abertura de sessão A2A**
  consultam o **PDP** do `022-Policy` com ***default deny*** (`AIOS-BUS-0002` /
  `AIOS-A2A-0001`). Com o PDP indisponível, `bus.policy.fail_mode=closed` nega — mas
  o **tráfego já autorizado continua**, para que uma falha de política não derrube o
  barramento inteiro.
- **Autorização de publicação**: cada subject tem um `producer_module` declarado; a
  publicação por outro módulo é recusada com `AIOS-BUS-0010`. Isso impede falsificação
  de eventos de domínio alheio — um agente não consegue forjar
  `agent.lifecycle.terminated`.
- **Isolamento de tenant**: `tenant` no subject **DEVE** casar com a conta autenticada
  (`AIOS-BUS-0003`). Cruzar contas exige *export/import* explícito, autorizado e
  auditado (FR-019).

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no 020-Communication | Mitigação |
|-----------------|-----------------------------|-----------|
| **S**poofing | Agente publica evento fingindo ser outro módulo (ex.: forjar `policy.decision.allowed`). | `producer_module` declarado por subject (`AIOS-BUS-0010`); conta NATS por tenant; identidade do par A2A verificada por mTLS/JWT (`AIOS-A2A-0005`). |
| **T**ampering | Alteração de envelope em trânsito ou de configuração de stream para desviar tráfego. | TLS em todas as conexões; `EnvelopeValidator` no publish; declaração de stream versionada (OCC) e alterada só via PEP; auditoria em `025`. |
| **R**epudiation | Negar ter publicado uma mensagem ou aberto uma sessão A2A. | `source` no envelope + `traceparent`; toda sessão A2A registrada em `comm.a2a_session` e auditada com capacidades negociadas. |
| **I**nformation disclosure | Assinar `aios.>` e ler tráfego de outros tenants; PII em payload ou na DLQ. | Conta por tenant (roteamento nem enxerga a outra conta); curinga restrito ao escopo autorizado; minimização obrigatória no payload (RFC-0001 §7); DLQ retém envelope sem PII e expira em 30 dias. |
| **D**enial of service | Produtor inundando o cluster; difusão explosiva; *slow consumer* travando entrega; payload gigante. | Cotas por tenant/agente (`AIOS-BUS-0007`), `fanout_limit` (`AIOS-BUS-0008`), `max_ack_pending` (`AIOS-BUS-0014`), `max_payload_bytes` (`AIOS-BUS-0006`), `max_subscriptions_agent`. |
| **E**levation of privilege | Criar *export* de subject entre contas para exfiltrar dados; abrir sessão A2A com capacidade não concedida. | Export exige capability específica e é auditado (FR-019); capacidades A2A são autorizadas item a item pelo PDP e **NÃO DEVEM** ser ampliadas após `Established` (invariante I4). |

### 12.3 LGPD / GDPR

- **Minimização**: o barramento **NÃO DEVE** ser usado para trafegar dado pessoal além
  do necessário; payloads carregam URNs opacos e referências (RFC-0001 §7). Quem
  precisa de conteúdo sensível referencia o objeto/registro no módulo dono.
- **Retenção**: streams têm `max_age` explícito (default 7 dias); a DLQ expira em 30
  dias. O barramento **não é arquivo**: retenção longa pertence a `025-Audit` e
  `005-Database`.
- **Direito ao esquecimento**: eventos já publicados que contenham referência a um
  titular expiram pela retenção do stream; o expurgo do dado em si é executado pelo
  `005-Database` (UC de expurgo). O 020 **não** reescreve mensagens já entregues —
  imutabilidade do log é propriedade do JetStream — e essa limitação é a razão de a
  minimização no payload ser obrigatória, não recomendada.
- **Segregação**: contas NATS por tenant garantem que o tráfego de um cliente jamais
  seja roteado por caminhos compartilhados de assinatura com outro.
- **Trilha**: sessões A2A (com quem, quando, quais capacidades, quanto volume) são
  registradas e auditáveis — inclusive as com pares externos.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `../002-ADR/README.md` (ADR-0004 fundadora)
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../004-API/`, `../005-Database/`, `../006-Kernel/`,
  `../007-Agent-Runtime/`, `../008-Agent-Lifecycle/`, `../009-Scheduler/`,
  `../014-Workflow/`, `../021-Security/`, `../022-Policy/`, `../024-Observability/`,
  `../025-Audit/`, `../026-Cost-Optimizer/`, `../027-Cluster/`, `../028-Deployment/`.

*Fim do Design Brief interno do módulo 020-Communication. Este documento governa a
geração dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e
este brief é um defeito do documento gerado.*
