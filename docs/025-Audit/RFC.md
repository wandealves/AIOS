---
Documento: RFC
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0250, ADR-0251, ADR-0253, ADR-0254, ADR-0255, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: 003-RFC, _DESIGN_BRIEF.md §11
---

# 025-Audit — Índice de RFCs

RFCs são registradas em `../003-RFC/`. Este documento lista as que afetam o módulo e o
impacto de cada uma.

---

## 1. RFCs consumidas (baseline)

| RFC | Título | Status | Impacto no 025-Audit |
|-----|--------|--------|----------------------|
| **RFC-0001** | [Architecture Baseline & Core Contracts](../003-RFC/RFC-0001-Architecture-Baseline.md) | Accepted | **Baseline herdada, não redefinida.** A **§5.8** é o contrato que este módulo torna executável: *toda ação privilegiada e toda decisão de agente DEVEM emitir registro de auditoria imutável (025)*. Além dela: URN (§5.1) para registros, selos, *holds* e comprovantes; envelope CloudEvents (§5.2–§5.3) para o domínio `audit`; envelope de erro (§5.4) para o domínio `AUD`; **idempotência (§5.5)**, que sustenta o dedupe por `event_id`; correlação (§5.6), preservada do fato original no registro; versionamento (§5.7); mTLS e fronteira de tenant (§6); **privacidade (§7)**, que motiva o apagamento criptográfico; registros IANA-like (§8). |

O módulo **não reescreve** nenhum desses contratos. A RFC-0001 §5.8 diz *que* o
registro imutável é obrigatório; este módulo especifica **como a imutabilidade é
construída, provada, retida e conciliada com o direito ao esquecimento**.

---

## 2. RFCs a propor por este módulo

| RFC | Título | Escopo normativo | Relação com RFC-0001 | Status |
|-----|--------|------------------|----------------------|--------|
| **RFC-0250** | AIOS Audit Record & Chain Integrity Contract | **Serialização canônica** do registro (normativa, para que o auditor recompute o mesmo `record_hash`); envelope canônico de auditoria (`actor`, `action`, `resource`, `outcome`, `decision_ref`, `subject_ref`); algoritmo de encadeamento (`prev_hash`) e particionamento da cadeia; construção da árvore de Merkle e formato do selo assinado; formato da **prova de inclusão**; cadeia de selos (`prev_seal_hash`); formato do **pacote probatório** e o procedimento de **verificação offline**; vetor de teste canônico. | Especializa §5.8 para o plano de auditoria; consome §5.1, §5.4, §5.5 e §5.6 sem redefini-los. | Proposed |
| **RFC-0251** | AIOS Retention, Legal Hold & Crypto-Erasure | Classes auditáveis (`record_class`) com retenção, base legal e volume esperado; semântica do *legal hold* e sua **precedência** sobre retenção e RTBF; protocolo de **apagamento criptográfico** (cifragem por titular, destruição de chave, o que sobrevive); formato e verificação do **comprovante**; o que permanece em `Shredded`/`Expired`; regras de arquivamento WORM e *object lock*. | Torna executável a §7 (privacidade) no contexto de uma trilha imutável. | Proposed |

### 2.1 A cláusula mais rígida da RFC-0250

A **serialização canônica** do `record_hash` é o único contrato do AIOS que não pode
mudar por evolução aditiva:

> Alterá-la invalidaria a verificação de **todo o histórico já registrado**. Uma nova
> versão exige (a) nova versão da RFC-0250, (b) um período em que **ambas** as formas
> sejam verificáveis, e (c) um selo de transição que ancore a mudança. Um vetor de
> teste fixo é validado no CI a cada build (`./Testing.md` §9): se o hash esperado
> mudar, o build para.

---

## 3. RFCs de outros módulos que este módulo consome

| RFC | Módulo | Por que importa aqui |
|-----|--------|----------------------|
| RFC-0210 | `../021-Security/` | Identidade dos produtores e atributos do `actor`; princípios que assinam *holds* e exportações. |
| RFC-0211 | `../021-Security/` | Identidade de workload e PKI: sustenta o mTLS dos produtores e, sobretudo, a **chave de assinatura de selo**, que não reside neste módulo. |
| RFC-0220 | `../022-Policy/` | Contrato de decisão: o `decision_ref` do registro aponta para o `decision_id` do journal do 022. |
| RFC-0240 | `../024-Observability/` | Convenções de sinal: as métricas `aios_aud_*` são registradas lá como `SignalDescriptor`. |
| RFC-0200 / RFC-0201 | `../020-Communication/` | Contrato do barramento: streams `AUDIT_*` e a semântica de `ACK` que sustenta o RPO 0. |

> A dependência de RFC-0211 é a mais crítica: se a chave de assinatura de selo vivesse
> neste módulo, comprometer o 025 bastaria para forjar selos e a defesa em profundidade
> de `./Security.md` §3 perderia uma camada inteira.

---

## 4. Relação com padrões externos

| Padrão | Uso | Grau de aderência |
|--------|-----|-------------------|
| **RFC 6962** (Certificate Transparency) | Inspiração para a árvore de Merkle, prova de inclusão e cadeia de selos. | **Parcial**: adota-se a estrutura de prova; não se adota o modelo de log público nem o de monitores independentes. |
| **RFC 3161** (Time-Stamp Protocol) | Modelo de âncora temporal. | **Opcional**: `aud.seal.external_anchor_enabled` permite ancorar o `seal_hash` em serviço de terceiro quando o contrato exigir. |
| **S3 Object Lock** (modo *compliance*) | Arquivamento WORM. | Total — é a terceira fonte de verificação. |
| **CloudEvents 1.0** | Envelope dos eventos do módulo. | Total, via RFC-0001 §5.2. |
| **RFC 7807** | Envelope de erro. | Total, via RFC-0001 §5.4. |
| **W3C Trace Context** | Correlação; o `traceparent` do **fato original** é preservado no registro. | Total, via RFC-0001 §5.6. |
| **LGPD / GDPR** | Retenção com base legal, *legal hold*, direito ao esquecimento por apagamento criptográfico. | Total no que é técnico; a interpretação jurídica é do DPO do tenant. |

A aderência **parcial** ao RFC 6962 é deliberada: Certificate Transparency assume um
log **público** com monitores independentes verificando continuamente. Aqui o log é
privado e multi-tenant; o papel dos monitores é cumprido pelo `IntegrityVerifier`
(interno), pela cópia WORM (independente do banco) e pelo **auditor externo com o
pacote probatório** (independente do AIOS inteiro).

---

## 5. Registro e evolução

- A numeração `RFC-0250`/`RFC-0251` segue a convenção de faixa por módulo já adotada
  por `../020-Communication/` (RFC-0200/0201), `../021-Security/` (RFC-0210/0211),
  `../022-Policy/` (RFC-0220/0221) e `../024-Observability/` (RFC-0240/0241).
- Novos `record_class`, `outcome`, `anomaly.kind` e `erasure.kind` são **aditivos** e
  registrados nas RFCs correspondentes.
- **Exceção absoluta**: a serialização canônica do `record_hash` (§2.1) e o formato do
  selo não evoluem de forma aditiva — mudança exige nova versão de RFC, coexistência
  verificável e selo de transição.
- Alteração incompatível do formato do **pacote probatório** exige que o validador de
  referência continue aceitando pacotes da versão anterior: um auditor pode voltar a
  verificar um pacote emitido anos antes.
- Nenhuma RFC deste módulo **PODE** contradizer a RFC-0001; alteração da baseline exige
  RFC que a obsolete, com período de coexistência.

*Fim de `RFC.md`.*
