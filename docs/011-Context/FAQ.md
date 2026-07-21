---
Documento: FAQ
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 022-Policy, 025-Audit, 020-Communication, 005-Database, 021-Security
---

# 011-Context — FAQ

> Este documento reúne perguntas frequentes, armadilhas comuns e esclarecimentos
> de escopo sobre o Context Manager. Ele NÃO substitui `./_DESIGN_BRIEF.md`; em
> caso de conflito aparente, o brief prevalece. Palavras normativas conforme
> RFC 2119/8174.

---

## 1. Conceituais

### 1.1 O `011-Context` decide qual modelo de LLM executa a tarefa?

**Não.** Essa é responsabilidade do `../017-Model-Router/`. O `011-Context`
recebe do `017` os **limites de janela** (`model_max_tokens`), o **tokenizer** e
os endpoints de *embedding*/sumarização para o `model_id` já escolhido — ele
consome esse contrato, nunca decide qual modelo é o alvo. Uma armadilha comum é
tratar `POST /v1/context/tokens/estimate` como um mecanismo de seleção de
modelo; ele apenas estima quantos tokens um payload consumiria **para um
`model_id` já dado** (não-responsabilidade N-03 do brief §1.3).

### 1.2 Qual é a diferença entre o que o `010-Memory` faz e o que o `011-Context`
faz? Os dois não sumarizam informação?

Sim, ambos podem produzir "resumos", mas com propósitos e ciclos de vida
diferentes. O `010-Memory` decide **o que é lembrado e por quanto tempo** — sua
sumarização (consolidação) é dirigida por `023-Learning`, versionada e
**reversível**, e vira memória canônica de longo prazo. O `011-Context` decide
**o que entra na janela agora e sob qual forma comprimida** — sua sumarização é
**efêmera e por chamada** (map-reduce hierárquico do `HierarchicalCompressor`).
O `011` PODE materializar um `SummaryNode` como cache de curta duração para
reuso por similaridade, mas esse nó **nunca** é memória canônica; ele expira
(`expires_at`) e não participa da consolidação do `010` (brief §1.3, nota de
fronteira).

### 1.3 O `011-Context` executa ferramentas ou hospeda o loop ReAct do agente?

**Não.** Isso é do `../015-Tool-Manager/` (execução/registro de ferramentas) e
do `../007-Agent-Runtime/` (loop cognitivo). O `011-Context` apenas **inclui**
resultados de ferramentas já executadas como fragmentos de tipo `tool_result`
(`source_kind` em `ContextFragment`, ver `./Database.md`) na montagem do bundle,
sujeitos ao mesmo orçamento e às mesmas regras de compressão que qualquer outro
fragmento (não-responsabilidade N-04).

### 1.4 "Janela de contexto" (`Context Window`) é o mesmo conceito em todo o
AIOS ou o `011` redefine algo?

É exatamente o termo do `../040-Glossary/Glossary.md`: "limite de tokens que um
modelo processa por chamada. Recurso escasso gerenciado pelo Context Manager."
O `011-Context` não redefine o conceito — ele é o subsistema que **gerencia**
esse recurso, análogo ao gerenciador de paginação/cache de memória virtual de
um SO clássico (ver `./Vision.md` §1).

---

## 2. Token Budgeting e Assembly

### 2.1 O que acontece se o contexto mínimo obrigatório (system + instruções)
já não couber no `token_budget`?

A montagem é rejeitada: transição `*→REJECTED` com erro `AIOS-CTX-0001`
(`BudgetInfeasible`, 422, não retriável — ver `./_DESIGN_BRIEF.md` §5.3 e §4.1).
O módulo **não** tenta "espremer" o mínimo obrigatório além do reconhecível —
isso produziria um contexto corrompido/inútil. A resposta de erro é uma
oportunidade para o chamador (tipicamente o `017-Model-Router`, via sugestão de
modelo maior) agir, não um "melhor esforço" silencioso.

### 2.2 Os pesos de `BudgetProfile` (`alloc_system`, `alloc_instruction`, ...)
podem somar mais ou menos que 1.0?

**Não.** A invariante de alocação (`alloc_system + alloc_instruction +
alloc_memory + alloc_history + alloc_tools = 1.0`) é validada em toda escrita;
uma violação retorna `AIOS-CTX-0008` (`InvalidBudgetProfile`, 422, não
retriável — brief §3.4, §5.3). Não existe normalização silenciosa como no
`Scheduler` para pesos de custo/latência: aqui a soma exata é exigida porque
representa 100% do orçamento de entrada, não uma ponderação relativa livre.

