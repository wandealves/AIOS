---
Documento: ADR-0005 — PostgreSQL + pgvector + Apache AGE como núcleo de persistência
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee
Módulos afetados: 001, 005, 010, 018, 019, 025
---

# ADR-0005: PostgreSQL + pgvector + Apache AGE como núcleo de persistência

## Contexto
O AIOS precisa de três modelos de dados: relacional (registro, tarefas,
auditoria), vetorial (memória semântica/embeddings) e grafo (conhecimento/
GraphRAG). Múltiplos bancos aumentam peças móveis, custo operacional e problemas
de consistência entre lojas.

## Problema
Adotar bancos especializados separados ou unificar em PostgreSQL com extensões?

## Alternativas
1. **Poliglota**: PostgreSQL (relacional) + vector DB dedicado (Milvus/Weaviate/Qdrant) + graph DB dedicado (Neo4j).
2. **PostgreSQL unificado**: `pgvector` (vetorial) + `Apache AGE` (grafo openCypher).
3. **PostgreSQL + apenas pgvector**, grafo emulado em tabelas.

## Análise
| Critério | Poliglota | PG + pgvector + AGE | PG + pgvector só |
|----------|-----------|---------------------|-------------------|
| Consistência entre modelos | Difícil (2PC/sync) | Transacional (mesmo DB) | Transacional |
| Escala vetorial (bi de vetores) | Excelente | Boa (limite) | Boa (limite) |
| Consultas grafo (Cypher) | Excelente (Neo4j) | Boa (AGE) | Fraca |
| Peças operacionais | Muitas | Uma | Uma |
| Backup/DR unificado | Não | Sim | Sim |
| Maturidade | Alta | Média | Alta |

Unificar reduz drasticamente complexidade e garante transações ACID entre os três
modelos no MVP. Escala extrema de vetores é o principal limite — endereçado por um
caminho de migração (índice vetorial externo) sem mudar a API de Memory (010).

## Escolha
Adotamos **Opção 2**: **PostgreSQL** como fonte da verdade, com **pgvector** para
memória vetorial e **Apache AGE** para grafo de conhecimento. Bancos dedicados
permanecem como *estratégia de escala futura*, atrás da API estável de Memory/
Knowledge.

## Consequências
**Positivas:** uma stack de dados; transações ACID entre relacional/vetorial/grafo;
backup/DR unificado; RLS multi-tenant.
**Negativas:** limites de escala vetorial/grafo em volumes extremos; tuning de
índices ANN (HNSW/IVFFlat) exigido.
**Neutras:** DDL e migrações centralizadas em `005-Database`.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Escala vetorial bilionária | Média | Alto | API estável + caminho p/ índice externo (010/Scalability). |
| Contenção no DB único | Média | Alto | Read-replicas, particionamento, CQRS, Redis à frente. |
| AGE menos maduro que Neo4j | Média | Médio | Consultas Cypher restritas/testadas; fallback tabular. |

## Trade-offs
Ganhamos simplicidade operacional e consistência transacional; abrimos mão do teto
de escala e da maturidade de engines especializadas (recuperáveis via migração).

## Referências
`../001-Architecture/Architecture.md` §8, §14; `../005-Database/`; `../010-Memory/`; `../019-GraphRAG/`.
