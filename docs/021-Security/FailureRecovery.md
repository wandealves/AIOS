---
Documento: FailureRecovery
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0214, ADR-0216, ADR-0219
RFCs relacionados: RFC-0001, RFC-0211
Depende de: _DESIGN_BRIEF.md §9, 027-Cluster, 029-Operations, Monitoring.md
---

# 021-Security — Falhas e Recuperação

Metas: **RTO ≤ 15 min**, **RPO ≤ 5 min** (NFR-009), com a CA raiz recuperável de
custódia *offline*.

**Princípio de degradação deste módulo:** sacrifica-se **emissão** antes de
**validação**. Um sistema que não emite credenciais novas degrada; um sistema que não
valida as existentes **para completamente**. Toda decisão arquitetural aqui — validação
local, TTL curto, cache de CRL — existe para manter essa ordem sob qualquer falha.

---

## 1. FMEA — Modos de Falha

| # | Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry | Severidade |
|---|---------------|----------|-----------|-------------|--------------------|-----------|
| F-01 | KMS/HSM indisponível | Health + `aios_sec_kms_failures_total` | **Emissão** suspensa (`AIOS-SEC-0010`); validação e mTLS **continuam** | Restaurar KMS; fila de emissão drena | Idempotente por `Idempotency-Key` | **P1** |
| F-02 | Perda da chave de assinatura corrente | Falha ao assinar | Chave anterior ainda verifica (sobreposição) | Promover chave `pending`; publicar JWKS; emitir `jwks.rotated` | Tokens antigos seguem válidos até expirar | **P1** |
| F-03 | Comprometimento de credencial | `ThreatSignalDetector` ou reporte externo | Revogação imediata (T-07) | Nova emissão; rotação forçada dos pares afetados | Revogação idempotente | **P1** |
| F-04 | Comprometimento de CA **intermediária** | Auditoria / alerta | Revogação da intermediária; emissão suspensa naquele ramo | Nova intermediária a partir da raiz *offline*; reemissão em massa de certificados de workload | Reemissão idempotente por workload | **P1** |
| F-05 | Comprometimento da CA **raiz** | Detecção fora de banda | Todo o mTLS interno é afetado | **Cerimônia de recuperação** (§5.3): nova raiz, *cross-signing* durante a transição | Procedimento manual documentado | **P1 crítico** |
| F-06 | PDP (`022`) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do PEP | *Fail-closed*: operações administrativas negadas; **autenticação e validação continuam** | Retry após meia-abertura do CB | P2 |
| F-07 | PostgreSQL (`security`) indisponível | Health | Emissão e revogação param; **validação por JWKS no `004` continua** | Failover do `../005-Database/`; reconciliação | Idempotente | **P1** |
| F-08 | Falha de propagação de revogação | `revocations_unpropagated > 0` | Verificadores consultam a CRL diretamente (fallback) | Reenvio pelo Outbox; alerta se exceder NFR-006 | Dedupe por `event.id` | **P1** |
| F-09 | Rajada de autenticação falha | `authn.failures` por princípio | Contenção **por princípio**, nunca global | Liberação após `lockout_s`; sinal a `025` | Determinístico | P2 |
| F-10 | Atestação falhando em massa | `attestation.failed` | Emissão negada aos workloads afetados | Investigar mudança de topologia (`027`) **antes** de relaxar a exigência | Nenhum retry cego | **P1** |
| F-11 | Relógio dessincronizado | Validação de `not_before`/`expires_at` | Tolerância `clock_skew_s` (60 s) | Corrigir NTP | — | P2 |
| F-12 | Vazamento de material em log/evento | Varredura contínua (FR-018) | Artefato quarentenado | **Rotacionar o material exposto primeiro**; depois expurgar e corrigir a origem | — | **P1** |
| F-13 | Perda de 2 das 3 réplicas | `up{job="aios-security-svc"} < 2` | Capacidade reduzida | Restaurar réplicas | Stateless: recuperação trivial | P2 |
| F-14 | Credencial de bootstrap comprometida | Auditoria de acesso ao schema | Acesso direto ao schema `security` | Rotação manual da credencial de bootstrap (`../029-Operations/`) | Procedimento documentado | **P1 crítico** |
| F-15 | Perfil de sandbox com assinatura inválida | Reportado pelo `../007-Agent-Runtime/` | Runtime **recusa** iniciar o agente | Republicar perfil; se não houve rotação de chave, tratar como adulteração | — | **P1** |

