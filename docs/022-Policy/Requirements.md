---
Documento: Requirements
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0010, ADR-0220..ADR-0229
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Testing.md
---

# 022-Policy — Requisitos (índice e rastreabilidade)

## 1. Escopo deste documento

Consolida os requisitos do módulo e estabelece a **matriz de rastreabilidade**
Requisito → Caso de uso → Teste. Os requisitos em si estão em
`./FunctionalRequirements.md` (FR-001..FR-023) e
`./NonFunctionalRequirements.md` (NFR-001..NFR-016); ambos derivam de
`./_DESIGN_BRIEF.md` §7.

## 2. Stakeholders

| Stakeholder | Interesse | Requisitos mais sensíveis |
|-------------|-----------|---------------------------|
| Módulos com PEP (`004`, `006`, `007`, `009`, `010`, `011`, `015`, `017`, `020`, `021`) | Decidir rápido e sem acoplamento; invalidação confiável de cache. | FR-001, FR-004, FR-005, FR-012, NFR-001, NFR-004 |
| Engenharia de segurança do tenant | Escrever, provar e reverter política. | FR-006..FR-010, NFR-011 |
| SRE / plantão (`../029-Operations/`) | Diagnosticar negações e degradações. | FR-003, FR-013, FR-016, NFR-007, NFR-010 |
| Auditoria / DPO | Reprodutibilidade, explicabilidade e minimização. | FR-015, FR-016, FR-023, NFR-009, NFR-014 |
| Arquitetura-chefe | Consistência com ADR-0008 e RFC-0001 §5.8. | FR-001, FR-018, FR-019, NFR-008 |
| `../025-Audit/` | Receber o fato de decisão íntegro. | FR-015, FR-021, NFR-014 |
| `../026-Cost-Optimizer/` | Ter o orçamento respeitado como atributo. | FR-013 |

## 3. Índice de requisitos funcionais

| ID | Resumo | Prioridade |
|----|--------|-----------|
| FR-001 | Avaliar decisão sob *default deny*. | Must |
| FR-002 | RBAC + ABAC com precedência canônica. | Must |
| FR-003 | `reason_code`, `matched_rules[]`, `bundle_version` em toda decisão. | Must |
| FR-004 | Obrigações na resposta. | Must |
| FR-005 | TTL de decisão (deny ≤ allow). | Must |
| FR-006 | Compilar, validar e analisar conflitos. | Must |
| FR-007 | Gate de testes golden antes de publicar. | Must |
| FR-008 | Simulação com diferencial. | Must |
| FR-009 | Assinatura + aprovação dupla na publicação. | Must |
| FR-010 | Rollback dentro da janela de retenção. | Must |
| FR-011 | Propagação do bundle ativo. | Must |
| FR-012 | Invalidação granular por `policy.decision.updated`. | Must |
| FR-013 | Resolução de atributos com timeout e criticidade. | Must |
| FR-014 | Exceções com prazo e aprovador distinto. | Must |
| FR-015 | Journal: 100% dos `deny`, amostra dos `allow`. | Must |
| FR-016 | Explicação reprodutível de decisão. | Must |
| FR-017 | Decisão por NATS request-reply. | Must |
| FR-018 | Autogoverno pelo bundle-raiz. | Must |
| FR-019 | Negar tudo sem bundle ativo. | Must |
| FR-020 | Permissões efetivas do sujeito. | Should |
| FR-021 | Eventos via Outbox transacional. | Must |
| FR-022 | Limites de complexidade do bundle. | Should |
| FR-023 | Nenhum atributo `sensitive` exposto. | Must |

## 4. Índice de requisitos não-funcionais

| ID | Atributo | Meta |
|----|----------|------|
| NFR-001 | Desempenho | p99 ≤ 5 ms (cache) / ≤ 15 ms (atributo remoto) |
| NFR-002 | Throughput | ≥ 50.000 decisões/s por réplica |
| NFR-003 | Disponibilidade | ≥ 99,99% |
| NFR-004 | Propagação | bundle ≤ 10 s; cache do PEP ≤ 5 s |
| NFR-005 | Escalabilidade | 10⁶ agentes · 10⁴ regras · 10⁵ vínculos |
| NFR-006 | Recuperação | RTO ≤ 15 min · RPO ≤ 5 min |
| NFR-007 | Explicabilidade | 100% dos `deny` com motivo |
| NFR-008 | *Default deny* | 0 `allow` sem regra casada |
| NFR-009 | Determinismo | 100% reproduzível |
| NFR-010 | Orçamento de avaliação | 100% ≤ 50 ms ou `deny` |
| NFR-011 | Qualidade da política | cobertura ≥ 90% |
| NFR-012 | Simulação | 10⁶ requisições ≤ 10 min |
| NFR-013 | Idempotência | 100% efeito único |
| NFR-014 | Observabilidade | 100% com trace e auditoria |
| NFR-015 | Integridade | 100% dos bundles verificados |
| NFR-016 | Isolamento | 0 decisões cross-tenant |

## 5. Matriz de rastreabilidade — Requisito → Caso de uso → Teste

