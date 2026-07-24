---
Documento: API
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0220, ADR-0223, ADR-0224, ADR-0225, ADR-0228
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §5, 004-API (catálogo de erros e registros), 020-Communication, 021-Security
---

# 022-Policy — Contratos de API

> Autenticação, envelope de erro, idempotência, correlação e versionamento seguem
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e §6 — **referenciados, não
> redefinidos**. Pacote gRPC: `aios.policy.v1`. Base REST: `/v1/policy`.

## 1. Superfícies e quando usar cada uma

| Superfície | Uso | Consumidores típicos |
|-----------|-----|----------------------|
| **gRPC** `aios.policy.v1` | Caminho quente de decisão (menor overhead, streaming de lote). | PEPs internos: `../006-Kernel/`, `../004-API/`, `../009-Scheduler/`, `../010-Memory/` |
| **NATS request-reply** `aios.<tenant>.policy.decision.request` | Chamadores que já vivem no barramento e querem preservar o contexto de mensagem. | `../006-Kernel/`, `../007-Agent-Runtime/`, `../020-Communication/` |
| **REST** `/v1/policy` | Autoria, governança, ferramentas, console e integração externa. | `../030-CLI/`, `../032-WebConsole/`, pipelines de CI do tenant |

Os três falam o **mesmo** contrato de decisão (RFC-0220). Divergência entre
superfícies é defeito.

## 2. Autenticação e cabeçalhos

| Item | Regra |
|------|-------|
| AuthN de serviço | **mTLS** com certificado de workload emitido pelo `../021-Security/` (RFC-0001 §6). |
| AuthN de usuário | Token OAuth2/OIDC validado no Gateway (`../004-API/`), claims propagadas. |
| `X-AIOS-Tenant` | **OBRIGATÓRIO**. Divergente do contexto autenticado ⇒ `403 AIOS-POL-0003`. |
| `traceparent`/`tracestate` | **OBRIGATÓRIO** (RFC-0001 §5.6). |
| `Idempotency-Key` | **OBRIGATÓRIO** em mutações administrativas (RFC-0001 §5.5). |
| `X-AIOS-Api-Version` | Versão solicitada (RFC-0001 §5.7). |

## 3. Decisão (caminho quente)

### 3.1 Operações

| Operação | REST | gRPC | NATS | Idempotente | Capability |
|----------|------|------|------|-------------|------------|
| Avaliar | `POST /v1/policy/decisions:evaluate` | `Evaluate` | `aios.<tenant>.policy.decision.request` | sim (leitura pura) | `pol:decision:evaluate` |
| Avaliar em lote | `POST /v1/policy/decisions:evaluateBatch` | `EvaluateBatch` | — | sim (leitura pura) | `pol:decision:evaluate` |
| Explicar | `GET /v1/policy/decisions/{decisionId}:explain` | `ExplainDecision` | — | sim (leitura) | `pol:decision:explain` |
| Listar decisões | `GET /v1/policy/decisions?subject=&from=&to=&pageToken=` | `ListDecisions` | — | sim (leitura) | `pol:decision:read` |

### 3.2 OpenAPI (recorte normativo)

