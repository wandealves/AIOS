---
Documento: FAQ (Perguntas Frequentes e Armadilhas)
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100..ADR-0109
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 001-Architecture, 011-Context, 017-Model-Router, 018-Knowledge, 022-Policy, 023-Learning, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 010 · Memory — FAQ

> Perguntas frequentes, armadilhas comuns e esclarecimentos de escopo do
> `MemoryService` (010). Todas as respostas derivam do `_DESIGN_BRIEF.md` e
> **não o contradizem**. Termos reutilizam o
> [Glossário](../040-Glossary/Glossary.md). Palavras normativas conforme RFC 2119.

---

## 1. Escopo e fronteiras

### 1.1 O que exatamente o MemoryService faz?

É o **gerenciador de memória cognitiva hierárquica** do AIOS — análogo a
MMU + swap + filesystem, porém para recursos cognitivos. Expõe quatro operações
canônicas: `remember(item, layer)`, `recall(query, filtros)`, `consolidate()` e
`forget(policy)` sobre **7 camadas** físicas (Working, Short-Term, Long-Term,
Semantic, Procedural, Episodic, Knowledge Graph), ocultando a heterogeneidade dos
backends (Redis/PostgreSQL/pgvector/AGE/MinIO). Ver [Vision.md](./Vision.md) e
[Architecture.md](./Architecture.md).

### 1.2 O que o módulo NÃO faz? (armadilha de escopo)

| Você quer... | Dono correto | Por quê |
|--------------|--------------|---------|
| Gerar embeddings / treinar modelos | **017-Model-Router** | 010 só *requisita* embeddings (N1) |
| Decidir *quando/como* aprender | **023-Learning** | 010 só *executa* a consolidação solicitada (N2) |
| Montar a janela de contexto / comprimir prompt | **011-Context** | 010 fornece itens; 011 monta o contexto (N3) |
| GraphRAG de conhecimento público/global | **018/019-Knowledge** | 010 guarda o grafo de memória *do agente*, não o de domínio (N4) |
| Autorizar acesso (PDP) | **022-Policy** | 010 é apenas PEP (N5) |
| Trilha de auditoria imutável | **025-Audit** | 010 só emite eventos que o Audit consome (N6) |
| Cobrar/orçar | **026-Cost-Optimizer** | 010 só *reporta* consumo (N9) |

### 1.3 Qual a diferença entre o Knowledge Graph do 010 e o Knowledge do 018/019?

O `KnowledgeGraphAdapter` (010) guarda o grafo de **memória consolidada de um
agente** (nós/arestas privados do agente/tenant, via Apache AGE). O conhecimento de
**domínio público** e o GraphRAG de recuperação global são responsabilidade do
**018/019-Knowledge** (N4). Não confunda "memória do agente" com "base de
conhecimento compartilhada".

---

## 2. Camadas e roteamento

### 2.1 Como escolho a camada certa?

Você informa `layer` explicitamente ou deixa o `LayerRouter` classificar pelo
`kind`/hint (FR-004). Mapa rápido (brief §3.3):

| Preciso guardar... | Camada | Backend | TTL default |
|--------------------|--------|---------|-------------|
| Raciocínio corrente (volátil) | Working | Redis | 15 min |
| Contexto recente da sessão | Short-Term | Redis + PG | 24 h |
| Memória consolidada durável | Long-Term | PostgreSQL | 90 dias |
| Fatos/conceitos gerais (busca semântica) | Semantic | PG + pgvector | sem TTL (decay) |
| Habilidades / "como fazer" | Procedural | PostgreSQL | sem TTL |
| Eventos/experiências com tempo | Episodic | PG + pgvector (part. tempo) | 365 dias |
| Relações entre memórias | Knowledge Graph | Apache AGE | sem TTL |

> **Armadilha:** enviar um `kind` incompatível com a `layer` retorna
> `422 AIOS-MEM-0020`. Ex.: `kind=edge` só faz sentido no Knowledge Graph.

### 2.2 Por que a Working Memory pode perder dados?

Por **design** (NFR-008, ADR-0007). A Working Memory é efêmera (Redis, RAM), com
RPO **não garantido** e reconstruível. Isso é o que permite escala a milhões de
agentes: só o *working set* de agentes ativos ocupa RAM. Se você precisa de
durabilidade, use Short-Term (projeção durável em PG) ou superior. Ver
[FailureRecovery.md §7](./FailureRecovery.md).

