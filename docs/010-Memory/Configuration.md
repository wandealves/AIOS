---
Documento: Configuration
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0103, ADR-0104, ADR-0105, ADR-0106
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 003-RFC (RFC-0001), 005-Database, 017-Model-Router, 026-Cost-Optimizer, 028-Deployment, 040-Glossary
---

# 010-Memory — Configuration

> Todas as chaves de configuração do `MemoryService`. **Deriva** da §8 do
> `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**. Namespace reservado: `memory.*`
> (brief §0). Palavras normativas conforme RFC 2119/8174.

## 1. Modelo de configuração

- **Namespace:** todas as chaves vivem sob `memory.*`.
- **Escopo (precedência crescente):** `global (G)` → `por tenant (T)` → `por agente (A)`.
  O valor mais específico prevalece; escopos `A` só se aplicam a chaves marcadas `A`.
- **Recarga (hot-reload):** chaves marcadas *Recarregável = sim* PODEM ser alteradas
  em runtime e entram em vigor sem reinício. Chaves *não* recarregáveis exigem
  **reindexação** (índice HNSW/embedding) ou reinício controlado — DEVEM ser tratadas
  como mudanças de migração (ver [Database.md](./Database.md) e [Deployment.md](./Deployment.md)).
- **Fonte:** valores default vêm do config server do plano de controle; overrides por
  tenant/agente ficam em `MemoryLayerConfig`/`MemoryQuota` (brief §3.2), recarregáveis.
- **Variáveis de ambiente:** cada chave mapeia para `AIOS_MEMORY_...` em maiúsculas,
  com `.` → `_` (ver §3). Env vars sobrepõem o default global no bootstrap.

```
 Precedência de resolução (maior vence):
   default (código) ◀ env AIOS_MEMORY_* ◀ config global (G) ◀ override tenant (T) ◀ override agente (A)
