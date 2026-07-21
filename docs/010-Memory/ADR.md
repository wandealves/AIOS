---
Documento: ADR.md
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0007, ADR-0008, ADR-0009, ADR-0010, ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001
Depende de: 001-Architecture, 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 002-ADR
---

# 010-Memory — Índice de ADRs

> **Escopo deste documento.** Este documento **NÃO** decide nada por si — é um
> **índice navegável** das *Architecture Decision Records* (ADR) que afetam o
> módulo 010-Memory. As decisões em si residem em `../002-ADR/`, no formato
> definido por `../002-ADR/TEMPLATE.md` (Contexto · Problema · Alternativas ·
> Análise · Escolha · Consequências · Riscos · Trade-offs). Este arquivo **DEVE**
> ser mantido sincronizado com `./_DESIGN_BRIEF.md` §11 e com
> `../002-ADR/README.md`; qualquer divergência é um defeito de documentação.

> **Convenção de numeração.** Conforme `./_DESIGN_BRIEF.md` §0/§11, a faixa
> `ADR-0100`–`ADR-0109` está **reservada** ao módulo 010-Memory (regra
> `módulo × 10`, evitando colisão com faixas de outros módulos — ex.:
> `009-Scheduler` usa `ADR-0090`–`ADR-0099`). Nenhuma outra numeração PODE ser
> usada por este módulo sem atualizar este índice e `../002-ADR/README.md`. Das
> dez posições reservadas, dez estão descritas neste batch inicial
> (`ADR-0100`–`ADR-0109`); não há posições livres remanescentes nesta faixa —
> qualquer decisão adicional exigirá primeiro uma revisão de escopo do brief
> (§11) antes de reivindicar numeração fora da faixa reservada.

---

## 1. ADRs Globais que Restringem o Memory

Estas ADRs foram decididas na fundação canônica do AIOS (`../002-ADR/`) e
**DEVEM** ser respeitadas por qualquer ADR específica do Memory — elas não são
redecididas aqui, apenas referenciadas e especializadas para o domínio de
memória hierárquica, consolidação e esquecimento.

| ADR | Título | Status | Como restringe o Memory |
|-----|--------|--------|----------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O `MemoryService` é o **gerenciador de memória cognitiva** do AIOS — análogo à MMU + swap + filesystem de um SO clássico, não um vector store de terceiros embutido numa lib de RAG. As quatro operações canônicas (`remember/recall/consolidate/forget`, §1 do brief) são syscalls de sistema (ver `Syscall Cognitiva` no `../040-Glossary/Glossary.md`), não plugins de framework. |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | O `MemoryService` vive inteiramente no **control plane** (.NET 10); o Agent Runtime (Python, plano de dados) NÃO DEVE acessar PostgreSQL/Redis diretamente — DEVE passar pela API do `MemoryService` via gRPC/NATS (fronteira crítica do brief §1.2, especializada em ADR-0107). |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | `MemoryApiFacade`, `LayerRouter`, `RecallEngine`, `ConsolidationEngine` e `ForgettingEngine` são implementados em **.NET 10**, garantindo o caminho quente de baixa latência exigido por NFR-001/NFR-003 (p99 ≤ 30 ms `remember` inline; ≤ 50 ms `recall` hot). |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | Todo evento de domínio (`aios.<tenant>.memory.item.*`, `aios.<tenant>.memory.consolidation.*`, §6 do brief) é publicado via **NATS/JetStream** através do `OutboxPublisher`, nunca via um bus alternativo; consumido de forma at-least-once com dedup por `event.id`. |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | Fundamenta diretamente o mapeamento físico das camadas Long-Term/Semantic/Procedural/Episodic (PostgreSQL), a busca vetorial ANN (Semantic/Episodic via `pgvector`/HNSW, especializada em ADR-0101) e o `KnowledgeGraphAdapter` (Apache AGE/openCypher). Este módulo é o principal consumidor de ambas as extensões no AIOS. |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e locks | Accepted | `WorkingMemoryStore` (TTL curto) e o lado quente do `ShortTermMemoryStore` usam **Redis** exclusivamente; `IdempotencyStore` e os locks distribuídos de consolidação por `(tenant, agent)` (§10 do brief) também residem em Redis — nenhum cache alternativo é introduzido no caminho quente. |
| [ADR-0007](../002-ADR/ADR-0007-Memoria-Hierarquica.md) | Memória hierárquica de sete camadas | Accepted | **Decisão fundacional herdada.** Este módulo é a materialização direta da ADR-0007: as sete camadas (Working, Short-Term, Long-Term, Semantic, Procedural, Episodic, Knowledge Graph), cada uma com política própria de retenção/consolidação/esquecimento, sob uma API de Memory uniforme, consolidação dirigida pelo `023-Learning`. Todas as ADRs `ADR-0100`–`ADR-0109` (§2) são **especializações** desta decisão fundacional, nunca uma redecisão dela. |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | Accepted | Fundamenta o papel do `MemoryPep` como **PEP**: toda operação (`remember/recall/consolidate/forget`) consulta o PDP (`022-Policy`) antes de executar, com postura *default deny* — indisponibilidade do PDP nega a operação (`AIOS-MEM-0004`), nunca permite por omissão. |
| [ADR-0009](../002-ADR/ADR-0009-Model-Router-Multiobjetivo.md) | Model Router multiobjetivo | Accepted | Estabelece o precedente de que o Memory **nunca** chama LLM diretamente (N1 do brief §1.2): todo embedding é requisitado ao `017-Model-Router` via `EmbeddingClient`, que herda do Router a escolha multiobjetivo de modelo de embedding por custo/qualidade/latência/disponibilidade. |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | Todo o `MemoryTelemetry` (spans OTel, métricas `aios_memory_*`, logs Serilog→Seq) e a emissão de eventos consumidos por `025-Audit` (`item.forgotten`, `item.purged`, `consolidation.*`) implementam esta ADR — observabilidade e proveniência não são adicionadas a posteriori, são parte do contrato de memória (R11 do brief). |

