---
Documento: Testing
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability, 025-Audit, 021-Security, 005-Database, 040-Glossary
---

# AIOS — Módulo 011 · Context — Testing

> Este documento especifica a **estratégia de testes** do `ContextService`,
> ligando cada `FR-NNN`/`NFR-NNN`/`UC-NNN` a uma suíte verificável. Os IDs de
> requisito e as metas numéricas (SLO/SLI) são **idênticos** aos fixados em
> [`FunctionalRequirements.md`](./FunctionalRequirements.md),
> [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md) e no
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) — divergência é defeito deste
> documento. Testes de carga/desempenho detalhados (metodologia, cargas,
> resultados-alvo) vivem em [Benchmark.md](./Benchmark.md); este documento
> cobre a **pirâmide de testes funcionais/de qualidade** e referencia o
> benchmark como o gate de desempenho. Palavras normativas conforme RFC
> 2119/8174.

---

## 1. Objetivo e estratégia geral

O `011-Context` **DEVE** ser validado por uma pirâmide de testes que garanta,
antes de qualquer release, que:

1. `tokens_in ≤ token_budget` em 100% dos bundles `SERVED` (FR-001) —
   **nunca** um teste opcional.
2. A **Context Compression Ratio** (NFR-004 ≥ 0,50) e a Δ **Task Completion
   Rate** (NFR-005 ≥ −2% absoluto) são reportadas **juntas** como *gate* de
   qualidade — nunca uma sem a outra ([NonFunctionalRequirements.md](./NonFunctionalRequirements.md)
   §4).
3. Nenhuma degradação, falha de dependência ou condição de borda produz um
   bundle acima do orçamento, uma resposta cross-tenant, ou uma falha
   silenciosa (FR-010, FR-011, NFR-014).

```
                         ┌────────────────────────────┐
                         │   Chaos / Degradação (§9)   │  poucos, caros, cenários
                         └─────────────┬──────────────┘
                     ┌──────────────────┴───────────────────┐
                     │   E2E (§7) — fluxos UC-001..UC-011      │  via API real
                     └─────────────────────┬────────────────────┘
                ┌────────────────────────────┴─────────────────────────────┐
                │  Integração / Contrato (§5, §6) — deps reais em container  │
                └──────────────────────────────┬─────────────────────────────┘
   ┌──────────────────────────────────────────────┴──────────────────────────────────────────────┐
   │                       Unitários (§4) — componentes isolados, muitos, rápidos                   │
   └───────────────────────────────────────────────────────────────────────────────────────────────┘
        Qualidade/Gate (§8) e Segurança (§10) atravessam todas as camadas, não são uma "camada extra".
```

---

## 2. Ambientes de teste

