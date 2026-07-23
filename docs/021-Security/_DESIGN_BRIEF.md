---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0001, ADR-0002, ADR-0008, ADR-0010 (globais, herdados); ADR-0210..ADR-0219 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — não redefinida); RFC-0210 (Identity & Credential Contract), RFC-0211 (Workload Identity & PKI Profile), a propor por este módulo
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database (schema `security`), 020-Communication (NATS), 022-Policy (PDP), 024-Observability, 025-Audit, 027-Cluster, 028-Deployment; consumido por 004 (JWKS/validação de token), 005 (roles de banco), 007 (perfis de sandbox e segredos), 020 (contas NKey/JWT), e por todos os módulos que precisam de identidade de serviço (mTLS)
---

# 021-Security — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 021-Security. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, subjects, **mTLS interno e validação de
> token no Gateway**), este documento **referencia** a
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §6 e **não o redefine**.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 0. Posição no AIOS e contrato de fronteira

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  CONSUMIDORES DE IDENTIDADE E SEGREDO                                    │
   │  004 API (valida JWT com JWKS) · 005 Database (roles) · 007 Runtime      │
   │  (perfis de sandbox, segredos) · 020 Comm (contas NKey/JWT) · todos      │
   │  os serviços (certificados mTLS)                                         │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ "quem é você?" · "qual credencial eu uso?" · "com o que eu isolo?"
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  021-SECURITY — RAIZ DE CONFIANÇA (este módulo)                          │
   │  · emite e revoga IDENTIDADE (humano, serviço, agente, par externo)      │
   │  · custodia SEGREDOS e CHAVES; emite credenciais dinâmicas e efêmeras    │
   │  · opera a PKI interna (mTLS) e o KMS (cifragem em repouso)              │
   │  · publica PERFIS DE ISOLAMENTO (seccomp, cgroups, egress) para o 007    │
   │  · NÃO decide autorização — isso é do 022-Policy                          │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ claims assinadas · certificados · segredos com lease
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  022-POLICY (PDP)  — consome claims e atributos para DECIDIR             │
   │  025-AUDIT         — registra o que foi emitido, usado e revogado         │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Fronteira em uma frase:** *o 021 responde **quem é** o solicitante e **com que
credencial** ele se prova; o 022 responde **se ele pode**; o 025 registra **que
aconteceu**.* Confundir essas três perguntas é o erro de desenho mais comum em
plataformas com governança — e a razão de os três módulos serem separados.

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O 021-Security é a **raiz de confiança** do AIOS — o análogo do subsistema de
credenciais, chaveiro e capacidades de um sistema operacional endurecido. Ele não
decide o que um sujeito pode fazer; ele estabelece, com prova criptográfica, **quem o
sujeito é** e **que credenciais ele legitimamente porta**, e fornece aos demais
módulos os materiais de segurança que eles não devem inventar por conta própria:
certificados, chaves, segredos, contas e perfis de isolamento.

A tese central do módulo é que **credencial de longa vida é dívida de segurança**.
Toda credencial emitida aqui tem **prazo curto, dono identificado, propósito declarado
e caminho de revogação** — de um certificado mTLS de serviço (TTL de horas) a uma
credencial dinâmica de banco (TTL de minutos). O ciclo de vida de credencial é tão
central que é ele, e não a identidade, a máquina de estados canônica do módulo (§4).

A segunda tese é que **isolamento é material de segurança versionado**. O perfil
`seccomp`/`cgroups`/egress com que o `007-Agent-Runtime` encaixota um agente não é
configuração dispersa em manifesto de deploy: é um artefato assinado, versionado e
auditável, publicado por este módulo — do contrário, o endurecimento de um sandbox
depende de quem editou o YAML por último.

### 1.2 Responsabilidades (o 021-Security DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Operar o **provedor de identidade OIDC** do AIOS: autenticar princípios humanos e clientes de serviço, emitir *access/refresh tokens* (JWT) e expor **introspecção**. |
| R-02 | Publicar e rotacionar o **JWKS** consumido pelo `004-API` para validação de token na borda, emitindo `security.jwks.rotated` a cada rotação. |
| R-03 | Operar a **PKI interna**: CA raiz *offline*, CAs intermediárias, e emissão de **certificados de workload de vida curta** para o mTLS obrigatório entre serviços (RFC-0001 §6). |
| R-04 | Estabelecer **identidade de workload** verificável (atestação do serviço/nó antes da emissão), de modo que um serviço não possa assumir a identidade de outro. |
| R-05 | Custodiar **segredos** versionados com *lease*, e emitir **credenciais dinâmicas** de vida curta para recursos: *roles* de PostgreSQL (`005`), contas NKey/JWT do NATS (`020`), chaves de MinIO, chaves de provedores de LLM (`017`). |
| R-06 | Operar o **KMS**: hierarquia de chaves (KEK/DEK), cifragem envelopada, rotação programada e sob demanda, e interface para HSM. |
| R-07 | Manter o **registro de perfis de isolamento** (`seccomp-bpf`, `cgroups v2`, namespaces, *egress allowlist*) versionados e assinados, consumidos pelo `007-Agent-Runtime`. |
| R-08 | Executar **revogação** de qualquer credencial em qualquer ponto do ciclo, propagando o efeito por evento (`security.token.revoked`) e por lista de revogação consultável. |
| R-09 | Integrar **identidade federada externa** (IdP corporativo do tenant; âncoras de confiança para pares A2A externos do `020`), sem que a confiança externa se torne confiança interna. |
| R-10 | Detectar e sinalizar **anomalias de autenticação** (força bruta, uso de credencial revogada, uso fora do escopo de emissão) como sinais para `024`/`025`, sem decidir bloqueio de política. |
| R-11 | Prover **serviços criptográficos** padronizados (assinatura, verificação, *hashing*, geração de aleatoriedade) e governar as suítes permitidas, evitando implementações ad hoc por módulo. |
| R-12 | Atuar como **PEP** em suas próprias operações administrativas (emitir, rotacionar, revogar, alterar perfil), consultando o **PDP** do `022-Policy` com *default deny*, e registrando tudo em `025-Audit`. |

