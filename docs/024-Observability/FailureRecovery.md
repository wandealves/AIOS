---
Documento: FailureRecovery
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability + SRE
ADRs relacionados: ADR-0010, ADR-0242, ADR-0243, ADR-0244, ADR-0248
RFCs relacionados: RFC-0001, RFC-0241
Depende de: _DESIGN_BRIEF.md §9, Monitoring.md, StateMachine.md, Deployment.md
---

# 024-Observability — Falhas e Recuperação

## 1. Princípio

Toda falha deste módulo resolve-se **preservando o sistema observado**. O 024 é
deliberadamente ***fail-open*** no caminho de ingestão (ADR-0243) — o oposto do
`../022-Policy/`, que é *fail-closed*. Sob saturação, descarta-se telemetria e
contabiliza-se o descarte; **jamais** se aplica backpressure ao emissor.

A consequência aceita é que uma falha do 024 produz **cegueira**, não indisponibilidade
— e cegueira é perigosa justamente por ser silenciosa. Por isso o módulo tem dois
controles que outros não têm: `aios_obs_dropped_total` (torna a perda audível) e o
`SelfTelemetry` fora de banda (§6).

## 2. FMEA

| # | Modo de falha | Efeito | Sev. | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---|---------------|--------|------|----------|-----------|-------------|--------------------|
| F-01 | Saturação do pipeline de ingestão | Perda de telemetria | Alta | `aios_obs_queue_depth`, `drop_ratio` | **Descarte contabilizado**; emissor não é bloqueado | HPA do `otel-gateway`; drena buffer do agente | Reenvio pelo SDK é opcional e limitado |
| F-02 | Explosão de cardinalidade | Armazenamento saturado; consultas lentas | Alta | `CardinalityGuard`, `cardinality_pressure` | Quarentena **do sinal infrator**, nunca do tenant | Dono corrige o descritor; reabilita (`Quarantined → Registered`) | Determinístico |
| F-03 | Prometheus indisponível | Métricas represadas | Média | Health/scrape | Métricas no buffer; **traces e logs seguem** | Failover; drenagem | Escrita idempotente por timestamp |
| F-04 | Tempo/Jaeger indisponível | Traces descartados após buffer | Média | Health | Traces isolados; métricas e logs seguem | Failover; retomada | Dedupe por `trace_id`+`span_id` |
| F-05 | Seq indisponível | Logs represados | Média | Health | Logs isolados; demais sinais seguem | Failover; drenagem | Dedupe por `event.id` |
| F-06 | PostgreSQL (`telemetry`) indisponível | Administração e SLO param | Média | Health/readiness | **Ingestão continua** com registro em cache | Failover do `../005-Database/`; reconciliação | Idempotente |
| F-07 | Alertmanager indisponível | Notificação atrasada | **Crítica** | Health/probe sintético | Alertas avaliados e **persistidos** | Retomada; entrega dos pendentes | Dedupe por `fingerprint` |
| F-08 | `alert-evaluator` parado | **Nenhum alerta do AIOS dispara** | **Crítica** | `ObservabilityAlertPipelineDown` [fora de banda] | — (falha global de detecção) | Redeploy; vigilância manual enquanto durar (RB-O-01) | Reavaliação idempotente |
| F-09 | `SelfTelemetry` indisponível | Observador cego sobre si | **Crítica** | Scrape direto falha | — | Redeploy; métricas de ingestão tratadas como não confiáveis | — |
| F-10 | *Tail sampling* fragmentado | Traces incompletos | Baixa | `trace_fragmented_total` | Trace individual | Roteamento consistente restabelecido; esperado durante rollout | — |
| F-11 | Silêncio esquecido em aberto | Cegueira sobre um alerta real | Média | Varredura de `expires_at`; `SilenceChurnHigh` | — | Expiração automática (teto 72 h); revisão semanal | Determinístico |
| F-12 | Alerta *flapping* | Fadiga de alerta → silêncio | Média | `alert_flapping_total` | Histerese (`for_duration`) e confirmação na resolução | Ajuste do limiar em nova versão da regra | — |
| F-13 | Ausência de dado como saúde | Falha invisível | **Crítica** | T-09 → `Expired` + `alert.nodata` | Estado e evento próprios | Investigar alvo antes de assumir normalidade | — |
| F-14 | Consulta abusiva | Armazenamento lento para todos | Média | `query_aborted_total` | Aborto da consulta, não do serviço | Ajuste do painel; *recording rule* | Retriable |
| F-15 | PDP (022) indisponível | Consulta e administração negadas | Média | Timeout/CB no `PolicyClient` | **Ingestão não é afetada** | Retry após meia-abertura | Retriable |
| F-16 | *Downsampling* atrasado além da retenção raw | **Perda definitiva** de resolução histórica | Alta | `DownsampleLag` | — | Escalar P1 antes de a camada `raw` expurgar (15 d) | — |
| F-17 | Chave de tokenização de PII rotacionada sem sobreposição | Correlação histórica quebrada | Média | Salto em `pii_redacted_total` sem correlação | — | Janela de sobreposição gerida pelo `../021-Security/` | — |