| Ambiente | Escopo | Dependências |
|----------|--------|--------------|
| **Local/CI — unitário** | Componentes isolados (`TokenBudgeter`, `RelevanceRanker`, `RedundancyEliminator`, `HierarchicalCompressor`, `EvictionManager`) | Nenhuma — *fakes*/*mocks* em memória. |
| **CI — integração** | `ContextService` completo + backends reais via *testcontainers* (Redis, PostgreSQL+`pgvector`, MinIO, NATS/JetStream) | `MemoryClient`/`ModelRouterClient` como *stubs* configuráveis (latência, erro, *timeout* injetáveis). |
| **Staging — E2E/contrato** | `ContextService` + `010-Memory`/`017-Model-Router`/`022-Policy` reais (ambiente compartilhado) | Rede/mTLS reais; dados sintéticos multi-tenant. |
| **Staging — chaos/degradação** | Injeção de falha controlada (kill de container, *circuit breaker*, latência artificial) | Mesmo ambiente de staging, isolado por tenant sintético. |
| **Staging — benchmark/carga** | Perfil dimensionado ~staging/produção | Ver [Benchmark.md](./Benchmark.md) §3 (ambiente dedicado, isolado de outros testes). |

---

## 3. Matriz de rastreabilidade Requisito → Teste

> Reproduz e **estende** [`Requirements.md`](./Requirements.md) §4 (que já
> fixa os nomes de suíte por `FR`); aqui a matriz é completada com os `NFR`
> ainda não cobertos naquela tabela e com o tipo de teste. Nomes de suíte são
> **canônicos** — reutilizados literalmente em CI.

### 3.1 Requisitos Funcionais

| FR | Caso(s) de Uso | Suíte de teste | Tipo |
|----|------------------|------------------|------|
| FR-001 | UC-001, UC-009 | `test_assemble_budget_invariant` | integração |
| FR-002 | UC-001, UC-008 | `test_budget_profile_allocation` | unitário |
| FR-003 | UC-001, UC-003 | `test_hierarchical_compression_ratio` | integração + qualidade |
| FR-004 | UC-001, UC-003 | `test_redundancy_dedup_no_loss` | unitário |
| FR-005 | UC-001 | `test_selective_retrieval_top_k` | integração |
| FR-006 | UC-002, UC-004 | `test_semantic_cache_hit_miss` | integração |
| FR-007 | UC-001, UC-007 | `test_token_counter_accuracy` | unitário + qualidade |
| FR-008 | UC-005, UC-006, UC-010 | `test_cache_invalidation_latency` | integração |
| FR-009 | UC-001, UC-002 | `test_event_emission_coverage` | integração |
| FR-010 | UC-009 | `test_graceful_degradation_010_017_down` | chaos |
| FR-011 | UC-001, UC-002, UC-004, UC-008 | `test_idempotency_replay` | contrato |
| FR-012 | UC-006, UC-011 | `test_rtbf_purge_end_to_end` | e2e |
| FR-013 | UC-001 (A2) | `test_fragment_offload_minio` | integração |
| FR-014 | UC-001 (A3), UC-003 (A3) | `test_summary_node_reuse` | integração |

### 3.2 Requisitos Não-Funcionais

| NFR | Meta | Suíte de teste | Tipo |
|-----|------|------------------|------|
| NFR-001 | `CacheLookup` p99 ≤ 20 ms | `bench_cache_lookup_latency` | benchmark ([Benchmark.md](./Benchmark.md)) |
| NFR-002 | `Assemble` p99 ≤ 150 ms (miss, sem sumarização) | `bench_assemble_latency_miss` | benchmark |
| NFR-003 | `Assemble` p99 ≤ 1200 ms (miss, com sumarização) | `bench_assemble_latency_summarize` | benchmark |
| NFR-004 | Compression Ratio ≥ 0,50 | `test_hierarchical_compression_ratio` (gate) | qualidade |
| NFR-005 | Δ Task Completion Rate ≥ −2% | `test_task_completion_delta_gate` | qualidade (`035-Benchmark`) |
| NFR-006 | Cache hit ratio ≥ 0,35 (regime permanente) | `bench_cache_hit_ratio_steady_state` | benchmark |
| NFR-007 | Custo evitado / bruto ≥ 0,25 | `test_cost_saved_ratio` | qualidade/benchmark |
| NFR-008 | Disponibilidade ≥ 99,95%/mês | `test_availability_probe_slo` | chaos + monitoramento |
| NFR-009 | Throughput ≥ 500 req/s/réplica | `bench_assemble_throughput` | benchmark |
| NFR-010 | Erro de estimativa de tokens ≤ 2% | `test_token_counter_accuracy` | unitário + qualidade |
| NFR-011 | Falso-*hit* de cache ≤ 0,5% | `test_semantic_cache_false_hit_rate` | qualidade (amostragem) |
| NFR-012 | RTO ≤ 5 min / RPO ≤ 60 s | `test_chaos_dependency_kill`, `test_outbox_atomicity` | chaos |
| NFR-013 | Cobertura de traces = 100% em caminhos críticos | `test_trace_coverage_critical_paths` | integração |
| NFR-014 | Vazamento de cache entre tenants = 0 | `test_rls_tenant_isolation_fuzz` | segurança |

---

## 4. Testes unitários

Cobrem lógica pura de componente, sem I/O de rede — rápidos (< 100 ms cada),
executados em todo *commit*.

| Componente | Casos de borda obrigatórios |
|------------|-------------------------------|
| `TokenBudgeter` | Soma `alloc_*` = 1,0 exata e com erro de arredondamento `1e-6`; violação retorna `AIOS-CTX-0008`; resolução de escopo `agent > task_type > tenant > global` (`test_budget_profile_allocation`). |
| `TokenCounter` | Erro de estimativa ≤ 2% em corpus multi-idioma; fallback quando tokenizer exato indisponível sinaliza `approximate=true` (`test_token_counter_accuracy`). |
| `RelevanceRanker` | Ordenação estável por (similaridade, recência, prioridade); empates determinísticos. |
| `RedundancyEliminator` | Colapso apenas quando `cosine ≥ context.dedup.cosine_threshold`; preserva item único mesmo acima do limiar quando é a única cobertura de uma entidade (`test_redundancy_dedup_no_loss`). |
| `HierarchicalCompressor` | Respeita `max_levels`; preserva `relevance_score ≥ p90`; aciona fallback `truncate` em erro simulado do `SummarizationClient`. |
| `EvictionManager` | TTL, LRU semântico e evento disparam remoção corretamente e de forma independente; nenhuma dupla contabilização de causa. |
| `ContextPolicyGuard` | *Fast path* de leitura do próprio escopo aplicado apenas quando `agent == owner`; toda operação privilegiada delega ao PDP simulado. |

---

## 5. Testes de contrato

Validam a superfície pública (REST/gRPC) e o barramento de eventos contra os
schemas normativos, sem depender de lógica de negócio profunda.

| Suíte | Cobre |
|-------|-------|
| `test_openapi_schema_conformance` | Toda rota de [API.md](./API.md) §5.1 responde conforme o *schema* OpenAPI publicado; erros seguem o envelope RFC 7807 (RFC-0001 §7) com `code = AIOS-CTX-<NNNN>`. |
| `test_grpc_proto_conformance` | Serviço `aios.context.v1.ContextService` ([API.md](./API.md) §5.2) compatível com o `.proto` versionado; nenhum campo obrigatório removido entre `v1` e sucessoras sem migração *expand/contract*. |
| `test_idempotency_replay` | Repetição de `assemble`/`cache store`/`budget upsert` com a **mesma** `Idempotency-Key` retorna o mesmo resultado por ≥ 24 h (FR-011); payload divergente com a mesma chave retorna `AIOS-CTX-0012` (409). |
| `test_cloudevents_schema_conformance` | Todo evento emitido ([Events.md](./Events.md) §5) valida contra o `dataschema` versionado referenciado; consumidores toleram campos desconhecidos (evolução aditiva, RFC-0001 §5.7). |
| `test_error_envelope_no_pii` | Nenhum `detail` de erro (`AIOS-CTX-<NNNN>`) contém texto de fragmento/prompt (RFC-0001 §6, brief §12.3). |

---

## 6. Testes de integração

Executam o `ContextService` completo contra backends reais em *testcontainers*
(Redis, PostgreSQL+`pgvector`, MinIO, NATS/JetStream), com `MemoryClient`/
`ModelRouterClient` como *stubs* controláveis.

| Suíte | Cenário |
|-------|---------|
| `test_assemble_budget_invariant` | Para uma matriz de `model_id`/`token_budget`/tamanho de contexto candidato, `tokens_in ≤ token_budget` em 100% das amostras `SERVED`; contexto mínimo excedente gera `AIOS-CTX-0001`. |
| `test_selective_retrieval_top_k` | `MemoryClient` *stub* retorna N candidatos > `top_k`; verifica truncamento antes do ranking e respeito a `memory_deadline_ms`. |
| `test_semantic_cache_hit_miss` | Sequência *miss* → *store* → *hit* (fast path `prompt_hash` e slow path `prompt_embedding`); `cache_outcome` refletido corretamente no bundle. |
| `test_cache_invalidation_latency` | Publica `memory.item.consolidated`/`deleted` sintético; mede tempo até `context.cache.invalidated` (**DEVE** ser ≤ 5 s, FR-008). |
| `test_event_emission_coverage` | Para cada transição terminal (`SERVED`/`REJECTED`/`FAILED`), exatamente um evento correspondente é publicado via outbox (FR-009). |
| `test_fragment_offload_minio` | Fragmento sintético acima/abaixo de `max_inline_bytes`; verifica `content_ref` vs. `content_inline` e reidratação transparente na leitura (FR-013). |
| `test_summary_node_reuse` | `SummaryNode` existente com `similarity ≥ reuse_threshold` e não expirado é reaproveitado em vez de nova chamada de sumarização (FR-014). |
| `test_trace_coverage_critical_paths` | 100% dos *spans* esperados (`assemble`, `cache_lookup`, `cache_store`, `compress`) presentes com `trace_id` propagado (NFR-013). |
| `test_outbox_atomicity` | Falha simulada de publicação NATS pós-commit; verifica que o evento permanece `pending` e é reenviado pelo `EventPublisher` no próximo ciclo (RPO ≤ 60 s, FM-07). |

---

## 7. Testes end-to-end (fluxos `UC-NNN`)

Executados em *staging* contra dependências reais (`010`, `017`, `022`),
cobrindo cada caso de uso de [UseCases.md](./UseCases.md) do início ao fim,
incluindo fluxos alternativos e de exceção documentados por caso de uso.

| UC | Fluxo coberto | Suíte |
|----|-----------------|-------|
| UC-001 | `assemble` completo, cache *miss*, compressão, persistência, outbox | `e2e_assemble_cold_path` |
| UC-002 | `assemble` servido por cache semântico (*hit*) | `e2e_assemble_cache_hit` |
| UC-003 | `compress` avulso com reuso de `SummaryNode` | `e2e_compress_standalone` |
| UC-004 | `cache store` seguido de *hit* subsequente | `e2e_cache_store_then_hit` |
| UC-005 | Invalidação por evento de mudança de origem | `e2e_cache_invalidation_event_driven` |
| UC-006 | Invalidação manual por escopo (`cache/invalidate`) | `e2e_cache_invalidate_manual` |
| UC-007 | `tokens/estimate` | `e2e_tokens_estimate` |
| UC-008 | Leitura/atualização de `BudgetProfile` | `e2e_budget_profile_upsert` |
| UC-009 | Degradação graciosa com `010`/`017` indisponíveis | `test_graceful_degradation_010_017_down` (chaos, §9) |
| UC-010 | Reação a `agent.lifecycle.suspended/terminated` | `e2e_agent_lifecycle_reaction` |
| UC-011 | Expurgo RTBF (por chamada manual e por evento a montante) | `test_rtbf_purge_end_to_end` |

---

## 8. Testes de qualidade (*gate* de compressão e cache)

> Estes testes são o **coração** da estratégia de qualidade do módulo — sua
> missão (brief §1.1) só é cumprida se compressão/cache **não degradam** a
> tarefa. Nenhuma release avança sem passar por este *gate*.

| Suíte | O que mede | Critério de aprovação |
|-------|------------|--------------------------|
| `test_hierarchical_compression_ratio` | **Context Compression Ratio** média em corpus de referência multi-tarefa. | ≥ 0,50 (NFR-004). |
| `test_task_completion_delta_gate` | Δ **Task Completion Rate** entre execução com compressão habilitada vs. baseline sem compressão, no conjunto de tarefas golden do `035-Benchmark`. | ≥ −2 pontos percentuais absolutos (NFR-005); reportado **junto** com a linha acima — falha se qualquer uma reprovar isoladamente. |
| `test_semantic_cache_false_hit_rate` | Fração de *hits* servidos que um avaliador (LLM-juiz calibrado ou anotação humana por amostragem) classifica como semanticamente inadequados. | ≤ 0,5% (NFR-011); violação sustentada aciona FM-08 ([FailureRecovery.md](./FailureRecovery.md)). |
| `test_cost_saved_ratio` | Custo evitado por *hit* de cache sobre custo bruto estimado, no mesmo corpus de referência. | ≥ 0,25 (NFR-007). |
| `test_redundancy_dedup_no_loss` (qualidade) | Taxa de "colapso falso" (perda de item semanticamente único) em corpus rotulado de *near-duplicates*. | Taxa de colapso falso reportada; **zero** tolerado para itens marcados como cobertura única no corpus de referência (FR-004). |

```
 corpus golden (tarefas rotuladas, 035-Benchmark)
        │
   ┌────▼──────────────┐        ┌──────────────────────┐
   │ baseline (sem      │        │ Context (com          │
   │ compressão/cache)  │        │ compressão + cache)   │
   └────┬──────────────┘        └──────────┬────────────┘
        │ Task Completion Rate baseline    │ Task Completion Rate + Compression Ratio
        └──────────────────┬───────────────┘
                            ▼
              Δ Task Completion Rate = rate_context − rate_baseline
              GATE: Δ ≥ −0.02  E  compression_ratio ≥ 0.50
```

---

## 9. Testes de caos e degradação

Executados em *staging* com injeção de falha controlada, cobrindo os modos
`FM-01`..`FM-14` de [FailureRecovery.md](./FailureRecovery.md).

| Suíte | Falha injetada | Verificação |
|-------|-------------------|-------------|
| `test_graceful_degradation_010_017_down` | Desliga `010-Memory` e/ou `017-Model-Router` durante `assemble` em curso | `Assemble` retorna `200` com `degraded=true` (nunca falha silenciosa), estado `DEGRADED` (FR-010, FM-01/FM-02). |
| `test_chaos_dependency_kill` | *Kill* de réplica do `context-service` sob carga | RTO ≤ 5 min; nenhuma perda de dado durável (RPO ≤ 60 s, NFR-012). |
| `test_backoff_circuit_breaker` | Falhas sequenciais simuladas em `ModelRouterClient`/`MemoryClient` | Circuito abre no limiar configurado (§4.1 de [FailureRecovery.md](./FailureRecovery.md)); *half-open* após 10 s; nenhum *retry* após esgotar tentativas. |
| `test_cache_backend_bypass` | Redis e/ou `pgvector` indisponíveis | `cache_outcome=BYPASS`; `assemble` segue pelo caminho `MISS` sem bloquear (FM-03). |
| `test_dlq_replay` | Consumidor de eventos falha 5x consecutivas | Mensagem migrada para DLQ; *replay* pós-correção não duplica efeito (dedup por `event.id`). |
| `test_outbox_atomicity` | Falha de publicação NATS pós-commit | Evento permanece `pending`, reenviado no próximo ciclo do `EventPublisher` (FM-07). |

---

## 10. Testes de segurança e isolamento

| Suíte | Cobre |
|-------|-------|
| `test_rls_tenant_isolation_fuzz` | *Fuzzing* de `tenant_id`/`X-AIOS-Tenant` divergente; nenhuma linha de outro tenant é retornável via RLS; zero vazamento de cache cross-tenant (NFR-014). |
| `test_authz_default_deny` | Operações privilegiadas (assemble para outro agente, `budget upsert`, `cache invalidate` em massa) sem *capability* retornam 403 (`AIOS-CTX-0011` ou negação do PDP). |
| `test_mtls_enforced` | Conexão interna sem certificado mTLS válido é recusada (Security.md §1.1). |
| `test_logging_no_pii_leak` | *Scan* de amostra de log estruturado não encontra `content_inline`/`response_ref`/texto de prompt fora da lista branca ([Logging.md](./Logging.md) §5). |
| `test_secrets_not_in_config_namespace` | Nenhuma chave `context.*`/`AIOS_CONTEXT_*` carrega segredo (DSN, credencial) — apenas *default*/faixa/escopo ([Configuration.md](./Configuration.md)). |

