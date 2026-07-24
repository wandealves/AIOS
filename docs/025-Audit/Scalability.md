---
Documento: Scalability
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0002, ADR-0005, ADR-0251, ADR-0252, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250
Depende de: _DESIGN_BRIEF.md §10, NonFunctionalRequirements.md, Benchmark.md, Deployment.md
---

# 025-Audit — Escalabilidade

## 1. O problema

Com 10⁶ agentes, cada ação privilegiada gera um registro imutável. Um tenant grande
produz da ordem de **10⁹ registros por ano** — e nenhum deles pode ser descartado,
amostrado ou perdido.

O gargalo natural de uma trilha encadeada é o **encadeamento em si**: cada registro
precisa do hash do anterior, o que serializa a escrita.

```
   ❌ cadeia única global                  ✅ cadeia por partição
      todo append serializa                   16 cadeias independentes por tenant
      (a trilha vira o gargalo do AIOS)       selagem e verificação paralelas
```

## 2. Estratégias

| Dimensão | Estratégia |
|----------|-----------|
| Cadeia particionada | `seq` monotônico **por partição** (`aud.chain.partitions_per_tenant`, default 16), não globalmente. Uma cadeia única seria um ponto de serialização absoluto (ADR-0252). |
| Escolha de partição | `hash(actor, resource) mod N` — distribuição estável e afinidade de verificação: registros relacionados caem na mesma cadeia. |
| Concorrência de `seq` | Sequência dedicada por partição com *advisory lock* curto; a contenção é limitada a 1/N do tráfego. |
| Escrita durável | Commit síncrono com replicação (`remote_apply`); o custo é aceito por desenho (NFR-001, ~10 ms do orçamento). |
| Selagem | Paralela por partição; a árvore de Merkle é construída por intervalo, sem bloquear a escrita. |
| Verificação | Amostragem contínua barata + verificação completa em **réplica de leitura**, para não competir com a ingestão. |
| Particionamento físico | `audit.record` particionada por `RANGE(occurred_at)` **mensal**, além da partição lógica de cadeia. |
| Arquivamento | Partições seladas vão para objeto WORM; consultas antigas leem o arquivo, não o banco quente. |
| Consulta | Índices pelos dois eixos que importam: `subject_ref` (privacidade) e `decision_ref` (investigação). |
| Payload | O expurgo por retenção remove `payload_cipher` mantendo a linha: o volume decresce sem quebrar a cadeia. |
| Isolamento multi-tenant | RLS + partição lógica por tenant; um tenant volumoso não desloca a cadeia de outro. |
| Rumo a bilhões | 10⁶ agentes ⇒ ~10⁹ registros/ano por tenant grande. Sustentado por: cadeia particionada, selagem paralela, expurgo de payload, arquivamento do histórico frio e consulta indexada. |

## 3. Modelo de escala

```
   Camada 0 — PRODUTORES (outbox próprio)
   ┌──────────────────────────────────────────────────────────────┐
   │  absorvem indisponibilidade do 025; reenviam com o mesmo     │
   │  Idempotency-Key — é o que torna o RPO 0 possível             │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼ eventos + API
   Camada 1 — audit-ingest (3–10 réplicas, stateless)
   ┌──────────────────────────────────────────────────────────────┐
   │  escala linear em CPU (hash + cifragem)                       │
   │  MAS a contenção de seq é por PARTIÇÃO: N partições = N       │
   │  fluxos de escrita independentes                              │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼
   Camada 2 — PostgreSQL primário + réplica síncrona
   ┌──────────────────────────────────────────────────────────────┐
   │  o commit síncrono é o piso de latência (~10 ms)              │
   │  NÃO escala horizontalmente: é o limite real de throughput    │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼
   Camada 3 — selagem, verificação e arquivamento (assíncronos)
   ┌──────────────────────────────────────────────────────────────┐
   │  paralelos por partição; verificação em RÉPLICA DE LEITURA    │
   └──────────────────────────────────────────────────────────────┘
```

O limite real de vazão é a **camada 2**: nem réplicas de aplicação nem partições
adicionais superam a capacidade de commit síncrono do banco. Aumentar
`partitions_per_tenant` reduz contenção de `seq`, não o custo de replicação.

## 4. Concorrência e locks

| Ponto | Mecanismo | Por quê |
|-------|-----------|---------|
| Atribuição de `seq` | Sequência dedicada por partição + *advisory lock* curto | O `prev_hash` exige ordem **dentro** da partição; entre partições não há ordem a manter |
| Cálculo de hash | Sem lock (função pura) | Paralelizável por natureza |
| Escrita | `INSERT` apenas; sem `UPDATE` de linha existente | Elimina a classe inteira de conflitos de atualização |
| Transição de estado | `UPDATE` restrito a `state`/`seal_ref`/remoção de payload, sob *trigger* | Não colide com a escrita de novos registros |
| Selagem | Líder por partição (eleição) | Dois selos concorrentes sobre o mesmo intervalo produziriam raízes divergentes |
| Verificação | Réplica de leitura, sem lock | Não interfere na ingestão |
| Expurgo por retenção | Lote com verificação de *hold* por registro | Prioriza correção sobre velocidade |
| Outbox | Líder único | Evita publicação duplicada |

