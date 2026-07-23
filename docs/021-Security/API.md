---
Documento: API
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0212, ADR-0213, ADR-0215, ADR-0217, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: _DESIGN_BRIEF.md §5, 004-API (registro de erros), 022-Policy
---

# 021-Security — Contratos de API

Autenticação, envelope de erro, idempotência, correlação e versionamento seguem
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e **não são redefinidos aqui**.

Este módulo expõe **duas superfícies distintas**:

1. **Endpoints OIDC padronizados** — seguem as especificações OAuth2/OIDC (RFC 6749,
   RFC 7636, RFC 7662, RFC 7009), **não** a convenção interna do AIOS. Isso é
   deliberado: clientes OIDC genéricos precisam funcionar sem adaptação.
2. **API administrativa** — `/v1/security` e gRPC `aios.security.v1`, seguindo todas as
   convenções internas.

---

## 1. Endpoints OIDC

| Endpoint | Método | Descrição |
|----------|--------|-----------|
| `/.well-known/openid-configuration` | GET | *Discovery* do emissor. |
| `/.well-known/jwks.json` | GET | **JWKS** consumido pelo `../004-API/` para validação local. |
| `/oauth2/authorize` | GET | Fluxo interativo (`authorization_code` + PKCE). |
| `/oauth2/token` | POST | `authorization_code`, `refresh_token`, `client_credentials`. |
| `/oauth2/introspect` | POST | Introspecção (RFC 7662). |
| `/oauth2/revoke` | POST | Revogação de token (RFC 7009). |
| `/oauth2/userinfo` | GET | Claims do princípio autenticado. |

### 1.1 Exemplo — JWKS com sobreposição durante a rotação

```http
GET /.well-known/jwks.json HTTP/1.1
```

```json
{
  "keys": [
    { "kty": "EC", "crv": "P-256", "kid": "01J9ZD1M4P6R8T0V2X4Z6B8D0F",
      "use": "sig", "alg": "ES256", "x": "…", "y": "…" },
    { "kty": "EC", "crv": "P-256", "kid": "01J9ZC0K3N5Q7R9T1V3X5Z7A9C",
      "use": "sig", "alg": "ES256", "x": "…", "y": "…" }
  ]
}
```

Duas chaves publicadas simultaneamente: a **nova** (que passará a assinar) e a
**anterior** (que ainda verifica tokens em circulação). A nova aparece aqui **antes**
de ser usada para assinar — inverter essa ordem produziria uma janela em que tokens
válidos seriam rejeitados por caches desatualizados (UC-009 E1).

---

## 2. API Administrativa — Operações

| Operação | REST | gRPC (rpc) | Idempotente | Capability |
|----------|------|-----------|-------------|------------|
| Criar princípio | `POST /v1/security/principals` | `CreatePrincipal` | sim (key) | `sec:principal:write` |
| Ler/listar princípios | `GET /v1/security/principals[/{urn}]` | `GetPrincipal`/`ListPrincipals` | sim (leitura) | `sec:principal:read` |
| Atualizar atributos | `PATCH /v1/security/principals/{urn}` | `UpdatePrincipal` | sim | `sec:principal:write` |
| Suspender princípio | `POST /v1/security/principals/{urn}:suspend` | `SuspendPrincipal` | sim | `sec:principal:disable` |
| Desabilitar princípio | `POST /v1/security/principals/{urn}:disable` | `DisablePrincipal` | sim | `sec:principal:disable` |
| Emitir credencial | `POST /v1/security/credentials` | `IssueCredential` | sim (key) | `sec:credential:issue` |
| Ler credencial (metadados) | `GET /v1/security/credentials/{urn}` | `GetCredential` | sim (leitura) | `sec:credential:read` |
| Rotacionar credencial | `POST /v1/security/credentials/{urn}:rotate` | `RotateCredential` | sim (key) | `sec:credential:rotate` |
| Revogar credencial | `POST /v1/security/credentials/{urn}:revoke` | `RevokeCredential` | sim | `sec:credential:revoke` |
| Consultar revogação | `GET /v1/security/revocations?fingerprint=` | `CheckRevocation` | sim (leitura) | `sec:revocation:read` |
| Listar revogações (CRL) | `GET /v1/security/revocations?since=` | `ListRevocations` | sim (leitura) | `sec:revocation:read` |
| Escrever segredo | `PUT /v1/security/secrets/{path}` | `PutSecret` | sim | `sec:secret:write` |
| Ler segredo (com lease) | `POST /v1/security/secrets/{path}:read` | `ReadSecret` | sim (leitura) | `sec:secret:read` |
| Emitir certificado de workload | `POST /v1/security/certificates` | `IssueWorkloadCert` | sim (key) | `sec:cert:issue` |
| Criar chave | `POST /v1/security/keys` | `CreateKey` | sim | `sec:key:manage` |
| Rotacionar chave | `POST /v1/security/keys/{urn}:rotate` | `RotateKey` | sim | `sec:key:manage` |
| Publicar perfil de sandbox | `POST /v1/security/sandbox-profiles` | `PublishSandboxProfile` | sim | `sec:sandbox:publish` |
| Ler perfil de sandbox | `GET /v1/security/sandbox-profiles/{name}/{version}` | `GetSandboxProfile` | sim (leitura) | `sec:sandbox:read` |
| Registrar confiança federada | `PUT /v1/security/federation/{issuer}` | `PutFederationTrust` | sim | `sec:federation:write` |
| Suspender confiança federada | `POST /v1/security/federation/{issuer}:suspend` | `SuspendFederationTrust` | sim | `sec:federation:write` |
| Assinar / verificar | `POST /v1/security/crypto:sign` / `:verify` | `Sign` / `Verify` | sim | `sec:crypto:use` |

