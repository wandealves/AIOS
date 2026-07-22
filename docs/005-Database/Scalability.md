---
Documento: Scalability
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0051, ADR-0054, ADR-0055, ADR-0057, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050
Depende de: _DESIGN_BRIEF.md §10, 006-Kernel (shard_key), 027-Cluster, Benchmark.md
---

# 005-Database — Escalabilidade e Concorrência

Meta central: sustentar **≥ 10⁶ agentes** e **≥ 10⁹ linhas** em tabelas particionadas
com crescimento **sub-linear** de latência (NFR-007).

A tese do módulo em uma frase: **o *working set* é o presente, não o histórico**. Todo
o desenho de particionamento e retenção existe para manter o índice ativo pequeno
enquanto o volume acumulado cresce sem limite.

---

## 1. Estratégias por Dimensão

| Dimensão | Estratégia |
|----------|-----------|
| Particionamento temporal | Tabelas de série (`kernel.syscall_log`, `audit.entry`, `<schema>.outbox`) usam **RANGE por tempo**, com pré-criação (`precreate_ahead` = 7) e expurgo por *drop de partição* — custo de retenção O(1), sem *bloat*. |
| Particionamento por tenant | Tabelas multi-tenant de alta cardinalidade (`memory.item`, `context.window`) usam **HASH sobre `tenant_id`**, distribuindo *hot tenants* e limitando o tamanho de índice por partição. |
| Sharding lógico | Alinhado ao sharding do plano de controle (`shard = hash(tenant_id, agent_id) mod N`, cf. `../006-Kernel/_DESIGN_BRIEF.md` §10): `shard_key` é chave de partição/índice, permitindo migrar faixas de shard para outro cluster sem reescrever a aplicação. |
| Leitura vs. escrita | **Réplicas de streaming** absorvem leituras elegíveis sob controle de lag (NFR-013); escrita sempre no primário. |
| Concorrência de linha | **OCC por `version`**; sem locks pessimistas no caminho quente. `SELECT … FOR UPDATE` restrito a seções curtas e justificadas. |
| Conexões | PgBouncer em modo `transaction` com **bulkhead por serviço** — nenhum serviço esgota o pool alheio. |
| Busca vetorial | HNSW por default (latência estável, sem treino); IVFFlat para conjuntos muito grandes com tolerância a *build*; `ef_search` ajustável por tenant. |
| Grafo (AGE) | Grafo por tenant, com profundidade máxima de travessia limitada — consulta de custo explosivo é contida antes de executar. |
| Backpressure | Cota de conexões + `statement_timeout` + rejeição precoce (`AIOS-DB-0006`, `Retry-After`) em vez de degradar todos os tenants. |
| Manutenção sem downtime | `CREATE/REINDEX INDEX CONCURRENTLY`, `ADD COLUMN` sem default volátil, backfill em lotes — codificados como regras do `DdlConventionValidator`. |

---

## 2. Modelo de Particionamento

```
   Tabela lógica  memory.item                Tabela lógica  kernel.syscall_log
   ┌───────────────────────────────┐        ┌─────────────────────────────────┐
   │ HASH(tenant_id) mod 16        │        │ RANGE(created_at) por dia       │
   │  p00 p01 p02 … p15            │        │  d-0  d-1  d-2 … d-89           │
   └───────────────────────────────┘        └───────────┬─────────────────────┘
        │ índice ativo por partição                     │ drop O(1) em d-90
        ▼                                               ▼
   working set pequeno e estável            retenção sem DELETE, sem bloat
```

**Por que as duas estratégias coexistem.** Séries temporais têm *acesso recente e
expurgo por idade* — RANGE resolve ambos. Tabelas de entidade têm *acesso por chave e
distribuição desigual entre tenants* — HASH resolve o segundo sem prejudicar o primeiro.
Usar RANGE onde caberia HASH cria partições vazias; o inverso torna a retenção uma
varredura.

---

## 3. Curva de Escala Esperada

| Volume acumulado | Partições ativas (série) | Índice ativo | p99 leitura por chave |
|------------------|--------------------------|--------------|-----------------------|
| 10⁷ linhas | 7 dias | pequeno | ≤ 5 ms |
| 10⁸ linhas | 30 dias | moderado | ≤ 5 ms |
| 10⁹ linhas | 90 dias (retenção) | **estável** | ≤ 5 ms |
| > 10⁹ (histórico expirado) | 90 dias | **estável** | ≤ 5 ms |

O ponto essencial: da terceira linha em diante a curva **achata**, porque o expurgo por
`DROP` remove exatamente o que sairia do *working set*. Sem retenção por partição, a
mesma carga produziria índices que crescem indefinidamente e p99 que degrada com o
tempo — o modo de falha mais comum em plataformas de agentes.

---

## 4. Concorrência

| Mecanismo | Onde | Motivo |
|-----------|------|--------|
| OCC (`version`) | Todas as entidades mutáveis | Conflito é raro; retry é mais barato que lock (NFR-012). |
| `pg_advisory_lock` global | Aplicação de migração | Exclusão mútua em nível de cluster (invariante I1). |
| Transação curta | Todo caminho quente | `pool_mode=transaction` exige que a transação seja a unidade de posse da conexão. |
| Sem `LISTEN/NOTIFY` | — | Incompatível com pooling em modo `transaction`; eventos vão pelo Outbox/NATS. |
| Sem *prepared statement* nomeado de sessão | — | Mesma razão; verificado por T-CONN-03. |

