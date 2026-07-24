---
Documento: RFC
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0240, ADR-0241, ADR-0242, ADR-0244, ADR-0245, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: 003-RFC, _DESIGN_BRIEF.md §11
---

# 024-Observability — Índice de RFCs

RFCs são registradas em `../003-RFC/`. Este documento lista as que afetam o módulo e o
impacto de cada uma.

---

## 1. RFCs consumidas (baseline)

| RFC | Título | Status | Impacto no 024-Observability |
|-----|--------|--------|------------------------------|
| **RFC-0001** | [Architecture Baseline & Core Contracts](../003-RFC/RFC-0001-Architecture-Baseline.md) | Accepted | **Baseline herdada, não redefinida.** A **§5.6** (cabeçalhos de correlação W3C Trace Context) e a **§5.8** (*todo serviço DEVE emitir traces/metrics/logs OTel com os campos de correlação*) são os contratos que este módulo torna executáveis. Além delas: URN (§5.1) para descritores, SLOs, regras e alertas; envelope CloudEvents (§5.2–§5.3) para o domínio `telemetry`; envelope de erro (§5.4) para o domínio `OBS`; idempotência (§5.5) nas mutações administrativas; versionamento (§5.7); mTLS e fronteira de tenant (§6); **minimização de dados pessoais (§7)**, que fundamenta a redação na borda; registros IANA-like (§8). |

O módulo **não reescreve** nenhum desses contratos. A RFC-0001 §5.8 diz *que* todo
serviço emite telemetria correlacionada; este módulo especifica **como o sinal é
declarado, admitido, governado, armazenado, medido contra objetivo e transformado em
alerta**.

---

## 2. RFCs a propor por este módulo

| RFC | Título | Escopo normativo | Relação com RFC-0001 | Status |
|-----|--------|------------------|----------------------|--------|
| **RFC-0240** | AIOS Telemetry Signal & Semantic Conventions | Gramática de nomes (`aios_<subsistema>_<nome>_<unidade>` para métrica; `<componente>.<evento>` para log); tipos e unidades permitidos; **atributos de recurso canônicos obrigatórios** (`service.name`, `service.version`, `deployment.environment`, `aios.tenant`, `aios.replica`, `aios.shard`); conjunto fechado de labels por sinal e a **lista de labels proibidos por identidade**; regras de buckets explícitos alinhados a limiares de SLO; classificação de dado (`operational`/`pii`/`sensitive`) e sua ligação com redação e retenção; formato do `SignalDescriptor` e seu ciclo de vida. | Especializa §5.6 e §5.8 para o plano de telemetria; consome §5.1, §5.4, §5.5 e §7 sem redefini-los. | Proposed |
| **RFC-0241** | AIOS SLO & Alerting Contract | Definição de SLI e SLO versionados; cálculo e semântica do **error budget** e do ledger; limiares canônicos de *multi-window multi-burn-rate*; ciclo de vida do alerta (`AlertState`, T-01..T-09, invariantes I1..I7); semântica de `Expired` (ausência de dado ≠ resolução); regras de deduplicação por `fingerprint`, silêncio com prazo e escalonamento; contrato do payload de alerta, **incluindo a obrigatoriedade do campo `runbook`**. | Usa o envelope de evento (§5.2–§5.3) para `telemetry.alert.*`, `telemetry.slo.*` e `telemetry.budget.*`. | Proposed |

---

## 3. RFCs de outros módulos que este módulo consome

| RFC | Módulo | Por que importa aqui |
|-----|--------|----------------------|
| RFC-0210 | `../021-Security/` | Identidade e credenciais dos emissores; atributos assertáveis usados no PEP de consulta. |
| RFC-0211 | `../021-Security/` | Identidade de workload e PKI: sustenta o mTLS que autentica cada emissor e protege a chave de tokenização de PII. |
| RFC-0220 | `../022-Policy/` | Contrato de decisão (`DecisionRequest`/`DecisionResponse`) usado pelo `QueryGateway` como PEP. |
| RFC-0200 / RFC-0201 | `../020-Communication/` | Contrato do barramento: subjects e streams JetStream usados pelos eventos do módulo. |

---

## 4. Relação com padrões externos

| Padrão | Uso | Grau de aderência |
|--------|-----|-------------------|
| **OpenTelemetry (OTLP)** | Protocolo de ingestão de métricas, traces e logs. | **Total** — o módulo não define protocolo próprio de emissão (`./API.md` §2). |
| **W3C Trace Context** | Correlação `traceparent`/`tracestate`. | Total, via RFC-0001 §5.6. |
| **OpenTelemetry Semantic Conventions** | Base dos atributos de recurso. | **Parcial**: RFC-0240 adota as convenções OTel e **acrescenta** os atributos `aios.*`; não as contradiz. |
| **Prometheus exposition / PromQL** | Formato de métrica e linguagem de consulta. | Total. |
| **CloudEvents 1.0** | Envelope dos eventos do módulo. | Total, via RFC-0001 §5.2. |
| **RFC 7807** | Envelope de erro das superfícies de consulta e administração. | Total, via RFC-0001 §5.4. |

> A aderência total ao OTLP é decisão de custo de integração: obrigar cada emissor a
> implementar uma API própria do AIOS transformaria a instrumentação em trabalho de
> integração para todos os 40 módulos.

---

## 5. Registro e evolução

- A numeração `RFC-0240`/`RFC-0241` segue a convenção de faixa por módulo já adotada
  por `../020-Communication/` (RFC-0200/0201), `../021-Security/` (RFC-0210/0211) e
  `../022-Policy/` (RFC-0220/0221).
- Novos valores de `signal_kind`, `retention_tier`, `reason` de descarte e
  `policyAction` são **aditivos** e registrados nas RFCs correspondentes; remoção ou
  renomeação exige nova versão da RFC e período de coexistência (RFC-0001 §5.7 e §9).
- Alteração incompatível da gramática de nomes (RFC-0240) exige **nova versão de
  descritor** para os sinais afetados — nunca reinterpretação retroativa, que faria
  séries históricas mudarem de significado.
- Nenhuma RFC deste módulo **PODE** contradizer a RFC-0001; alteração da baseline exige
  RFC que a obsolete, com período de coexistência.

*Fim de `RFC.md`.*
