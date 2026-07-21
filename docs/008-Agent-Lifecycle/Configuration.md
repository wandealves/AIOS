---
Documento: Configuration.md
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0083, ADR-0084, ADR-0085
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 009-Scheduler, 027-Cluster, 028-Deployment
---

# 008-Agent-Lifecycle — Configuration

> **Escopo.** Catálogo completo das chaves de configuração do Lifecycle
> Manager: tipo, default, faixa válida, escopo de aplicação
> (`global`/`tenant`/`agente`), se é recarregável em runtime (*hot reload*)
> e a variável de ambiente correspondente. Fonte normativa:
> `_DESIGN_BRIEF.md` §8. **Nenhum valor aqui contradiz o brief** — este
> documento apenas detalha mecânica de carregamento, precedência,
> validação e exemplos operacionais.

---

## 1. Convenções de configuração do módulo 008

| Aspecto | Regra |
|---------|-------|
| Prefixo de env var | `AIOS_LIFECYCLE_` + chave em maiúsculas com `_` no lugar de `.` (ex.: `spawn.target_p99_ms` → `AIOS_LIFECYCLE_SPAWN_TARGET_P99_MS`). |
| Formato de arquivo | `appsettings.json` (.NET 10 `IConfiguration`) + overlay por ambiente (`appsettings.Production.json`) + variáveis de ambiente (precedência mais alta) + overrides por tenant/agente em PostgreSQL/labels. |
| Precedência (maior→menor) | 1) Override por **agente** (label no ACB, ex. `hibernation.idle_ttl_s` customizado por `spawn`) → 2) Override por **tenant** (config em banco/labels de tenant) → 3) Variável de ambiente → 4) `appsettings.{Environment}.json` → 5) `appsettings.json` (defaults do brief). |
| Hot reload | Chaves marcadas **sim** são observadas via `IOptionsMonitor<T>` (.NET) e reagem a mudança de configuração sem reiniciar o processo. Chaves **não** exigem *rolling restart* coordenado e/ou migração de dados. |
| Validação | Toda chave é validada contra a faixa declarada na inicialização (*fail-fast*) e em cada *reload*; valor fora da faixa é rejeitado e o valor anterior é mantido, com log estruturado (`Logging.md`) e métrica de rejeição (`Metrics.md`). |
| Escopo `global` | Aplica-se a todas as réplicas do serviço de ciclo de vida no cluster. |
| Escopo `tenant` | Pode ser sobreposto por tenant (ex.: tenant premium com `spawn.timeout_ms` maior ou `hibernation.idle_ttl_s` menor para liberar RAM mais cedo). |
| Escopo `agente` | Pode ser sobreposto por agente individual via `labels` no momento do `spawn`, respeitando tetos definidos no escopo `tenant`. |

---

## 2. Catálogo completo de chaves

