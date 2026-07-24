---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0001, ADR-0002, ADR-0008, ADR-0010 (globais, herdados); ADR-0220..ADR-0229 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — não redefinida); RFC-0220 (Policy Decision Contract), RFC-0221 (Policy Bundle Format & Distribution), a propor por este módulo
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database (schema `policy`), 020-Communication (NATS request-reply e JetStream), 021-Security (identidade e atributos assertáveis), 024-Observability, 025-Audit, 026-Cost-Optimizer (atributos de orçamento); consumido como PDP por **todos** os módulos que possuem PEP — 004-API, 006-Kernel, 007-Agent-Runtime, 008-Agent-Lifecycle, 009-Scheduler, 010-Memory, 011-Context, 015-Tool-Manager, 017-Model-Router, 020-Communication, 021-Security
---

# 022-Policy — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 022-Policy. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, subjects, **obrigatoriedade de PEP/PDP
> com *default deny***), este documento **referencia** a
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.8 e a
> `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`, e **não os redefine**.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 0. Posição no AIOS e contrato de fronteira

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  PEPs (pontos de aplicação) — em TODO módulo com ação privilegiada        │
   │  004 RouteAuthorizer · 006 CapabilityEnforcer · 007 PEP local ·          │
   │  009 AdmissionController · 010 MemoryPep · 015/017 brokers ·             │
   │  020 SubscriptionBroker/A2AGateway · 021 operações administrativas       │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ DecisionRequest (subject, action, resource, environment)
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  022-POLICY — PONTO DE DECISÃO (PDP) (este módulo)                       │
   │  · avalia RBAC + ABAC sobre um BUNDLE de política versionado e assinado  │
   │  · responde allow|deny com MOTIVO, OBRIGAÇÕES e TTL de cache             │
   │  · governa o ciclo de vida da política: autoria → teste → simulação →    │
   │    publicação → rollback                                                 │
   │  · NÃO aplica a decisão (quem aplica é o PEP) e NÃO estabelece identidade│
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ decisão + trilha de explicabilidade
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  021-SECURITY  — forneceu QUEM é o sujeito e seus atributos assertáveis  │
   │  026-COST      — forneceu atributos de orçamento consumido               │
   │  025-AUDIT     — registra QUE a decisão ocorreu, de forma imutável       │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Fronteira em uma frase:** *o `021` responde **quem é** o solicitante; o `022`
responde **se ele pode**; o PEP do módulo chamador **impede ou deixa passar**; o
`025` registra **que aconteceu**.* O 022 nunca executa a ação que autoriza — e essa
separação é o que permite auditar a autorização independentemente de quem a aplica.

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O 022-Policy é o **monitor de referência** do AIOS — o análogo do subsistema de
controle de acesso obrigatório de um sistema operacional endurecido, com uma
diferença essencial: as regras não estão compiladas no núcleo, são **dado
versionado**. Todo caminho privilegiado do sistema — abrir uma syscall cognitiva,
gastar orçamento, ler memória de outro agente, invocar uma ferramenta, assinar uma
sessão A2A, emitir uma credencial — passa por uma pergunta que só este módulo
responde: *este sujeito pode executar esta ação sobre este recurso, neste contexto?*

A primeira tese do módulo é que **política é dado, não código de aplicação**: ela é
autorada, versionada, compilada, testada, simulada, assinada, publicada e
**revertida** como um artefato de release próprio (§4). Espalhar `if (user.IsAdmin)`
pelos serviços produz um sistema cuja postura de segurança ninguém consegue
enunciar — muito menos provar a um auditor.

A segunda tese é que **a decisão precisa ser explicável**. Um `deny` sem motivo é
indistinguível de um defeito, e a reação operacional a um `deny` inexplicado é
sempre a mesma: afrouxar a política. Por isso toda resposta deste módulo carrega
`reason_code`, as regras casadas e a versão exata do bundle que a produziu — e essa
combinação é suficiente para **reproduzir** a decisão *a posteriori* (NFR-009).

A terceira tese é que **ausência de política é negação, não permissão**. Um tenant
sem bundle ativo não é um tenant liberado: é um tenant que nega tudo (invariante
I5). *Default deny* (ADR-0008, RFC-0001 §5.8) não é um default de configuração —
é o comportamento do módulo quando falta configuração, quando falta atributo, quando
falta tempo de avaliação e quando falta o próprio PDP.

### 1.2 Responsabilidades (o 022-Policy DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Ser o **PDP** único do AIOS: avaliar `DecisionRequest` (sujeito, ação, recurso, ambiente) e responder `allow`/`deny` sob ***default deny***. |
| R-02 | Avaliar **RBAC** (papéis e vínculos) e **ABAC** (atributos de sujeito, recurso, ação e ambiente) no mesmo ciclo de decisão, com precedência determinística (§4.6). |
| R-03 | Retornar toda decisão com **explicabilidade**: `reason_code`, regras casadas, versão do bundle e `decision_id` correlacionável. |
| R-04 | Retornar **obrigações** (`obligations`) que o PEP DEVE cumprir para que o `allow` seja válido (ex.: redigir PII, exigir MFA, limitar tokens, impor perfil de sandbox). |
| R-05 | Retornar um **TTL de cache** por decisão, permitindo ao PEP memorizar sem perder capacidade de revogação rápida. |
| R-06 | Governar o **ciclo de vida do policy bundle** (§4): autoria, compilação, análise de conflitos, teste, simulação, assinatura, publicação, supersessão e **rollback**. |
| R-07 | Distribuir o bundle ativo a todas as réplicas de avaliação e sinalizar a mudança por `policy.bundle.updated`, invalidando os caches dos PEPs. |
| R-08 | Executar **testes de política** obrigatórios (casos golden) e barrar a publicação de bundle que não atinja a cobertura mínima. |
| R-09 | Executar **simulação/dry-run** de um bundle candidato contra tráfego de decisão registrado, produzindo o **diferencial** de decisões antes de qualquer publicação. |
| R-10 | Detectar **conflitos e regras inalcançáveis** (contradição, sombreamento, permissão redundante) na compilação. |
| R-11 | Resolver **atributos** de fontes externas (`021` princípios, `026` orçamento, `024` sinais, contexto da requisição) com cache curto e política de falha explícita. |
| R-12 | Gerir **exceções (waivers)** temporárias, com prazo máximo, aprovador identificado e expiração automática. |
| R-13 | Manter o **journal de decisões** de curta retenção para explicabilidade e simulação, e emitir a trilha correspondente para `../025-Audit/`. |
| R-14 | Emitir `policy.decision.denied` e `policy.decision.updated` para observabilidade e invalidação granular de cache nos PEPs. |
| R-15 | Autorizar as **suas próprias** operações administrativas por um bundle-raiz de autogoverno, com separação de funções e aprovação dupla (§12.1). |
| R-16 | Expor a decisão pelas três superfícies exigidas pelos chamadores: **gRPC** (interno, caminho quente), **REST** (administração e ferramentas) e **NATS request-reply** (`aios.<tenant>.policy.decision.request`). |