```

## 2. Catálogo de chaves (`memory.*`)

Tabela canônica derivada da §8 do brief. Tipos: `int`, `float`, `bool`, `duration`
(sufixos `s/h/d`), `bytes` (`KiB/MiB/GiB`), `enum`, `urn`, `string`.

| Chave | Tipo | Default | Faixa | Escopo | Recarregável |
|-------|------|---------|-------|--------|--------------|
| `memory.embedding.dim` | int | `1024` | 256–4096 | G | **não** (reindex) |
| `memory.embedding.model` | urn | `text-embed-default` | URN de modelo | T | sim |
| `memory.item.inline_max_bytes` | bytes | `65536` (64 KiB) | 1 KiB–1 MiB | T | sim |
| `memory.hnsw.m` | int | `16` | 8–64 | G/T | **não** (reindex) |
| `memory.hnsw.ef_construction` | int | `128` | 32–512 | G | **não** (reindex) |
| `memory.hnsw.ef_search` | int | `64` | 16–512 | T | sim |
| `memory.recall.default_k` | int | `10` | 1–200 | T/A | sim |
| `memory.recall.min_score` | float | `0.30` | 0.0–1.0 | T/A | sim |
| `memory.recall.fusion` | enum | `rrf` | `rrf\|weighted` | T | sim |
| `memory.layer.working.ttl` | duration | `900s` | 60s–3600s | T/A | sim |
| `memory.layer.short_term.ttl` | duration | `86400s` | 1h–7d | T/A | sim |
| `memory.layer.long_term.ttl` | duration | `90d` | 7d–730d / ∞ | T | sim |
| `memory.layer.episodic.ttl` | duration | `365d` | 30d–1825d / ∞ | T | sim |
| `memory.decay.half_life` | duration | `30d` | 1d–365d | T/A | sim |
| `memory.decay.threshold` | float | `0.15` | 0.0–1.0 | T/A | sim |
| `memory.quota.max_items` | int | `1000000` | ≥ 0 | T/A/layer | sim |
| `memory.quota.max_bytes` | bytes | `10GiB` | ≥ 0 | T/A/layer | sim |
| `memory.consolidation.threshold` | int | `500` | ≥ 1 | T/A | sim |
| `memory.consolidation.auto` | bool | `true` | bool | T | sim |
| `memory.consolidation.version.retention` | int | `10` | 1–100 | T | sim |
| `memory.forget.grace_period` | duration | `7d` | 0–90d | T | sim |
| `memory.rtbf.enabled` | bool | `true` | bool | T | sim |
| `memory.backpressure.max_inflight` | int | `1000` | 10–100000 | G | sim |
| `memory.blob.bucket` | string | `aios-memory` | nome S3 | G | **não** |

### 2.1 Grupos funcionais

- **Embeddings/índice vetorial** (`memory.embedding.*`, `memory.hnsw.*`): governam a
  geração via [017-Model-Router](../017-Model-Router/API.md) e o índice HNSW/pgvector.
  `dim`, `m`, `ef_construction` **não** são recarregáveis pois alteram o índice físico
  (exigem reindex online, brief §10; `ADR-0101`).
- **Recuperação** (`memory.recall.*`): defaults de `k`, `min_score` e estratégia de
  fusão (`ADR-0104`). Todos recarregáveis; overrides por agente permitem *tuning* fino.
- **Camadas/TTL** (`memory.layer.*.ttl`): TTL efetivo por camada (mapa da §3.3 do brief).
- **Decaimento/esquecimento** (`memory.decay.*`, `memory.forget.*`, `memory.rtbf.*`):
  políticas de esquecimento controlado (`ADR-0103`).
- **Cotas/backpressure** (`memory.quota.*`, `memory.backpressure.*`): controle de
  crescimento e admissão (`ADR-0105`), reporta a [026-Cost-Optimizer](../026-Cost-Optimizer/Metrics.md).
- **Blobs** (`memory.blob.*`): externalização MinIO (`ADR-0106`).

## 3. Variáveis de ambiente (`AIOS_MEMORY_...`)

Mapeamento determinístico: prefixo `AIOS_MEMORY_`, restante em maiúsculas com `.` → `_`.
Aplicam-se no bootstrap (escopo global); overrides por tenant/agente NÃO passam por env.

| Chave | Variável de ambiente | Exemplo |
|-------|----------------------|---------|
| `memory.embedding.dim` | `AIOS_MEMORY_EMBEDDING_DIM` | `1024` |
| `memory.embedding.model` | `AIOS_MEMORY_EMBEDDING_MODEL` | `urn:aios:_:model:text-embed-default` |
| `memory.item.inline_max_bytes` | `AIOS_MEMORY_ITEM_INLINE_MAX_BYTES` | `65536` |
| `memory.hnsw.m` | `AIOS_MEMORY_HNSW_M` | `16` |
| `memory.hnsw.ef_construction` | `AIOS_MEMORY_HNSW_EF_CONSTRUCTION` | `128` |
| `memory.hnsw.ef_search` | `AIOS_MEMORY_HNSW_EF_SEARCH` | `64` |
| `memory.recall.default_k` | `AIOS_MEMORY_RECALL_DEFAULT_K` | `10` |
| `memory.recall.min_score` | `AIOS_MEMORY_RECALL_MIN_SCORE` | `0.30` |
| `memory.recall.fusion` | `AIOS_MEMORY_RECALL_FUSION` | `rrf` |
| `memory.backpressure.max_inflight` | `AIOS_MEMORY_BACKPRESSURE_MAX_INFLIGHT` | `1000` |
| `memory.blob.bucket` | `AIOS_MEMORY_BLOB_BUCKET` | `aios-memory` |
| `memory.rtbf.enabled` | `AIOS_MEMORY_RTBF_ENABLED` | `true` |

> Segredos (credenciais Redis/PostgreSQL/MinIO/NATS) NÃO DEVEM ser passados por estas
> chaves de domínio; usam o cofre de segredos descrito em [Security.md](./Security.md)
> e [Deployment.md](./Deployment.md).

## 4. Exemplo de configuração (YAML)

Exemplo de overlay por tenant (recarregável) mais defaults globais:

```yaml
memory:
  embedding:
    dim: 1024                 # global; reindex se alterado
    model: urn:aios:acme:model:text-embed-3-large   # override do tenant
  item:
    inline_max_bytes: 131072  # 128 KiB para 'acme'
  hnsw:
    m: 16
    ef_construction: 128
    ef_search: 96             # tenant privilegia recall sobre latência
  recall:
    default_k: 10
    min_score: 0.35
    fusion: rrf
  layer:
    working:   { ttl: 900s }
    short_term:{ ttl: 86400s }
    long_term: { ttl: 90d }
    episodic:  { ttl: 365d }
  decay:
    half_life: 30d
    threshold: 0.15
  quota:
    max_items: 2000000        # cota elevada para 'acme'
    max_bytes: 20GiB
  consolidation:
    threshold: 500
    auto: true
    version:
      retention: 10
  forget:
    grace_period: 7d
  rtbf:
    enabled: true
  backpressure:
    max_inflight: 1000
  blob:
    bucket: aios-memory