---

## 11. Cobertura-alvo e critérios de *gate*

| Métrica de teste | Alvo |
|-------------------|------|
| Cobertura de linha — componentes de domínio (`TokenBudgeter`, `RelevanceRanker`, `RedundancyEliminator`, `HierarchicalCompressor`, `EvictionManager`) | ≥ 80% |
| Cobertura de branch — validação de `BudgetProfile`/invariantes | 100% dos caminhos de erro (`AIOS-CTX-0001`, `0008`, `0012`, `0013`, `0014`) |
| Cobertura de código de erro | 100% dos `AIOS-CTX-<NNNN>` catalogados ([API.md](./API.md) §5.3) têm ao menos um teste de contrato ou integração dedicado. |
| Cobertura de evento | 100% dos eventos emitidos/consumidos ([Events.md](./Events.md) §3/§4) têm teste de emissão ou de reação. |
| Cobertura de transição de estado | Toda transição de [StateMachine.md](./StateMachine.md) (`ContextBundle` e `SemanticCacheEntry`) exercida por ao menos um teste de integração ou E2E. |

### 11.1 *Pipeline* de *gate* de CI/CD (ASCII)

```
 commit/PR
    │
    ▼
 ┌─────────────┐   falha   ┌───────────────┐
 │ unit + lint │──────────▶│ bloqueia merge │
 └──────┬──────┘           └───────────────┘
    │ ok
    ▼
 ┌───────────────────────┐   falha   ┌───────────────┐
 │ contrato + integração  │──────────▶│ bloqueia merge │
 └──────┬─────────────────┘           └───────────────┘
    │ ok
    ▼
 ┌───────────────────────┐   falha   ┌────────────────────────┐
 │ qualidade (§8, gate)   │──────────▶│ bloqueia deploy staging │
 │ compression + Δ TCR    │           │ (não bloqueia merge)    │
 └──────┬─────────────────┘           └────────────────────────┘
    │ ok
    ▼
 ┌───────────────────────┐   falha   ┌────────────────────────┐
 │ e2e + chaos (staging)  │──────────▶│ bloqueia promoção a prod │
 └──────┬─────────────────┘           └────────────────────────┘
    │ ok
    ▼
 ┌───────────────────────┐   falha   ┌────────────────────────┐
 │ benchmark (Benchmark.md│──────────▶│ bloqueia promoção a prod │
 │ NFR-001/002/003/009)   │           │ (regressão de SLO)      │
 └──────┬─────────────────┘           └────────────────────────┘
    │ ok
    ▼
 promoção a produção (rollout conforme Deployment.md §6)
```

