---
Documento: Requirements
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0008, ADR-0010, ADR-0250..ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Testing.md
---

# 025-Audit — Requisitos (índice e rastreabilidade)

## 1. Escopo deste documento

Consolida os requisitos do módulo e estabelece a **matriz de rastreabilidade**
Requisito → Caso de uso → Teste. Os requisitos estão em
`./FunctionalRequirements.md` (FR-001..FR-023) e
`./NonFunctionalRequirements.md` (NFR-001..NFR-017); ambos derivam de
`./_DESIGN_BRIEF.md` §7.

## 2. Stakeholders

| Stakeholder | Interesse | Requisitos mais sensíveis |
|-------------|-----------|---------------------------|
| Auditor externo | Verificar sem confiar no operador do AIOS. | FR-005, FR-006, FR-017; NFR-007, NFR-013 |
| DPO / jurídico | Retenção legal, *hold* e direito ao esquecimento. | FR-008..FR-013; NFR-010, NFR-011 |
| Segurança / CISO | Detectar tentativa de adulteração da evidência. | FR-004, FR-006, FR-018; NFR-007 |
| Módulos produtores | Registrar o fato sem risco de perdê-lo. | FR-001, FR-002; NFR-001, NFR-006 |
| Investigador de incidente (`../029-Operations/`) | Reconstruir a cronologia. | FR-015, FR-016; NFR-012 |
| `../005-Database/` | Saber quando suspender expurgo. | FR-009, FR-021 |
| `../022-Policy/` | Ter suas decisões registradas de forma imutável. | FR-001, FR-003 |
| `../024-Observability/` | Fronteira clara: telemetria não é prova. | FR-023; NFR-016 |

## 3. Índice de requisitos funcionais

| ID | Resumo | Prioridade |
|----|--------|-----------|
| FR-001 | Ingestão por evento e API, idempotente. | Must |
| FR-002 | Confirmar só após durabilidade (`AIOS-AUD-0005`). | Must |
| FR-003 | Envelope canônico de auditoria. | Must |
| FR-004 | Encadeamento por hash. | Must |
| FR-005 | Selo periódico com Merkle assinada. | Must |
| FR-006 | Verificação de integridade contínua e completa. | Must |
| FR-007 | Detecção de lacuna e classe silenciosa. | Must |
| FR-008 | Retenção legal por classe com base legal. | Must |
| FR-009 | *Legal hold* com aprovação dupla e `case_ref`. | Must |
| FR-010 | *Hold* suspende retenção e apagamento. | Must |
| FR-011 | Apagamento criptográfico por destruição de chave. | Must |
| FR-012 | Cadeia e metadados preservados após apagamento. | Must |
| FR-013 | Comprovante verificável, registrado na trilha. | Must |
| FR-014 | Arquivamento WORM com *object lock*. | Must |
| FR-015 | Consulta com PEP e escopo por tenant. | Must |
| FR-016 | Consulta à trilha é fato auditável. | Must |
| FR-017 | Pacote probatório verificável offline. | Must |
| FR-018 | Detecção de anomalias de auditoria. | Must |
| FR-019 | Append-only: nenhuma alteração de registro. | Must |
| FR-020 | PEP em consulta e governança. | Must |
| FR-021 | Eventos via Outbox transacional. | Must |
| FR-022 | Registro de classes auditáveis. | Should |
| FR-023 | Nenhum conteúdo auditado fora da trilha. | Must |

## 4. Índice de requisitos não-funcionais

