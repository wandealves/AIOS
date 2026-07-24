---
Documento: Testing
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability + QA
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0243, ADR-0244, ADR-0247, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Requirements.md
---

# 024-Observability — Estratégia de Testes

## 1. O que é difícil de testar aqui

Este módulo tem uma característica incômoda: **suas falhas são silenciosas**. Um
pipeline que descarta 5% da telemetria não devolve erro a ninguém; um alerta que nunca
dispara não gera reclamação. Por isso a estratégia enfatiza três verificações que, em
outros módulos, seriam secundárias:

1. **Testes de não-interferência** — provar que o observador não afeta o observado.
2. **Testes de perda contabilizada** — provar que todo descarte aparece em métrica.
3. **Probes sintéticos** — provar que a capacidade de alertar existe, testando-a de
   fora.

## 2. Pirâmide e cobertura-alvo

| Nível | Escopo | Cobertura-alvo | Ferramenta |
|-------|--------|----------------|-----------|
| Unitário | `SignalAdmission`, `CardinalityGuard`, `PiiRedactor`, `TraceSampler`, `SloEngine`, `AlertEvaluator` | ≥ 90% de linhas; **100%** dos ramos da FSM de alerta | xUnit |
| Integração | Pipeline + Prometheus/Tempo/Seq + PostgreSQL + Redis | ≥ 80% | Testcontainers |
| Contrato | OTLP (emissores), eventos, API de consulta/administração | 100% das operações de `./API.md` | Pact + JSON Schema |
| E2E | Emissão → admissão → armazenamento → SLO → alerta → notificação | 100% dos UC-001..UC-015 | Compose + cenários |
| Carga | NFR-001..NFR-003, NFR-013 | — | k6 + gerador OTLP |
| Caos | `./FailureRecovery.md` §8 | 8 experimentos | Chaos harness |
| **Sintético contínuo** | NFR-005, NFR-007 | probes em produção | Probe dedicado |

## 3. Testes por fronteira testável

As interfaces de `./ClassDiagrams.md` §2 são os pontos de injeção:

| Fronteira | Duplo de teste | O que se verifica |
|-----------|----------------|-------------------|
| `ITelemetryIngestGateway` | Emissor sintético com medição de latência própria | Não-interferência: latência do emissor inalterada sob saturação |
| `ISignalRegistry` | PostgreSQL real (Testcontainers) | Validação de convenção; imutabilidade após `Registered` |
| `ICardinalityGuard` | Redis real + gerador de séries | Quarentena no limiar correto; isolamento por sinal |
| `IPiiRedactor` | Corpus com PII conhecida | 0 vazamento; tokenização estável |
| `ITraceSampler` | Traces sintéticos com e sem erro | 100% de retenção de erro e cauda |
| `IAlertEvaluator` | Relógio controlado | Histerese, escalonamento, `nodata`, anti-*flapping* |
| `IQueryGateway` | PDP simulado | Fail-closed; isolamento de tenant |

## 4. Casos de teste — requisitos funcionais