> **Ausência deliberada:** não existe operação que **releia o material** de uma
> credencial ou certificado já emitido (invariante I5). `GetCredential` retorna
> metadados — `kind`, `state`, `purpose`, `fingerprint`, prazos — e nada mais.

---

## 3. OpenAPI (extrato normativo)

```yaml
openapi: 3.1.0
info:
  title: AIOS Security API
  version: "1.0.0"
servers:
  - url: https://api.aios.local/v1/security
components:
  parameters:
    Traceparent:    { name: traceparent,     in: header, required: true, schema: { type: string } }
    Tenant:         { name: X-AIOS-Tenant,   in: header, required: true, schema: { type: string } }
    IdempotencyKey: { name: Idempotency-Key, in: header, required: true, schema: { type: string } }
  schemas:
    Credential:
      type: object
      description: METADADOS da credencial. Nunca contém material secreto.
      required: [urn, principalUrn, kind, state, purpose, fingerprint, notBefore, expiresAt]
      properties:
        urn:              { type: string, example: "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G" }
        principalUrn:     { type: string }
        kind:             { type: string, enum: [access_token, refresh_token, workload_cert, db_role, nats_account, object_key, provider_key, signing_key] }
        state:            { type: string, enum: [Requested, Issued, Active, Rotating, Superseded, Revoked, Expired, Failed] }
        purpose:          { type: string, example: "postgres:memory_rw" }
        fingerprint:      { type: string, description: "sha256 do material público ou do jti" }
        notBefore:        { type: string, format: date-time }
        expiresAt:        { type: string, format: date-time, description: "SEMPRE preenchido (invariante I1)" }
        rotatedTo:        { type: string, nullable: true }
        revokedAt:        { type: string, format: date-time, nullable: true }
        revocationReason: { type: string, nullable: true }
    IssueRequest:
      type: object
      required: [principalUrn, kind, purpose]
      properties:
        principalUrn: { type: string }
        kind:         { type: string }
        purpose:      { type: string }
        ttlSeconds:   { type: integer, description: "Opcional; limitado ao máximo do tipo" }
    Problem:   # RFC 7807 / RFC-0001 §5.4 — NÃO redefinido aqui
      $ref: "https://docs.aios/schemas/problem.json"
paths:
  /credentials:
    post:
      summary: Emite uma credencial (T-01 → T-02 → T-04)
      parameters: [ { $ref: '#/components/parameters/Traceparent' },
                    { $ref: '#/components/parameters/Tenant' },
                    { $ref: '#/components/parameters/IdempotencyKey' } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: '#/components/schemas/IssueRequest' } } }
      responses:
        "201":
          description: "Emitida. O MATERIAL vem nesta resposta e NÃO é recuperável depois."
        "403": { description: "AIOS-SEC-0002 (PDP) ou AIOS-SEC-0006 (atestação)" }
        "422": { description: "AIOS-SEC-0004 (TTL acima do máximo)" }
        "429": { description: "AIOS-SEC-0013 (limite de credenciais ativas)" }
        "503": { description: "AIOS-SEC-0010 (KMS indisponível) — retriable" }
  /credentials/{urn}:
    get:
      summary: Metadados da credencial (nunca o material)
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/Credential' } } } }
```

---

## 4. gRPC (extrato do `.proto`)

