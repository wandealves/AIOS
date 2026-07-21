---
Documento: Vision
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 022-Policy, 025-Audit, 020-Communication, 005-Database, 021-Security
---

# AIOS — Módulo 011 · Context — Vision

> Documento derivado da FONTE ÚNICA DE VERDADE `./_DESIGN_BRIEF.md` (§1). Termos
> canônicos vêm de `../040-Glossary/Glossary.md`; contratos centrais de
> `../003-RFC/RFC-0001-Architecture-Baseline.md`. Este documento **NÃO** redefine
> nenhum dos dois — apenas os referencia. Palavras normativas conforme RFC 2119/8174.

## 1. Propósito

O `ContextService` (módulo 011) é o **gerenciador de janela de contexto** do
AIOS — análogo cognitivo do subsistema de **paginação/cache de memória virtual**
de um sistema operacional clássico, porém aplicado ao recurso mais escasso e mais
caro de um agente de IA: os tokens que cabem em uma única chamada de LLM. Assim
como um gerenciador de paginação decide o que permanece residente em RAM, o que é
compactado/paginado para disco e o que é servido por cache antes de tocar o
dispositivo lento, o `011-Context` decide, a cada chamada de inferência, **o que
entra na janela agora**, **sob qual forma comprimida** e **o que pode ser servido
por cache semântico** sem sequer invocar o modelo.

O módulo materializa, para o **Context Window** (`../040-Glossary/Glossary.md`),
a promessa global do AIOS de ser o **gerenciador de recursos cognitivos** de
agentes de IA (`../000-Vision/Vision.md`). Ele expõe uma superfície estável
(`assemble`, `compress`, `cache lookup/store/invalidate`, `tokens estimate`,
`budgets get/upsert`) que esconde, de agentes e do Agent Runtime, a complexidade
de token budgeting, sumarização hierárquica, deduplicação e indexação vetorial de
cache — da mesma forma que o `010-Memory` esconde a heterogeneidade dos backends
de memória por trás de quatro verbos cognitivos.

## 2. Problema que resolve

Sem um gerenciador dedicado de janela de contexto, cada agente (ou o `017-Model-
Router`, ou o `007-Agent-Runtime`) reimplementaria — de forma inconsistente e sem
governança — a decisão de o que caber no prompt, quanto gastar recuperando
memória e quando reaproveitar uma resposta já computada. Os problemas centrais
que o módulo DEVE resolver são:

| Problema | Consequência sem o módulo | Como 011-Context resolve |
|----------|---------------------------|---------------------------|
| Orçamento de tokens não respeitado | Erros de "context length exceeded" no provedor de LLM, retrabalho, latência imprevisível | `TokenBudgeter` aloca por seção a partir do `token_budget` do modelo alvo (R-01, R-02) |
| Contexto bruto grande demais | Custo de inferência alto, latência alta, degradação de qualidade por diluição de sinal | Compressão/sumarização hierárquica map-reduce (R-03), meta `Context Compression Ratio ≥ 0,50` (NFR-004) |
| Fragmentos redundantes/near-duplicate | Tokens desperdiçados repetindo a mesma informação sob formas distintas | `RedundancyEliminator` colapsa near-duplicates por similaridade de *embedding* (R-04) |
| Recall de memória não seletivo | `010-Memory` devolveria tudo o que é "relevante", estourando o orçamento | Recuperação seletiva top-K rankeada dentro do orçamento (R-05) |
| Reinferências desnecessárias | Custo e latência repetidos para prompts semanticamente equivalentes | Cache semântico de dois níveis (Redis L1 + `pgvector` L2), chave por similaridade (R-06) |
| Estimativa de tokens imprecisa | Orçamento calculado errado, montagens que estouram o limite real do provedor | Contagem/estimativa por tokenizer do modelo alvo via `017` (R-07, erro ≤ 2%, NFR-010) |
| Falta de telemetria de custo/eficácia | Impossível otimizar compressão/cache sem medir | Métricas (`Context Compression Ratio`, cache hit ratio) e eventos `context.window.*` (R-08) |
| Vazamento entre tenants via cache | Um tenant recebendo resposta cacheada de outro tenant — violação grave de isolamento | RLS + namespace de cache por `tenant_id` + `similarity_threshold` estrito (R-09, NFR-014) |
| Falha em cascata quando dependências caem | Um `010`/`017` indisponível travaria toda inferência | Degradação graciosa: contexto truncado seguro em vez de falha total (R-10, FM-01..FM-05) |

