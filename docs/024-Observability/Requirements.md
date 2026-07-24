---
Documento: Requirements
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0240..ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Testing.md
---

# 024-Observability — Requisitos (índice e rastreabilidade)

## 1. Escopo deste documento

Consolida os requisitos do módulo e estabelece a **matriz de rastreabilidade**
Requisito → Caso de uso → Teste. Os requisitos estão em
`./FunctionalRequirements.md` (FR-001..FR-024) e
`./NonFunctionalRequirements.md` (NFR-001..NFR-017); ambos derivam de
`./_DESIGN_BRIEF.md` §7.

## 2. Stakeholders

| Stakeholder | Interesse | Requisitos mais sensíveis |
|-------------|-----------|---------------------------|
| Todos os módulos emissores | Instrumentar sem risco de a telemetria travá-los. | FR-001, FR-002, FR-009; NFR-002, NFR-006 |
| SRE / plantão (`../029-Operations/`) | Ser acordado só pelo que importa, com procedimento. | FR-011..FR-016; NFR-005, NFR-008, NFR-016 |
| Dono de serviço | Saber se está dentro do objetivo e quanto budget resta. | FR-010, FR-020; NFR-012 |
| Plataforma / arquitetura | Impedir crescimento sem limite da telemetria. | FR-003..FR-006, FR-019; NFR-009, NFR-010 |
| DPO / auditoria | Telemetria não pode virar repositório de PII. | FR-007, FR-023; NFR-014 |
| `../022-Policy/` | Autorizar consulta; receber sinal de degradação. | FR-017, FR-018; NFR-017 |
| `../028-Deployment/` | Gate objetivo de release por error budget. | FR-010, FR-024 |
| `../025-Audit/` | Fronteira clara: telemetria não é prova. | FR-018, FR-023 |

## 3. Índice de requisitos funcionais

| ID | Resumo | Prioridade |
|----|--------|-----------|
| FR-001 | Ingestão OTLP (gRPC/HTTP) sob mTLS. | Must |
| FR-002 | Sem backpressure: descarte contabilizado. | Must |
| FR-003 | Admitir só sinal registrado; quarentena sem derrubar o lote. | Must |
| FR-004 | Validar nome, tipo, unidade e labels. | Must |
| FR-005 | Orçamento de cardinalidade por tenant e serviço. | Must |
| FR-006 | Proibir label de identidade em métrica. | Must |
| FR-007 | Redação de PII na borda. | Must |
| FR-008 | *Tail sampling* com 100% de erros e caudas. | Must |
| FR-009 | Enriquecimento com atributos de recurso canônicos. | Must |
| FR-010 | SLI, SLO e ledger de error budget. | Must |
| FR-011 | Alerta *multi-window multi-burn-rate* com histerese. | Must |
| FR-012 | Ciclo de vida do alerta com deduplicação. | Must |
| FR-013 | Runbook obrigatório por regra. | Must |
| FR-014 | Silêncio com prazo, justificativa e autor. | Must |
| FR-015 | Silêncio suprime notificação, não observação. | Must |
| FR-016 | Ausência de dado → `Expired`, nunca `Resolved`. | Must |
| FR-017 | Consulta unificada com isolamento por tenant. | Must |
| FR-018 | PEP em consulta e administração. | Must |
| FR-019 | Retenção em camadas com *downsampling*. | Must |
| FR-020 | Dashboards e runbooks como código. | Should |
| FR-021 | Meta-observabilidade por caminho independente. | Must |
| FR-022 | Eventos via Outbox transacional. | Must |
| FR-023 | Direito ao esquecimento sobre telemetria. | Must |
| FR-024 | Anotação de rollout e política nos dashboards. | Should |

## 4. Índice de requisitos não-funcionais

