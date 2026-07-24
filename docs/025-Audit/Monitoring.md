---
Documento: Monitoring
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit + SRE
ADRs relacionados: ADR-0010, ADR-0251, ADR-0254, ADR-0255, ADR-0256
RFCs relacionados: RFC-0001, RFC-0250
Depende de: Metrics.md, NonFunctionalRequirements.md, FailureRecovery.md, 024-Observability, 029-Operations
---

# 025-Audit — Monitoramento

> A telemetria deste módulo é servida pelo `../024-Observability/`. A fronteira vale
> nos dois sentidos: o 024 **observa** o 025 (métricas, alertas), e o 025 **registra**
> os fatos de governança do 024 (silêncios, mudanças de SLO). Nenhum dos dois substitui
> o outro (ADR-0246).

## 1. Golden signals

| Sinal | Métrica | Alvo |
|-------|---------|------|
| **Latência** | `aios_aud_append_latency_p99` | ≤ 20 ms (NFR-001) |
| **Tráfego** | `rate(aios_aud_records_total[1m])` | ≥ 50.000/s por réplica (NFR-002) |
| **Erros** | `aios_aud_durable_reject_ratio`, `aios_aud_ingest_rejected_total` | ~0 fora de incidente |
| **Saturação** | `aios_aud_unsealed_records`, `aios_aud_consumer_lag_s` | não-selados < 3× o intervalo; lag < 30 s |

> Leitura contraintuitiva: `aios_aud_durable_ack_failed_total` **não é uma métrica de
> erro do módulo** — é a recusa funcionando (ADR-0255). O que se monitora é a
> **persistência** da recusa, que indica problema no banco, não no serviço.

## 2. Rastreamento (traces)

100% das operações geram span OTel (NFR-017), com atributos:

| Atributo do span | Origem |
|------------------|--------|
| `aios.tenant` | `X-AIOS-Tenant` ou `aios.tenant` do evento |
| `aios.audit.record_class` | Classe do registro |
| `aios.audit.partition_key` | Partição da cadeia |
| `aios.audit.seq` | Sequência atribuída |
| `aios.audit.producer_module` | Produtor do fato |
| `aios.audit.outcome` | Resultado do fato registrado |

Spans filhos: `aud.normalize`, `aud.cipher`, `aud.chain.append`, `aud.commit`,
`aud.seal.merkle`, `aud.seal.sign`, `aud.verify.sample`, `aud.query`.

**Nunca** vira atributo de span: `subject_ref`, `record_hash`, conteúdo do payload
(`./Security.md` §5).

## 3. SLO / SLI e error budget

| SLO | SLI | Janela | Error budget |
|-----|-----|--------|--------------|
| Disponibilidade ≥ 99,95% (NFR-003) | Fração de escritas confirmadas sem `5xx` não atribuível ao banco | 30 d | 21 min/mês |
| p99 de escrita ≤ 20 ms (NFR-001) | `aios_aud_append_latency_p99` | 7 d | 1% das janelas de 5 min |
| Selagem ≤ 60 s (NFR-004) | `aios_aud_seal_lag_s` p99 | 7 d | 1% dos registros |
| **Integridade 100%** (NFR-007) | `aios_aud_chain_break_total` + `aios_aud_worm_mismatch_total` | contínua | **0** (não há budget) |
| **Completude 100%** (NFR-005) | `aios_aud_sequence_gap_total` | contínua | **0** (não há budget) |
| **Durabilidade RPO 0** (NFR-006) | Registros confirmados perdidos | contínua | **0** (não há budget) |

Três SLOs sem error budget — nenhum outro módulo do AIOS tem essa característica. São
garantias, não metas: um único rompimento invalida a trilha como prova.

## 4. Alertas Prometheus

