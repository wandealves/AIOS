---
Documento: Scalability
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0002, ADR-0006, ADR-0221, ADR-0222
RFCs relacionados: RFC-0001, RFC-0221
Depende de: _DESIGN_BRIEF.md §10, NonFunctionalRequirements.md, Benchmark.md, Deployment.md
---

# 022-Policy — Escalabilidade

## 1. O problema

O PDP está no caminho de **toda** ação privilegiada do AIOS. Com 10⁶ agentes ativos, o
pico agregado é da ordem de **10⁵ decisões/s**. Um monitor de referência que não
sustente esse volume não é contornado por engano — é contornado de propósito, e a
governança vira opcional.

```
   ❌ política consultada por linha de código      ✅ bundle compilado em memória
      cada regra = uma query ao banco                 índice por ação/recurso
      (PDP vira gargalo e SPOF do AIOS)               PEP cacheia por TTL curto
```

## 2. Estratégias

| Dimensão | Estratégia |
|----------|-----------|
| Serviço de decisão | **Stateless** e replicado; o bundle compilado vive em memória de cada réplica. Nenhuma decisão consulta o banco no caminho quente (invariante estrutural S2). |
| Cache no PEP | O TTL emitido por decisão faz o chamador absorver a maior parte do tráfego repetitivo; o custo é a revogação não instantânea, mitigado por TTL curto + `policy.decision.updated` granular. |
| Indexação de regras | O `BundleCompiler` indexa por `(dominio, objeto, verbo)` e por padrão de recurso: a avaliação percorre o subconjunto casável, não as 10⁴ regras. |
| Avaliação sem I/O bloqueante | RBAC e ABAC operam sobre estruturas em memória; a única I/O possível é a resolução de atributo externo, com timeout rígido e cache. |
| Cache de atributos | Redis + cache local por réplica, TTL por fonte; atributo `sensitive` cacheado por digest, nunca por valor em claro. |
| Particionamento do journal | `RANGE(decided_at)` diário: expurgo e simulação por janela viram operações de partição. |
| Concorrência de publicação | OCC por `version` + índice único parcial sobre `Active`: duas publicações concorrentes não produzem dois bundles ativos — a segunda falha com `AIOS-POL-0006`. |
| Simulação isolada | `policy-simulator` em pool dedicado, lendo partições do journal: carga analítica **não** compete com o caminho quente (NFR-012). |
| Backpressure | Limite por tenant (`max_eval_per_s_per_tenant`) com `Retry-After`, aplicado no `DecisionGateway` **antes** de qualquer resolução de atributo. |

## 3. Modelo de escala

```
   Camada 0 — PEP (no chamador)
   ┌──────────────────────────────────────────────────────────────┐
   │  cache de decisão por TTL   →  absorve ~90–99% do tráfego     │
   └───────────────────────────┬──────────────────────────────────┘
                               │ cache miss
   Camada 1 — policy-api (horizontal, sem estado compartilhado)
   ┌──────────────────────────────────────────────────────────────┐
   │  réplica #1 … #N   ·  cada uma com o bundle COMPILADO em RAM │
   │  escala linear: +réplica = +50.000 decisões/s                │
   └───────────────────────────┬──────────────────────────────────┘
                               │ só quando falta atributo
   Camada 2 — fontes de atributo (021 / 026 / 024), com cache e timeout
   ┌──────────────────────────────────────────────────────────────┐
   │  Redis (compartilhado) + cache local por réplica              │
   └──────────────────────────────────────────────────────────────┘

   PostgreSQL fica FORA do caminho quente — participa de autoria,
   publicação e journal (assíncrono), nunca da decisão.
```

Não há sharding do serviço de decisão: como a avaliação é *stateless* e o bundle cabe
em memória (≤ 64 MiB por bundle), qualquer réplica responde qualquer requisição de
qualquer tenant. Introduzir sharding por tenant traria afinidade e desbalanceamento
sem ganho — o oposto do que o `../009-Scheduler/` precisa fazer com estado de fila.

## 4. Concorrência e locks

