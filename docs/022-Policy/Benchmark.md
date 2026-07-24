---
Documento: Benchmark
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0222, ADR-0223, ADR-0226
RFCs relacionados: RFC-0001
Depende de: NonFunctionalRequirements.md, Metrics.md, Testing.md, Scalability.md
---

# 022-Policy — Benchmark

## 1. Objetivo

Medir, de forma reprodutível, se o PDP sustenta as metas de `./NonFunctionalRequirements.md`
— em especial NFR-001 (p99 ≤ 5 ms), NFR-002 (≥ 50.000 decisões/s por réplica),
NFR-004 (propagação ≤ 10 s), NFR-005 (escala) e NFR-012 (simulação de 10⁶ em ≤ 10 min).

O benchmark existe para responder a uma pergunta específica: **o monitor de referência
cabe no caminho quente de toda ação privilegiada do AIOS?** Se não couber, a
consequência não é lentidão — é a tentação de criar caminhos que o contornem.

## 2. Ambiente de referência

| Item | Especificação |
|------|---------------|
| Nó de aplicação | 8 vCPU, 16 GiB, SSD NVMe |
| Réplicas medidas | 1 (isolada) para NFR-002; 3 para propagação |
| PostgreSQL | 16, 4 vCPU, 16 GiB, dedicado |
| Redis | 7, 2 vCPU, 4 GiB |
| NATS/JetStream | 3 nós, 2 vCPU cada |
| Rede | RTT intra-cluster ≤ 0,3 ms |
| Gerador de carga | k6 (gRPC), nó separado |
| Bundle | `bundle-realistic` (800 regras, 40 papéis, 5.000 vínculos) salvo indicação |

Toda medição descarta os primeiros 60 s (aquecimento do JIT e dos caches) e reporta
mediana de 5 execuções.

## 3. Cargas e KPIs

| # | Carga | Perfil | KPI | Alvo |
|---|-------|--------|-----|------|
| B-01 | Decisão, atributos em cache | 100% cache hit; 70% allow / 30% deny | p50/p95/**p99** | p99 ≤ **5 ms** |
| B-02 | Decisão, atributo remoto | 50% exige `budget.*` (latência 5 ms) | p99 | ≤ **15 ms** |
| B-03 | Vazão máxima por réplica | Rampa até saturação de CPU | decisões/s a p99 ≤ 5 ms | ≥ **50.000/s** |
| B-04 | Lote | `EvaluateBatch` de 64 | decisões/s | ≥ 1,5× do unitário |
| B-05 | Escala de bundle | 10³ → 10⁴ regras | p99 vs. nº de regras | crescimento **sub-linear** |
| B-06 | Escala de vínculos | 10³ → 10⁵ vínculos | p99 do `RbacResolver` | ≤ 1,5 ms |
| B-07 | Propagação | Publicação com 3 réplicas | tempo até convergência | ≤ **10 s** |
| B-08 | Simulação | 10⁶ decisões do `journal-1M` | duração; impacto no p99 de produção | ≤ **10 min**; impacto ≤ 5% |
| B-09 | Compilação | `bundle-stress` (10⁴ regras) | duração da compilação | ≤ 30 s |
| B-10 | Regra patológica | Condições de custo alto | % dentro do orçamento | 100% ≤ 50 ms ou `deny` |

## 4. Resultados-alvo (baseline de referência)

| KPI | Alvo | Baseline esperado | Limite de regressão |
|-----|------|-------------------|---------------------|
| B-01 p99 | ≤ 5 ms | ~3,2 ms | +10% |
| B-01 p50 | — | ~0,9 ms | +15% |
| B-02 p99 | ≤ 15 ms | ~11 ms | +10% |
| B-03 vazão | ≥ 50.000/s | ~62.000/s | −10% |
| B-05 p99 @ 10⁴ regras | ≤ 5 ms | ~4,1 ms | +10% |
| B-06 p99 RBAC @ 10⁵ vínculos | ≤ 1,5 ms | ~1,1 ms | +15% |
| B-07 convergência | ≤ 10 s | ~2,5 s | +50% |
| B-08 duração | ≤ 10 min | ~6 min | +20% |
| B-09 compilação | ≤ 30 s | ~12 s | +25% |

Regressão além do limite **bloqueia o merge** (`./Testing.md` §8).

## 5. Metodologia da simulação (B-08)

```
   journal-1M (10⁶ registros, particionado por dia)
        │
        ├─ policy-simulator lê partição a partição (streaming, sem carregar tudo)
        │      │
        │      └─▶ DecisionEngine em modo sombra (o MESMO da produção — S4)
        │              │
        │              └─▶ agrega diff por rule_key
        │
        └─ produção sob carga B-01 em paralelo   ← mede o impacto cruzado
```

O ponto medido não é apenas a duração: é o **impacto no p99 de produção**. Se a
simulação degradasse o caminho quente, a consequência prática seria as equipes
evitarem simular — e o gate de publicação viraria formalidade.

## 6. Comparação com alternativas de arquitetura

| Arranjo | p99 de decisão | Revogação | Auditabilidade | Escolha |
|---------|----------------|-----------|----------------|---------|
| **PDP central, bundle em memória** (adotado) | ~3,2 ms | ≤ 5 s via evento | Central e completa | ADR-0222 |
| PDP central com regras em banco por consulta | ~25–60 ms | imediata | Central | Descartado: viola NFR-001 |
| Avaliação embarcada no PEP (bundle distribuído) | ~0,2 ms | fragmentada por réplica | Dispersa | Mantido como opção futura (RFC-0221) |
| Cache no PEP + PDP central (adotado em conjunto) | ~0 na maioria | TTL + evento | Central | ADR-0222 |

A combinação adotada — cache no PEP **mais** avaliação central em memória — é o que
faz a maioria das decisões custar zero e as demais custarem milissegundos, sem abrir
mão de um ponto único de auditoria.

## 7. Reprodutibilidade

| Item | Regra |
|------|-------|
| Fixtures | Versionadas (`bundle-realistic`, `journal-1M`) com digest publicado |
| Sementes | Geração de carga com semente fixa |
| Configuração | Perfil de produção (`./Configuration.md` §9), exceto amostragem do journal em 1,0 |
| Registro | Cada execução grava: versão do serviço, `compiled_digest`, ambiente, resultados brutos |
| Frequência | A cada release e semanalmente em homologação |
| Publicação | Resultados anexados ao release em `../035-Benchmark/` |

## 8. Interpretação dos resultados

| Observação | Diagnóstico provável | Ação |
|------------|----------------------|------|
| p99 sobe, p50 estável | Cauda por resolução de atributo | Verificar `aios_pol_attribute_resolve_ms` por fonte |
| p50 e p99 sobem juntos | Bundle maior ou regra custosa | Comparar `aios_pol_bundle_rules` e `compile_duration_ms` entre versões |
| Vazão cai sem CPU saturada | Contenção de lock ou GC | Analisar alocação no `AbacEvaluator` |
| Propagação lenta | Backlog de outbox ou consumo do stream | `aios_pol_outbox_pending`, saúde do NATS |
| Simulação lenta | I/O de partição do journal | Verificar índice `ix_journal_bundle` e paralelismo de leitura |

## 9. Referências

- Metas: `./NonFunctionalRequirements.md` · Métricas: `./Metrics.md`
- Testes: `./Testing.md` · Escala: `./Scalability.md`
- Metodologia global de benchmark: `../035-Benchmark/`

*Fim de `Benchmark.md`.*
