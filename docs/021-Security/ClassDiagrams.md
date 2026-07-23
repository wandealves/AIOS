---
Documento: ClassDiagrams
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0212, ADR-0214, ADR-0215, ADR-0216, ADR-0217
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: _DESIGN_BRIEF.md §2, Architecture.md, StateMachine.md, Database.md
---

# 021-Security — Estruturas, Interfaces e Contratos

Diagramas ASCII das estruturas internas do `aios-security-svc` e do
`aios-security-ca` (.NET 10). Nomes de componente conforme `./_DESIGN_BRIEF.md` §2.1.

---

## 1. Visão de Pacotes

```
   Aios.Security.Api          → endpoints OIDC + REST/gRPC administrativo
   Aios.Security.Identity     → IdentityProvider, TokenService, PrincipalStore,
                                FederationConnector
   Aios.Security.Pki          → CertificateAuthority, WorkloadAttestor   (processo isolado)
   Aios.Security.Vault        → SecretStore, CredentialBroker
   Aios.Security.Keys         → KeyManagementService, CryptoService, JwksPublisher
   Aios.Security.Lifecycle    → CredentialStateMachine, RevocationService
   Aios.Security.Isolation    → SandboxProfileRegistry
   Aios.Security.Detection    → ThreatSignalDetector
   Aios.Security.Platform     → PolicyClient, EventEmitter, SecurityTelemetry
```

Regras de dependência:
- `Api` → (`Identity`, `Vault`, `Keys`, `Lifecycle`, `Isolation`, `Detection`) → `Platform`.
- **`Pki` roda em processo separado** e é alcançado apenas por gRPC interno com mTLS —
  a superfície que emite certificados não compartilha memória com a que serve
  endpoints OIDC públicos.
- `Platform` **NÃO DEVE** depender de nenhum outro pacote (evita ciclo).

---

## 2. Diagrama de Componentes e Relações

```
  ┌──────────────────────┐        ┌────────────────────────┐
  │  IdentityProvider    │───────▶│     TokenService       │
  │──────────────────────│        │────────────────────────│
  │ + Authorize(req)     │        │ + Issue(principal,ttl) │
  │ + Token(grant)       │        │ + Introspect(jwt)      │
  │ + Consent(...)       │        │ + Revoke(jti)          │
  └──────┬───────────────┘        └───────┬────────────────┘
         │                                 │ assina com
         ▼                                 ▼
  ┌──────────────────────┐        ┌────────────────────────┐    ┌──────────────────┐
  │   PrincipalStore     │        │ KeyManagementService   │───▶│  JwksPublisher   │
  │ + Get/Upsert         │        │ + ActiveSigningKey()   │    │ + Publish()      │
  │ + Attributes(urn)    │        │ + Wrap/Unwrap(dek)     │    │ + Rotate()       │
  │ + Disable(urn)       │        │ + Rotate(role)         │    └──────────────────┘
  └──────┬───────────────┘        └───────┬────────────────┘
         │                                 │              ┌──────────────────┐
         │                                 ├─────────────▶│  CryptoService   │
         │                                 │              │ + Sign/Verify    │
         ▼                                 │              │ + Hash/Random    │
  ┌──────────────────────┐                 │              └──────────────────┘
  │ FederationConnector  │                 │
  │ + PutTrust(issuer)   │                 ▼
  │ + MapExternal(jwt)   │        ┌────────────────────────┐
  └──────────────────────┘        │      SecretStore       │
                                   │ + Put(path, value)     │
  ┌──────────────────────┐        │ + Read(path) : Lease   │
  │ CredentialStateMach. │◀──────▶│ (envelope KEK/DEK)     │
  │ (FSM §4: T-01..T-09) │        └───────┬────────────────┘
  └──────┬───────────────┘                 │
         │                                  ▼
         │                        ┌────────────────────────┐
         ├───────────────────────▶│   CredentialBroker     │──▶ 005 (db_role)
         │                        │ + IssueFor(resource)   │──▶ 020 (nats_account)
         │                        │ + Rotate(urn)          │──▶ 017 (provider_key)
         │                        └───────┬────────────────┘
         ▼                                 │ (workload_cert)
  ┌──────────────────────┐                 ▼
  │  RevocationService   │        ╔════════════════════════════════════════╗
  │ + Revoke(urn,reason) │        ║  PROCESSO ISOLADO: aios-security-ca     ║
  │ + IsRevoked(fp)      │        ║ ┌────────────────────┐ ┌─────────────┐ ║
  │ + List(since)        │        ║ │ CertificateAuthor. │◀│WorkloadAttes│ ║
  └──────────────────────┘        ║ │ + Issue(csr,purp.) │ │ + Attest(req)│ ║
                                   ║ │ + Renew(urn)       │ └─────────────┘ ║
  ┌──────────────────────┐        ║ └─────────┬──────────┘                 ║
  │SandboxProfileRegistry│        ╚═══════════│════════════════════════════╝
  │ + Publish(profile)   │                    │ assinada por
  │ + Get(name,version)  │                    ▼
  │ (assina com signing) │            ┌───────────────┐
  └──────────────────────┘            │  CA RAIZ      │ offline
                                       └───────────────┘
  ┌──────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
  │ThreatSignalDetec.│  │ PolicyClient │  │ EventEmitter │  │SecurityTelemetry │
  │ + Observe(authn) │  │ + Decide(req)│  │ + Enqueue(e) │  │ + Span(name)     │
  └──────────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘
```

