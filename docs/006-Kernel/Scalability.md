---
Documento: Scalability
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0006 (Redis Estado Quente), ADR-0061 (ACB & OCC), ADR-0062 (Token-bucket atômico em Redis), ADR-0065 (Sharding do ACB)
RFCs relacionados: RFC-0001 (baseline), RFC-0007 (ACB & Resource Quota Model, a propor)
Depende de: 001-Architecture, 006-Kernel/Architecture (a publicar), 009-Scheduler, 008-Lifecycle, 027-HA-DR
---

# 006-Kernel — Scalability

> **Escopo.** Este documento especifica o modelo de escala do Kernel: como ele
> escala horizontalmente, como particiona/shardeia o `AgentControlBlock` (ACB),
> como resolve concorrência sem locks pessimistas, como aplica backpressure e
> quais são os limites teóricos conhecidos no caminho para milhões de agentes.
> Consolida e aprofunda `_DESIGN_BRIEF.md` §10, sem contradizê-lo.

---

## 1. Modelo de Escala Geral

O Kernel Service é projetado como **stateless horizontalmente escalável**: nenhuma
réplica mantém estado que não possa ser reconstruído a partir de PostgreSQL
(fonte da verdade) ou Redis (estado quente). Isso decorre diretamente de R-02 e
R-05 do brief (ACB Store autoritativo externalizado; coordenação de ciclo de vida
via sagas idempotentes) e permite que a capacidade do Kernel cresça
**linearmente com o número de réplicas**, limitada apenas pela capacidade das
dependências compartilhadas (PostgreSQL, Redis, NATS, PDP).

| Dimensão | Estratégia de escala |
|----------|-------------------------|
| Escala do **serviço Kernel** | **Horizontal** — réplicas idênticas sem afinidade obrigatória (§2). |
| Escala do **estado do ACB** | **Particionamento lógico (sharding)** + separação hot/cold (§3). |
| Escala de **cotas concorrentes** | **Estruturas atômicas distribuídas** (token-bucket Lua em Redis) (§5). |
| Escala de **eventos** | Delegada ao NATS/JetStream (particionamento por stream/subject, fora do escopo deste documento — ver `../020-Communication/Scalability.md`). |
| Escala de **decisão de política** | Cache local de decisão por réplica com TTL curto, reduzindo fan-out ao PDP (`Security.md` §3.2). |

---

## 2. Escala Horizontal do Serviço

```
                     ┌──────────────── Gateway (YARP) — health-aware LB ────────────────┐
                     ▼                          ▼                            ▼
             ┌───────────────┐         ┌───────────────┐            ┌───────────────┐
             │ Kernel réplica │  ...    │ Kernel réplica │    ...     │ Kernel réplica │
             │      i         │         │      j         │            │      k         │
             │ (stateless)    │         │ (stateless)    │            │ (stateless)    │
             └───────┬────────┘         └───────┬────────┘            └───────┬────────┘
                     │                          │                              │
                     └──────────────┬───────────┴───────────────┬─────────────┘
                                    ▼                            ▼
                         Redis (ACB quente, cotas)      PostgreSQL (fonte da verdade,
                         cluster particionado                particionado por tenant_id
                         por slot                             + shard_key)
```

- Qualquer réplica **DEVE** poder processar qualquer syscall de qualquer
  tenant/agente — não há particionamento de tráfego obrigatório no Gateway.
- Réplicas **PODEM** manter cache local em memória de decisões de PEP recém
  usadas e de ACBs recém acessados (otimização de latência), mas esse cache
  **NÃO É** fonte da verdade e **DEVE** ser invalidável (TTL curto ou invalidação
  por evento).
- Adicionar/remover réplicas **NÃO REQUER** coordenação de estado — é uma
  operação puramente de infraestrutura (ver `Deployment.md` §9).

---

## 3. Particionamento (Sharding) do ACB

### 3.1 Função de sharding

```
shard_key = hash(tenant_id, agent_id) mod N        (N = kernel.acb.shard_count, default 64)
```

- `shard_key` é um campo **gerado e persistido** na tabela `kernel.acb`
  (`_DESIGN_BRIEF.md` §3.1), calculado uma única vez na criação do ACB.
- `N` (`kernel.acb.shard_count`) **NÃO É recarregável em runtime** — alterar `N`
  redistribui todos os `shard_key` existentes e exige uma **migração controlada**
  (rebalanceamento), nunca uma mudança de configuração *hot*.