```yaml
openapi: 3.1.0
info: { title: AIOS Policy API, version: "1.0" }
paths:
  /v1/policy/decisions:evaluate:
    post:
      operationId: evaluateDecision
      parameters:
        - { name: X-AIOS-Tenant, in: header, required: true, schema: { type: string } }
        - { name: traceparent,   in: header, required: true, schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/DecisionRequest' }
      responses:
        '200':
          description: Decisão avaliada (allow OU deny — deny NÃO é erro)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/DecisionResponse' }
        '403': { $ref: '#/components/responses/PolicyError' }   # AIOS-POL-0003
        '429': { $ref: '#/components/responses/PolicyError' }   # AIOS-POL-0010
        '504': { $ref: '#/components/responses/PolicyError' }   # AIOS-POL-0011
components:
  schemas:
    DecisionRequest:
      type: object
      required: [subject, action, resource, environment]
      properties:
        subject:
          type: object
          required: [urn]
          properties:
            urn:    { type: string, example: "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6" }
            roles:  { type: array, items: { type: string } }
            claims: { type: object, additionalProperties: true }
        action:
          type: string
          description: "Forma canônica <dominio>:<objeto>:<verbo>; formas curtas são normalizadas."
          example: "memory:item:write"
        resource:
          type: string
          example: "urn:aios:acme:memory:01J9ZA00000000000000000000"
        environment:
          type: object
          required: [tenant]
          properties:
            tenant:    { type: string }
            priority:  { type: integer }
            sourceIp:  { type: string }
            quotaRef:  { type: string }
        attributes:
          type: object
          additionalProperties: true
          description: "Atributos fornecidos pelo PEP; não substituem fontes required."
    DecisionResponse:
      type: object
      required: [effect, reasonCode, matchedRules, decisionTtlMs, bundleVersion, decisionId, evaluatedAt]
      properties:
        effect:        { type: string, enum: [allow, deny] }
        reasonCode:    { type: string, example: "no_matching_rule" }
        matchedRules:  { type: array, items: { type: string } }
        obligations:
          type: array
          items:
            type: object
            required: [kind]
            properties:
              kind:       { type: string, example: "redact_pii" }
              parameters: { type: object, additionalProperties: { type: string } }
        decisionTtlMs: { type: integer, example: 3000 }
        bundleVersion: { type: integer, example: 42 }
        decisionId:    { type: string, example: "01J9ZB00000000000000000000" }
        evaluatedAt:   { type: string, format: date-time }
```

### 3.3 Obrigações (`obligations`)

| `kind` | Parâmetros | Quem cumpre | Semântica |
|--------|-----------|-------------|-----------|
| `redact_pii` | `fields` | PEP do módulo dono do dado | Remover/tokenizar campos antes de devolver. |
| `require_mfa` | — | `../004-API/` | Exigir segundo fator antes de prosseguir. |
| `limit_tokens` | `max` | `../006-Kernel/`, `../017-Model-Router/` | Teto de tokens para a operação autorizada. |
| `sandbox_profile` | `name`, `version` | `../007-Agent-Runtime/` | Perfil de isolamento mínimo (publicado pelo `../021-Security/`). |
| `audit_level` | `level` | PEP chamador | Elevar o nível de trilha para esta operação. |
| `ttl_override` | `ms` | PEP chamador | Reduzir o TTL de cache desta decisão específica. |

> **Regra normativa (ADR-0225).** Um PEP que receba uma obrigação que **não sabe
> cumprir DEVE tratar a decisão como `deny`**. Ignorar obrigação desconhecida
> transformaria uma condição de segurança em comentário — e o `allow` deixaria de
> significar o que a política escreveu.

### 3.4 Exemplos

**Requisição (allow com obrigação):**

```http
POST /v1/policy/decisions:evaluate HTTP/1.1
X-AIOS-Tenant: acme
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{ "subject": { "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
               "roles": ["agent.analyst"] },
  "action": "memory:item:read",
  "resource": "urn:aios:acme:memory:01J9ZA00000000000000000000",
  "environment": { "tenant": "acme", "priority": 5 } }
```

```json
HTTP/1.1 200 OK

{ "effect": "allow",
  "reasonCode": "matched_allow",
  "matchedRules": ["mem.read.own-agent"],
  "obligations": [ { "kind": "redact_pii", "parameters": { "fields": "email,cpf" } } ],
  "decisionTtlMs": 3000,
  "bundleVersion": 42,
  "decisionId": "01J9ZB00000000000000000000",
  "evaluatedAt": "2026-07-23T10:15:00.123Z" }
```

**Explicação da decisão:**

```http
GET /v1/policy/decisions/01J9ZB00000000000000000000:explain HTTP/1.1
X-AIOS-Tenant: acme
```