| Requisito | Caso(s) de uso | Teste(s) | Documento de projeto |
|-----------|----------------|----------|----------------------|
| FR-001 | UC-001 | T-POL-001 | `./API.md`, `./SequenceDiagrams.md` §2 |
| FR-002 | UC-001 | T-POL-002 | `./StateMachine.md` §6 |
| FR-003 | UC-001, UC-010 | T-POL-003 | `./API.md` §3 |
| FR-004 | UC-001 | T-POL-004 | `./API.md` §3.3 |
| FR-005 | UC-001 | T-POL-005 | `./Configuration.md` §2 |
| FR-006 | UC-003 | T-POL-006 | `./ClassDiagrams.md` §3 |
| FR-007 | UC-004 | T-POL-007 | `./Testing.md` §4 |
| FR-008 | UC-005 | T-POL-008 | `./Database.md` §3.9 |
| FR-009 | UC-006 | T-POL-009 | `./Security.md` §2 |
| FR-010 | UC-007 | T-POL-010 | `./StateMachine.md` §4 |
| FR-011 | UC-014 | T-POL-011 | `./Events.md` §3 |
| FR-012 | UC-008, UC-009 | T-POL-012 | `./Events.md` §3 |
| FR-013 | UC-011 | T-POL-013 | `./FailureRecovery.md` §3 |
| FR-014 | UC-008 | T-POL-014 | `./Database.md` §3.6 |
| FR-015 | UC-001, UC-010 | T-POL-015 | `./Logging.md` §4 |
| FR-016 | UC-010 | T-POL-016 | `./API.md` §3.4 |
| FR-017 | UC-001 | T-POL-017 | `./API.md` §4 |
| FR-018 | UC-015 | T-POL-018 | `./Security.md` §2.3 |
| FR-019 | UC-001 | T-POL-019 | `./StateMachine.md` §5 |
| FR-020 | UC-012 | T-POL-020 | `./Examples.md` §5 |
| FR-021 | UC-014 | T-POL-021 | `./Events.md` §5 |
| FR-022 | UC-003 | T-POL-022 | `./Configuration.md` §2 |
| FR-023 | UC-011 | T-POL-023 | `./Security.md` §4 |
| NFR-001 | UC-001 | T-POL-101 | `./Benchmark.md` §3 |
| NFR-002 | UC-001, UC-002 | T-POL-102 | `./Benchmark.md` §3 |
| NFR-003 | todos | T-POL-103 | `./Monitoring.md` §3 |
| NFR-004 | UC-014 | T-POL-104 | `./Events.md` §3 |
| NFR-005 | UC-001 | T-POL-105 | `./Scalability.md` §5 |
| NFR-006 | UC-007 | T-POL-106 | `./FailureRecovery.md` §5 |
| NFR-007 | UC-010 | T-POL-107 | `./Logging.md` §4 |
| NFR-008 | UC-001 | T-POL-108 | `./Monitoring.md` §4 |
| NFR-009 | UC-010 | T-POL-109 | `./Testing.md` §5 |
| NFR-010 | UC-011 | T-POL-110 | `./FailureRecovery.md` §3 |
| NFR-011 | UC-004 | T-POL-111 | `./Testing.md` §4 |
| NFR-012 | UC-005 | T-POL-112 | `./Benchmark.md` §5 |
| NFR-013 | UC-006 | T-POL-113 | `./API.md` §6 |
| NFR-014 | todos | T-POL-114 | `./Monitoring.md` §2 |
| NFR-015 | UC-006, UC-014 | T-POL-115 | `./Security.md` §3 |
| NFR-016 | UC-001 | T-POL-116 | `./Database.md` §5 |

## 6. Cobertura reversa — Caso de uso → Requisitos

| UC | Nome | Requisitos exercitados |
|----|------|------------------------|
| UC-001 | Avaliar decisão de autorização | FR-001..FR-005, FR-015, FR-017, FR-019, NFR-001, NFR-008, NFR-016 |
| UC-002 | Avaliar decisões em lote | FR-001, NFR-002 |
| UC-003 | Autorar e validar bundle | FR-006, FR-022 |
| UC-004 | Executar testes golden | FR-007, NFR-011 |
| UC-005 | Simular bundle candidato | FR-008, NFR-012 |
| UC-006 | Publicar bundle | FR-009, NFR-013, NFR-015 |
| UC-007 | Reverter bundle (rollback) | FR-010, NFR-006 |
| UC-008 | Conceder/revogar exceção | FR-012, FR-014 |
| UC-009 | Vincular sujeito a papel | FR-012 |
| UC-010 | Explicar decisão | FR-003, FR-015, FR-016, NFR-007, NFR-009 |
| UC-011 | Degradação de fonte de atributo | FR-013, FR-023, NFR-010 |
| UC-012 | Consultar permissões efetivas | FR-020 |
| UC-013 | Registrar fonte de atributo | FR-013 |
| UC-014 | Propagar bundle e invalidar caches | FR-011, FR-021, NFR-004, NFR-015 |
| UC-015 | Alterar o bundle-raiz (autogoverno) | FR-018 |

## 7. Glossário local

Este módulo **não** define termos próprios: `PEP`, `PDP`, `Policy Engine`,
`Capability` e `Default Deny` são reutilizados de `../040-Glossary/Glossary.md` sem
redefinição. Os termos operacionais abaixo são **nomes de artefatos deste módulo**,
descritos (não redefinidos) em `./_DESIGN_BRIEF.md`:

| Termo | Onde é especificado |
|-------|---------------------|
| *Policy bundle* | `./_DESIGN_BRIEF.md` §3.1, `./StateMachine.md` |
| *Obrigação* (`obligation`) | `./_DESIGN_BRIEF.md` §5.1, `./API.md` §3.3 |
| *Waiver* / exceção | `./_DESIGN_BRIEF.md` §3.6 |
| `reason_code` | `./_DESIGN_BRIEF.md` §4.7 |
| Bundle-raiz (`scope = platform`) | `./_DESIGN_BRIEF.md` §12.1, `./Security.md` §2.3 |

> Se algum desses termos passar a ser usado fora do módulo, **DEVERIA** ser promovido
> ao `../040-Glossary/Glossary.md` por PR próprio — nunca definido localmente em
> duplicidade.

## 8. Referências

- `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- `./UseCases.md` · `./Testing.md`
- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim de `Requirements.md`.*