| Ponto | Mecanismo | Por quê |
|-------|-----------|---------|
| Avaliação | **Sem lock**; leitura de estrutura imutável em memória | Bundle compilado é imutável; troca é por referência atômica (S3) |
| Troca de versão | Referência atômica; avaliações em curso terminam na versão antiga | Decisão híbrida entre duas políticas seria inauditável |
| Publicação | OCC + índice único parcial | Impede dois `Active` sem serializar todo o fluxo administrativo |
| Concessão de exceção | OCC por `version` | Concessões concorrentes ao mesmo sujeito não se perdem |
| Escrita do journal | Buffer em memória + escrita em lote | Tira o PostgreSQL do caminho de resposta (S6) |
| Contadores de limite | Redis (`INCR` com janela) | Contenção **por tenant**, nunca global |

## 5. Limites teóricos e observados

| Dimensão | Limite de projeto | Fator limitante | Ação ao aproximar |
|----------|-------------------|-----------------|-------------------|
| Decisões/s por réplica | 50.000 (NFR-002) | CPU (avaliação de expressões) | HPA por CPU e p99 |
| Decisões/s agregadas | ~10⁶ com 20 réplicas | Nenhum estado compartilhado | Adicionar réplicas |
| Regras por bundle | 10.000 (`pol.bundle.max_rules`) | Índice de regras e memória | Refatorar política; particionar por escopo |
| Tamanho do artefato | 64 MiB (`pol.bundle.max_compiled_mb`) | RAM por réplica × nº de tenants ativos | Carga sob demanda por tenant |
| Vínculos por tenant | 10⁵ (NFR-005) | Grafo de papéis em memória | Preferir vínculo por grupo a vínculo por sujeito |
| Agentes | 10⁶ | Cache no PEP + TTL | — |
| Fontes de atributo `required` | Σ timeouts ≤ 50 ms | Orçamento de avaliação (V-02) | Reclassificar fontes como `optional` |
| Journal | ~10⁴ escritas/s por instância de banco | I/O do PostgreSQL | Reduzir `allow_sample_rate`; partições menores |

### 5.1 Onde a escala realmente dói

O limite prático não é o número de decisões: é o produto **nº de tenants ativos ×
tamanho do bundle** ocupando RAM em cada réplica. Com 500 tenants de 64 MiB, uma
réplica precisaria de 32 GiB só de política. Mitigações, na ordem:

1. **Carga sob demanda**: réplica carrega o bundle no primeiro pedido do tenant e o
   descarta por LRU após inatividade.
2. **Limite real por bundle** bem abaixo do teto: a maioria dos tenants opera com
   centenas de regras, não milhares.
3. **Afinidade suave por tenant** no balanceador (não sharding rígido), aumentando a
   taxa de acerto do bundle já carregado.

## 6. Rumo a milhões de agentes

```
   10⁶ agentes  ──▶  ~10⁵ decisões/s no pico agregado
        │
        ├── cache no PEP        → a maioria das decisões NUNCA chega ao 022
        ├── avaliação em memória→ sem I/O por decisão
        ├── réplicas sem estado → escala linear até o limite do balanceador
        ├── propagação por evento→ nenhuma réplica faz polling de política
        └── journal amostrado   → o volume de escrita não acompanha o de decisão
```

O que **não** escala e precisa de disciplina:

| Antipadrão | Efeito em escala | Alternativa |
|-----------|------------------|-------------|
| Vínculo de papel por sujeito individual | 10⁶ linhas em `role_binding`; grafo enorme em memória | Vincular **grupos**; usar ABAC sobre atributo do sujeito |
| Regra com condição sobre atributo remoto `required` | Toda decisão vira chamada externa | Reclassificar como `optional` ou pré-resolver no PEP |
| `allow_sample_rate = 1.0` em produção | Journal com 10⁵ escritas/s | Manter 0,05; `deny` já é 100% |
| TTL de decisão = 0 no PEP | Elimina a camada 0; multiplica a carga por ~50× | Usar o TTL emitido pelo PDP |
| Bundle único gigante para todos os escopos | Recompilação e propagação lentas | Separar `platform` de `tenant`; refatorar por domínio |

## 7. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §10
- Medição: `./Benchmark.md` · Metas: `./NonFunctionalRequirements.md`
- Implantação e HPA: `./Deployment.md` §2.1 · Falhas: `./FailureRecovery.md`
- Plataforma: `../027-Cluster/`

*Fim de `Scalability.md`.*
