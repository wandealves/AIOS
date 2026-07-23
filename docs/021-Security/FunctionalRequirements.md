---
Documento: FunctionalRequirements
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0210, ADR-0211, ADR-0212, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0217, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: _DESIGN_BRIEF.md §7.1, 022-Policy, 025-Audit, 007-Agent-Runtime, 020-Communication
---

# 021-Security — Requisitos Funcionais

Requisitos derivados de `./_DESIGN_BRIEF.md` §7.1. Prioridade em **MoSCoW**. Palavras
normativas conforme RFC 2119. Todo FR é verificável por ao menos um caso de teste em
`./Testing.md`.

---

## 1. Tabela de Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-001 | O módulo DEVE operar um emissor OIDC com os fluxos `authorization_code` (com PKCE) e `client_credentials`. | Must | Suíte de conformidade OIDC verde; `/.well-known/openid-configuration` e `/oauth2/introspect` disponíveis. | Brief §1.2 R-01 |
| FR-002 | O módulo DEVE publicar o JWKS com sobreposição de chaves durante a rotação. | Must | Tokens assinados pela chave anterior permanecem válidos até expirar; `security.jwks.rotated` emitido. | Brief §1.2 R-02 |
| FR-003 | O módulo DEVE emitir certificados de workload de vida curta para mTLS, **após atestação** do solicitante. | Must | Emissão sem atestação → `AIOS-SEC-0006`; TTL default ≤ 24 h. | Brief §1.2 R-03/R-04 |
| FR-004 | O módulo DEVE emitir credenciais dinâmicas para PostgreSQL, NATS, MinIO e provedores de LLM. | Must | Credencial funciona no recurso alvo e expira no TTL declarado. | Brief §1.2 R-05 |
| FR-005 | O módulo DEVE rotacionar credenciais com janela de sobreposição, sem downtime do consumidor. | Must | Durante `Rotating`, antiga e nova funcionam; após a janela, apenas a nova (invariante I2). | Brief §4 T-05/T-06 |
| FR-006 | O módulo DEVE revogar credencial em qualquer estado não-terminal e propagar o efeito. | Must | Entrada em `security.revocation` + evento `token.revoked` na **mesma transação** (invariante I4). | Brief §1.2 R-08 |
| FR-007 | O módulo DEVE recusar emissão sem expiração ou com TTL acima do máximo do tipo. | Must | `expires_at` sempre preenchido (I1); TTL excessivo → `AIOS-SEC-0004`. | Brief §3.2, P-01 |
| FR-008 | O módulo DEVE custodiar segredos versionados, cifrados por envelope, com leitura sob *lease*. | Must | Nenhum segredo em claro no banco; leitura fora do lease → `AIOS-SEC-0014`. | Brief §1.2 R-05, §3.3 |
| FR-009 | O módulo DEVE operar o KMS com hierarquia KEK/DEK e rotação programada. | Must | Rotação de KEK recifra apenas as DEK, não os dados. | Brief §1.2 R-06 |
| FR-010 | O módulo DEVE publicar perfis de isolamento versionados e **assinados**. | Must | Perfil publicado é imutável (`AIOS-SEC-0012`); assinatura verificável pelo `../007-Agent-Runtime/`. | Brief §1.2 R-07 |
| FR-011 | O módulo DEVERIA integrar IdP externo por tenant, mapeando identidade externa a princípio interno com **escopo reduzido**. | Should | `issuer` desconhecido → `AIOS-AUTHN-0006`; escopo fora do permitido → `AIOS-AUTHN-0007`. | Brief §1.2 R-09 |
| FR-012 | O módulo DEVERIA manter âncoras de confiança para pares A2A externos do `../020-Communication/`. | Should | Suspender a confiança encerra as sessões A2A correspondentes. | Brief §1.2 R-09 |
| FR-013 | O módulo DEVERIA detectar e sinalizar anomalias de autenticação **sem** decidir bloqueio. | Should | Rajada de falhas → `AIOS-AUTHN-0005` + evento `authn.anomaly`; nenhuma decisão de política tomada localmente. | Brief §1.2 R-10 |
| FR-014 | O módulo DEVE desabilitar princípio e revogar em cascata suas credenciais. | Must | Todas as credenciais do princípio → `Revoked`; evento `principal.disabled` emitido. | Brief §4 T-07 |
| FR-015 | O módulo DEVERIA prover serviços criptográficos com governança de suítes permitidas. | Should | Algoritmo fora de `sec.key.allowed_suites` → `AIOS-SEC-0011`. | Brief §1.2 R-11 |
| FR-016 | O módulo DEVE consultar o PDP antes de toda operação administrativa. | Must | Sem capability → `AIOS-SEC-0002`; decisão registrada em `../025-Audit/`. | Brief §1.2 R-12 |
| FR-017 | O módulo DEVE emitir todos os eventos de `./Events.md` via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. | Brief §1.2 R-12 |
| FR-018 | O módulo **NÃO DEVE** expor material secreto em API de leitura, log, evento ou métrica. | Must | Varredura automatizada de respostas, logs, eventos e métricas não encontra material sensível. | Brief §12.2 |
| FR-019 | O módulo DEVERIA alertar sobre credenciais próximas da expiração sem rotação programada. | Should | Evento `credential.expiring` emitido em ≤ 25% do TTL restante. | Brief §6.1 |
| FR-020 | O módulo DEVE suportar o gatilho de direito ao esquecimento, emitindo `security.rtbf.requested`. | Must | Evento consumido por `../010-Memory/` e `../005-Database/`; comprovante correlacionado em `../025-Audit/`. | Brief §12.3 |