| ID | Atributo | Meta |
|----|----------|------|
| NFR-001 | Desempenho (ingestão) | p99 ≤ 50 ms |
| NFR-002 | Overhead no observado | ≤ 3% CPU · ≤ 1 ms p99 |
| NFR-003 | Throughput | ≥ 2.000.000 dp/s · ≥ 200.000 spans/s por réplica |
| NFR-004 | Disponibilidade (ingestão) | ≥ 99,9% |
| NFR-005 | Disponibilidade (alerta) | ≥ 99,95% |
| NFR-006 | Perda de telemetria | ≤ 0,1%; 100% contabilizado |
| NFR-007 | Frescor | ≤ 10 s (p99) |
| NFR-008 | Tempo de detecção | ≤ 2 min |
| NFR-009 | Cardinalidade | ≤ 10⁶ séries/tenant; 0 labels de identidade |
| NFR-010 | Escalabilidade | 10⁶ agentes · 500 serviços |
| NFR-011 | Recuperação | RTO ≤ 15 min · RPO ≤ 5 min (controle) / 1 min (bruto) |
| NFR-012 | Retenção | 15 d / 90 d / 400 d · traces 7 d · logs 14 d |
| NFR-013 | Latência de consulta | p95 ≤ 2 s |
| NFR-014 | Privacidade | 0 PII em sinal `operational` |
| NFR-015 | Idempotência | 100% efeito único |
| NFR-016 | Cobertura de runbook | 100% válidos e revisados ≤ 180 d |
| NFR-017 | Isolamento | 0 consultas cross-tenant |

## 5. Matriz de rastreabilidade — Requisito → Caso de uso → Teste

| Requisito | Caso(s) de uso | Teste(s) | Documento de projeto |
|-----------|----------------|----------|----------------------|
| FR-001 | UC-001 | T-OBS-001 | `./API.md` §2 |
| FR-002 | UC-001 | T-OBS-002 | `./FailureRecovery.md` §2 |
| FR-003 | UC-002, UC-003 | T-OBS-003 | `./StateMachine.md` §5 |
| FR-004 | UC-002 | T-OBS-004 | `./API.md` §4 |
| FR-005 | UC-003 | T-OBS-005 | `./Database.md` §3 |
| FR-006 | UC-002, UC-003 | T-OBS-006 | `./Metrics.md` §6 |
| FR-007 | UC-004 | T-OBS-007 | `./Security.md` §4 |
| FR-008 | UC-005 | T-OBS-008 | `./Configuration.md` §3 |
| FR-009 | UC-001 | T-OBS-009 | `./ClassDiagrams.md` §2 |
| FR-010 | UC-006, UC-007 | T-OBS-010 | `./Database.md` §5 |
| FR-011 | UC-007, UC-008 | T-OBS-011 | `./StateMachine.md` §6 |
| FR-012 | UC-008..UC-011 | T-OBS-012 | `./StateMachine.md` §3 |
| FR-013 | UC-013 | T-OBS-013 | `./Database.md` §7 |
| FR-014 | UC-010 | T-OBS-014 | `./Database.md` §8 |
| FR-015 | UC-010, UC-011 | T-OBS-015 | `./StateMachine.md` §5 |
| FR-016 | UC-011 | T-OBS-016 | `./Monitoring.md` §5 |
| FR-017 | UC-012 | T-OBS-017 | `./API.md` §3 |
| FR-018 | UC-012, UC-013 | T-OBS-018 | `./Security.md` §2 |
| FR-019 | UC-014 | T-OBS-019 | `./Database.md` §11 |
| FR-020 | UC-013 | T-OBS-020 | `./Examples.md` §7 |
| FR-021 | UC-001 | T-OBS-021 | `./FailureRecovery.md` §6 |
| FR-022 | UC-008 | T-OBS-022 | `./Events.md` §5 |
| FR-023 | UC-015 | T-OBS-023 | `./Security.md` §6 |
| FR-024 | UC-013 | T-OBS-024 | `./Events.md` §4 |
| NFR-001 | UC-001 | T-OBS-101 | `./Benchmark.md` §3 |
| NFR-002 | UC-001 | T-OBS-102 | `./Benchmark.md` §4 |
| NFR-003 | UC-001 | T-OBS-103 | `./Benchmark.md` §3 |
| NFR-004 | todos | T-OBS-104 | `./Monitoring.md` §3 |
| NFR-005 | UC-008 | T-OBS-105 | `./Monitoring.md` §3 |
| NFR-006 | UC-001 | T-OBS-106 | `./FailureRecovery.md` §2 |
| NFR-007 | UC-012 | T-OBS-107 | `./Benchmark.md` §3 |
| NFR-008 | UC-007, UC-008 | T-OBS-108 | `./Monitoring.md` §4 |
| NFR-009 | UC-003 | T-OBS-109 | `./Scalability.md` §5 |
| NFR-010 | UC-001 | T-OBS-110 | `./Scalability.md` §5 |
| NFR-011 | UC-014 | T-OBS-111 | `./FailureRecovery.md` §5 |
| NFR-012 | UC-014 | T-OBS-112 | `./Database.md` §11 |
| NFR-013 | UC-012 | T-OBS-113 | `./Benchmark.md` §3 |
| NFR-014 | UC-004, UC-015 | T-OBS-114 | `./Security.md` §4 |
| NFR-015 | UC-006, UC-013 | T-OBS-115 | `./API.md` §7 |
| NFR-016 | UC-013 | T-OBS-116 | `./Monitoring.md` §5 |
| NFR-017 | UC-012 | T-OBS-117 | `./Database.md` §12 |

