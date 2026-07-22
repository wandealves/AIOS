---
Documento: Benchmark
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0005, ADR-0054, ADR-0057, ADR-0058
RFCs relacionados: RFC-0001
Depende de: NonFunctionalRequirements.md, Testing.md, Scalability.md, 035-Benchmark
---

# 005-Database — Benchmark

Metodologia e KPIs para verificar os NFR de desempenho e escala. Os alvos são os de
`./NonFunctionalRequirements.md` — este documento define **como medi-los**, não os
redefine. A metodologia geral do projeto está em `../035-Benchmark/`.

---

## 1. Ambiente de Referência

| Item | Especificação |
|------|---------------|
| Primário | 16 vCPU, 64 GiB RAM, NVMe local (≥ 100k IOPS aleatório), PostgreSQL 16 |
| Réplicas | 2 × 8 vCPU, 32 GiB RAM, mesma classe de disco, streaming assíncrono + 1 síncrona |
| Pooling | PgBouncer, `pool_mode=transaction`, `default_pool_size=50` |
| Gerador de carga | 4 nós, 8 vCPU cada, na mesma rede (RTT < 0,5 ms) |
| PostgreSQL | `shared_buffers=16GiB`, `effective_cache_size=40GiB`, `synchronous_commit=on` |
| Extensões | `pgvector` (HNSW), `age` |

Toda medição é feita **após aquecimento de 10 min** e descarta o primeiro decil de
amostras. Resultados sem o ambiente descrito **NÃO DEVEM** ser comparados com estes
alvos.

---

## 2. KPIs e Alvos

| KPI | Alvo | NFR | Cenário |
|-----|------|-----|---------|
| p99 leitura por chave | ≤ **5 ms** | NFR-001 | C-1 |
| p99 escrita de linha única | ≤ **15 ms** | NFR-002 | C-1 |
| Throughput sustentado | ≥ **20.000 tx/s** | NFR-004 | C-2 |
| p99 busca vetorial top-10 (10⁷ vetores) | ≤ **50 ms** | NFR-003 | V-1 |
| Recall@10 | ≥ **0,95** | NFR-006 | V-2 |
| Degradação com 10⁹ linhas históricas | sub-linear | NFR-007 | S-1 |
| Bloqueio por migração `expand` | ≤ **200 ms** | NFR-010 | M-1 |
| Lag de replicação p95 | ≤ **500 ms** | NFR-013 | R-1 |
| Bloat após ciclo de retenção | ≤ **20%** | NFR-016 | S-2 |
| RTO de restauração PITR | ≤ **15 min** | NFR-009 | D-1 |

---

## 3. Perfil de Carga Sintética AIOS

A carga não é `pgbench` puro: ela reproduz o padrão real do AIOS, dominado por
leitura por chave, escrita pequena e busca vetorial.

| Operação | Peso | Tabela representativa |
|----------|------|-----------------------|
| Leitura de ACB por URN | 35% | `kernel.acb` |
| Escrita de `syscall_log` | 20% | `kernel.syscall_log` (particionada por dia) |
| Recuperação de memória (vetorial top-10) | 15% | `memory.item` |
| Escrita de item de memória + embedding | 10% | `memory.item` |
| Leitura de janela de contexto | 10% | `context.window` |
| Inserção em `outbox` + leitura do relay | 5% | `<schema>.outbox` |
| Consultas de catálogo/admin | 5% | `platform.*` |

Distribuição de tenants: **Zipf (s = 1,1)** sobre 1.000 tenants — reproduz o cenário
realista de *hot tenants*, que é justamente onde o particionamento por hash prova (ou
não) seu valor.

---

## 4. Cenários OLTP

### C-1 — Latência sob carga moderada
- Carga: 5.000 tx/s, mix da §3, 30 min.
- Mede: p50/p95/p99 por `kind` (`pk`, `write`, `scan`).
- **Critério:** p99 pk ≤ 5 ms; p99 write ≤ 15 ms.

### C-2 — Throughput máximo sustentável
- Rampa: 5k → 30k tx/s em degraus de 5k, 10 min por degrau.
- Mede: taxa de erro, p99, saturação de conexões, CPU do primário.
- **Critério:** ≥ 20.000 tx/s com p99 dentro de C-1 e erro < 0,1%.

### C-3 — Contenção entre serviços (bulkhead)
- Serviço A satura seu pool (100% de uso) enquanto B mantém carga nominal.
- **Critério:** p99 de B **não degrada mais que 10%**; A recebe `AIOS-DB-0006` em vez
  de drenar o pool global.

---

## 5. Cenários Vetoriais

