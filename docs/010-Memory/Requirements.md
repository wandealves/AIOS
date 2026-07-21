---
Documento: Requirements
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007, ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Requisitos (Índice e Rastreabilidade)

> Documento de consolidação. Não introduz requisitos novos: indexa os `FR-NNN`
> (`./FunctionalRequirements.md`) e `NFR-NNN` (`./NonFunctionalRequirements.md`),
> ambos derivados de `./_DESIGN_BRIEF.md` §7, e fornece a **matriz de rastreabilidade**
> `Requisito → UseCase → Teste`. Termos vêm de `../040-Glossary/Glossary.md` — sem
> redefinir. Palavras normativas conforme RFC 2119/8174.

## 1. Stakeholders

| Stakeholder | Interesse | Requisitos-chave |
|-------------|-----------|------------------|
| Agent Runtime (007) | Lembrar/recuperar via API, sem acessar datastore | FR-001, FR-002, NFR-001, NFR-003 |
| Context Manager (011) | Recall seletivo para a janela de contexto | FR-002, FR-013, NFR-002 |
| Learning (023) | Consolidar sem regredir; rollback | FR-005, FR-007, FR-014, NFR-011 |
| Cost-Optimizer (026) | Cotas, consumo e poda | FR-008, NFR-009 |
| Security/Audit (021/025) | RTBF auditável, isolamento | FR-011, NFR-010 |
| Model Router (017) | Fonte única de embeddings | FR-010 |
| Operador de plataforma | Config por tenant, SLOs, disponibilidade | NFR-004..NFR-008, NFR-012, NFR-013 |

## 2. Índice de Requisitos Funcionais (resumo)

Fonte canônica e detalhamento: `./FunctionalRequirements.md`.

| ID | Prioridade | Requisito (resumo) |
|----|-----------|--------------------|
| FR-001 | Must | `remember(item, layer)` persiste e retorna URN + estado. |
| FR-002 | Must | `recall(query, filtros)` retorna lista rankeada. |
| FR-003 | Must | Busca ANN `pgvector`/HNSW em Semantic/Episodic. |
| FR-004 | Must | Roteamento automático para a camada correta. |
| FR-005 | Must | `consolidate()` promove itens de forma versionada. |
| FR-006 | Must | `forget(policy)` aplica ttl/decay/lru/quota/rtbf. |
| FR-007 | Must | Rollback de consolidação restaura versão anterior. |
| FR-008 | Must | Cotas por (tenant, agente, camada) + backpressure. |
| FR-009 | Must | Externalização de blobs grandes para MinIO. |
| FR-010 | Must | Embeddings exclusivamente via Model Router (017). |
| FR-011 | Must | Direito ao esquecimento (RTBF) rastreável. |
| FR-012 | Should | Reidratação de itens `ARCHIVED` sob recall. |
| FR-013 | Should | Recall por travessia de grafo (Apache AGE). |
| FR-014 | Could | Deduplicação semântica na consolidação. |

## 3. Índice de Requisitos Não-Funcionais (resumo)

Fonte canônica e detalhamento: `./NonFunctionalRequirements.md`.

| ID | Atributo | Meta (resumo) |
|----|----------|---------------|
| NFR-001 | Latência `remember` | p99 ≤ 30 ms (inline) / ≤ 120 ms (embedding). |
| NFR-002 | Recall Rate | ≥ 0,95; Recall@10 ANN ≥ 0,95. |
| NFR-003 | Latência `recall` | p99 ≤ 50/150/250 ms (hot/ANN/grafo). |
| NFR-004 | Throughput | ≥ 5.000 remember/s; ≥ 2.000 recall/s por shard. |
| NFR-005 | Disponibilidade | ≥ 99,95%. |
| NFR-006 | Durabilidade | RPO ≤ 5 min (camadas duráveis). |
| NFR-007 | RTO | ≤ 15 min. |
| NFR-008 | Working Memory | efêmera; perda tolerada. |
| NFR-009 | Crescimento | uso < 100% da cota (`usage_ratio` < 1,0). |
| NFR-010 | Isolamento | zero vazamento cross-tenant. |
| NFR-011 | Consolidação sem regressão | Recall pós ≥ pré − 0,01. |
| NFR-012 | Escala vetorial | ≥ 10^8 vetores/tenant, p99 ANN ≤ 150 ms. |
| NFR-013 | Observabilidade | 100% ops com trace OTel + correlação. |

## 4. Matriz de rastreabilidade `Requisito → UseCase → Teste`

> Os `UC-NNN` são definidos em `./UseCases.md` e os casos de teste em `./Testing.md`.
> Esta matriz é a fonte de verdade da cobertura: todo FR/NFR **DEVE** mapear para ao
> menos um caso de uso e um teste. Se um `UC` desta matriz não existir em
> `./UseCases.md`, isso é defeito de documentação e bloqueia merge.