### 2.3 `reserved_output_ratio` reduz o `token_budget` de entrada ou é somado
ao limite do modelo?

Reduz. `token_budget` (orçamento de **entrada**) é derivado de
`model_max_tokens - reserved_output_tokens`, e `reserved_output_tokens` é
calculado a partir de `reserved_output_ratio` (default `0.25`) do
`BudgetProfile` efetivo. Ou seja, se o modelo tem uma janela de 128k tokens e
`reserved_output_ratio=0.25`, o orçamento de entrada é de até ~96k tokens — os
32k restantes são reservados para a geração da resposta e nunca disputados
pelos fragmentos de contexto (brief §3.4, §7.1 FR-002).

### 2.4 Por que `tokens_raw` e `tokens_in` existem em `ContextBundle` — não
bastaria armazenar só o resultado final?

Porque a métrica-norte do módulo, **Context Compression Ratio**
(`1 - tokens_in/tokens_raw`), depende de ambos, e porque a auditoria de
eficácia de compressão (NFR-004) e a depuração de casos em que a compressão
"não ajudou o suficiente" exigem saber quanto havia **antes** de comprimir, não
apenas o resultado final (brief §3.1).

---

## 3. Compressão e Sumarização

### 3.1 O `011-Context` sempre sumariza com um modelo de LLM (custo adicional)?

Não necessariamente. `context.compression.method` (default `hierarchical`)
PODE ser configurado como `extractive` (seleção/corte sem gerar texto novo) ou
`truncate` (determinístico, sem custo de inferência) por tenant/`task_type`.
A sumarização hierárquica (`hierarchical`) é a mais poderosa em preservação de
semântica, mas envolve ao menos uma chamada ao modelo de sumarização (roteado
via `017`, tipicamente um modelo barato — `context.compression.
summarization_model_hint=cheap-small`), o que eleva a meta de latência p99 de
150 ms (sem sumarização, NFR-002) para 1200 ms (com uma chamada, NFR-003).

### 3.2 O que acontece se a chamada ao modelo de sumarização falhar?

O módulo aplica **fallback determinístico de truncamento** (`truncate`) para
ainda respeitar o orçamento, emite `AIOS-CTX-0004` (`CompressionFailed`, 500,
retriável) para telemetria/observabilidade, mas **entrega um bundle utilizável**
em vez de propagar a falha (estado `COMPRESSING → DEGRADED` com recuperação —
brief §4.1, §9 FM-04). Isso é consistente com o princípio de degradação
graciosa (`Vision.md` §5, princípio 4).

### 3.3 Como o `RedundancyEliminator` decide que dois fragmentos são
"near-duplicate"?

Por similaridade de cosseno entre seus *embeddings*: `cos ≥
context.dedup.cosine_threshold` (default `0.95`). Quando dois ou mais
fragmentos ultrapassam esse limiar, apenas um é mantido (tipicamente o de maior
`relevance_score`), e os demais são marcados `included=false` no
`ContextFragment` — mas **permanecem persistidos** para fins de auditoria e
depuração, apenas não entram no `content` final do bundle (FR-004, brief §3.2).

### 3.4 Por que a compressão é "hierárquica" e não uma única passada de
sumarização?

Porque uma única passada sobre um conjunto grande de fragmentos heterogêneos
tende a diluir sinal e perder detalhes específicos. A abordagem map-reduce
multinível (`context.compression.max_levels`, default `3`) sumariza em grupos
menores primeiro (nível 0 → folhas), depois combina esses sumários em níveis
progressivamente mais altos — preservando melhor itens de alta relevância
(`score ≥ p90`, critério de aceite de FR-003) do que uma compressão "tudo de
uma vez". Cada nível produz um `SummaryNode` rastreável (`level`, `parent_id`).

---

## 4. Cache Semântico

### 4.1 O cache semântico serve qualquer resposta "parecida o suficiente"?

**Não.** Um *hit* só é servido quando `similarity ≥ similarity_threshold`
(default `0.92`, configurável por tenant/`task_type`) **e** a entrada não está
`INVALIDATED`/expirada. O limiar é deliberadamente conservador: a taxa de
falso-*hit* (resposta inadequada servida) tem meta **≤ 0,5%** (NFR-011) — em
caso de dúvida sobre a suficiência da similaridade, o sistema prefere um *miss*
(recomputar) a arriscar entregar uma resposta semanticamente incompatível
(`Vision.md` §5, princípio 3; ADR-0112).