```json
{ "decisionId": "01J9ZB00000000000000000000",
  "effect": "allow", "reasonCode": "matched_allow", "bundleVersion": 42,
  "evaluated": [
    { "ruleKey": "mem.read.own-agent", "effect": "allow", "priority": 50, "matched": true },
    { "ruleKey": "mem.read.cross-agent", "effect": "deny", "priority": 40, "matched": false,
      "discardedBecause": "subject_match: agente não pertence ao grupo 'supervisors'" }
  ],
  "attributesUsed": ["principal.groups", "resource.data_class"],
  "attributesDigest": "sha256:1f3a…",
  "reproducible": true }
```

### 3.5 Semântica de `deny`

Um `deny` é uma resposta **`200`** com `effect = "deny"` — **não** um erro HTTP. Os
códigos `AIOS-POL-*` cobrem falhas de administração e de avaliação. Confundir negação
com falha faria todo PEP tratar política restritiva como incidente — e a reação a
incidente é sempre afrouxar.

## 4. gRPC (`aios.policy.v1`)

```proto
syntax = "proto3";
package aios.policy.v1;

service PolicyDecision {
  rpc Evaluate       (DecisionRequest)      returns (DecisionResponse);
  rpc EvaluateBatch  (BatchDecisionRequest) returns (BatchDecisionResponse);
  rpc ExplainDecision(ExplainRequest)       returns (Explanation);
}

service PolicyAdmin {
  rpc CreateBundle     (CreateBundleRequest)  returns (BundleRef);
  rpc PutBundleContent (BundleContent)        returns (BundleRef);
  rpc ValidateBundle   (BundleRefRequest)     returns (ValidationReport);
  rpc RunPolicyTests   (BundleRefRequest)     returns (TestReport);
  rpc RunSimulation    (SimulationRequest)    returns (SimulationRef);
  rpc PublishBundle    (PublishRequest)       returns (BundleRef);
  rpc RollbackBundle   (RollbackRequest)      returns (BundleRef);
  rpc GetBundle        (BundleRefRequest)     returns (Bundle);
  rpc ListBundles      (ListBundlesRequest)   returns (ListBundlesResponse);
  rpc PutRole          (Role)                 returns (RoleRef);
  rpc CreateRoleBinding(RoleBinding)          returns (RoleBindingRef);
  rpc DeleteRoleBinding(RoleBindingRefRequest)returns (Empty);
  rpc GetEffectivePermissions(SubjectRequest) returns (EffectivePermissions);
  rpc GrantException   (ExceptionRequest)     returns (ExceptionRef);
  rpc RevokeException  (ExceptionRefRequest)  returns (Empty);
  rpc PutAttributeSource(AttributeSource)     returns (AttributeSourceRef);
  rpc PutPolicyTest    (PolicyTest)           returns (PolicyTestRef);
}

message DecisionRequest {
  SubjectRef subject     = 1;
  string     action      = 2;
  string     resource    = 3;
  Environment environment = 4;
  map<string,string> attributes = 5;
}

message DecisionResponse {
  Effect              effect          = 1;   // EFFECT_ALLOW | EFFECT_DENY
  string              reason_code     = 2;
  repeated string     matched_rules   = 3;
  repeated Obligation obligations     = 4;
  int32               decision_ttl_ms = 5;
  int32               bundle_version  = 6;
  string              decision_id     = 7;
  string              evaluated_at    = 8;   // RFC 3339 UTC
}

enum Effect { EFFECT_UNSPECIFIED = 0; EFFECT_ALLOW = 1; EFFECT_DENY = 2; }
```

> `Effect` **não** possui valores `INDETERMINATE` ou `NOT_APPLICABLE`: o colapso
> acontece dentro do módulo (ADR-0224). Adicioná-los ao enum seria convidar cada
> cliente a interpretá-los.