## 5. Limites teóricos e observados

| Dimensão | Limite de projeto | Fator limitante | Ação ao aproximar |
|----------|-------------------|-----------------|-------------------|
| Registros/s por réplica | 50.000 (NFR-002) | CPU (hash + cifragem) | HPA por CPU |
| Registros/s por tenant | ~200.000 | **Commit síncrono do banco** | Particionar o banco por tenant; revisar `synchronous_commit` **jamais** |
| Partições por tenant | 256 (`aud.chain.partitions_per_tenant`) | Complexidade de verificação | Aumentar exige **migração** (a sequência é por partição) |
| Registros por tenant | 10⁹ (NFR-009) | Índices e volume de payload | Expurgo por retenção; arquivamento WORM |
| Verificação completa | 10⁹ em ≤ 4 h (NFR-008) | I/O da réplica de leitura | Paralelizar por partição |
| Consulta | 100.000 registros (`aud.query.max_records`) | I/O e decifragem | Refinar filtro; usar `decision_ref`/`subject_ref` |
| Exportação | 10⁶ registros (NFR-013) | Montagem de provas de Merkle | Fatiar por janela |
| Payload ativo | — | Disco | `retention_days` por classe; o expurgo mantém a linha |

### 5.1 Onde a escala realmente dói

O limite prático **não** é o número de registros — é a combinação de três coisas:

1. **Commit síncrono** (camada 2): o piso de ~10 ms por transação. Mitigação: lote na
   API (`batch_max_size = 500`), que amortiza a replicação por 500 registros.
2. **Volume de payload cifrado**: 10⁹ registros × payload médio domina o disco.
   Mitigação: retenção por classe e expurgo que **mantém a linha** e remove só o
   conteúdo (`./Database.md` §11.1).
3. **Verificação completa**: recomputar 10⁹ hashes é I/O-bound. Mitigação: réplica de
   leitura dedicada, paralelismo por partição e amostragem contínua entre as
   verificações completas.

Nenhum dos três se resolve adicionando réplicas de aplicação.

## 6. Rumo a bilhões de registros

```
   10⁶ agentes ──▶ ~10⁹ registros/ano por tenant grande
        │
        ├── cadeia particionada     → N fluxos de escrita independentes
        ├── lote na API             → amortiza o commit síncrono
        ├── selagem paralela        → não bloqueia a escrita
        ├── expurgo de payload      → volume decresce, cadeia permanece
        ├── arquivamento WORM       → histórico frio sai do banco quente
        └── verificação amostrada   → detecção contínua sem custo total
```

Antipadrões que quebram a escala:

| Antipadrão | Efeito em escala | Alternativa |
|-----------|------------------|-------------|
| Cadeia única global por tenant | Serializa **toda** a escrita do tenant | `partitions_per_tenant` ≥ 16 |
| Escrita registro a registro na API | Cada um paga o commit síncrono completo | `AppendBatch` (500) |
| Payload gordo (dump completo do objeto) | Disco domina; consulta lenta | Payload com o mínimo para provar a ação (NR-10) |
| Retenção uniforme longa para todas as classes | Custo linear na classe de maior volume | `retention_days` **por classe** |
| Verificação completa contínua no primário | Compete com a ingestão | Amostragem + réplica de leitura |
| Consulta sem `subject_ref`/`decision_ref` | Varredura de partições inteiras | Índices dos dois eixos; janela temporal |
| Aumentar partições sem migração | Sequência quebra a partir do ponto de mudança | Chave **não-recarregável**; exige migração |
| Relaxar `synchronous_commit` para ganhar vazão | **Destrói o RPO 0** — a garantia que define o módulo | Particionar o banco; nunca relaxar |

> O último antipadrão é o mais tentador durante um pico: reduzir a durabilidade
> multiplica a vazão imediatamente. É também o único item desta tabela que transforma
> um problema de desempenho em **perda irreversível de prova** — e por isso
> `aud.db.synchronous_commit` é validado no *startup* (`./Configuration.md` §9, V-05).

## 7. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §10
- Medição: `./Benchmark.md` · Metas: `./NonFunctionalRequirements.md`
- Implantação e HPA: `./Deployment.md` §2.1 · Falhas: `./FailureRecovery.md`
- Modelo físico e retenção: `./Database.md` §11 · Plataforma: `../027-Cluster/`

*Fim de `Scalability.md`.*