---

## 2. ADRs Específicas do Módulo 010-Memory (Propostas)

> **Estado de redação.** As ADRs abaixo estão **reservadas** na faixa
> `ADR-0100`–`ADR-0109` e seu conteúdo (Contexto/Problema/Alternativas/Análise/
> Escolha/Consequências/Riscos/Trade-offs) está **descrito no
> `./_DESIGN_BRIEF.md` §11** deste módulo, que é a fonte única de verdade até
> que cada ADR seja escrita como arquivo próprio em `../002-ADR/` e tenha seu
> `Status` promovido de `Proposed` para `Accepted`. Nenhum destes arquivos
> existe fisicamente em `../002-ADR/` no momento desta versão do índice; este
> documento resume o conteúdo pretendido para orientar quem for redigi-las.

| ADR | Título | Status | Resumo da decisão (1 linha) | Módulos afetados |
|-----|--------|--------|------------------------------|-------------------|
| ADR-0100 | Modelo canônico de 7 camadas de memória e mapeamento a backends físicos | Proposed | Fixar o mapeamento canônico Working→Redis, Short-Term→Redis+PG, Long-Term→PG, Semantic→PG+pgvector, Procedural→PG, Episodic→PG+pgvector (particionado por tempo) e Knowledge Graph→Apache AGE, como especialização física e definitiva de ADR-0007. | 010, 005 |
| ADR-0101 | Estratégia de embeddings e índice vetorial (pgvector + HNSW) e parâmetros por tenant | Proposed | Adotar `pgvector` com índice HNSW (`hnsw.m`, `hnsw.ef_construction`, `hnsw.ef_search` configuráveis por tenant) como único mecanismo de busca ANN das camadas Semantic/Episodic, com `memory.embedding.dim` fixo por instalação (reindex ao mudar). | 010, 005, 017 |
| ADR-0102 | Consolidação versionada e rollback como defesa contra *catastrophic forgetting* | Proposed | Toda consolidação DEVE gravar `ConsolidationVersion` (snapshot pré-imagem) antes de mutar itens, e todo `ConsolidationJob` DEVE ser revertível a uma versão anterior sob falha ou regressão de Memory Recall Rate (NFR-011), conforme a máquina de estados `PENDING→SNAPSHOTTING→RUNNING→VALIDATING→COMMITTED` do brief §4.2. | 010, 023 |
| ADR-0103 | Políticas de esquecimento controlado (TTL, decaimento, LRU, cota, RTBF) e sua precedência | Proposed | Definir as cinco estratégias de `ForgettingPolicy` (`ttl\|decay\|lru\|quota\|rtbf`) e sua ordem de precedência quando concorrentes, respeitando `legal_hold` como bloqueio absoluto salvo RTBF autorizado (invariante §4.1 do brief). | 010, 021, 025 |
| ADR-0104 | Fusão e re-ranqueamento de recall híbrido (lexical + vetorial + grafo; RRF vs. weighted) | Proposed | Adotar Reciprocal Rank Fusion (RRF) como estratégia default de fusão entre os três modos de recuperação (`lexical\|vector\|graph`), com fusão `weighted` disponível como alternativa configurável por tenant (`memory.recall.fusion`). | 010, 011 |
| ADR-0105 | Cotas de memória e backpressure por (tenant, agente, camada) | Proposed | `QuotaManager` contabiliza itens/bytes/vetores por `(tenant, agent, layer)` em Redis (contadores quentes) com espelho em PostgreSQL, negando escrita (`AIOS-MEM-0010`) ou represando (`AIOS-MEM-0011`) ao exceder, e reportando consumo ao `026-Cost-Optimizer`. | 010, 026 |
| ADR-0106 | Externalização de blobs grandes (MinIO content-addressed) e limiar inline | Proposed | Conteúdo acima de `memory.item.inline_max_bytes` é externalizado para MinIO por endereçamento por conteúdo (SHA-256), com deduplicação automática e TTL alinhado à `retention_class` do item; PostgreSQL mantém apenas `content_ref` + metadados + embedding. | 010 |
| ADR-0107 | Fronteira de acesso Runtime↔Memory (sem acesso direto a datastore; gRPC/NATS) | Proposed | O Agent Runtime (Python, plano de dados) NÃO DEVE conectar-se diretamente a Redis/PostgreSQL/MinIO/AGE do Memory — toda leitura/escrita passa pela API gRPC/REST do `MemoryApiFacade`, especializando a regra de camadas de `../001-Architecture/Architecture.md` §6 e ADR-0002. | 010, 007, 001 |
| ADR-0108 | Particionamento/sharding de memória vetorial e Episodic (tempo) rumo a milhões de agentes | Proposed | Vetores particionados por `shard = hash(tenant_id, agent_id) mod N` (alinhado a `001-Architecture` §12); Episodic particionado adicionalmente por tempo (`RANGE` mensal); HNSW reindexável online por partição sem downtime, mirando ≥ 10⁸ vetores/tenant (NFR-012). | 010, 001, 027 |
| ADR-0109 | Modelo de retenção legal, base legal por item e direito ao esquecimento (LGPD/GDPR) | Proposed | Todo `MemoryItem` DEVE carregar `legal_basis` (`consent\|contract\|legitimate_interest\|legal_obligation`) e `retention_class` (`ephemeral\|standard\|extended\|legal_hold`); expurgo por RTBF é assíncrono, rastreável, e remove vetor + blob + tombstone após `grace_period` configurável. | 010, 021, 025 |

