---
Documento: Monitoring
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0215, ADR-0217
RFCs relacionados: RFC-0001
Depende de: Metrics.md, NonFunctionalRequirements.md, 024-Observability, 025-Audit, 029-Operations
---

# 021-Security — Monitoramento

Dashboards e alertas da raiz de confiança. Todos os limiares derivam dos SLO de
`./NonFunctionalRequirements.md` — divergência é defeito.

> **Particularidade deste módulo.** Boa parte do monitoramento aqui **não** é de
> desempenho: é de **postura de segurança**. Métricas como "credenciais sem expiração"
> ou "falhas de verificação de assinatura de perfil" não têm limiar a calibrar — o
> valor aceitável é zero, e qualquer desvio é incidente.

---

## 1. Golden Signals do Módulo

| Sinal | Métrica principal | Meta |
|-------|-------------------|------|
| **Latência** | `aios_sec_issue_latency_ms`, `aios_sec_introspect_latency_ms` | p99 ≤ 50 ms (token) · ≤ 150 ms (cert) · ≤ 10 ms (introspecção) |
| **Tráfego** | `aios_sec_authn_total`, `aios_sec_issue_total` | ≥ 5.000 emissões/s sustentáveis |
| **Erros** | `aios_sec_authn_failures_total`, `aios_sec_kms_failures_total` | sem picos sustentados |
| **Saturação** | `aios_sec_credentials_active`, `aios_sec_kms_latency_ms` | dentro do envelope de `./Deployment.md` |

Sinais de **postura** (específicos deste módulo): credenciais sem expiração, TTL
mediano, propagação de revogação, idade das chaves, falhas de atestação e achados da
varredura anti-vazamento.

---

## 2. Dashboards

### 2.1 `SEC / Postura` (o dashboard mais importante)
- As **cinco métricas que devem ser zero** (`./Metrics.md` §8) em destaque, cada uma
  com sua última ocorrência não-zero.
- TTL mediano das credenciais emitidas por tipo, com linha em 24 h.
- Idade de cada chave `active` versus seu período de rotação.
- Configurações proibidas ativas (`aios_sec_prohibited_config_active`).

### 2.2 `SEC / Autenticação`
- Autenticações por tipo e resultado; taxa de falha.
- Falhas por motivo **agregado** (nunca por princípio — `./Metrics.md` §0).
- Contenções aplicadas e anomalias sinalizadas.
- Latência dos fluxos OIDC.

### 2.3 `SEC / Credenciais`
- Credenciais ativas por tipo; emissões e rotações por hora.
- Credenciais próximas da expiração sem rotação (FR-019).
- Distribuição de TTL concedido (histograma).

### 2.4 `SEC / Revogação`
- Revogações por motivo; tamanho da lista de revogação.
- Distribuição do tempo de propagação, com linha em 30 s.
- Revogações não propagadas (deve ser zero na maior parte do tempo).

### 2.5 `SEC / Atestação e PKI`
- Atestações por resultado; falhas por natureza da divergência.
- Certificados de workload ativos e próximos da expiração.
- Idade das CAs intermediárias.

### 2.6 `SEC / Chaves e KMS`
- Chaves por papel e estado; idade da KEK e da chave de assinatura.
- Latência e falhas do HSM/KMS.
- Segredos pendentes de rotação.

---

## 3. Regras de Alerta (Prometheus)