### 1.3 Não-Responsabilidades (o 022-Policy NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | **Aplicar** a decisão — bloquear a chamada, cortar o tráfego, derrubar o agente. O 022 decide; o PEP aplica. | módulo chamador (PEP) |
| NR-02 | Estabelecer **identidade**, autenticar sujeito ou emitir/validar credencial. O 022 **consome** claims e atributos já assertados. | `021-Security` |
| NR-03 | Definir o **modelo de papéis do negócio** de um tenant como código do módulo — os papéis são dado do tenant, não constante do produto. | tenant (via API de autoria) |
| NR-04 | Persistir a **trilha de auditoria imutável** (o 022 emite os fatos; o Audit os torna prova). | `025-Audit` |
| NR-05 | Definir ou contabilizar **orçamento e custo**. O 022 **lê** o consumo como atributo ABAC; o limite e a contabilidade são do dono. | `026-Cost-Optimizer` |
| NR-06 | Aplicar **cotas e rate-limit** de recurso (tokens, syscalls/s, memória). | `006-Kernel`, `026-Cost-Optimizer` |
| NR-07 | Executar o **isolamento** (sandbox, seccomp, cgroups) — o 022 PODE exigir um perfil por obrigação, mas não o aplica nem o publica. | `007-Agent-Runtime` (aplica), `021-Security` (publica) |
| NR-08 | Decidir **placement, prioridade ou preempção** de tarefa; o 022 apenas autoriza a operação de preempção quando consultado. | `009-Scheduler` |
| NR-09 | Classificar dado (`data_class`) ou aplicar **RLS**; o 022 usa a classificação como atributo. | `005-Database` + módulo dono |
| NR-10 | Detectar ameaça ou anomalia comportamental; o 022 **consome** o sinal como atributo. | `021-Security`, `025-Audit`, `024-Observability` |
| NR-11 | Responder a incidente (bloquear sujeito, encerrar sessão, acionar plantão). O 022 muda a **política**; a resposta operacional é de outro. | `029-Operations` |
| NR-12 | Ser cache de decisão dos PEPs. O TTL é **emitido** pelo 022; o cache vive no PEP. | módulo chamador (PEP) |
| NR-13 | Servir como *feature store* de atributos ou fonte da verdade de qualquer atributo que resolve. | módulo dono do atributo |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **DecisionGateway** | *Front door* da decisão nas três superfícies (gRPC, REST, NATS request-reply); valida schema do `DecisionRequest`, extrai correlação (RFC-0001 §5.6), rejeita `tenant` divergente e aplica limite de avaliações. | DecisionEngine, PolicyTelemetry; ← 020-Communication |
| **DecisionEngine** | Orquestra o ciclo de avaliação: normaliza a ação, resolve atributos, executa RBAC e ABAC, combina resultados pela precedência canônica (§4.6), monta obrigações e calcula o TTL. É **puro** dado o bundle e os atributos — mesma entrada, mesma saída (NFR-009). | ActionNormalizer, RbacResolver, AbacEvaluator, AttributeResolver, ObligationComposer, BundleRuntime |
| **ActionNormalizer** | Normaliza o identificador de ação para a forma canônica `<dominio>:<objeto>:<verbo>`, aceitando as formas curtas legadas dos PEPs (ex.: verbo de syscall do `006`) sem exigir mudança do chamador. | BundleRuntime |
| **RbacResolver** | Resolve papéis efetivos do sujeito (vínculos diretos, herança de papel, papéis derivados de grupo) e as permissões que eles concedem. | BundleRuntime; → PostgreSQL (`policy.role*`) |
| **AbacEvaluator** | Avalia as condições sobre atributos de sujeito, recurso, ação e ambiente; opera sobre expressões **compiladas**, sem interpretação de texto no caminho quente. | BundleRuntime, AttributeResolver |
| **AttributeResolver** | Resolve atributos externos (princípio/atributos do `021`, orçamento do `026`, sinais do `024`, classificação do dono do dado) com cache curto, *timeout* rígido e criticidade declarada por fonte. Atributo **obrigatório** não resolvido ⇒ `indeterminate` ⇒ `deny`. | AttributeCache; → 021, 024, 026 |
| **ObligationComposer** | Compõe as obrigações do `allow`, deduplica, resolve conflitos entre obrigações (a mais restritiva vence) e impõe o limite máximo por decisão. | BundleRuntime |
| **BundleRuntime** | Mantém em memória o bundle **compilado e ativo** por tenant (índice de regras por ação/recurso, expressões compiladas, grafo de papéis); troca atômica de versão sem interromper avaliações em curso. | BundleRegistry, BundleDistributor |
| **BundleCompiler** | Compila o bundle-fonte em artefato executável: valida sintaxe e referências, resolve herança de papéis, indexa regras, calcula métricas de complexidade e rejeita bundle acima dos limites. | ConflictAnalyzer; → PostgreSQL |
| **ConflictAnalyzer** | Detecta contradições (`allow` e `deny` para o mesmo par ação/recurso sem prioridade), sombreamento e regras inalcançáveis; em `strict` bloqueia a compilação, em `warn` anota. | BundleCompiler |
| **BundleRegistry** | Registro autoritativo e versionado dos bundles e seu estado na FSM (§4); guarda o artefato compilado, a assinatura e a proveniência (quem autorou, quem aprovou). | → PostgreSQL (`policy.policy_bundle`), 021 (assinatura) |
| **BundleDistributor** | Ativa o bundle, propaga a nova versão a todas as réplicas e emite `policy.bundle.updated`; garante que nenhuma réplica sirva versão anterior após a janela de propagação. | BundleRegistry, EventEmitter |
| **PolicyTestRunner** | Executa os casos golden (`policy.policy_test`) contra o bundle candidato; mede cobertura de regras exercitadas; barra a publicação abaixo do mínimo configurado. | BundleCompiler, BundleRuntime |
| **SimulationEngine** | Reavalia tráfego registrado no `DecisionJournal` contra um bundle candidato e produz o **diferencial** (`allow→deny`, `deny→allow`) por regra e por sujeito, sem qualquer efeito em produção. | DecisionEngine (modo sombra), DecisionJournal |
| **ExceptionManager** | Concede, renova e revoga exceções temporárias (waivers) com prazo máximo, aprovador e escopo; expira automaticamente e emite `policy.decision.updated`. | BundleRuntime, EventEmitter; → PostgreSQL |
| **DecisionJournal** | Registra decisões para explicabilidade e simulação: **100% dos `deny`** e uma amostra dos `allow`; retenção curta, com a trilha imutável delegada ao `025`. | → PostgreSQL (particionado), 025-Audit |
| **ExplainService** | Responde "por que esta decisão?": reproduz a avaliação sobre a versão exata do bundle e devolve o caminho de regras casadas e atributos usados. | BundleRegistry, DecisionJournal, DecisionEngine |
| **SelfGovernanceGuard** | PEP **interno** do próprio módulo: autoriza autoria, publicação, rollback e concessão de exceção contra o **bundle-raiz** imutável, com separação de funções e aprovação dupla (§12.1). | BundleRuntime, BundleRegistry |
| **EventEmitter** | Publica eventos CloudEvents via **Outbox transacional** em `policy.outbox`. | → PostgreSQL, NATS/JetStream |
| **PolicyTelemetry** | Spans/métricas `aios_pol_*`/logs OTel; emissão de auditoria para `025`, **sem** vazar valores de atributo sensíveis. | → 024-Observability, 025-Audit |

### 2.2 Diagrama de Componentes (ASCII)