```yaml
groups:
- name: audit-critical
  rules:
  - alert: AuditChainBreak
    expr: increase(aios_aud_chain_break_total[5m]) > 0
    for: 0m
    labels: { severity: critical, page: immediate, security: true }
    annotations:
      summary: "QUEBRA DE CADEIA — integridade da trilha comprometida"
      runbook: "./Monitoring.md#rb-a-01"

  - alert: AuditWormMismatch
    expr: increase(aios_aud_worm_mismatch_total[15m]) > 0
    for: 0m
    labels: { severity: critical, page: immediate, security: true }
    annotations:
      summary: "Divergência entre banco e cópia WORM — provável adulteração"
      runbook: "./Monitoring.md#rb-a-02"

  - alert: AuditSequenceGap
    expr: increase(aios_aud_sequence_gap_total[5m]) > 0
    for: 0m
    labels: { severity: critical, page: immediate }
    annotations:
      summary: "Lacuna de sequência — registro pode ter sido perdido"
      runbook: "./Monitoring.md#rb-a-03"

  - alert: AuditDurableRejectSustained
    expr: aios_aud_durable_reject_ratio > 0.01
    for: 5m
    labels: { severity: critical }
    annotations:
      summary: "Escritas recusadas por durabilidade — produtores acumulando outbox"
      runbook: "./Monitoring.md#rb-a-05"

  - alert: AuditVerifyStale
    expr: aios_aud_verify_staleness_s{mode="sample"} > 3600
    for: 15m
    labels: { severity: critical }
    annotations:
      summary: "Verificação de integridade parada há mais de 1 h"
      runbook: "./Monitoring.md#rb-a-06"

- name: audit-warning
  rules:
  - alert: AuditSealOverdue
    expr: aios_aud_unsealed_total > 0 and histogram_quantile(0.99, rate(aios_aud_seal_lag_s_bucket[10m])) > 60
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "Selagem atrasada (NFR-004)", runbook: "./Monitoring.md#rb-a-04" }

  - alert: AuditClassSilent
    expr: aios_aud_class_volume_ratio < 0.2
    for: 30m
    labels: { severity: warning }
    annotations: { summary: "Classe auditável silenciosa — produtor pode ter parado de emitir", runbook: "./Monitoring.md#rb-a-07" }

  - alert: AuditAppendLatencyHigh
    expr: aios_aud_append_latency_p99 > 20
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "p99 de escrita durável acima do SLO (NFR-001)", runbook: "./Monitoring.md#rb-a-08" }

  - alert: AuditArchiveLag
    expr: histogram_quantile(0.99, rate(aios_aud_archive_lag_s_bucket[30m])) > 7200
    for: 30m
    labels: { severity: warning }
    annotations: { summary: "Arquivamento WORM atrasado — 3ª fonte de verificação indisponível", runbook: "./Monitoring.md#rb-a-09" }

  - alert: AuditHoldWithoutExpiryAging
    expr: aios_aud_holds_active{has_expiry="false"} > 0 and histogram_quantile(0.99, aios_aud_hold_age_days) > 90
    for: 24h
    labels: { severity: warning }
    annotations: { summary: "Legal hold sem prazo há mais de 90 dias — revisão obrigatória", runbook: "./Monitoring.md#rb-a-10" }

  - alert: AuditErasureLatencyHigh
    expr: histogram_quantile(0.95, rate(aios_aud_erasure_latency_h_bucket[24h])) > 24
    for: 1h
    labels: { severity: warning }
    annotations: { summary: "Apagamento criptográfico acima do prazo (NFR-011)", runbook: "./Monitoring.md#rb-a-11" }

  - alert: AuditConsumerLagHigh
    expr: aios_aud_consumer_lag_s > 300
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "Consumidor JetStream atrasado — fatos ainda não registrados", runbook: "./Monitoring.md#rb-a-12" }

  - alert: AuditQueryDeniedSpike
    expr: rate(aios_aud_query_denied_total[15m]) > 1
    for: 15m
    labels: { severity: warning, security: true }
    annotations: { summary: "Consultas à trilha sendo negadas repetidamente — possível sondagem", runbook: "./Monitoring.md#rb-a-13" }
```

## 5. Runbooks

