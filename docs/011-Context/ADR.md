---
Documento: ADR.md
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0007, ADR-0008, ADR-0009, ADR-0010, ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001
Depende de: 001-Architecture, 005-Database, 010-Memory, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 002-ADR
---

# 011-Context — Índice de ADRs

> **Escopo deste documento.** Este documento **NÃO** decide nada por si — é um
> **índice navegável** das *Architecture Decision Records* (ADR) que afetam o
> módulo `011-Context`. As decisões em si residem em `../002-ADR/`, no formato
> definido por `../002-ADR/TEMPLATE.md` (Contexto · Problema · Alternativas ·
> Análise · Escolha · Consequências · Riscos · Trade-offs). Este arquivo **DEVE**
> ser mantido sincronizado com `./_DESIGN_BRIEF.md` §11 e com
> `../002-ADR/README.md`; qualquer divergência é um defeito de documentação.

> **Convenção de numeração.** Conforme `./_DESIGN_BRIEF.md` §11, a faixa
> `ADR-0110`–`ADR-0119` está **reservada** ao módulo `011-Context` (regra
> `módulo × 10`, evitando colisão com faixas de outros módulos — ex.:
> `010-Memory` usa `ADR-0100`–`ADR-0109`, `012-Planning` usará
> `ADR-0120`–`ADR-0129`). Nenhuma outra numeração PODE ser usada por este
> módulo sem atualizar este índice e `../002-ADR/README.md`. Das dez posições
> reservadas, dez estão descritas neste batch inicial (`ADR-0110`–`ADR-0119`);
> não há posições livres remanescentes nesta faixa — qualquer decisão
> adicional exigirá primeiro uma revisão de escopo do brief (§11) antes de
> reivindicar numeração fora da faixa reservada.

---

## 1. ADRs Globais que Restringem o Context

Estas ADRs foram decididas na fundação canônica do AIOS (`../002-ADR/`) e
**DEVEM** ser respeitadas por qualquer ADR específica do Context — elas não
são redecididas aqui, apenas referenciadas e especializadas para o domínio de
montagem de janela de contexto, compressão hierárquica, recuperação seletiva e
cache semântico.

| ADR | Título | Status | Como restringe o Context |
|-----|--------|--------|----------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O `ContextAssembler` é o **gerenciador de cache/paginação cognitivo** do AIOS — análogo ao subsistema de cache/paginação de um SO clássico (decide o que "mantém na página" na janela ativa, o que "compacta" e o que "evita recomputar"), não um wrapper de biblioteca de *prompt engineering*. As operações canônicas (`assemble/compress/cache lookup/cache store`, §1 do brief) são tratadas como funções de sistema análogas a syscalls de gerência de memória virtual, não plugins de framework de RAG. |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | O `ContextService` vive inteiramente no **control plane** (.NET 10); o Agent Runtime (Python, plano de dados) NÃO DEVE montar contexto localmente nem acessar PostgreSQL/Redis/MinIO do Context diretamente — DEVE passar pela API do `ContextApiGateway` via gRPC/REST (fronteira especializada em ADR-0117/ADR-0119). |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | `ContextAssembler`, `TokenBudgeter`, `TokenCounter`, `SelectiveRetriever`, `RelevanceRanker`, `RedundancyEliminator`, `HierarchicalCompressor` e `SemanticCacheManager` são implementados em **.NET 10**, garantindo o caminho quente de baixa latência exigido por NFR-001/NFR-002 (p99 ≤ 20 ms `CacheLookup`; ≤ 150 ms `Assemble` sem sumarização). |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | Todo evento de domínio (`aios.<tenant>.context.window.*`, `aios.<tenant>.context.cache.*`, `aios.<tenant>.context.budget.updated`, §6 do brief) é publicado via **NATS/JetStream** através do `EventPublisher`, nunca via um bus alternativo; consumido de forma at-least-once com dedup por `event.id`. |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | Fundamenta diretamente a persistência de `ContextBundle`/`ContextFragment`/`SemanticCacheEntry`/`SummaryNode` em PostgreSQL, e a busca vetorial ANN (L2 do cache semântico e ranking de fragmentos) via **pgvector** (HNSW/IVFFLAT, especializada em ADR-0111/ADR-0112). Este módulo NÃO usa Apache AGE — não mantém grafo de conhecimento (N-05 do brief, propriedade de `018/019-Knowledge`). |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e locks | Accepted | O **L1 do `SemanticCacheManager`** (cache semântico sub-ms por shard, TTL curto) reside exclusivamente em **Redis**; locks distribuídos `SET NX PX` por `(tenant, cache_key)` (§10 do brief, anti *thundering herd* em cache-fill) e o registro de idempotência também residem em Redis — nenhum cache alternativo é introduzido no caminho quente. |
| [ADR-0007](../002-ADR/ADR-0007-Memoria-Hierarquica.md) | Memória hierárquica de sete camadas | Accepted | **Fronteira respeitada, não redecidida.** O `011-Context` NÃO É uma camada de memória e NÃO decide o que é lembrado nem por quanto tempo (N-02 do brief) — consome `recall` seletivo do `010-Memory` via `MemoryClient` e decide apenas o que entra na janela **agora** e sob qual forma comprimida. `SummaryNode` materializado por este módulo é cache efêmero de curta duração, nunca memória canônica (§1.3 do brief). |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | Accepted | Fundamenta o papel do `ContextPolicyGuard` como **PEP**: toda operação privilegiada (assemble para outro agente, upsert de `BudgetProfile`, invalidação em massa de cache) consulta o PDP (`022-Policy`) antes de executar, com postura *default deny* — indisponibilidade do PDP nega a operação, nunca permite por omissão. |
| [ADR-0009](../002-ADR/ADR-0009-Model-Router-Multiobjetivo.md) | Model Router multiobjetivo | Accepted | Estabelece o precedente de que o Context **nunca** chama provedores de LLM diretamente para inferência final (N-01 do brief): limites de janela, tokenizer, modelo de embedding e modelo de sumarização são todos requisitados ao `017-Model-Router` via `ModelRouterClient`, que herda do Router a escolha multiobjetivo por custo/qualidade/latência/disponibilidade (ex.: `context.compression.summarization_model_hint = cheap-small`). |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | Todo o `TelemetryEmitter` (spans OTel, métricas `aios_context_*` como **Context Compression Ratio** e cache hit ratio, logs Serilog→Seq) e a emissão de eventos consumidos por `025-Audit` implementam esta ADR — observabilidade e proveniência não são adicionadas a posteriori, são parte do contrato de montagem de contexto (R-08 do brief). |