---

## 3. Contratos (interfaces públicas)

```
  interface ITokenService
  ────────────────────────────────────────────────────────────────────────────
  + IssueAsync(PrincipalRef, Scope[], Ttl)   : TokenPair    // access + refresh
  + IntrospectAsync(jwt)                     : Introspection
  + RevokeAsync(jti, Reason)                 : void
  Invariantes: TTL ≤ sec.token.max_ttl_s (I1); assinatura com a chave `active`

  interface ICredentialService
  ────────────────────────────────────────────────────────────────────────────
  + IssueAsync(IssueRequest, IdempotencyKey) : Credential   // T-01 → T-02 → T-04
  + GetAsync(Urn)                            : Credential   // METADADOS apenas
  + RotateAsync(Urn, IdempotencyKey)         : Credential   // T-05 → T-06
  + RevokeAsync(Urn, Reason)                 : Credential   // T-07 (irreversível)
  Invariantes: I1 (expiração obrigatória), I2 (no máximo duas utilizáveis),
               I3 (revogação irreversível), I5 (material não relegível)
  ⚠ NÃO EXISTE  GetMaterialAsync(Urn)  — por desenho (I5).

  interface ICertificateAuthority        // processo isolado
  ────────────────────────────────────────────────────────────────────────────
  + IssueWorkloadCertAsync(Csr, Purpose, AttestationProof) : Certificate
  + RenewAsync(Urn)                                        : Certificate
  Invariante: emissão SEM AttestationProof válida é impossível (AIOS-SEC-0006)

  interface ISecretStore
  ────────────────────────────────────────────────────────────────────────────
  + PutAsync(path, value)      : SecretVersion   // versões imutáveis
  + ReadAsync(path)            : SecretLease     // valor + prazo do lease
  Invariante: ciphertext no banco; valor em claro nunca persistido nem logado

  interface IKeyManagementService
  ────────────────────────────────────────────────────────────────────────────
  + ActiveSigningKeyAsync(role)      : KeyRef
  + WrapAsync(dek) / UnwrapAsync(..) : byte[]
  + RotateAsync(role)                : KeyRef     // pending → active → rotating
  Invariante: chave `retired` NÃO é destruída enquanto houver artefato dependente

  interface IRevocationService
  ────────────────────────────────────────────────────────────────────────────
  + RevokeAsync(Urn, Reason)   : RevocationEntry  // grava + emite (I4)
  + IsRevokedAsync(fingerprint): bool             // consultado pelos verificadores
  + ListAsync(since)           : RevocationEntry[]

  interface ISandboxProfileRegistry
  ────────────────────────────────────────────────────────────────────────────
  + PublishAsync(SandboxProfile) : SandboxProfile  // assina; versão imutável
  + GetAsync(name, version)      : SandboxProfile  // com assinatura, para verificação
  Invariante: perfil publicado é imutável (AIOS-SEC-0012)

  interface IPolicyClient   // PEP → PDP (022), default deny
  ────────────────────────────────────────────────────────────────────────────
  + DecideAsync(DecisionRequest) : Decision
  Falha do PDP ⇒ Deny para operações administrativas; autenticação e validação seguem
```

A ausência deliberada de `GetMaterialAsync` no `ICredentialService` é o ponto mais
importante deste documento: a impossibilidade de reler o material não é uma política
que alguém precisa lembrar de aplicar — é a ausência de um método na interface.

---

## 4. Estruturas de Dados (projeção das entidades)