```protobuf
syntax = "proto3";
package aios.security.v1;

import "google/protobuf/timestamp.proto";

service SecurityService {
  rpc CreatePrincipal       (Principal)            returns (Principal);
  rpc GetPrincipal          (PrincipalRef)         returns (Principal);
  rpc DisablePrincipal      (DisableRequest)       returns (Principal);
  rpc IssueCredential       (IssueRequest)         returns (IssuedCredential);
  rpc GetCredential         (CredentialRef)        returns (Credential);       // metadados
  rpc RotateCredential      (RotateRequest)        returns (Credential);
  rpc RevokeCredential      (RevokeRequest)        returns (Credential);
  rpc CheckRevocation       (FingerprintRef)       returns (RevocationStatus);
  rpc ListRevocations       (ListRevocationsReq)   returns (RevocationList);
  rpc PutSecret             (PutSecretRequest)     returns (SecretVersion);
  rpc ReadSecret            (SecretRef)            returns (SecretLease);
  rpc IssueWorkloadCert     (CsrRequest)           returns (IssuedCertificate);
  rpc RotateKey             (KeyRef)               returns (KeyMaterial);
  rpc PublishSandboxProfile (SandboxProfile)       returns (SandboxProfile);
  rpc GetSandboxProfile     (ProfileRef)           returns (SandboxProfile);   // com assinatura
  rpc PutFederationTrust    (FederationTrust)      returns (FederationTrust);
  rpc Sign                  (SignRequest)          returns (Signature);
  rpc Verify                (VerifyRequest)        returns (VerifyResult);
}

// Resposta de emissão: ÚNICA mensagem que carrega material.
message IssuedCredential {
  Credential metadata = 1;
  string material = 2;        // ⚠ entregue UMA vez; não recuperável (invariante I5)
  google.protobuf.Timestamp material_valid_until = 3;
}

message Credential {          // usada em toda leitura posterior
  string urn = 1;
  string principal_urn = 2;
  string kind = 3;
  string state = 4;
  string purpose = 5;
  string fingerprint = 6;
  google.protobuf.Timestamp not_before = 7;
  google.protobuf.Timestamp expires_at = 8;   // sempre presente
  string rotated_to = 9;
  string revocation_reason = 10;
}

message SandboxProfile {
  string name = 1;
  int32  profile_version = 2;
  string seccomp_json = 3;
  string cgroups_json = 4;
  repeated string namespaces = 5;
  repeated string egress_allowlist = 6;
  bytes  signature = 7;         // verificada pelo 007 antes de aplicar
  string signed_by = 8;
}
```

A separação entre `IssuedCredential` (com material) e `Credential` (sem) é o contrato
que torna a invariante I5 verificável em tempo de compilação para os consumidores.

---

## 5. Catálogo de Códigos de Erro

Domínios reservados: **`SEC`** (0001–0099) e **`AUTHN`** (0001–0099), registrados no
catálogo mantido por `../004-API/` (RFC-0001 §8).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-SEC-0001` | 404 | não | Princípio, credencial, segredo ou perfil não encontrado. |
| `AIOS-SEC-0002` | 403 | não | Operação negada pelo PDP (*default deny*). |
| `AIOS-SEC-0003` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-SEC-0004` | 422 | não | TTL solicitado acima do máximo do tipo de credencial. |
| `AIOS-SEC-0005` | 409 | não | Transição de estado inválida (viola a FSM). |
| `AIOS-SEC-0006` | 403 | não | Atestação de workload falhou. |
| `AIOS-SEC-0007` | 410 | não | Credencial revogada (irreversível — emita uma nova). |
| `AIOS-SEC-0008` | 423 | não | Princípio suspenso ou desabilitado. |
| `AIOS-SEC-0009` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-SEC-0010` | 503 | sim | KMS/HSM indisponível: emissão suspensa. |
| `AIOS-SEC-0011` | 422 | não | Suíte criptográfica não permitida. |
| `AIOS-SEC-0012` | 409 | não | Perfil de sandbox imutável: publique nova versão. |
| `AIOS-SEC-0013` | 429 | sim | Limite de credenciais ativas por princípio excedido. |
| `AIOS-SEC-0014` | 412 | não | Segredo lido fora do *lease* válido. |
| `AIOS-AUTHN-0001` | 401 | não | Credenciais inválidas. |
| `AIOS-AUTHN-0002` | 401 | não | Token expirado. |
| `AIOS-AUTHN-0003` | 401 | não | Assinatura inválida ou `issuer` desconhecido. |
| `AIOS-AUTHN-0004` | 401 | não | Segundo fator exigido e não fornecido. |
| `AIOS-AUTHN-0005` | 429 | sim | Excesso de tentativas de autenticação. |
| `AIOS-AUTHN-0006` | 401 | não | Identidade federada não confiável. |
| `AIOS-AUTHN-0007` | 403 | não | Escopo fora do permitido para a identidade federada. |

### 5.1 Regra de mensagens não-oráculo

`AIOS-AUTHN-0001` **NÃO DEVE** distinguir "princípio inexistente" de "credencial
incorreta". Ambos produzem exatamente a mesma resposta, com o mesmo tempo de execução:

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json

{
  "type": "https://docs.aios/errors/invalid-credentials",
  "title": "Invalid Credentials",
  "status": 401,
  "code": "AIOS-AUTHN-0001",
  "detail": "As credenciais fornecidas não são válidas.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "retriable": false
}
```

