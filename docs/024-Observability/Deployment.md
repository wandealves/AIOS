---
Documento: Deployment
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0243, ADR-0244, ADR-0248
RFCs relacionados: RFC-0001, RFC-0240
Depende de: Architecture.md, Configuration.md, 021-Security, 027-Cluster, 028-Deployment
---

# 024-Observability — Implantação

## 1. Topologia

```
   ┌────────────────────────────────────────────────────────────────────────┐
   │  CADA NÓ DO CLUSTER                                                    │
   │  ┌──────────┐ ┌──────────┐ ┌──────────┐      ┌───────────────────────┐│
   │  │ serviço  │ │ serviço  │ │ serviço  │─────▶│ otel-agent (DaemonSet)││
   │  │ (006)    │ │ (022)    │ │ (007)    │ OTLP │ + buffer em disco 2GiB││
   │  └──────────┘ └──────────┘ └──────────┘local └───────────┬───────────┘│
   └────────────────────────────────────────────────────────┬─┴────────────┘
                                                            │ OTLP/mTLS
                        roteamento consistente por hash(trace_id)
                                                            ▼
        ┌───────────────────┬───────────────────┬───────────────────┐
        │ otel-gateway #1   │ otel-gateway #2   │ otel-gateway #3   │
        │ admissão·PII·tail │                   │                   │
        └─────────┬─────────┴─────────┬─────────┴─────────┬─────────┘
                  ▼                   ▼                   ▼
           ┌────────────┐      ┌────────────┐      ┌────────────┐
           │ Prometheus │      │ Tempo/     │      │    Seq     │
           │ + MinIO LT │      │ Jaeger     │      │            │
           └─────┬──────┘      └────────────┘      └────────────┘
                 │                                        ▲
   ┌─────────────▼──────────────────────────────────────┐ │
   │  observability-api (registro·SLO·alerta·consulta)  │─┘
   └──┬──────────────┬──────────────┬───────────────────┘
      ▼              ▼              ▼
  PostgreSQL   slo-evaluator   alert-evaluator ──▶ Alertmanager ──▶ 029-Operations
  (telemetry)                                  └─▶ NATS aios.*.telemetry.*
      ▲
      └── observability-outbox-relay (líder único)
```

Duas camadas de coleta (agente por nó → gateway central) porque os dois objetivos são
incompatíveis em uma só: o agente precisa estar **perto do emissor** (latência de
export ~1 ms) e o gateway precisa **ver o trace inteiro** para amostrar por cauda
(ADR-0244).

## 2. Contêineres e recursos

| Serviço | Imagem base | Réplicas (prod) | CPU (req/lim) | RAM (req/lim) | Disco |
|---------|-------------|-----------------|---------------|---------------|-------|
| `otel-agent` | OTel Collector (contrib) | 1 por nó (DaemonSet) | 0,25 / 1 vCPU | 256 / 512 MiB | 2 GiB (buffer) |
| `otel-gateway` | OTel Collector + processadores | 3 – 12 (HPA) | 2 / 6 vCPU | 4 / 12 GiB | efêmero |
| `observability-api` | .NET 10 (Alpine) | 2 – 8 | 1 / 3 vCPU | 1 / 3 GiB | — |
| `slo-evaluator` | .NET 10 | 2 | 1 / 2 vCPU | 1 / 2 GiB | — |
| `alert-evaluator` | .NET 10 | 2 | 1 / 2 vCPU | 1 / 2 GiB | — |
| `observability-outbox-relay` | .NET 10 | 2 (1 ativa) | 0,25 / 1 vCPU | 256 / 512 MiB | — |

Armazenamento (dimensionado em `../028-Deployment/`): Prometheus com retenção local de
15 d + objeto em MinIO para 400 d; Tempo/Jaeger com 7 d; Seq com 14 d.

### 2.1 Autoescala (HPA)

| Serviço | Métrica | Alvo |
|---------|---------|------|
| `otel-gateway` | CPU | 65% |
| `otel-gateway` | `aios_obs_queue_depth` | < 60% de `obs.ingest.queue_size` |
| `otel-gateway` | `aios_obs_dropped_total` (taxa) | ~0 |
| `observability-api` | p95 de consulta | ≤ 2 s (NFR-013) |