### 1.3 Não-Responsabilidades (o 021-Security NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | **Decidir autorização.** O 021 estabelece identidade e atributos; a decisão sobre o que o sujeito pode fazer é do PDP. | `022-Policy` |
| NR-02 | Definir o **modelo de papéis e regras** (RBAC/ABAC): quais papéis existem, o que cada um concede. O 021 **asserta** os atributos; o `022` os interpreta. | `022-Policy` |
| NR-03 | Persistir a **trilha de auditoria imutável** (o 021 emite os fatos; o Audit os torna prova). | `025-Audit` |
| NR-04 | Ser **PEP de borda** para rotas de API ou interceptar tráfego de aplicação. | `004-API` |
| NR-05 | **Executar** o sandbox do agente (criar namespaces, aplicar cgroups). O 021 publica o **perfil**; quem aplica é o runtime. | `007-Agent-Runtime` |
| NR-06 | Armazenar dados de negócio ou aplicar RLS. O 021 fornece as *roles*; o modelo físico e a RLS são do banco. | `005-Database` |
| NR-07 | Transportar mensagens ou gerenciar contas do barramento. O 021 **emite** o NKey/JWT; o `020` provisiona a conta e a opera. | `020-Communication` |
| NR-08 | Definir topologia de rede, firewall, segmentação ou WAF. | `028-Deployment`, `027-Cluster` |
| NR-09 | Cifrar colunas de negócio ou decidir classificação de dado (`data_class`). O 021 fornece as chaves; a classificação é do dono do schema. | `005-Database` + módulo dono |
| NR-10 | Bloquear um sujeito por comportamento (resposta a incidente). O 021 **sinaliza** a anomalia; a decisão de bloquear é política. | `022-Policy`, `029-Operations` |
| NR-11 | Definir orçamento/custo de qualquer recurso. | `026-Cost-Optimizer` |
| NR-12 | Ser o repositório de **conteúdo** pessoal do titular (o 021 guarda identificadores e credenciais, não o dado de negócio do usuário). | módulo dono do dado |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **IdentityProvider** | Emissor OIDC: fluxos `authorization_code` (humanos, com PKCE) e `client_credentials` (serviços); *consent*; sessão; introspecção. | TokenService, PrincipalStore, FederationConnector |
| **TokenService** | Emite, renova, introspecta e revoga JWT; aplica TTL curto por tipo de princípio; assina com a chave corrente do `KeyManagementService`. | KeyManagementService, RevocationService |
| **JwksPublisher** | Publica o conjunto de chaves públicas (JWKS) com sobreposição durante a rotação; emite `security.jwks.rotated`. | KeyManagementService, EventEmitter |
| **PrincipalStore** | Registro autoritativo de princípios (humano, serviço, agente, par externo) e de seus **atributos assertáveis** (tenant, grupos, escopos) — insumo do PDP. | → PostgreSQL (`security.*`) |
| **CertificateAuthority** | PKI interna: CA raiz *offline*, intermediárias online, emissão e renovação automática de certificados de workload de vida curta; publica CRL/status. | KeyManagementService, WorkloadAttestor |
| **WorkloadAttestor** | Verifica a identidade do solicitante **antes** de emitir certificado ou credencial (atestação de nó/serviço), impedindo que um workload assuma a identidade de outro. | → 027-Cluster, 028-Deployment |
| **SecretStore** | Cofre de segredos versionados com *lease* e renovação; cifragem envelopada com DEK do KMS; trilha de leitura. | KeyManagementService; → PostgreSQL |
| **CredentialBroker** | Emite **credenciais dinâmicas** por recurso: *role* de PostgreSQL (`005`), conta NKey/JWT do NATS (`020`), chave de MinIO, chave de provedor de LLM (`017`); orquestra a rotação sem downtime (dupla credencial). | SecretStore, CertificateAuthority, PolicyClient |
| **KeyManagementService** | Hierarquia KEK/DEK, cifragem envelopada, rotação programada, interface HSM, *key derivation* e destruição segura. | → HSM/KMS externo (opcional) |
| **RevocationService** | Revoga credenciais em qualquer estado, mantém a lista de revogação consultável e propaga por `security.token.revoked`. | EventEmitter; → 020-Communication |
| **FederationConnector** | Integra IdP externo do tenant (OIDC/SAML) e mantém âncoras de confiança para pares A2A externos; mapeia identidade externa → princípio interno com escopo reduzido. | PrincipalStore, PolicyClient |
| **SandboxProfileRegistry** | Catálogo versionado e assinado de perfis de isolamento (`seccomp`, `cgroups`, namespaces, egress) consumidos pelo `007`. | KeyManagementService (assinatura); → 007 |
| **CryptoService** | Primitivas padronizadas (assinar, verificar, *hash*, aleatoriedade) e governança de suítes permitidas; nega algoritmo fora da lista. | KeyManagementService |
| **ThreatSignalDetector** | Detecta anomalias de autenticação (rajada de falhas, uso de credencial revogada, uso fora do escopo/origem de emissão) e emite sinais — **não** bloqueia. | EventEmitter; → 024, 025 |
| **PolicyClient** | PEP das operações administrativas do próprio módulo; circuit breaker, cache curto, *fail-closed*. | → 022-Policy |
| **EventEmitter** | Publica eventos CloudEvents via **Outbox transacional** em `security.outbox`. | → PostgreSQL, NATS/JetStream |
| **SecurityTelemetry** | Spans/métricas `aios_sec_*`/logs OTel; emissão de auditoria para `025`, **sem** vazar material sensível. | → 024-Observability, 025-Audit |

### 2.2 Diagrama de Componentes (ASCII)

```
        OIDC (humanos/serviços)        gRPC interno (emissão/rotação/revogação)
                    │                                    │
   ┌────────────────▼────────────────────────────────────▼──────────────────────┐
   │                    SECURITY SERVICE (021 · .NET 10)                          │
   │                                                                              │
   │  ┌──────────────────┐      ┌──────────────────┐     ┌────────────────────┐  │
   │  │ IdentityProvider │─────▶│  TokenService    │────▶│  JwksPublisher     │──┼─▶ 004-API
   │  │ (OIDC, sessão)   │      │ (JWT, TTL curto) │     │ (rotação + evento) │  │   (JWKS)
   │  └────────┬─────────┘      └────────┬─────────┘     └────────────────────┘  │
   │           │                         │                                        │
   │  ┌────────▼─────────┐      ┌────────▼──────────┐    ┌────────────────────┐  │
   │  │  PrincipalStore  │◀────▶│ FederationConnec. │    │ RevocationService  │──┼─▶ 020
   │  │ (princípios +    │      │ (IdP externo,     │    │ (CRL + evento)     │  │  (revoga)
   │  │  atributos)      │      │  âncoras A2A)     │    └────────────────────┘  │
   │  └────────┬─────────┘      └───────────────────┘                            │
   │           │                                                                  │
   │  ┌────────▼─────────┐  atesta  ┌──────────────────┐                          │
   │  │ CertificateAuth. │◀────────▶│ WorkloadAttestor │                          │
   │  │ (PKI, mTLS)      │          └──────────────────┘                          │
   │  └────────┬─────────┘                                                        │
   │           │            ┌──────────────────┐      ┌────────────────────────┐ │
   │  ┌────────▼─────────┐  │  SecretStore     │─────▶│  CredentialBroker      │─┼─▶ 005 (roles)
   │  │ KeyManagement    │◀─│ (lease, versão)  │      │ (credenciais dinâmicas)│ │  020 (NKey)
   │  │ Service (KEK/DEK)│  └──────────────────┘      └────────────────────────┘ │  017 (API keys)
   │  └────────┬─────────┘                                                        │
   │           │            ┌──────────────────────┐  ┌────────────────────────┐ │
   │           ├───────────▶│ SandboxProfileRegistry│─┼─▶ 007-Agent-Runtime     │ │
   │           │            │ (seccomp/cgroups/egr.)│  └────────────────────────┘ │
   │           │            └──────────────────────┘                              │
   │  ┌────────▼─────────┐  ┌──────────────────────┐  ┌────────────────────────┐ │
   │  │  CryptoService   │  │ ThreatSignalDetector │  │ PolicyClient (PEP)     │─┼─▶ 022
   │  └──────────────────┘  └──────────────────────┘  └────────────────────────┘ │
   │  ┌────────────────────────────────────────────────────────────────────────┐ │
   │  │ EventEmitter (Outbox → JetStream)  ·  SecurityTelemetry (OTel + audit) │ │
   │  └────────────────────────────────────────────────────────────────────────┘ │
   └──────────┬──────────────────────┬───────────────────────┬───────────────────┘
              ▼                      ▼                       ▼
     PostgreSQL (schema security)  HSM / KMS externo   NATS aios.*.security.*
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN, ULID) e às convenções físicas do
> `../005-Database/Database.md` §2: `tenant_id` + RLS em toda tabela multi-tenant,
> `version bigint` para OCC, `timestamptz` em UTC. Schema: **`security`**.
>
> **Regra absoluta deste modelo:** nenhuma tabela armazena **material secreto em
> claro**. Segredos e chaves privadas são gravados **cifrados por envelope** (DEK do
> KMS) ou permanecem no HSM; o que a tabela guarda é metadado, referência e *hash*.

### 3.1 Entidade `Principal` — tabela `security.principal` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:principal:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento. |
| `kind` | `text` | NOT NULL, CHECK ∈ {human, service, agent, external} | Natureza do princípio. |
| `external_ref` | `text` | NULL | Identificador no IdP externo (quando federado). |
| `display_name` | `text` | NOT NULL | Nome legível (não é PII sensível por si). |
| `attributes` | `jsonb` | NOT NULL default '{}' | Atributos **assertáveis** (grupos, escopos, unidade) — insumo do PDP (`022`). |
| `status` | `text` | NOT NULL, CHECK ∈ {active, suspended, disabled} | Situação do princípio. |
| `mfa_required` | `boolean` | NOT NULL default false | Exige segundo fator (aplicável a `human`). |
| `last_authn_at` | `timestamptz` | NULL | Última autenticação bem-sucedida. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, kind, external_ref)` (federação);
`btree(tenant_id, status)`; `btree(tenant_id, last_authn_at)`.