### 2.1 Justificativa de cada ADR proposta (contexto resumido)

| ADR | Por que uma ADR (e não um detalhe de implementação)? |
|-----|-------------------------------------------------------|
| ADR-0100 | O mapeamento camada↔backend físico é a especialização concreta de uma decisão fundacional (ADR-0007) e determina o custo de mudança de qualquer camada (ex.: trocar Episodic de PG+pgvector para outro backend implicaria migração de dados de produção) — decisão estrutural de longo prazo, não um detalhe de configuração. |
| ADR-0101 | A escolha de HNSW (vs. IVFFlat, vs. índice externo dedicado) e a fixação de `memory.embedding.dim` por instalação (não recarregável em runtime, exige reindex) tem custo de migração alto e afeta diretamente NFR-002 (Recall@10 ≥ 0,95) e NFR-012 (escala a 10⁸ vetores) — decisão de modelo de dados, não de tuning pontual. |
| ADR-0102 | A obrigatoriedade de snapshot versionado antes de toda consolidação é a principal defesa arquitetural contra *catastrophic forgetting* (risco identificado na própria ADR-0007) — mudar essa garantia depois de adotada quebraria a semântica de rollback consumida por `023-Learning` e pela auditoria (`025`). |
| ADR-0103 | A ordem de precedência entre TTL/decaimento/LRU/cota/RTBF quando múltiplas políticas competem pelo mesmo item determina comportamento observável de retenção de dados de tenants — decisão de contrato de comportamento, com implicações diretas em conformidade LGPD/GDPR (`legal_hold`). |
| ADR-0104 | RRF vs. fusão ponderada (`weighted`) determina como relevância/recência/saliência/escopo são combinadas no ranking final de `recall` — escolher a estratégia default afeta a experiência de todo consumidor (`011-Context` incluso) e é reversível apenas com re-tuning de configuração em produção. |
| ADR-0105 | O modelo de contabilização de cotas (Redis quente + espelho em PG, granularidade `(tenant, agent, layer)`) define o comportamento de backpressure (NFR-009) e a integração com `026-Cost-Optimizer` — decisão de modelo de dados e de degradação sob saturação, não um detalhe de contador. |
| ADR-0106 | O limiar inline vs. externalizado e o esquema content-addressed (SHA-256, dedup automática) definem a fronteira entre PostgreSQL e MinIO para todo o módulo — mudar o esquema de endereçamento depois de adotado exige reprocessar referências existentes. |
| ADR-0107 | Preservar a fronteira Runtime↔Memory como API gRPC/NATS (nunca acesso direto a datastore) é o que garante a separação control/data plane (ADR-0002) especificamente para memória — decisão de segurança e de evolução independente dos backends físicos sem quebrar o Agent Runtime. |
| ADR-0108 | A estratégia de sharding determinístico e particionamento temporal do Episodic é o que viabiliza a escala a milhões de agentes (visão global do AIOS) mantendo p99 de ANN ≤ 150 ms — decisão que atravessa a fronteira com `027-Cluster`/`001-Architecture`. |
| ADR-0109 | O modelo de base legal e classes de retenção por item é obrigação regulatória (LGPD/GDPR) que atravessa todas as sete camadas — decisão de conformidade com efeito estrutural em todo o schema de `MemoryItem`, não uma opção de configuração local. |