| ID | Atributo | Meta |
|----|----------|------|
| NFR-001 | Desempenho (escrita durável) | p99 ≤ 20 ms |
| NFR-002 | Throughput | ≥ 50.000 registros/s por réplica |
| NFR-003 | Disponibilidade | ≥ 99,95% |
| NFR-004 | Latência de selagem | 100% em ≤ 60 s |
| NFR-005 | Completude | 100%; lacuna em ≤ 5 min |
| NFR-006 | Durabilidade | **RPO = 0** · RTO ≤ 15 min |
| NFR-007 | Integridade | 100% verificável; detecção ≤ 1 ciclo |
| NFR-008 | Verificação completa | 10⁹ registros em ≤ 4 h |
| NFR-009 | Escalabilidade | 10⁹ registros/tenant · 10⁶ produtores |
| NFR-010 | Retenção | 100% de aderência; 0 expurgo sob *hold* |
| NFR-011 | Apagamento criptográfico | ≤ 24 h; 0 payload legível |
| NFR-012 | Latência de consulta | p95 ≤ 2 s |
| NFR-013 | Exportação | 10⁶ registros em ≤ 30 min |
| NFR-014 | Idempotência | 100% efeito único |
| NFR-015 | Isolamento | 0 cross-tenant |
| NFR-016 | Confidencialidade | 0 conteúdo fora da trilha |
| NFR-017 | Observabilidade | 100% com trace |

## 5. Matriz de rastreabilidade — Requisito → Caso de uso → Teste

| Requisito | Caso(s) de uso | Teste(s) | Documento de projeto |
|-----------|----------------|----------|----------------------|
| FR-001 | UC-001, UC-002 | T-AUD-001 | `./API.md` §2 |
| FR-002 | UC-002 | T-AUD-002 | `./FailureRecovery.md` §2 |
| FR-003 | UC-001 | T-AUD-003 | `./Database.md` §2 |
| FR-004 | UC-001 | T-AUD-004 | `./StateMachine.md` §5 |
| FR-005 | UC-003 | T-AUD-005 | `./Database.md` §3 |
| FR-006 | UC-004, UC-006 | T-AUD-006 | `./StateMachine.md` §7 |
| FR-007 | UC-005 | T-AUD-007 | `./Monitoring.md` §4 |
| FR-008 | UC-007, UC-011 | T-AUD-008 | `./Database.md` §4 |
| FR-009 | UC-008, UC-009 | T-AUD-009 | `./Security.md` §2 |
| FR-010 | UC-008 | T-AUD-010 | `./StateMachine.md` §3 |
| FR-011 | UC-010 | T-AUD-011 | `./Security.md` §6 |
| FR-012 | UC-010, UC-011 | T-AUD-012 | `./StateMachine.md` §5 |
| FR-013 | UC-010 | T-AUD-013 | `./Database.md` §7 |
| FR-014 | UC-015 | T-AUD-014 | `./Deployment.md` §5 |
| FR-015 | UC-012 | T-AUD-015 | `./API.md` §3 |
| FR-016 | UC-012 | T-AUD-016 | `./Logging.md` §4 |
| FR-017 | UC-014 | T-AUD-017 | `./API.md` §4 |
| FR-018 | UC-006 | T-AUD-018 | `./Monitoring.md` §5 |
| FR-019 | UC-001 | T-AUD-019 | `./Database.md` §2 |
| FR-020 | UC-012, UC-014 | T-AUD-020 | `./Security.md` §2 |
| FR-021 | UC-008 | T-AUD-021 | `./Events.md` §5 |
| FR-022 | UC-007 | T-AUD-022 | `./Database.md` §4 |
| FR-023 | UC-012 | T-AUD-023 | `./Security.md` §5 |
| NFR-001 | UC-002 | T-AUD-101 | `./Benchmark.md` §3 |
| NFR-002 | UC-001 | T-AUD-102 | `./Benchmark.md` §3 |
| NFR-003 | todos | T-AUD-103 | `./Monitoring.md` §3 |
| NFR-004 | UC-003 | T-AUD-104 | `./Monitoring.md` §4 |
| NFR-005 | UC-005 | T-AUD-105 | `./Monitoring.md` §4 |
| NFR-006 | UC-002 | T-AUD-106 | `./FailureRecovery.md` §7 |
| NFR-007 | UC-004, UC-006 | T-AUD-107 | `./Benchmark.md` §5 |
| NFR-008 | UC-004 | T-AUD-108 | `./Benchmark.md` §3 |
| NFR-009 | UC-001 | T-AUD-109 | `./Scalability.md` §5 |
| NFR-010 | UC-008, UC-011 | T-AUD-110 | `./Database.md` §11 |
| NFR-011 | UC-010 | T-AUD-111 | `./Security.md` §6 |
| NFR-012 | UC-012 | T-AUD-112 | `./Benchmark.md` §3 |
| NFR-013 | UC-014 | T-AUD-113 | `./Benchmark.md` §3 |
| NFR-014 | UC-002 | T-AUD-114 | `./API.md` §7 |
| NFR-015 | UC-012, UC-014 | T-AUD-115 | `./Database.md` §12 |
| NFR-016 | UC-012 | T-AUD-116 | `./Logging.md` §6 |
| NFR-017 | todos | T-AUD-117 | `./Monitoring.md` §2 |