---

## 3. Recuperação (recall)

### 3.1 O que significa `mode` em `recall`?

Controla o motor de busca do `RecallEngine` (brief §5.1):

| `mode` | Busca | Latência-alvo (NFR-003) |
|--------|-------|--------------------------|
| `lexical` | texto/keywords | baixa |
| `vector` | ANN via pgvector/HNSW (Semantic/Episodic) | p99 ≤ 150 ms |
| `graph` | travessia Apache AGE | p99 ≤ 250 ms |
| `hybrid` (default) | lexical + vetorial + grafo, fundidos por RRF | conforme camadas |

### 3.2 O que é Memory Recall Rate e por que ≥ 0,95?

É a métrica-chave do módulo (glossário): fração de informação relevante recuperada
quando necessária. O SLO é **≥ 0,95** (NFR-002). Ela também é o *gate* da
consolidação: se uma consolidação regride o Recall Rate em mais de 0,01, há
*rollback* automático (NFR-011, ADR-0102). Exposta em `GET /v1/memory/stats` e na
métrica `aios_memory_recall_rate`.

### 3.3 Por que meu recall às vezes vem `degraded=true`?

Degradação graciosa (F5/F1): se o Apache AGE ou o Redis estão indisponíveis, o
`recall` retorna o melhor resultado possível pelos caminhos restantes e marca
`degraded=true`, em vez de falhar. `mode=graph` explícito com AGE fora retorna
`503 AIOS-MEM-0031`. Ver [FailureRecovery.md §6](./FailureRecovery.md).

### 3.4 Por que um item recém-criado não apareceu na busca vetorial?

Provavelmente o embedding ainda não foi gerado (Model Router em *backlog*, F3): o
item existe com `state=INGESTED` e é recuperável por `lexical`, mas ainda não por
`vector` até o *backfill* do embedding concluir. A escrita **não** falhou.

---

## 4. Consolidação e esquecimento

### 4.1 Consolidação apaga meus dados originais?

Não sem rede de segurança. Toda consolidação grava uma `ConsolidationVersion`
(pré-imagem) **antes** de mutar (invariante do brief §4.1) e mantém linhagem em
`parent_ids`. Se algo regride, o *rollback* restaura a versão anterior (FR-007,
ADR-0102). É a defesa contra *catastrophic forgetting*.

### 4.2 `forget` apaga o item imediatamente?

Não. `forget` move itens elegíveis para `FORGET_PENDING` → `FORGOTTEN` (tombstone),
e só depois do `memory.forget.grace_period` (default 7d) ocorre o `PURGED` físico
(remoção de vetor + blob + tombstone). Isso preserva auditabilidade (brief §4.1
invariante iii).

### 4.3 Qual a precedência entre as estratégias de esquecimento?

As estratégias são `ttl`, `decay`, `lru`, `quota`, `rtbf` (brief §4/§5). A ordem de
precedência é objeto da **ADR-0103**. Regra dura: **RTBF** e cota podem forçar
esquecimento; `legal_hold` bloqueia tudo (`423 AIOS-MEM-0050`) **exceto** RTBF
autorizado.

### 4.4 Como funciona o Direito ao Esquecimento (RTBF)?

Via `DELETE /v1/memory/items/{id}` ou `forget(strategy=rtbf)`. É assíncrono,
rastreável, retorna `202 AIOS-MEM-0051` e emite `memory.item.purged` para auditoria
(025). Remove vetor + blob + tombstone. Um item em `legal_hold` só é purgável sob
RTBF **autorizado** (LGPD/GDPR, brief §12.3, ADR-0109).

---

## 5. Idempotência, eventos e consistência

### 5.1 Como reintento uma escrita com segurança?

**Reutilizando a mesma `Idempotency-Key`.** O servidor persiste o resultado por
chave por ≥ 24h (`IdempotencyStore`, RFC-0001 §5.5) e retorna o resultado
memoizado. Uma chave **nova** cria um item **novo** — essa é a armadilha mais comum.
Reusar a chave com payload divergente retorna `409 AIOS-MEM-0003`.

### 5.2 Os eventos são exactly-once?

Não — são **at-least-once** (JetStream + Outbox). Consumidores DEVEM deduplicar por
`event.id` (RFC-0001 §5.2). O Outbox garante atomicidade DB+evento (RPO evento = 0).
O efeito líquido é *exactly-once* via idempotência.

