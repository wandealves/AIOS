---
Documento: Requirements
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability, 025-Audit, 021-Security, 020-Communication, 005-Database, 040-Glossary
---

# AIOS — Módulo 011 · Context — Requisitos (Índice Consolidado)

> Este documento é o **índice consolidado** de requisitos do `011-Context`.
> Ele **não redefine** requisitos: consolida, rastreia e contextualiza o que já
> está fixado em [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) §7,
> [`FunctionalRequirements.md`](./FunctionalRequirements.md) e
> [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md). Onde houver
> qualquer divergência aparente, o brief prevalece. Palavras normativas conforme
> RFC 2119/8174 (**DEVE**, **NÃO DEVE**, **DEVERIA**, **PODE**). Contratos
> centrais (URN `urn:aios:<tenant>:<tipo>:<id>`, envelope CloudEvents, subjects
> `aios.<tenant>.<dominio>.<entidade>.<acao>`, envelope de erro RFC 7807
> `AIOS-<DOMINIO>-<NNNN>`, `Idempotency-Key`, correlação
> `traceparent`/`X-AIOS-Tenant`) vêm de
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) e são apenas
> referenciados, nunca redefinidos.

## 1. Propósito e escopo dos requisitos

O `011-Context` é o **gerenciador de janela de contexto** do AIOS: monta o
`ContextBundle` ótimo por chamada de LLM dentro do `token_budget` do modelo
alvo, por meio de **compressão/sumarização hierárquica**, **remoção de
redundância**, **recuperação seletiva** junto ao `010-Memory` e **cache
semântico** (Redis + `pgvector`), minimizando tokens e reinferências enquanto
preserva a **Task Completion Rate**. Este documento cobre:

- Requisitos **funcionais** (`FR-001..FR-014`, detalhados em
  [`FunctionalRequirements.md`](./FunctionalRequirements.md));
- Requisitos **não-funcionais** (`NFR-001..NFR-014`, detalhados em
  [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md));
- Requisitos **derivados de dependências** (contratos que o `011` consome de
  outros módulos e que restringem seu próprio comportamento);
- A **matriz de rastreabilidade** Requisito → Caso de Uso → Teste;
- Um **glossário local** que reutiliza (sem redefinir) o
  [Glossário canônico](../040-Glossary/Glossary.md).

### 1.1 Fora de escopo (ver "Não-responsabilidades", brief §1.3)

O `011` **NÃO DEVE** decidir qual modelo executa a tarefa (`017`), persistir
memória de longo prazo ou aplicar esquecimento (`010`), executar
ferramentas/hospedar o loop ReAct (`015`/`007`), manter o grafo de
conhecimento (`018`/`019`), definir políticas RBAC/ABAC (`022`, apenas
consome como PEP) ou ser a fonte da verdade de auditoria (`025`, apenas
emite eventos auditáveis).

## 2. Stakeholders

| Stakeholder | Papel | Interesse principal |
|-------------|-------|----------------------|
| **AgentRuntime (007)** | Consumidor primário da API | Receber contexto ótimo, dentro do orçamento, com baixa latência. |
| **Model-Router (017)** | Fonte de limites/tokenizer/embedding/sumarização | Contrato estável de limites de modelo e endpoints de suporte. |
| **Memory (010)** | Fonte de recall seletivo | Recall rápido e correto; consumidor de eventos de invalidação do `011`. |
| **Policy Engine (022)** | PDP | Decisões de autorização consistentes para operações privilegiadas. |
| **Observability (024)** | Consumidor de telemetria | Métricas/traces confiáveis para SLO e troubleshooting. |
| **Audit (025)** | Consumidor de eventos auditáveis | Trilha rastreável de invalidações/expurgos RTBF. |
| **Security (021)** | Dono de AuthN/mTLS/RTBF | Isolamento multi-tenant e conformidade LGPD/GDPR. |
| **Operador de Plataforma** | Configura `BudgetProfile` e observa métricas | Ajustar trade-off compressão/qualidade/custo por tenant. |
| **Cost Optimizer (026)** | Consumidor indireto (via métricas de custo evitado) | Reduzir **Reasoning Cost per Successful Task**. |
| **Product/Arquitetura AIOS** | Dono do backlog do módulo | Módulo entregue conforme brief, sem contradição. |

## 3. Glossário local (reutilizado de `../040-Glossary/Glossary.md`)

> Nenhum termo abaixo é redefinido — apenas reunido para leitura rápida deste
> módulo. Definições completas estão no Glossário canônico.