| ID | Alerta | Procedimento |
|----|--------|--------------|
| **RB-A-01** | `AuditChainBreak` | 1) **Incidente de segurança P1**; acionar CISO imediatamente. 2) **Congelar a escrita da partição afetada** para preservar o estado. 3) Comparar o registro com a raiz de Merkle do **selo assinado** e com a **cópia WORM** — duas fontes independentes do banco. 4) Se WORM e selo concordam entre si e divergem do banco: **adulteração no banco**; a evidência íntegra é a do WORM. 5) Preservar *logs* de acesso ao banco e verificar se o `tr_record_immutable` foi removido. 6) **Não** "corrigir" o registro: a conclusão entra como **registro de compensação**. |
| **RB-A-02** | `AuditWormMismatch` | 1) **P1 de segurança**. 2) O objeto WORM não pode ser sobrescrito — a divergência indica alteração **no banco** ou defeito de serialização. 3) Verificar se houve mudança na serialização canônica (RFC-0250) sem nova versão de RFC — causa não maliciosa mais provável. 4) Se não houve, tratar como RB-A-01. |
| **RB-A-03** | `AuditSequenceGap` | 1) **P1**: um registro pode ter sido perdido — o defeito mais grave possível aqui. 2) Verificar `aios_aud_durable_ack_failed_total` no período: recusas indicam que o produtor **sabe** e vai reenviar (menos grave). 3) Verificar se houve `DELETE` no banco (o *trigger* deveria ter impedido). 4) Correlacionar com o produtor esperado da classe: o fato existiu? 5) Registrar a conclusão em `audit.anomaly.resolution` — uma lacuna fechada sem explicação é pior que uma aberta. |
| **RB-A-04** | `AuditSealOverdue` | 1) Verificar `aios_aud_signing_latency_ms` e a disponibilidade da chave no `../021-Security/`. 2) Se o KMS está fora, a selagem **retomará retroativamente** — a ingestão não foi afetada. 3) Verificar o `audit-sealer` (eleição de líder por partição). 4) A janela sem selo é janela de risco documentada: registrar o período. |
| **RB-A-05** | `AuditDurableRejectSustained` | 1) A recusa está **correta** — o problema é a durabilidade. 2) Verificar a **réplica síncrona** do PostgreSQL (`synchronous_commit = remote_apply`): latência de rede, réplica fora. 3) Verificar `aios_aud_commit_share`: se dominar a latência, é o banco. 4) Comunicar aos produtores que os outbox estão crescendo. 5) **Não** desligar `require_durable_ack` como contorno — a chave é não-recarregável exatamente por isso. |
| **RB-A-06** | `AuditVerifyStale` | 1) Sem verificação, uma adulteração passaria despercebida indefinidamente. 2) Verificar o `audit-verifier` e a réplica de leitura. 3) Executar verificação completa manual na partição mais recente ao restabelecer. |
| **RB-A-07** | `AuditClassSilent` | 1) Identificar `record_class` e `producer_module`. 2) Confirmar com o dono se o produtor está operando — **a ausência de registro pode significar ausência de auditoria, não de atividade**. 3) Se o produtor foi descomissionado, marcar a classe `deprecated` com registro do motivo. 4) **Não** simplesmente reduzir `expected_rate_per_day` para silenciar. |
| **RB-A-08** | `AuditAppendLatencyHigh` | 1) Verificar `aios_aud_commit_share`: dominante ⇒ ir para RB-A-05. 2) Verificar CPU do `audit-ingest` (hash + cifragem). 3) Verificar contenção de `seq` por partição — considerar aumentar `aud.chain.partitions_per_tenant` (**exige migração**). 4) Escalar réplicas se a saturação for de CPU. |
| **RB-A-09** | `AuditArchiveLag` | 1) Verificar MinIO e a credencial do bucket. 2) Enquanto durar, a verificação completa opera com **duas** fontes (banco + selo) em vez de três. 3) Se persistir além de 24 h, escalar: o expurgo por retenção **depende** do arquivamento prévio e ficará bloqueado. |
| **RB-A-10** | `AuditHoldWithoutExpiryAging` | 1) Listar os *holds* sem prazo com `case_ref`. 2) Levar ao jurídico: o caso ainda está aberto? 3) *Hold* cujo caso encerrou **DEVE** ser liberado — mantê-lo indefinidamente preserva dado pessoal sem base atual. 4) Registrar a revisão, mesmo quando a conclusão é manter. |
| **RB-A-11** | `AuditErasureLatencyHigh` | 1) **Risco de conformidade**: o prazo do RTBF está sendo excedido. 2) Verificar a fila do `CryptoShredder` e a disponibilidade do KMS para destruição de chave. 3) Verificar se há *holds* bloqueando — nesse caso não é atraso, é recusa legítima (`aios_aud_erasure_blocked_total`). |
| **RB-A-12** | `AuditConsumerLagHigh` | 1) Fatos ocorridos ainda não estão na trilha — janela de cegueira. 2) Verificar o `audit-ingest` e o JetStream. 3) O dado **não está perdido** (JetStream retém), mas uma investigação no período veria a trilha incompleta. |
| **RB-A-13** | `AuditQueryDeniedSpike` | 1) Tratar como **sinal de segurança**: alguém está tentando ler a trilha sem autorização. 2) Identificar o princípio pelas negações registradas (a negação **é** um registro). 3) Correlacionar com `../021-Security/` (anomalia de autenticação) e `../022-Policy/` (decisões negadas). 4) Escalar ao CISO se o padrão persistir. |

## 6. Dashboards

| Painel | Conteúdo | Público |
|--------|----------|---------|
| **Audit — Ingestão** | Registros/s por classe e produtor, p99 de escrita, parcela de commit, recusas por durabilidade, lag de consumidor | SRE |
| **Audit — Integridade** | `chain_break`, `worm_mismatch`, `sequence_gap` (todos devem ser 0), idade da última verificação, registros verificados | Segurança + SRE |
| **Audit — Selagem e arquivo** | Não-selados por partição, `seal_lag`, latência de assinatura, atraso de arquivamento | SRE |
| **Audit — Governança** | *Holds* ativos (com e sem prazo), idade dos *holds*, apagamentos e bloqueios, exportações | DPO + jurídico |
| **Audit — Completude** | Volume por classe vs. esperado, classes silenciosas, produtores por classe | Plataforma |
| **Audit — Acesso** | Consultas por tipo de princípio, negações, exportações solicitadas | Segurança + auditoria |

> O painel **Integridade** é o único do AIOS em que "tudo zero" é o resultado
> desejado. Um valor diferente de zero em qualquer uma das três séries não é
> degradação — é a trilha deixando de valer como prova.

## 7. Referências

- Métricas: `./Metrics.md` · Logs: `./Logging.md`
- Metas: `./NonFunctionalRequirements.md` · Falhas: `./FailureRecovery.md`
- Plantão e escalonamento: `../029-Operations/` · Plataforma: `../024-Observability/`
- Segurança: `./Security.md` §3 (defesa em profundidade)

*Fim de `Monitoring.md`.*
