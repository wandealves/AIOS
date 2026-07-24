---
Documento: Configuration
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0252, ADR-0253, ADR-0254, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §8, NonFunctionalRequirements.md, Deployment.md
---

# 025-Audit — Configuração

Prefixo de variável de ambiente: **`AIOS_AUD_`**. A chave `aud.seal.interval_s`
corresponde a `AIOS_AUD_SEAL_INTERVAL_S`.

Escopos: `global` (instalação), `tenant`, `record_class`. Quando uma chave existe em
mais de um escopo, vale a **mais específica**, limitada pelo teto do escopo mais geral
— um tenant **NÃO DEVE** conseguir configurar-se para além do que a plataforma
permite.

## 1. Ingestão

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `aud.ingest.require_durable_ack` | bool | `true` | {true,false} | global | **não** | Confirma só após durabilidade (invariante I9, `AIOS-AUD-0005`). |
| `aud.ingest.batch_max_size` | int | 500 | 1–5000 | global | sim | Registros por lote na API. |
| `aud.ingest.max_rate_per_s_per_tenant` | int | 100000 | 100–1000000 | tenant | sim | Limite de ingestão por tenant. |

> `aud.ingest.require_durable_ack` é **não-recarregável** e `true` por decisão
> arquitetural (ADR-0255). Desligá-lo trocaria indisponibilidade visível por lacuna
> invisível — e a lacuna é o defeito que este módulo existe para impedir.

## 2. Cadeia e selagem

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `aud.chain.partitions_per_tenant` | int | 16 | 1–256 | tenant | **não** | Partições da cadeia; alterar exige migração (a sequência é por partição). |
| `aud.chain.hash_algorithm` | enum | `sha256` | {sha256,sha3-256} | global | **não** | Algoritmo de hash da cadeia. |
| `aud.seal.interval_s` | int | 60 | 10–3600 | global | sim | Intervalo de selagem (NFR-004). |
| `aud.seal.signing_key_ref` | secret | — | — | global | **não** | Chave de assinatura de selo (custodiada no `../021-Security/`). |
| `aud.seal.external_anchor_enabled` | bool | `false` | {true,false} | tenant | sim | Ancoragem externa do selo, quando o contrato exigir. |

`aud.chain.hash_algorithm` e `aud.chain.partitions_per_tenant` são não-recarregáveis
porque ambos mudam o **cálculo do `record_hash`** ou a estrutura da sequência:
alterá-los em runtime produziria uma cadeia que não fecha a partir daquele ponto.

## 3. Verificação e completude

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `aud.verify.interval_s` | int | 900 | 60–86400 | global | sim | Ciclo de verificação contínua por amostragem (NFR-007). |
| `aud.verify.sample_size` | int | 1000 | 10–100000 | global | sim | Registros por ciclo de verificação. |
| `aud.completeness.gap_alert_s` | int | 300 | 30–86400 | global | sim | Prazo para sinalizar lacuna (NFR-005). |
| `aud.completeness.silence_ratio` | decimal | 0.2 | 0.0–1.0 | record_class | sim | Fração do volume esperado abaixo da qual a classe é considerada silenciosa. |

## 4. Retenção e *legal hold*

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `aud.retention.default_days` | int | 1825 | 30–7300 | record_class | sim | Retenção legal default (5 anos). |
| `aud.retention.min_days` | int | 365 | 30–7300 | global | **não** | **Piso absoluto**: nenhuma classe pode reter menos. |
| `aud.hold.review_interval_days` | int | 90 | 7–365 | tenant | sim | Periodicidade da revisão obrigatória de *holds* ativos. |
| `aud.hold.require_dual_approval` | bool | `true` | {true,false} | global | **não** | Aprovador distinto do solicitante (`AIOS-AUD-0009`). |
| `aud.erasure.max_latency_h` | int | 24 | 1–720 | tenant | sim | Prazo para efetivar o apagamento criptográfico (NFR-011). |