---

## 12. Fixtures e dados de teste

| Fixture | Uso | Observação |
|---------|-----|-------------|
| Corpus rotulado de *near-duplicates* | `test_redundancy_dedup_no_loss` | Pares/clusters de fragmentos sintéticos com anotação de "cobertura única" vs. "redundante". |
| Conjunto golden de tarefas (`035-Benchmark`) | `test_task_completion_delta_gate`, `test_hierarchical_compression_ratio` | Compartilhado com o benchmark de qualidade global; versionado e reprodutível ([Benchmark.md](./Benchmark.md) §9). |
| Corpus multi-idioma de contagem de tokens | `test_token_counter_accuracy` | Amostras por família de tokenizer suportada via `017`. |
| Dataset sintético de prompts/PII simulada | `test_logging_no_pii_leak`, `test_rtbf_purge_end_to_end` | PII **sintética**, nunca dado real de tenant — gerada para o teste. |
| *Stubs* de `MemoryClient`/`ModelRouterClient` | Testes de integração/chaos | Parametrizáveis por latência, taxa de erro e *payload* de resposta. |

---

## 13. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-------------------|--------------------------|
| Gate de qualidade (§8) caro/lento para rodar em todo PR. | Executado no *pipeline* de promoção a *staging*, não em todo *commit* (§11.1); unit/contrato/integração cobrem o essencial por *commit*. |
| Falso-*hit* de cache difícil de medir de forma determinística. | Combinação de amostragem/avaliação offline periódica com corpus golden versionado, reduzindo variância entre execuções ([Benchmark.md](./Benchmark.md)). |
| Testes de caos introduzem instabilidade no ambiente de *staging* compartilhado. | Isolamento por tenant sintético dedicado a testes de caos; execução em janela agendada, nunca concorrente com *e2e* de outros times. |
| Corpus golden desatualizado em relação a tarefas reais em produção. | Revisão periódica do corpus alinhada ao ciclo de release do `035-Benchmark`; amostragem de produção alimenta atualização do corpus (com anonimização). |
| **Alternativa descartada:** medir Δ Task Completion Rate apenas em produção (sem *gate* pré-release). | Rejeitada — vazaria regressões de qualidade a usuários antes da detecção; contraria o princípio de "gate de qualidade" citado em NFR-004/005. |
| **Alternativa descartada:** cobertura de linha como único critério de qualidade de teste. | Rejeitada — cobertura de branch/erro/evento/estado (§11) é mais representativa para um sistema orientado a estado e contrato como o `011`. |

---

## 14. Ver também

- [`Requirements.md`](./Requirements.md) §4 — matriz consolidada
  Requisito → Caso de Uso → Teste (fonte dos nomes de suíte por FR).
- [`FunctionalRequirements.md`](./FunctionalRequirements.md),
  [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md) — IDs e
  metas canônicas.
- [`UseCases.md`](./UseCases.md) — fluxos `UC-001..UC-011` detalhados.
- [`StateMachine.md`](./StateMachine.md) — transições exercidas pelos testes.
- [`Benchmark.md`](./Benchmark.md) — metodologia de carga/desempenho (gate
  final do pipeline, §11.1).
- [`FailureRecovery.md`](./FailureRecovery.md) — matriz FMEA usada em §9.
- [`Logging.md`](./Logging.md), [`Metrics.md`](./Metrics.md) — instrumentação
  verificada por `test_trace_coverage_critical_paths`/`test_logging_no_pii_leak`.
- Glossário: [Idempotency, Row-Level Security, Semantic Cache](../040-Glossary/Glossary.md).