### 3.2 Entidade `Credential` — tabela `security.credential` (multi-tenant, RLS)

Modelo **unificado** para toda credencial emitida pelo módulo — token, certificado,
segredo dinâmico, conta NKey. É a entidade governada pela FSM da §4.

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:credential:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `principal_urn` | `text` | NOT NULL, FK→security.principal.urn | Dono da credencial. |
| `kind` | `text` | NOT NULL, CHECK ∈ {access_token, refresh_token, workload_cert, db_role, nats_account, object_key, provider_key, signing_key} | Tipo. |
| `state` | `text` | NOT NULL, CHECK ∈ CredentialState | Estado da FSM (§4). |
| `purpose` | `text` | NOT NULL | Propósito declarado (ex.: `postgres:memory_rw`). |
| `material_ref` | `text` | NULL | Ponteiro para o material cifrado (`security.secret`) ou identificador no HSM. **Nunca o material em si.** |
| `fingerprint` | `text` | NOT NULL | `sha256` do material público (certificado) ou do `jti` — permite correlacionar sem expor. |
| `issued_at` | `timestamptz` | NOT NULL | Emissão. |
| `not_before` | `timestamptz` | NOT NULL | Início de validade. |
| `expires_at` | `timestamptz` | NOT NULL | Fim de validade (**sempre preenchido**: não há credencial eterna). |
| `rotated_to` | `text` | NULL, FK→security.credential.urn | Sucessora, durante a rotação. |
| `revoked_at` | `timestamptz` | NULL | Momento da revogação. |
| `revocation_reason` | `text` | NULL | `compromise`, `superseded`, `principal_disabled`, `policy`, `operator`. |
| `issued_by` | `text` | NOT NULL | Identidade do emissor (serviço/operador). |
| `idempotency_key` | `text` | NULL, UNIQUE(tenant_id,idempotency_key) | RFC-0001 §5.5. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `btree(tenant_id, principal_urn, state)`;
`btree(tenant_id, expires_at)` (varredura de expiração); `UNIQUE(fingerprint)`;
`btree(kind, state)`.

> **`expires_at` é NOT NULL por desenho.** Se o esquema permitisse credencial sem
> expiração, a exceção viraria regra na primeira urgência operacional.

### 3.3 Entidade `Secret` — tabela `security.secret` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:secret:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `path` | `text` | NOT NULL, UNIQUE(tenant_id,path,version_no) | Caminho lógico (ex.: `db/memory/rw`). |
| `version_no` | `int` | NOT NULL | Versão do segredo (imutável após escrita). |
| `ciphertext` | `bytea` | NOT NULL | Material cifrado por envelope (DEK). |
| `dek_ref` | `text` | NOT NULL, FK→security.key.urn | DEK usada na cifragem. |
| `lease_ttl` | `interval` | NOT NULL default '1 hour' | Duração da concessão de leitura. |
| `rotation_period` | `interval` | NULL | Periodicidade de rotação automática. |
| `last_rotated_at` | `timestamptz` | NULL | Última rotação. |
| `created_at` | `timestamptz` | NOT NULL | Criação da versão. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, path, version_no)`;
`btree(tenant_id, path, version_no DESC)` (leitura da versão corrente).

### 3.4 Entidade `KeyMaterial` — tabela `security.key`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:key:<ULID>`. |
| `role` | `text` | NOT NULL, CHECK ∈ {kek, dek, signing, ca_root, ca_intermediate} | Papel da chave. |
| `algorithm` | `text` | NOT NULL | Suíte (ex.: `ES256`, `AES-256-GCM`, `Ed25519`). |
| `state` | `text` | NOT NULL, CHECK ∈ {pending, active, rotating, retired, destroyed} | Situação. |
| `hsm_backed` | `boolean` | NOT NULL default false | Material reside em HSM. |
| `public_material` | `bytea` | NULL | Parte pública (chaves assimétricas). |
| `wrapped_material` | `bytea` | NULL | Material privado **cifrado pela KEK** (NULL se `hsm_backed`). |
| `parent_key` | `text` | NULL, FK→security.key.urn | KEK que envolve esta chave. |
| `activated_at` | `timestamptz` | NULL | Início de uso. |
| `retire_after` | `timestamptz` | NULL | Fim de uso para **assinar** (segue válida para **verificar**). |
| `destroyed_at` | `timestamptz` | NULL | Destruição segura. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `btree(role, state)`; `btree(retire_after)`.

### 3.5 Entidade `SandboxProfile` — tabela `security.sandbox_profile`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:sandboxprofile:<ULID>`. |
| `name` | `text` | NOT NULL | Nome (ex.: `agent-default`, `agent-network-restricted`). |
| `profile_version` | `int` | NOT NULL, UNIQUE(name,profile_version) | Versão imutável. |
| `seccomp` | `jsonb` | NOT NULL | Lista de syscalls permitidas/negadas. |
| `cgroups` | `jsonb` | NOT NULL | Limites de CPU, memória, PIDs, I/O. |
| `namespaces` | `text[]` | NOT NULL | Namespaces exigidos (pid, net, mnt, ipc, uts, user). |
| `egress_allowlist` | `text[]` | NOT NULL default '{}' | Destinos de rede permitidos (a **decisão** de permitir é do `022`). |
| `signature` | `bytea` | NOT NULL | Assinatura do perfil (chave `signing`). |
| `signed_by` | `text` | NOT NULL, FK→security.key.urn | Chave que assinou. |
| `status` | `text` | NOT NULL, CHECK ∈ {draft, published, deprecated} | Situação. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(urn)`; `UNIQUE(name, profile_version)`; `btree(status)`.

### 3.6 Entidade `RevocationEntry` — tabela `security.revocation` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | PK (ULID) | Identidade da revogação. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `credential_urn` | `text` | NOT NULL, FK→security.credential.urn | Credencial revogada. |
| `fingerprint` | `text` | NOT NULL | Para consulta rápida pelos verificadores. |
| `reason` | `text` | NOT NULL | Motivo (§3.2). |
| `effective_at` | `timestamptz` | NOT NULL | Início do efeito. |
| `expires_at` | `timestamptz` | NOT NULL | Quando a entrada pode sair da lista (= expiração natural da credencial). |
| `propagated_at` | `timestamptz` | NULL | Confirmação de propagação por evento. |

Índices: `PK(id)`; `UNIQUE(fingerprint)`; `btree(tenant_id, effective_at DESC)`;
`btree(expires_at)`.

> A entrada de revogação **só pode ser removida após a expiração natural da
> credencial**: antes disso, esquecer a revogação equivaleria a desfazê-la.

### 3.7 Entidade `FederationTrust` — tabela `security.federation_trust` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:federationtrust:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `kind` | `text` | NOT NULL, CHECK ∈ {idp, a2a_peer} | IdP corporativo ou par A2A externo. |
| `issuer` | `text` | NOT NULL, UNIQUE(tenant_id,issuer) | *Issuer* confiável. |
| `jwks_uri` | `text` | NULL | Endpoint de chaves do par. |
| `trust_anchor` | `bytea` | NULL | Certificado âncora (mTLS de par externo). |
| `allowed_scopes` | `text[]` | NOT NULL default '{}' | Escopos que a identidade externa **pode** receber internamente. |
| `status` | `text` | NOT NULL, CHECK ∈ {active, suspended, revoked} | Situação. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, issuer)`; `btree(tenant_id, kind, status)`.

