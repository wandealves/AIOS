---
Documento: Architecture
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0001, ADR-0008, ADR-0210, ADR-0211, ADR-0212, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0217, ADR-0218
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: 001-Architecture, 005-Database, 020-Communication, 022-Policy, 024-Observability, 025-Audit, 027-Cluster, 028-Deployment
---

# 021-Security — Arquitetura do Módulo

Este documento deriva de `./_DESIGN_BRIEF.md` §0, §2, §3 e §10. Onde houver
divergência, o brief prevalece e este documento é o defeito.

---

## 1. Visão C4 — Nível 1: Contexto

```
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐
   │ 030-CLI      │  │ 032-WebConsole│  │ IdP externo  │  │ Par A2A externo│
   │ 031-SDK      │  │ (operadores) │  │ (do tenant)  │  │ (outra org)    │
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └────────┬───────┘
          │ OIDC            │ OIDC+MFA        │ federação         │ âncora mTLS
          └─────────────────┴────────┬────────┴───────────────────┘
                                     ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │                 021-SECURITY — Raiz de Confiança                          │
   │  OIDC · JWKS · PKI (mTLS) · KMS · Cofre · Perfis de sandbox · Revogação   │
   └───┬─────────┬──────────┬──────────┬──────────┬──────────┬────────────────┘
       │         │          │          │          │          │
       ▼         ▼          ▼          ▼          ▼          ▼
   004-API   005-Database 007-Runtime 017-Router 020-Comm  022-Policy
   (JWKS)    (roles)      (perfis,    (chaves    (NKey/JWT) (PDP: consome
                           segredos)   de LLM)              claims; decide)
                                                              │
                                                              ▼
                                                         025-Audit (prova)
```

**Leitura.** O 021 tem dois modos de interação, e distingui-los é essencial:

- **Emissão e revogação** (síncrono, baixo volume): serviços pedem credenciais; o 021
  atesta, autoriza no PDP, emite e registra.
- **Validação** (altíssimo volume): **não passa pelo 021**. O `004-API` valida JWT
  localmente com o JWKS cacheado; serviços validam mTLS com a cadeia da CA. Se cada
  requisição do AIOS consultasse este módulo, ele seria simultaneamente o gargalo e o
  ponto único de falha total do sistema (ADR-0211).

---

## 2. Visão C4 — Nível 2: Contêineres

| Contêiner | Tecnologia | Papel | Réplicas |
|-----------|-----------|-------|----------|
| `aios-security-svc` | .NET 10 | OIDC, emissão, revogação, cofre, perfis, API administrativa. **Stateless**. | 3 (é dependência de bootstrap) |
| `aios-security-ca` | .NET 10 (processo isolado) | CA intermediária online: emissão de certificados de workload. Isolado do resto por superfície e por privilégio. | 2 |
| CA **raiz** | *offline* | Assina apenas as intermediárias, em cerimônia. **Não é um contêiner** — vive em custódia física/HSM. | — |
| HSM / KMS externo | opcional | Guarda KEK e chave da CA raiz. | externo |
| PostgreSQL (`security`) | PG16 (via `005`) | Princípios, credenciais, segredos cifrados, chaves (metadados), perfis. | externo |
| Redis | Redis 7 (via `028`) | Contadores de contenção de autenticação; cache de lista de revogação. | externo |
| NATS | (via `020`) | Eventos `aios.*.security.*`. | externo |

```
   ┌──────────────────┐        ┌────────────────────┐
   │ aios-security-svc│───────▶│ aios-security-ca ×2│──▶ certificados de workload
   │  ×3 (stateless)  │  gRPC  │ (CA intermediária) │
   └───┬─────┬────────┘        └─────────┬──────────┘
       │     │                            │ assinada por
       │     │                            ▼
       │     │                    ┌───────────────┐
       │     │                    │  CA RAIZ      │  offline · custódia · HSM
       │     │                    │  (cerimônia)  │
       │     │                    └───────────────┘
       │     └──▶ HSM / KMS externo (KEK, chave da raiz)
       │
       ├──▶ PostgreSQL (schema security)   ├──▶ Redis (contenção, cache CRL)
       └──▶ NATS (aios.*.security.*)
```

