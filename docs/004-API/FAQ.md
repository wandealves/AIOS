---
Documento: FAQ
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0043
RFCs relacionados: RFC-0001
Depende de: ./Vision.md, ./_DESIGN_BRIEF.md
---

# 004-API — Perguntas Frequentes

## O Gateway autoriza tudo que um agente pode fazer?

**Não.** O Gateway é **PEP grosso de rota** — decide apenas se
tenant/role pode chamar aquela rota. A autorização fina de capacidade (ex.:
"este agente pode invocar esta ferramenta específica?") é decidida pelo PEP
local do serviço de destino (ex.: `006-Kernel`), que consulta o mesmo PDP
(`022-Policy`) com um contexto mais específico. Ver `./Security.md` §1.2 e
Não-Responsabilidade NR-02 do `_DESIGN_BRIEF.md`.

## Por que o Gateway não decide política localmente, já que ele já consulta o PDP?

Porque isso duplicaria a lógica de decisão de política em múltiplos lugares
e quebraria a separação PEP/PDP estabelecida pela `ADR-0008` (global). O
Gateway **aplica** decisões (`allow`/`deny`); ele nunca **decide** política
de negócio — mesmo o *default deny* estrutural é uma postura de segurança
fixa, não uma "política" no sentido de regra de negócio configurável
localmente.

## O que acontece se `021-Security` ou `022-Policy` caírem?

O Gateway degrada graciosamente: usa cache local (JWKS/decisão) até o TTL
configurado e, expirado o cache, aplica `api.auth.fail_mode=closed`
(default) — **nega** a requisição em vez de permitir sem validação. Essa é
uma escolha de segurança deliberada, documentada em `./FailureRecovery.md`
§3. `fail_mode=open` existe apenas como opção de configuração para
ambientes de teste isolados, nunca recomendado em produção.

## Por que os registros de contrato (`RouteDefinition`, `ApiVersion`, etc.) não têm `tenant_id`?

Porque eles são **metadados de plataforma** — o mesmo catálogo de rotas,
versões, códigos de erro e schemas de evento é compartilhado por todos os
tenants do AIOS. Aplicar RLS por `tenant_id` não faria sentido semântico
(não existe uma "rota do tenant acme" diferente de uma "rota do tenant
globex"). Já o estado de borda específico de cada tenant
(`RateLimitPolicy`, `IdempotencyRecord`) carrega `tenant_id` + RLS
normalmente. Esta decisão está documentada e será ratificada em
`ADR-0043` — ver `./Database.md` §5.

## Por que existe um tenant reservado `_platform` nos eventos?

Porque eventos de ciclo de vida de registro de contrato (ex.: uma nova rota
foi registrada) não pertencem a nenhum tenant específico — são eventos
sobre a **plataforma em si**. Usar `_platform` como convenção de tenant
reservado (em vez de, por exemplo, omitir o campo `tenant`) mantém o
envelope CloudEvents da RFC-0001 §5.2 uniforme em 100% dos eventos do AIOS,
sem exceção de schema. Ver `./Events.md` e `ADR-0048` (a propor).

## O Gateway pode ficar sem rotear enquanto o PostgreSQL do `ContractRegistry` está fora do ar?

Não completamente. Leitura de rotas continua funcionando a partir do
**snapshot em memória** carregado por cada réplica; apenas **mutações** de
registro (novas rotas, novas versões) ficam bloqueadas até o PostgreSQL
voltar. Ver `./FailureRecovery.md` — modo de falha "PostgreSQL
(`ContractRegistry`) indisponível".

## Por que uma versão `Retired` não pode voltar a ser roteável?

Porque `Retired` é o único estado **terminal** da FSM de `ApiVersion`
(invariante estrutural, ver `./StateMachine.md` §2.5) — uma vez retirada, a
major não é mais mantida, testada ou coberta por SLO. Reativar uma major
retirada exigiria reintroduzi-la como uma **nova** major (novo `T-01`), não
reverter o estado.

## O que o módulo explicitamente NÃO faz?

Reproduzido de `./Vision.md` §3.2: lógica de negócio de qualquer módulo,
autorização fina de capacidade, emissão/revogação de tokens, persistência de
estado de negócio, trilha de auditoria imutável (o Gateway emite sinais, o
Audit persiste), *scheduling*/*placement* de agentes, sandbox de execução,
definição de topologia de *deployment*, cálculo de custo/orçamento, o
broker de mensageria em si, e a fonte de verdade de política RBAC/ABAC.

## Um cliente pode contornar o rate limit abrindo múltiplas conexões?

Não de forma eficaz: o `RateLimiter` aplica o *token-bucket* por
`(tenant, rota)`, não por conexão TCP/HTTP individual — o limite é agregado
no nível de identidade autenticada (claim `tenant` do JWT), não no nível de
transporte. Ver `./API.md` §6 (`AIOS-API-0004`) e `./Scalability.md` §3.

## Como um módulo novo se integra ao Gateway?

Via processo de PR + ADR/RFC de módulo, registrando sua `ApiVersion`
(`Draft`) e suas `RouteDefinition`s através da superfície de administração
(`./API.md` §1, papel `api-registry-admin`) — nunca por configuração
estática *hardcoded* no Gateway. Ver `./UseCases.md` UC-009/UC-010.

*Fim de `FAQ.md`.*
