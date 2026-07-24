---
Documento: Architecture
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0221, ADR-0222, ADR-0223
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: 001-Architecture, 005-Database, 020-Communication, 021-Security, 024-Observability, 025-Audit, 026-Cost-Optimizer
---

# 022-Policy — Architecture

## 1. Visão C4 — Nível 1 (Contexto)

```
   ┌───────────────────────────────────────────────────────────────────────────┐
   │                     PEPs — todo módulo com ação privilegiada               │
   │  004 RouteAuthorizer · 006 CapabilityEnforcer · 007 PEP local ·           │
   │  009 AdmissionController · 010 MemoryPep · 015/017 brokers ·              │
   │  020 SubscriptionBroker/A2AGateway · 021 operações administrativas        │
   └──────────────┬────────────────────────────────────────────────────────────┘
                  │ DecisionRequest (gRPC · NATS request-reply · REST)
                  ▼
   ┌───────────────────────────────────────────────────────────────────────────┐
   │                      022-POLICY — PDP (este módulo)                        │
   │  avalia RBAC+ABAC · responde allow|deny + motivo + obrigações + TTL       │
   │  governa o ciclo de vida do policy bundle (autoria→…→rollback)            │
   └───┬────────────┬─────────────┬──────────────┬───────────────┬─────────────┘
       │            │             │              │               │
       ▼            ▼             ▼              ▼               ▼
   021-Security  026-Cost     024-Observ.    025-Audit      005-Database
   (princípios,  (orçamento   (sinais de     (trilha        (schema policy)
    atributos,    consumido)   risco)         imutável)
    assinatura)
                        ▲
                        │ eventos aios.<tenant>.policy.*
                  020-Communication (NATS/JetStream)
```

## 2. Visão C4 — Nível 2 (Contêiner)

| Contêiner | Tecnologia | Papel | Justificativa |
|-----------|-----------|-------|---------------|
| `policy-api` | .NET 10 (ASP.NET Core + gRPC) | Superfície de decisão e de administração; hospeda o `BundleRuntime`. | Control plane em .NET (ADR-0003); o caminho quente exige runtime compilado com baixa latência de GC sob carga. |
| `policy-simulator` | .NET 10 (worker pool dedicado) | Executa simulação e testes golden fora do caminho quente. | Isolar carga analítica do p99 de produção (NFR-012). |
| `policy-outbox-relay` | .NET 10 (worker) | Publica o outbox transacional em JetStream. | Garante at-least-once sem 2PC (padrão já adotado por `../021-Security/`). |
| PostgreSQL (schema `policy`) | PostgreSQL 16 | Registro de bundles, regras, papéis, vínculos, exceções, journal. | Unificação relacional (ADR-0005); RLS por `tenant_id`. |
| Redis | Redis 7 | Cache de atributos externos e contadores de limite por tenant. | Estado quente (ADR-0006). |
| NATS / JetStream | NATS 2.x | Eventos `aios.<tenant>.policy.*` e request-reply de decisão. | Barramento primário (ADR-0004). |

```
   ┌────────────────────────────────────────────────────────────────────────┐
   │  policy-api  (N réplicas, stateless, bundle compilado em memória)      │
   │   ├─ gRPC  aios.policy.v1        (caminho quente, PEPs internos)       │
   │   ├─ REST  /v1/policy            (autoria, operação, ferramentas)      │
   │   └─ NATS  aios.<t>.policy.decision.request  (request-reply)           │
   └───────┬──────────────────┬───────────────────────┬────────────────────┘
           │                  │                       │
   ┌───────▼───────┐  ┌───────▼────────┐     ┌────────▼─────────┐
   │ PostgreSQL    │  │ Redis          │     │ NATS / JetStream │
   │ schema policy │  │ cache atributo │     │ POLICY_* streams │
   └───────────────┘  └────────────────┘     └──────────────────┘
           ▲                                          ▲
   ┌───────┴────────────┐                    ┌────────┴─────────────┐
   │ policy-simulator   │                    │ policy-outbox-relay  │
   │ (pool dedicado)    │                    │ (outbox → JetStream) │
   └────────────────────┘                    └──────────────────────┘
```