## 3. Escopo

### 3.1 Dentro de escopo (in)

O módulo DEVE prover, conforme brief §1.2:

- **Assembly de contexto** — montar um `ContextBundle` ótimo por chamada de LLM
  respeitando o `token_budget` do modelo alvo (R-01).
- **Token budgeting** — alocar orçamento por seção (system, instruções, memória,
  histórico, ferramentas, saída reservada) via `BudgetProfile` (R-02).
- **Compressão/sumarização hierárquica** — redução de tokens por map-reduce
  multinível, preservando semântica relevante (R-03).
- **Remoção de redundância** — deduplicação e eliminação de sobreposição entre
  fragmentos candidatos (R-04).
- **Recuperação seletiva** — recall filtrado ao `010-Memory`, rankeado por
  relevância dentro do orçamento (R-05).
- **Cache semântico** — Redis (L1) + `pgvector` (L2), chave por *embedding* e por
  hash exato, com invalidação event-driven (R-06).
- **Contagem/estimativa de tokens** — por tokenizer do modelo alvo, via contrato
  do `017-Model-Router` (R-07).
- **Emissão de telemetria e eventos** — métricas OTel/Prometheus e eventos NATS
  `aios.<tenant>.context.window.*`/`context.cache.*`/`context.budget.*` (R-08).
- **Governança e isolamento** — PEP/PDP (`022`), isolamento multi-tenant por
  `tenant_id`, trilha de auditoria via eventos consumidos por `025` (R-09).
- **Degradação graciosa** — contexto viável (truncamento seguro) quando `010`/
  `017` degradam (R-10).

### 3.2 Fora de escopo (out / não-objetivos explícitos)

O módulo **NÃO DEVE** assumir as responsabilidades abaixo (brief §1.3); cada uma
tem dono correto e é fronteira dura:

| # | Não-objetivo | Dono correto |
|---|--------------|--------------|
| N-01 | Chamar diretamente provedores de LLM para inferência final da tarefa. | `../017-Model-Router/` |
| N-02 | Persistir memória de longo prazo, consolidar ou esquecer itens de memória. | `../010-Memory/` |
| N-03 | Decidir qual modelo executa a tarefa ou seus limites de janela. | `../017-Model-Router/` |
| N-04 | Executar ferramentas/tools ou hospedar o loop ReAct. | `../015-Tool-Manager/`, `../007-Agent-Runtime/` |
| N-05 | Manter o grafo de conhecimento / GraphRAG. | `../018-Knowledge/`, `../019-GraphRAG/` |
| N-06 | Definir políticas RBAC/ABAC (apenas consumir como PEP). | `../022-Policy/` |
| N-07 | Escalonar agentes/tarefas ou aplicar cotas de CPU/RAM. | `../009-Scheduler/`, `../006-Kernel/` |
| N-08 | Ser a fonte da verdade de auditoria (apenas emite eventos auditáveis). | `../025-Audit/` |

> **Fronteira crítica: compressão vs. sumarização de memória.** O `010-Memory`
> decide o que **é lembrado** e por quanto tempo; o `011-Context` decide o que
> **entra na janela agora** e sob qual forma comprimida. A sumarização feita aqui
> é **efêmera e por chamada**; sumários reutilizáveis de longo prazo pertencem ao
> `010`. O `011` PODE materializar `SummaryNode` como cache de curta duração,
> **nunca** como memória canônica (brief §1.3, nota final).

## 4. Personas atendidas