```
    gRPC aios.policy.v1        REST /v1/policy        NATS aios.<t>.policy.decision.request
              │                       │                            │
   ┌──────────▼───────────────────────▼────────────────────────────▼─────────────┐
   │                     POLICY SERVICE (022 · .NET 10)                            │
   │                                                                               │
   │  ┌────────────────────┐                                                       │
   │  │  DecisionGateway   │  valida · correlaciona · limita                        │
   │  └─────────┬──────────┘                                                       │
   │            ▼                                                                  │
   │  ┌────────────────────┐    ┌──────────────────┐    ┌──────────────────────┐  │
   │  │   DecisionEngine   │───▶│ ActionNormalizer │    │  AttributeResolver   │──┼─▶ 021 (atributos)
   │  │  (ciclo puro §4.6) │    └──────────────────┘    │  (timeout, cache,    │  │  026 (orçamento)
   │  └───┬────────┬───────┘                            │   criticidade)       │  │  024 (sinais)
   │      │        │           ┌──────────────────┐     └──────────────────────┘  │
   │      │        ├──────────▶│   RbacResolver   │                                │
   │      │        │           └──────────────────┘                                │
   │      │        ├──────────▶┌──────────────────┐    ┌──────────────────────┐   │
   │      │        │           │  AbacEvaluator   │    │  ObligationComposer  │   │
   │      │        │           └──────────────────┘    └──────────────────────┘   │
   │      │        ▼                                                               │
   │      │  ┌──────────────────────────────────────────────────────────────┐     │
   │      │  │  BundleRuntime  (bundle COMPILADO e ATIVO, em memória)        │     │
   │      │  └───────▲──────────────────────────────▲───────────────────────┘     │
   │      │          │ troca atômica                │ ativa                        │
   │      │  ┌───────┴────────┐  ┌───────────────┐  │  ┌────────────────────────┐ │
   │      │  │ BundleCompiler │─▶│ BundleRegistry│──┴─▶│  BundleDistributor     │─┼─▶ PEPs
   │      │  │  + Conflict    │  │ (versões, FSM,│     │ (propaga + evento)     │ │  (invalidam
   │      │  │    Analyzer    │  │  assinatura)  │     └────────────────────────┘ │   cache)
   │      │  └────────────────┘  └───────┬───────┘                                │
   │      │          ▲                   │                                        │
   │      │  ┌───────┴────────┐  ┌───────▼────────┐  ┌────────────────────────┐  │
   │      │  │ PolicyTestRunner│  │SimulationEngine│  │   ExceptionManager     │  │
   │      │  │ (gate golden)   │  │ (diferencial)  │  │  (waiver com prazo)    │  │
   │      │  └─────────────────┘  └───────┬────────┘  └────────────────────────┘  │
   │      ▼                               │                                        │
   │  ┌────────────────────┐  ┌───────────▼──────┐  ┌────────────────────────┐   │
   │  │  DecisionJournal   │◀─│  ExplainService  │  │ SelfGovernanceGuard    │   │
   │  │ (deny 100% · allow │  │ (reproduz decisão│  │ (bundle-raiz, SoD,     │   │
   │  │  amostrado)        │  │  por versão)     │  │  aprovação dupla)      │   │
   │  └────────────────────┘  └──────────────────┘  └────────────────────────┘   │
   │  ┌─────────────────────────────────────────────────────────────────────────┐│
   │  │ EventEmitter (Outbox → JetStream)  ·  PolicyTelemetry (OTel + auditoria) ││
   │  └─────────────────────────────────────────────────────────────────────────┘│
   └──────────┬───────────────────────┬────────────────────────┬─────────────────┘
              ▼                       ▼                        ▼
      PostgreSQL (schema policy)   Redis (cache atributo)   NATS aios.*.policy.*
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN, ULID) e às convenções físicas do
> `../005-Database/Database.md` §2: `tenant_id` + RLS em toda tabela multi-tenant,
> `version bigint` para OCC, `timestamptz` em UTC. Schema: **`policy`**.
>
> **Regra deste modelo:** a fonte da verdade é o **bundle**, não o banco de decisões.
> Nenhuma decisão é persistida como autorização durável: decisão é efêmera, tem TTL
> e é sempre reproduzível a partir de (bundle, atributos, requisição).

### 3.1 Entidade `PolicyBundle` — tabela `policy.policy_bundle` (multi-tenant, RLS)

Unidade de release da política. É a entidade governada pela FSM da §4.

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>` (tipo `policy`, RFC-0001 §5.1). |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento. |
| `scope` | `text` | NOT NULL, CHECK ∈ {tenant, platform} | `platform` reservado ao bundle-raiz de autogoverno (§12.1). |
| `bundle_version` | `int` | NOT NULL, UNIQUE(tenant_id,scope,bundle_version) | Versão monotônica por tenant+escopo. |
| `state` | `text` | NOT NULL, CHECK ∈ BundleState | Estado da FSM (§4.1). |
| `source_digest` | `text` | NOT NULL | `sha256` do bundle-fonte (regras + papéis + vínculos). |
| `compiled_artifact` | `bytea` | NULL | Artefato compilado (índices e expressões); NULL enquanto `Draft`. |
| `compiled_digest` | `text` | NULL | `sha256` do artefato compilado — chave de reprodutibilidade. |
| `signature` | `bytea` | NULL | Assinatura do artefato (chave `signing` do `021`). |
| `signed_by` | `text` | NULL | URN da chave que assinou (`../021-Security/`). |
| `rule_count` | `int` | NOT NULL default 0 | Regras no bundle (limite: `pol.bundle.max_rules`). |
| `test_coverage` | `numeric(5,4)` | NULL | Fração de regras exercitadas pelos testes golden. |
| `authored_by` | `text` | NOT NULL | Princípio autor. |
| `approved_by` | `text` | NULL | Princípio aprovador — **DEVE** diferir de `authored_by` (§12.1). |
| `activated_at` | `timestamptz` | NULL | Momento em que entrou em vigor. |
| `superseded_by` | `text` | NULL, FK→policy.policy_bundle.urn | Versão sucessora. |
| `rollback_of` | `text` | NULL, FK→policy.policy_bundle.urn | Preenchido quando esta ativação é um rollback. |
| `idempotency_key` | `text` | NULL, UNIQUE(tenant_id,idempotency_key) | RFC-0001 §5.5. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, scope, bundle_version)`;
`UNIQUE(tenant_id, scope) WHERE state = 'Active'` (impõe fisicamente a invariante I1);
`btree(tenant_id, state)`; `btree(compiled_digest)`.

> O índice único parcial sobre `state = 'Active'` **não é otimização**: é a
> materialização da invariante "no máximo um bundle ativo por tenant+escopo". Deixar
> essa garantia apenas na aplicação é convidar duas verdades simultâneas de política.

### 3.2 Entidade `PolicyRule` — tabela `policy.policy_rule` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>` (regra pertence ao domínio `policy`). |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `bundle_urn` | `text` | NOT NULL, FK→policy.policy_bundle.urn | Bundle dono (regra é **imutável** após sair de `Draft`). |
| `rule_key` | `text` | NOT NULL, UNIQUE(bundle_urn,rule_key) | Identificador legível e estável (ex.: `mem.read.own-agent`). |
| `effect` | `text` | NOT NULL, CHECK ∈ {allow, deny} | Efeito da regra. |
| `priority` | `int` | NOT NULL default 100 | Desempate explícito (§4.6); menor valor decide primeiro. |
| `action_pattern` | `text` | NOT NULL | Padrão canônico `<dominio>:<objeto>:<verbo>`, com curinga por segmento (ex.: `memory:*:read`). |
| `resource_pattern` | `text` | NOT NULL | Padrão de URN do recurso (ex.: `urn:aios:{tenant}:memory:*`). |
| `subject_match` | `jsonb` | NOT NULL default '{}' | Restrição por papel, grupo ou tipo de princípio (RBAC). |
| `condition` | `jsonb` | NOT NULL default '{}' | Expressão ABAC sobre atributos (AST compilável, não texto livre). |
| `obligations` | `jsonb` | NOT NULL default '[]' | Obrigações impostas quando a regra decide `allow`. |
| `decision_ttl_ms` | `int` | NULL | TTL sugerido para esta decisão; NULL usa o default de configuração. |
| `description` | `text` | NOT NULL | Intenção da regra em linguagem natural — obrigatória para revisão. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(urn)`; `UNIQUE(bundle_urn, rule_key)`;
`btree(bundle_urn, priority)`; `gin(condition jsonb_path_ops)`.

### 3.3 Entidade `Role` — tabela `policy.role` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `bundle_urn` | `text` | NOT NULL, FK→policy.policy_bundle.urn | Bundle dono. |
| `role_key` | `text` | NOT NULL, UNIQUE(bundle_urn,role_key) | Nome do papel (ex.: `agent.operator`). |
| `inherits` | `text[]` | NOT NULL default '{}' | Papéis herdados (grafo **DEVE** ser acíclico — validado na compilação). |
| `permissions` | `jsonb` | NOT NULL default '[]' | Pares `{action_pattern, resource_pattern}` concedidos. |
| `is_privileged` | `boolean` | NOT NULL default false | Papel sensível: exige aprovação dupla para vincular (§12.1). |
| `description` | `text` | NOT NULL | Intenção do papel. |

Índices: `PK(urn)`; `UNIQUE(bundle_urn, role_key)`; `btree(tenant_id, is_privileged)`.

### 3.4 Entidade `RoleBinding` — tabela `policy.role_binding` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `bundle_urn` | `text` | NOT NULL, FK→policy.policy_bundle.urn | Bundle dono. |
| `role_key` | `text` | NOT NULL | Papel vinculado. |
| `subject_urn` | `text` | NOT NULL | Princípio, grupo ou agente (`urn:aios:<tenant>:principal\|agent:<ULID>`). |
| `subject_kind` | `text` | NOT NULL, CHECK ∈ {principal, group, agent, service} | Natureza do vínculo. |
| `scope_pattern` | `text` | NOT NULL default '*' | Restringe o vínculo a um subconjunto de recursos. |
| `not_after` | `timestamptz` | NULL | Vínculo com prazo (NULL = permanente enquanto o bundle viger). |
| `granted_by` | `text` | NOT NULL | Quem concedeu — insumo de auditoria. |

Índices: `PK(urn)`; `UNIQUE(bundle_urn, role_key, subject_urn, scope_pattern)`;
`btree(tenant_id, subject_urn)`; `btree(not_after)`.

### 3.5 Entidade `AttributeSource` — tabela `policy.attribute_source`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:policy:<ULID>`. |
| `source_key` | `text` | NOT NULL, UNIQUE | Prefixo do atributo (ex.: `principal`, `budget`, `threat`, `resource`). |
| `provider_module` | `text` | NOT NULL | Módulo dono (`021`, `026`, `024`, ...). |
| `criticality` | `text` | NOT NULL, CHECK ∈ {required, optional} | `required` não resolvido ⇒ `indeterminate` ⇒ `deny`. |
| `resolve_timeout_ms` | `int` | NOT NULL default 20 | Orçamento de resolução. |
| `cache_ttl_s` | `int` | NOT NULL default 30 | TTL do cache do atributo. |
| `sensitive` | `boolean` | NOT NULL default false | Valor **NÃO DEVE** aparecer em log, métrica ou journal (apenas o `sha256`). |
| `status` | `text` | NOT NULL, CHECK ∈ {active, degraded, disabled} | Situação da fonte. |

Índices: `PK(urn)`; `UNIQUE(source_key)`; `btree(status)`.

### 3.6 Entidade `PolicyException` — tabela `policy.policy_exception` (multi-tenant, RLS)

Waiver temporário: a válvula de escape que impede que a urgência operacional vire
afrouxamento permanente da política.

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `subject_urn` | `text` | NOT NULL | Sujeito beneficiado. |
| `action_pattern` | `text` | NOT NULL | Ação liberada. |
| `resource_pattern` | `text` | NOT NULL | Recurso alvo. |
| `justification` | `text` | NOT NULL | Motivo declarado — obrigatório. |
| `requested_by` | `text` | NOT NULL | Solicitante. |
| `approved_by` | `text` | NOT NULL | Aprovador — **DEVE** diferir de `requested_by`. |
| `effective_from` | `timestamptz` | NOT NULL | Início. |
| `expires_at` | `timestamptz` | NOT NULL | Fim — **sempre preenchido**, limitado por `pol.exception.max_ttl_h`. |
| `revoked_at` | `timestamptz` | NULL | Revogação antecipada. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `btree(tenant_id, subject_urn, expires_at)`; `btree(expires_at)`.

> `expires_at` é NOT NULL **por desenho**, pela mesma razão que `expires_at` de
> credencial é NOT NULL no `021`: exceção sem prazo é política nova aprovada por
> ninguém.

### 3.7 Entidade `DecisionRecord` — tabela `policy.decision_journal` (multi-tenant, RLS, particionada)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `decision_id` | `char(26)` | PK (ULID) parte da chave composta | Identidade da decisão, devolvida ao PEP. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `decided_at` | `timestamptz` | NOT NULL, chave de partição | Momento da decisão (partição `RANGE`). |
| `subject_urn` | `text` | NOT NULL | Sujeito. |
| `action` | `text` | NOT NULL | Ação canônica. |
| `resource_urn` | `text` | NOT NULL | Recurso. |
| `effect` | `text` | NOT NULL, CHECK ∈ {allow, deny} | Efeito **externo** (§4.6: `indeterminate`/`not_applicable` colapsam em `deny`). |
| `reason_code` | `text` | NOT NULL | Motivo estável (§4.7). |
| `matched_rules` | `text[]` | NOT NULL default '{}' | `rule_key` das regras casadas. |
| `bundle_urn` | `text` | NOT NULL | Bundle que decidiu. |
| `bundle_version` | `int` | NOT NULL | Versão — chave de reprodutibilidade. |
| `attributes_digest` | `text` | NOT NULL | `sha256` do conjunto de atributos usados (valores sensíveis **não** são gravados). |
| `obligations` | `jsonb` | NOT NULL default '[]' | Obrigações emitidas. |
| `latency_us` | `int` | NOT NULL | Latência da avaliação. |
| `pep_module` | `text` | NOT NULL | Módulo chamador. |
| `trace_id` | `text` | NOT NULL | Correlação W3C (RFC-0001 §5.6). |

Chave primária: `PRIMARY KEY (tenant_id, decided_at, decision_id)`.
Índices: `btree(tenant_id, subject_urn, decided_at DESC)`;
`btree(tenant_id, effect, decided_at DESC)`; `btree(bundle_version, decided_at DESC)`.

> Retenção curta (`pol.journal.retention_days`, default 30). A prova de longo prazo
> é da trilha imutável do `../025-Audit/` — duplicar retenção aqui seria criar uma
> segunda fonte da verdade de auditoria, com regras de expurgo divergentes.