| Chave | Tipo | Default | Faixa válida | Escopo | Recarregável | Variável de ambiente |
|-------|------|---------|--------------|--------|---------------|------------------------|
| `spawn.target_p99_ms` | int | `250` | `50–2000` | global/tenant | sim | `AIOS_LIFECYCLE_SPAWN_TARGET_P99_MS` |
| `spawn.timeout_ms` | int | `1500` | `250–10000` | global/tenant | sim | `AIOS_LIFECYCLE_SPAWN_TIMEOUT_MS` |
| `warmpool.min_per_shard` | int | `4` | `0–256` | tenant/shard | sim | `AIOS_LIFECYCLE_WARMPOOL_MIN_PER_SHARD` |
| `warmpool.max_per_shard` | int | `64` | `0–1024` | tenant/shard | sim | `AIOS_LIFECYCLE_WARMPOOL_MAX_PER_SHARD` |
| `hibernation.idle_ttl_s` | int | `300` | `30–86400` | tenant/agente | sim | `AIOS_LIFECYCLE_HIBERNATION_IDLE_TTL_S` |
| `hibernation.ram_pressure_threshold_pct` | int | `85` | `50–99` | global | sim | `AIOS_LIFECYCLE_HIBERNATION_RAM_PRESSURE_THRESHOLD_PCT` |
| `hibernation.storage_tier` | enum | `warm` | `hot\|warm\|cold` | tenant | sim | `AIOS_LIFECYCLE_HIBERNATION_STORAGE_TIER` |
| `checkpoint.interval_s` | int | `120` | `10–3600` | tenant/agente | sim | `AIOS_LIFECYCLE_CHECKPOINT_INTERVAL_S` |
| `checkpoint.codec` | enum | `msgpack+zstd` | `json+zstd\|msgpack+zstd` | global | **não** | `AIOS_LIFECYCLE_CHECKPOINT_CODEC` |
| `checkpoint.max_working_memory_mib` | int | `256` | `1–4096` | tenant | sim | `AIOS_LIFECYCLE_CHECKPOINT_MAX_WORKING_MEMORY_MIB` |
| `checkpoint.encryption` | enum (fixo) | `aes-256-gcm` | fixo | global | **não** | `AIOS_LIFECYCLE_CHECKPOINT_ENCRYPTION` |
| `lease.ttl_ms` | int | `5000` | `1000–30000` | global | sim | `AIOS_LIFECYCLE_LEASE_TTL_MS` |
| `lease.renew_ms` | int | `1500` | `500–10000` | global | sim | `AIOS_LIFECYCLE_LEASE_RENEW_MS` |
| `reconcile.interval_ms` | int | `2000` | `500–60000` | global | sim | `AIOS_LIFECYCLE_RECONCILE_INTERVAL_MS` |
| `reconcile.runtime_dead_after_ms` | int | `15000` | `3000–120000` | global | sim | `AIOS_LIFECYCLE_RECONCILE_RUNTIME_DEAD_AFTER_MS` |
| `migration.max_concurrent_per_node` | int | `8` | `1–128` | global/node | sim | `AIOS_LIFECYCLE_MIGRATION_MAX_CONCURRENT_PER_NODE` |
| `migration.saga_timeout_ms` | int | `120000` | `10000–600000` | global | sim | `AIOS_LIFECYCLE_MIGRATION_SAGA_TIMEOUT_MS` |
| `recovery.rto_budget_s` | int | `900` | `60–3600` | global | sim | `AIOS_LIFECYCLE_RECOVERY_RTO_BUDGET_S` |
| `retention.terminated_ttl_days` | int | `30` | `1–3650` | tenant | sim | `AIOS_LIFECYCLE_RETENTION_TERMINATED_TTL_DAYS` |
| `retention.checkpoint_ttl_days` | int | `7` | `1–365` | tenant | sim | `AIOS_LIFECYCLE_RETENTION_CHECKPOINT_TTL_DAYS` |
| `outbox.publish_batch` | int | `256` | `1–2048` | global | sim | `AIOS_LIFECYCLE_OUTBOX_PUBLISH_BATCH` |

> Todos os valores acima são reproduzidos **exatamente** de
> `_DESIGN_BRIEF.md` §8 — nenhuma faixa, default ou escopo foi alterado.

### 2.1 Descrição funcional de cada chave

| Chave | Descrição |
|-------|-----------|
| `spawn.target_p99_ms` | Meta de latência p99 para spawn/materialização (`Ready→Running`), usada pelo `SpawnManager` para decidir dimensionamento do `WarmPoolManager` e para o SLI de `NFR-001`. |
| `spawn.timeout_ms` | Timeout absoluto de uma tentativa de spawn/wake antes de retornar `AIOS-LIFECYCLE-0006`; sempre `>` `spawn.target_p99_ms`. |
| `warmpool.min_per_shard` | Reserva mínima de runtimes pré-aquecidos por tenant/shard mantida pelo `WarmPoolManager`, mesmo sob baixa demanda (evita cold-start no primeiro pico). |
| `warmpool.max_per_shard` | Teto de runtimes pré-aquecidos por tenant/shard; acima disso, excedente é liberado para conter custo ocioso. |
| `hibernation.idle_ttl_s` | Tempo de ociosidade (`now - updated_at` em `Suspended`) até o `HibernationController` disparar `hibernate` (T6). `Configuration.md` §5 detalha o efeito de valores extremos. |
| `hibernation.ram_pressure_threshold_pct` | Percentual de uso de RAM do nó/shard que antecipa hibernação de candidatos de menor prioridade, independentemente do `idle_ttl_s` (proteção contra storm de RAM). |
| `hibernation.storage_tier` | Camada de storage-alvo (`hot`/`warm`/`cold`) para o snapshot do `SnapshotStore` ao hibernar; afeta latência de wake (tiers mais frias são mais baratas, porém mais lentas). |
| `checkpoint.interval_s` | Intervalo de checkpoint automático periódico enquanto o agente está `Running`/`Suspended`, usado para sustentar a meta de RPO (`NFR-006`, ≤ 5 min). |
| `checkpoint.codec` | Formato de serialização do checkpoint (ACB + working memory). Mudança de codec **não é recarregável** — exige coexistência de leitores para ambos os formatos durante uma janela de migração (ver §5). |
| `checkpoint.max_working_memory_mib` | Tamanho máximo de working memory aceito em um checkpoint; acima disso, `CheckpointService` rejeita com `AIOS-LIFECYCLE-0011` (payload de serialização excede limite operacional). |
| `checkpoint.encryption` | Algoritmo de cifragem em repouso do payload de checkpoint (fixo em `aes-256-gcm`, chave por tenant gerida por `021-Security`); não configurável — presente apenas para documentar o contrato. |
| `lease.ttl_ms` | TTL do lease distribuído por agente (`LeaseManager`, Redis `SET NX PX`); expira automaticamente se o coordenador titular falhar sem liberar. |
| `lease.renew_ms` | Intervalo de renovação do lease pelo coordenador titular; **DEVE** ser menor que `lease.ttl_ms` (recomendado ≤ 1/3) para tolerar jitter de rede sem perder o lease em operação longa. |
| `reconcile.interval_ms` | Período do loop do `ReconciliationController` que varre `ix_acb_desired_mismatch` (ver `Database.md` §2.2) em busca de deriva estado×desejado. |
| `reconcile.runtime_dead_after_ms` | Ausência de heartbeat do runtime (007) por este período classifica o runtime como morto, disparando reconciliação para `Ready/Running` (a partir do último checkpoint) ou `Failed`. |
| `migration.max_concurrent_per_node` | Teto de migrações simultâneas por nó de origem/destino, usado pelo `MigrationOrchestrator` para não saturar rede/storage em migração em lote (ex.: `cluster.node.draining`). |
| `migration.saga_timeout_ms` | Timeout total da saga de migração (`quiesce→cutover`); excedido, aciona compensação (`MigrationJob.status=compensated`, `AIOS-LIFECYCLE-0010`). |
| `recovery.rto_budget_s` | Orçamento de tempo de recuperação usado pelo `ReconciliationController` para decidir entre nova tentativa de materialização e transição para `Failed` (`NFR-005`, RTO ≤ 15 min). |
| `retention.terminated_ttl_days` | Dias após `terminated_at` antes do `TombstoneManager` expurgar checkpoints/snapshots de um agente `Terminated`/`Failed`. |
| `retention.checkpoint_ttl_days` | Dias de retenção de checkpoints individuais (independente do estado do agente), usado para calcular `checkpoint.expires_at`. |
| `outbox.publish_batch` | Tamanho do lote de mensagens lidas por ciclo pelo relay do `LifecycleEventPublisher` (Outbox → JetStream). |