---

## 3. Rastreabilidade ADR → Componente → Requisito

| ADR | Componente(s) afetado(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) |
|-----|-------------------------------------------------------|-------------------------------|
| ADR-0100 | `WorkingMemoryStore`, `ShortTermMemoryStore`, `LongTermMemoryStore`, `SemanticMemoryStore`, `ProceduralMemoryStore`, `EpisodicMemoryStore`, `KnowledgeGraphAdapter` | FR-004, NFR-006, NFR-008 |
| ADR-0101 | `EmbeddingClient`, `SemanticMemoryStore`, `EpisodicMemoryStore` | FR-003, NFR-002, NFR-012 |
| ADR-0102 | `ConsolidationEngine`, `ConsolidationVersionManager` | FR-005, FR-007, NFR-011 |
| ADR-0103 | `ForgettingEngine`, `RetentionScheduler` | FR-006, FR-011, NFR-009 |
| ADR-0104 | `RecallEngine` | FR-002, FR-013, NFR-003 |
| ADR-0105 | `QuotaManager` | FR-008, NFR-009 |
| ADR-0106 | `BlobStoreAdapter`, `LayerRouter` | FR-009 |
| ADR-0107 | `MemoryApiFacade`, `MemoryPep` | NFR-005, NFR-010 |
| ADR-0108 | `LayerRouter`, `SemanticMemoryStore`, `EpisodicMemoryStore` | NFR-004, NFR-012 |
| ADR-0109 | `MemoryItem` (modelo de dados), `ForgettingEngine` | FR-011, NFR-010 |

---

## 4. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `ADR-01NN-<slug>.md` em `../002-ADR/` usando
   `../002-ADR/TEMPLATE.md`, preenchendo todas as seções obrigatórias
   (Contexto, Problema, Alternativas, Análise, Escolha, Consequências, Riscos,
   Trade-offs).
2. O conteúdo **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste módulo nem
   a decisão fundacional `ADR-0007`; caso a análise revele necessidade de
   divergência, o brief DEVE ser atualizado primeiro (o brief é a fonte única
   de verdade).
3. Revisão pelo Arquiteto-Chefe e pelo(s) módulo(s) afetado(s) listados na
   tabela da §3, com atenção especial a `011-Context` (consumidor de `recall`
   seletivo), `023-Learning` (dirige consolidação e depende de rollback) e
   `021-Security`/`025-Audit` (RTBF, `legal_hold`, trilha de esquecimento).
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da ADR e neste
   índice; o `../002-ADR/README.md` global é atualizado com a nova linha.
5. ADRs aceitas **NÃO SÃO** editadas — mudanças de rumo geram uma nova ADR que
   marca a anterior como `Superseded by ADR-XXXX`. A faixa `ADR-0100`–`ADR-0109`
   está integralmente reservada por este batch; qualquer decisão adicional do
   módulo que não caiba em nenhuma das dez ADRs listadas DEVE primeiro revisar
   o escopo do `_DESIGN_BRIEF.md` §11 antes de solicitar numeração fora da faixa.

---

## 5. Referências

- Índice global de ADRs: `../002-ADR/README.md`
- Template de ADR: `../002-ADR/TEMPLATE.md`
- Decisão fundacional herdada: `../002-ADR/ADR-0007-Memoria-Hierarquica.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §11
- Arquitetura do módulo: `./Architecture.md`
- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Índice de RFCs do módulo: `./RFC.md`

*Fim do índice de ADRs do módulo 010-Memory.*