**F-05** e **F-14** são as duas falhas verdadeiramente críticas: ambas afetam a raiz de
confiança e ambas exigem procedimento manual com aprovação dupla. Elas são a razão de as
cerimônias serem ensaiadas em staging (T-CER-01, T-CER-02).

---

## 2. O que NUNCA para

Esta tabela é o compromisso operacional do módulo — e a razão de várias decisões
arquiteturais aparentemente indiretas:

| Falha | Emissão | Validação de token | mTLS existente | Autenticação |
|-------|---------|--------------------|----------------|--------------|
| KMS fora (F-01) | ✗ | ✓ | ✓ | ✓ (até o TTL) |
| PostgreSQL fora (F-07) | ✗ | ✓ | ✓ | ✗ |
| PDP fora (F-06) | ✗ (admin) | ✓ | ✓ | ✓ |
| NATS fora | ✗ (eventos) | ✓ | ✓ | ✓ |
| **021 completamente fora** | ✗ | **✓** | **✓** | ✗ |

A última linha é a mais importante: mesmo com o módulo inteiro indisponível, o AIOS
**continua operando** com as credenciais já emitidas, até que elas expirem. Essa
propriedade é consequência direta de ADR-0211 (validação local) e é o que torna
alcançável um SLO de 99,99% em um serviço que, se estivesse no caminho quente, teria de
ser praticamente infalível.

---

## 3. Detecção — Sinais Primários

| Sinal | Fonte | Falhas que revela |
|-------|-------|-------------------|
| `aios_sec_kms_failures_total` | serviço | F-01 |
| `aios_sec_jwks_key_count` | serviço | F-02 |
| `aios_sec_attestation_failures_total{failure_kind}` | serviço | F-10, tentativa de spoofing |
| `aios_sec_revocations_unpropagated` | serviço | F-08 |
| `aios_sec_authn_failures_total{reason="revoked"}` | serviço | F-03, F-08 |
| `aios_sec_leak_scan_findings_total` | job de varredura | F-12 |
| `aios_sec_sandbox_signature_failures_total` | `007` | F-15 |
| `up{job="aios-security-svc"}` | Prometheus | F-13 |
| Auditoria de acesso ao schema | `025` | F-14 |

---

## 4. Estratégia de Retry

| Erro | `retriable` | Política do cliente |
|------|-------------|---------------------|
| `AIOS-SEC-0010` (KMS fora) | sim | Backoff exponencial 1 s → 60 s; a credencial atual continua válida — **não** há urgência de emitir. |
| `AIOS-SEC-0009` (conflito OCC) | sim | Reler, reaplicar, repetir (máx. 5). |
| `AIOS-SEC-0013` (limite de credenciais) | sim | Revogar credenciais ociosas antes de repetir. |
| `AIOS-AUTHN-0005` (contenção) | sim | Respeitar `lockout_s`; repetir antes **prolonga** a contenção. |
| `AIOS-SEC-0002`, `-0003`, `-0004`, `-0006`, `-0007`, `-0011`, `-0012`, `AIOS-AUTHN-0001`, `-0006`, `-0007` | **não** | Corrigir a causa; repetir não resolve. |

Toda mutação repetida **DEVE** reusar a mesma `Idempotency-Key` (RFC-0001 §5.5).
Especificamente para emissão: repetir sem a chave produziria uma **segunda credencial
válida**, ampliando a superfície sem que ninguém notasse.

---

## 5. Procedimentos de Recuperação

### 5.1 KMS/HSM indisponível (F-01)

```
 1. Confirmar que a VALIDAÇÃO segue operando (é o mais importante — verificar no 004).
 2. Comunicar: novas emissões suspensas; credenciais existentes intactas.
 3. Restaurar o KMS/HSM (fornecedor, rede, capacidade).
 4. A fila de emissão drena sozinha (idempotente).
 5. Verificar credenciais que expiraram durante a janela e reemitir.
 6. Post-mortem: o TTL vigente deu margem suficiente para a janela de indisponibilidade?
```
O passo 6 é o aprendizado recorrente: TTL curto demais reduz a tolerância a falhas do
KMS; TTL longo demais amplia a janela de uso de material vazado. O equilíbrio (24 h) é
uma escolha, não um default.

