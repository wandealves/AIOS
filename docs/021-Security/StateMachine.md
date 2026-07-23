---
Documento: StateMachine
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0212, ADR-0213, ADR-0216
RFCs relacionados: RFC-0001, RFC-0210
Depende de: _DESIGN_BRIEF.md §4, API.md, Events.md, Database.md
---

# 021-Security — Máquinas de Estado

A entidade com ciclo de vida governado é a **credencial** (`security.credential`), e o
modelo é deliberadamente **unificado**: *access token*, certificado de workload, *role*
de banco e conta NATS percorrem os mesmos estados (ADR-0212). Isso dá uma política de
rotação, uma trilha e um caminho de revogação — em vez de quatro subsistemas com
comportamentos sutilmente diferentes, um dos quais inevitavelmente esqueceria de
implementar a revogação.

`security.key` tem ciclo próprio (§5); `security.principal` tem um ciclo simples (§6).

Reprodução detalhada de `./_DESIGN_BRIEF.md` §4; divergência é defeito deste documento.

---

## 1. Estados (`CredentialState`)

| Estado | Descrição | Ação de entrada | Terminal? |
|--------|-----------|-----------------|-----------|
| `Requested` | Emissão solicitada; atestação e autorização em curso. | Registra `principal_urn`, `kind`, `purpose`, TTL pedido. | não |
| `Issued` | Material gerado e entregue **uma única vez**; antes de `not_before`. | Grava `fingerprint`, `material_ref` cifrado, `expires_at`. | não |
| `Active` | Válida e utilizável. | Emite `security.credential.issued`. | não |
| `Rotating` | Sucessora emitida; ambas válidas na janela de sobreposição. | Grava `rotated_to`; emite `credential.rotating`. | não |
| `Superseded` | Substituída; não deve mais ser usada. | Encerra a sobreposição. | **sim** |
| `Revoked` | Invalidada antes do prazo. | Grava `security.revocation`; emite `token.revoked`. | **sim** |
| `Expired` | Prazo natural encerrado. | Elegível a expurgo conforme retenção. | **sim** |
| `Failed` | Emissão não concluída. | Grava causa; emite `attestation.failed` quando aplicável. | **sim** |

---

## 2. Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação de saída |
|---|-----------|---------|------------------------------|----------------|
| T-01 | ∅ → `Requested` | `IssueCredential` | Capability `sec:credential:issue` ∧ princípio `active` ∧ `purpose` declarado ∧ TTL ≤ máximo do tipo. | Persiste a solicitação. |
| T-02 | `Requested` → `Issued` | atestação e autorização concluídas | `WorkloadAttestor` confirmou a identidade ∧ PDP `allow` ∧ material gerado e cifrado. | Entrega o material (uma vez). |
| T-03 | `Requested` → `Failed` | atestação/autorização negada, ou erro de KMS | Atestação falhou ∨ PDP negou ∨ KMS/HSM indisponível. | `AIOS-SEC-0006`/`-0002`/`-0010`. |
| T-04 | `Issued` → `Active` | `not_before` alcançado | Relógio ≥ `not_before`. | Emite `credential.issued`. |
| T-05 | `Active` → `Rotating` | `RotateCredential` | Sucessora em `Issued`/`Active` ∧ janela de sobreposição > 0. | Emite `credential.rotating`. |
| T-06 | `Rotating` → `Superseded` | fim da janela | Sucessora `Active` ∧ janela decorrida ∧ sem uso recente da antiga. | — |
| T-07 | `Issued`/`Active`/`Rotating` → `Revoked` | `RevokeCredential` ∨ princípio desabilitado ∨ suspeita de comprometimento | Capability `sec:credential:revoke` ∨ cascata de `principal.disable`. | Grava revogação + evento (I4). |
| T-08 | `Issued`/`Active`/`Rotating` → `Expired` | `expires_at` alcançado | Relógio ≥ `expires_at`. | — |
| T-09 | `Superseded` → `Revoked` | revogação retroativa | Comprometimento identificado após a substituição. | Grava revogação + evento. |

Gatilho recebido em estado que não admite a transição retorna `AIOS-SEC-0005` e
**NÃO DEVE** alterar o estado.

---

## 3. Invariantes