## 3. Visão C4 — Nível 3 (Componente)

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
   │  │  (ciclo puro)      │    └──────────────────┘    │  (timeout, cache,    │  │  026 (orçamento)
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
   │      │  │  + Conflict    │  │ (versões, FSM,│     │ (propaga + evento)     │ │
   │      │  │    Analyzer    │  │  assinatura)  │     └────────────────────────┘ │
   │      │  └────────────────┘  └───────┬───────┘                                │
   │      │          ▲                   │                                        │
   │      │  ┌───────┴─────────┐ ┌───────▼────────┐  ┌────────────────────────┐  │
   │      │  │ PolicyTestRunner│ │SimulationEngine│  │   ExceptionManager     │  │
   │      │  └─────────────────┘ └───────┬────────┘  └────────────────────────┘  │
   │      ▼                              │                                        │
   │  ┌────────────────────┐  ┌──────────▼───────┐  ┌────────────────────────┐   │
   │  │  DecisionJournal   │◀─│  ExplainService  │  │ SelfGovernanceGuard    │   │
   │  └────────────────────┘  └──────────────────┘  └────────────────────────┘   │
   │  ┌─────────────────────────────────────────────────────────────────────────┐│
   │  │ EventEmitter (Outbox → JetStream)  ·  PolicyTelemetry (OTel + auditoria) ││
   │  └─────────────────────────────────────────────────────────────────────────┘│
   └──────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Responsabilidade por componente

| Componente | Responsabilidade | Depende de |
|------------|------------------|-----------|
| **DecisionGateway** | *Front door* nas três superfícies; valida schema, extrai correlação (RFC-0001 §5.6), rejeita `tenant` divergente, aplica limite por tenant. | DecisionEngine, PolicyTelemetry |
| **DecisionEngine** | Orquestra o ciclo: normaliza ação → resolve atributos → RBAC → ABAC → combina (§4) → obrigações → TTL. **Puro** dado (bundle, atributos). | ActionNormalizer, RbacResolver, AbacEvaluator, AttributeResolver, ObligationComposer, BundleRuntime |
| **ActionNormalizer** | Normaliza a ação para `<dominio>:<objeto>:<verbo>`, aceitando as formas curtas legadas dos PEPs. | BundleRuntime |
| **RbacResolver** | Papéis efetivos do sujeito (vínculos, herança, grupos) e permissões concedidas. | BundleRuntime |
| **AbacEvaluator** | Condições sobre atributos, em expressões **compiladas** (sem interpretação de texto no caminho quente). | BundleRuntime, AttributeResolver |
| **AttributeResolver** | Atributos externos com cache curto, timeout rígido e criticidade; `required` não resolvido ⇒ `deny`. | Redis; → 021, 024, 026 |
| **ObligationComposer** | Compõe, deduplica e limita obrigações; conflito resolve pela mais restritiva. | BundleRuntime |
| **BundleRuntime** | Bundle compilado ativo em memória, com troca atômica de versão. | BundleRegistry, BundleDistributor |
| **BundleCompiler** | Compila, valida referências, indexa regras, mede complexidade. | ConflictAnalyzer |
| **ConflictAnalyzer** | Contradição, sombreamento e regras inalcançáveis. | — |
| **BundleRegistry** | Registro versionado, estado da FSM, artefato e assinatura. | → PostgreSQL, 021 |
| **BundleDistributor** | Ativa, propaga às réplicas e emite `policy.bundle.updated`. | BundleRegistry, EventEmitter |
| **PolicyTestRunner** | Casos golden + cobertura; barra publicação. | BundleCompiler, BundleRuntime |
| **SimulationEngine** | Reavalia tráfego do journal e produz o diferencial. | DecisionEngine (sombra), DecisionJournal |
| **ExceptionManager** | Waivers com prazo, aprovador e expiração automática. | BundleRuntime, EventEmitter |
| **DecisionJournal** | 100% dos `deny` e amostra dos `allow`; retenção curta. | → PostgreSQL, 025 |
| **ExplainService** | Reproduz a decisão sobre a versão exata do bundle. | BundleRegistry, DecisionJournal, DecisionEngine |
| **SelfGovernanceGuard** | PEP interno contra o bundle-raiz (`scope = platform`). | BundleRuntime, BundleRegistry |
| **EventEmitter** | Outbox transacional → JetStream. | → PostgreSQL, NATS |
| **PolicyTelemetry** | Spans/métricas `aios_pol_*`, logs OTel, auditoria para `025`. | → 024, 025 |

## 4. Fronteiras arquiteturais

| Fronteira | Contrato | Documento |
|-----------|----------|-----------|
| PEP → PDP | `DecisionRequest`/`DecisionResponse` (RFC-0220) | `./API.md` |
| PDP → PEPs (invalidação) | `policy.bundle.updated`, `policy.decision.updated` | `./Events.md` |
| PDP → `021-Security` | atributos assertáveis do princípio; assinatura do bundle | `../021-Security/API.md` |
| PDP → `026-Cost-Optimizer` | atributo `budget.*` | `../026-Cost-Optimizer/` |
| PDP → `025-Audit` | fato de decisão (trilha imutável) | `../025-Audit/` |
| PDP ↔ `020-Communication` | request-reply em `aios.<tenant>.policy.decision.request` | `../020-Communication/API.md` |

> A fronteira mais importante é negativa: **o 022 não aplica nada**. Ele responde e
> registra. Quem impede é o PEP do chamador — e é por isso que a decisão pode ser
> auditada independentemente de quem a aplicou (`./_DESIGN_BRIEF.md` §0).

## 5. Padrões arquiteturais adotados

| Padrão | Onde | Por quê | ADR |
|--------|------|---------|-----|
| **Reference monitor** (PEP/PDP) | Todo o módulo | Decisão única, completa e à prova de bypass. | ADR-0008 |
| **Policy as data / policy as release** | `BundleRegistry`, FSM (§`./StateMachine.md`) | Política evolui sem redeploy, com histórico e rollback. | ADR-0221 |
| **Avaliação central com bundle em memória** | `BundleRuntime` | Latência previsível sem I/O no caminho quente; alternativa (avaliação embarcada no PEP) descartada por dificultar revogação e auditoria. | ADR-0222 |
| **Deny-overrides com prioridade explícita** | `DecisionEngine` | Combinação determinística e auditável. | ADR-0223 |
| **Transactional Outbox** | `EventEmitter` | Evento e mudança de estado na mesma transação (invariante I4). | ADR-0004 |
| **Optimistic Concurrency Control** | Todas as mutações | Sem locks longos; conflito explícito (`AIOS-POL-0006`). | ADR-0005 |
| **Circuit breaker + timeout rígido** | `AttributeResolver` | Fonte lenta não pode consumir o orçamento de avaliação. | ADR-0008 |
| **Shadow evaluation (dry-run)** | `SimulationEngine` | Medir o impacto antes de publicar. | ADR-0226 |
| **Sidecar-free** | — | O PDP é um serviço, não uma biblioteca embarcada; ver alternativas (§6). | ADR-0222 |

## 6. Alternativas descartadas

| Alternativa | Por que foi descartada | ADR |
|-------------|------------------------|-----|
| **Autorização ad hoc no código de cada serviço** | Inauditável, inconsistente, impossível de evoluir sem redeploy. | ADR-0008 |
| **RBAC puro centralizado** | Não expressa contexto (orçamento consumido, sensibilidade do dado, horário) — insuficiente para agentes autônomos. | ADR-0008 |
| **Avaliação embarcada no PEP (bundle distribuído a cada serviço)** | Ganha latência, mas fragmenta a revogação (cada réplica com sua cópia), dificulta a explicabilidade central e multiplica a superfície de manipulação do bundle. Mantido como opção futura sob RFC-0221. | ADR-0222 |
| **Expor `indeterminate` ao PEP** | Cada módulo inventaria seu tratamento e algum trataria "não sei" como "pode". | ADR-0224 |
| **Persistir decisões como autorizações duráveis** | Criaria uma segunda fonte da verdade de permissão, divergente do bundle. | ADR-0221 |
| **Retenção longa do journal no próprio 022** | Duplicaria a trilha do `../025-Audit/` com regras de expurgo divergentes. | ADR-0229 |
| **Política editável em vigor** | Destrói a reprodutibilidade da decisão (invariante I2). | ADR-0221 |

## 7. Decisões estruturais e seus riscos

| Decisão | Risco assumido | Mitigação |
|---------|----------------|-----------|
| PDP central no caminho de toda ação privilegiada | Gargalo e SPOF | Stateless replicado, bundle em memória, cache no PEP com TTL, limite por tenant (`./Scalability.md`). |
| Bundle inteiro em memória por réplica | Consumo de RAM cresce com o nº de tenants | `pol.bundle.max_compiled_mb` (64 MiB) e limite de regras; carga sob demanda por tenant ativo. |
| Cache de decisão no PEP | Revogação não instantânea | TTL curto (3 s allow / 1 s deny) + `policy.decision.updated` granular (NFR-004). |
| Autogoverno (o PDP autoriza a si mesmo) | Escalada de privilégio | Bundle-raiz imutável com precedência absoluta, separação de funções, aprovação dupla (`./Security.md` §2). |

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` (§2 componentes, §10 escalabilidade)
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`, `./ADR.md`
- Detalhamento: `./ClassDiagrams.md`, `./SequenceDiagrams.md`, `./Database.md`,
  `./API.md`, `./Events.md`, `./Scalability.md`

*Fim de `Architecture.md`.*