Mapeamento de erro gRPC: `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo
`code` (RFC-0001 §5.4).

## 5. Administração — operações e capabilities

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Criar bundle | `POST /v1/policy/bundles` | `CreateBundle` | sim (key) | `pol:bundle:author` |
| Editar conteúdo (só em `Draft`) | `PUT /v1/policy/bundles/{urn}/rules` | `PutBundleContent` | sim | `pol:bundle:author` |
| Validar/compilar | `POST /v1/policy/bundles/{urn}:validate` | `ValidateBundle` | sim | `pol:bundle:author` |
| Rodar testes golden | `POST /v1/policy/bundles/{urn}:test` | `RunPolicyTests` | sim | `pol:bundle:test` |
| Simular | `POST /v1/policy/bundles/{urn}:simulate` | `RunSimulation` | sim (key) | `pol:bundle:simulate` |
| Publicar | `POST /v1/policy/bundles/{urn}:publish` | `PublishBundle` | sim (key) | `pol:bundle:publish` |
| Rollback | `POST /v1/policy/bundles/{urn}:rollback` | `RollbackBundle` | sim (key) | `pol:bundle:rollback` |
| Ler/listar bundles | `GET /v1/policy/bundles[/{urn}]` | `GetBundle`/`ListBundles` | sim | `pol:bundle:read` |
| Ler simulação | `GET /v1/policy/simulations/{urn}` | — | sim | `pol:bundle:simulate` |
| Definir papel | `PUT /v1/policy/roles/{roleKey}` | `PutRole` | sim | `pol:role:write` |
| Vincular papel | `POST /v1/policy/role-bindings` | `CreateRoleBinding` | sim (key) | `pol:binding:write` |
| Remover vínculo | `DELETE /v1/policy/role-bindings/{urn}` | `DeleteRoleBinding` | sim | `pol:binding:write` |
| Permissões efetivas | `GET /v1/policy/subjects/{urn}/effective` | `GetEffectivePermissions` | sim | `pol:subject:read` |
| Conceder exceção | `POST /v1/policy/exceptions` | `GrantException` | sim (key) | `pol:exception:grant` |
| Revogar exceção | `POST /v1/policy/exceptions/{urn}:revoke` | `RevokeException` | sim | `pol:exception:revoke` |
| Fonte de atributo | `PUT /v1/policy/attribute-sources/{key}` | `PutAttributeSource` | sim | `pol:attribute:manage` |
| Caso de teste | `PUT /v1/policy/tests/{testKey}` | `PutPolicyTest` | sim | `pol:bundle:test` |

Toda operação desta seção passa pelo `SelfGovernanceGuard`, avaliada contra o
**bundle-raiz** de escopo `platform` (`./Security.md` §2.3) — o PDP autoriza a si
mesmo com *default deny*.

**Exemplo de publicação:**

```http
POST /v1/policy/bundles/urn:aios:acme:policy:01J9ZC00000000000000000000:publish HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZD00000000000000000000
Content-Type: application/json

{ "approvedBy": "urn:aios:acme:principal:01J9ZE00000000000000000000",
  "simulationUrn": "urn:aios:acme:policy:01J9ZF00000000000000000000" }
```

```json
HTTP/1.1 200 OK
{ "urn": "urn:aios:acme:policy:01J9ZC00000000000000000000",
  "bundleVersion": 43, "state": "Active",
  "activatedAt": "2026-07-23T10:20:00.000Z",
  "supersededUrn": "urn:aios:acme:policy:01J9ZB90000000000000000000" }