### 3.8 Relações (ASCII)

```
   security.principal(1) ───< security.credential(*) ───1 security.revocation(0..1)
        │                            │
        │                            ├──[material_ref]──▶ security.secret(*)
        │                            └──[rotated_to]────▶ security.credential (sucessora)
        │
        ├──< security.federation_trust(*)      (identidade externa → princípio interno)
        │
   security.key(1) ──[parent_key]──< security.key(*)      (KEK envolve DEK/signing)
        │                                   │
        ├──[dek_ref]────────────────────────┴──▶ security.secret
        └──[signed_by]──────────────────────────▶ security.sandbox_profile

   security.sandbox_profile ──consumido por──▶ 007-Agent-Runtime (SandboxManager)
```

---

## 4. Máquina de Estados Canônica da `Credential`

> A entidade com ciclo de vida governado é a **credencial** — e o modelo é
> deliberadamente **unificado**: token, certificado de workload, *role* de banco e
> conta NATS percorrem os mesmos estados. Isso permite uma única política de rotação,
> uma única trilha e um único caminho de revogação, em vez de quatro subsistemas com
> comportamentos sutilmente diferentes.
>
> `security.key` tem ciclo próprio, mais simples (§4.4).

### 4.1 Estados (`CredentialState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Requested` | Emissão solicitada; atestação/autorização em curso. | não |
| `Issued` | Material gerado e entregue ao solicitante; ainda antes de `not_before`. | não |
| `Active` | Válida e utilizável (`not_before` ≤ agora < `expires_at`). | não |
| `Rotating` | Sucessora emitida; ambas válidas durante a janela de sobreposição. | não |
| `Superseded` | Substituída pela sucessora; não deve mais ser usada. | **sim** |
| `Revoked` | Invalidada antes do prazo; consta na lista de revogação. | **sim** |
| `Expired` | Prazo natural encerrado. | **sim** |
| `Failed` | Emissão não concluída (atestação falhou, PDP negou, erro de material). | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Requested` | `IssueCredential` | Capability `sec:credential:issue` ∧ princípio em `status = active` ∧ propósito declarado ∧ TTL solicitado ≤ TTL máximo do tipo. |
| T-02 | `Requested` → `Issued` | atestação e autorização concluídas | `WorkloadAttestor` confirmou a identidade do solicitante ∧ PDP retornou `allow` ∧ material gerado e cifrado. |
| T-03 | `Requested` → `Failed` | atestação/autorização negada ou erro | Atestação falhou ∨ PDP negou ∨ falha do KMS/HSM. |
| T-04 | `Issued` → `Active` | `not_before` alcançado | Relógio ≥ `not_before` (permite emissão antecipada em rotação programada). |
| T-05 | `Active` → `Rotating` | `RotateCredential` (programada ou sob demanda) | Sucessora em `Issued`/`Active` ∧ janela de sobreposição configurada > 0. |
| T-06 | `Rotating` → `Superseded` | fim da janela de sobreposição | Sucessora em `Active` ∧ decorrida a janela ∧ nenhum uso recente da antiga registrado. |
| T-07 | `Active`/`Rotating`/`Issued` → `Revoked` | `RevokeCredential` | Capability `sec:credential:revoke` ∨ princípio → `disabled` ∨ suspeita de comprometimento. Entrada gravada em `security.revocation`. |
| T-08 | `Active`/`Rotating`/`Issued` → `Expired` | `expires_at` alcançado | Relógio ≥ `expires_at`. |
| T-09 | `Superseded` → `Revoked` | revogação retroativa | Comprometimento identificado após a substituição (a antiga ainda pode estar em cache de terceiros). |

**Invariantes:**
(I1) Toda credencial em estado não-terminal tem `expires_at` no futuro; não existe
credencial sem expiração.
(I2) Em `Rotating`, **exatamente duas** credenciais do mesmo `purpose` e
`principal_urn` estão utilizáveis — a antiga e sua `rotated_to`. Três é defeito.
(I3) `Revoked` **NÃO DEVE** transitar para nenhum outro estado; a revogação é
irreversível — recuperar acesso exige **nova** emissão.
(I4) Toda transição para `Revoked` grava entrada em `security.revocation` e emite
`aios.<tenant>.security.token.revoked` via Outbox, na mesma transação.
(I5) O material privado **nunca** é relido após a emissão: o que se guarda é
referência cifrada e `fingerprint`. Reemissão substitui, não recupera.
(I6) Transição só ocorre com o `version` esperado (OCC); conflito → `AIOS-SEC-0009`.

### 4.3 Diagrama de estados (ASCII)

```
        IssueCredential (T-01)
   ∅ ─────────────────────────▶ ┌───────────┐
                                 │ Requested │
                                 └─────┬─────┘
          atestação + PDP ok (T-02)    │    negado/erro (T-03)
        ┌───────────────────────────── ┤────────────────────────┐
        ▼                               │                        ▼
   ┌──────────┐                         │                  ┌──────────┐
   │  Issued  │                         │                  │  Failed  │ (terminal)
   └────┬─────┘                         │                  └──────────┘
        │ not_before (T-04)             │
        ▼                                │
   ┌──────────┐   RotateCredential (T-05)│      ┌─────────────┐
   │  Active  │─────────────────────────────────▶│  Rotating   │
   └────┬─────┘                                  └──────┬──────┘
        │                                                │ fim da janela (T-06)
        │                                                ▼
        │                                        ┌──────────────┐
        │                                        │  Superseded  │ (terminal)
        │                                        └──────┬───────┘
        │  revogação (T-07)                             │ revogação retroativa (T-09)
        ├───────────────────────────┐                   │
        │  expiração (T-08)         ▼                   ▼
        │                    ┌────────────┐      ┌────────────┐
        └───────────────────▶│  Expired   │      │  Revoked   │ (terminal, irreversível)
                             │ (terminal) │      └────────────┘
                             └────────────┘
```

### 4.4 Ciclo de `KeyMaterial`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `pending` | Chave gerada, ainda não em uso. | `active` |
| `active` | Em uso para assinar/cifrar. | `rotating` |
| `rotating` | Sucessora ativa; esta ainda **verifica/decifra**, mas não assina/cifra. | `retired` |
| `retired` | Não assina nem cifra; permanece para **verificar** assinaturas antigas. | `destroyed` |
| `destroyed` | Material destruído com segurança. | — (terminal) |

```
   pending ──ativa──▶ active ──rotaciona──▶ rotating ──retire_after──▶ retired
                                                                          │
                                    (só após nenhum artefato depender dela)
                                                                          ▼
                                                                     destroyed