---

## 3. Diagrama de precedência de configuração (ASCII)

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  Resolução de valor efetivo para uma chave em uma dada operação    │
   └────────────────────────────────────────────────────────────────────┘

   hibernate(tenant=acme, agent=..., labels={hibernation.idle_ttl_s: 60})
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │ 1. Override por AGENTE (labels do ACB     │  maior precedência
        │    definidos no spawn ou atualizados)     │
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 2. Override por TENANT (config em banco   │
        │    ou labels de tenant)                   │
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 3. Variável de ambiente                   │
        │    (AIOS_LIFECYCLE_HIBERNATION_IDLE_TTL_S)│
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 4. appsettings.{Environment}.json         │
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 5. appsettings.json (default do brief)    │  menor precedência
        │    ex.: hibernation.idle_ttl_s=300         │
        └─────────────────────────────────────────┘
```

---

## 4. Exemplos de configuração por ambiente

### 4.1 `appsettings.json` (defaults — refletem exatamente o brief §8)

```json
{
  "Lifecycle": {
    "Spawn": {
      "TargetP99Ms": 250,
      "TimeoutMs": 1500
    },
    "WarmPool": {
      "MinPerShard": 4,
      "MaxPerShard": 64
    },
    "Hibernation": {
      "IdleTtlS": 300,
      "RamPressureThresholdPct": 85,
      "StorageTier": "warm"
    },
    "Checkpoint": {
      "IntervalS": 120,
      "Codec": "msgpack+zstd",
      "MaxWorkingMemoryMib": 256,
      "Encryption": "aes-256-gcm"
    },
    "Lease": {
      "TtlMs": 5000,
      "RenewMs": 1500
    },
    "Reconcile": {
      "IntervalMs": 2000,
      "RuntimeDeadAfterMs": 15000
    },
    "Migration": {
      "MaxConcurrentPerNode": 8,
      "SagaTimeoutMs": 120000
    },
    "Recovery": {
      "RtoBudgetS": 900
    },
    "Retention": {
      "TerminatedTtlDays": 30,
      "CheckpointTtlDays": 7
    },
    "Outbox": {
      "PublishBatch": 256
    }
  }
}
```

### 4.2 Override via variáveis de ambiente (tenant premium, produção)

```bash
# Tenant "acme" opera com SLA mais folgado para spawn e hibernação mais
# agressiva para liberar RAM antes, dado alto volume de agentes cold.
export AIOS_LIFECYCLE_SPAWN_TIMEOUT_MS=3000
export AIOS_LIFECYCLE_HIBERNATION_IDLE_TTL_S=120
export AIOS_LIFECYCLE_WARMPOOL_MIN_PER_SHARD=16
export AIOS_LIFECYCLE_RECOVERY_RTO_BUDGET_S=600
```

### 4.3 Override por agente (label no `spawn`)

```json
POST /v1/agents
{
  "priorityClass": "batch",
  "policyRef": "urn:aios:acme:policy:batch-agent",
  "quotaRef": "urn:aios:acme:quota:batch",
  "labels": {
    "hibernation.idle_ttl_s": "30",
    "checkpoint.interval_s": "600"
  }
}
```

O override acima **DEVE** ser validado pelo `LifecycleCoordinator` contra o
teto/piso definidos no escopo `tenant` antes de ser aplicado; um valor fora
da faixa global (§2) é rejeitado com `AIOS-LIFECYCLE-0002`-like validação de
payload (mapeado ao envelope de erro padrão RFC 7807) antes mesmo de
persistir o ACB.

---

## 5. Chaves não recarregáveis — impacto operacional

| Chave | Por que não é recarregável | Procedimento para alterar |
|-------|-------------------------------|------------------------------|
| `checkpoint.codec` | Checkpoints existentes em MinIO foram serializados no codec vigente no momento da criação; trocar o codec em runtime deixaria o `CheckpointService` incapaz de decidir qual formato usar para `restore` sem versionar o metadado. | Introduzir migração de codec em duas fases: (1) `CheckpointService` passa a **ler ambos** os codecs (retrocompatibilidade, sinalizada em `RFC-0080`); (2) após janela de coexistência, novo codec vira default via release coordenado, nunca hot-reload. |
| `checkpoint.encryption` | Fixo por decisão de segurança (ADR-0082); alternar algoritmo de cifragem em runtime comprometeria a decifragem de checkpoints já gravados sem um processo de rotação de chave coordenado com `021-Security`. | Rotação/alteração de algoritmo de cifragem é tratada como procedimento de segurança formal em `../021-Security/`, não como configuração operacional comum. |

Todas as demais chaves listadas no §2 são observadas via
`IOptionsMonitor<T>` e aplicadas **sem reinício de processo**; a propagação
para todas as réplicas do serviço de ciclo de vida ocorre por invalidação
de cache local acionada por mecanismo interno de configuração do control
plane (não um subject NATS público).

---

## 6. Validação e comportamento em falha

- Na inicialização, o serviço de ciclo de vida **DEVE** validar todas as
  chaves contra as faixas do §2; violação de faixa em qualquer chave
  **DEVE** impedir o boot do serviço (*fail-fast*), com log estruturado
  indicando a chave e o valor rejeitado.
- Em *hot reload*, um valor fora da faixa **DEVE** ser rejeitado mantendo o
  valor efetivo anterior, emitindo log de nível `Error` e métrica
  `aios_lifecycle_config_reload_rejected_total{key}` (ver `Metrics.md`).
- `lease.renew_ms` **DEVE** ser rejeitado se configurado maior ou igual a
  `lease.ttl_ms` (violaria a garantia de renovação antes da expiração,
  comprometendo INV1 do brief — serialização de transições por agente).
- `retention.checkpoint_ttl_days` **DEVE** ser rejeitado se resultar em
  `expires_at` anterior a `created_at + checkpoint.interval_s` para o mesmo
  agente (checkpoints não podem expirar antes do próximo checkpoint
  periódico ser criado, sob risco de janela sem snapshot válido).
- `hibernation.ram_pressure_threshold_pct` **DEVERIA** disparar um alerta
  operacional (`Monitoring.md`) quando ativado por pressão de RAM (em vez
  de por `idle_ttl_s`), pois indica que o cluster está próximo do limite de
  capacidade planejado.

---

## 7. Referências

- Fonte normativa das chaves e seus defaults: `./_DESIGN_BRIEF.md` §8.
- Consumo das chaves pelos componentes: `./Architecture.md` (`SpawnManager`, `WarmPoolManager`, `HibernationController`, `CheckpointService`, `LeaseManager`, `ReconciliationController`, `MigrationOrchestrator`, `LifecycleEventPublisher`).
- Impacto de `checkpoint.codec`/`checkpoint.encryption` no modelo físico: `./Database.md` §2.4.
- Consumo de `lease.ttl_ms`/`lease.renew_ms` no fluxo de coordenação: `./SequenceDiagrams.md`.
- Métricas de configuração/reload: `./Metrics.md`.
- Topologia de deployment e overlays por ambiente: `../028-Deployment/`.
- Modos de falha relacionados a timeouts/leases/sagas: `./FailureRecovery.md`.