### 3.8 Entidade `PolicyTest` — tabela `policy.policy_test` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `test_key` | `text` | NOT NULL, UNIQUE(tenant_id,test_key) | Nome do caso golden. |
| `request` | `jsonb` | NOT NULL | `DecisionRequest` de entrada (sujeito, ação, recurso, ambiente, atributos fixados). |
| `expected_effect` | `text` | NOT NULL, CHECK ∈ {allow, deny} | Efeito esperado. |
| `expected_reason` | `text` | NULL | `reason_code` esperado (quando o motivo importa). |
| `is_mandatory` | `boolean` | NOT NULL default true | Caso obrigatório: falha bloqueia a publicação. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, test_key)`; `btree(tenant_id, is_mandatory)`.

### 3.9 Entidade `SimulationRun` — tabela `policy.simulation_run` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:policy:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `candidate_bundle` | `text` | NOT NULL, FK→policy.policy_bundle.urn | Bundle candidato. |
| `baseline_bundle` | `text` | NOT NULL, FK→policy.policy_bundle.urn | Bundle ativo usado como linha de base. |
| `window_from` | `timestamptz` | NOT NULL | Início da janela de tráfego reavaliado. |
| `window_to` | `timestamptz` | NOT NULL | Fim da janela. |
| `evaluated_count` | `bigint` | NOT NULL default 0 | Requisições reavaliadas. |
| `flipped_to_deny` | `bigint` | NOT NULL default 0 | `allow` → `deny` (risco de quebra operacional). |
| `flipped_to_allow` | `bigint` | NOT NULL default 0 | `deny` → `allow` (risco de ampliação de acesso). |
| `diff_by_rule` | `jsonb` | NOT NULL default '{}' | Diferencial agregado por `rule_key`. |
| `status` | `text` | NOT NULL, CHECK ∈ {running, completed, failed} | Situação. |
| `completed_at` | `timestamptz` | NULL | Conclusão. |

Índices: `PK(urn)`; `btree(tenant_id, candidate_bundle)`; `btree(status)`.

### 3.10 Relações (ASCII)

```
   policy.policy_bundle(1) ──< policy.policy_rule(*)
        │  │                                         (regras imutáveis após Draft)
        │  ├──< policy.role(*) ──[inherits]──▶ policy.role   (grafo acíclico)
        │  │         ▲
        │  │         │ role_key
        │  └──< policy.role_binding(*) ──▶ subject_urn (principal|group|agent|service)
        │
        ├──[superseded_by]──▶ policy.policy_bundle (sucessora)
        ├──[rollback_of]────▶ policy.policy_bundle (versão restaurada)
        │
        └──< policy.simulation_run(*) ──[baseline_bundle]──▶ policy.policy_bundle

   policy.attribute_source(*) ──alimenta──▶ AbacEvaluator ──▶ policy.decision_journal(*)
   policy.policy_exception(*) ──sobrepõe (com prazo)──▶ decisão
   policy.policy_test(*)      ──barra publicação──────▶ policy.policy_bundle
```

---

## 4. Máquina de Estados Canônica do `PolicyBundle`

> A entidade com ciclo de vida governado é o **bundle de política**, não a decisão:
> a decisão é efêmera (existe pelo TTL e é reproduzível), enquanto o bundle é o
> artefato que precisa de autoria, revisão, prova, publicação e reversão. Tratar a
> política como release — e não como linha de configuração — é o que torna possível
> responder "o que mudou na segurança do sistema entre ontem e hoje?".

### 4.1 Estados (`BundleState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Draft` | Em autoria; regras, papéis e vínculos ainda mutáveis. | não |
| `Validating` | Compilação e análise de conflitos em curso. | não |
| `Validated` | Compila, sem conflito bloqueante; **imutável** a partir daqui. | não |
| `Tested` | Passou nos casos golden obrigatórios com cobertura ≥ mínima. | não |
| `Simulated` | Reavaliado contra tráfego registrado; diferencial disponível para revisão. | não |
| `Active` | Em vigor. No máximo **um** por (tenant, escopo). | não |
| `Superseded` | Substituído por versão posterior; retido para rollback e reprodutibilidade. | não |
| `RolledBack` | Estava ativo e foi revertido; **não** volta a vigorar sem nova publicação. | não |
| `Rejected` | Reprovado na compilação, no conflito ou nos testes. | **sim** |
| `Archived` | Fora da janela de retenção de rollback; mantido apenas como registro. | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Draft` | `CreateBundle` | Capability `pol:bundle:author` ∧ tenant do autor = tenant do bundle. |
| T-02 | `Draft` → `Validating` | `ValidateBundle` | `rule_count` ≤ `pol.bundle.max_rules` ∧ toda regra com `description` não vazia. |
| T-03 | `Validating` → `Validated` | compilação concluída | Sintaxe válida ∧ referências de papel/atributo resolvidas ∧ grafo de herança acíclico ∧ (nenhum conflito ∨ `pol.conflict.mode = warn`). |
| T-04 | `Validating` → `Rejected` | compilação falhou | Erro de sintaxe (`AIOS-POL-0004`) ∨ referência inexistente (`AIOS-POL-0012`) ∨ conflito em modo `strict` (`AIOS-POL-0008`) ∨ complexidade excedida (`AIOS-POL-0016`). |
| T-05 | `Validated` → `Tested` | `RunPolicyTests` | 100% dos casos `is_mandatory` passaram ∧ `test_coverage` ≥ `pol.test.min_coverage`. |
| T-06 | `Validated` → `Rejected` | testes reprovados | Qualquer caso obrigatório falhou ∨ cobertura abaixo do mínimo (`AIOS-POL-0007`). |
| T-07 | `Tested` → `Simulated` | `RunSimulation` | Existe bundle `Active` como linha de base ∧ janela de tráfego dentro da retenção do journal. |
| T-08 | `Simulated` → `Active` | `PublishBundle` | Capability `pol:bundle:publish` ∧ `approved_by ≠ authored_by` ∧ artefato assinado e assinatura válida (`AIOS-POL-0014`) ∧ diferencial revisado. |
| T-09 | `Tested` → `Active` | `PublishBundle` com `skip_simulation` | **Somente** no primeiro bundle do tenant (não há linha de base) ∨ escopo `platform` em *bootstrap* — sempre com aprovação dupla registrada. |
| T-10 | `Active` → `Superseded` | ativação de versão posterior | Sucessora em `Active` na mesma transação ∧ `superseded_by` preenchido (invariante I1 preservada). |
| T-11 | `Active` → `RolledBack` | `RollbackBundle` | Capability `pol:bundle:rollback` ∧ versão alvo em `Superseded` dentro da retenção (`AIOS-POL-0015`) ∧ alvo passa a `Active` na mesma transação. |
| T-12 | `Superseded` → `Active` | `RollbackBundle` (alvo) | Artefato compilado e assinatura ainda válidos ∧ dentro de `pol.bundle.rollback_retention_versions`. |
| T-13 | `Superseded`/`RolledBack` → `Archived` | expurgo de retenção | Fora da janela de rollback ∧ nenhuma simulação ou explicação pendente referenciando a versão. |
| T-14 | `Draft` → `Rejected` | `DiscardBundle` | Capability `pol:bundle:author`; descarte explícito do rascunho. |

**Invariantes:**
(I1) Existe **no máximo um** bundle em `Active` por `(tenant_id, scope)` — garantido
fisicamente pelo índice único parcial (§3.1). Dois é defeito, não degradação.
(I2) O bundle é **imutável** a partir de `Validated`: qualquer alteração exige nova
`bundle_version`. Editar política em vigor destrói a reprodutibilidade da decisão.
(I3) `Active` só é alcançável a partir de `Simulated` (fluxo normal), de `Tested`
(T-09, casos de *bootstrap*) ou de `Superseded` (rollback). **Nunca** de `Draft` ou
`Validated` — publicar o que não foi testado é o defeito que este módulo existe para
tornar impossível.
(I4) Toda transição para `Active` grava a mudança e emite
`aios.<tenant>.policy.bundle.updated` via Outbox **na mesma transação**.
(I5) Se **não** existir bundle `Active` para o `(tenant, scope)` solicitado, o PDP
responde `deny` a **toda** requisição, com `reason_code = no_active_bundle`. Ausência
de política é negação, nunca permissão.
(I6) Versões em `Superseded` **DEVEM** ser retidas com artefato compilado íntegro
enquanto estiverem na janela de rollback — sem o artefato exato, nem o rollback nem a
explicação de decisões passadas são possíveis.
(I7) Toda transição ocorre sob OCC pelo `version` esperado; conflito → `AIOS-POL-0006`.

### 4.3 Diagrama de estados (ASCII)

```
        CreateBundle (T-01)
   ∅ ──────────────────────▶ ┌─────────┐ ── DiscardBundle (T-14) ──┐
                              │  Draft  │                            │
                              └────┬────┘                            │
                    Validate (T-02)│                                 │
                                   ▼                                 │
                            ┌────────────┐   falha (T-04)            │
                            │ Validating │──────────────────────────▶│
                            └─────┬──────┘                           ▼
                       compila ok │(T-03)                     ┌────────────┐
                                  ▼                           │  Rejected  │ (terminal)
                            ┌────────────┐  testes falham     └────────────┘
                            │ Validated  │──────(T-06)───────────────▲
                            └─────┬──────┘                           │
                     testes ok (T-05)                                │
                                  ▼                                  │
                            ┌────────────┐                           │
                            │   Tested   │───── bootstrap (T-09) ────┼──┐
                            └─────┬──────┘                           │  │
                    simulação (T-07)                                 │  │
                                  ▼                                  │  │
                            ┌────────────┐                           │  │
                            │ Simulated  │                           │  │
                            └─────┬──────┘                           │  │
                     publica (T-08)                                  │  │
                                  ▼                                  │  ▼
                            ┌────────────┐  nova versão (T-10)  ┌──────────────┐
                            │   Active   │─────────────────────▶│  Superseded  │
                            └─────┬──────┘                      └──────┬───────┘
                     rollback (T-11)                    rollback alvo (T-12)
                                  ▼                                   │
                            ┌────────────┐                            │
                            │ RolledBack │                            │
                            └─────┬──────┘                            │
                                  │      expurgo de retenção (T-13)   │
                                  └───────────────┬───────────────────┘
                                                  ▼
                                          ┌────────────┐
                                          │  Archived  │ (terminal)
                                          └────────────┘
```

### 4.4 Ciclo de vida de uma decisão (efêmero, não persistido como estado)