| Persona | Necessidade | Operação típica |
|---------|-------------|-------------------|
| **Agent Runtime (007)** | Montar o contexto ótimo antes de cada chamada de inferência, sem conhecer detalhes de compressão/cache. | `POST /v1/context/assemble` (ou `Assemble` gRPC) |
| **Model Router (017)** | Saber quantos tokens um payload consumiria antes de escolher/roteirizar modelo; fornece limites/tokenizer/embedding ao `011`. | `POST /v1/context/tokens/estimate`; papel de provedor via `ModelRouterClient` |
| **Memory (010)** | Ser consultado seletivamente (não em massa) e ter suas mudanças refletidas em cache/sumários derivados. | Consumo de `memory.item.consolidated`/`memory.item.deleted`; alvo de `SelectiveRetriever` |
| **Operador de plataforma** | Ajustar `BudgetProfile`, limiares de similaridade e TTLs por tenant/task_type. | `GET/PUT /v1/context/budgets/{scope}`; chaves `context.*` |
| **Security/Audit (021/025)** | Garantir isolamento de tenant no cache e expurgo rastreável de contexto/cache com PII. | Eventos `context.cache.evicted`/`invalidated`; auditoria via `025` |
| **Cost-Optimizer (026)** | Conhecer `tokens_saved`/`cost_saved_usd` do cache semântico para relatórios de economia. | Consumo de `context.window.cache_hit` |

## 5. Princípios de design

1. **O contexto é um recurso escasso, não um texto livre.** Toda montagem DEVE
   respeitar um orçamento explícito (`token_budget`) derivado do limite real do
   modelo (`017`) — nunca um "melhor esforço" sem teto (R-01, R-02, FR-001).
2. **Comprimir preservando desempenho, não apenas tokens.** A meta não é a menor
   contagem de tokens possível, mas o melhor par (tokens, Task Completion Rate);
   uma compressão que degrada a tarefa além do limite aceitável é uma regressão,
   não uma otimização (NFR-004, NFR-005).
3. **Cache é ganho, nunca corrupção.** Um *hit* de cache semântico só é servido
   quando `similarity ≥ similarity_threshold`; a dúvida DEVE resolver-se a favor
   de recomputar (miss), nunca de servir uma resposta possivelmente incorreta —
   *false-hit* é tratado como defeito de segurança, não de desempenho
   (NFR-011, ADR-0112).
4. **Degradar graciosamente, nunca falhar em cascata.** Quando `010`/`017`
   degradam, o módulo DEVE entregar um contexto viável truncado com segurança em
   vez de propagar a falha para a inferência inteira (R-10, FM-01..FM-05).
5. **Isolamento por padrão.** `tenant_id` + Row-Level Security em toda tabela;
   namespace de cache e de subjects NATS por tenant; *default deny* no PEP
   (R-09, RFC-0001 §5.8, NFR-014).
6. **Não reinventar contratos.** URN, envelope de evento, envelope de erro,
   idempotência e correlação vêm da RFC-0001 (§5) — o módulo os **reutiliza**
   integralmente; a única extensão proposta é a RFC-0011 (contrato de *assembly*
   e de cache semântico).
7. **Delegar, não decidir sozinho.** Limites/tokenizer/*embedding*/sumarização
   vêm do `017`; recall de memória vem do `010`; autorização vem do `022`;
   auditoria é **emitida**, não armazenada aqui (não-objetivos §3.2).
8. **Observável de ponta a ponta.** 100% dos *assemblies* emitem trace OTel e
   pelo menos um evento (`assembled`, `cache_hit`, `cache_miss` ou `rejected`)
   (FR-009, NFR-013).

## 6. Relação com a visão global do AIOS

O AIOS trata agentes como "processos" e recursos cognitivos como recursos de um
sistema operacional (`../000-Vision/Vision.md`, `../001-Architecture/
Architecture.md`). Dentro dessa visão, o `011-Context` é o par cognitivo do
gerenciador de **paginação/cache de memória virtual**: onde o `010-Memory`
responde "o que existe e por quanto tempo", o `011-Context` responde "o que cabe
agora, comprimido como, e o que já sei a resposta sem perguntar de novo". Ele é
consumidor direto do `017-Model-Router` (limites de janela, tokenizer, modelos de
*embedding* e sumarização) e do `010-Memory` (recall seletivo), e fornecedor de
contexto pronto para o `007-Agent-Runtime`/`006-Kernel` orquestrarem a inferência
final. O módulo consome integralmente a baseline de contratos da RFC-0001 e
propõe a **RFC-0011** (*Context Assembly & Semantic Cache Contract*) para
formalizar o contrato de `assemble`, a semântica de chave de cache por
similaridade e a definição normativa de **Context Compression Ratio**.

