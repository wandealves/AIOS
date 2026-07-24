---
Documento: Examples
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0225, ADR-0227
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: API.md, UseCases.md, Configuration.md
---

# 022-Policy — Exemplos

> Exemplos usam o tenant `acme` e os endpoints reais de `./API.md`. O CLI é o do
> `../030-CLI/`; o SDK, o do `../031-SDK/`.

## 1. Hello world — perguntar ao PDP

```bash
curl -sS -X POST https://api.aios.local/v1/policy/decisions:evaluate \
  -H "X-AIOS-Tenant: acme" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -H "Content-Type: application/json" \
  -d '{
        "subject":  { "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
                      "roles": ["agent.analyst"] },
        "action":   "memory:item:read",
        "resource": "urn:aios:acme:memory:01J9ZA00000000000000000000",
        "environment": { "tenant": "acme" }
      }'
```

```json
{ "effect": "allow", "reasonCode": "matched_allow",
  "matchedRules": ["mem.read.own-agent"],
  "obligations": [ { "kind": "redact_pii", "parameters": { "fields": "email,cpf" } } ],
  "decisionTtlMs": 3000, "bundleVersion": 43,
  "decisionId": "01J9ZB00000000000000000000",
  "evaluatedAt": "2026-07-23T10:15:00.123Z" }
```

**Leia a obrigação.** O `allow` só é válido se o chamador redigir `email` e `cpf`. Um
PEP que ignore a obrigação está aplicando uma decisão que o PDP não tomou (ADR-0225).

## 2. Lado do PEP — cache, TTL e obrigações (C#)

```csharp
// PEP do módulo chamador (ex.: 006-Kernel CapabilityEnforcer)
var key = (subjectUrn, action, resourceUrn);

if (_cache.TryGet(key, out var cached))          // camada 0: a maioria para aqui
    return cached;

var decision = await _pdp.EvaluateAsync(new DecisionRequest(
    Subject: subject, Action: action, Resource: resourceUrn,
    Environment: new(Tenant: tenantId, Priority: priority)));

// Obrigação desconhecida ⇒ trate como deny (ADR-0225)
foreach (var o in decision.Obligations)
    if (!_obligationHandlers.ContainsKey(o.Kind))
        return Decision.Deny("obligation_unsatisfiable");

_cache.Set(key, decision, TimeSpan.FromMilliseconds(decision.DecisionTtlMs));
return decision;
```

```csharp
// Invalidação: dois eventos, dois comportamentos (Events.md §2)
bus.Subscribe("aios.acme.policy.bundle.updated",   _ => _cache.ClearTenant("acme"));
bus.Subscribe("aios.acme.policy.decision.updated", e => _cache.ClearSubject(e.Data.SubjectUrn));
```

## 3. Decisão pelo barramento (NATS request-reply)

```csharp
var decision = await bus.RequestAsync<DecisionRequest, DecisionResponse>(
    subject: "aios.acme.policy.decision.request",
    request: new DecisionRequest(subject: agentUrn, action: "memory:item:write", resource: memUrn),
    timeout: TimeSpan.FromSeconds(5));        // bus.request.timeout_ms
```

Mesmo contrato do gRPC (RFC-0220); `traceparent` é preservado pelo barramento
(`../020-Communication/`).

## 4. Escrever um bundle de política

```bash
aios policy bundle create --tenant acme --description "Baseline de memória e ferramentas"
# → urn:aios:acme:policy:01J9ZC00000000000000000000  (state=Draft, version=44)
```

```json
PUT /v1/policy/bundles/urn:aios:acme:policy:01J9ZC00000000000000000000/rules

{ "roles": [
    { "roleKey": "agent.analyst", "inherits": ["agent.basic"], "isPrivileged": false,
      "description": "Agente de análise: lê memória própria, invoca ferramentas aprovadas.",
      "permissions": [ { "actionPattern": "memory:item:read",
                         "resourcePattern": "urn:aios:acme:memory:*" } ] },
    { "roleKey": "agent.basic", "inherits": [], "isPrivileged": false,
      "description": "Capacidades mínimas de qualquer agente.",
      "permissions": [] } ],

  "rules": [
    { "ruleKey": "mem.read.own-agent", "effect": "allow", "priority": 50,
      "actionPattern": "memory:item:read",
      "resourcePattern": "urn:aios:acme:memory:*",
      "subjectMatch": { "role": "agent.analyst" },
      "condition": { "eq": [ { "attr": "resource.owner_agent" }, { "attr": "subject.urn" } ] },
      "obligations": [ { "kind": "redact_pii", "parameters": { "fields": "email,cpf" } } ],
      "description": "Analista lê itens de memória de sua própria propriedade, com PII redigida." },

    { "ruleKey": "mem.read.cross-agent", "effect": "deny", "priority": 40,
      "actionPattern": "memory:item:read",
      "resourcePattern": "urn:aios:acme:memory:*",
      "subjectMatch": { "notInGroup": "supervisors" },
      "condition": { "ne": [ { "attr": "resource.owner_agent" }, { "attr": "subject.urn" } ] },
      "dispensable": false,
      "description": "Ninguém fora de supervisores lê memória de outro agente. Não dispensável por waiver." },

    { "ruleKey": "tool.external.block-over-budget", "effect": "deny", "priority": 30,
      "actionPattern": "tool:*:invoke",
      "resourcePattern": "urn:aios:acme:tool:*",
      "condition": { "lt": [ { "attr": "budget.remaining_usd" }, 1.0 ] },
      "description": "Bloqueia ferramentas quando o orçamento restante do tenant é menor que US$ 1." } ]
}
```

Notas de desenho, todas verificáveis:

- `priority` **menor decide primeiro**: `mem.read.cross-agent` (40) vence
  `mem.read.own-agent` (50) quando ambas casam.
- `dispensable: false` impede que um waiver converta essa negação em `allow`.
- `budget.remaining_usd` vem de `../026-Cost-Optimizer/` como atributo `required` —
  se a fonte cair, a regra nega (`attribute_unavailable`), não libera.
- `description` é obrigatória: uma regra sem intenção declarada é irrevisável.

## 5. Do rascunho à produção

```bash
# 1. Compilar e analisar conflitos
aios policy bundle validate urn:aios:acme:policy:01J9ZC...
# → state=Validated  rules=3  conflicts=0  compiledDigest=sha256:9c1e…

# 2. Testes golden (gate obrigatório)
aios policy bundle test urn:aios:acme:policy:01J9ZC...
# → state=Tested  mandatory=12/12  coverage=0.94

# 3. Simular contra o tráfego real dos últimos 7 dias
aios policy bundle simulate urn:aios:acme:policy:01J9ZC... --window 7d
# → evaluated=842391  flippedToDeny=12  flippedToAllow=0  truncated=false
#   top: tool.external.block-over-budget (12 → deny)

# 4. Revisar o diferencial e publicar (aprovador ≠ autor)
aios policy bundle publish urn:aios:acme:policy:01J9ZC... \
  --approved-by urn:aios:acme:principal:01J9ZE... \
  --idempotency-key 01J9ZD00000000000000000000
# → state=Active  version=44  supersedes=43
```

Os 12 `flippedToDeny` são o valor da simulação: **antes** de publicar, sabe-se
exatamente quantas operações que hoje passam deixarão de passar, e por qual regra.

## 6. Rollback

```bash
aios policy bundle rollback urn:aios:acme:policy:01J9ZC... --target-version 43
# → v44 RolledBack · v43 Active · propagação concluída em 2,4 s
```

Rollback de **política** ≠ rollback de **deploy** (`./Deployment.md` §6). Em incidente,
confirme qual dos dois é o problema antes de agir.

## 7. Exceção temporária (waiver)

```bash
aios policy exception grant \
  --subject urn:aios:acme:principal:01J9ZE... \
  --action "db:migration:apply" \
  --resource "urn:aios:acme:*" \
  --ttl 24h \
  --justification "Janela de migração AIOS-1234, aprovada no CAB de 2026-07-22" \
  --approved-by urn:aios:acme:principal:01J9ZK...
# → urn:aios:acme:policy:01J9ZH...  expiresAt=2026-07-24T11:00:00Z
```

Restrições que o comando aplica: prazo ≤ 72 h (`pol.exception.max_ttl_h`),
`--approved-by` diferente do solicitante e justificativa não vazia. Um waiver renovado
mais de 3 vezes em 30 dias dispara `PolicyExceptionChurnHigh` — sinal de que aquilo
deveria virar regra revisada (`./Monitoring.md` RB-P-09).

## 8. Investigar uma negação

```bash
# O agente recebeu 403; o PEP registrou decisionId
aios policy decision explain 01J9ZB00000000000000000000
```

```json
{ "decisionId": "01J9ZB00000000000000000000",
  "effect": "deny", "reasonCode": "explicit_deny", "bundleVersion": 44,
  "evaluated": [
    { "ruleKey": "mem.read.cross-agent", "effect": "deny", "priority": 40, "matched": true },
    { "ruleKey": "mem.read.own-agent", "effect": "allow", "priority": 50, "matched": true,
      "discardedBecause": "priority 50 > 40 (regra deny de menor priority decidiu)" }
  ],
  "attributesUsed": ["resource.owner_agent", "subject.urn", "principal.groups"],
  "attributesDigest": "sha256:1f3a…", "reproducible": true }
```

A explicação mostra não só a regra que decidiu, mas **a que quase decidiu e por que
foi descartada** — que é normalmente o que o desenvolvedor precisa para corrigir o
agente (ou a política).

## 9. Permissões efetivas de um sujeito

```bash
aios policy subject effective urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6
```

```json
{ "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "roles": ["agent.analyst", "agent.basic"],
  "permissions": [
    { "actionPattern": "memory:item:read", "resourcePattern": "urn:aios:acme:memory:*",
      "from": "role:agent.analyst" } ],
  "activeExceptions": [] }
```

Esta é uma projeção do **RBAC**: não substitui a avaliação, porque regras ABAC dependem
de contexto (orçamento, horário, dono do recurso) que só existe na requisição.

## 10. Caso de teste golden (gate de publicação)

```json
PUT /v1/policy/tests/mem-cross-agent-denied

{ "request": {
    "subject":  { "urn": "urn:aios:acme:agent:01J9ZAAA000000000000000000",
                  "roles": ["agent.analyst"] },
    "action":   "memory:item:read",
    "resource": "urn:aios:acme:memory:01J9ZBBB000000000000000000",
    "environment": { "tenant": "acme" },
    "attributes": { "resource.owner_agent": "urn:aios:acme:agent:01J9ZCCC000000000000000000",
                    "principal.groups": "[]" } },
  "expectedEffect": "deny",
  "expectedReason": "explicit_deny",
  "isMandatory": true }
```

Um caso obrigatório reprovado barra a publicação (`AIOS-POL-0007`) — é assim que a
política de um tenant ganha rede de segurança sem depender de revisão manual perfeita.

## 11. Referências

- Contratos: `./API.md` · Eventos: `./Events.md`
- Casos de uso: `./UseCases.md` · Configuração: `./Configuration.md`
- CLI e SDK: `../030-CLI/`, `../031-SDK/`
- FAQ: `./FAQ.md`

*Fim de `Examples.md`.*