```
   DecisionRequest ─▶ normaliza ação ─▶ resolve atributos ─▶ RBAC ─▶ ABAC
                                              │                        │
                                    (required faltando)          (regras casadas)
                                              │                        │
                                              ▼                        ▼
                                       indeterminate           combina por §4.6
                                              │                        │
                                              └───────▶ deny ◀─────────┘
                                                          │
                                        allow + obligations + ttl + reason_code
                                                          │
                                       ▼ journal (deny 100% · allow amostrado) ▼
                                                   resposta ao PEP
```

A decisão **não** vira estado durável em nenhuma tabela: ela expira pelo TTL no cache
do PEP e é reconstruível por `ExplainService` a partir de
(`bundle_version`, `DecisionRequest`, `attributes_digest`).

### 4.5 Efeitos internos e efeito externo

| Efeito interno | Significado | Efeito externo entregue ao PEP | `reason_code` típico |
|----------------|-------------|--------------------------------|----------------------|
| `allow` | Alguma regra `allow` casou e nenhuma `deny` de precedência maior. | `allow` | `matched_allow` |
| `deny` | Regra `deny` explícita casou. | `deny` | `explicit_deny` |
| `not_applicable` | Nenhuma regra casou. | **`deny`** | `no_matching_rule` |
| `indeterminate` | Atributo `required` indisponível, orçamento de avaliação estourado ou bundle ausente. | **`deny`** | `attribute_unavailable`, `eval_timeout`, `no_active_bundle` |

> O PEP **nunca** vê `not_applicable` nem `indeterminate`. Expor uma terceira via
> convidaria cada módulo a inventar o seu próprio tratamento — e algum deles trataria
> "não sei" como "pode".

### 4.6 Precedência canônica de combinação

Ordem determinística de avaliação — a mesma entrada produz sempre a mesma saída
(NFR-009):

1. **Bundle ausente ou inativo** ⇒ `deny` (`no_active_bundle`). Nada mais é avaliado.
2. **`deny` explícito de escopo `platform`** (bundle-raiz) vence qualquer regra de tenant.
3. **Menor `priority` decide**; em empate de `priority`, **`deny` vence `allow`**.
4. **Exceção (waiver) vigente** PODE converter `deny` em `allow`, **exceto** quando a
   regra `deny` estiver marcada como não-dispensável ou for de escopo `platform`.
5. Nenhuma regra casou ⇒ `deny` (`no_matching_rule`).

### 4.7 Códigos de motivo (`reason_code`) estáveis

`matched_allow` · `explicit_deny` · `no_matching_rule` · `no_active_bundle` ·
`attribute_unavailable` · `eval_timeout` · `role_not_bound` · `scope_mismatch` ·
`tenant_mismatch` · `binding_expired` · `exception_applied` · `exception_expired` ·
`obligation_unsatisfiable` · `platform_deny`.

Estes valores são **contrato**: aparecem na resposta, no journal, nas métricas e na
auditoria, e **NÃO DEVEM** ser renomeados sem incremento de versão da RFC-0220.

---

## 5. Superfície de API (REST, gRPC e NATS request-reply)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.policy.v1`. Base REST: `/v1/policy`. O caminho quente de decisão é
> **gRPC** (ou NATS request-reply, para chamadores que já vivem no barramento); REST
> é a superfície de autoria e operação.

### 5.1 Decisão (caminho quente)

| Operação | REST | gRPC (rpc) | NATS | Idempotente | Capability |
|----------|------|-----------|------|-------------|------------|
| Avaliar uma decisão | `POST /v1/policy/decisions:evaluate` | `Evaluate` | `aios.<tenant>.policy.decision.request` (request-reply) | sim (leitura pura) | `pol:decision:evaluate` |
| Avaliar em lote | `POST /v1/policy/decisions:evaluateBatch` | `EvaluateBatch` | — | sim (leitura pura) | `pol:decision:evaluate` |
| Explicar decisão | `GET /v1/policy/decisions/{decision_id}:explain` | `ExplainDecision` | — | sim (leitura) | `pol:decision:explain` |
| Consultar decisões do sujeito | `GET /v1/policy/decisions?subject=&from=&to=` | `ListDecisions` | — | sim (leitura) | `pol:decision:read` |

**`DecisionRequest` (campos mínimos — RFC-0220):** `subject` (URN + claims/papéis
propagados), `action`, `resource` (URN), `environment` (`tenant`, hora, origem,
prioridade, referências de cota/orçamento) e `attributes` opcionais fornecidos pelo
PEP. O conjunto é compatível com o já documentado em `../006-Kernel/Security.md` §3.1.

**`DecisionResponse`:** `effect` (`allow`|`deny`), `reason_code` (§4.7),
`matched_rules[]`, `obligations[]`, `decision_ttl_ms`, `bundle_version`,
`decision_id`, `evaluated_at`.

> **Regra normativa da obrigação:** um PEP que receba uma obrigação que **não sabe
> cumprir** DEVE tratar a decisão como `deny`. Ignorar obrigação desconhecida
> transformaria uma condição de segurança em comentário.

### 5.2 Autoria e ciclo de vida do bundle

| Operação | REST | gRPC (rpc) | Idempotente | Capability |
|----------|------|-----------|-------------|------------|
| Criar bundle (rascunho) | `POST /v1/policy/bundles` | `CreateBundle` | sim (key) | `pol:bundle:author` |
| Editar rascunho (regras/papéis/vínculos) | `PUT /v1/policy/bundles/{urn}/rules` | `PutBundleContent` | sim | `pol:bundle:author` |
| Validar/compilar | `POST /v1/policy/bundles/{urn}:validate` | `ValidateBundle` | sim | `pol:bundle:author` |
| Rodar testes golden | `POST /v1/policy/bundles/{urn}:test` | `RunPolicyTests` | sim | `pol:bundle:test` |
| Simular contra tráfego | `POST /v1/policy/bundles/{urn}:simulate` | `RunSimulation` | sim (key) | `pol:bundle:simulate` |
| Publicar (ativar) | `POST /v1/policy/bundles/{urn}:publish` | `PublishBundle` | sim (key) | `pol:bundle:publish` |
| Rollback para versão anterior | `POST /v1/policy/bundles/{urn}:rollback` | `RollbackBundle` | sim (key) | `pol:bundle:rollback` |
| Ler bundle / listar versões | `GET /v1/policy/bundles[/{urn}]` | `GetBundle`/`ListBundles` | sim (leitura) | `pol:bundle:read` |
| Ler diferencial de simulação | `GET /v1/policy/simulations/{urn}` | `GetSimulation` | sim (leitura) | `pol:bundle:simulate` |

### 5.3 Papéis, vínculos, exceções e atributos

| Operação | REST | gRPC (rpc) | Idempotente | Capability |
|----------|------|-----------|-------------|------------|
| Definir papel | `PUT /v1/policy/roles/{role_key}` | `PutRole` | sim | `pol:role:write` |
| Vincular sujeito a papel | `POST /v1/policy/role-bindings` | `CreateRoleBinding` | sim (key) | `pol:binding:write` |
| Remover vínculo | `DELETE /v1/policy/role-bindings/{urn}` | `DeleteRoleBinding` | sim | `pol:binding:write` |
| Listar permissões efetivas do sujeito | `GET /v1/policy/subjects/{urn}/effective` | `GetEffectivePermissions` | sim (leitura) | `pol:subject:read` |
| Conceder exceção (waiver) | `POST /v1/policy/exceptions` | `GrantException` | sim (key) | `pol:exception:grant` |
| Revogar exceção | `POST /v1/policy/exceptions/{urn}:revoke` | `RevokeException` | sim | `pol:exception:revoke` |
| Registrar fonte de atributo | `PUT /v1/policy/attribute-sources/{key}` | `PutAttributeSource` | sim | `pol:attribute:manage` |
| Gerir casos de teste | `PUT /v1/policy/tests/{test_key}` | `PutPolicyTest` | sim | `pol:bundle:test` |

