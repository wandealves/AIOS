---
Documento: Vision
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 000-Vision, 001-Architecture, 005-Database, 011-Context, 017-Model-Router, 018-Knowledge, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Vision

> Documento derivado da FONTE ÚNICA DE VERDADE `./_DESIGN_BRIEF.md` (§1). Termos
> canônicos vêm de [Glossário](../040-Glossary/Glossary.md); contratos centrais de
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md). Este documento **NÃO**
> redefine nenhum dos dois — apenas os referencia e os especializa para o domínio
> de memória. Palavras normativas conforme RFC 2119/8174.

---

## 1. Propósito do Módulo

O `MemoryService` (módulo 010) é o **gerenciador de memória cognitiva
hierárquica** do AIOS — análogo à MMU + swap + filesystem de um sistema
operacional clássico, porém para **recursos cognitivos** em vez de páginas de
RAM (ver [Visão Global](../000-Vision/Vision.md) §7.1, tabela de subsistemas:
"Memory Manager — MMU + swap + FS — Working/Short/Long/Semantic/Procedural/
Episodic — módulo 010").

Sua razão de existir é dar a todo agente — e a todos os módulos que raciocinam
sobre o passado de um agente ([011-Context](../011-Context/), [023-Learning
](../023-Learning/), [018/019-Knowledge](../018-Knowledge/)) — uma **API
uniforme** sobre uma realidade fisicamente heterogênea: sete camadas de
memória, cada uma com backend, política de retenção e característica de
acesso próprios. O módulo DEVE oferecer essa superfície
(`remember`/`recall`/`consolidate`/`forget`) escondendo dos agentes onde e como
cada memória é persistida, recuperada, consolidada e esquecida.

O módulo materializa, para memória, a promessa global do AIOS de ser o
**gerenciador de recursos cognitivos** de agentes de IA. Assim como um SO
abstrai discos, páginas e caches por trás de `read`/`write`/`mmap`, o
`MemoryService` abstrai Redis, PostgreSQL (`pgvector` + Apache AGE) e MinIO por
trás de quatro verbos cognitivos estáveis (brief §1.1, R1–R11):

1. **API uniforme de memória (R1)** — `remember(item, layer)`,
   `recall(query, filtros)`, `consolidate()` e `forget(policy)`, estáveis e
   versionados sobre REST/gRPC (`aios.memory.v1`).
2. **Hierarquia de sete camadas (R2)** — Working, Short-Term, Long-Term,
   Semantic, Procedural, Episodic e Knowledge Graph, cada uma mapeada a um
   backend físico segundo a tabela canônica do brief §3.3.
3. **Recuperação híbrida rankeada (R3)** — busca lexical + vetorial ANN
   (`pgvector`/HNSW) + travessia de grafo (Apache AGE), fundidas por RRF e
   re-rankeadas por relevância/recência/saliência/escopo, em colaboração com
   `011-Context`.
4. **Consolidação e esquecimento controlados (R4, R5)** — promoção versionada
   entre camadas (com `rollback` como defesa contra *catastrophic forgetting*)
   e políticas declarativas de retenção/decaimento/expurgo que contêm o
   crescimento de memória por tenant/agente sob cota (R6).

---

## 2. Problema que o Módulo Resolve

Agentes de IA precisam lembrar seletivamente, esquecer com controle e
consolidar aprendizado sem regredir — em escala multi-tenant, com privacidade e
cotas. Sem um módulo dedicado, cada agente reinventaria persistência, correria
risco de *catastrophic forgetting*, vazaria dados entre tenants e cresceria de
forma descontrolada.

### 2.1 Problemas centrais e como o módulo os resolve

| Problema | Consequência sem o módulo | Como 010-Memory resolve |
|----------|---------------------------|--------------------------|
| Heterogeneidade de armazenamento | Cada agente acopla-se diretamente a Redis/PG/MinIO | API uniforme (R1) sobre 7 camadas (R2), roteada pelo `LayerRouter` |
| Recuperação irrelevante ou lenta | Contexto ruim, alucinação, retrabalho | Recall híbrido rankeado (R3); Memory Recall Rate ≥ 0,95 (NFR-002) |
| *Catastrophic forgetting* | Aprendizado destrói memória prévia correta | Consolidação versionada e reversível (R4); rollback automático (FR-007, NFR-011) |
| Crescimento descontrolado de armazenamento | Custo e latência explodem sem limite | Esquecimento controlado (R5) + cotas por camada/tenant/agente (R6) |
| Vazamento cross-tenant / PII | Violação LGPD/GDPR, incidente de segurança | RLS + `tenant_id` + direito ao esquecimento rastreável (R10) |
| Chamada direta a LLM ou a datastore | Quebra de camadas arquiteturais e de segurança | Embeddings só via `017-Model-Router` (R7); Agent Runtime nunca acessa datastore direto |

### 2.2 Relação com as limitações estruturais da Visão Global

A [Visão Global](../000-Vision/Vision.md) §2 identifica oito limitações
estruturais dos agentes de IA atuais (P1–P8). O `010-Memory` é a resposta
direta a duas delas, e o alicerce sobre o qual uma terceira é construída por
outro módulo:

| Limitação da Visão | Como o Memory endereça |
|---------------------|--------------------------|
| **P1 — Memória de curto prazo limitada à janela de contexto** | Working/Short-Term cobrem o horizonte imediato; Long-Term/Semantic/Episodic/Knowledge Graph preservam estado **além** da janela de contexto de qualquer chamada de inferência, recuperável sob demanda via `recall()`. O `011-Context` consome essa capacidade para montar a janela final. |
| **P6 — Aprendizado inexistente ou ingênuo** | O Memory não decide *quando/como* aprender (N2), mas é o substrato versionado sobre o qual `023-Learning` materializa consolidação auditável e reversível. |
| **P7 — Baixa observabilidade** | Toda operação de memória emite trace OTel e contribui para a **Memory Recall Rate**, tornando mensurável se o agente recupera o que precisa quando precisa. |

Complementarmente, o Memory é nomeado explicitamente na mitigação de dois
riscos estratégicos globais (Visão Global §10): **RSK-01** (*catastrophic
forgetting*, mitigado por consolidação versionada + rollback) e **RSK-02**
(crescimento descontrolado de memória, mitigado por retenção + cotas,
co-mitigado com `026-Cost-Optimizer`).

### 2.3 Por que uma hierarquia de 7 camadas e não um único armazenamento vetorial

Uma alternativa mais simples — tratar toda memória como um único índice
vetorial genérico (o padrão "RAG simples") — foi descartada porque:

- **Confundiria escopos temporais e semânticos incompatíveis.** Working
  (milissegundos a minutos, RAM) e Semantic (sem TTL, decaimento lento) têm
  requisitos de latência/volatilidade opostos; um backend único penaliza um
  dos dois extremos.
- **Impediria políticas de retenção diferenciadas por camada**, exigidas por
  LGPD/GDPR e pela meta de crescimento controlado (RSK-02) — `memory.layer.
  *.ttl`, `memory.decay.half_life` são configuráveis por camada.
- **Ignoraria a diferença entre "como fazer" (Procedural) e "o que aconteceu"
  (Episodic)** — estruturas e semânticas de recuperação distintas de
  Semantic (fatos gerais).
- **Não acomodaria GraphRAG** — relações entre itens de memória do agente
  (causalidade, hierarquia, correferência) exigem travessia de grafo
  (`KnowledgeGraphAdapter`, Apache AGE), inviável eficientemente como busca
  vetorial pura.

A hierarquia de sete camadas (ADR-0100) traduz, no domínio cognitivo, o que um
SO clássico já resolveu para memória física (registradores → cache → RAM →
swap → disco): níveis com compromissos distintos, unificados por uma API que
**oculta**, sem **eliminar**, a heterogeneidade.

---

## 3. Escopo

### 3.1 Dentro de escopo (in)

O módulo DEVE prover, conforme brief §1.1:

- **API uniforme de memória** (`remember`/`recall`/`consolidate`/`forget` +
  `getItem`/`getStats`) sobre REST/gRPC versionada (R1).
- **Hierarquia de 7 camadas** — Working, Short-Term, Long-Term, Semantic,
  Procedural, Episodic, Knowledge Graph (R2).
- **Recuperação híbrida rankeada** — lexical + vetorial (ANN `pgvector`/HNSW)
  + travessia de grafo (Apache AGE), com fusão RRF e re-rank (R3).
- **Consolidação dirigida** por `023-Learning`, versionada e reversível (R4).
- **Esquecimento controlado** por TTL, decaimento, LRU, cota e RTBF (R5).
- **Cotas e contabilização** por camada/tenant/agente, com backpressure (R6).
- **Embeddings via `017-Model-Router`** — nunca chamada direta a LLM (R7).
- **Externalização de blobs grandes** para MinIO, content-addressed (R8).
- **Eventos de domínio** `memory.*` via padrão Outbox (R9).
- **Isolamento multi-tenant e privacidade** — RLS, base legal, direito ao
  esquecimento (R10).
- **Observabilidade** — métrica-chave **Memory Recall Rate** e telemetria OTel
  correlacionada (R11).

### 3.2 Fora de escopo (out / não-objetivos explícitos)

O módulo **NÃO DEVE** assumir as responsabilidades abaixo (brief §1.2); cada
uma tem dono correto e é fronteira dura:

| # | Não-objetivo | Dono correto | Por quê |
|---|--------------|---------------|---------|
| N1 | Treinar/fine-tunar modelos ou gerar embeddings por conta própria | [017-Model-Router](../017-Model-Router/) | O Memory apenas *requisita* embeddings; centraliza custo/qualidade de inferência em um único módulo. |
| N2 | Decidir **quando/como** aprender uma política | [023-Learning](../023-Learning/) | O Memory *executa* a consolidação solicitada; a estratégia de aprendizado é domínio do Learning. |
| N3 | Montar a janela de contexto final / compressão de prompt / cache semântico de inferência | [011-Context](../011-Context/) | O Memory devolve itens rankeados; caber no orçamento de tokens é otimização de outro recurso escasso. |
| N4 | Persistir grafo de conhecimento público e GraphRAG global | [018/019-Knowledge](../018-Knowledge/) | O `KnowledgeGraphAdapter` do Memory é o grafo *do agente*, não o conhecimento compartilhado entre tenants. |
| N5 | Autorizar acesso (PDP) — atua apenas como PEP | [022-Policy](../022-Policy/) | O `MemoryPep` consulta decisão, nunca a define — mesma separação estrutural do `006-Kernel`. |
| N6 | Trilha de auditoria imutável — apenas emite eventos | [025-Audit](../025-Audit/) | O Memory emite; a durabilidade/imutabilidade é especializada do Audit. |
| N7 | Ciclo de vida do agente (spawn/suspend/kill) — apenas reage | [006-Kernel](../006-Kernel/), [008-Agent-Lifecycle](../008-Agent-Lifecycle/) | O Memory reage a eventos de ciclo de vida; nunca os inicia. |
| N8 | Orquestração de workflow/saga de negócio | [014-Workflow](../014-Workflow/) | Consolidação/esquecimento têm FSM próprias, mas não orquestram processos multi-módulo. |
| N9 | Cobrança/orçamento — apenas reporta consumo | [026-Cost-Optimizer](../026-Cost-Optimizer/) | O Memory reporta uso; definir/cobrar orçamento é FinOps. |

> **Fronteira crítica.** O Working/Short-Term Memory de um agente pertence ao
> **plano de controle**. O Agent Runtime (Python, plano de dados) **NÃO DEVE**
> acessar PostgreSQL/Redis diretamente — DEVE passar pela API do
> `MemoryService` via gRPC/NATS (regra de camadas de
> [001-Architecture](../001-Architecture/Architecture.md) §6; ADR-0107). Isso
> vale mesmo quando o acesso direto seria tecnicamente possível na rede
> interna: contorná-lo quebra capability enforcement, cota e consolidação
> versionada.

---

## 4. Personas Atendidas

O Memory é majoritariamente um módulo de **infraestrutura cognitiva interna**.
A [Visão Global](../000-Vision/Vision.md) §5 já nomeia o **Cientista de IA**
como persona primária deste módulo. A tabela abaixo detalha a relação
específica de cada persona:

| Persona | Necessidade | Operação/interação típica |
|---------|-------------|-----------------------------|
| **Cientista de IA** | Entender e melhorar qualidade de recuperação e estratégia de esquecimento. | Consulta `Memory Recall Rate`, ajusta `memory.hnsw.*`/`memory.decay.*`, avalia consolidação via [Benchmark.md](./Benchmark.md). |
| **Agent Runtime** (plano de dados) | Lembrar e recuperar sem acessar datastore diretamente. | `remember`/`recall` via gRPC (`aios.memory.v1`). |
| **Context Manager (011)** | Recall seletivo para montar a janela de contexto. | `recall` filtrado / evento `context.recall.requested`. |
| **Learning (023)** | Consolidar aprendizado sem regredir memória. | `consolidate`, `rollback` / evento `learning.consolidation.requested`. |
| **Desenvolvedor de Agentes** | Uma API estável, sem gerenciar sete backends distintos. | Indiretamente via SDK/Kernel, usando `remember`/`recall` no loop de raciocínio. |
| **Operador de Plataforma (SRE)** | Visibilidade de saturação de cota e crescimento de armazenamento. | Métricas `aios_memory_*`, `RetentionScheduler`, reindexação HNSW. |
| **Arquiteto Enterprise** | Conformidade de retenção e rastreabilidade de consolidação. | Define `retention_class` e `ForgettingPolicy` por tenant. |
| **CISO / Compliance (021/025)** | Executar direito ao esquecimento auditável. | `DELETE` (RTBF) / evento `security.rtbf.requested`; audita `item.forgotten`/`item.purged`. |
| **Gestor de Custo (FinOps, 026)** | Conhecer consumo e disparar poda por cota. | `getStats`, eventos `quota.exceeded`/`item.decayed`. |
| **Agente de IA** (consumidor não-humano) | Um contrato estável e idempotente. | Chamador direto (via Kernel) de `remember/recall/consolidate/forget`. |

---

## 5. Princípios de Design da Memória

Estes princípios especializam, para este módulo, os 10 princípios invioláveis
da [Visão Global](../000-Vision/Vision.md) §6. Toda ADR do módulo
(`ADR-0100`..`ADR-0109`) DEVE ser avaliada contra eles.

1. **Uniformidade sobre heterogeneidade.** Quatro verbos cognitivos DEVEM
   esconder sete backends físicos — nenhum chamador precisa conhecer o
   backend de uma camada (especializa o Princípio 7 — Extensibilidade sem
   fork).
2. **Nunca gerar embeddings, sempre requisitar.** O Memory NÃO DEVE chamar
   modelo de inferência diretamente; toda geração de vetor passa pelo
   `017-Model-Router` via `EmbeddingClient`.
3. **Reversibilidade contra *catastrophic forgetting*.** Toda consolidação
   DEVE gravar `ConsolidationVersion` (pré-imagem) antes de mutar e ser
   reversível (R4, FR-007) — especializa o Princípio 6 (idempotência e
   recuperação).
4. **Esquecer é uma feature, não um bug.** O esquecimento controlado (R5) é
   primeiro-classe e expresso como política declarativa (`ForgettingPolicy`),
   nunca como lógica ad-hoc; crescimento é contido por cota + poda.
5. **Isolamento por padrão.** `tenant_id` + Row-Level Security em toda
   operação; *default deny* no `MemoryPep`; namespace NATS por tenant (R10,
   RFC-0001 §5.8) — especializa os Princípios 2 e 3 da Visão Global.
6. **Cotas medidas e aplicadas por (tenant, agente, camada).** Crescimento de
   memória NÃO DEVE ser ilimitado; toda escrita passa pelo `QuotaManager`
   antes de persistir (mitigação direta de RSK-02).
7. **Idempotência antes de performance.** Toda mutação exposta (`remember`,
   `consolidate`, `forget`) DEVE ser segura para repetir via
   `Idempotency-Key` (RFC-0001 §5.5); otimizações de latência NÃO DEVEM
   comprometer essa garantia.
8. **Estado quente é acelerador, nunca fonte de verdade.** Redis (Working/
   Short-Term hot) é cache/estado volátil; PostgreSQL (+`pgvector`+AGE) é a
   fonte da verdade durável — qualquer inconsistência resolve a favor do
   PostgreSQL.
9. **Não reinventar contratos.** URN, envelope de evento, envelope de erro,
   idempotência e correlação vêm da RFC-0001 (§5) — o módulo os **reutiliza**,
   nunca os redefine.
10. **Observável de ponta a ponta.** 100% das operações DEVEM produzir trace
    OTel correlacionado e contribuir para a **Memory Recall Rate** (R11,
    NFR-013) — inclusive `recall`s vazios, cujo dado é tão valioso quanto o
    de sucesso.

---

## 6. Relação com a Visão Global do AIOS

O AIOS trata agentes como "processos" e recursos cognitivos como recursos de
SO (ver [Visão Global](../000-Vision/Vision.md) e
[Arquitetura Global](../001-Architecture/Architecture.md)). Dentro dessa
visão, o `010-Memory` é o subsistema de **memória** — par do `011-Context`
(atenção/janela), do `017-Model-Router` (inferência) e do `023-Learning`
(adaptação). Três relações merecem destaque:

- **V-R6 (memória hierárquica com consolidação e esquecimento)** — é a meta de
  visão da qual este módulo é a implementação direta e integral (Visão Global
  §9); toda a decomposição em sete camadas, a FSM de `MemoryItem` e as
  políticas de `ForgettingEngine` existem para satisfazê-la.
- **RSK-01 (*catastrophic forgetting*)** — mitigado em conjunto com
  `023-Learning`: o Memory fornece versionamento (`ConsolidationVersionManager`)
  e rollback automático (NFR-011); o Learning decide a política de aprendizado
  que dispara consolidação.
- **RSK-02 (crescimento descontrolado de memória)** — mitigado diretamente
  pelo Memory (`ForgettingEngine`, `QuotaManager`) em coordenação com
  `026-Cost-Optimizer`.

O módulo consome integralmente a baseline de contratos da RFC-0001 e propõe a
**RFC-0100** (*Memory Consolidation & Forgetting Contract*) para formalizar a
consolidação dirigida e a semântica de esquecimento inter-módulos.

O Memory também opera como o **ponto de tradução** entre a linguagem de
"paginação de memória" do SO clássico e a linguagem de "camada cognitiva" do
AIOS: onde um SO clássico decide o que fica em RAM e o que vai para swap/disco
por recência e frequência de acesso, o Memory decide o que permanece `ACTIVE`
em Redis/PostgreSQL quente e o que se torna `ARCHIVED` em MinIO frio, por
`decay_score`, `access_count` e `salience` (ver [Architecture.md](./Architecture.md)).

### 6.1 Exemplo ilustrativo (ciclo de vida cognitivo)

```
Agente observa um fato relevante numa tarefa
   │  remember(fact, layer=short_term)         → item ACTIVE (evento memory.item.stored)
   ▼
Uso repetido reforça o item (access_count↑, salience↑)
   │  consolidate() dirigido por 023           → SHORT_TERM → SEMANTIC (versionado)
   ▼
Semanas depois, recall(query, mode=hybrid)     → item recuperado, rankeado por relevância
   │  (Memory Recall Rate contabilizado)
   ▼
Sem uso, decay_score cai abaixo de θ           → DECAYING → FORGET_PENDING
   │  forget(policy=decay)                      → FORGOTTEN (tombstone) → PURGED após grace
```

Este ciclo (detalhado na FSM de [StateMachine.md](./StateMachine.md), derivada
do brief §4.1) resume a proposta de valor do módulo: **lembrar o que importa,
esquecer com controle e consolidar sem regredir.**

---

## 7. Não-Objetivos Explícitos (deste módulo)

Além das não-responsabilidades já listadas em §3.2 (fronteira de
propriedade), o módulo declara os seguintes não-objetivos de
**produto/design**, para evitar *scope creep*:

- NÃO tem por objetivo se tornar um **motor de aprendizado** — qualquer
  heurística de "quando consolidar" ou "que política de esquecimento aplicar"
  que exija aprendizado supervisionado/por reforço pertence a
  `023-Learning`; o Memory permanece um executor determinístico e auditável
  de políticas declarativas.
- NÃO tem por objetivo prover **UI** de qualquer natureza — inspeção humana do
  conteúdo de memória ocorre via `032-WebConsole`, nunca diretamente do
  serviço.
- NÃO tem por objetivo ser um **vector database genérico de propósito geral**
  fora do domínio de agentes do AIOS — a API é desenhada em torno do ciclo de
  vida cognitivo de `MemoryItem`, não como substituto de um banco vetorial
  standalone.
- NÃO tem por objetivo hospedar o **grafo de conhecimento de domínio
  público** consumido por GraphRAG global (isso é `018/019-Knowledge`); seu
  `KnowledgeGraphAdapter` é escopado à memória consolidada de um agente.
- NÃO tem por objetivo, na v1.0, garantir **consistência forte cross-região**
  para as camadas duráveis — a meta é RPO ≤ 5 min / RTO ≤ 15 min (NFR-006/
  NFR-007), não replicação síncrona multi-região.

---

## 8. Métrica-Norte

A métrica-norte do módulo é a **Memory Recall Rate**
([Glossário](../040-Glossary/Glossary.md)): fração de informação relevante
efetivamente recuperada quando necessária. A meta é **≥ 0,95** (NFR-002 em
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md)), e ela também
guarda a consolidação: uma consolidação que a regride além de 0,01 DEVE sofrer
rollback automático (NFR-011). Ver [Metrics.md](./Metrics.md).