## 3. Detecção e isolamento por destino

```
   ┌───────────────────────────────────────────────────────────────────┐
   │  otel-gateway                                                     │
   │                                                                   │
   │   fila ──┬──▶ [métricas] ──[CB]──▶ Prometheus   ✗ down            │
   │          │        └── represa no buffer, NÃO derruba os demais    │
   │          ├──▶ [traces]  ──[CB]──▶ Tempo         ✓                 │
   │          └──▶ [logs]    ──[CB]──▶ Seq           ✓                 │
   └───────────────────────────────────────────────────────────────────┘
```

Cada destino tem *circuit breaker* próprio (bulkhead): a queda do Prometheus **NÃO
DEVE** interromper a ingestão de traces e logs. Por isso a readiness do
`otel-gateway` **não** falha por destino indisponível enquanto houver buffer
(`./Deployment.md` §4) — se falhasse, a queda de um armazenamento derrubaria toda a
coleta, e o emissor, que não deveria nem perceber, perderia os três sinais.

## 4. Retries, backoff e DLQ

| Operação | Retry | Backoff | DLQ |
|----------|-------|---------|-----|
| Export OTLP (emissor → agente) | Sim, limitado pelo SDK | Exponencial curto | Buffer em disco do agente |
| Agente → gateway | Sim | Exponencial (200 ms → 30 s) | Buffer em disco (2 GiB) |
| Gateway → armazenamento | Sim | Exponencial com jitter | Descarte após esgotar o buffer, **contabilizado** |
| Publicação de outbox | Sim | Exponencial (200 ms → 30 s) | `TELEMETRY_DLQ` após 10 tentativas |
| Consumo de evento | Sim | Exponencial | DLQ do stream de origem (`../020-Communication/`) |
| Entrega de notificação | Sim | Exponencial por canal | Canal de fallback; alerta `NotifyLatencyHigh` |

> Não há retry **do gateway para o emissor**: o gateway nunca pede que o emissor
> reenvie. Fazê-lo transformaria a saturação do observador em carga adicional no
> observado — exatamente o efeito que o fail-open existe para evitar.

Mensagens em `TELEMETRY_DLQ` exigem inspeção humana: um `telemetry.alert.firing` não
publicado significa que `../029-Operations/` não foi notificado por evento, ainda que
a notificação direta pelo Alertmanager tenha ocorrido.

## 5. Degradação graciosa

Ordem de sacrifício, do primeiro ao último:

```
   1. Traces amostrados          ← perde-se profundidade de depuração
   2. Logs de nível baixo        ← perde-se contexto fino
   3. Métricas de baixa prioridade (retention_tier = long)
   4. Consulta e administração   ← não se muda a observabilidade; ainda se observa
   5. MÉTRICAS DE SLO            ← perde-se a medida de conformidade
   6. CAMINHO DE ALERTA          ← nunca; ninguém saberia que há algo errado
```

Um sistema sem traces é difícil de depurar. Um sistema sem **alerta** é um sistema em
que ninguém sabe que há o que depurar — por isso o caminho de alerta tem SLO mais alto
(99,95%) que o de ingestão (99,9%) e é o último a cair.

## 6. Quem observa o observador