---

## 2. ADRs Específicas do Módulo 011-Context (Propostas)

> **Estado de redação.** As ADRs abaixo estão **reservadas** na faixa
> `ADR-0110`–`ADR-0119` e seu conteúdo (Contexto/Problema/Alternativas/Análise/
> Escolha/Consequências/Riscos/Trade-offs) está **descrito no
> `./_DESIGN_BRIEF.md` §11** deste módulo, que é a fonte única de verdade até
> que cada ADR seja escrita como arquivo próprio em `../002-ADR/` e tenha seu
> `Status` promovido de `Proposed` para `Accepted`. Nenhum destes arquivos
> existe fisicamente em `../002-ADR/` no momento desta versão do índice; este
> documento resume o conteúdo pretendido para orientar quem for redigi-las.

| ADR | Título | Status | Resumo da decisão (1 linha) | Módulos afetados |
|-----|--------|--------|------------------------------|-------------------|
| ADR-0110 | Compressão hierárquica (map-reduce summarization) vs. truncamento como padrão | Proposed | Adotar sumarização hierárquica map-reduce multinível (`context.compression.method = hierarchical`, `max_levels ≤ 5`) como estratégia **default** de compressão, com `extractive`/`truncate` como fallbacks configuráveis por tenant/task_type, preservando itens de relevância ≥ p90 (FR-003). | 011, 017 |
| ADR-0111 | Cache semântico em dois níveis (Redis L1 + pgvector L2) com chave por similaridade | Proposed | Fixar a arquitetura de cache em **dois níveis** — Redis L1 (sub-ms, TTL curto, por shard) e PostgreSQL/pgvector L2 (durável, HNSW/IVFFLAT) — como único mecanismo de `SemanticCacheManager`, com `prompt_hash` como fast path exato e `prompt_embedding` como slow path por similaridade (NFR-001, NFR-006). | 011, 005, 006 |
| ADR-0112 | Limiar de similaridade e política precision/recall do cache semântico | Proposed | Definir `context.cache.similarity_threshold` default `0.92` (faixa `0.80–0.99`) como limiar mínimo de aceitação de hit, balanceando taxa de falso-hit (NFR-011 ≤ 0,5%) contra cache hit ratio (NFR-006 ≥ 0,35), com `reuse_threshold` (`0.95`) mais estrito para reuso de `SummaryNode`. | 011, 017 |
| ADR-0113 | Algoritmo de token budgeting e alocação por seção (system/instrução/memória/histórico/tools) | Proposed | Adotar `BudgetProfile` com pesos de alocação por seção (`alloc_system`, `alloc_instruction`, `alloc_memory`, `alloc_history`, `alloc_tools`) somando `1.0` (invariante validada em escrita, `AIOS-CTX-0008`) e `reserved_output_ratio` default `0.25`, calculado pelo `TokenBudgeter` a partir dos limites do modelo fornecidos pelo `017`. | 011, 017 |
| ADR-0114 | Registro de tokenizers multi-provedor: estimativa vs. contagem exata via `017` | Proposed | O `TokenCounter` DEVE consultar o registro de tokenizers do `017-Model-Router` para contagem exata por modelo, com fallback de **estimativa** (erro ≤ 2%, NFR-010) quando o tokenizer exato está indisponível (`AIOS-CTX-0003`), nunca inventando uma contagem sem sinalizar o modo usado. | 011, 017 |
| ADR-0115 | Modelo de sumarização (roteado a modelo barato via `017`) e verificação de fidelidade | Proposed | Sumarização é roteada a um modelo de custo reduzido via hint (`context.compression.summarization_model_hint`), com verificação de fidelidade mínima (preservação de entidades/itens de alta relevância) antes de aceitar o sumário no `ContextBundle`; falha de fidelidade aciona fallback `truncate` determinístico (`AIOS-CTX-0004`). | 011, 017, 026 |
| ADR-0116 | Política de invalidação e eviction (TTL + LRU semântico + event-driven) | Proposed | O `EvictionManager` combina três gatilhos — TTL (`context.cache.default_ttl_seconds`), LRU semântico (por `last_hit_at`/pressão de memória) e invalidação **event-driven** reagindo a `memory.item.consolidated`/`deleted`, `knowledge.node.updated`, `agent.lifecycle.terminated` — como única fonte de remoção de `SemanticCacheEntry`, nunca eviction ad hoc fora deste componente. | 011, 010, 018/019 |
| ADR-0117 | Offload de fragmentos grandes em MinIO vs. armazenamento inline em PostgreSQL | Proposed | `ContextFragment.content_inline` é usado para payloads ≤ `context.limits.max_inline_bytes` (default `16384` bytes); acima do limiar, o conteúdo é externalizado ao MinIO e referenciado por `content_ref`, mantendo PostgreSQL enxuto para o caminho quente de leitura/ranking. | 011, 005 |
| ADR-0118 | Degradação graciosa e truncamento seguro quando `010`/`017` indisponíveis | Proposed | Sob indisponibilidade de `010-Memory` (timeout de recall) ou `017-Model-Router` (limites/tokenizer/embedding/sumarização), o `ContextAssembler` DEVE transicionar para `DEGRADED` e entregar um bundle truncado seguro (system+history mínimo) em vez de falhar, respeitando `context.degraded.allow_truncate_fallback`. | 011, 010, 017 |
| ADR-0119 | Isolamento de cache por tenant e defesa contra cache poisoning/falso-hit | Proposed | Namespace de cache, subjects NATS e RLS por `tenant_id` são a única fronteira de isolamento entre tenants no `SemanticCacheManager`; falso-hit acima de NFR-011 (≤ 0,5%) aciona elevação temporária de `similarity_threshold` e quarentena do cluster de embeddings afetado (FM-08 do brief), nunca compartilhamento implícito de entradas entre tenants. | 011, 021, 022 |