---

## 9. Riscos Estratégicos Específicos do Módulo

Complementando os riscos globais da Visão ([Visão Global](../000-Vision/Vision.md)
§10), o Memory carrega riscos próprios por ser o guardião do estado cognitivo
mais volumoso e mais sensível (PII) da plataforma:

| Risco | Descrição | Mitigação | Decisão/Referência |
|-------|-----------|-----------|----------------------|
| RSK-M01 | Consolidação corrompe conhecimento previamente correto (regressão de qualidade). | Snapshot versionado obrigatório antes de mutar; validação pós-consolidação; rollback automático se Recall Rate regredir além de 0,01 (NFR-011). | ADR-0102, brief §4.2/§9 (F6) |
| RSK-M02 | Crescimento vetorial explosivo torna ANN inviável em latência (índice HNSW degrada). | Cotas por (tenant, agente, camada); particionamento por tenant; reindexação online; meta ≥ 10⁸ vetores/tenant, p99 ≤ 150 ms (NFR-012). | ADR-0101, ADR-0108, brief §10 |
| RSK-M03 | Esquecimento agressivo demais remove informação ainda necessária (perda irrecuperável). | `grace_period` configurável antes de `PURGED`; tombstone com linhagem preservada em `FORGOTTEN`; `legal_hold` bloqueia expurgo salvo RTBF autorizado. | ADR-0103, brief §4.1 (invariantes), §12.3 |
| RSK-M04 | Vazamento de PII entre tenants ou em eventos/logs. | RLS obrigatória em toda tabela; flag `pii` + redação/tokenização em payloads de evento; erro sem `detail` sensível. | ADR-0107, ADR-0109, brief §12.2 (STRIDE) |
| RSK-M05 | Indisponibilidade do Model Router (017) bloqueia escrita de memória semântica/episódica. | Item persiste em `INGESTED` sem embedding; geração eventual em backlog; recall degrada para lexical/grafo. | brief §9 (F3) |