### 5.2 Comprometimento de CA intermediária (F-04)

```
 1. Revogar a intermediária; suspender emissão naquele ramo.
 2. Emitir nova intermediária a partir da raiz OFFLINE (cerimônia, aprovação dupla).
 3. Reemitir certificados de workload — TTL de 24 h significa que a frota se renova
    naturalmente em um dia, mas em incidente força-se a renovação imediata.
 4. Publicar a revogação da intermediária a todos os verificadores.
 5. Auditar todo certificado emitido pela intermediária comprometida.
```

### 5.3 Comprometimento da CA raiz (F-05) — cerimônia

```
 1. Declarar incidente crítico; acionar a custódia (aprovação dupla, presencial).
 2. Gerar nova raiz em HSM.
 3. CROSS-SIGNING: a nova raiz assina uma intermediária; durante a transição,
    ambas as cadeias são aceitas pelos verificadores.
 4. Reemitir toda a hierarquia; migrar os verificadores para a nova âncora.
 5. Revogar a raiz antiga somente APÓS a migração completa dos verificadores.
 6. Post-mortem com revisão do modelo de custódia.
```
O passo 5 é onde a pressa causa dano: revogar a raiz antiga antes de todos os
verificadores confiarem na nova derruba o mTLS interno inteiro — transformando um
incidente de segurança em indisponibilidade total.

### 5.4 Vazamento de material (F-12)

```
 1. ROTACIONAR o material exposto — antes de qualquer outra coisa.
 2. Revogar a credencial comprometida (T-07) e confirmar a propagação.
 3. Só então expurgar o artefato que vazou (log, evento, painel).
 4. Corrigir a origem do vazamento (código, template de log, campo de métrica).
 5. Auditar todo uso da credencial desde a emissão (via fingerprint).
```
A ordem é deliberada: expurgar o log primeiro apagaria a evidência do alcance do
vazamento, sem reduzir o risco — o material já circulou.

---

## 6. Testes de Recuperação

| Exercício | Frequência | Critério |
|-----------|-----------|----------|
| Chaos: KMS indisponível por 10 min | Mensal | Validação e mTLS intactos; fila drena ao voltar (T-CHAOS-01). |
| Chaos: PDP indisponível | Trimestral | Administração negada; autenticação continua (T-PEP-02). |
| Chaos: PostgreSQL indisponível | Trimestral | Validação por JWKS continua (T-CHAOS-03). |
| DR drill: restauração do schema | Trimestral | RTO ≤ 15 min; segredos ilegíveis sem a KEK (T-DR-01). |
| **Cerimônia: rotação de KEK** | Semestral (staging) | DEK recifradas, dados intactos, aprovação dupla (T-CER-01). |
| **Cerimônia: recuperação de CA raiz** | Anual (staging) | *Cross-signing* funciona; mTLS restabelecido (T-CER-02). |
| Rotação da credencial de bootstrap | Semestral | Executada sem indisponibilidade; documentada. |

As duas cerimônias são ensaiadas em `staging` porque o custo de descobrir um passo
faltante durante um comprometimento real é inaceitável.

---

## 7. Degradação Graciosa — Ordem de Sacrifício

```
   1º  Operações administrativas (criar princípio, publicar perfil, federação)
   2º  Rotação programada de credenciais
   3º  Emissão de credenciais novas
   4º  Introspecção centralizada (clientes caem para validação local)
   ─────────────────── nunca sacrificado ───────────────────
   ·  Validação de tokens já emitidos (é local — não depende do 021)
   ·  mTLS com certificados já emitidos
   ·  Propagação de revogação (se necessário, por consulta direta à CRL)
```

A quarta linha e a última merecem nota conjunta: quando tudo mais falha, a **revogação**
continua sendo a operação prioritária — porque é a única capaz de conter um
comprometimento em andamento. É por isso que existe o caminho de fallback por consulta
direta à CRL, independente do barramento.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §9
- Alertas e runbooks: `./Monitoring.md` · `../029-Operations/`
- Testes de caos e cerimônias: `./Testing.md` §10
- Erros: `./API.md` §5 · Escala: `./Scalability.md`