## 7. Exemplo ilustrativo (ciclo de vida de uma montagem)

```
Agent Runtime solicita assemble() para uma nova chamada de LLM
   │  RECEIVED → POLICY_CHECK (PEP consulta 022)             → allow
   ▼
CACHE_LOOKUP: prompt normalizado tem entrada semanticamente equivalente?
   │  similarity ≥ threshold → HIT: serve do cache, emite cache_hit  (fim rápido)
   │  MISS/BYPASS → segue para orçamento
   ▼
BUDGET_ALLOCATED: TokenBudgeter aloca por seção a partir do limite do modelo (017)
   ▼
RETRIEVING: SelectiveRetriever pede recall seletivo ao 010-Memory (top-K)
   ▼
RANKING: RelevanceRanker rankeia candidatos por similaridade + recência + prioridade
   ▼
COMPRESSING: RedundancyEliminator remove near-duplicates;
             HierarchicalCompressor sumariza map-reduce até caber no orçamento
   ▼
ASSEMBLED: tokens_in ≤ token_budget → persiste ContextBundle + fragments
   ▼
SERVED: entrega ao chamador, publica evento window.assembled (outbox),
        registra compression_ratio e cache_outcome=MISS para telemetria
```

Este ciclo (detalhado na FSM de `./StateMachine.md`, derivada do brief §4.1)
resume a proposta de valor do módulo: **caber no orçamento, comprimir sem perder
o que importa, e nunca recomputar o que já se sabe.**

## 8. Métrica-norte

A métrica-norte do módulo é a **Context Compression Ratio**
(`../040-Glossary/Glossary.md`): a redução de tokens obtida **preservando** o
desempenho de tarefa. A meta é **≥ 0,50** em média (NFR-004 em
`./NonFunctionalRequirements.md`), mas ela é sempre lida em par com a métrica de
guarda **Δ Task Completion Rate vs. sem compressão ≥ -2% (absoluto)** (NFR-005):
uma compressão que reduz mais tokens às custas de mais de 2 pontos percentuais de
qualidade de tarefa não é uma vitória — é uma regressão que DEVE disparar
ajuste de `BudgetProfile`/`min_compression_ratio` ou revisão do método de
compressão (ADR-0110). Em paralelo, o **cache hit ratio** em regime estável
(≥ 0,35, NFR-006) e o **custo evitado / custo bruto** (≥ 0,25, NFR-007) medem a
segunda dimensão de valor do módulo: reinferências evitadas.

## 9. Riscos e alternativas (resumo)

| Risco | Mitigação | Decisão |
|-------|-----------|---------|
| Cache semântico serve resposta errada (*false-hit*) | Limiar de similaridade estrito + auditoria de amostragem + quarentena de cluster afetado | ADR-0111, ADR-0112, ADR-0119 |
| Sumarização perde informação crítica | Map-reduce hierárquico com verificação de fidelidade; guarda de Task Completion Rate (NFR-005) | ADR-0110, ADR-0115 |
| Estimativa de tokens imprecisa gera estouro de orçamento real | Tokenizer exato via `017` quando disponível; erro ≤ 2% monitorado (NFR-010) | ADR-0114 |
| Indisponibilidade de `010`/`017` trava toda inferência | Estado `DEGRADED` com truncamento seguro em vez de falha total | ADR-0118 |
| Vazamento de contexto entre tenants via cache | RLS + namespace por tenant + limiar estrito + teste de fuzz de `tenant` (NFR-014) | ADR-0111, ADR-0119 |
| Cache cresce sem controle e degrada latência | Eviction TTL + LRU semântico + `max_entries_per_tenant` | ADR-0116 |

Decisões DEVEM apontar para as ADRs acima (índice em `./ADR.md`); nada neste
documento decide arquitetura localmente.
