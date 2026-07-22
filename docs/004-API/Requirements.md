---
Documento: Requirements
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0042, ADR-0043, ADR-0044, ADR-0045, ADR-0046, ADR-0047, ADR-0048, ADR-0049
RFCs relacionados: RFC-0001, RFC-0040, RFC-0041
Depende de: ./FunctionalRequirements.md, ./NonFunctionalRequirements.md, ./UseCases.md, ./Testing.md
---

# 004-API — Requisitos (Índice Consolidado)

> Este documento consolida `./FunctionalRequirements.md` e
> `./NonFunctionalRequirements.md`, define os *stakeholders* do módulo e fixa a
> **matriz de rastreabilidade** Requisito → Caso de Uso → Teste. Os termos
> técnicos usados aqui reutilizam `../040-Glossary/Glossary.md` sem redefinição.

## 1. Stakeholders

| Stakeholder | Interesse principal | Documentos relevantes |
|-------------|----------------------|--------------------------|
| Desenvolvedor de Agentes (via `031-SDK`) | Contrato estável, versionado, previsível. | `./API.md`, `./Examples.md` |
| Operador de Plataforma (SRE) | Latência/disponibilidade de borda, controles operacionais. | `./Monitoring.md`, `./Configuration.md` |
| Arquiteto Enterprise | Consistência dos registros centrais de contrato. | `./Database.md`, `./Events.md` |
| CISO / Compliance | Cobertura de AuthN/AuthZ de borda, auditoria de sinais de segurança. | `./Security.md` |
| Integrador Externo / Parceiro | Estabilidade e depreciação previsível da API pública. | `./API.md`, `./StateMachine.md` |
| Times donos dos módulos upstream (`006`–`019`, `021`–`027`) | Roteamento correto, isolamento de falha, publicação de contrato. | `./Architecture.md`, `./FailureRecovery.md` |

## 2. Índice de requisitos

- Requisitos funcionais: `FR-001`..`FR-015` + `FR-011a`..`FR-011c` — ver
  `./FunctionalRequirements.md`.
- Requisitos não-funcionais: `NFR-001`..`NFR-012` — ver
  `./NonFunctionalRequirements.md`.
- Casos de uso: `UC-001`..`UC-015` — ver `./UseCases.md`.

## 3. Matriz de rastreabilidade — Requisito → Caso de Uso → Teste

| Requisito | Caso(s) de Uso | Cenário(s) de teste (`./Testing.md`) |
|-----------|------------------|------------------------------------------|
| FR-001 | UC-001 | TC-API-001 (roteamento *happy path*) |
| FR-002 | UC-002 | TC-API-002 (token inválido/expirado) |
| FR-003 | UC-003 | TC-API-003 (autorização de rota negada) |
| FR-004 | UC-004 | TC-API-004 (rate limit excedido) |
| FR-005 | UC-001, UC-013 | TC-API-005 (correlação end-to-end) |
| FR-006 | UC-005 | TC-API-006 (idempotência: repetição e reuse divergente) |
| FR-007 | UC-006 | TC-API-007 (validação de schema) |
| FR-008 | UC-002, UC-003, UC-014 | TC-API-008 (envelope de erro normalizado) |
| FR-009 | UC-007, UC-008 | TC-API-009 (versionamento: deprecated e retired) |
| FR-010 | UC-015 | TC-API-010 (agregação de contrato) |
| FR-011 | UC-009, UC-010, UC-011, UC-012 | TC-API-011 (registro de contrato governado) |
| FR-011a | UC-009 | TC-API-011 (papel `api-registry-admin`) |
| FR-011b | UC-010, UC-011, UC-012 | TC-API-011 (guardas da FSM de `ApiVersion`) |
| FR-011c | UC-016 | TC-API-016 (catálogo público sem AuthN) |
| FR-012 | UC-013 | TC-API-012 (SSE por tenant) |
| FR-013 | UC-014 | TC-API-013 (circuit breaker/bulkhead) |
| FR-014 | UC-002 | TC-API-014 (tenant divergente) |
| FR-015 | UC-013 | TC-API-015 (gRPC-Web) |
| NFR-001/002 | UC-001 | Benchmark `./Benchmark.md` §"Carga sustentada" |
| NFR-003/006 | UC-014 | Chaos/DR drill `./FailureRecovery.md` |
| NFR-004 | UC-013 | Teste de escala `./Scalability.md` |
| NFR-007 | UC-004 | Teste concorrente de token-bucket |
| NFR-008/010 | UC-002, UC-005 | Pentest + teste de *replay* |
| NFR-011 | UC-015 | TC-API-010 |
| NFR-012 | UC-014 | Chaos test de falha isolada |

## 4. Glossário local

Este módulo **não redefine** termos — todos os termos técnicos (Tenant, URN,
Idempotency, PEP/PDP, CloudEvents, Circuit Breaker, Bulkhead, mTLS, OIDC)
DEVEM ser lidos a partir de `../040-Glossary/Glossary.md`. Termos específicos
de contrato deste módulo (`RouteDefinition`, `ApiVersion`,
`ContractRegistry`) são definidos operacionalmente em `./Database.md` e
`./ClassDiagrams.md`, e não constituem redefinição de termos globais.

*Fim de `Requirements.md`.*