A separação do `aios-security-ca` em processo próprio é deliberada: a superfície que
emite certificados não deveria compartilhar espaço de memória com a que serve
endpoints OIDC públicos.

---

## 3. Visão C4 — Nível 3: Componentes

Os componentes são normativos em `./_DESIGN_BRIEF.md` §2.1. Resumo com a fronteira:

| Componente | Responsabilidade sintética | Fronteira (o que NÃO faz) |
|------------|---------------------------|---------------------------|
| `IdentityProvider` | Fluxos OIDC, sessão, consentimento. | Não decide o que o sujeito pode fazer. |
| `TokenService` | Emite/renova/introspecta/revoga JWT. | Não valida em nome do `004` (validação é local lá). |
| `JwksPublisher` | Publica e rotaciona o conjunto de chaves públicas. | Não distribui chave privada. |
| `PrincipalStore` | Princípios e **atributos assertáveis**. | Não define o significado dos atributos (é do `022`). |
| `CertificateAuthority` | PKI: emissão e renovação de certificados. | Não decide topologia de rede. |
| `WorkloadAttestor` | Verifica a identidade do solicitante antes de emitir. | Não monitora o workload depois. |
| `SecretStore` | Segredos versionados, *lease*, cifragem envelopada. | Não interpreta o conteúdo do segredo. |
| `CredentialBroker` | Credenciais dinâmicas por recurso; rotação. | Não opera o recurso (banco, NATS). |
| `KeyManagementService` | KEK/DEK, envelope, rotação, HSM. | Não cifra dado de negócio por conta própria. |
| `RevocationService` | Revoga e propaga; mantém a lista. | Não decide **quando** revogar por comportamento. |
| `FederationConnector` | IdP externo e âncoras A2A. | Não converte confiança externa em interna. |
| `SandboxProfileRegistry` | Perfis assinados e versionados. | Não aplica o perfil (é do `007`). |
| `CryptoService` | Primitivas e governança de suítes. | Não implementa algoritmo próprio. |
| `ThreatSignalDetector` | Sinaliza anomalia de autenticação. | Não bloqueia — sinaliza. |
| `PolicyClient` | PEP das próprias operações. | Não decide (é PDP do `022`). |
| `EventEmitter` / `SecurityTelemetry` | Outbox → NATS; OTel + auditoria. | Não emite material sensível. |

Diagrama completo em `./_DESIGN_BRIEF.md` §2.2; interfaces em `./ClassDiagrams.md` §3.

---

## 4. Padrões Arquiteturais Adotados

| Padrão | Onde | Por quê | ADR |
|--------|------|---------|-----|
| **Token assinado com validação local** | OIDC + JWKS | Remove o 021 do caminho quente; validação O(1) sem rede. | ADR-0211 |
| **Ciclo de vida unificado de credencial** | `security.credential` | Uma política de rotação, uma trilha, um caminho de revogação — em vez de quatro subsistemas divergentes. | ADR-0212 |
| **Credenciais efêmeras com rotação por sobreposição** | `CredentialBroker` | Elimina credencial eterna sem exigir downtime na troca. | ADR-0213 |
| **CA raiz offline + intermediárias online** | PKI | Comprometer uma intermediária é recuperável; comprometer a raiz não deveria ser possível remotamente. | ADR-0214 |
| **Atestação antes da emissão** | `WorkloadAttestor` | Impede que um workload assuma a identidade de outro na rede interna. | ADR-0215 |
| **Cifragem envelopada (KEK/DEK)** | KMS | Rotacionar a KEK não exige recifrar todos os dados — apenas as DEK. | ADR-0216 |
| **Isolamento como artefato assinado** | `SandboxProfileRegistry` | Endurecimento deixa de depender de configuração dispersa e vira material verificável. | ADR-0217 |
| **Escopo reduzido na federação** | `FederationConnector` | Confiança externa não vira confiança interna. | ADR-0218 |
| **PEP → PDP com *fail-closed* seletivo** | `PolicyClient` | *Default deny* (RFC-0001 §5.8) sem derrubar a validação de credenciais existentes. | ADR-0219 |
| **Outbox transacional** | `EventEmitter` | Revogação e seu evento commitam juntos (invariante I4). | herdado de ADR-0006 |