### 5.4 Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínio reservado: **`POL`** (0001–0099), registrado no
> catálogo mantido por `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0220**).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-POL-0001` | 404 | não | Bundle, regra, papel, vínculo, exceção ou decisão não encontrado. |
| `AIOS-POL-0002` | 403 | não | Operação administrativa negada pelo autogoverno (*default deny*, §12.1). |
| `AIOS-POL-0003` | 403 | não | Tenant divergente do contexto autenticado (RFC-0001 §6). |
| `AIOS-POL-0004` | 422 | não | Bundle não compila: erro de sintaxe ou expressão inválida. |
| `AIOS-POL-0005` | 409 | não | Transição de estado inválida do bundle (viola a FSM §4). |
| `AIOS-POL-0006` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-POL-0007` | 422 | não | Testes obrigatórios reprovados ou cobertura abaixo do mínimo. |
| `AIOS-POL-0008` | 409 | não | Conflito de regras detectado em modo `strict` (contradição, sombreamento). |
| `AIOS-POL-0009` | 503 | sim | Fonte de atributo `required` indisponível: decisão indeterminada ⇒ `deny`. |
| `AIOS-POL-0010` | 429 | sim | Limite de avaliações por tenant excedido. |
| `AIOS-POL-0011` | 504 | sim | Orçamento de tempo de avaliação estourado; a decisão devolvida é `deny`. |
| `AIOS-POL-0012` | 422 | não | Referência a papel, atributo ou fonte inexistente no bundle. |
| `AIOS-POL-0013` | 403 | não | Exceção expirada, revogada ou sem aprovação válida. |
| `AIOS-POL-0014` | 412 | não | Bundle sem assinatura ou com assinatura inválida. |
| `AIOS-POL-0015` | 409 | não | Rollback impossível: versão alvo arquivada ou fora da janela de retenção. |
| `AIOS-POL-0016` | 422 | não | Complexidade do bundle acima do limite (regras, profundidade ou tamanho compilado). |
| `AIOS-POL-0017` | 409 | não | Grafo de herança de papéis cíclico. |
| `AIOS-POL-0018` | 403 | não | Aprovação dupla exigida e ausente (autor = aprovador, ou papel privilegiado). |
| `AIOS-POL-0019` | 422 | não | Obrigação desconhecida ou incompatível com o PEP declarado. |
| `AIOS-POL-0020` | 409 | não | Bundle imutável: edite criando uma nova versão. |

> **Nota deliberada sobre `deny` e erro.** Um `deny` **não é erro**: é uma resposta
> `200` com `effect = deny`. Os códigos acima cobrem falhas de *administração* e de
> *avaliação*. Confundir negação com falha faria todo PEP tratar política restritiva
> como incidente — e a reação a incidente é sempre afrouxar.

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3. `<dominio>` =
> **`policy`** — domínio **já previsto** na RFC-0001 §5.3 e cujo subject
> `aios.<tenant>.policy.decision.denied` é exemplo normativo da própria RFC. Entrega
> at-least-once; dedupe por `event.id`.
>
> **Regra absoluta:** nenhum evento deste módulo transporta valor de atributo marcado
> como `sensitive`, nem o conteúdo do recurso avaliado. Eventos carregam URN, ação,
> `reason_code`, `rule_key` e versão de bundle.

### 6.1 Eventos emitidos (produtor: 022-Policy)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios.<tenant>.policy.bundle.updated` | `aios.policy.bundle.updated` | Bundle → `Active` (T-08/T-09/T-12). **Sinal canônico de invalidação total de cache** nos PEPs. | `POLICY_BUNDLE` |
| `aios.<tenant>.policy.bundle.rolledback` | `aios.policy.bundle.rolledback` | Bundle ativo revertido (T-11). | `POLICY_BUNDLE` |
| `aios.<tenant>.policy.bundle.rejected` | `aios.policy.bundle.rejected` | Candidato reprovado na compilação, conflito ou testes (T-04/T-06). | `POLICY_BUNDLE` |
| `aios.<tenant>.policy.decision.updated` | `aios.policy.decision.updated` | Mudança **granular** que invalida decisões cacheadas: vínculo criado/removido, exceção concedida/revogada, papel alterado. | `POLICY_DECISION` |
| `aios.<tenant>.policy.decision.denied` | `aios.policy.decision.denied` | Decisão `deny` emitida (amostragem configurável para `no_matching_rule`; **sempre** para `explicit_deny` e `platform_deny`). | `POLICY_DECISION` |
| `aios.<tenant>.policy.exception.granted` | `aios.policy.exception.granted` | Waiver concedido, com prazo e aprovador. | `POLICY_GOVERNANCE` |
| `aios.<tenant>.policy.exception.expired` | `aios.policy.exception.expired` | Waiver expirou ou foi revogado. | `POLICY_GOVERNANCE` |
| `aios.<tenant>.policy.simulation.completed` | `aios.policy.simulation.completed` | Simulação concluída com diferencial disponível. | `POLICY_GOVERNANCE` |
| `aios._platform.policy.attribute.degraded` | `aios.policy.attribute.degraded` | Fonte de atributo `required` degradada — sinaliza que decisões podem cair em `deny` por indisponibilidade. | `POLICY_GOVERNANCE` |