```yaml
groups:
- name: aios-security-posture      # postura: valor aceitável é ZERO
  rules:
  - alert: SecCredentialWithoutExpiry
    expr: aios_sec_credentials_without_expiry > 0
    for: 1m
    labels: { severity: P1, module: "021-Security" }
    annotations:
      summary: "Existe credencial ativa sem expiração (viola P-01 e invariante I1)"
      description: "A credencial DEVE ser revogada, não regularizada."
      runbook: "../029-Operations/#sec-eternal-credential"

  - alert: SecSandboxSignatureFailure
    expr: increase(aios_sec_sandbox_signature_failures_total[5m]) > 0
    for: 1m
    labels: { severity: P1 }
    annotations:
      summary: "Perfil de sandbox com assinatura inválida (NFR-015)"
      description: "Isolamento não garantido: o runtime DEVE recusar iniciar o agente."
      runbook: "../029-Operations/#sec-profile-signature"

  - alert: SecMaterialLeakDetected
    expr: increase(aios_sec_leak_scan_findings_total[5m]) > 0
    for: 1m
    labels: { severity: P1 }
    annotations:
      summary: "Material sensível encontrado em log/evento/métrica/resposta (NFR-008)"
      description: "Rotacionar imediatamente o material exposto e expurgar o artefato."
      runbook: "../029-Operations/#sec-leak-response"

  - alert: SecRotationOverlapViolation
    expr: aios_sec_rotation_overlap_violations > 0
    for: 2m
    labels: { severity: P1 }
    annotations:
      summary: "Mais de duas credenciais utilizáveis por purpose (invariante I2)"

  - alert: SecProhibitedConfigActive
    expr: aios_sec_prohibited_config_active > 0
    for: 1m
    labels: { severity: P1 }
    annotations:
      summary: "Configuração proibida em produção ativa"
      runbook: "../029-Operations/#sec-config-review"

- name: aios-security-critical
  rules:
  - alert: SecServiceDown
    expr: sum(up{job="aios-security-svc"}) < 2
    for: 1m
    labels: { severity: P1 }
    annotations:
      summary: "Menos de 2 réplicas do 021 ativas — SLO de 99,99% em risco"
      description: "Validação de tokens existentes continua; emissão está degradada."

  - alert: SecKmsUnavailable
    expr: increase(aios_sec_kms_failures_total[5m]) > 10
    for: 2m
    labels: { severity: P1 }
    annotations:
      summary: "KMS/HSM indisponível — emissão suspensa (AIOS-SEC-0010)"
      description: "Validação e mTLS existentes NÃO são afetados."
      runbook: "../029-Operations/#sec-kms-down"

  - alert: SecAttestationMismatch
    expr: increase(aios_sec_attestation_failures_total{failure_kind="service_identity_mismatch"}[15m]) > 0
    for: 1m
    labels: { severity: P1 }
    annotations:
      summary: "Divergência entre serviço declarado e observado na atestação"
      description: "Pode ser defeito de deploy — ou tentativa de assumir identidade alheia."
      runbook: "../029-Operations/#sec-attestation-mismatch"

  - alert: SecRevocationNotPropagating
    expr: aios_sec_revocations_unpropagated > 0
    for: 2m
    labels: { severity: P1 }
    annotations:
      summary: "Revogação sem propagação além da meta de 30 s (NFR-006)"
      description: "Credencial revogada pode continuar sendo aceita."
      runbook: "../029-Operations/#sec-revocation-stuck"

- name: aios-security-warning
  rules:
  - alert: SecCredentialTtlDrift
    expr: histogram_quantile(0.5, sum by (le) (rate(aios_sec_credential_ttl_hours_bucket[24h]))) > 24
    for: 1h
    labels: { severity: P2 }
    annotations: { summary: "TTL mediano das credenciais acima de 24 h (NFR-010)" }

  - alert: SecKeyRotationOverdue
    expr: aios_sec_key_age_days{role="signing"} > 90 or aios_sec_key_age_days{role="kek"} > 365
    for: 1h
    labels: { severity: P2 }
    annotations:
      summary: "Chave além do período de rotação (NFR-011)"
      runbook: "../029-Operations/#sec-key-rotation"

  - alert: SecAuthnFailureSpike
    expr: sum(rate(aios_sec_authn_failures_total[5m])) > 50
    for: 10m
    labels: { severity: P2 }
    annotations: { summary: "Pico sustentado de falhas de autenticação" }

  - alert: SecRevokedCredentialInUse
    expr: increase(aios_sec_authn_failures_total{reason="revoked"}[15m]) > 5
    for: 5m
    labels: { severity: P2 }
    annotations:
      summary: "Uso repetido de credencial revogada"
      description: "Cache desatualizado — ou tentativa com material vazado."

  - alert: SecIssueLatencyHigh
    expr: histogram_quantile(0.99, sum by (le) (rate(aios_sec_issue_latency_ms_bucket{kind="workload_cert"}[10m]))) > 150
    for: 10m
    labels: { severity: P2 }
    annotations: { summary: "p99 de emissão de certificado acima de 150 ms (NFR-002)" }

  - alert: SecCredentialsExpiringUnrotated
    expr: aios_sec_credentials_expiring_soon > 0
    for: 15m
    labels: { severity: P2 }
    annotations: { summary: "Credenciais próximas da expiração sem rotação programada (FR-019)" }

  - alert: SecFederationScopeDenials
    expr: increase(aios_sec_federation_scope_denials_total[1h]) > 10
    for: 30m
    labels: { severity: P3 }
    annotations: { summary: "IdP externo solicitando escopos além do permitido — revisar acordo" }
```

---

## 4. SLO, SLI e Error Budget