> Escalar o gateway **não** resolve explosão de cardinalidade: mais réplicas coletam
> mais séries, e o gargalo passa para o armazenamento. O sinal correto nesse caso é
> `aios_obs_active_series` e a ação é quarentenar o sinal infrator
> (`./Monitoring.md` RB-O-03).

## 3. Docker Compose (desenvolvimento)

```yaml
services:
  otel-gateway:
    image: aios/otel-gateway:0.1
    environment:
      AIOS_OBS_INGEST_FAIL_MODE: "open"
      AIOS_OBS_SIGNAL_REGISTRATION_REQUIRED: "false"   # apenas local (Configuration §11)
      AIOS_OBS_TRACE_HEAD_SAMPLE_RATIO: "1.0"
      AIOS_OBS_PII_REDACTION_ENABLED: "true"           # true em TODOS os ambientes
      AIOS_OBS_PROMETHEUS_REMOTE_WRITE_URL: "http://prometheus:9090/api/v1/write"
      AIOS_OBS_TRACE_STORE_ENDPOINT: "http://tempo:4317"
      AIOS_OBS_LOG_STORE_ENDPOINT: "http://seq:5341"
    ports: ["4317:4317", "4318:4318"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:13133/healthz/ready"]
      interval: 10s
      timeout: 2s
      retries: 3

  observability-api:
    image: aios/observability-api:0.1
    environment:
      AIOS_OBS_DB_CONNECTION_STRING: "Host=postgres;Database=aios;Username=telemetry"
      AIOS_OBS_POLICY_FAIL_MODE: "open"                # apenas local
      AIOS_OBS_ALERT_RUNBOOK_REQUIRED: "false"         # apenas local
    depends_on: [postgres, prometheus]
    ports: ["8080:8080", "8081:8081"]

  alert-evaluator:
    image: aios/alert-evaluator:0.1
    environment:
      AIOS_OBS_ALERT_NO_DATA_TIMEOUT_S: "600"
      AIOS_OBS_ALERTMANAGER_URL: "http://alertmanager:9093"
    depends_on: [prometheus, postgres]

  observability-outbox-relay:
    image: aios/observability-outbox-relay:0.1
    environment:
      AIOS_OBS_OUTBOX_POLL_INTERVAL_MS: "200"
    depends_on: [postgres, nats]
```

## 4. Health e readiness

| Endpoint | Verifica | Falha significa |
|----------|----------|-----------------|
| `GET /healthz/live` | Processo respondendo | Reiniciar contêiner |
| `GET /healthz/ready` (`otel-gateway`) | Receptores OTLP escutando; fila abaixo do limiar; destinos de escrita alcançáveis **ou** buffer disponível | Retirar do balanceamento |
| `GET /healthz/ready` (`observability-api`) | PostgreSQL alcançável; registro de sinais carregado; PDP alcançável | Retirar do balanceamento |
| `GET /healthz/startup` | Migrações aplicadas + registro de sinais em cache | Ainda inicializando |
| `GET /metrics` (`SelfTelemetry`) | **Sempre responde**, mesmo com o pipeline saturado | — |

> **Regra crítica de readiness do `otel-gateway`:** destino de escrita indisponível
> **NÃO DEVE**, sozinho, tirar a réplica do balanceamento enquanto houver buffer. Se
> tirasse, a queda do Prometheus derrubaria toda a ingestão — e o emissor, que não
> deveria nem perceber, perderia também os traces e os logs. A degradação é **por
> destino**, não global (`./FailureRecovery.md` §3).
>
> `SelfTelemetry` (`GET /metrics`) é *scrapeado* **diretamente** pelo Prometheus, sem
> passar pelo pipeline. Se dependesse do pipeline, a única falha que ele não
> conseguiria reportar seria exatamente a falha do pipeline.

## 5. Segredos e identidade

| Segredo | Origem | Rotação | Injeção |
|---------|--------|---------|---------|
| Credencial PostgreSQL | Credencial dinâmica do `../021-Security/` (`db_role`) | ≤ 1 h | Arquivo em `tmpfs` |
| Credencial Redis | `../021-Security/` | 90 d | Variável de ambiente |
| Conta NKey/JWT do NATS | `../021-Security/` (`nats_account`) | 7 d | Arquivo em `tmpfs` |
| Certificado mTLS de workload (gateway e agentes) | PKI do `../021-Security/`, após atestação | ≤ 24 h | Socket local do agente de identidade |
| Chave de tokenização de PII | `../021-Security/` (`signing_key`) | 90 d | Memória, nunca em disco |
| Credenciais dos armazenamentos (Prometheus/Tempo/Seq) | `../021-Security/` | 90 d | Arquivo em `tmpfs` |

