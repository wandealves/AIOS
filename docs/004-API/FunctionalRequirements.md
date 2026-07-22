---
Documento: FunctionalRequirements
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0042, ADR-0043, ADR-0044, ADR-0045, ADR-0046, ADR-0047
RFCs relacionados: RFC-0001, RFC-0040, RFC-0041
Depende de: ./_DESIGN_BRIEF.md §7.1, ./UseCases.md, ./Testing.md
---

# 004-API — Requisitos Funcionais

> Todos os requisitos abaixo são reproduzidos e detalhados a partir de
> `./_DESIGN_BRIEF.md` §7.1 (fonte única de verdade). Os IDs `FR-NNN` são
> **congelados** a partir deste documento e reutilizados literalmente em
> `./Requirements.md`, `./UseCases.md`, `./Testing.md` e `./SequenceDiagrams.md`.
> Prioridade em escala MoSCoW: **Must**, **Should**, **Could**, **Won't (nesta
> versão)**.

## 1. Tabela consolidada

| ID | Requisito | Prioridade | Critério de aceite | Origem (brief) |
|----|-----------|-----------|--------------------|-----------------|
| FR-001 | O Gateway DEVE terminar REST e gRPC no *edge* e rotear conforme a `RouteDefinition` ativa. | Must | Toda rota registrada resolve ao *cluster* upstream correto; 100% de cobertura em contract test. | §7.1 R-01 |
| FR-002 | O Gateway DEVE validar o token OAuth2/OIDC (JWT) via JWKS de `021-Security` antes de rotear. | Must | Token ausente/inválido/expirado retorna `AIOS-API-0002`; a requisição nunca alcança o upstream. | §7.1 R-02 |
| FR-003 | O Gateway DEVE consultar o PDP (`022-Policy`) para autorização de rota antes de encaminhar. | Must | Sem decisão do PDP = *deny*; requisição retorna `AIOS-API-0003`; a decisão é auditada. | §7.1 R-03 |
| FR-004 | O Gateway DEVE aplicar rate limiting/quota por tenant e por rota. | Must | Excesso retorna `429`/`AIOS-API-0004` com `Retry-After`. | §7.1 R-04 |
| FR-005 | O Gateway DEVE propagar/gerar os cabeçalhos de correlação obrigatórios (RFC-0001 §5.6) em toda chamada. | Must | 100% das requisições upstream carregam `traceparent` + `X-AIOS-Tenant`. | §7.1 R-05 |
| FR-006 | O Gateway DEVE garantir idempotência de borda por `Idempotency-Key` (retenção ≥ 24h). | Must | Repetição retorna resultado idêntico; payload divergente com a mesma chave retorna `AIOS-API-0012`. | §7.1 R-06 |
| FR-007 | O Gateway DEVERIA validar o payload contra o schema registrado (OpenAPI/proto) antes de encaminhar. | Should | Payload não conforme retorna `AIOS-API-0007` sem chamar o upstream. | §7.1 R-07 |
| FR-008 | O Gateway DEVE normalizar todo erro (próprio ou de upstream) no envelope RFC-0001 §5.4. | Must | 100% das respostas de erro conformes ao schema RFC 7807 do AIOS. | §7.1 R-08 |
| FR-009 | O Gateway DEVE resolver a versão via `/vN` + header e sustentar coexistência de ≥ 2 majors por módulo. | Must | Versão `Retired` retorna `410`; versão `Deprecated` inclui `Sunset`; nunca há < 2 majors coexistentes sem plano de retirada aprovado. | §7.1 R-09 |
| FR-010 | O Gateway DEVE agregar e publicar o contrato unificado (OpenAPI + proto) dos módulos upstream. | Must | `/v1/api/openapi.json` reflete os contratos ativos em ≤ 60 s após publicação upstream (NFR-011). | §7.1 R-10 |
| FR-011 | O Gateway DEVE manter os registros de RFC-0001 §8 (tipos URN, subjects, códigos de erro, `dataschema`). | Must | Toda entrada nova exige PR + ADR/RFC referenciado; o catálogo é consultável via API. | §7.1 R-11 |
| FR-012 | O Gateway DEVERIA suportar *streaming* SSE de eventos de tarefa por tenant. | Should | Cliente recebe eventos `task.execution.*` do seu tenant em ≤ 2 s de latência de entrega. | §7.1 R-12 |
| FR-013 | O Gateway DEVE isolar upstreams com falha via circuit breaker por serviço. | Must | Falha de um módulo não degrada a latência de rotas de outros módulos (comprovado em chaos test). | §7.1 R-13 |
| FR-014 | O Gateway DEVE rejeitar `X-AIOS-Tenant` divergente do tenant autenticado. | Must | Retorna `AIOS-API-0016`; evento `api.request.rejected` é emitido. | §7.1 R-14 |
| FR-015 | O Gateway DEVERIA transcodificar REST↔gRPC (gRPC-Web) para o Web Console. | Should | Console consome serviços gRPC internos sem cliente gRPC nativo no *browser*. | §7.1 R-15 |

## 2. Requisitos de administração/registro (superfície `RegistryService`)

Complementam FR-011 e derivam diretamente da superfície de API descrita em
`./_DESIGN_BRIEF.md` §5.1; cada operação é rastreável por FR-011 e testada
como caso de uso específico (ver `./UseCases.md` UC-009..UC-012).

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-011a | O Gateway DEVE permitir registrar/atualizar uma `RouteDefinition` apenas ao papel `api-registry-admin`. | Must | Tentativa sem o papel retorna `403`/`AIOS-API-0003`; mutação bem-sucedida emite `api.route.registered`/`api.route.updated`. |
| FR-011b | O Gateway DEVE permitir publicar/depreciar/retirar uma `ApiVersion` respeitando as guardas da FSM (`./StateMachine.md`). | Must | Transição inválida é rejeitada; transição válida emite o evento correspondente e é OCC-segura. |
| FR-011c | O Gateway DEVE expor consulta pública (sem AuthN) ao catálogo de erros, tipos de recurso e schemas de evento. | Must | `GET /v1/api/errors`, `/v1/api/resource-types`, `/v1/api/events/schemas` respondem `200` sem autenticação. |

## 3. Requisitos explicitamente fora de escopo (Won't, nesta versão)

Derivados das Não-Responsabilidades do brief (§1.3); listados aqui para
fechar o espaço de requisitos e evitar ambiguidade de escopo.

| Requisito excluído | Dono real | Referência |
|----------------------|-----------|------------|
| Autorização fina de capacidade/recurso | Módulo de destino + `022-Policy` | NR-02 |
| Emissão/renovação/revogação de tokens OAuth2/OIDC | `021-Security` | NR-03 |
| Persistência de estado de negócio (ACB, memória, planos) | Módulo dono do dado | NR-04 |
| Persistência de trilha de auditoria imutável | `025-Audit` | NR-05 |
| *Scheduling*/*placement* de agentes ou tarefas | `009-Scheduler` | NR-06 |

## 4. Rastreabilidade

Ver a matriz consolidada Requisito → UseCase → Teste em `./Requirements.md`
§3. Todo `FR-NNN` DEVE aparecer em ao menos um `UC-NNN` de `./UseCases.md` e
em ao menos um cenário de `./Testing.md`.

*Fim de `FunctionalRequirements.md`.*