```
   ┌──────────────────────────────────────────────────────────────┐
   │  pipeline de ingestão (pode saturar, pode cair)              │
   │      emissores → agente → gateway → armazenamentos           │
   └──────────────────────────────────────────────────────────────┘
                            ⊘  (SEM dependência)
   ┌──────────────────────────────────────────────────────────────┐
   │  SelfTelemetry  ──scrape DIRETO──▶  Prometheus               │
   │  · queue_depth · drop_ratio · saúde dos destinos             │
   │  · probe sintético de alerta (1×/min)                        │
   └──────────────────────────────────────────────────────────────┘
```

Se a meta-observabilidade dependesse do pipeline, a **única** falha que ela não
conseguiria reportar seria exatamente a falha do pipeline. Os alertas
`ObservabilityAlertPipelineDown` e `ObservabilitySelfTelemetryDown`
(`./Monitoring.md` §4) dependem apenas desse caminho fora de banda.

O **probe sintético de alerta** fecha o círculo: uma vez por minuto, uma condição
artificial dispara um alerta de teste e mede o tempo até a notificação. É a única
verificação possível de uma capacidade cuja falha é, por definição, silenciosa.

## 7. RTO / RPO

| Cenário | RTO | RPO | Procedimento |
|---------|-----|-----|--------------|
| Perda de réplica do gateway | ~0 (balanceador) | ~0 (buffer do agente) | Réplicas restantes atendem; HPA repõe |
| Perda total do gateway | ≤ 15 min | **≤ 1 min** | Buffer em disco do agente cobre a janela (`obs.ingest.disk_buffer_mb`) |
| Perda do PostgreSQL | ≤ 15 min | ≤ 5 min | Failover; ingestão seguiu com registro em cache |
| Perda do Prometheus | ≤ 15 min | ≤ 1 min | Buffer do gateway; drenagem após failover |
| Perda do `alert-evaluator` | **≤ 15 min** | 0 (estado em banco) | Redeploy; reavaliação reconstrói instâncias abertas |
| Perda de região | ≤ 15 min | ≤ 5 min (controle) / 1 min (bruto) | `../027-Cluster/`; SLOs e regras são artefatos replicados |

O RPO é **diferenciado por desenho** (NFR-011): 5 min para o **plano de controle**
(SLOs, regras, alertas, silêncios — reconstruíveis do banco) e 1 min para
**telemetria bruta** (limitado pelo buffer em disco). Exigir RPO de 5 min para o dado
bruto tornaria o pipeline mais caro e mais frágil que o sistema que ele observa.

## 8. Testes de caos

| Experimento | Hipótese | Critério de sucesso |
|-------------|----------|---------------------|
| Saturar o gateway com 5× a carga nominal | Emissores não percebem | Latência do emissor inalterada; `dropped_total` reflete a perda; `ingest.degraded` emitido |
| Derrubar o Prometheus por 10 min | Traces e logs seguem | 0 perda de trace/log; métricas drenadas do buffer |
| Derrubar o PostgreSQL por 10 min | Ingestão continua | 0 erro de ingestão; administração retorna `AIOS-OBS-0011` |
| Introduzir label de identidade em um sinal | Quarentena, não colapso | Sinal quarentenado; demais sinais do módulo intactos |
| Derrubar o `alert-evaluator` | Detectado fora de banda | `ObservabilityAlertPipelineDown` em ≤ 5 min pelo probe sintético |
| Remover um alvo de coleta sem `node.decommissioned` | `Expired`, não `Resolved` | `telemetry.alert.nodata` emitido; **0** transições a `Resolved` |
| Rollout do gateway com tráfego | Fragmentação temporária | `trace_fragmented_total` sobe e normaliza; sem perda de métrica |
| Injetar PII em sinal `operational` | Redação atua | Nenhuma PII persistida; `pii_undeclared_total` incrementado |

O sexto experimento é o mais importante: verifica a invariante I6 na prática. Se um
alvo removido produzir `Resolved`, o sistema de alerta está mentindo — e mentindo
exatamente no caso em que a falha é real.

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §9
- Alertas e runbooks: `./Monitoring.md` §4 e §5
- FSM e invariantes: `./StateMachine.md` §5
- Implantação e readiness: `./Deployment.md` §4 · Escala: `./Scalability.md`

*Fim de `FailureRecovery.md`.*
