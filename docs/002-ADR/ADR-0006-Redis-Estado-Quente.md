---
Documento: ADR-0006 — Redis para estado quente, locks e rate-limiting
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee
Módulos afetados: 001, 009, 010, 011, 021
---

# ADR-0006: Redis para estado quente, locks distribuídos e rate-limiting

## Contexto
Caminhos quentes (decisão de scheduling, ACB ativo, rate-limit no gateway, cache
de contexto/memória, locks distribuídos) exigem latência sub-milissegundo, incompatível
com ida ao PostgreSQL a cada operação.

## Problema
Como servir estado quente e primitivas de coordenação com latência sub-ms sem
sobrecarregar o PostgreSQL?

## Alternativas
1. **PostgreSQL para tudo** (sem camada quente).
2. **Redis** como cache/estado quente/locks/rate-limit.
3. **Cache in-process** por serviço (sem store compartilhado).
4. **Memcached** (cache puro).

## Análise
| Critério | Só PG | Redis | In-process | Memcached |
|----------|-------|-------|------------|-----------|
| Latência | ms+ | sub-ms | sub-ms | sub-ms |
| Compartilhado entre réplicas | Sim | Sim | Não | Sim |
| Locks distribuídos | Difícil | Sim (Redlock) | Não | Não |
| Estruturas ricas (ZSET, streams) | — | Sim | — | Não |
| Rate-limit | Difícil | Nativo | Inconsistente | Limitado |
| Persistência opcional | Sim | Sim (AOF/RDB) | Não | Não |

Cache in-process não serve estado compartilhado entre réplicas stateless.
Memcached não tem locks/estruturas. Redis cobre cache, locks, filas rápidas,
rate-limit e leaderboards de prioridade (ZSET) com latência sub-ms.

## Escolha
Adotamos **Redis** como camada de estado quente: cache de leitura, ACB quente,
rate-limiting, locks distribuídos e filas de prioridade do Scheduler. PostgreSQL
permanece a **fonte da verdade**; Redis é derivável/reconstruível.

## Consequências
**Positivas:** latência sub-ms no caminho quente; primitivas de coordenação
prontas; alivia o PostgreSQL.
**Negativas:** mais um sistema stateful para HA; risco de inconsistência
cache↔DB.
**Neutras:** Redis é tratado como cache/estado derivável, nunca fonte única.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Inconsistência cache/DB | Média | Médio | TTL + invalidação por evento; write-through onde crítico. |
| Perda de dados voláteis | Média | Baixo | Estado reconstruível do PG; AOF onde necessário. |
| Falha do Redis primário | Baixa | Alto | Redis Sentinel/Cluster; réplica por AZ (027). |

## Trade-offs
Ganhamos latência e primitivas de coordenação; pagamos com mais um componente
stateful e a disciplina de invalidação de cache.

## Referências
`../001-Architecture/Architecture.md` §9; `../009-Scheduler/`; `../011-Context/`.