- O sharding serve a dois propósitos: (a) **localidade de cache** — uma réplica
  que atendeu recentemente um shard tem maior probabilidade de cache hit local;
  (b) **particionamento físico do PostgreSQL** por `shard_key`/`tenant_id`
  (`Database.md`), permitindo *table partitioning* e paralelismo de manutenção
  (vacuum, reindex) sem contenção global.

### 3.2 Diagrama de roteamento por shard (otimização, não requisito de correção)

```
   Syscall(tenant=acme, agent=01J9...)
                │
                ▼
      shard = hash(acme, 01J9...) mod 64  =  23
                │
      ┌─────────┴──────────┐
      │  Réplica com afinidade   SIM → cache local quente → resposta rápida
      │  lógica ao shard 23?     │
      └─────────┬──────────┘
                │ NÃO (ou cache miss)
                ▼
      Busca em Redis (ACB quente) → cache miss → PostgreSQL (fonte da verdade)
                │
                ▼
          Popula cache local + Redis, responde
```

### 3.3 Estado quente (hot) vs. frio (cold)

| Estado do ACB | Onde reside | Latência de acesso | Custo de RAM/infra |
|-----------------|--------------|------------------------|------------------------|
| `Pending`/`Admitted`/`Running`/`Suspended` | Redis (projeção quente) + PostgreSQL (durável) | Sub-milissegundo (Redis) | Alto por agente ativo |
| `Hibernated` | Somente PostgreSQL + snapshot durável em MinIO (via `008-Lifecycle`) | Dezenas–centenas de ms (materialização sob demanda) | Baixíssimo — zero RAM, apenas linha em PostgreSQL |
| `Terminated`/`Failed` | Somente PostgreSQL (retido para auditoria) | Não aplicável (somente leitura histórica) | Mínimo |

A transição para `Hibernated` (T-08 da FSM, `_DESIGN_BRIEF.md` §4) é o mecanismo
central que permite suportar **≥ 10⁶ ACBs** (NFR-006) com custo de infraestrutura
sub-linear: a vasta maioria dos agentes registrados em um instante qualquer está
`Hibernated` (custo ≈ 1 linha de banco), e apenas uma fração pequena está
`Running`/`Suspended` consumindo Redis/RAM.

---

## 4. Concorrência sobre o ACB

### 4.1 Controle de concorrência otimista (OCC)

Toda mutação do ACB **DEVE** usar o campo `version` (`_DESIGN_BRIEF.md` §3.1)
como controle de concorrência otimista — **não** locks pessimistas no caminho
quente:

```
UPDATE kernel.acb
   SET state = 'Running', runtime_ref = $1, version = version + 1, updated_at = now()
 WHERE urn = $2 AND version = $3   -- version esperado, lido antes da transição
```

- Se `UPDATE` afeta 0 linhas → conflito de versão → o chamador **DEVE** reler o
  ACB e reavaliar a transição (a FSM pode já ter avançado por outro caminho
  concorrente, ex.: `kill` disparado enquanto `resume` estava em voo).
- Retentativas são limitadas por `kernel.acb.optimistic_retry_max` (default 5);
  esgotadas as tentativas, retorna `AIOS-KERNEL-0009` (409) — a operação **É**
  idempotente por natureza (reaplicar a mesma intenção é seguro).
- **Por que OCC e não locks pessimistas** (`SELECT ... FOR UPDATE` ou lock
  distribuído por padrão): conflitos de escrita concorrente sobre o **mesmo**
  ACB são estatisticamente raros (um agente tipicamente tem um único "dono"
  lógico por vez); pagar o custo de um lock em 100% das transições para proteger
  contra um conflito raro degradaria a latência do caminho quente (viola
  NFR-001/NFR-002). Ver ADR-0061.

### 4.2 Locks distribuídos (exceção controlada)

Locks distribuídos (Redis, TTL curto) **SÃO PERMITIDOS** apenas onde uma
transição de FSM tem efeito colateral externo não-idempotente e não pode ser
resolvida por retry de OCC isoladamente (ex.: coordenar `checkpoint` concorrente
com `kill` para o mesmo agente, evitando snapshot parcialmente escrito). Regras:

- Lock **DEVE** ter TTL curto (ordem de segundos) para nunca travar
  indefinidamente sob falha do detentor.
- Lock **NÃO DEVE** ser usado como substituto de OCC para transições de estado
  simples — é reservado a coordenação de efeito colateral externo.
- Preferir sempre **reduzir a necessidade de lock** via desenho de saga
  (compensação) a introduzir mais locks.

---

## 5. Cotas Concorrentes

### 5.1 Token-bucket atômico