```

Uma chave de assinatura **NÃO DEVE** ser destruída enquanto existir artefato válido
assinado por ela — do contrário, tokens e perfis legítimos tornam-se inverificáveis.
Por isso `retired` é um estado longo, e `destroyed` é decisão deliberada.

---

## 5. Superfície de API (REST e gRPC)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.security.v1`. Base REST: `/v1/security`, além dos **endpoints OIDC
> padronizados** (que seguem as especificações OAuth2/OIDC, não a convenção interna).

### 5.1 Endpoints OIDC (padrão externo)

| Endpoint | Descrição |
|----------|-----------|
| `GET /.well-known/openid-configuration` | *Discovery* do emissor. |
| `GET /.well-known/jwks.json` | **JWKS** consumido pelo `../004-API/` para validar tokens. |
| `POST /oauth2/token` | `authorization_code` (PKCE) e `client_credentials`. |
| `POST /oauth2/introspect` | Introspecção (RFC 7662). |
| `POST /oauth2/revoke` | Revogação de token (RFC 7009). |
| `GET /oauth2/authorize` | Fluxo interativo (humanos). |

### 5.2 Operações administrativas

| Operação | REST | gRPC (rpc) | Idempotente | Capability |
|----------|------|-----------|-------------|------------|
| Criar princípio | `POST /v1/security/principals` | `CreatePrincipal` | sim (key) | `sec:principal:write` |
| Ler/listar princípios | `GET /v1/security/principals[/{urn}]` | `GetPrincipal`/`ListPrincipals` | sim (leitura) | `sec:principal:read` |
| Atualizar atributos | `PATCH /v1/security/principals/{urn}` | `UpdatePrincipal` | sim | `sec:principal:write` |
| Desabilitar princípio | `POST /v1/security/principals/{urn}:disable` | `DisablePrincipal` | sim | `sec:principal:disable` |
| Emitir credencial | `POST /v1/security/credentials` | `IssueCredential` | sim (key) | `sec:credential:issue` |
| Ler credencial (metadados) | `GET /v1/security/credentials/{urn}` | `GetCredential` | sim (leitura) | `sec:credential:read` |
| Rotacionar credencial | `POST /v1/security/credentials/{urn}:rotate` | `RotateCredential` | sim (key) | `sec:credential:rotate` |
| Revogar credencial | `POST /v1/security/credentials/{urn}:revoke` | `RevokeCredential` | sim | `sec:credential:revoke` |
| Consultar revogação | `GET /v1/security/revocations?fingerprint=` | `CheckRevocation` | sim (leitura) | `sec:revocation:read` |
| Escrever segredo | `PUT /v1/security/secrets/{path}` | `PutSecret` | sim | `sec:secret:write` |
| Ler segredo (com lease) | `POST /v1/security/secrets/{path}:read` | `ReadSecret` | sim (leitura) | `sec:secret:read` |
| Emitir certificado de workload | `POST /v1/security/certificates` | `IssueWorkloadCert` | sim (key) | `sec:cert:issue` |
| Criar/rotacionar chave | `POST /v1/security/keys[/{urn}:rotate]` | `CreateKey`/`RotateKey` | sim | `sec:key:manage` |
| Publicar perfil de sandbox | `POST /v1/security/sandbox-profiles` | `PublishSandboxProfile` | sim | `sec:sandbox:publish` |
| Ler perfil de sandbox | `GET /v1/security/sandbox-profiles/{name}/{version}` | `GetSandboxProfile` | sim (leitura) | `sec:sandbox:read` |
| Registrar confiança federada | `PUT /v1/security/federation/{issuer}` | `PutFederationTrust` | sim | `sec:federation:write` |
| Assinar/verificar payload | `POST /v1/security/crypto:sign` / `:verify` | `Sign`/`Verify` | sim | `sec:crypto:use` |

### 5.3 Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínios reservados: **`SEC`** (0001–0099) e **`AUTHN`**
> (0001–0099), registrados no catálogo mantido por `../004-API/` (RFC-0001 §8).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-SEC-0001` | 404 | não | Princípio, credencial, segredo ou perfil não encontrado. |
| `AIOS-SEC-0002` | 403 | não | Operação negada pelo PDP (*default deny*). |
| `AIOS-SEC-0003` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-SEC-0004` | 422 | não | TTL solicitado acima do máximo permitido para o tipo de credencial. |
| `AIOS-SEC-0005` | 409 | não | Transição de estado inválida (viola a FSM §4). |
| `AIOS-SEC-0006` | 403 | não | Atestação de workload falhou: solicitante não comprovou a identidade. |
| `AIOS-SEC-0007` | 410 | não | Credencial revogada (irreversível — emita uma nova). |
| `AIOS-SEC-0008` | 423 | não | Princípio suspenso ou desabilitado. |
| `AIOS-SEC-0009` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-SEC-0010` | 503 | sim | KMS/HSM indisponível: emissão suspensa. |
| `AIOS-SEC-0011` | 422 | não | Suíte criptográfica não permitida. |
| `AIOS-SEC-0012` | 409 | não | Perfil de sandbox imutável: publique uma nova versão. |
| `AIOS-SEC-0013` | 429 | sim | Limite de emissões por princípio excedido. |
| `AIOS-SEC-0014` | 412 | não | Segredo lido fora do *lease* válido. |
| `AIOS-AUTHN-0001` | 401 | não | Credenciais inválidas. |
| `AIOS-AUTHN-0002` | 401 | não | Token expirado. |
| `AIOS-AUTHN-0003` | 401 | não | Assinatura de token inválida ou `issuer` desconhecido. |
| `AIOS-AUTHN-0004` | 401 | não | Segundo fator exigido e não fornecido. |
| `AIOS-AUTHN-0005` | 429 | sim | Excesso de tentativas de autenticação (contenção de força bruta). |
| `AIOS-AUTHN-0006` | 401 | não | Identidade federada não confiável (`issuer` não registrado ou suspenso). |
| `AIOS-AUTHN-0007` | 403 | não | Escopo solicitado fora do permitido para a identidade federada. |

> **Nota deliberada sobre mensagens de erro.** `AIOS-AUTHN-0001` **NÃO DEVE**
> distinguir "princípio inexistente" de "senha incorreta": a distinção é um oráculo de
> enumeração de contas. O `detail` do envelope de erro é genérico; o motivo real vai
> **apenas** para a auditoria e para o `ThreatSignalDetector`.

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3. `<dominio>` =
> **`security`**, valor a ser registrado no registro de domínios mantido por
> `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0210**) — subjects deste
> domínio já são consumidos por `004`, `010` e `020`. Entrega at-least-once; dedupe
> por `event.id`.
>
> **Regra absoluta:** nenhum evento deste módulo transporta material secreto. Eventos
> carregam URN, `fingerprint`, motivo e horário — nunca token, chave ou segredo.