| Termo | Definição resumida | Módulo de origem |
|-------|---------------------|-------------------|
| **Context Window (Janela de Contexto)** | Limite de tokens que um modelo processa por chamada; recurso escasso gerenciado por este módulo. | `011` |
| **Context Compression Ratio** | Métrica: redução de tokens preservando desempenho da tarefa. `1 − tokens_in/tokens_raw`. | `011`, `024` |
| **Semantic Cache (Cache Semântico)** | Cache indexado por similaridade semântica que evita reinferências. | `011` |
| **Cognitive Resource (Recurso Cognitivo)** | Recurso gerenciado pelo AIOS específico de IA (memória, contexto, atenção, inferência, conhecimento). | `000`, `010`, `011` |
| **Embedding** | Vetor denso que representa semântica de um item; armazenado em `pgvector`. | `010`, `019` |
| **Idempotency (Idempotência)** | Propriedade de uma operação produzir o mesmo efeito se aplicada uma ou várias vezes; obrigatória em mutações. | `001`, `004` |
| **Outbox** | Padrão transacional que garante atomicidade entre escrita no banco e publicação de evento. | `001`, `020` |
| **PEP / PDP** | PEP intercepta e aplica; PDP decide. Base da governança por política. | `022` |
| **Row-Level Security (RLS)** | Mecanismo do PostgreSQL para isolamento por linha (por `tenant_id`). | `005`, `021` |
| **Task Completion Rate** | Métrica: % de tarefas concluídas com sucesso; usada para validar que a compressão não degrada a tarefa. | `024`, `035` |
| **Hot State (Estado quente)** | Estado acessado com alta frequência, mantido em Redis para latência sub-ms (usado pelo cache L1). | `001`, `010` |
| **Multi-tenancy** | Capacidade de servir múltiplos tenants isolados na mesma plataforma. | `001`, `021` |

## 4. Matriz de rastreabilidade: Requisito → Caso de Uso → Teste

> A rastreabilidade completa por caso de uso está em
> [`UseCases.md`](./UseCases.md) §13; o detalhamento de cada FR está em
> [`FunctionalRequirements.md`](./FunctionalRequirements.md) §3. A coluna
> "Teste" referencia a suíte que valida o requisito em
> [`Testing.md`](./Testing.md) (a ser detalhada no lote de qualidade do
> módulo).

| Requisito | Caso(s) de Uso | Teste (suíte) | NFR relacionado |
|-----------|------------------|-----------------|-------------------|
| FR-001 — Assembly dentro do orçamento | UC-001, UC-009 | `test_assemble_budget_invariant` | NFR-002, NFR-003 |
| FR-002 — Alocação por seção (`BudgetProfile`) | UC-001, UC-008 | `test_budget_profile_allocation` | NFR-002 |
| FR-003 — Compressão hierárquica | UC-001, UC-003 | `test_hierarchical_compression_ratio` | NFR-004, NFR-005 |
| FR-004 — Remoção de redundância | UC-001, UC-003 | `test_redundancy_dedup_no_loss` | NFR-004 |
| FR-005 — Recuperação seletiva (010) | UC-001 | `test_selective_retrieval_top_k` | NFR-002 |
| FR-006 — Cache semântico | UC-002, UC-004 | `test_semantic_cache_hit_miss` | NFR-001, NFR-006, NFR-007, NFR-011 |
| FR-007 — Contagem/estimativa de tokens | UC-001, UC-007 | `test_token_counter_accuracy` | NFR-010 |
| FR-008 — Invalidação event-driven | UC-005, UC-006, UC-010 | `test_cache_invalidation_latency` | NFR-011 |
| FR-009 — Eventos e métricas | UC-001, UC-002 | `test_event_emission_coverage` | NFR-013 |
| FR-010 — Degradação graciosa | UC-009 | `test_graceful_degradation_010_017_down` | NFR-008, NFR-012 |
| FR-011 — Idempotência | UC-001, UC-002, UC-004, UC-008 | `test_idempotency_replay` | — |
| FR-012 — Expurgo LGPD | UC-006, UC-011 | `test_rtbf_purge_end_to_end` | NFR-014 |
| FR-013 — Offload MinIO | UC-001 (A2) | `test_fragment_offload_minio` | — |
| FR-014 — Reuso de `SummaryNode` | UC-001 (A3), UC-003 (A3) | `test_summary_node_reuse` | NFR-007 |

## 5. Requisitos derivados de dependências

O `011-Context` **consome** (nunca redefine) contratos de outros módulos; os
requisitos abaixo são restrições impostas por essas dependências e **DEVEM**
ser respeitados por qualquer implementação do módulo.