| SLO | SLI | Janela | Budget |
|-----|-----|--------|--------|
| Disponibilidade ≥ 99,99% (NFR-005) | fração de minutos com emissão e autenticação atendidas | 30 dias | **4,4 min** |
| p99 emissão de token ≤ 50 ms (NFR-002) | fração de janelas de 5 min na meta | 30 dias | 1% das janelas |
| Propagação ≤ 30 s (NFR-006) | `aios_sec_revocation_propagation_ms` p99 | 30 dias | 1% das revogações |
| Confidencialidade (NFR-008) | `aios_sec_leak_scan_findings_total` | contínuo | **0** |
| Vida curta (NFR-010) | `aios_sec_credentials_without_expiry` | contínuo | **0** |
| Integridade de perfil (NFR-015) | `aios_sec_sandbox_signature_failures_total` | contínuo | **0** |

Ao esgotar o budget de disponibilidade (4,4 min — o mais apertado do AIOS), **qualquer**
mudança no módulo é congelada até revisão. As metas com budget zero não têm política de
esgotamento: elas têm resposta a incidente.

---

## 5. Runbooks Associados

| Alerta | Runbook (`../029-Operations/`) | Primeira ação |
|--------|-------------------------------|---------------|
| `SecCredentialWithoutExpiry` | `#sec-eternal-credential` | Identificar a credencial e **revogá-la**; investigar como o caminho de criação a permitiu (a *constraint* do banco deveria ter impedido). |
| `SecSandboxSignatureFailure` | `#sec-profile-signature` | Confirmar com o `../007-Agent-Runtime/` que nenhum agente iniciou; verificar se houve rotação recente da chave de assinatura; se não, tratar como adulteração. |
| `SecMaterialLeakDetected` | `#sec-leak-response` | **Rotacionar o material exposto antes de qualquer outra coisa**; depois expurgar o artefato e corrigir a origem. A ordem importa. |
| `SecKmsUnavailable` | `#sec-kms-down` | Confirmar que a validação segue operando (é o mais importante); restaurar o KMS; a fila de emissão drena sozinha. |
| `SecAttestationMismatch` | `#sec-attestation-mismatch` | Comparar serviço declarado × observado; se não for defeito de deploy conhecido, tratar como incidente de segurança e isolar o nó. |
| `SecRevocationNotPropagating` | `#sec-revocation-stuck` | Verificar `security.outbox` e o NATS; se o barramento estiver fora, orientar os verificadores a consultar a CRL diretamente. |
| `SecKeyRotationOverdue` | `#sec-key-rotation` | Rotação de assinatura é automatizável; a de KEK exige cerimônia com aprovação dupla (UC-010). |
| `SecServiceDown` | `#sec-service-down` | Verificar que tokens existentes continuam válidos; restaurar réplicas; **não** ampliar TTLs como paliativo. |

---

## 6. Correlação com Auditoria

Alertas deste módulo **DEVEM** ser investigados junto com a trilha de `../025-Audit/`,
não isoladamente. Métricas dizem *que* algo aconteceu; a auditoria diz *quem* fez, *com
qual capability* e *sobre qual recurso* — e é ela que permite reconstruir um incidente.

| Alerta | Consulta correspondente na auditoria |
|--------|--------------------------------------|
| `SecCredentialWithoutExpiry` | Emissões com TTL fora do padrão e seus autores. |
| `SecAttestationMismatch` | Todas as tentativas do mesmo `nodeRef` na janela. |
| `SecMaterialLeakDetected` | Emissões da credencial exposta e quem a leu desde então. |
| `SecRevokedCredentialInUse` | Origem das tentativas e horário da revogação original. |
| `SecProhibitedConfigActive` | Quem alterou a configuração, quando e de qual valor. |

---

## 7. Observação de Longo Prazo

- Evolução do TTL mediano por tipo de credencial — erosão lenta do princípio P-01 é o
  padrão de degradação mais comum em sistemas de identidade.
- Crescimento de `aios_sec_credentials_active` frente ao número de agentes: divergência
  indica credenciais que não estão sendo expiradas ou revogadas.
- Proporção de autenticações federadas versus locais por tenant.
- Tendência de `aios_sec_attestation_failures_total` após cada mudança de topologia
  (`027`/`028`) — picos costumam ser defeito de deploy, não ataque.
- Tamanho da lista de revogação: crescimento sustentado sugere TTLs longos demais (uma
  lista grande é sintoma de credenciais que demoram a expirar sozinhas).

---

## 8. Referências

- Métricas: `./Metrics.md` · SLO: `./NonFunctionalRequirements.md`
- Logs: `./Logging.md` · Falhas e cerimônias: `./FailureRecovery.md`
- Auditoria: `../025-Audit/` · Runbooks: `../029-Operations/`