```

Bloco equivalente por variáveis de ambiente (bootstrap global):

```bash
AIOS_MEMORY_EMBEDDING_DIM=1024
AIOS_MEMORY_HNSW_M=16
AIOS_MEMORY_HNSW_EF_SEARCH=64
AIOS_MEMORY_RECALL_DEFAULT_K=10
AIOS_MEMORY_RECALL_MIN_SCORE=0.30
AIOS_MEMORY_BACKPRESSURE_MAX_INFLIGHT=1000
AIOS_MEMORY_BLOB_BUCKET=aios-memory
AIOS_MEMORY_RTBF_ENABLED=true
```

## 5. Validação, hot-reload e efeitos

| Chave | Validação | Efeito ao alterar |
|-------|-----------|-------------------|
| `memory.embedding.dim` | inteiro na faixa; DEVE casar com a dimensão do índice HNSW | **Reindex obrigatório**; incompatibilidade em escrita → `AIOS-MEM-0021`. |
| `memory.hnsw.m` / `ef_construction` | faixa | Reindex online (brief §10); sem downtime. |
| `memory.hnsw.ef_search` | faixa | Hot-reload; troca custo/qualidade de recall imediatamente. |
| `memory.recall.min_score` | 0.0–1.0 | Hot-reload; filtra resultados abaixo do limiar. |
| `memory.quota.max_items/bytes` | ≥ 0 | Hot-reload; reduzir abaixo do uso corrente dispara poda (ForgettingEngine) e `AIOS-MEM-0010`. |
| `memory.consolidation.auto` | bool | Hot-reload; habilita/desabilita jobs automáticos por threshold. |
| `memory.forget.grace_period` | 0–90d | Hot-reload; afeta janela `FORGOTTEN→PURGED`. |
| `memory.rtbf.enabled` | bool | Hot-reload; `false` recusa novos RTBF (não afeta os já agendados). |

**Regras normativas:**
- Uma alteração de chave **não recarregável** DEVE ser rejeitada em runtime e tratada
  como migração planejada.
- Reduzir uma cota abaixo do uso corrente NÃO DEVE apagar dados imediatamente; DEVE
  agendar poda respeitando `retention_class`/`legal_hold` (ver [Security.md](./Security.md)).
- Todo hot-reload DEVE emitir log estruturado com `tenant_id`/`trace_id` e métrica de
  auditoria de configuração (ver [024-Observability](../024-Observability/Logging.md)).

## 6. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-----------------|--------------------------|
| Alterar `embedding.dim` sem reindex quebra buscas. | Chave não recarregável; validação de dimensão em escrita (`AIOS-MEM-0021`); procedimento de reindex documentado. |
| Overrides por agente proliferam e dificultam auditoria. | Precedência explícita; overrides persistidos e versionados; defaults sãos no escopo G. |
| Cota reduzida causa perda inadvertida. | Poda gradual respeitando retenção legal; nunca purga `legal_hold` sem RTBF. |
| `ef_search` alto degrada latência (NFR-003). | Faixa limitada (16–512); default conservador; tuning por tenant com monitoramento. |

Decisões governadas por `ADR-0101` (embeddings/índice), `ADR-0103` (esquecimento),
`ADR-0104` (fusão), `ADR-0105` (cotas) e `ADR-0106` (blobs) — ver [ADR.md](./ADR.md).