| Dependência | Requisito derivado imposto ao `011` | Consequência de violação |
|-------------|----------------------------------------|-----------------------------|
| `010-Memory` | O `011` **NÃO DEVE** acessar `LayerStores`/PostgreSQL de memória diretamente; toda recuperação passa por `MemoryClient` (recall via API/gRPC). | Acoplamento indevido; quebra de fronteira (brief §1.3, N-02). |
| `017-Model-Router` | O `011` **NÃO DEVE** chamar provedores de LLM diretamente para inferência final; limites de modelo, tokenizer, embedding e sumarização vêm exclusivamente de `017`. | Quebra de fronteira (N-01, N-03); inconsistência de custo/roteamento. |
| `022-Policy` | Toda operação privilegiada (assemble para outro agente, upsert de `BudgetProfile`, invalidação em massa) **DEVE** consultar o PDP antes de executar; postura *default deny*. | Vazamento de autorização; violação do modelo de segurança (brief §12). |
| `021-Security` | AuthN via OIDC/mTLS obrigatório; `tenant` do token **DEVE** igualar `X-AIOS-Tenant`. | `AIOS-CTX-0011`; risco de vazamento cross-tenant. |
| `024-Observability` | Toda operação **DEVE** emitir trace OTel com `tenant_id`/`trace_id`/`bundle_id` e métricas `aios_context_*`. | Perda de observabilidade; SLOs não verificáveis (NFR-013). |
| `025-Audit` | Invalidações/expurgos **DEVEM** emitir eventos consumíveis pelo Audit, com `actor` e `event.id`. | Ausência de trilha auditável (repúdio, brief §12.2). |
| `020-Communication` | Publicação de eventos **DEVE** seguir o envelope CloudEvents e subjects `aios.<tenant>.context.*`, via NATS/JetStream com semântica at-least-once + idempotência por `event.id`. | Inconsistência de barramento; consumidores não conseguem deduplicar. |
| `005-Database` | Todas as tabelas **DEVEM** ter `tenant_id` e Row-Level Security; uso de `pgvector` para embeddings. | Vazamento cross-tenant (NFR-014); inconsistência de modelo físico. |

## 6. Requisitos de dados e identidade (resumo)

O módulo introduz três novos `<tipo>` de URN a serem registrados em
`../004-API/Resources.md` (proposta formalizada na RFC-0011): **`context`**
(bundle), **`ctxfrag`** (fragmento) e **`ctxcache`** (entrada de cache). Toda
entidade **DEVE** portar `tenant_id` obrigatório com RLS. Detalhamento completo
do modelo de dados está em [`Database.md`](./Database.md) e
[`ClassDiagrams.md`](./ClassDiagrams.md); aqui apenas se registra a exigência
de rastreabilidade:

| Entidade | URN | Requisito relacionado |
|----------|-----|--------------------------|
| `ContextBundle` | `urn:aios:<tenant>:context:<ulid>` | FR-001, FR-002, FR-011, FR-013 |
| `ContextFragment` | `urn:aios:<tenant>:ctxfrag:<ulid>` | FR-003, FR-004, FR-005, FR-013 |
| `SemanticCacheEntry` | `urn:aios:<tenant>:ctxcache:<ulid>` | FR-006, FR-008, FR-011, FR-012 |
| `BudgetProfile` | `urn:aios:<tenant>:policy:<ulid>` (subtipo `context-budget`) | FR-002 |
| `SummaryNode` | (interno, sem URN público — cache de curta duração) | FR-003, FR-014 |

## 7. Não-objetivos explícitos de requisitos

- Este documento **NÃO DEVE** introduzir metas de SLO/SLI novas fora das já
  fixadas em [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md);
  qualquer meta numérica citada aqui é apenas referência cruzada.
- Este documento **NÃO DEVE** propor novos estados de máquina — estes
  pertencem a [`StateMachine.md`](./StateMachine.md), fiel ao brief §4.
- Este documento **NÃO DEVE** redefinir códigos de erro — o catálogo
  `AIOS-CTX-<NNNN>` vive no brief §5.3 e em [`API.md`](./API.md).

## 8. Riscos de requisitos em aberto

| Risco | Descrição | Mitigação proposta | Rastreado em |
|-------|-----------|----------------------|---------------|
| Trade-off compressão vs. qualidade mal calibrado por tenant | `BudgetProfile`/`min_compression_ratio` genéricos podem comprimir demais tarefas sensíveis. | Perfis por `task_type`; benchmark de qualidade como gate de release. | NFR-004, NFR-005; ADR-0110 |
| Falso-*hit* de cache silencioso | Erros de similaridade mal calibrada não são óbvios ao chamador. | Amostragem/eval offline periódica; `similarity_threshold` conservador por padrão (0,92). | NFR-011; ADR-0112, ADR-0119 |
| Contrato de tipos de URN ainda "proposto" (RFC-0011 não aprovada) | `context`/`ctxfrag`/`ctxcache` dependem de aprovação formal em `004-API/Resources.md`. | Tratar como *Draft* até aprovação; não bloquear implementação interna. | Brief §11; RFC-0011 |
| Dependência de `017` para tokenizer exato nem sempre disponível | Estimativa aproximada pode gerar `BudgetInfeasible` espúrio ou orçamento mal calculado. | Registro de tokenizers multi-provedor (ADR-0114); erro ≤ 2% monitorado. | NFR-010; ADR-0114 |