### 6.1 Eventos emitidos (produtor: 021-Security)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios._platform.security.jwks.rotated` | `aios.security.jwks.rotated` | Chave de assinatura rotacionada; JWKS atualizado. | `SEC_PLATFORM` |
| `aios._platform.security.ca.rotated` | `aios.security.ca.rotated` | CA intermediária rotacionada. | `SEC_PLATFORM` |
| `aios._platform.security.sandboxprofile.published` | `aios.security.sandboxprofile.published` | Novo perfil de isolamento publicado. | `SEC_PLATFORM` |
| `aios.<tenant>.security.credential.issued` | `aios.security.credential.issued` | Credencial → `Active` (T-04). | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.credential.rotating` | `aios.security.credential.rotating` | Rotação iniciada (T-05) — sinal para o consumidor adotar a nova. | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.token.revoked` | `aios.security.token.revoked` | Credencial → `Revoked` (T-07/T-09). | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.credential.expiring` | `aios.security.credential.expiring` | Credencial próxima da expiração sem rotação. | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.principal.disabled` | `aios.security.principal.disabled` | Princípio desabilitado; credenciais em cascata. | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.authn.anomaly` | `aios.security.authn.anomaly` | Sinal de anomalia de autenticação (sem decisão de bloqueio). | `SEC_THREAT` |
| `aios.<tenant>.security.attestation.failed` | `aios.security.attestation.failed` | Workload não comprovou identidade (T-03). | `SEC_THREAT` |
| `aios.<tenant>.security.rtbf.requested` | `aios.security.rtbf.requested` | Pedido de direito ao esquecimento iniciado (consumido por `010`/`005`/`025`). | `SEC_PRIVACY` |
| `aios.<tenant>.security.federation.suspended` | `aios.security.federation.suspended` | Confiança federada suspensa/revogada. | `SEC_PLATFORM` |

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do módulo |
|-------------------|----------|----------------|
| `aios.<tenant>.policy.bundle.updated` | `022-Policy` | Invalida cache do `PolicyClient`; reavalia emissões pendentes. |
| `aios.<tenant>.agent.lifecycle.terminated` | `006-Kernel` | Revoga credenciais emitidas ao agente extinto. |
| `aios._platform.cluster.node.decommissioned` | `027-Cluster` | Revoga certificados de workload do nó removido. |
| `aios.<tenant>.audit.anomaly.detected` | `025-Audit` | Correlaciona com sinais próprios do `ThreatSignalDetector`. |
| `aios._platform.deployment.rollout.started` | `028-Deployment` | Suspende rotações de credencial durante a janela de rollout. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Operar emissor OIDC com `authorization_code` (PKCE) e `client_credentials`. | Must | Fluxos exercitados por teste de conformidade OIDC; `discovery` e `introspect` disponíveis. |
| FR-002 | Publicar JWKS com sobreposição de chaves durante a rotação. | Must | Tokens assinados pela chave antiga permanecem válidos até expirar; `jwks.rotated` emitido. |
| FR-003 | Emitir certificados de workload de vida curta para mTLS, com atestação prévia. | Must | Emissão sem atestação → `AIOS-SEC-0006`; TTL default ≤ 24 h. |
| FR-004 | Emitir credenciais dinâmicas para PostgreSQL, NATS, MinIO e provedores de LLM. | Must | Credencial válida no recurso alvo e expirando no TTL declarado. |
| FR-005 | Rotacionar credenciais com janela de sobreposição, sem downtime do consumidor. | Must | Durante `Rotating`, ambas funcionam; após a janela, só a nova (invariante I2). |
| FR-006 | Revogar credencial em qualquer estado não-terminal e propagar o efeito. | Must | Entrada em `security.revocation` + evento `token.revoked` na mesma transação (I4). |
| FR-007 | Recusar emissão de credencial sem expiração ou com TTL acima do máximo do tipo. | Must | `expires_at` sempre preenchido; TTL excessivo → `AIOS-SEC-0004`. |
| FR-008 | Custodiar segredos versionados com cifragem envelopada e leitura sob *lease*. | Must | Nenhum segredo em claro no banco; leitura fora do lease → `AIOS-SEC-0014`. |
| FR-009 | Operar KMS com hierarquia KEK/DEK e rotação programada. | Must | Rotação de KEK não exige recifrar todos os dados (apenas as DEK). |
| FR-010 | Publicar perfis de isolamento versionados e **assinados**. | Must | Perfil publicado é imutável (`AIOS-SEC-0012`); assinatura verificável pelo `007`. |
| FR-011 | Integrar IdP externo por tenant e mapear identidade externa a princípio interno com escopo reduzido. | Should | Escopo fora do permitido → `AIOS-AUTHN-0007`; `issuer` desconhecido → `AIOS-AUTHN-0006`. |
| FR-012 | Manter âncoras de confiança para pares A2A externos do `020`. | Should | Suspensão de confiança encerra sessões A2A correspondentes. |
| FR-013 | Detectar e sinalizar anomalias de autenticação sem decidir bloqueio. | Should | Rajada de falhas → `AIOS-AUTHN-0005` + evento `authn.anomaly`. |
| FR-014 | Desabilitar princípio e revogar em cascata suas credenciais. | Must | Todas as credenciais do princípio → `Revoked`; evento `principal.disabled`. |
| FR-015 | Prover serviços criptográficos com governança de suítes. | Should | Algoritmo fora da lista → `AIOS-SEC-0011`. |
| FR-016 | Consultar o PDP antes de toda operação administrativa. | Must | Sem capability → `AIOS-SEC-0002`; decisão auditada. |
| FR-017 | Emitir todos os eventos de §6 via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. |
| FR-018 | Não expor material secreto em API de leitura, log, evento ou métrica. | Must | Auditoria automatizada de resposta/log não encontra material sensível. |
| FR-019 | Alertar sobre credenciais próximas da expiração sem rotação programada. | Should | Evento `credential.expiring` em ≤ 25% do TTL restante. |
| FR-020 | Suportar o gatilho de direito ao esquecimento, emitindo `security.rtbf.requested`. | Must | Evento consumido por `010`/`005`; comprovante correlacionado em `025`. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (validação) | p99 de **introspecção** de token ≤ **10 ms**; validação local por JWKS no `004` não consulta o `021`. | `aios_sec_introspect_latency_ms`. |
| NFR-002 | Desempenho (emissão) | p99 de emissão de token ≤ **50 ms**; de certificado de workload ≤ **150 ms**. | `aios_sec_issue_latency_ms{kind}`. |
| NFR-003 | Desempenho (segredo) | p99 de leitura de segredo com lease ≤ **20 ms**. | `aios_sec_secret_read_latency_ms`. |
| NFR-004 | Throughput | ≥ **5.000 emissões/s** e ≥ **20.000 introspecções/s** por réplica. | Benchmark `Benchmark.md`. |
| NFR-005 | Disponibilidade | ≥ **99,99%** (é dependência de bootstrap de todo o sistema). | Uptime probe; *error budget* mensal. |
| NFR-006 | Propagação de revogação | Revogação efetiva em **≤ 30 s** em todos os verificadores. | Tempo entre `token.revoked` e recusa observada. |
| NFR-007 | Escalabilidade | ≥ **10⁶** princípios e ≥ **10⁷** credenciais ativas com custo sub-linear. | Teste de escala `Scalability.md`. |
| NFR-008 | Confidencialidade | **0** ocorrências de material secreto em log, evento, métrica ou resposta de API. | Varredura automatizada contínua (FR-018). |
| NFR-009 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min**; CA raiz recuperável de custódia *offline*. | DR drill; cerimônia de recuperação de raiz. |
| NFR-010 | Vida curta de credencial | TTL mediano de credencial ativa ≤ **24 h**; **0** credenciais sem expiração. | `aios_sec_credential_ttl_hours` (histograma). |
| NFR-011 | Rotação | **100%** das chaves com rotação dentro do período configurado. | `aios_sec_key_age_days` vs. política. |
| NFR-012 | Observabilidade | **100%** das operações com trace OTel e auditoria correlacionada — **sem** material sensível. | Cobertura de spans; varredura de conteúdo. |
| NFR-013 | Idempotência | Repetições de mutação produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key`. |
| NFR-014 | Resistência a força bruta | Contenção ativa em ≤ **5** tentativas falhas por princípio por minuto. | `aios_sec_authn_failures_total`; teste de contenção. |
| NFR-015 | Integridade de perfil | **100%** dos perfis de sandbox consumidos com assinatura verificada pelo `007`. | Teste de contrato com `007-Agent-Runtime`. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global`, `tenant`, `principal`, `credential_kind`. Recarregável indica *hot
> reload*. Prefixo de env var: **`AIOS_SEC_`**.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `sec.token.access_ttl_s` | 900 | 60–3600 | global/tenant | sim | TTL do *access token* (15 min). |
| `sec.token.refresh_ttl_s` | 86400 | 3600–2592000 | global/tenant | sim | TTL do *refresh token* (24 h). |
| `sec.token.max_ttl_s` | 3600 | 60–86400 | global | sim | Teto absoluto para qualquer token (`AIOS-SEC-0004`). |
| `sec.cert.workload_ttl_s` | 86400 | 3600–604800 | global | sim | TTL de certificado de workload (24 h). |
| `sec.cert.renew_before_ratio` | 0.5 | 0.1–0.9 | global | sim | Fração do TTL a partir da qual a renovação é disparada. |
| `sec.credential.rotation_overlap_s` | 300 | 0–86400 | credential_kind | sim | Janela de sobreposição em `Rotating` (invariante I2). |
| `sec.credential.db_role_ttl_s` | 3600 | 300–86400 | credential_kind | sim | TTL de credencial dinâmica de PostgreSQL. |
| `sec.credential.nats_account_ttl_s` | 604800 | 3600–2592000 | credential_kind | sim | TTL de conta NKey/JWT do NATS. |
| `sec.credential.max_active_per_principal` | 32 | 1–1024 | principal | sim | Limite de credenciais ativas (`AIOS-SEC-0013`). |
| `sec.secret.default_lease_ttl_s` | 3600 | 60–86400 | tenant | sim | Duração default do *lease* de leitura. |
| `sec.secret.rotation_period_days` | 90 | 1–365 | tenant | sim | Periodicidade default de rotação de segredo. |
| `sec.key.signing_rotation_days` | 90 | 7–730 | global | sim | Rotação da chave de assinatura de token. |
| `sec.key.kek_rotation_days` | 365 | 30–1825 | global | **não** | Rotação da KEK (exige cerimônia). |
| `sec.key.retire_grace_days` | 30 | 1–365 | global | sim | Permanência em `retired` antes de `destroyed`. |
| `sec.key.hsm_enabled` | `false` | {true,false} | global | **não** | Usa HSM para KEK e CA raiz. |
| `sec.key.allowed_suites` | `["ES256","EdDSA","AES-256-GCM"]` | lista | global | sim | Suítes permitidas (`AIOS-SEC-0011`). |
| `sec.authn.max_failures_per_min` | 5 | 1–100 | principal | sim | Limiar de contenção de força bruta (`AIOS-AUTHN-0005`). |
| `sec.authn.lockout_s` | 300 | 30–86400 | principal | sim | Duração da contenção após o limiar. |
| `sec.authn.mfa_required_for_human` | `true` | {true,false} | tenant | sim | Exige segundo fator para princípios humanos. |
| `sec.revocation.propagation_target_s` | 30 | 1–300 | global | sim | Meta de propagação de revogação (NFR-006). |
| `sec.revocation.list_cache_ttl_s` | 10 | 1–60 | global | sim | TTL do cache da lista de revogação nos verificadores. |
| `sec.attestation.required` | `true` | {true,false} | global | **não** | Exige atestação antes de emitir credencial de workload. |
| `sec.federation.max_scope_expansion` | 0 | 0–5 | tenant | sim | Escopos adicionais que uma identidade federada pode ganhar. **0 = nenhum.** |
| `sec.policy.fail_mode` | `closed` | {closed,open} | global | sim | Comportamento se o PDP estiver indisponível. |
| `sec.sandbox.default_profile` | `agent-default` | nome | tenant | sim | Perfil aplicado quando o agente não especifica outro. |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| KMS/HSM indisponível | Health do `KeyManagementService` | Emissão suspensa (`AIOS-SEC-0010`); **validação de token existente continua** | Restaurar KMS; retomar fila de emissão | Emissão idempotente por `Idempotency-Key` |
| Perda da chave de assinatura corrente | Falha ao assinar | Sobreposição: chave anterior ainda verifica | Promover chave `pending`; publicar JWKS; emitir `jwks.rotated` | Tokens antigos seguem válidos até expirar |
| Comprometimento de credencial | Sinal do `ThreatSignalDetector` ou reporte | Revogação imediata (T-07) | Emissão de nova credencial; rotação forçada dos pares afetados | Revogação idempotente |
| Comprometimento de CA intermediária | Auditoria/alerta | Revogação da intermediária; emissão suspensa naquele ramo | Nova intermediária a partir da raiz *offline*; reemissão em massa de certificados de workload | Reemissão idempotente por workload |
| Comprometimento da CA **raiz** | Detecção fora de banda | Todo o mTLS interno é afetado | Cerimônia de recuperação com a custódia *offline*; nova raiz; *cross-signing* durante a transição | Procedimento manual documentado em `../029-Operations/` |
| PDP (022) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do PEP | `fail_mode=closed`: operações administrativas negadas; **autenticação e validação continuam** | Retry após meia-abertura do CB |
| PostgreSQL (`security`) indisponível | Health | Emissão e revogação param; **validação por JWKS no `004` continua** | Failover do `005`; reconciliação de estado | Idempotente |
| Falha de propagação de revogação | `revocation.propagated_at` nulo além da meta | Verificadores consultam a lista diretamente (fallback) | Reenvio pelo Outbox; alerta se exceder NFR-006 | Dedupe por `event.id` |
| Rajada de autenticação falha | `authn.failures` por princípio | Contenção por princípio, **não** global | Liberação após `lockout_s`; sinal a `025` | Determinístico |
| Atestação falhando em massa | `attestation.failed` | Emissão negada para os workloads afetados | Investigar mudança de topologia (`027`) antes de relaxar a exigência | Nenhum retry cego |
| Relógio dessincronizado | Validação de `not_before`/`expires_at` | Tolerância de *skew* configurada (60 s) | Corrigir NTP; **nunca** ampliar a tolerância como "correção" | — |
| Vazamento suspeito em log | Varredura contínua (FR-018) | Log quarentenado | Rotação das credenciais expostas; expurgo do log; incidente P1 | — |

**Metas de recuperação:** **RTO ≤ 15 min**, **RPO ≤ 5 min**. Degradação graciosa em
ordem: sacrifica-se primeiro **emissão** e **rotação**, depois **operações
administrativas**, preservando por último **autenticação** e **validação de credenciais
já emitidas** — porque um sistema que não emite credenciais novas se degrada; um sistema
que não valida as existentes **para completamente**.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Validação distribuída | Tokens são **JWT assinados**: o `004-API` valida **localmente** com o JWKS cacheado, sem chamar o `021` por requisição. Essa é a decisão que permite ao módulo não ser gargalo do caminho quente. |
| Custo da revogação | O preço da validação local é a revogação não ser instantânea. Mitiga-se com **TTL curto** (15 min) + lista de revogação com cache de 10 s, alcançando propagação ≤ 30 s (NFR-006). |
| Emissão | Serviço **stateless**; estado em PostgreSQL, material em KMS/HSM. Réplicas escalam por CPU (assinatura é o custo dominante). |
| Cache de chaves | Chaves de assinatura em memória com rotação por evento; DEK cacheadas por TTL curto, nunca persistidas em disco fora do envelope. |
| Concorrência de rotação | OCC por `version` + *advisory lock* por `purpose`+`principal`, garantindo a invariante I2 (nunca três credenciais utilizáveis). |
| Particionamento | `security.credential` particionada por `RANGE(expires_at)` — a varredura de expiração e o expurgo tornam-se O(1) por partição (`../005-Database/`). |
| Contenção de autenticação | Contadores por princípio em Redis; contenção é **por princípio**, nunca global — um ataque a uma conta não pode negar serviço a todas. |
| Federação | Validação de token externo usa JWKS do IdP em cache; falha do IdP externo afeta **apenas** aquele tenant. |
| Rumo a milhões | 10⁶ agentes ⇒ 10⁷ credenciais ativas. Sustentado por: validação local (sem chamada por requisição), TTL curto (o conjunto ativo é sempre pequeno em relação ao histórico), particionamento por expiração e emissão em lote no *boot* do agente. |

```
   ❌ validação centralizada           ✅ validação local + revogação curta
      cada requisição → 021               JWT assinado + JWKS cacheado
      (021 vira gargalo global)           021 só na EMISSÃO e na REVOGAÇÃO