### 4.2 Qual a diferença entre o cache L1 (Redis) e o L2 (`pgvector`)?

L1 (Redis) é o caminho **rápido** (`CacheLookup` p99 ≤ 20 ms, NFR-001):
sub-milissegundo, TTL curto (`context.cache.l1_redis_ttl_seconds`, default
300 s), particionado por *shard*. L2 (`pgvector`, índice HNSW por padrão) é o
armazenamento **durável** de entradas de cache, com busca ANN (*approximate
nearest neighbor*) de menor latência que uma varredura exaustiva, porém mais
lenta que L1. Um *hit* em L1 evita completamente a busca vetorial em L2 — L2 só
é consultado em *miss* de L1 (`_DESIGN_BRIEF.md` §10).

### 4.3 Existe um caminho "exato" além da busca por similaridade?

Sim — `prompt_hash` (hash do prompt normalizado) é o **fast path** de
correspondência exata, verificado antes ou em paralelo à busca por
`prompt_embedding` (**slow path** de similaridade). O fast path é
particularmente importante como *fallback* quando o serviço de *embedding* do
`017` está indisponível (`AIOS-CTX-0009`): o cache degrada para
correspondência exata em vez de ficar completamente inoperante (brief §9,
FM-05).

### 4.4 Quando uma entrada de cache é invalidada automaticamente, sem
intervenção manual?

Por evento: `aios.<tenant>.memory.item.consolidated` e
`aios.<tenant>.memory.item.deleted` (produzidos por `010-Memory`) disparam
invalidação das entradas de cache/`SummaryNode` derivadas do item alterado, com
meta de latência **≤ 5 s** após o evento (FR-008). Da mesma forma,
`aios.<tenant>.knowledge.node.updated` (`018`/`019`) invalida cache derivado de
conhecimento, e `aios.<tenant>.model.registry.updated` (`017`) invalida
orçamentos/limites cacheados. Isso evita servir uma resposta cacheada que ficou
semanticamente desatualizada em relação à sua origem.

### 4.5 O que é "cache poisoning" no contexto do `011`, e como o módulo se
defende?