### 2.1 Justificativa de cada ADR proposta (contexto resumido)

| ADR | Por que uma ADR (e não um detalhe de implementação)? |
|-----|-------------------------------------------------------|
| ADR-0110 | A escolha de sumarização hierárquica map-reduce como padrão (vs. truncamento simples ou extração puramente lexical) determina diretamente a **Context Compression Ratio** (NFR-004 ≥ 0,50) e o trade-off com Task Completion Rate (NFR-005) — decisão estrutural de algoritmo de compressão, não um parâmetro de tuning pontual. |
| ADR-0111 | A topologia de dois níveis (Redis L1 + pgvector L2) determina a curva de latência de todo o módulo (NFR-001/NFR-002) e o modelo de custo de infraestrutura — mudar essa topologia depois de adotada implica migração de dados de cache em produção e reengenharia do `SemanticCacheManager`. |
| ADR-0112 | O limiar de similaridade é o principal *dial* de trade-off precision/recall do cache semântico: valores baixos aumentam hit ratio mas elevam falso-hit (risco de servir resposta semanticamente inadequada); valores altos protegem qualidade mas reduzem economia de custo (NFR-007) — decisão de política de qualidade, não um valor de configuração arbitrário. |
| ADR-0113 | O algoritmo de alocação de orçamento por seção determina o que sobrevive à montagem de contexto sob pressão de espaço (system vs. memória vs. histórico vs. tools) — decisão de contrato de comportamento observável por todo agente consumidor, com invariante formal (`soma = 1.0`) que não pode divergir silenciosamente entre implementações. |
| ADR-0114 | A escolha entre contagem exata (tokenizer real do provedor) e estimativa determina a precisão de todo o *budgeting* (NFR-010 ≤ 2% de erro) — decisão de fonte de verdade de tokens que atravessa a fronteira com `017-Model-Router` e não pode ser decidida isoladamente por cada chamador. |
| ADR-0115 | O roteamento de sumarização a um modelo barato (vs. sempre o modelo alvo da tarefa) tem implicação direta de custo (`026-Cost-Optimizer`) e de risco de fidelidade (sumário que descarta informação crítica) — decisão que precisa de critério explícito de verificação, não apenas "usar o barato". |
| ADR-0116 | Combinar três gatilhos de eviction (TTL, LRU semântico, event-driven) em vez de apenas TTL fixo é o que garante que o cache nunca sirva dados obsoletos após consolidação/exclusão de memória a montante (FR-008: invalidação ≤ 5 s) — decisão de consistência entre módulos, não um detalhe de expiração local. |
| ADR-0117 | O limiar inline vs. externalizado define a fronteira entre PostgreSQL e MinIO para todo o módulo, com impacto direto em I/O do caminho quente — mudar esse esquema depois de adotado exige reprocessar referências de fragmentos existentes. |
| ADR-0118 | A política de degradação graciosa é o que impede que uma dependência externa (`010`/`017`) indisponível derrube o módulo inteiro — decisão de resiliência estrutural (NFR-008, disponibilidade ≥ 99,95%) que precisa de contrato explícito de "o que é um bundle aceitável sob degradação". |
| ADR-0119 | O isolamento de cache por tenant é a principal defesa contra vazamento de informação entre tenants (STRIDE: Information Disclosure) e contra cache poisoning — decisão de segurança com efeito estrutural sobre todo o schema de `SemanticCacheEntry`, não uma opção de configuração local. |