Contadores de cota (`tokens`, `cpu_ms`, `cost_usd`, `tools`, `syscalls_per_sec`)
são mantidos em **Redis**, com reserva/consumo/liberação executados via **script
Lua atômico** (evita condição de corrida entre `EXISTS`/`GET` e `DECRBY`
separados):

```
-- Pseudocódigo do script Lua de reserva atômica (simplificado)
local key = KEYS[1]                    -- subject_urn + janela
local requested = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local window_seconds = tonumber(ARGV[3])

local current = tonumber(redis.call("GET", key) or "0")
if current + requested > limit then
    return {0, current}                -- rejeitado; saldo atual para diagnóstico
end
redis.call("INCRBY", key, requested)
redis.call("EXPIRE", key, window_seconds)
return {1, current + requested}        -- aceito; novo saldo
```

- A atomicidade do script Lua (executado single-threaded pelo Redis) elimina a
  janela de corrida entre verificação e incremento, garantindo overshoot de cota
  `hard` ≤ 1% mesmo sob alta concorrência (NFR-009) — o overshoot residual vem
  de reconciliação com PostgreSQL (limites), não do contador em si.
- PostgreSQL (`kernel.quota`) é a fonte da verdade dos **limites**; Redis é a
  fonte da verdade dos **contadores quentes** dentro da janela corrente. Uma
  reconciliação periódica corrige divergência (ex.: perda parcial de Redis,
  ver `FailureRecovery.md`).

### 5.2 Enforcement `hard` vs. `soft`

| Modo | Comportamento na rejeição | Uso típico |
|------|-------------------------------|--------------|
| `hard` | Rejeita a syscall (`AIOS-QUOTA-0001/0002/0003`, 429) e emite `agent.quota.exceeded`. | Tenants em produção com orçamento estrito. |
| `soft` | Permite a execução, mas registra e emite `agent.quota.warning` ao cruzar limiar configurável (ex.: 80%). | Ambientes de teste/observação de padrão de consumo antes de aplicar `hard`. |

---

## 6. Backpressure e Isolamento de Noisy Neighbor

| Camada | Mecanismo | Efeito |
|--------|-----------|--------|
| Por agente | `syscalls_per_sec` (token-bucket, §5.1) | Um agente não pode saturar o Kernel sozinho. |
| Por tenant | Cota `agent_count`, `cost_usd_limit`, `tokens_limit` agregados no escopo `tenant` | Um tenant não pode consumir capacidade além do seu orçamento, protegendo outros tenants (*multi-tenancy* segura). |
| Admissão de novos agentes | Delegada ao `009-Scheduler` (admission control) | `spawn` não é aceito incondicionalmente; backpressure de capacidade de execução é decidida fora do Kernel, mas o Kernel propaga a rejeição como `AIOS-KERNEL-0003`. |
| Fila/stream de eventos | Limites de retenção/tamanho por stream JetStream | Evita que um produtor rápido esgote armazenamento compartilhado do barramento (detalhado em `../020-Communication/`). |
| Fan-out de sub-agentes | Limite de profundidade/largura de árvore de `spawn` (via cota `agent_count` por tenant e política de `022`) | Evita custo combinatório O(N²) de coordenação entre agentes (Glossário: *Fan-out*). |

**Princípio de rejeição precoce:** sob saturação, o Kernel **DEVERIA** rejeitar
rapidamente com `429` (retriable) em vez de enfileirar syscalls indefinidamente
ou degradar a latência de todas as requisições igualmente — rejeição precoce e
localizada preserva o SLO das demais requisições (isolamento de *noisy
neighbor*, alinhado a NFR-004).

---

## 7. Limites Teóricos e Capacidade

| Métrica | Meta/limite conhecido | Fonte |
|---------|--------------------------|-------|
| Throughput por réplica (mix de controle) | ≥ 20.000 syscalls/s | NFR-004 |
| Latência p99 `spawn` (cold → running) | ≤ 250 ms | NFR-001 |
| Latência p99 syscall de controle (`suspend`/`resume`/`kill`/`get_quota`) | ≤ 20 ms | NFR-002 |
| Latência p99 decisão de PEP | ≤ 8 ms (cache hit) / ≤ 25 ms (cache miss) | NFR-003 |
| ACBs suportados (majoritariamente `Hibernated`) | ≥ 10⁶ | NFR-006 |
| Overshoot de cota `hard` sob concorrência | ≤ 1% | NFR-009 |
| Faixa de `kernel.acb.shard_count` | 1–4096 | Configuration (`_DESIGN_BRIEF.md` §8) |

### 7.1 Capacity Planning — estimativa por réplica