É o risco de uma entrada de cache — legítima ou manipulada — passar a servir
respostas inadequadas de forma sistemática (por exemplo, um limiar de
similaridade calibrado frouxo demais gerando falsos-*hits* em massa). A defesa
combina: limiar conservador por padrão (§4.1), auditoria por amostragem/*eval*
offline (NFR-011), e um modo de resposta a incidente — elevar
`similarity_threshold` e invalidar o *cluster* afetado em quarentena
(FM-08, ADR-0119). Não é tratado como "só" um problema de desempenho; é
primariamente um problema de **integridade de resposta**.

### 4.6 Posso desabilitar o cache semântico para um tenant específico?

Sim, via `context.cache.enabled=false` (escopo tenant/agent, recarregável em
runtime). Quando desabilitado, toda montagem segue diretamente para
`BUDGET_ALLOCATED` com `cache_outcome=BYPASS` — nenhuma tentativa de
`CACHE_LOOKUP` é feita, e nenhuma entrada nova é criada em `CacheStore` para
esse escopo (`_DESIGN_BRIEF.md` §8, §4.1).

---

## 5. Recuperação Seletiva e Integração com `010-Memory`

### 5.1 O `011-Context` chama o `010-Memory` para "trazer tudo que for
relevante"?

**Não.** `SelectiveRetriever` solicita recall **seletivo**, limitado por
`context.retrieval.top_k` (default `32`) e por um *deadline* estrito
(`context.retrieval.memory_deadline_ms`, default `120 ms`) — não uma busca
aberta. A responsabilidade de rankear o que efetivamente entra no orçamento é
do `RelevanceRanker` do próprio `011`, que combina o resultado do recall com
similaridade, recência e prioridade antes de decidir o que compor no bundle
(brief §2.1, FR-005).

### 5.2 O que acontece se o `010-Memory` não responder dentro do *deadline*?

O estado transita para `DEGRADED` (não para `FAILED`): o `011-Context` monta um
bundle com o contexto disponível (tipicamente system + instruções + histórico
recente), sem os fragmentos de memória que não chegaram a tempo, e emite
`AIOS-CTX-0005` (`MemoryRecallTimeout`, 504, retriável) para telemetria. O
bundle resultante ainda é `SERVED` — parcial, mas utilizável (brief §4.1,
FM-02, princípio de degradação graciosa).

### 5.3 O `011-Context` grava algo de volta no `010-Memory` (por exemplo, o
próprio `ContextBundle` como um novo item de memória)?

**Não.** O fluxo é estritamente de leitura em relação ao `010`: o `011` **lê**
via recall seletivo e **reage** a eventos de mudança de origem (invalidação de
cache), mas nunca escreve itens de memória. Persistir/consolidar memória é
exclusividade do `010-Memory` (não-responsabilidade N-02 do brief §1.3).

---

## 6. Eventos, API e Integração

### 6.1 Por que `assemble` retorna um `ContextBundle` completo e não apenas o
texto final do prompt?

Porque o chamador (tipicamente o `007-Agent-Runtime`) precisa de metadados
operacionais além do conteúdo: `tokens_in`, `compression_ratio`,
`cache_outcome`, `bundle_id` (para auditoria/depuração posterior via `GET
/v1/context/bundles/{bundle_id}`) e `status`. Reduzir a resposta a uma string
esconderia informação necessária para observabilidade e para decisões
subsequentes do chamador (por exemplo, decidir se um `compression_ratio` muito
alto merece investigação).

### 6.2 Que evento devo consumir para saber que um bundle foi de fato servido
com sucesso?

`aios.<tenant>.context.window.assembled` — emitido na transição
`ASSEMBLED→SERVED`, após persistência e publicação via *outbox* confirmadas.
Não confunda com `context.window.cache_hit`: este último é emitido quando a
resposta veio inteiramente do cache (sem passar pelo pipeline de retrieve/rank/
compress); ambos representam "sucesso", mas por caminhos distintos, úteis para
distinguir em dashboards de eficácia de cache (`./Events.md`).

### 6.3 Devo usar REST ou gRPC para o caminho quente de `assemble` em
produção?

O caminho **recomendado para uso interno em produção** é gRPC
(`aios.context.v1.ContextService.Assemble`), por menor overhead de
serialização e alinhamento com as metas de latência p99 (NFR-001..NFR-003).
REST (`POST /v1/context/assemble`) é preferencial para integrações
administrativas, testes manuais e chamadas externas via Gateway YARP.

### 6.4 O `011-Context` publica eventos mesmo quando a montagem é rejeitada?

Sim. `aios.<tenant>.context.window.rejected` é emitido para toda transição
`*→REJECTED`, carregando `code` e `reason` — isso é essencial para que
operadores e o próprio `017-Model-Router` (que pode reagir sugerindo um modelo
com janela maior) tenham visibilidade de rejeições, em vez de apenas o
chamador individual ver o erro síncrono (`./Events.md`, brief §6.1).

---

## 7. Segurança e Multi-tenancy

### 7.1 Um `tenant_id` dentro do payload de `assemble` pode divergir do
`X-AIOS-Tenant` autenticado?

**Não.** O `tenant` do token autenticado DEVE igualar `X-AIOS-Tenant`; qualquer
divergência é rejeitada com `AIOS-CTX-0011` (`TenantMismatch`, 403, não
retriável), independentemente do que o payload interno declare — regra
herdada de RFC-0001 §6 e não uma decisão local do `011` (brief §12.1).

### 7.2 O cache semântico é compartilhado entre tenants para economizar
recursos (ex.: prompts genéricos comuns)?

**Não, por padrão e por desenho de segurança.** Toda entrada de
`SemanticCacheEntry` tem `tenant_id` obrigatório com Row-Level Security, e o
namespace de cache (chaves Redis, partições `pgvector`) é isolado por tenant
(`_DESIGN_BRIEF.md` §12.1, NFR-014). Compartilhar cache entre tenants exigiria
uma ADR explícita de exceção com justificativa e controles adicionais — não é
o comportamento padrão do módulo.

### 7.3 Upsert de `BudgetProfile` ou invalidação em massa de cache exigem
alguma autorização especial?

Sim — são operações privilegiadas que o `ContextPolicyGuard` (PEP local)
**sempre** encaminha ao PDP do `022-Policy` antes de executar, com *default
deny*. Diferente de uma consulta de leitura simples (`GetBudget`), mutações que
afetam todo um escopo (tenant/agent/task_type) não são auto-serviço livre
(brief §2.1, §12.1).

### 7.4 Conteúdo com PII pode ficar em cache indefinidamente?

Não deveria. Entradas de cache com PII DEVEM respeitar TTL reduzido e ser
expurgáveis; o expurgo rastreável por `source_urn`/tenant (FR-012) é propagado
por `aios.<tenant>.memory.item.deleted`, invalidando/evictando a entrada
correspondente e emitindo `context.cache.evicted` + registro auditável (brief
§12.3). *Embeddings* de PII são tratados como dado pessoal — não reversíveis a
texto por si só, mas isolados por tenant como qualquer outro dado sensível.

---

## 8. Evolução e Extensibilidade

### 8.1 Trocar `context.compression.method` de `hierarchical` para `extractive`
muda a API externa ou o schema de `ContextBundle`?

**Não.** O contrato de `ContextBundle`/`ContextFragment` (`compression_method`
por fragmento é um enum que já inclui `extractive`) permanece estável; apenas a
estratégia interna do `HierarchicalCompressor`/pipeline muda. Essa
estabilidade é o que permite ajustar a estratégia de compressão por
tenant/`task_type` sem quebrar clientes (brief §8, chave `context.compression.
method`).

### 8.2 Um novo `<tipo>` de URN pode ser adicionado além de `context`,
`ctxfrag` e `ctxcache`?

Só via RFC/ADR de módulo que estenda o registro de `<tipo>` em
`../004-API/Resources.md`, seguindo o processo definido pela RFC-0001 §8. Os
três tipos atuais (`context`, `ctxfrag`, `ctxcache`) foram propostos pela
RFC-0011; adicionar um quarto tipo (por exemplo, para um novo tipo de artefato
de cache) exige o mesmo processo formal — não é uma decisão de implementação
local.

### 8.3 Adicionar um novo `source_kind` de `ContextFragment` (além de
`system`/`instruction`/`memory`/`knowledge`/`history`/`tool_result`) é uma
mudança compatível?

É aditiva (compatível) se consumidores existentes tolerarem valores
desconhecidos, conforme a regra geral de evolução de eventos/schemas da
RFC-0001 §5.7. Redefinir o significado de um `source_kind` existente, porém, é
incompatível e exige nova versão de RFC/ADR do módulo — o mesmo princípio
aplicado a `service_class` no `009-Scheduler`.

---

## 9. Armadilhas Comuns (resumo rápido)

| Armadilha | Por que é um erro | Correção |
|-----------|---------------------|----------|
| Tratar `EstimateTokens` como seleção de modelo | Estima tokens para um `model_id` já dado; não escolhe modelo. | Consultar `017-Model-Router` para escolha de modelo; usar `011` só para a estimativa. |
| Assumir que `cache_outcome=HIT` sempre economiza a chamada de LLM completa | É exatamente isso, mas só ocorre se `similarity ≥ threshold`; abaixo disso é MISS garantido. | Não reduzir manualmente `similarity_threshold` para "forçar" mais hits — eleva risco de falso-hit (NFR-011). |
| Configurar pesos de `BudgetProfile` que não somam exatamente 1.0 | Retorna `AIOS-CTX-0008`; não há normalização automática como em outros módulos. | Validar a soma antes do `PUT /v1/context/budgets/{scope}`. |
| Reenviar `assemble`/`cache store` com mesma `Idempotency-Key` e payload diferente | Gera conflito (`AIOS-CTX-0012`), não uma atualização silenciosa. | Gerar nova `Idempotency-Key` para requisições semanticamente diferentes. |
| Esperar que fragmentos marcados `included=false` tenham sido apagados | Eles permanecem persistidos para auditoria; apenas não entram no bundle final. | Filtrar por `included=true` ao consumir o conteúdo montado. |
| Achar que o `011` consolida/aprende memória de longo prazo a partir do que monta | Sumarização aqui é efêmera e por chamada; consolidação é exclusiva do `010`. | Usar `010-Memory` (`consolidate`) para qualquer aprendizado que deva persistir. |
| Ignorar o estado `DEGRADED` como se fosse falha | `DEGRADED` ainda produz um bundle `SERVED`, apenas parcial. | Tratar `DEGRADED` como sucesso parcial observável, não como erro do chamador. |

---

## 10. Referências

- Design Brief: `./_DESIGN_BRIEF.md`
- Visão do módulo: `./Vision.md`
- Máquina de estados: `./StateMachine.md`
- Contratos completos de API: `./API.md`
- Catálogo de eventos: `./Events.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`

---

*Fim do FAQ do Módulo 011-Context.*