---

## 5. Decisões Tecnológicas e Alternativas Descartadas

| Escolha | Alternativas consideradas | Por que a escolha | Trade-off aceito |
|---------|---------------------------|-------------------|------------------|
| **IdP próprio (OIDC)** no plano de controle | Keycloak/Auth0 como dependência obrigatória | O AIOS precisa emitir identidade para **workloads e agentes**, não só para humanos, com atestação e TTL de minutos. Federação com IdP externo cobre o caso corporativo (FR-011). | Custo de manter conformidade OIDC; mitigado por suíte de testes de conformidade. |
| **JWT assinado + validação local** | Token opaco + introspecção por requisição | Escala: 10⁶ agentes não podem gerar uma chamada ao 021 por requisição. | Revogação não é instantânea; mitigada por TTL de 15 min + lista com cache de 10 s (NFR-006). |
| **PKI interna própria** | Certificados de CA pública; malha de serviço com CA embutida | Certificados de workload com TTL de horas e atestação não são atendidos por CA pública; manter a CA no 021 evita acoplar a identidade à malha. | Operar PKI é responsabilidade séria — daí a raiz *offline* e a cerimônia documentada. |
| **Ciclo unificado de credencial** | Um subsistema por tipo (tokens, certs, segredos) | Uma FSM, uma trilha, uma política de rotação; reduz o risco de um tipo ficar sem revogação implementada. | Abstração precisa acomodar tipos heterogêneos — resolvido com `kind` + `purpose`. |
| **Cofre embutido** | HashiCorp Vault como dependência obrigatória | Reduz superfície operacional e evita mais um sistema no caminho de bootstrap; a interface HSM cobre o requisito de custódia forte. | Menos recursos que um cofre dedicado; `hsm_enabled` permite elevar a garantia. |
| **Sinalizar, não bloquear** (anomalias) | Bloqueio automático no próprio 021 | Bloqueio é decisão de política/operacional; embutir no 021 criaria um segundo PDP informal. | Resposta a incidente depende de `022`/`029` estarem operando. |

---

## 6. Fluxo Arquitetural Principal — Emissão de Credencial de Workload

```
 Serviço(010)  security-svc  WorkloadAttestor  PDP(022)  security-ca  KMS  PG  NATS
      │             │              │              │           │        │    │    │
      ├─ IssueCredential(purpose=postgres:memory_rw) ─────────────────────────────▶
      │             │ T-01 Requested              │           │        │    │    │
      │             ├─ atesta identidade ─────────▶           │        │    │    │
      │             │◀─ ok (nó, serviço, versão) ─┤           │        │    │    │
      │             ├─ Decide(sec:credential:issue) ─────────▶│        │    │    │
      │             │◀──────── allow ─────────────────────────┤        │    │    │
      │             ├─ gera material ────────────────────────────────▶ │    │    │
      │             │◀─ material + DEK ──────────────────────────────── ┤    │    │
      │             ├─ BEGIN; grava credential(Issued); secret cifrado; outbox; COMMIT ▶
      │             │ T-02 Issued → T-04 Active   │           │        │    │    │
      │◀─ credencial (material entregue UMA vez) ─┤           │        │    │    │
      │             │                                                       ╌╌╌╌▶│
      │             │            aios.<tenant>.security.credential.issued        │
```

O material é entregue **uma única vez** (invariante I5): o banco guarda apenas
referência cifrada e `fingerprint`. Perder a credencial não é recuperável — é
**reemitível**, o que é uma propriedade, não uma limitação.

