---
Documento: ADR
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010; ADR-0240..ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: 002-ADR, _DESIGN_BRIEF.md §11
---

# 024-Observability — Índice de ADRs

Faixa reservada: **`ADR-0240`..`ADR-0249`** (regra `NNN × 10`). As ADRs são registradas
em `../002-ADR/` — este documento é o índice com o impacto de cada decisão sobre o
módulo.

---

## 1. ADRs globais herdadas

| ADR | Decisão | Impacto no 024-Observability | Status |
|-----|---------|------------------------------|--------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | O módulo é o subsistema de **instrumentação** do SO, não uma biblioteca de logging. | Accepted |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Telemetria atravessa os dois planos; a correlação por `trace_id` é o que os costura. | Accepted |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Eventos do módulo trafegam por JetStream; **a telemetria em si, não** (`./Events.md` §5). | Accepted |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL unificado | Schema `telemetry` guarda o **plano de controle**; séries, traces e logs vivem fora. | Accepted |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente | Contadores de cardinalidade e limites de ingestão. | Accepted |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | O módulo é **PEP** nas superfícies de consulta e administração — e **não** na ingestão. | Accepted |
| **[ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md)** | **Observabilidade e auditoria por construção** | **Decisão-mãe deste módulo.** Fixa OTel em todo caminho, correlação obrigatória e a separação entre telemetria e trilha imutável. Todo o 024 é a materialização da metade "observabilidade" desta ADR. | Accepted |

---

## 2. ADRs a propor por este módulo

| ADR | Título | Escopo da decisão | Documentos afetados | Status |
|-----|--------|-------------------|---------------------|--------|
| ADR-0240 | Domínio de erro `OBS` e registro do domínio de eventos `telemetry` | Reserva de `AIOS-OBS-0001..0099`; registro de subjects e streams `TELEMETRY_*` conforme RFC-0001 §8. | `API.md`, `Events.md` | Proposed |
| ADR-0241 | Registro obrigatório de sinal: telemetria como contrato declarado, não emergente | Todo sinal é declarado, versionado e admitido antes de ser armazenado; o desconhecido é quarentenado. É o que permite governar nome, unidade, labels e retenção de forma uniforme. | `StateMachine.md`, `Database.md`, `API.md` | Proposed |
| ADR-0242 | Orçamento de cardinalidade por tenant e proibição de labels de identidade | A decisão de escala dominante: o limite é **séries ativas**, não bytes; identidade nunca é label. | `Scalability.md`, `Metrics.md`, `Database.md` | Proposed |
| ADR-0243 | Fail-open da telemetria: descarte contabilizado em vez de backpressure no observado | A decisão de disponibilidade dominante: a observabilidade nunca pode derrubar o observado. Assimétrica em relação ao `../022-Policy/`, que é fail-closed. | `FailureRecovery.md`, `Security.md`, `Benchmark.md` | Proposed |
| ADR-0244 | *Tail-based sampling* com retenção obrigatória de erros e caudas | Reter 100% do que explica falha, amostrar o resto — em vez de uma fatia estatística cega que descartaria 90% dos erros. | `Architecture.md`, `Configuration.md`, `Benchmark.md` | Proposed |
| ADR-0245 | SLO como dado versionado e error budget como sinal de primeira classe | Objetivo, SLI e budget são artefatos com versão, ledger e histórico; o consumo de budget é o que decide o ritmo de mudança do sistema. | `Database.md`, `StateMachine.md`, `Events.md` | Proposed |
| ADR-0246 | Fronteira telemetria × auditoria (024 × 025): amostrável vs. imutável | Define o que cada subsistema prova e evita duas fontes da verdade com regras de retenção divergentes. | `Vision.md`, `Logging.md`, `Security.md` | Proposed |
| ADR-0247 | Dashboards, regras de alerta e runbooks como código versionado; alerta sem runbook é inválido | Torna painel, alerta e procedimento revisáveis e testáveis; impede o alerta que só produz fadiga. | `Database.md`, `Monitoring.md`, `Testing.md` | Proposed |
| ADR-0248 | Retenção em camadas com *downsampling* em vez de retenção única | Custo proporcional ao valor do dado ao longo do tempo; viabiliza 400 d de tendência sem 400 d de resolução nativa. | `Database.md`, `Configuration.md`, `Scalability.md` | Proposed |
| ADR-0249 | Redação de PII na borda de ingestão, não no armazenamento | Redigir depois significaria que o dado já esteve em claro em disco — e em backups, réplicas e índices. | `Security.md`, `ClassDiagrams.md`, `Logging.md` | Proposed |

---

## 3. Decisões que este módulo **não** toma

| Assunto | Onde a decisão vive |
|---------|---------------------|
| Quais métricas cada módulo emite e o que significam | Módulo emissor (`owner_module` do descritor) |
| Formato e retenção da trilha imutável | `../025-Audit/` |
| Autorização de consulta (modelo de papéis e regras) | `../022-Policy/` |
| Resposta operacional ao alerta (mitigação, escalonamento humano) | `../029-Operations/` |
| Decisão de rollback de release ou de política | `../028-Deployment/`, `../022-Policy/` |
| Orçamento financeiro e custo de inferência | `../026-Cost-Optimizer/` |
| Identidade dos coletores e chave de tokenização | `../021-Security/` |
| Topologia de nós e região de armazenamento | `../027-Cluster/`, `../028-Deployment/` |

O 024 **mede** o custo, mas não define orçamento; **detecta** a degradação, mas não
decide o rollback; **redige** PII, mas não é dono da chave. Manter essa disciplina é o
que evita que o módulo de observabilidade vire o depósito de toda responsabilidade
transversal do sistema.

---

## 4. Registro e evolução

- Novas ADRs deste módulo **DEVEM** usar a faixa `ADR-0240..ADR-0249` e ser
  registradas em `../002-ADR/README.md`.
- Esgotada a faixa, a extensão **DEVE** ser negociada com Arquitetura-Chefe — não se
  invade a faixa de outro módulo.
- ADRs não são editadas após `Accepted`: são **substituídas**
  (`Superseded by ADR-XXXX`), conforme `../002-ADR/README.md`.
- Uma decisão que contrarie a RFC-0001 ou o ADR-0010 **NÃO DEVE** ser tomada aqui:
  exige RFC que obsolete a baseline.

*Fim de `ADR.md`.*