### V-1 — Latência de busca
- Conjunto: 10⁷ vetores de 1536 dimensões, índice HNSW (`m=16`, `ef_construction=64`).
- Carga: 500 buscas/s top-10, `ef_search=64`.
- **Critério:** p99 ≤ 50 ms.

### V-2 — Recall vs. latência
- Varredura de `ef_search` ∈ {32, 64, 128, 256}, comparando com busca exata em amostra
  de 10.000 consultas.

| `ef_search` | Recall@10 esperado | p99 esperado | Uso recomendado |
|-------------|--------------------|--------------|-----------------|
| 32 | ~0,90 | ~20 ms | Não atende NFR-006 |
| **64** | **≥ 0,95** | **≤ 50 ms** | **Default** |
| 128 | ~0,98 | ~85 ms | Tenant com exigência de qualidade |
| 256 | ~0,99 | ~160 ms | Avaliação offline |

- **Critério:** o default (`64`) satisfaz simultaneamente NFR-003 e NFR-006.

### V-3 — Custo de reindexação online
- `REINDEX CONCURRENTLY` sobre 10⁷ vetores sob carga de escrita.
- **Critério:** nenhum bloqueio de escrita; degradação de p99 de leitura ≤ 25% durante
  a janela.

---

## 6. Cenários de Escala

### S-1 — Histórico profundo
- `kernel.syscall_log` com 10⁹ linhas em 90 partições diárias.
- Mede: p99 de leitura por chave e de varredura no período recente.
- **Critério:** p99 de leitura por chave **não cresce** com o histórico (o *working
  set* é o presente, não o passado).

### S-2 — Retenção sob carga
- Ciclo de retenção (`DROP` de 3 partições) durante carga de 10k tx/s.
- **Critério:** duração do `DROP` < 1 s por partição; p99 sem pico; `bloat` ≤ 20%.

### S-3 — Hot tenant
- Um tenant com 30% do volume total, tabelas com `hash_modulus = 16`.
- **Critério:** a partição do *hot tenant* não degrada as demais em mais de 15%.

---

## 7. Cenários de Migração e Replicação

### M-1 — Custo de DDL
| Operação | Bloqueio esperado | Aceito? |
|----------|-------------------|---------|
| `ADD COLUMN` sem default volátil | < 10 ms | sim |
| `CREATE INDEX CONCURRENTLY` (10⁸ linhas) | ~0 (sem bloqueio de escrita) | sim |
| `ADD COLUMN … DEFAULT <volátil>` | reescrita completa da tabela | **não** — reescrever como expand/contract |
| `ALTER COLUMN TYPE` | reescrita completa | **não** — nova coluna + backfill |
| `DROP COLUMN` (fase `contract`) | < 10 ms | sim, fora da janela de rollout |

- **Critério:** toda migração aceita mede ≤ 200 ms de bloqueio (NFR-010).

### R-1 — Lag sob escrita intensa
- 20k escritas/s por 20 min, medindo lag por réplica.
- **Critério:** p95 ≤ 500 ms; nenhuma réplica sai do pool de leitura em regime normal.

---

## 8. Cenário de Recuperação

### D-1 — PITR completo
- Base de 500 GiB, restauração a partir do último full + replay de WAL.
- Mede: tempo até o cluster aceitar escrita (RTO) e o intervalo de dados perdido (RPO).
- **Critério:** **RTO ≤ 15 min**, **RPO ≤ 5 min** (alvo operacional ≤ 1 min).

---

## 9. Reprodutibilidade

| Requisito | Detalhe |
|-----------|---------|
| Versionamento | Scripts de carga versionados junto às migrações do módulo. |
| Sementes fixas | Geração de dados e distribuição Zipf com semente registrada. |
| Isolamento | Nenhuma outra carga no cluster durante a medição. |
| Registro | Cada execução publica: versão do PostgreSQL/extensões, configuração efetiva, hash do script, resultados brutos. |
| Comparação | Regressão > 10% em qualquer KPI **DEVE** bloquear a promoção da versão. |

---

## 10. Baseline e Histórico

| Versão | Data | p99 pk | tx/s | p99 vetorial | Recall@10 | Observação |
|--------|------|--------|------|--------------|-----------|------------|
| 0.1 (alvo) | — | ≤ 5 ms | ≥ 20.000 | ≤ 50 ms | ≥ 0,95 | Metas de projeto; primeira medição pendente de ambiente de referência. |

A tabela acima é preenchida a cada release e serve de baseline para detectar regressão.

---

## 11. Referências

- Metas: `./NonFunctionalRequirements.md` · Testes: `./Testing.md`
- Escala: `./Scalability.md` · Metodologia global: `../035-Benchmark/`
- Modelo físico e índices: `./Database.md`