```
  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ Principal                    │  1   * │ Credential                    │
  │──────────────────────────────│───────▶│───────────────────────────────│
  │ Urn          : Urn           │        │ Urn             : Urn         │
  │ TenantId     : string (RLS)  │        │ TenantId        : string (RLS)│
  │ Kind         : {human,       │        │ PrincipalUrn    : Urn         │
  │                 service,     │        │ Kind            : CredKind    │
  │                 agent,       │        │ State           : CredState   │
  │                 external}    │        │ Purpose         : string      │
  │ ExternalRef  : string?       │        │ MaterialRef     : string?     │
  │ DisplayName  : string        │        │ Fingerprint     : Sha256      │
  │ Attributes   : jsonb ────────┼──▶ 022 │ IssuedAt        : DateTime    │
  │   (assertáveis; o PDP        │ (PDP)  │ NotBefore       : DateTime    │
  │    interpreta, o 021 asserta)│        │ ExpiresAt       : DateTime ⚠  │
  │ Status       : {active,      │        │   (NOT NULL — invariante I1)  │
  │                 suspended,   │        │ RotatedTo       : Urn?        │
  │                 disabled}    │        │ RevokedAt       : DateTime?   │
  │ MfaRequired  : bool          │        │ RevocationReason: string?     │
  │ Version      : long (OCC)    │        │ IssuedBy        : Urn         │
  └──────┬───────────────────────┘        │ Version         : long (OCC)  │
         │                                 └──────────┬────────────────────┘
         │ *                                          │ 0..1
         ▼                                            ▼
  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ FederationTrust              │        │ RevocationEntry               │
  │──────────────────────────────│        │───────────────────────────────│
  │ Urn          : Urn           │        │ Id           : Ulid           │
  │ Kind         : {idp,a2a_peer}│        │ CredentialUrn: Urn            │
  │ Issuer       : string        │        │ Fingerprint  : Sha256         │
  │ JwksUri      : string?       │        │ Reason       : string         │
  │ TrustAnchor  : byte[]?       │        │ EffectiveAt  : DateTime       │
  │ AllowedScopes: string[]      │        │ ExpiresAt    : DateTime       │
  │ Status       : {active,      │        │ PropagatedAt : DateTime?      │
  │                 suspended,   │        └───────────────────────────────┘
  │                 revoked}     │
  └──────────────────────────────┘

  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ KeyMaterial                  │  1   * │ Secret                        │
  │──────────────────────────────│───────▶│───────────────────────────────│
  │ Urn          : Urn           │ dek_ref│ Urn          : Urn            │
  │ Role         : {kek, dek,    │        │ Path         : string         │
  │                 signing,     │        │ VersionNo    : int (imutável) │
  │                 ca_root,     │        │ Ciphertext   : byte[] ⚠       │
  │                 ca_interm.}  │        │   (envelope; nunca em claro)  │
  │ Algorithm    : string        │        │ DekRef       : Urn            │
  │ State        : KeyState      │        │ LeaseTtl     : TimeSpan       │
  │ HsmBacked    : bool          │        │ RotationPeriod: TimeSpan?     │
  │ PublicMaterial : byte[]?     │        └───────────────────────────────┘
  │ WrappedMaterial: byte[]?     │
  │ ParentKey    : Urn?  (KEK)   │        ┌───────────────────────────────┐
  │ RetireAfter  : DateTime?     │        │ SandboxProfile                │
  │ DestroyedAt  : DateTime?     │───────▶│───────────────────────────────│
  └──────────────────────────────┘signed_by│ Name         : string        │
                                           │ ProfileVersion: int          │
                                           │ Seccomp      : jsonb         │
                                           │ Cgroups      : jsonb         │
                                           │ Namespaces   : string[]      │
                                           │ EgressAllowlist: string[]    │
                                           │ Signature    : byte[]        │
                                           │ Status       : {draft,       │
                                           │        published, deprecated}│
                                           └───────────────────────────────┘
```

`Urn` e `Sha256` são *value objects* imutáveis; `Urn` valida o formato da RFC-0001 §5.1
sem redefini-lo.

---

## 5. Invariantes de Estrutura

| # | Invariante | Onde é garantido |
|---|-----------|------------------|
| C-01 | `Credential.ExpiresAt` é sempre preenchido e futuro na emissão. | `NOT NULL` no schema + guarda de T-01 (invariante I1) |
| C-02 | Não existe operação de leitura do material privado. | Ausência do método na interface (invariante I5) |
| C-03 | `Credential.State = Revoked` é terminal absoluto. | `ICredentialService` (invariante I3) |
| C-04 | No máximo **duas** credenciais utilizáveis por `Purpose`+`PrincipalUrn`. | *Advisory lock* na rotação (invariante I2) |
| C-05 | `Secret.Ciphertext` nunca contém material em claro; `VersionNo` é imutável. | `ISecretStore.PutAsync` |
| C-06 | `SandboxProfile` publicado é imutável e sempre assinado. | `ISandboxProfileRegistry` → `AIOS-SEC-0012` |
| C-07 | `KeyMaterial` em `retired` não é destruída enquanto houver artefato dependente. | `IKeyManagementService` |
| C-08 | `FederationTrust.AllowedScopes` limita o escopo interno concedido a identidade externa. | `FederationConnector` → `AIOS-AUTHN-0007` |
| C-09 | Toda mutação incrementa `Version` (OCC); conflito → `AIOS-SEC-0009`. | Camada de persistência |

---

## 6. Referências

- Componentes e diagrama-fonte: `./_DESIGN_BRIEF.md` §2
- Arquitetura: `./Architecture.md` · FSM: `./StateMachine.md`
- Modelo físico: `./Database.md` · API: `./API.md`