| Requisito | UseCase(s) | Teste(s) | Componente responsável |
|-----------|-----------|----------|------------------------|
| FR-001 | UC-001 (remember) | TC-remember-persist, TC-remember-idem | MemoryApiFacade, LayerRouter |
| FR-002 | UC-002 (recall) | TC-recall-rank, TC-recall-filters | RecallEngine |
| FR-003 | UC-002, UC-003 (recall ANN) | TC-ann-recall@10 | SemanticMemoryStore, EpisodicMemoryStore |
| FR-004 | UC-001 | TC-route-layer, TC-route-incompat (`AIOS-MEM-0020`) | LayerRouter |
| FR-005 | UC-004 (consolidate) | TC-consolidate-versioned | ConsolidationEngine, ConsolidationVersionManager |
| FR-006 | UC-005 (forget) | TC-forget-policies, TC-legal-hold (`AIOS-MEM-0050`) | ForgettingEngine |
| FR-007 | UC-006 (rollback) | TC-rollback-restore, TC-rollback-invalid (`AIOS-MEM-0042`) | ConsolidationVersionManager |
| FR-008 | UC-007 (cota) | TC-quota-deny (`AIOS-MEM-0010/0011`) | QuotaManager |
| FR-009 | UC-001 | TC-blob-externalize | BlobStoreAdapter |
| FR-010 | UC-001 | TC-embedding-via-017, TC-embed-fail (`AIOS-MEM-0030`) | EmbeddingClient |
| FR-011 | UC-008 (RTBF) | TC-rtbf-purge, TC-rtbf-audit | ForgettingEngine, OutboxPublisher |
| FR-012 | UC-002 | TC-rehydrate-archived | LayerStores, BlobStoreAdapter |
| FR-013 | UC-003 | TC-graph-traversal (`mode=graph`) | KnowledgeGraphAdapter |
| FR-014 | UC-004 | TC-dedup-merge (`parent_ids`) | ConsolidationEngine |
| NFR-001 | UC-001 | TC-load-remember-latency | MemoryTelemetry |
| NFR-002 | UC-002, UC-004 | TC-bench-recall-rate | RecallEngine |
| NFR-003 | UC-002 | TC-load-recall-latency | RecallEngine |
| NFR-004 | UC-001, UC-002 | TC-load-throughput | Domínio de Memória |
| NFR-005 | — (operacional) | TC-availability-drill | Serviço |
| NFR-006 | UC-004 | TC-dr-rpo | LongTermMemoryStore, SemanticMemoryStore |
| NFR-007 | — (operacional) | TC-failover-rto | Serviço |
| NFR-008 | UC-001 | TC-working-kill | WorkingMemoryStore |
| NFR-009 | UC-007 | TC-usage-ratio | QuotaManager, ForgettingEngine |
| NFR-010 | UC-002, UC-008 | TC-rls-fuzz-tenant (`AIOS-MEM-0005`) | MemoryPep, todos os Stores |
| NFR-011 | UC-004, UC-006 | TC-ab-recall-regression | ConsolidationEngine |
| NFR-012 | UC-002 | TC-bench-vector-scale | SemanticMemoryStore, EpisodicMemoryStore |
| NFR-013 | todos | TC-span-audit | MemoryTelemetry |

## 5. Glossário local (referências, sem redefinição)

Todos os termos abaixo são definidos em `../040-Glossary/Glossary.md`; este documento
apenas aponta onde cada um aparece.

| Termo | Onde é usado neste módulo |
|-------|---------------------------|
| Working / Short-Term / Long-Term / Semantic / Procedural / Episodic Memory | Camadas físicas (brief §3.3) |
| Memory Recall Rate | Métrica-norte (NFR-002) |
| Forgetting (Esquecimento controlado) | FR-006, ForgettingEngine |
| pgvector | Busca ANN (FR-003, NFR-012) |
| Quota (Cota) | FR-008, NFR-009 |
| Backpressure | FR-008, `AIOS-MEM-0011` |
| ACB / Agent Runtime / Control Plane / Data Plane | Fronteira Runtime↔Memory (`./Architecture.md` §6) |

Contratos centrais (URN, envelope de evento, envelope de erro, idempotência,
correlação, subjects) são da RFC-0001 (`../003-RFC/RFC-0001-Architecture-Baseline.md`)
e **não** são redefinidos aqui.

## 6. Riscos de requisitos e lacunas

| Item | Observação | Encaminhamento |
|------|------------|----------------|
| Alinhamento de `UC-NNN` | A matriz §4 assume os `UC` de `./UseCases.md`; o autor daquele doc **DEVE** manter os mesmos IDs. | Sincronizar na revisão. |
| Metas numéricas | Valores de SLO/SLI **DEVEM** ser idênticos em Monitoring/Benchmark/FailureRecovery/Configuration/Scalability. | Auditoria de consistência (`../033-DeveloperGuide/DocumentationCI.md`). |
| Novos termos | Nenhum termo novo foi criado; caso surja, sinalizar para o `../040-Glossary/Glossary.md`. | Curadoria do glossário. |