> `bundle.updated` e `decision.updated` são **deliberadamente distintos**:
> o primeiro é a bomba de vácuo (invalida tudo do tenant, é raro); o segundo é o
> bisturi (invalida o afetado, é frequente). Ter apenas o primeiro faria toda
> concessão de papel derrubar o cache de todos os PEPs do tenant.

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do módulo |
|-------------------|----------|----------------|
| `aios.<tenant>.security.principal.disabled` | `021-Security` | Invalida vínculos e decisões cacheadas do princípio; emite `policy.decision.updated`. |
| `aios.<tenant>.security.token.revoked` | `021-Security` | Invalida decisões associadas à credencial revogada. |
| `aios.<tenant>.security.authn.anomaly` | `021-Security` | Atualiza o atributo `threat.*` do sujeito (insumo ABAC). |
| `aios.<tenant>.cost.budget.exhausted` | `026-Cost-Optimizer` | Atualiza o atributo `budget.*`; regras dependentes passam a negar sem esperar o TTL do cache. |
| `aios.<tenant>.agent.lifecycle.terminated` | `006-Kernel` | Expira vínculos e exceções do agente extinto. |
| `aios.<tenant>.audit.anomaly.detected` | `025-Audit` | Atualiza atributo de risco; PODE disparar reavaliação de decisões `allow` cacheadas. |
| `aios._platform.deployment.rollout.started` | `028-Deployment` | Suspende publicações automáticas de bundle durante a janela de rollout. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Avaliar `DecisionRequest` e responder `allow`/`deny` sob *default deny*. | Must | Requisição sem regra casada retorna `deny` com `reason_code = no_matching_rule`. |
| FR-002 | Avaliar RBAC e ABAC no mesmo ciclo, com a precedência canônica de §4.6. | Must | Suíte de conformidade cobre os 5 passos de precedência; resultado determinístico. |
| FR-003 | Retornar `reason_code`, `matched_rules[]` e `bundle_version` em **toda** decisão. | Must | 100% das respostas carregam os três campos; `deny` sem motivo é falha de teste. |
| FR-004 | Retornar obrigações que o PEP DEVE cumprir, com limite máximo por decisão. | Must | Obrigação desconhecida pelo PEP ⇒ tratada como `deny` (teste de contrato). |
| FR-005 | Retornar TTL de decisão, nunca superior ao configurado e menor para `deny`. | Must | `deny_ttl_ms` ≤ `default_ttl_ms` verificado em todas as respostas. |
| FR-006 | Compilar o bundle, validar referências e detectar conflitos e regras inalcançáveis. | Must | Contradição sem prioridade → `AIOS-POL-0008` em modo `strict`. |
| FR-007 | Barrar publicação de bundle que não passe nos casos golden obrigatórios. | Must | Falha de caso obrigatório → `AIOS-POL-0007`; bundle permanece fora de `Active`. |
| FR-008 | Simular bundle candidato contra tráfego registrado e produzir o diferencial. | Must | Relatório com `flipped_to_deny`/`flipped_to_allow` por `rule_key`, sem efeito em produção. |
| FR-009 | Publicar bundle com assinatura verificável e aprovação dupla. | Must | `approved_by = authored_by` → `AIOS-POL-0018`; assinatura inválida → `AIOS-POL-0014`. |
| FR-010 | Reverter para a versão anterior dentro da janela de retenção. | Must | Rollback ativa a versão alvo e emite `bundle.updated` + `bundle.rolledback`. |
| FR-011 | Propagar o bundle ativo a todas as réplicas e sinalizar por `bundle.updated`. | Must | Nenhuma réplica serve versão anterior após a janela de propagação (NFR-004). |
| FR-012 | Emitir `policy.decision.updated` em mudança granular (vínculo, exceção, papel). | Must | PEP invalida apenas o afetado; medido por teste de contrato com `006`/`007`. |
| FR-013 | Resolver atributos externos com timeout e criticidade declarada por fonte. | Must | Fonte `required` indisponível ⇒ `deny` com `attribute_unavailable` (`AIOS-POL-0009`). |
| FR-014 | Conceder exceções temporárias com prazo máximo, justificativa e aprovador distinto. | Must | `expires_at` sempre preenchido e ≤ `pol.exception.max_ttl_h`; expiração automática. |
| FR-015 | Registrar 100% dos `deny` e uma amostra dos `allow` no journal de decisões. | Must | Contagem de `deny` no journal = contagem na métrica de decisões negadas. |
| FR-016 | Explicar qualquer decisão registrada, reproduzindo-a sobre a versão exata do bundle. | Must | `ExplainDecision` devolve o mesmo `effect` e as mesmas regras casadas. |
| FR-017 | Responder decisões também por NATS request-reply em `policy.decision.request`. | Must | Contrato exercitado com `020-Communication` (teste Pact). |
| FR-018 | Autorizar as próprias operações administrativas pelo bundle-raiz de autogoverno. | Must | Operação sem capability → `AIOS-POL-0002`; bundle-raiz não editável pelo fluxo comum. |
| FR-019 | Negar tudo quando não houver bundle ativo para o tenant. | Must | Tenant sem `Active` → `deny` com `no_active_bundle` (invariante I5). |
| FR-020 | Expor as permissões efetivas de um sujeito (RBAC resolvido + vínculos vigentes). | Should | Resultado coerente com decisões reais em amostragem cruzada. |
| FR-021 | Emitir todos os eventos de §6 via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. |
| FR-022 | Rejeitar bundle acima dos limites de complexidade configurados. | Should | Excesso de regras/profundidade → `AIOS-POL-0016` na compilação, não em runtime. |
| FR-023 | Não expor valor de atributo `sensitive` em resposta, log, métrica, evento ou journal. | Must | Varredura automatizada não encontra valor sensível; apenas `sha256`. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (decisão) | p99 de `Evaluate` ≤ **5 ms** com atributos em cache; ≤ **15 ms** com resolução remota de atributo. | Histograma `aios_pol_decision_latency_ms{path}`. |
| NFR-002 | Throughput | ≥ **50.000 decisões/s** por réplica com bundle compilado em memória. | Benchmark `Benchmark.md`. |
| NFR-003 | Disponibilidade | ≥ **99,99%** — o PDP está no caminho de toda ação privilegiada do AIOS. | Uptime probe; *error budget* mensal. |
| NFR-004 | Propagação de política | Bundle ativo em **100%** das réplicas em ≤ **10 s**; invalidação de cache nos PEPs em ≤ **5 s** (p99). | Tempo entre `bundle.updated` e primeira decisão com a nova versão. |
| NFR-005 | Escalabilidade | ≥ **10⁶** agentes, ≥ **10⁴** regras por bundle e ≥ **10⁵** vínculos por tenant com custo de decisão sub-linear. | Teste de escala `Scalability.md`. |
| NFR-006 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min**; bundle ativo reconstruível a partir do registro assinado. | DR drill. |
| NFR-007 | Explicabilidade | **100%** das decisões `deny` com `reason_code` e regras casadas. | Auditoria do journal; contagem de `deny` sem motivo = 0. |
| NFR-008 | *Default deny* verificável | **0** decisões `allow` sem regra explícita casada. | Varredura contínua do journal (`matched_rules` vazio ∧ `effect = allow` ⇒ defeito). |
| NFR-009 | Determinismo | Mesma requisição + mesma `bundle_version` + mesmos atributos ⇒ **mesma** decisão, em **100%** das reproduções. | `ExplainDecision` comparado ao journal em amostragem contínua. |
| NFR-010 | Orçamento de avaliação | **100%** das avaliações concluem em ≤ `pol.decision.eval_budget_ms` (default 50 ms) **ou** retornam `deny`. | `aios_pol_eval_timeout_total`; nunca há avaliação pendente sem resposta. |
| NFR-011 | Qualidade da política | Cobertura de regras exercitadas por teste ≥ **90%** antes de qualquer publicação. | `test_coverage` do bundle no gate de publicação. |
| NFR-012 | Simulação | Reavaliar **10⁶** requisições registradas em ≤ **10 min**, sem impacto no p99 de produção. | Benchmark de simulação com carga concorrente. |
| NFR-013 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key` (RFC-0001 §5.5). |
| NFR-014 | Observabilidade | **100%** das decisões com trace OTel e auditoria correlacionada em `025`, sem valor sensível. | Cobertura de spans; varredura de conteúdo. |
| NFR-015 | Integridade do bundle | **100%** dos bundles ativos com assinatura verificada na carga. | Verificação na ativação e no *startup* da réplica. |
| NFR-016 | Isolamento de tenant | **0** decisões que atravessem a fronteira de tenant. | RLS + validação estrutural; `AIOS-POL-0003` monitorado. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global`, `tenant`, `bundle`, `attribute_source`. Recarregável indica *hot
> reload*. Prefixo de env var: **`AIOS_POL_`**.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `pol.decision.default_ttl_ms` | 3000 | 0–60000 | global/tenant | sim | TTL sugerido para decisões `allow` no cache do PEP. |
| `pol.decision.deny_ttl_ms` | 1000 | 0–10000 | global/tenant | sim | TTL para `deny` — **DEVE** ser ≤ o de `allow` (revogação rápida). |
| `pol.decision.eval_budget_ms` | 50 | 5–500 | global | sim | Orçamento de avaliação; estouro ⇒ `deny` (`AIOS-POL-0011`). |
| `pol.decision.batch_max_size` | 64 | 1–512 | global | sim | Máximo de decisões por `EvaluateBatch`. |
| `pol.decision.max_eval_per_s_per_tenant` | 100000 | 100–1000000 | tenant | sim | Limite de avaliações (`AIOS-POL-0010`). |
| `pol.journal.allow_sample_rate` | 0.05 | 0.0–1.0 | tenant | sim | Amostragem de `allow` no journal. `deny` é **sempre** 1,0 e não é configurável. |
| `pol.journal.retention_days` | 30 | 1–365 | tenant | sim | Retenção do journal de decisões (a prova longa é do `025`). |
| `pol.bundle.max_rules` | 10000 | 100–100000 | global | sim | Limite de regras por bundle (`AIOS-POL-0016`). |
| `pol.bundle.max_compiled_mb` | 64 | 1–512 | global | sim | Tamanho máximo do artefato compilado por bundle. |
| `pol.bundle.signature_required` | `true` | {true,false} | global | **não** | Exige assinatura válida para ativar (`AIOS-POL-0014`). |
| `pol.bundle.propagation_target_s` | 10 | 1–120 | global | sim | Meta de propagação do bundle ativo (NFR-004). |
| `pol.bundle.rollback_retention_versions` | 10 | 1–100 | tenant | sim | Versões `Superseded` retidas para rollback (invariante I6). |
| `pol.bundle.dual_approval_required` | `true` | {true,false} | tenant | **não** | Exige `approved_by ≠ authored_by` (`AIOS-POL-0018`). |
| `pol.test.min_coverage` | 0.90 | 0.0–1.0 | tenant | sim | Cobertura mínima de regras para publicar (NFR-011). |
| `pol.test.required_before_publish` | `true` | {true,false} | global | **não** | Torna o gate de teste obrigatório. |
| `pol.simulation.required_before_publish` | `true` | {true,false} | tenant | sim | Exige simulação quando existe bundle ativo (T-08). |
| `pol.simulation.max_requests` | 1000000 | 1000–10000000 | tenant | sim | Teto de requisições reavaliadas por simulação. |
| `pol.conflict.mode` | `strict` | {strict,warn} | tenant | sim | Conflito de regras bloqueia (`strict`) ou apenas anota (`warn`). |
| `pol.attribute.resolve_timeout_ms` | 20 | 1–200 | attribute_source | sim | Timeout default de resolução de atributo. |
| `pol.attribute.cache_ttl_s` | 30 | 0–3600 | attribute_source | sim | TTL default do cache de atributo. |
| `pol.attribute.fail_mode` | `closed` | {closed,open} | global | sim | Comportamento com fonte `required` indisponível. `open` **NÃO DEVERIA** ser usado em produção. |
| `pol.exception.max_ttl_h` | 72 | 1–720 | tenant | sim | Prazo máximo de um waiver. |
| `pol.exception.require_dual_approval` | `true` | {true,false} | tenant | **não** | Exige aprovador distinto do solicitante. |
| `pol.obligation.max_per_decision` | 8 | 1–32 | global | sim | Limite de obrigações por decisão. |
| `pol.selfgov.root_bundle_immutable` | `true` | {true,false} | global | **não** | Bundle-raiz (`scope = platform`) só muda por cerimônia (§12.1). |
| `pol.deny_event.sample_rate` | 0.1 | 0.0–1.0 | tenant | sim | Amostragem de `decision.denied` para `no_matching_rule`; `explicit_deny` e `platform_deny` são sempre emitidos. |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| PDP indisponível (visto pelo PEP) | Timeout/CB no `PolicyClient` do chamador | O PEP aplica seu próprio `fail_mode` (default `closed`, RFC-0001 §5.8) | Réplicas voltam com bundle assinado carregado do registro | Avaliação é leitura pura: retry sempre seguro |
| PostgreSQL (`policy`) indisponível | Health | **Avaliação continua** com o bundle em memória; autoria, publicação e journal param | Failover do `005`; journal represado é drenado | Idempotente |
| Fonte de atributo `required` indisponível | Timeout no `AttributeResolver` | Apenas as regras que dependem daquele atributo | `deny` com `attribute_unavailable`; evento `attribute.degraded`; retomada automática | Sem retry cego dentro do orçamento de avaliação |
| Bundle ativo corrompido ou assinatura inválida | Verificação na carga (NFR-015) | Réplica recusa servir com bundle inválido e sai do balanceamento | Recarrega do registro; se persistir, rollback para a versão anterior íntegra | Carga idempotente |
| Publicação de bundle que quebra produção | Salto em `deny` após `bundle.updated`; alerta de taxa de negação | Impacto confinado ao tenant | **Rollback (T-11)** para a versão anterior; simulação obrigatória evita a recorrência | Rollback idempotente por `Idempotency-Key` |
| Falha de propagação a uma réplica | Réplica reporta `bundle_version` defasada | Réplica retirada do balanceamento antes de decidir com versão velha | Repropagação; alerta se exceder `propagation_target_s` | Dedupe por `event.id` |
| Explosão de complexidade de política | Métrica de latência de avaliação e tamanho compilado | Bundle rejeitado na compilação (`AIOS-POL-0016`), nunca em produção | Refatoração da política; limites reavaliados | — |
| Estouro do orçamento de avaliação | `aios_pol_eval_timeout_total` | Decisão individual, não a réplica | `deny` (`eval_timeout`); investigação da regra custosa | Retry do chamador é seguro |
| Tempestade de decisões (tenant abusivo) | `max_eval_per_s_per_tenant` | Limite **por tenant**, nunca global | `AIOS-POL-0010` com `Retry-After`; cache do PEP absorve o restante | Retriable |
| Waiver esquecido em aberto | Varredura de `expires_at` | — | Expiração automática + `exception.expired`; alerta em concessões renovadas repetidamente | Determinístico |
| Perda do journal de decisões | Gap de sequência | Explicabilidade recente degradada | Reconstrução parcial pela trilha do `025`; simulação usa janela menor | — |
| Divergência entre réplicas (versões diferentes) | Métrica `aios_pol_active_bundle_version` por réplica | Réplica divergente retirada | Repropagação forçada; alarme P1 — duas políticas simultâneas violam I1 | — |

**Metas de recuperação:** **RTO ≤ 15 min**, **RPO ≤ 5 min**. Degradação graciosa em
ordem: sacrifica-se primeiro **simulação e testes**, depois **autoria e publicação**,
depois o **journal**, preservando por último a **avaliação de decisão** — porque um
AIOS que não publica política nova opera com a política de ontem; um AIOS que não
decide **não executa nada**, já que toda ação privilegiada depende de uma decisão.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Serviço de decisão | **Stateless** e replicado; o bundle compilado vive em memória de cada réplica. Nenhuma decisão consulta o banco no caminho quente — essa é a decisão que impede o PDP de virar o gargalo global. |
| Cache no PEP | O TTL emitido por decisão (§5.1) faz o chamador absorver a maior parte do tráfego repetitivo; o custo é a revogação não instantânea, mitigado por TTL curto + `decision.updated` granular. |
| Indexação de regras | O `BundleCompiler` indexa regras por `(dominio, objeto, verbo)` e por padrão de recurso: a avaliação percorre o subconjunto casável, não as 10⁴ regras. |
| Avaliação sem I/O bloqueante | RBAC e ABAC operam sobre estruturas em memória; a única I/O possível é a resolução de atributo externo, com timeout rígido e cache. |
| Cache de atributos | Redis + cache local por réplica, com TTL por fonte; atributo `sensitive` é cacheado por digest, nunca por valor em claro. |
| Particionamento do journal | `policy.decision_journal` particionada por `RANGE(decided_at)`: expurgo e simulação por janela tornam-se operações de partição. |
| Concorrência de publicação | OCC por `version` + índice único parcial sobre `Active` (§3.1): duas publicações concorrentes não podem produzir dois bundles ativos — a segunda falha com `AIOS-POL-0006`. |
| Simulação isolada | O `SimulationEngine` roda em réplicas dedicadas (pool separado) e lê partições do journal: carga de simulação **não** compete com o caminho quente (NFR-012). |
| Backpressure | Limite por tenant (`max_eval_per_s_per_tenant`) com `Retry-After`; rejeição precoce no `DecisionGateway` antes de qualquer resolução de atributo. |
| Rumo a milhões | 10⁶ agentes ⇒ da ordem de 10⁵ decisões/s no pico agregado. Sustentado por: cache no PEP (a maioria das decisões nunca chega ao 022), avaliação em memória sem I/O, réplicas horizontais sem estado compartilhado e propagação de bundle por evento em vez de *polling*. |