A **chave de tokenização de PII** merece nota: ela permite correlacionar ocorrências do
mesmo titular sem revelar a identidade. Rotacioná-la quebra a correlação histórica —
por isso a rotação é programada, com janela de sobreposição gerida pelo
`../021-Security/`, e não sob demanda.

## 6. Estratégia de rollout

| Etapa | Ação | Critério de avanço |
|-------|------|--------------------|
| 1 | Aplicar migrações (expand, CV-11) | DDL < 200 ms; `runbook` antes de `alert_rule` (`./Database.md` §13) |
| 2 | Rollout do `otel-agent` (DaemonSet, *surge* de 10%) | Sem aumento de `dropped_total`; buffer drenando |
| 3 | Rollout canário do `otel-gateway` (1 réplica) | p99 de ingestão ≤ 50 ms; sem fragmentação de trace |
| 4 | Rollout do `observability-api`, `slo-evaluator`, `alert-evaluator` | Sem alerta `ObservabilityAlertPipelineDown` |
| 5 | Contract das migrações | Após 2 releases sem rollback |

**Ordem obrigatória:** o `otel-agent` sobe **antes** do `otel-gateway` em uma
atualização de protocolo, e desce **depois** — o agente tem buffer, o gateway não.
Inverter a ordem produz perda de telemetria exatamente durante a janela em que ela é
mais útil para avaliar o próprio rollout.

**Rollout do `otel-gateway` e *tail sampling*:** durante a troca de réplicas, spans do
mesmo trace podem cair em gateways diferentes, produzindo traces fragmentados. Isso é
**esperado e temporário**; a métrica `aios_obs_trace_fragmented_total` sobe e volta ao
normal. Não é motivo de rollback.

## 7. Ordem de bootstrap da instalação

```
   1. 005-Database   → schema `telemetry`, migrações (runbook antes de alert_rule)
   2. 021-Security   → PKI ativa; certificados de workload para agentes e gateways
   3. armazenamentos → Prometheus, Tempo/Jaeger, Seq, Alertmanager
   4. 024 gateways   → recebem OTLP; registro de sinais de plataforma carregado
   5. 024 agentes    → DaemonSet em cada nó
   6. demais módulos → passam a emitir telemetria
   7. 022-Policy     → PEP de consulta passa a autorizar
```

> Entre os passos 4 e 6 o sistema já observa a si mesmo. Antes do passo 4, **não há
> telemetria** — e é por isso que a ordem importa: subir os módulos de negócio antes
> do coletor significa perder exatamente a janela de bootstrap, que é quando mais
> falhas de configuração aparecem.

## 8. Dependências de implantação

| Dependência | Tipo | Falha na implantação significa |
|-------------|------|-------------------------------|
| PostgreSQL (`telemetry`) | Forte para administração | `observability-api` não fica *ready*; **ingestão continua** com registro em cache |
| `../021-Security/` (PKI) | Forte | Gateways não aceitam emissores (mTLS) |
| Prometheus | Forte para métricas | Métricas represadas no buffer; traces e logs seguem |
| Tempo/Jaeger | Fraca | Traces descartados após o buffer; demais sinais seguem |
| Seq | Fraca | Logs represados; demais sinais seguem |
| Alertmanager | Forte para notificação | Alertas avaliados e persistidos; entrega atrasada |
| NATS/JetStream | Fraca | Eventos represados no outbox |
| `../022-Policy/` | Forte para consulta | Consulta negada (`fail_mode=closed`); **ingestão não é afetada** |
| Redis | Fraca | Contagem de cardinalidade degrada para amostragem local |

## 9. Referências

- Arquitetura: `./Architecture.md` · Configuração: `./Configuration.md`
- Falhas: `./FailureRecovery.md` · Escala: `./Scalability.md`
- Plataforma: `../027-Cluster/`, `../028-Deployment/`, `../029-Operations/`
- Segredos e identidade: `../021-Security/`

*Fim de `Deployment.md`.*