O motivo real (`principal_not_found` × `bad_secret`) vai **apenas** para
`../025-Audit/` e para o `ThreatSignalDetector`. Diferenciar as respostas — inclusive
pelo tempo de resposta — transformaria o endpoint em um oráculo de enumeração de contas.

---

## 6. Exemplos

### 6.1 Emitir credencial dinâmica de banco

```bash
curl -sX POST https://api.aios.local/v1/security/credentials \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9ZD3P6R8T0V2X4Z6B8D0F2H" \
  -H "Content-Type: application/json" \
  -d '{ "principalUrn": "urn:aios:acme:principal:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
        "kind": "db_role",
        "purpose": "postgres:memory_rw",
        "ttlSeconds": 3600 }'
```

```json
{
  "metadata": {
    "urn": "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G",
    "kind": "db_role",
    "state": "Active",
    "purpose": "postgres:memory_rw",
    "fingerprint": "sha256:7c4a…9e21",
    "notBefore": "2026-07-22T16:00:00Z",
    "expiresAt": "2026-07-22T17:00:00Z"
  },
  "material": {
    "username": "memory_rw_01J9ZD2N",
    "password": "<entregue uma única vez — não recuperável>"
  }
}
```

Uma leitura posterior de `GET /v1/security/credentials/{urn}` retorna **apenas** o bloco
`metadata`.

### 6.2 Revogar credencial

```bash
curl -sX POST https://api.aios.local/v1/security/credentials/urn:…:01J9ZD2N/revoke \
  -H "X-AIOS-Tenant: acme" \
  -H "Content-Type: application/json" \
  -d '{ "reason": "compromise", "detail": "chave observada em repositório público" }'
```

```json
{
  "urn": "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G",
  "state": "Revoked",
  "revokedAt": "2026-07-22T16:12:44Z",
  "revocationReason": "compromise",
  "propagationTargetSeconds": 30
}
```

Tentar reverter:

```json
{
  "code": "AIOS-SEC-0005",
  "status": 409,
  "title": "Invalid State Transition",
  "detail": "Revoked é terminal e irreversível. Emita uma nova credencial.",
  "retriable": false
}
```

### 6.3 Verificar revogação (usado pelos verificadores)

```bash
curl -s "https://api.aios.local/v1/security/revocations?fingerprint=sha256:7c4a…9e21"
```

```json
{ "revoked": true, "reason": "compromise", "effectiveAt": "2026-07-22T16:12:44Z" }
```

### 6.4 Ler perfil de sandbox (consumido pelo `007`)

```bash
curl -s https://api.aios.local/v1/security/sandbox-profiles/agent-default/3
```

```json
{
  "name": "agent-default",
  "profileVersion": 3,
  "namespaces": ["pid", "net", "mnt", "ipc", "uts", "user"],
  "cgroups": { "cpuMs": 2000, "memoryMiB": 512, "pidsMax": 64 },
  "egressAllowlist": ["api.openai.com:443", "api.anthropic.com:443"],
  "signature": "MEUCIQ…",
  "signedBy": "urn:aios:_platform:key:01J9ZD4Q7S9U1W3Y5A7C9E1G3J"
}
```

O `../007-Agent-Runtime/` **verifica a assinatura antes de aplicar**; assinatura
inválida faz o runtime recusar iniciar o agente (NFR-015).

---

## 7. Versionamento e Compatibilidade

- API administrativa versionada por caminho (`/v1/…`) e header `X-AIOS-Api-Version`
  (RFC-0001 §5.7); mudanças incompatíveis coexistem por ≥ 2 majors.
- Endpoints OIDC seguem as RFCs correspondentes e **não** são versionados pelo esquema
  interno — clientes genéricos dependem dos caminhos padronizados.
- `kid` no JWKS permite adicionar chaves sem quebrar validadores.
- Códigos de erro são estáveis; novos casos recebem números novos.
- Novos `kind` de credencial são **aditivos**; consumidores **DEVEM** ignorar valores
  desconhecidos em vez de falhar.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §5 · Casos de uso: `./UseCases.md`
- FSM: `./StateMachine.md` · Eventos: `./Events.md` · Exemplos: `./Examples.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Registro de erros: `../004-API/`