| ID | Requisito | Cenário | Resultado esperado |
|----|-----------|---------|--------------------|
| T-OBS-001 | FR-001 | Lote OTLP de cada tipo, mTLS válido | Aceito; visível na consulta em ≤ 10 s |
| T-OBS-002 | FR-002 | Saturar o gateway com 5× a carga nominal | **Latência do emissor inalterada**; `dropped_total` reflete a perda; nenhum `429` |
| T-OBS-003 | FR-003 | Lote com sinal registrado + desconhecido | Registrado persiste; desconhecido quarentenado; lote **não** rejeitado |
| T-OBS-004 | FR-004 | Registrar `kernel_latency` (fora da convenção) | `AIOS-OBS-0004`; estado `Rejected` |
| T-OBS-005 | FR-005 | Gerar séries até 101% do orçamento | Quarentena no limiar; `cardinality.exceeded` emitido; demais sinais intactos |
| T-OBS-006 | FR-006 | Registrar métrica com label `agent_urn` | `AIOS-OBS-0004`; em runtime, label descartado e contabilizado |
| T-OBS-007 | FR-007 | Log com `user.email` em sinal `operational` | Valor ausente do armazenamento; `pii_redacted_total` e `pii_undeclared_total` incrementados |
| T-OBS-008 | FR-008 | 1.000 traces: 50 com erro, 50 lentos, 900 normais | 100% dos 100 primeiros retidos; ~10% dos demais |
| T-OBS-009 | FR-009 | Lote sem `service.version` | Atributos canônicos injetados; 100% dos sinais enriquecidos |
| T-OBS-010 | FR-010 | SLO com SLI conhecido | Ledger reproduz o SLI; `budget_consumed` correto |
| T-OBS-011 | FR-011 | Degradação abrupta (queima rápida) | Alerta em ≤ 2 min; oscilação abaixo do limiar **não** dispara |
| T-OBS-012 | FR-012 | Mesma condição em duas avaliações | Uma única instância aberta por `fingerprint` |
| T-OBS-013 | FR-013 | Publicar regra sem `runbook_urn` | `AIOS-OBS-0008`; FK do banco também rejeita |
| T-OBS-014 | FR-014 | Silêncio sem `expires_at`; silêncio de 200 h | `AIOS-OBS-0013` em ambos |
| T-OBS-015 | FR-015 | Silenciar alerta e depois resolver a condição | Instância vai a `Resolved` normalmente; **avaliação nunca parou** |
| T-OBS-016 | FR-016 | Remover alvo de coleta sem `node.decommissioned` | `Expired` + `alert.nodata`; **0** transições a `Resolved` |
| T-OBS-017 | FR-017 | Consulta com `X-AIOS-Tenant` de outro tenant | `AIOS-OBS-0003`; RLS bloqueia mesmo com bypass de aplicação |
| T-OBS-018 | FR-018 | Consulta sem capability | Negada pelo PDP; decisão auditada |
| T-OBS-019 | FR-019 | Avançar o relógio 16 dias | Camada raw expurgada; agregada de 5 min disponível |
| T-OBS-020 | FR-020 | Dashboard referenciando sinal quarentenado | `AIOS-OBS-0009` |
| T-OBS-021 | FR-021 | Saturar o pipeline e consultar `SelfTelemetry` | Responde normalmente (caminho fora de banda) |
| T-OBS-022 | FR-022 | Crash entre commit e publicação | Evento publicado após recuperação; nenhum perdido |
| T-OBS-023 | FR-023 | `security.rtbf.requested` para um titular | Ocorrências pseudonimizadas nas três lojas; comprovante em `../025-Audit/` |
| T-OBS-024 | FR-024 | `deployment.rollout.started` | Anotação visível no dashboard em ≤ 30 s |

## 5. Casos de teste — requisitos não-funcionais

| ID | NFR | Método | Critério |
|----|-----|--------|----------|
| T-OBS-101 | NFR-001 | Carga sustentada | p99 de aceitação ≤ 50 ms |
| T-OBS-102 | NFR-002 | Serviço de referência com e sem instrumentação | ≤ 3% CPU e ≤ 1 ms no p99 |
| T-OBS-103 | NFR-003 | Carga por réplica isolada | ≥ 2.000.000 dp/s; ≥ 200.000 spans/s |
| T-OBS-104 | NFR-004 | Injeção de falhas + probe | ≥ 99,9% na janela |
| T-OBS-105 | NFR-005 | **Probe sintético de alerta** (1×/min) | ≥ 99,95%; alerta artificial notifica |
| T-OBS-106 | NFR-006 | Operação normal e sob saturação | ≤ 0,1% normal; 100% do descarte contabilizado |
| T-OBS-107 | NFR-007 | Probe fim a fim | Sinal visível em ≤ 10 s (p99) |
| T-OBS-108 | NFR-008 | Degradação controlada | Alerta em ≤ 2 min |
| T-OBS-109 | NFR-009 | Gerador de cardinalidade | Teto respeitado; **0** labels de identidade em produção |
| T-OBS-110 | NFR-010 | 500 emissores sintéticos | Custo cresce sub-linearmente |
| T-OBS-111 | NFR-011 | DR drill | RTO ≤ 15 min; RPO ≤ 5 min (controle) / 1 min (bruto) |
| T-OBS-112 | NFR-012 | Avanço de relógio por camada | Aderência de 100% às janelas de retenção |
| T-OBS-113 | NFR-013 | Consulta de painel típico | p95 ≤ 2 s; consultas fora do orçamento abortadas |
| T-OBS-114 | NFR-014 | Varredura do armazenamento | **0** PII em sinal `operational` |
| T-OBS-115 | NFR-015 | Replay com mesma `Idempotency-Key` | Efeito único; mesma resposta |
| T-OBS-116 | NFR-016 | Runbook com `last_reviewed_at` de 200 d | `runbook_stale_total` > 0; alerta dispara |
| T-OBS-117 | NFR-017 | Consulta cross-tenant por 3 caminhos (REST, gRPC, direto) | Bloqueada em todos |