## 6. Cobertura reversa — Caso de uso → Requisitos

| UC | Nome | Requisitos exercitados |
|----|------|------------------------|
| UC-001 | Ingerir telemetria por OTLP | FR-001, FR-002, FR-009, FR-021; NFR-001..NFR-003, NFR-006 |
| UC-002 | Registrar um novo sinal | FR-003, FR-004, FR-006; NFR-015 |
| UC-003 | Quarentenar sinal por cardinalidade | FR-003, FR-005, FR-006; NFR-009 |
| UC-004 | Redigir PII na borda | FR-007; NFR-014 |
| UC-005 | Amostrar traces (*tail-based*) | FR-008 |
| UC-006 | Definir e publicar um SLO | FR-010; NFR-015 |
| UC-007 | Avaliar error budget e sinalizar exaustão | FR-010, FR-011; NFR-008 |
| UC-008 | Disparar, notificar e escalonar alerta | FR-011, FR-012, FR-022; NFR-005, NFR-008 |
| UC-009 | Reconhecer alerta | FR-012 |
| UC-010 | Silenciar alerta | FR-014, FR-015 |
| UC-011 | Resolver alerta e tratar ausência de dado | FR-012, FR-015, FR-016 |
| UC-012 | Consultar telemetria | FR-017, FR-018; NFR-007, NFR-013, NFR-017 |
| UC-013 | Publicar dashboard e runbook | FR-013, FR-018, FR-020, FR-024; NFR-016 |
| UC-014 | Aplicar retenção e *downsampling* | FR-019; NFR-011, NFR-012 |
| UC-015 | Direito ao esquecimento sobre telemetria | FR-023; NFR-014 |

## 7. Glossário local

Este módulo **não** define termos próprios. Os artefatos abaixo são **nomes de
estruturas deste módulo**, descritos (não redefinidos) em `./_DESIGN_BRIEF.md`:

| Termo | Onde é especificado |
|-------|---------------------|
| `SignalDescriptor` | `./_DESIGN_BRIEF.md` §3.1, `./StateMachine.md` §5 |
| Orçamento de cardinalidade | `./_DESIGN_BRIEF.md` §3.2, `./Scalability.md` |
| SLO / error budget / *burn rate* | `./_DESIGN_BRIEF.md` §3.3–§3.4, §4.5 |
| `AlertInstance` e `fingerprint` | `./_DESIGN_BRIEF.md` §3.6, `./StateMachine.md` §3 |
| Camada de retenção (`raw`/`short`/`medium`/`long`) | `./_DESIGN_BRIEF.md` §3.9 |

> Termos consagrados de observabilidade (SLI, SLO, error budget, *span*, *trace*)
> **DEVERIAM** ser promovidos ao `../040-Glossary/Glossary.md` por PR próprio, caso
> passem a ser usados fora deste módulo — nunca definidos localmente em duplicidade.

## 8. Referências

- `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- `./UseCases.md` · `./Testing.md`
- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim de `Requirements.md`.*