| # | Invariante | Como é garantido | Violação |
|---|-----------|------------------|----------|
| I1 | Toda credencial em estado não-terminal tem `expires_at` no futuro. **Não existe credencial sem expiração.** | `NOT NULL` no schema + guarda de T-01. | `AIOS-SEC-0004`; métrica `aios_sec_credentials_without_expiry` = 0 |
| I2 | Em `Rotating`, **exatamente duas** credenciais do mesmo `purpose`+`principal` são utilizáveis. | *Advisory lock* por `purpose`+`principal` na rotação. | Três utilizáveis ⇒ incidente P1 |
| I3 | `Revoked` **NÃO DEVE** transitar para nenhum estado. A revogação é **irreversível**. | Guarda de transição. | `AIOS-SEC-0005` |
| I4 | Toda transição para `Revoked` grava `security.revocation` **e** o evento `token.revoked` na **mesma transação**. | Outbox transacional. | Revogação sem propagação ⇒ defeito grave |
| I5 | O material privado **nunca** é relido após a emissão: guarda-se referência cifrada e `fingerprint`. | Ausência de caminho de leitura na API. | Exposição ⇒ incidente P1 (NFR-008) |
| I6 | Transição só ocorre com o `version` esperado (OCC). | Camada de persistência. | `AIOS-SEC-0009` |

A invariante **I3** merece justificativa. Permitir "desrevogar" pareceria conveniente
em um incidente com falso positivo, mas criaria um caminho de restauração de acesso que
não passa por emissão — ou seja, sem atestação, sem nova decisão do PDP e com um
registro de auditoria ambíguo. Reemitir custa segundos e preserva toda a cadeia de
controle.

---

## 4. Diagrama de estados (ASCII)

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
        │  revogação (T-07)                             │ retroativa (T-09)
        ├───────────────────────────┐                   │
        │  expiração (T-08)         ▼                   ▼
        │                    ┌────────────┐      ┌────────────┐
        └───────────────────▶│  Expired   │      │  Revoked   │ (terminal, irreversível)
                             │ (terminal) │      └────────────┘
                             └────────────┘
```

---

## 5. Ciclo de `KeyMaterial`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `pending` | Gerada, ainda não em uso. | `active` |
| `active` | Em uso para **assinar/cifrar**. | `rotating` |
| `rotating` | Sucessora ativa; esta **apenas verifica/decifra**. | `retired` |
| `retired` | Não assina nem cifra; permanece para verificar assinaturas antigas. | `destroyed` |
| `destroyed` | Material destruído com segurança. | — (terminal) |

```
   pending ──ativa──▶ active ──rotaciona──▶ rotating ──retire_after──▶ retired
                                                                          │
                          (só quando nenhum artefato válido depende dela) │
                                                                          ▼
                                                                     destroyed
```

**Regra normativa:** uma chave de assinatura **NÃO DEVE** ser destruída enquanto
existir artefato válido assinado por ela — token, perfil de sandbox ou certificado. Por
isso `retired` é um estado longo (`sec.key.retire_grace_days` = 30) e `destroyed` é
decisão deliberada, jamais automática. Destruir cedo transforma artefatos legítimos em
inverificáveis, o que é uma negação de serviço autoinfligida.

---

## 6. Ciclo de `Principal`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `active` | Pode autenticar e receber credenciais. | `suspended`, `disabled` |
| `suspended` | Temporariamente impedido (contenção, investigação). Credenciais existentes **permanecem** válidas até expirar. | `active`, `disabled` |
| `disabled` | Encerrado. Dispara **revogação em cascata** de todas as credenciais. | — (terminal para fins operacionais) |

```
   active ──suspende──▶ suspended ──reativa──▶ active
      │                     │
      └──── desabilita ─────┴────▶ disabled ──▶ revoga em cascata (T-07 × N)
                                      │
                                      └─▶ evento security.principal.disabled
```

A diferença entre `suspended` e `disabled` é intencional: suspensão é medida
**reversível** de contenção (o princípio para de obter credenciais novas), enquanto
desabilitação é **encerramento** e invalida tudo que já foi emitido.

---

## 7. Mapa Estado → Evento → Erro

| Estado alcançado | Evento emitido | Erro típico ao tentar sair indevidamente |
|------------------|----------------|------------------------------------------|
| `Active` (credencial) | `aios.<tenant>.security.credential.issued` | — |
| `Rotating` | `aios.<tenant>.security.credential.rotating` | `AIOS-SEC-0005` |
| `Revoked` | `aios.<tenant>.security.token.revoked` | `AIOS-SEC-0005` (não há "desrevogar") |
| `Failed` (atestação) | `aios.<tenant>.security.attestation.failed` | `AIOS-SEC-0006` |
| `disabled` (princípio) | `aios.<tenant>.security.principal.disabled` | `AIOS-SEC-0008` em qualquer uso |
| `active` (chave, após rotação) | `aios._platform.security.jwks.rotated` | — |
| — (perfil publicado) | `aios._platform.security.sandboxprofile.published` | `AIOS-SEC-0012` ao tentar alterar |

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §4
- API e catálogo de erros: `./API.md` · Eventos: `./Events.md`
- Casos de uso UC-003..UC-007, UC-009, UC-010: `./UseCases.md`
- Sequências correspondentes: `./SequenceDiagrams.md`