```

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada: `ADR-0210`..`ADR-0219`** (regra 021×10). Registrar em
> `../002-ADR/`. **ADR-0008** (governança por política, *default deny*) é herdada e
> define a fronteira entre este módulo e o `022-Policy`.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0210 | Fronteira identidade × autorização (021 × 022) e registro do domínio de eventos `security`. |
| ADR-0211 | JWT assinado com validação local por JWKS vs. introspecção centralizada. |
| ADR-0212 | Modelo unificado de credencial: um ciclo de vida para token, certificado, *role* e conta. |
| ADR-0213 | Credenciais de vida curta com rotação por sobreposição; proibição de credencial sem expiração. |
| ADR-0214 | PKI interna com raiz *offline* e certificados de workload de TTL ≤ 24 h. |
| ADR-0215 | Atestação de workload obrigatória antes da emissão de credencial de serviço. |
| ADR-0216 | Hierarquia KEK/DEK com cifragem envelopada e caminho para HSM. |
| ADR-0217 | Perfis de sandbox como artefato assinado e versionado, publicado por `021` e aplicado por `007`. |
| ADR-0218 | Federação com escopo reduzido: confiança externa não se converte em confiança interna. |
| ADR-0219 | Domínios de erro (`SEC`, `AUTHN`), mensagens não-oráculo e política de *fail-closed* seletivo. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida) — §6 fixa mTLS interno e validação de token no Gateway. |
| RFC-0210 | AIOS Identity & Credential Contract (princípios, atributos assertáveis, ciclo de credencial, revogação). | A propor por este módulo. |
| RFC-0211 | AIOS Workload Identity & PKI Profile (atestação, formato do certificado de workload, hierarquia de CA). | A propor por este módulo. |

---

## 12. Decisões de Segurança

> Este módulo **é** o subsistema de segurança; esta seção trata da segurança **do
> próprio módulo** — a superfície mais crítica do AIOS, porque comprometê-la
> compromete todo o resto.

### 12.1 AuthN / AuthZ

- **AuthN do próprio módulo**: operadores autenticam-se por OIDC com **MFA
  obrigatório** (`sec.authn.mfa_required_for_human`); serviços, por **mTLS** com
  certificado de workload emitido após atestação.
- **AuthZ**: o 021 é **PEP** de si mesmo. Emitir, rotacionar, revogar, publicar perfil
  e gerenciar chave exigem capability avaliada pelo **PDP** do `022-Policy`, com
  ***default deny*** — inclusive para operadores humanos.
- **Separação de funções**: quem administra princípios **NÃO DEVERIA** ser quem
  administra chaves; operações sobre a CA raiz e a KEK exigem **aprovação dupla**.
- **Fail-closed seletivo**: com o PDP indisponível, operações administrativas são
  negadas, mas **autenticação e validação de credenciais existentes continuam** — a
  alternativa transformaria uma falha de política em queda total do AIOS.
- **Isolamento de tenant**: RLS por `tenant_id` em `principal`, `credential`, `secret`,
  `revocation` e `federation_trust`; `tenant` divergente → `AIOS-SEC-0003`.

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no 021-Security | Mitigação |
|-----------------|------------------------|-----------|
| **S**poofing | Workload assume a identidade de outro para obter credencial de banco ou conta NATS privilegiada. | **Atestação obrigatória** antes da emissão (`AIOS-SEC-0006`); certificado de workload com TTL ≤ 24 h; `purpose` declarado e verificado. |
| **T**ampering | Alteração de perfil de sandbox para afrouxar o isolamento; adulteração de token. | Perfis **assinados** e imutáveis por versão (`AIOS-SEC-0012`), verificados pelo `007` (NFR-015); JWT assinado; toda mutação sob OCC e auditada. |
| **R**epudiation | Negar ter emitido, usado ou revogado uma credencial. | `issued_by`, `fingerprint`, horários e motivo em `security.credential`/`revocation`; trilha imutável em `../025-Audit/`. |
| **I**nformation disclosure | Vazamento de segredo em log, evento, métrica ou resposta; enumeração de contas por mensagem de erro. | Nenhum material em claro no banco (envelope); FR-018 com varredura contínua; erros de AuthN **não-oráculo** (§5.3); métricas sem `principal` como label. |
| **D**enial of service | Força bruta contra um princípio; inundação de pedidos de emissão; ataque ao KMS. | Contenção **por princípio** (nunca global); `max_active_per_principal`; limites de emissão (`AIOS-SEC-0013`); validação local no `004` mantém o sistema operando mesmo sob pressão no `021`. |
| **E**levation of privilege | Identidade federada ganhando escopo interno; auto-emissão de credencial privilegiada; abuso da CA. | `max_scope_expansion = 0` por default (ADR-0218); toda emissão passa pelo PDP; CA raiz *offline* com aprovação dupla; separação de funções. |

### 12.3 LGPD / GDPR

- **Minimização**: o módulo guarda **identificadores e credenciais**, não conteúdo
  pessoal de negócio. `display_name` e `external_ref` são o mínimo necessário para
  operar a identidade.
- **Base legal e retenção**: `security.principal` é `pii` e declara base legal
  (execução de contrato/obrigação legal); credenciais expiradas são expurgadas
  conforme retenção, preservando apenas o metadado exigido para auditoria.
- **Direito ao esquecimento**: o módulo é o **gatilho** — emite
  `aios.<tenant>.security.rtbf.requested`, consumido por `010-Memory`, `005-Database` e
  `025-Audit`, e desabilita o princípio com revogação em cascata. O expurgo do dado de
  negócio é executado pelos módulos donos.
- **Trilha de acesso a segredo**: toda leitura de segredo é registrada (quem, quando,
  qual caminho, qual lease) — sem o valor lido.
- **Segregação**: RLS por tenant; chaves de tenant distintas sob KEK distintas quando o
  contrato exigir separação criptográfica.
- **Transferência internacional**: identidades federadas de IdP em outra jurisdição são
  registradas em `federation_trust` com escopo explícito, permitindo avaliação de
  adequação por tenant.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§6 segurança)
- Decisões: `../002-ADR/README.md` (ADR-0008 fronteira de governança)
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../004-API/`, `../005-Database/`, `../006-Kernel/`,
  `../007-Agent-Runtime/`, `../010-Memory/`, `../017-Model-Router/`,
  `../020-Communication/`, `../022-Policy/`, `../024-Observability/`,
  `../025-Audit/`, `../027-Cluster/`, `../028-Deployment/`, `../029-Operations/`.

*Fim do Design Brief interno do módulo 021-Security. Este documento governa a geração
dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e este
brief é um defeito do documento gerado.*
