---
Documento: Metrics
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0215, ADR-0217
RFCs relacionados: RFC-0001
Depende de: 024-Observability, NonFunctionalRequirements.md, Monitoring.md
---

# 021-Security — Catálogo de Métricas

Convenção: `aios_<subsistema>_<nome>_<unidade>`, subsistema = **`sec`**. Exportação
**OpenTelemetry → Prometheus** (`../024-Observability/`). Labels comuns: `service`,
`instance`, `env`.

> **Regra de cardinalidade e de privacidade.** `principal_urn`, `credential_urn`,
> `fingerprint`, `tenant_id` e `secret_path` **NÃO DEVEM** ser labels de métrica — por
> cardinalidade (10⁶ princípios) **e** por privacidade: um painel de métricas não deve
> permitir enumerar princípios ou inferir quem está falhando ao autenticar. O recorte
> individual é feito por traces e auditoria (`./Logging.md`).

---

## 1. Autenticação

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_authn_total` | counter | `kind` (human\|service\|external), `result` (success\|failure) | 1 | Tentativas de autenticação. | NFR-014 |
| `aios_sec_authn_failures_total` | counter | `reason` (bad_credentials\|expired\|mfa_missing\|revoked\|untrusted_issuer) | 1 | Falhas por motivo **agregado** (nunca por princípio). | NFR-014 |
| `aios_sec_authn_lockouts_total` | counter | `kind` | 1 | Contenções por força bruta aplicadas. | NFR-014 |
| `aios_sec_authn_latency_ms` | histogram | `kind`, `flow` (code\|client_credentials\|refresh) | ms | Latência do fluxo de autenticação. | NFR-001 |
| `aios_sec_mfa_challenges_total` | counter | `result` | 1 | Desafios de segundo fator. | — |

`aios_sec_authn_failures_total{reason="revoked"}` merece atenção especial: uso de
credencial revogada indica ou cache desatualizado (defeito de propagação) ou tentativa
deliberada com material vazado.

---

## 2. Emissão e Ciclo de Vida de Credencial

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_issue_total` | counter | `kind`, `result` (issued\|failed) | 1 | Emissões de credencial. | NFR-004 |
| `aios_sec_issue_latency_ms` | histogram | `kind` | ms | Latência de emissão. | NFR-002 |
| `aios_sec_credentials_active` | gauge | `kind` | 1 | Credenciais em `Active`/`Rotating`. | NFR-007 |
| `aios_sec_credential_ttl_hours` | histogram | `kind` | h | TTL concedido — **mediana deve ser ≤ 24 h**. | NFR-010 |
| `aios_sec_credentials_without_expiry` | gauge | — | 1 | Credenciais ativas sem expiração. **Deve ser sempre 0.** | NFR-010 |
| `aios_sec_rotations_total` | counter | `kind`, `trigger` (scheduled\|on_demand\|renewal) | 1 | Rotações executadas. | NFR-013 |
| `aios_sec_rotation_overlap_violations` | gauge | — | 1 | Casos com > 2 credenciais utilizáveis por `purpose` (invariante I2). **Deve ser 0.** | — |
| `aios_sec_credentials_expiring_soon` | gauge | `kind` | 1 | Ativas com < 25% do TTL restante e sem rotação. | FR-019 |

Buckets de `aios_sec_credential_ttl_hours`: `[0.25, 1, 4, 12, 24, 72, 168, 720]` — o
bucket **24 h** é a meta de NFR-010, e a cauda acima de 168 h (uma semana) é o sinal de
que o princípio P-01 está sendo erodido.

---

## 3. Revogação

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_revocations_total` | counter | `reason` (compromise\|superseded\|principal_disabled\|policy\|operator) | 1 | Revogações executadas. | FR-006 |
| `aios_sec_revocation_propagation_ms` | histogram | — | ms | Intervalo entre a revogação e a confirmação de propagação. | NFR-006 |
| `aios_sec_revocations_unpropagated` | gauge | — | 1 | Revogações além da meta sem propagação confirmada. | NFR-006 |
| `aios_sec_revocation_list_size` | gauge | — | 1 | Entradas ativas na lista de revogação. | — |

Buckets de `aios_sec_revocation_propagation_ms`: `[100, 500, 1000, 5000, 10000, 30000,
60000, 300000]` — o bucket **30.000 ms** é a meta de NFR-006.

---

## 4. Segredos e Chaves

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_secret_read_total` | counter | `result` | 1 | Leituras de segredo (**sem** `path` como label). | FR-008 |
| `aios_sec_secret_read_latency_ms` | histogram | — | ms | Latência de leitura com lease. | NFR-003 |
| `aios_sec_secret_lease_violations_total` | counter | — | 1 | Leituras fora do lease (`AIOS-SEC-0014`). | — |
| `aios_sec_secrets_pending_rotation` | gauge | — | 1 | Segredos além do `rotation_period`. | NFR-011 |
| `aios_sec_key_age_days` | gauge | `role` (kek\|signing\|ca_intermediate) | d | Idade da chave `active` por papel. | NFR-011 |
| `aios_sec_keys_by_state` | gauge | `role`, `state` | 1 | Distribuição de chaves por estado. | — |
| `aios_sec_kms_latency_ms` | histogram | `op` (wrap\|unwrap\|sign) | ms | Latência do KMS/HSM. | NFR-002 |
| `aios_sec_kms_failures_total` | counter | `op` | 1 | Falhas do KMS/HSM (`AIOS-SEC-0010`). | NFR-005 |