### 5.3 O Agent Runtime pode ler o PostgreSQL/Redis direto?

**Não** (fronteira crítica do brief §1.2, ADR-0107). O Agent Runtime (Python, plano
de dados) DEVE passar pela API do `MemoryService` (gRPC/NATS). Acesso direto ao
datastore viola a regra de camadas de [001-Architecture](../001-Architecture/Architecture.md).

---

## 6. Cotas, escala e performance

### 6.1 O que acontece quando estouro a cota?

Backpressure: `429 AIOS-MEM-0011` (fila de escrita saturada) ou `429 AIOS-MEM-0010`
(cota de itens/bytes/vetores excedida). Ambos são *retriable*. O `ForgettingEngine`
dispara poda para manter `aios_memory_usage_ratio` < 1,0 (NFR-009). Ver
[Scalability.md §6](./Scalability.md).

### 6.2 Até quantos vetores por tenant?

Meta ≥ **10^8 vetores/tenant** com p99 ANN ≤ 150 ms (NFR-012), via particionamento
por tenant e HNSW por partição (ADR-0108). Caminho evolutivo para índice externo
previsto (ADR-0101/ADR-0005) se o teto for atingido.

### 6.3 Posso mudar `memory.embedding.dim` ou `memory.hnsw.m` a quente?

Não. Essas chaves **não são recarregáveis** — exigem *reindex* (brief §8).
Recarregáveis a quente incluem `memory.hnsw.ef_search`, `memory.recall.default_k`,
TTLs e cotas. Ver [Configuration.md](./Configuration.md).

### 6.4 Como o custo de RAM escala com milhões de agentes?

Escala com **agentes ativos simultâneos**, não com o total. Agentes suspensos viram
*cold agents* (estado em PG/MinIO), reidratados sob `recall` (brief §10). Ver
[Scalability.md §4](./Scalability.md).

---

## 7. Segurança e multi-tenancy

### 7.1 Como o isolamento entre tenants é garantido?

Row-Level Security (RLS) obrigatória em toda tabela por `tenant_id`, namespace NATS
por tenant (`aios.<tenant>.*`) e verificação de que o `tenant` do recurso coincide
com o contexto autenticado — divergência retorna `403 AIOS-MEM-0005`. Meta:
**zero vazamento cross-tenant** (NFR-010). Ver [Security.md](./Security.md).

### 7.2 Onde ficam os dados sensíveis / PII?

Conteúdo sensível fica em PostgreSQL/MinIO sob RLS — **nunca** no envelope de
evento. Itens carregam `pii`, `legal_basis` e `retention_class`; PII é
redigida/tokenizada em logs e eventos (minimização, RFC-0001 §7). Erros RFC 7807
não vazam `detail` sensível.

---

## 8. Armadilhas comuns (resumo)

| Armadilha | Sintoma | Correção |
|-----------|---------|----------|
| Reusar `Idempotency-Key` nova a cada retry | itens duplicados | Reusar a MESMA key no reintento (§5.1) |
| Esperar durabilidade da Working Memory | dados somem | Usar Short-Term+ para durabilidade (§2.2) |
| `mode=graph` com AGE fora | `503 AIOS-MEM-0031` | Usar `hybrid` (degrada sem grafo) (§3.3) |
| Buscar item recém-criado por `vector` | não aparece | Aguardar *backfill* de embedding ou usar `lexical` (§3.4) |
| `kind` incompatível com `layer` | `422 AIOS-MEM-0020` | Alinhar `kind`↔`layer` (§2.1) |
| Purgar item em `legal_hold` | `423 AIOS-MEM-0050` | Só via RTBF autorizado (§4.4) |
| Runtime lendo PG/Redis direto | viola camadas | Usar API do MemoryService (§5.3) |
| Trocar `embedding.dim` a quente | inconsistência de índice | Requer reindex (§6.3) |

---

## 9. Referências

- [_DESIGN_BRIEF.md](./_DESIGN_BRIEF.md) — fonte única de verdade.
- [Examples.md](./Examples.md) — exemplos executáveis dos fluxos citados.
- [FailureRecovery.md](./FailureRecovery.md) — degradação e recuperação.
- [Scalability.md](./Scalability.md) — cotas, sharding, cold agents.
- [ADR.md](./ADR.md) / [RFC.md](./RFC.md) — decisões referenciadas.
- [Glossário](../040-Glossary/Glossary.md) — termos canônicos.