| Cenário | Réplicas estimadas (referência inicial, sujeita a `Benchmark.md`) |
|---------|------------------------------------------------------------------------|
| 100.000 syscalls/s agregados (mix de controle) | ≥ 5 réplicas (com margem de 20% acima do throughput nominal por réplica). |
| 10⁶ ACBs totais, ~1% ativos simultaneamente (`Running`/`Suspended`) | ~10.000 ACBs quentes em Redis; réplicas dimensionadas pelo throughput de syscall, não pela contagem total de ACBs (que reside majoritariamente `Hibernated`, custo quase nulo por unidade). |
| Pico de `spawn` (rajada de novos agentes) | Absorvido primariamente pela admissão do `009-Scheduler`; Kernel apenas propaga backpressure — não é o gargalo dominante nesse cenário. |

> Estes números são **estimativas de referência**; a validação empírica
> definitiva (carga sintética, ponto de saturação real, ponto de ruptura) é
> responsabilidade de `./Benchmark.md`, que **DEVE** ser mantido consistente com
> estas metas ou justificar desvio via ADR.

---

## 8. Caminho para Milhões de Agentes

A combinação de estratégias que permite escalar o Kernel de milhares para
milhões de ACBs geridos é:

1. **Predominância de agentes *cold*** — a maioria dos ACBs em qualquer instante
   está `Hibernated`, custando apenas uma linha de PostgreSQL (§3.3).
2. **Sharding determinístico** — `hash(tenant,agent) mod N` distribui carga e
   habilita particionamento físico do banco (§3.1, `Database.md`).
3. **OCC em vez de locks pessimistas** — elimina contenção de lock como gargalo
   de escala vertical por linha de ACB (§4.1).
4. **Cache de decisão de PEP** — reduz fan-out síncrono ao PDP em ordens de
   grandeza sob alta taxa de syscalls repetidas (`Security.md` §3.2).
5. **Cotas atômicas locais ao Redis** — evitam round-trip ao PostgreSQL no
   caminho quente de enforcement (§5.1).
6. **Limites de fan-out em árvores de `spawn`** — evitam custo combinatório de
   coordenação entre agentes-filhos (§6, Glossário: *Fan-out*).
7. **Escala horizontal stateless do serviço** — capacidade de throughput cresce
   linearmente adicionando réplicas, sem redesenho (§2).

Esta combinação é consistente com a estratégia global de escala descrita em
`../001-Architecture/Architecture.md` §12, à qual este documento se alinha sem
redefinir.

---

## 9. Riscos de Escala e Mitigação

| Risco | Descrição | Mitigação |
|-------|-----------|-------------|
| Hot shard | Um único `tenant`/`agent` com volume anormal de syscalls concentra carga em um shard lógico. | Cotas por agente/tenant limitam o volume máximo individual (§5); rebalanceamento de `shard_count` via migração controlada se necessário. |
| Reparticionamento custoso | Alterar `kernel.acb.shard_count` exige recalcular `shard_key` de todos os ACBs existentes. | Migração planejada com dupla-escrita temporária ou *backfill* em lote fora do caminho quente; não é operação de configuração *hot*. |
| Cache stale de decisão de PEP mascarando revogação | Cache local de `allow` sobrevive além do necessário após revogação de capability. | Invalidação por evento `policy.bundle.updated` (§3.2 de `Security.md`) além do TTL. |
| Crescimento não-linear de índices no PostgreSQL sob 10⁶+ linhas | Índices `btree(tenant_id, last_active_at)` e `btree(shard_key)` degradam sem particionamento físico. | Particionamento de tabela por `tenant_id`/`shard_key` (`Database.md`); manutenção (vacuum/reindex) por partição, não global. |
| Redis como ponto de contenção único de cotas | Alto volume de scripts Lua concorrentes no mesmo nó Redis. | Cluster Redis com distribuição de chaves por `subject_urn` (hash slot nativo do Redis Cluster). |

---

## 10. Referências

- Modelo de dados e sharding do ACB: `./_DESIGN_BRIEF.md` §3, §10.
- Máquina de estados (hibernação/resume): `./_DESIGN_BRIEF.md` §4.
- Requisitos não-funcionais relacionados: `./NonFunctionalRequirements.md` (NFR-001, NFR-004, NFR-006, NFR-009).
- Modelo físico de banco e particionamento: `./Database.md`.
- Topologia de implantação e autoescalonamento operacional: `./Deployment.md`.
- Modos de falha sob perda de Redis/NATS: `./FailureRecovery.md`.
- Estratégia global de escala do AIOS: `../001-Architecture/Architecture.md` §12.
- Glossário: `../040-Glossary/Glossary.md` (Shard, Hot State, Cold Agent, Fan-out, Backpressure).