## 6. Cobertura reversa — Caso de uso → Requisitos

| UC | Nome | Requisitos exercitados |
|----|------|------------------------|
| UC-001 | Registrar um fato por evento | FR-001, FR-003, FR-004, FR-019; NFR-002, NFR-009 |
| UC-002 | Registrar um fato por API direta | FR-001, FR-002; NFR-001, NFR-006, NFR-014 |
| UC-003 | Selar um intervalo | FR-005; NFR-004 |
| UC-004 | Verificar integridade | FR-006; NFR-007, NFR-008 |
| UC-005 | Detectar lacuna ou classe silenciosa | FR-007; NFR-005 |
| UC-006 | Detectar quebra de cadeia | FR-006, FR-018; NFR-007 |
| UC-007 | Registrar classe auditável | FR-008, FR-022 |
| UC-008 | Aplicar *legal hold* | FR-009, FR-010, FR-021; NFR-010 |
| UC-009 | Liberar *legal hold* | FR-009 |
| UC-010 | Executar apagamento criptográfico | FR-011, FR-012, FR-013; NFR-011 |
| UC-011 | Expurgar payload por fim de retenção | FR-008, FR-012; NFR-010 |
| UC-012 | Consultar a trilha | FR-015, FR-016, FR-020, FR-023; NFR-012, NFR-015, NFR-016 |
| UC-013 | Obter prova de inclusão | FR-005, FR-006 |
| UC-014 | Exportar pacote probatório | FR-017, FR-020; NFR-013, NFR-015 |
| UC-015 | Arquivar em WORM | FR-014; NFR-007 |

## 7. Glossário local

Este módulo **não** define termos próprios. **Event Sourcing** é reutilizado de
`../040-Glossary/Glossary.md` sem redefinição. Os artefatos abaixo são **nomes de
estruturas deste módulo**, descritos (não redefinidos) em `./_DESIGN_BRIEF.md`:

| Termo | Onde é especificado |
|-------|---------------------|
| `AuditRecord` e envelope canônico | `./_DESIGN_BRIEF.md` §3.1, `./Database.md` §2 |
| Cadeia de hash (`prev_hash`/`record_hash`) | `./_DESIGN_BRIEF.md` §3.1, `./StateMachine.md` §5 |
| Selo e raiz de Merkle | `./_DESIGN_BRIEF.md` §3.2, `./Database.md` §3 |
| `record_class` (classe auditável) | `./_DESIGN_BRIEF.md` §3.3 |
| *Legal hold* | `./_DESIGN_BRIEF.md` §3.4, §4.5 |
| Apagamento criptográfico (*crypto-shredding*) | `./_DESIGN_BRIEF.md` §3.5, §12.3 |
| Pacote probatório | `./_DESIGN_BRIEF.md` §5.2, `./API.md` §4 |

> *Legal hold*, *crypto-shredding* e *pacote probatório* **DEVERIAM** ser promovidos ao
> `../040-Glossary/Glossary.md` por PR próprio, caso passem a ser usados fora deste
> módulo — nunca definidos localmente em duplicidade.

## 8. Referências

- `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- `./UseCases.md` · `./Testing.md`
- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim de `Requirements.md`.*