Decisões DEVEM apontar para as ADRs acima (índice em [ADR.md](./ADR.md));
nada neste documento decide arquitetura localmente.

---

## 10. Referências

- Visão Global do AIOS: [../000-Vision/Vision.md](../000-Vision/Vision.md)
- Arquitetura Global: [../001-Architecture/Architecture.md](../001-Architecture/Architecture.md)
- Glossário: [../040-Glossary/Glossary.md](../040-Glossary/Glossary.md)
- Contratos centrais: [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md)
- Design Brief interno (fonte de verdade do módulo): [./_DESIGN_BRIEF.md](./_DESIGN_BRIEF.md)
- Índice do módulo: [./README.md](./README.md)
- Próximo documento na cadeia: [./Architecture.md](./Architecture.md)
- Perguntas frequentes: [./FAQ.md](./FAQ.md)
- Exemplos executáveis: [./Examples.md](./Examples.md)
- Módulos acoplados: [011-Context](../011-Context/), [017-Model-Router](../017-Model-Router/),
  [022-Policy](../022-Policy/), [023-Learning](../023-Learning/),
  [024-Observability](../024-Observability/), [025-Audit](../025-Audit/),
  [026-Cost-Optimizer](../026-Cost-Optimizer/), [006-Kernel](../006-Kernel/),
  [018-Knowledge](../018-Knowledge/), [021-Security](../021-Security/),
  [005-Database](../005-Database/).

---

*Fim de `Vision.md`. Próximo documento na cadeia do módulo:* [`./Architecture.md`](./Architecture.md).