---

## 2. Regras de Negócio Transversais

| # | Regra | Aplicação |
|---|-------|-----------|
| RN-01 | Toda mutação **DEVE** aceitar `Idempotency-Key` e produzir efeito único (RFC-0001 §5.5). | FR-003..FR-006, FR-010 |
| RN-02 | Toda operação privilegiada **DEVE** ser auditada em `../025-Audit/` com identidade, capability, decisão e resultado. | FR-016 e todas as mutações |
| RN-03 | Nenhum documento ou operação **PODE** redefinir contratos centrais (mTLS interno, validação no Gateway, URN, envelope) — eles vêm da RFC-0001 §5–§6. | Todos |
| RN-04 | **Revogação é irreversível**: `Revoked` não transita para nenhum estado (invariante I3). Recuperar acesso exige nova emissão. | FR-006, FR-014 |
| RN-05 | O material privado **NÃO DEVE** ser relido após a emissão (invariante I5): guarda-se referência cifrada e `fingerprint`. | FR-003, FR-004, FR-008 |
| RN-06 | Mensagens de erro de autenticação **NÃO DEVEM** distinguir "princípio inexistente" de "credencial incorreta" — isso seria um oráculo de enumeração. O motivo real vai só para auditoria. | FR-013, `AIOS-AUTHN-0001` |
| RN-07 | Sob PDP indisponível, o módulo nega **operações novas** (`fail-closed`), mas **NÃO DEVE** interromper autenticação nem validação de credenciais já emitidas. | FR-016 |

---

## 3. Exemplo de Verificação — FR-007

```bash
# Tentativa de emitir token com TTL de 30 dias (máximo do tipo: 1 h)
curl -sX POST https://api.aios.local/v1/security/credentials \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9ZD0K3N5Q7R9T1V3X5Z7A9C" \
  -H "Content-Type: application/json" \
  -d '{ "principalUrn": "urn:aios:acme:principal:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
        "kind": "access_token",
        "purpose": "api:read",
        "ttlSeconds": 2592000 }'
```

```json
{
  "type": "https://docs.aios/errors/ttl-exceeds-maximum",
  "title": "Requested TTL Exceeds Maximum For Credential Kind",
  "status": 422,
  "code": "AIOS-SEC-0004",
  "detail": "ttlSeconds=2592000 excede sec.token.max_ttl_s=3600 para kind=access_token.",
  "retriable": false
}
```

Não há caminho para contornar esse limite via API. Elevá-lo exige alterar configuração
global sob capability própria e fica registrado — porque a exceção "só desta vez" é
exatamente como credenciais eternas nascem.

---

## 4. Rastreabilidade

A matriz FR → UC → Teste está em `./Requirements.md` §3. Cada FR aparece literalmente,
com o mesmo identificador, em `./UseCases.md`, `./Testing.md` e — quando tem meta
numérica — em `./NonFunctionalRequirements.md`.

---

## 5. Referências

- Brief: `./_DESIGN_BRIEF.md` §7.1
- Requisitos não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