---

## 3. Rastreabilidade ADR → Componente → Requisito

| ADR | Componente(s) afetado(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) |
|-----|-------------------------------------------------------|-------------------------------|
| ADR-0110 | `HierarchicalCompressor`, `TokenCounter` | FR-003, NFR-004, NFR-005 |
| ADR-0111 | `SemanticCacheManager` | FR-006, NFR-001, NFR-006 |
| ADR-0112 | `SemanticCacheManager`, `EmbeddingClient` | FR-006, NFR-006, NFR-011 |
| ADR-0113 | `TokenBudgeter` | FR-002, NFR-002 |
| ADR-0114 | `TokenCounter`, `ModelRouterClient` | FR-007, NFR-010 |
| ADR-0115 | `HierarchicalCompressor`, `ModelRouterClient` | FR-003, NFR-003, NFR-005 |
| ADR-0116 | `EvictionManager`, `SemanticCacheManager` | FR-008, NFR-006 |
| ADR-0117 | `BundleStore` | FR-013 |
| ADR-0118 | `ContextAssembler`, `SelectiveRetriever` | FR-010, NFR-008 |
| ADR-0119 | `SemanticCacheManager`, `ContextPolicyGuard` | NFR-011, NFR-014 |

---

## 4. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `ADR-011N-<slug>.md` em `../002-ADR/` usando
   `../002-ADR/TEMPLATE.md`, preenchendo todas as seções obrigatórias
   (Contexto, Problema, Alternativas, Análise, Escolha, Consequências, Riscos,
   Trade-offs).
2. O conteúdo **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste módulo nem
   as decisões fundacionais `ADR-0001`–`ADR-0010` (em especial `ADR-0007`,
   cuja fronteira com o `010-Memory` este módulo respeita sem redecidir); caso
   a análise revele necessidade de divergência, o brief DEVE ser atualizado
   primeiro (o brief é a fonte única de verdade).
3. Revisão pelo Arquiteto-Chefe e pelos módulos afetados listados na §2, com
   atenção especial a `010-Memory` (fonte do `recall` seletivo consumido por
   `SelectiveRetriever`), `017-Model-Router` (limites de janela, tokenizer,
   embedding e modelo de sumarização) e `022-Policy`/`021-Security` (PEP/PDP,
   isolamento de cache por tenant).
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da ADR e neste
   índice; o `../002-ADR/README.md` global é atualizado com a nova linha.
5. ADRs aceitas **NÃO SÃO** editadas — mudanças de rumo geram uma nova ADR que
   marca a anterior como `Superseded by ADR-XXXX`. A faixa `ADR-0110`–`ADR-0119`
   está integralmente reservada por este batch; qualquer decisão adicional do
   módulo que não caiba em nenhuma das dez ADRs listadas DEVE primeiro revisar
   o escopo do `_DESIGN_BRIEF.md` §11 antes de solicitar numeração fora da
   faixa.

---

## 5. Referências

- Índice global de ADRs: `../002-ADR/README.md`
- Template de ADR: `../002-ADR/TEMPLATE.md`
- Decisão fundacional de memória (fronteira respeitada): `../002-ADR/ADR-0007-Memoria-Hierarquica.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §11
- Arquitetura do módulo: `./Architecture.md`
- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Índice de RFCs do módulo: `./RFC.md`

*Fim do índice de ADRs do módulo 011-Context.*