## 5. Arquivamento WORM

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `aud.worm.object_lock_days` | int | 1825 | 30–7300 | global | **não** | Duração do *object lock* no arquivamento. |
| `aud.worm.archive_after_s` | int | 3600 | 60–86400 | global | sim | Prazo entre selagem e arquivamento. |
| `aud.worm.bucket` | string | — | — | global | **não** | Bucket com *object lock* habilitado. |

> **`aud.retention.min_days` e `aud.worm.object_lock_days` são não-recarregáveis por
> desenho.** São os dois valores que um operador sob pressão teria incentivo para
> reduzir a fim de "liberar espaço" — e reduzi-los em runtime destruiria prova que a
> lei exige preservar. Alterá-los exige *deploy* revisado, que por sua vez é auditado.

## 6. Consulta e exportação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `aud.query.max_duration_s` | int | 30 | 1–300 | tenant | sim | Orçamento de tempo por consulta (`AIOS-AUD-0013`). |
| `aud.query.max_records` | int | 100000 | 100–10000000 | tenant | sim | Teto de registros por consulta. |
| `aud.export.max_records` | int | 1000000 | 1000–100000000 | tenant | sim | Teto por exportação (`AIOS-AUD-0015`). |
| `aud.export.package_ttl_h` | int | 72 | 1–720 | tenant | sim | Validade do link do pacote probatório. |
| `aud.export.require_dual_approval` | bool | `true` | {true,false} | tenant | **não** | Aprovador distinto para exportação. |

## 7. Infraestrutura e dependências

| Chave | Tipo | Default | Escopo | Recarregável | Descrição |
|-------|------|---------|--------|--------------|-----------|
| `aud.policy.fail_mode` | enum | `closed` | global | sim | PEP de consulta e governança — *fail-closed*. |
| `aud.db.connection_string` | secret | — | global | **não** | PostgreSQL (schema `audit`); credencial dinâmica do `../021-Security/`. |
| `aud.db.synchronous_commit` | enum | `remote_apply` | global | **não** | Confirmação após aplicação na réplica — sustenta o RPO 0 (NFR-006). |
| `aud.nats.url` | string | — | global | **não** | Barramento (`../020-Communication/`). |
| `aud.minio.endpoint` | string | — | global | **não** | Armazenamento WORM. |
| `aud.outbox.poll_interval_ms` | int | 200 | global | sim | Intervalo do `audit-outbox-relay`. |
| `aud.otel.endpoint` | string | — | global | **não** | Coletor OTel (`../024-Observability/`). |

`aud.db.synchronous_commit = remote_apply` é o que transforma a promessa de RPO 0 em
configuração concreta: o commit só retorna após a réplica **aplicar** a alteração.

Segredos **NÃO DEVEM** vir de arquivo em imagem: são injetados em runtime pelo
`../021-Security/` (`./Deployment.md` §5).

## 8. As chaves que não existem

Algumas configurações **deliberadamente não existem** neste módulo:

| Configuração ausente | Por quê |
|----------------------|---------|
| "Permitir alteração de registro" | Uma capability de alteração é uma capability de apagar evidência (ADR-0258). |
| "Desabilitar encadeamento" | Sem cadeia, a imutabilidade volta a ser declarada, não verificável. |
| "Desabilitar registro de consulta" | Tornaria a consulta à trilha a única ação privilegiada não auditada (ADR-0257). |
| "Ignorar legal hold" | O *hold* sobrepõe retenção e RTBF por obrigação legal (ADR-0254). |
| "Modo rápido sem durabilidade" | Ver §1. |

Uma chave de configuração é um caminho de ataque documentado. Estas cinco não têm
forma segura de existir.

## 9. Precedência e validação

```
   default do produto
        ↓ sobrescrito por
   configuração global da instalação
        ↓ sobrescrito por
   configuração do tenant        (limitada pelo teto global)
        ↓ sobrescrito por
   configuração de record_class
```

Validações na carga (falham o *startup*, não a primeira escrita):