```
   ❌ política consultada por linha de código      ✅ bundle compilado em memória
      cada regra = uma query ao banco                 índice por ação/recurso
      (PDP vira gargalo e SPOF do AIOS)               PEP cacheia por TTL curto
```

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada: `ADR-0220`..`ADR-0229`** (regra 022×10). Registrar em
> `../002-ADR/`. **ADR-0008** (governança por política, *default deny*) é herdada e é
> a decisão-mãe deste módulo; **ADR-0010** (auditoria por construção) define a
> fronteira com o `025-Audit`.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0220 | Domínio de erro `POL` e registro do domínio de eventos `policy` (subjects, streams). |
| ADR-0221 | Política como artefato de release: bundle compilado, versionado, assinado e reversível. |
| ADR-0222 | Avaliação central com bundle em memória vs. avaliação embarcada no PEP. |
| ADR-0223 | Precedência canônica de combinação (deny-overrides com prioridade explícita). |
| ADR-0224 | Colapso de `indeterminate`/`not_applicable` em `deny` na fronteira do PDP. |
| ADR-0225 | Obrigações na resposta de decisão e regra do "PEP que não cumpre, nega". |
| ADR-0226 | Gate obrigatório de testes golden e simulação antes da publicação. |
| ADR-0227 | Exceções (waivers) com prazo máximo e aprovação dupla como alternativa ao afrouxamento permanente. |
| ADR-0228 | Autogoverno do PDP: bundle-raiz de escopo `platform`, separação de funções e cerimônia de alteração. |
| ADR-0229 | Journal de decisões de retenção curta no `022` vs. trilha imutável no `025`. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida) — §5.8 fixa PEP/PDP e *default deny*. |
| RFC-0220 | AIOS Policy Decision Contract (`DecisionRequest`/`DecisionResponse`, `reason_code`, obrigações, TTL). | A propor por este módulo. |
| RFC-0221 | AIOS Policy Bundle Format & Distribution (esquema de regra/papel, compilação, assinatura, propagação). | A propor por este módulo. |

---

## 12. Decisões de Segurança

> Este módulo **decide** a segurança de todo o resto. Sua superfície de ataque não é
> o dado que guarda — é a **decisão que emite**: comprometer o 022 não vaza segredo,
> concede acesso.

### 12.1 AuthN / AuthZ e o problema do autogoverno

- **AuthN**: operadores autenticam-se por OIDC no `021-Security` com MFA; serviços
  chamadores autenticam-se por **mTLS** com certificado de workload (RFC-0001 §6). O
  022 **não** autentica ninguém por conta própria.
- **AuthZ das operações de decisão**: `Evaluate` exige a capability
  `pol:decision:evaluate` do chamador — um PEP só pode perguntar em nome do **seu**
  tenant; `tenant` divergente → `AIOS-POL-0003`.
- **AuthZ das operações administrativas — o autogoverno**: o PDP autoriza a si mesmo,
  o que só é seguro se a política que o governa **não puder ser alterada pelo mesmo
  caminho que ela governa**. Por isso:
  - existe um **bundle-raiz** de `scope = platform`, imutável pelo fluxo comum
    (`pol.selfgov.root_bundle_immutable`), que define quem pode autorar, testar,
    simular, publicar, reverter e conceder exceção;
  - suas regras `deny` têm **precedência absoluta** sobre qualquer regra de tenant
    (§4.6, passo 2) — nenhum tenant pode se conceder o que a plataforma nega;
  - alterá-lo exige **cerimônia** com aprovação dupla e registro em `025`, fora do
    caminho de API comum.
- **Separação de funções**: quem **autora** a política **NÃO DEVE** ser quem a
  **publica** (`approved_by ≠ authored_by`, `AIOS-POL-0018`); quem solicita um waiver
  não pode aprová-lo. Vincular papel marcado `is_privileged` exige aprovação dupla.
- **Fail-closed**: com fonte de atributo `required` indisponível, orçamento estourado
  ou bundle ausente, a resposta é `deny`. `pol.attribute.fail_mode = open` existe
  apenas para ambientes de teste e **NÃO DEVERIA** ser habilitado em produção sem
  exceção documentada em ADR.
- **Isolamento de tenant**: RLS por `tenant_id` em bundles, regras, papéis, vínculos,
  exceções, testes, simulações e journal; um bundle de tenant **nunca** avalia
  requisição de outro.

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no 022-Policy | Mitigação |
|-----------------|----------------------|-----------|
| **S**poofing | Um serviço se passa por PEP de outro módulo/tenant e pergunta em nome de terceiros, ou responde a `policy.decision.request` fingindo ser o PDP. | mTLS com certificado de workload atestado pelo `021`; validação de `tenant` do chamador (`AIOS-POL-0003`); no barramento, o subject de decisão é restrito por conta NATS (`020`), de modo que só o PDP pode responder. |
| **T**ampering | Alteração de regra em bundle ativo; injeção de vínculo de papel privilegiado; adulteração do artefato compilado em trânsito ou em disco. | Bundle **imutável** a partir de `Validated` (I2, `AIOS-POL-0020`); artefato **assinado** e verificado na carga (NFR-015, `AIOS-POL-0014`); toda mutação sob OCC, com autor, aprovador e trilha em `025`. |
| **R**epudiation | Negar ter autorado, aprovado, publicado, revertido ou concedido exceção. | `authored_by`/`approved_by`/`granted_by`/`requested_by` obrigatórios; journal com `decision_id` e `bundle_version`; trilha imutável em `../025-Audit/`. |
| **I**nformation disclosure | Vazamento de valores de atributo sensível em log, métrica, evento ou explicação; uso do `ExplainService` para enumerar a política de outro tenant; inferência da política por sondagem de decisões. | Atributos `sensitive` gravados só como `sha256` (FR-023); `Explain` sob capability e restrito ao tenant; limite de avaliações por tenant limita sondagem; métricas sem `subject_urn` como label. |
| **D**enial of service | Regra patológica que explode o tempo de avaliação; tempestade de decisões de um tenant; publicação que faz todo `allow` virar `deny`. | Limites de complexidade na **compilação** (`AIOS-POL-0016`), não em runtime; orçamento de avaliação por decisão (`AIOS-POL-0011`); limite por tenant (`AIOS-POL-0010`); gate de simulação expõe o salto de `deny` **antes** da publicação. |
| **E**levation of privilege | Autoconcessão de papel privilegiado; waiver eterno; edição do bundle-raiz pelo caminho comum; PEP ignorando obrigação para obter acesso mais amplo. | Aprovação dupla (`AIOS-POL-0018`); `expires_at` obrigatório em waiver (§3.6); bundle-raiz imutável com precedência absoluta (§12.1); obrigação desconhecida ⇒ `deny` no PEP (FR-004), verificado por teste de contrato. |

### 12.3 LGPD / GDPR

- **Minimização**: o módulo avalia **atributos**, não conteúdo. O journal guarda
  URNs, ação, motivo e o **digest** dos atributos — nunca o valor de um atributo
  `sensitive` nem o conteúdo do recurso avaliado.
- **Base legal e retenção**: o journal é dado operacional de segurança, com retenção
  curta (`pol.journal.retention_days`, default 30 dias); a retenção longa exigida por
  obrigação legal é responsabilidade do `../025-Audit/`, que declara a própria base.
- **Direito ao esquecimento**: ao consumir `security.rtbf.requested` (via
  `principal.disabled`), o módulo expira vínculos e exceções do titular e pseudonimiza
  o `subject_urn` no journal, preservando as contagens agregadas necessárias à
  auditoria de segurança — o registro de que *uma* decisão ocorreu não pode
  desaparecer, mas *quem* pode ser dissociado.
- **Decisão automatizada**: quando uma política nega acesso a um titular com base em
  atributos pessoais, a explicabilidade obrigatória (FR-003/FR-016) é o que permite
  atender ao direito de revisão — um `deny` sem motivo seria, além de um defeito
  operacional, um problema de conformidade.
- **Segregação**: RLS por tenant em todas as tabelas; políticas de um tenant são
  invisíveis a outro, inclusive por `Explain` e por simulação.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md` (*PEP/PDP*, *Policy Engine*, *Capability*,
  *Default Deny* — reutilizados, não redefinidos)
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§5.3 domínio
  `policy`, §5.4 envelope de erro, §5.5 idempotência, §5.6 correlação, §5.8 PEP/PDP)
- Decisões: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`,
  `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../004-API/`, `../005-Database/`, `../006-Kernel/`,
  `../007-Agent-Runtime/`, `../009-Scheduler/`, `../010-Memory/`, `../011-Context/`,
  `../015-Tool-Manager/`, `../017-Model-Router/`, `../020-Communication/`,
  `../021-Security/`, `../024-Observability/`, `../025-Audit/`,
  `../026-Cost-Optimizer/`, `../028-Deployment/`, `../029-Operations/`.

*Fim do Design Brief interno do módulo 022-Policy. Este documento governa a geração
dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e este
brief é um defeito do documento gerado.*