Conflitos de OCC retornam `AIOS-DB-0004` (retriable) e são resolvidos por releitura +
reaplicação, com no máximo 5 tentativas.

---

## 5. Escala Horizontal

| Componente | Escala | Limite prático |
|------------|--------|----------------|
| `aios-database-svc` | Horizontal (stateless) | Sem limite relevante; o advisory lock serializa apenas migrações. |
| Réplicas de leitura | Horizontal | Custo de replicação cresce com o número de réplicas; ~5 é o ponto de retorno decrescente. |
| PgBouncer | Horizontal | Milhares de conexões lógicas por instância. |
| **Primário** | **Vertical** | **É o gargalo estrutural de escrita.** |

O primário é o limite arquitetural conhecido do módulo. O caminho de crescimento, em
ordem de preferência:

```
   1. Reduzir escrita        → retenção mais agressiva, telemetria amostrada
   2. Aliviar leitura        → mais réplicas, cache no módulo consumidor
   3. Escalar verticalmente  → mais CPU/RAM/IOPS no primário
   4. Dividir por shard      → clusters por faixa de shard_key (../027-Cluster/)
```

O passo 4 é viável **porque** `shard_key` já existe nas tabelas desde o início: a
divisão é uma operação de infraestrutura, não uma reescrita de aplicação. Essa é a
razão de exigir `shard_key` mesmo quando há um único cluster.

---

## 6. Backpressure

```
   consumidor ──▶ PgBouncer ──▶ PostgreSQL
        ▲             │              │
        │             │ fila cheia   │ statement_timeout
        │             ▼              ▼
        └── AIOS-DB-0006 ──── AIOS-DB-0007 ──▶ cliente reduz concorrência
             (Retry-After)      (aborta)
```

A rejeição é **precoce e explícita**: é preferível recusar 5% das requisições de um
serviço saturado a degradar o p99 de todos os tenants. `Retry-After` orienta o cliente
a recuar, e não a repetir imediatamente.

---

## 7. Teste de Escala (T-SCALE-01)

| Parâmetro | Valor |
|-----------|-------|
| Tenants | 1.000 (distribuição Zipf s = 1,1) |
| Agentes simulados | 10⁶ |
| Linhas em `kernel.syscall_log` | 10⁹ em 90 partições diárias |
| Vetores em `memory.item` | 10⁷ (1536 dim.) |
| Carga | mix da §3 de `./Benchmark.md`, 20k tx/s |
| Duração | 24 h |

**Critérios:** p99 de leitura por chave estável ao longo das 24 h; ciclo de retenção
sem pico de latência; *bloat* ≤ 20%; nenhuma partição futura faltante.

---

## 8. Limites Conhecidos e Mitigações

| Limite | Impacto | Mitigação |
|--------|---------|-----------|
| Escrita concentrada no primário | Teto global de tx/s | Ordem de crescimento da §5; sharding por faixa em `../027-Cluster/`. |
| `hash_modulus` fixo | Reparticionar exige migração de dados | Escolher potência de 2 com folga; documentar como decisão de longo prazo. |
| Índice HNSW cresce com o conjunto | RAM do primário | Particionar tabelas vetoriais por tenant; considerar IVFFlat acima de 10⁸ vetores. |
| Travessia AGE de custo exponencial | DoS acidental | `max_traversal_depth` = 6 + `statement_timeout`. |
| Modo `transaction` limita padrões de acesso | Erros sutis em runtime | Regra de lint + T-CONN-03 + `dev` também em modo `transaction`. |
| Retenção longa obrigatória (auditoria, 7 anos) | Volume que não pode ser expurgado | Partições antigas em armazenamento mais barato; `audit` é o único schema com esse perfil. |

---

## 9. Rumo a Milhões

A combinação que sustenta 10⁶ agentes:

1. **Partição temporal** mantém o índice ativo proporcional aos últimos dias.
2. **Partição por hash de tenant** impede que um *hot tenant* domine um índice.
3. **Retenção por `DROP`** torna o custo de expurgo independente do volume.
4. **Índice parcial no `outbox`** faz a varredura do relay ser O(pendentes).
5. **Réplicas de leitura** deslocam a maior parte do tráfego para fora do primário.
6. **`shard_key` presente desde o dia 1** deixa a divisão em clusters como opção
   disponível, não como reescrita.

Nenhum desses seis é uma otimização tardia: todos são propriedades do esquema, impostas
pelo `DdlConventionValidator` no momento em que a tabela é criada.

---

## 10. Referências

- Brief: `./_DESIGN_BRIEF.md` §10
- Modelo físico e particionamento: `./Database.md` §5
- Medição: `./Benchmark.md` · Falhas sob carga: `./FailureRecovery.md`
- Sharding do plano de controle: `../006-Kernel/` · Cluster: `../027-Cluster/`