| # | Validação | Erro |
|---|-----------|------|
| V-01 | `aud.retention.default_days` ≥ `aud.retention.min_days` | *startup* recusado |
| V-02 | `aud.worm.object_lock_days` ≥ `aud.retention.min_days` | *startup* recusado (o arquivo não pode expirar antes do dado) |
| V-03 | `aud.seal.interval_s` < `aud.completeness.gap_alert_s` | *startup* recusado (senão todo intervalo pareceria atrasado) |
| V-04 | `aud.worm.archive_after_s` ≥ `aud.seal.interval_s` | *startup* recusado (não se arquiva o que não foi selado) |
| V-05 | `aud.db.synchronous_commit` ∈ {`remote_apply`, `remote_write`} | *startup* recusado — `local`/`off` violariam o RPO 0 |
| V-06 | `aud.seal.signing_key_ref` resolve para chave ativa no `../021-Security/` | *startup* recusado |
| V-07 | Chave não-recarregável alterada em runtime | ignorada + log `Error` + métrica |

> V-07 registra em `Error`, não em `Warning` como nos demais módulos: tentar alterar
> uma chave não-recarregável **deste** módulo é, por si, um sinal que merece
> investigação.

## 10. Exemplo — `appsettings.Production.json`

```json
{
  "Audit": {
    "Ingest":     { "RequireDurableAck": true, "BatchMaxSize": 500,
                    "MaxRatePerSPerTenant": 100000 },
    "Chain":      { "PartitionsPerTenant": 16, "HashAlgorithm": "sha256" },
    "Seal":       { "IntervalS": 60, "ExternalAnchorEnabled": false },
    "Verify":     { "IntervalS": 900, "SampleSize": 1000 },
    "Completeness": { "GapAlertS": 300, "SilenceRatio": 0.2 },
    "Retention":  { "DefaultDays": 1825, "MinDays": 365 },
    "Hold":       { "ReviewIntervalDays": 90, "RequireDualApproval": true },
    "Erasure":    { "MaxLatencyH": 24 },
    "Worm":       { "ObjectLockDays": 1825, "ArchiveAfterS": 3600 },
    "Query":      { "MaxDurationS": 30, "MaxRecords": 100000 },
    "Export":     { "MaxRecords": 1000000, "PackageTtlH": 72, "RequireDualApproval": true },
    "Policy":     { "FailMode": "closed" },
    "Db":         { "SynchronousCommit": "remote_apply" }
  }
}
```

Equivalente por variável de ambiente:

```bash
AIOS_AUD_INGEST_REQUIRE_DURABLE_ACK=true
AIOS_AUD_CHAIN_PARTITIONS_PER_TENANT=16
AIOS_AUD_SEAL_INTERVAL_S=60
AIOS_AUD_RETENTION_MIN_DAYS=365
AIOS_AUD_WORM_OBJECT_LOCK_DAYS=1825
AIOS_AUD_HOLD_REQUIRE_DUAL_APPROVAL=true
AIOS_AUD_DB_SYNCHRONOUS_COMMIT=remote_apply
```

## 11. Perfis por ambiente

| Chave | Produção | Homologação | Desenvolvimento |
|-------|----------|-------------|-----------------|
| `aud.ingest.require_durable_ack` | `true` | `true` | `true` |
| `aud.chain.partitions_per_tenant` | 16 | 8 | 1 |
| `aud.seal.interval_s` | 60 | 60 | 10 |
| `aud.verify.interval_s` | 900 | 900 | 60 |
| `aud.retention.default_days` | 1825 | 90 | 7 |
| `aud.retention.min_days` | 365 | 30 | 1 |
| `aud.worm.object_lock_days` | 1825 | 30 | 1 |
| `aud.hold.require_dual_approval` | `true` | `true` | `false` |
| `aud.policy.fail_mode` | `closed` | `closed` | `open` |

> `aud.ingest.require_durable_ack` é `true` em **todos** os ambientes, inclusive
> desenvolvimento. É a única garantia deste módulo que não admite relaxamento por
> conveniência: um desenvolvedor que aprende a conviver com perda silenciosa em
> desenvolvimento não reconhece o defeito em produção.

## 12. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §8
- Metas associadas: `./NonFunctionalRequirements.md`
- Erros: `./API.md` §6 · Implantação: `./Deployment.md`
- Segredos: `../021-Security/` · Operação: `./Monitoring.md`

*Fim de `Configuration.md`.*