## 6. Testes de contrato

| Contrato | Contraparte | Verificação |
|----------|-------------|-------------|
| OTLP + convenções semânticas (RFC-0240) | Todos os módulos emissores | Nome, tipo, unidade, labels e atributos de recurso conformes |
| `telemetry.alert.*` | `../029-Operations/` | Schema + presença obrigatória do campo `runbook` |
| `telemetry.budget.exhausted` | `../028-Deployment/`, `../022-Policy/` | Schema + semântica de `policyAction` |
| `telemetry.cardinality.exceeded` | Módulos donos de sinal | Presença de `offendingLabels` (acionável) |
| Consulta (PromQL/TraceQL/logs) | `../032-WebConsole/`, `../030-CLI/` | Contrato de resposta e paginação |
| PEP de consulta | `../022-Policy/` | `DecisionRequest` conforme RFC-0220 |
| RTBF | `../021-Security/`, `../025-Audit/` | Comprovante correlacionado |

## 7. Fixtures e ambientes

| Fixture | Conteúdo |
|---------|----------|
| `signals-baseline` | 50 descritores registrados (um por módulo emissor típico) |
| `traffic-nominal` | 500k dp/s, 50k spans/s, 20k logs/s por 10 min |
| `traffic-burst` | 5× nominal por 2 min — teste de não-interferência |
| `cardinality-bomb` | Sinal com label de 10⁶ valores distintos |
| `pii-corpus` | Logs e spans com PII conhecida e rotulada |
| `traces-mixed` | 1.000 traces: 5% erro, 5% lento, 90% normal |
| `slo-degrading` | Série sintética que consome budget a taxas conhecidas |

| Ambiente | Uso | Diferenças de produção |
|----------|-----|------------------------|
| Local (Compose) | Unidade/integração | `registration_required=false`, `runbook_required=false`, `head_sample_ratio=1.0` |
| CI efêmero | Todos os níveis exceto carga longa | Igual a produção nas chaves de governança |
| Homologação | E2E, carga, caos, probes | **Idêntico** a produção (`./Configuration.md` §11) |

> `obs.pii.redaction_enabled = true` em **todos** os ambientes, inclusive os de teste:
> o `pii-corpus` existe para verificar a redação, não para exercitar o caminho sem ela.

## 8. Gates de qualidade (CI)

Um PR **NÃO DEVE** ser mesclado se qualquer item falhar:

- [ ] Unitários ≥ 90%; **100%** dos ramos da FSM de alerta.
- [ ] Nenhum teste de contrato quebrado (Pact).
- [ ] `T-OBS-002` verde — **não-interferência**: o teste que prova que o observador não
      afeta o observado.
- [ ] `T-OBS-006`, `T-OBS-109` verdes — nenhum label de identidade.
- [ ] `T-OBS-007`, `T-OBS-114` verdes — nenhuma PII persistida.
- [ ] `T-OBS-016` verde — ausência de dado nunca vira `Resolved`.
- [ ] `T-OBS-021` verde — meta-observabilidade independente.
- [ ] Benchmark sem regressão > 10% vs. baseline (`./Benchmark.md`).
- [ ] Migrações validadas (CV-11/CV-12) e na ordem correta (`runbook` antes de `alert_rule`).
- [ ] Todas as métricas novas do módulo **registradas** como `SignalDescriptor`.
- [ ] Nenhum `TODO` nos 26 documentos do módulo.

## 9. Probes sintéticos em produção

| Probe | Frequência | O que prova |
|-------|-----------|-------------|
| **Alerta fim a fim** | 1×/min | Condição artificial dispara, notifica e resolve — NFR-005 |
| **Frescor** | 1×/min | Sinal emitido aparece na consulta em ≤ 10 s — NFR-007 |
| **Consulta** | 5×/min | Painel de referência responde em ≤ 2 s — NFR-013 |
| **Redação** | 1×/h | Sinal com PII sintética é redigido — NFR-014 |

O probe de **alerta** é o mais importante do módulo: é a única forma de verificar uma
capacidade cuja falha é, por definição, silenciosa. Um sistema de alerta quebrado não
alerta que está quebrado — e sem esse probe, a descoberta viria pelo primeiro incidente
não detectado.

## 10. Referências

- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Rastreabilidade: `./Requirements.md` §5 · Casos de uso: `./UseCases.md`
- Fronteiras testáveis: `./ClassDiagrams.md` §2 · Caos: `./FailureRecovery.md` §8
- Desempenho: `./Benchmark.md`

*Fim de `Testing.md`.*