```

## 6. Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínio reservado: **`POL`** (0001–0099), registrado em
> `../004-API/Errors.md` conforme RFC-0001 §8 (ratificação em **ADR-0220**).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-POL-0001` | 404 | não | Bundle, regra, papel, vínculo, exceção ou decisão não encontrado. |
| `AIOS-POL-0002` | 403 | não | Operação administrativa negada pelo autogoverno (*default deny*). |
| `AIOS-POL-0003` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-POL-0004` | 422 | não | Bundle não compila: erro de sintaxe ou expressão inválida. |
| `AIOS-POL-0005` | 409 | não | Transição de estado inválida do bundle (viola a FSM). |
| `AIOS-POL-0006` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-POL-0007` | 422 | não | Testes obrigatórios reprovados ou cobertura abaixo do mínimo. |
| `AIOS-POL-0008` | 409 | não | Conflito de regras em modo `strict` (contradição, sombreamento). |
| `AIOS-POL-0009` | 503 | sim | Fonte de atributo `required` indisponível (superfície administrativa). |
| `AIOS-POL-0010` | 429 | sim | Limite de avaliações por tenant excedido. |
| `AIOS-POL-0011` | 504 | sim | Orçamento de tempo de avaliação estourado. |
| `AIOS-POL-0012` | 422 | não | Referência a papel, atributo ou fonte inexistente. |
| `AIOS-POL-0013` | 403 | não | Exceção expirada, revogada, sem aprovação ou com prazo acima do máximo. |
| `AIOS-POL-0014` | 412 | não | Bundle sem assinatura ou com assinatura inválida. |
| `AIOS-POL-0015` | 409 | não | Rollback impossível: alvo arquivado ou fora da retenção. |
| `AIOS-POL-0016` | 422 | não | Complexidade acima do limite (regras, profundidade, tamanho, lote). |
| `AIOS-POL-0017` | 409 | não | Grafo de herança de papéis cíclico. |
| `AIOS-POL-0018` | 403 | não | Aprovação dupla exigida e ausente. |
| `AIOS-POL-0019` | 422 | não | Obrigação desconhecida ou incompatível com o PEP declarado. |
| `AIOS-POL-0020` | 409 | não | Bundle imutável: edite criando nova versão. |

**Envelope (RFC 7807), exemplo:**

```json
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{ "type": "https://docs.aios/errors/dual-approval-required",
  "title": "Dual Approval Required",
  "status": 403,
  "code": "AIOS-POL-0018",
  "detail": "A publicação exige aprovador distinto do autor.",
  "instance": "urn:aios:acme:policy:01J9ZC00000000000000000000",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-23T10:20:00.000Z",
  "retriable": false }
```

## 7. Idempotência, paginação e versionamento

| Aspecto | Regra |
|---------|-------|
| Idempotência | `Idempotency-Key` obrigatório em mutações; resultado persistido ≥ 24 h (RFC-0001 §5.5); repetição devolve o mesmo resultado (NFR-013). |
| Avaliação | `Evaluate` é leitura pura: repetir é sempre seguro e não exige chave. |
| Paginação | `pageToken`/`pageSize` (máx. 200) em `ListBundles` e `ListDecisions`; token opaco e estável. |
| Versionamento | Caminho `/v1` + `X-AIOS-Api-Version`; gRPC em `aios.policy.v1`; mudança incompatível coexiste por ≥ 2 majors (RFC-0001 §5.7). |
| Evolução do contrato de decisão | Novos `reason_code` e `obligation.kind` são **aditivos**; clientes **DEVEM** tolerar valores desconhecidos de `reasonCode` e **DEVEM** negar em `obligation.kind` desconhecido (§3.3). |

## 8. Limites operacionais

| Limite | Valor | Chave | Erro |
|--------|-------|-------|------|
| Avaliações por tenant | 100.000/s | `pol.decision.max_eval_per_s_per_tenant` | `AIOS-POL-0010` |
| Itens por lote | 64 | `pol.decision.batch_max_size` | `AIOS-POL-0016` |
| Orçamento de avaliação | 50 ms | `pol.decision.eval_budget_ms` | `AIOS-POL-0011` |
| Obrigações por decisão | 8 | `pol.obligation.max_per_decision` | `AIOS-POL-0019` |
| Regras por bundle | 10.000 | `pol.bundle.max_rules` | `AIOS-POL-0016` |
| Página máxima | 200 | — | `AIOS-POL-0016` |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §5
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7
- Registros (erros, domínios): `../004-API/Errors.md`, `../004-API/Events.md`
- Barramento: `../020-Communication/API.md` · Eventos: `./Events.md`
- Estruturas: `./ClassDiagrams.md` · Exemplos: `./Examples.md`

*Fim de `API.md`.*