---

## 5. Atestação e Certificados

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_attestation_total` | counter | `result` (ok\|failed) | 1 | Atestações realizadas. | FR-003 |
| `aios_sec_attestation_failures_total` | counter | `failure_kind` (service_identity_mismatch\|node_unknown\|image_mismatch\|proof_expired) | 1 | Falhas por natureza. | NFR-008 |
| `aios_sec_attestation_latency_ms` | histogram | — | ms | Latência da atestação. | NFR-002 |
| `aios_sec_certs_active` | gauge | — | 1 | Certificados de workload válidos. | NFR-007 |
| `aios_sec_cert_expiring_soon` | gauge | — | 1 | Certificados com < 25% do TTL e sem renovação. | FR-019 |

---

## 6. Perfis de Isolamento e Federação

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_sandbox_profiles_published` | gauge | `status` | 1 | Perfis por situação. | FR-010 |
| `aios_sec_sandbox_signature_failures_total` | counter | — | 1 | Verificações de assinatura reprovadas (reportadas pelo `../007-Agent-Runtime/`). **Deve ser 0.** | NFR-015 |
| `aios_sec_federation_trusts` | gauge | `kind`, `status` | 1 | Confianças federadas registradas. | FR-011 |
| `aios_sec_federation_authn_total` | counter | `result` | 1 | Autenticações via IdP externo. | FR-011 |
| `aios_sec_federation_scope_denials_total` | counter | — | 1 | Escopos recusados a identidade federada (`AIOS-AUTHN-0007`). | ADR-0218 |

---

## 7. Governança e Plataforma

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_sec_policy_decisions_total` | counter | `capability`, `decision` (allow\|deny) | 1 | Decisões do PDP para operações do módulo. | FR-016 |
| `aios_sec_introspect_latency_ms` | histogram | — | ms | Latência de introspecção. | NFR-001 |
| `aios_sec_jwks_key_count` | gauge | — | 1 | Chaves publicadas no JWKS (2 durante rotação). | FR-002 |
| `aios_sec_outbox_pending` | gauge | — | 1 | Eventos aguardando publicação. | FR-017 |
| `aios_sec_config_pending_reload` | gauge | — | 1 | Chaves não-recarregáveis alteradas aguardando reinício. | — |
| `aios_sec_leak_scan_findings_total` | counter | `surface` (log\|event\|metric\|api_response) | 1 | Achados da varredura anti-vazamento. **Deve ser 0.** | NFR-008 |
| `aios_sec_prohibited_config_active` | gauge | `key` | 1 | Configurações proibidas em produção ativas (`./Configuration.md` §5). **Deve ser 0.** | — |

---

## 8. As Cinco Métricas que Devem Ser Zero

Este módulo tem um conjunto incomum de métricas cujo **único valor aceitável é zero**.
Elas não são indicadores de tendência: qualquer valor diferente de zero é incidente.

| Métrica | O que um valor > 0 significa |
|---------|------------------------------|
| `aios_sec_credentials_without_expiry` | Existe credencial eterna — violação de P-01 e da invariante I1. |
| `aios_sec_rotation_overlap_violations` | Mais de duas credenciais utilizáveis por `purpose` — violação da invariante I2. |
| `aios_sec_sandbox_signature_failures_total` | Um agente foi iniciado (ou tentou ser) com perfil não verificado — isolamento não garantido. |
| `aios_sec_leak_scan_findings_total` | Material sensível apareceu em log, evento, métrica ou resposta. |
| `aios_sec_prohibited_config_active` | Uma configuração proibida em produção está ativa. |

---

## 9. Exemplos de Consulta

```promql
# NFR-010 — nenhuma credencial eterna (deve ser sempre 0)
aios_sec_credentials_without_expiry > 0

# NFR-010 — TTL mediano das credenciais emitidas nas últimas 24h
histogram_quantile(0.5, sum by (le) (rate(aios_sec_credential_ttl_hours_bucket[24h]))) > 24

# NFR-006 — propagação de revogação acima da meta
histogram_quantile(0.99, sum by (le) (rate(aios_sec_revocation_propagation_ms_bucket[1h]))) > 30000

# NFR-011 — chave de assinatura fora do período de rotação
aios_sec_key_age_days{role="signing"} > 90

# Falhas de atestação por divergência de identidade (sinal de segurança)
increase(aios_sec_attestation_failures_total{failure_kind="service_identity_mismatch"}[15m]) > 0

# JWKS com apenas uma chave durante janela de rotação esperada (defeito de publicação)
aios_sec_jwks_key_count < 2 and on() aios_sec_keys_by_state{role="signing",state="rotating"} > 0
```

---

## 10. Referências

- NFR e metas: `./NonFunctionalRequirements.md`
- Alertas e dashboards: `./Monitoring.md` · Logs: `./Logging.md`
- Plataforma de observabilidade: `../024-Observability/`