Fluxos de falha (atestação reprovada, PDP negando, KMS fora) em
`./SequenceDiagrams.md` §5–§7.

---

## 7. Fronteiras de Confiança

```
   ┌─── Zona externa ────────────────────────────────────────────────────┐
   │ IdP do tenant · pares A2A externos · clientes                       │
   └──────────────────────────┬──────────────────────────────────────────┘
                              │ OIDC/SAML federado · mTLS com âncora registrada
                              │ ESCOPO REDUZIDO (max_scope_expansion = 0)
   ┌──────────────────────────▼──────────────────────────────────────────┐
   │ Zona de serviços (mTLS obrigatório, certificados de vida curta)     │
   │  aios-security-svc · demais módulos                                 │
   └──────────────────────────┬──────────────────────────────────────────┘
                              │ acesso a material sob PEP + atestação
   ┌──────────────────────────▼──────────────────────────────────────────┐
   │ Zona de material criptográfico                                      │
   │  aios-security-ca (intermediária) · KMS/HSM · segredos cifrados      │
   └──────────────────────────┬──────────────────────────────────────────┘
                              │ cerimônia, aprovação dupla, custódia física
   ┌──────────────────────────▼──────────────────────────────────────────┐
   │ CA RAIZ — offline. Sem caminho de rede. Sem exceção.                │
   └─────────────────────────────────────────────────────────────────────┘
```

Detalhamento em `./Security.md`.

---

## 8. Atributos de Qualidade Arquiteturalmente Significativos

| Atributo | Decisão arquitetural que o sustenta | NFR |
|----------|-------------------------------------|-----|
| Disponibilidade (99,99%) | 3 réplicas stateless + validação local que não depende do serviço. | NFR-005 |
| Latência do sistema | O 021 fora do caminho quente de validação. | NFR-001 |
| Confidencialidade | Envelope KEK/DEK; material entregue uma vez; varredura anti-vazamento. | NFR-008 |
| Contenção de comprometimento | TTL curto + revogação propagada + atestação. | NFR-006, NFR-010 |
| Integridade do isolamento | Perfis assinados verificados pelo consumidor. | NFR-015 |
| Recuperabilidade | Raiz *offline* recuperável; sobreposição de chaves. | NFR-009 |

---

## 9. Riscos Arquiteturais

| Risco | Impacto | Mitigação |
|-------|---------|-----------|
| Comprometimento da CA raiz | Todo o mTLS interno cai | Raiz *offline*, custódia com aprovação dupla, cerimônia documentada em `../029-Operations/`; *cross-signing* previsto na transição. |
| Revogação não instantânea (custo do JWT) | Janela de uso indevido até o TTL | TTL de 15 min + lista de revogação com cache de 10 s; meta de propagação ≤ 30 s (NFR-006). |
| 021 indisponível no *bootstrap* | Sistema não sobe | 3 réplicas; credenciais existentes continuam válidas; ordem de inicialização documentada em `./Deployment.md`. |
| Atestação fraca em ambiente de contêiner | Workload forja identidade | Atestação baseada em atributos verificáveis do orquestrador (`027`/`028`); `sec.attestation.required` é não-recarregável. |
| Explosão de credenciais ativas | Custo e superfície | `max_active_per_principal`; particionamento por `expires_at`; expurgo automático. |
| Vazamento em log de outro módulo | Segredo exposto fora do 021 | Material entregue uma vez, com orientação de não persistir; varredura contínua (FR-018) e rotação imediata ao detectar. |

---

## 10. Referências

- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6
- Brief do módulo: `./_DESIGN_BRIEF.md`
- FSM de credencial: `./StateMachine.md` · API: `./API.md` · Eventos: `./Events.md`
- Escala: `./Scalability.md` · Falhas: `./FailureRecovery.md` · Segurança: `./Security.md`
- Decisões: `./ADR.md` · Especificações: `./RFC.md`
