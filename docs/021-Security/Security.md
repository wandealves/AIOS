---
Documento: Security
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0008, ADR-0010 (globais); ADR-0210, ADR-0211, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: _DESIGN_BRIEF.md §12, 022-Policy, 025-Audit, 004-API, 007-Agent-Runtime
---

# 021-Security — Segurança

Este módulo **é** o subsistema de segurança do AIOS; este documento trata da segurança
**do próprio módulo** — a superfície mais crítica do sistema, porque comprometê-la
compromete todo o resto.

Contratos centrais (mTLS interno, validação de token no Gateway, `tenant` como
fronteira, minimização) são normativos na
`../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7 e **não são redefinidos aqui**.

---

## 1. Autenticação do próprio módulo

| Ator | Mecanismo | Exigência adicional |
|------|-----------|---------------------|
| Operador humano | OIDC do próprio módulo | **MFA obrigatório** (`sec.authn.mfa_required_for_human`) |
| Serviço interno | **mTLS** com certificado de workload | Certificado emitido após atestação; TTL ≤ 24 h |
| `aios-security-ca` ↔ `aios-security-svc` | mTLS em rede isolada | Único caminho de entrada da CA |
| Operação sobre CA raiz / KEK | Presencial ou por cerimônia | **Aprovação dupla** + separação de funções |

Um detalhe importante: o módulo autentica os operadores **com o próprio IdP que
opera**. Isso é aceitável porque a alternativa (um segundo sistema de identidade só
para administrar o primeiro) apenas moveria o problema. O controle compensatório é a
exigência de MFA, de aprovação dupla para operações sobre material de raiz e de
auditoria integral.

---

## 2. Autorização (o módulo como PEP de si mesmo)

O 021 **NÃO** decide sobre si mesmo: toda operação administrativa monta um
`DecisionRequest` e consulta o **PDP** do `../022-Policy/`, com ***default deny***
(RFC-0001 §5.8).

| Capability | Operações | Sensibilidade |
|------------|-----------|---------------|
| `sec:principal:read` / `:write` | Ler / criar / atualizar princípios | baixa / média |
| `sec:principal:disable` | Suspender / desabilitar (revogação em cascata) | **alta** |
| `sec:credential:issue` | Emitir credencial | **alta** |
| `sec:credential:rotate` | Rotacionar | média |
| `sec:credential:revoke` | Revogar | **alta** |
| `sec:credential:read` / `sec:revocation:read` | Metadados / CRL | baixa |
| `sec:secret:read` / `:write` | Ler / escrever segredo | **alta** |
| `sec:cert:issue` | Emitir certificado de workload | **alta** |
| `sec:key:manage` | Criar / rotacionar chave | **crítica** (aprovação dupla para KEK e CA) |
| `sec:sandbox:publish` | Publicar perfil de isolamento | **crítica** |
| `sec:federation:write` | Registrar / suspender confiança externa | **crítica** |
| `sec:crypto:use` | Assinar / verificar | média |

**Separação de funções (normativa):** o princípio que administra `sec:principal:*`
**NÃO DEVERIA** ter `sec:key:manage`. Concentrar ambos permitiria criar um princípio,
conceder-lhe capacidades e emitir-lhe credenciais sem qualquer segundo par de olhos.

**Fail-closed seletivo:** com o PDP indisponível, operações administrativas são
negadas (`AIOS-SEC-0002`), mas **autenticação e validação de credenciais existentes
continuam**. A alternativa converteria uma falha do serviço de política em queda total
do AIOS.

---

## 3. Proteção do Material Criptográfico

```
   ┌──────────────────────────────────────────────────────────────┐
   │ CA RAIZ · offline · HSM ou custódia física                   │  ← sem rede
   │  assina apenas intermediárias, em cerimônia                  │
   └────────────────────────┬─────────────────────────────────────┘
                            │ (plurianual)
   ┌────────────────────────▼─────────────────────────────────────┐
   │ CA INTERMEDIÁRIA · aios-security-ca · rede isolada           │
   │  emite certificados de workload (TTL ≤ 24 h)                 │
   └────────────────────────┬─────────────────────────────────────┘
                            │ (diária, automática)
   ┌────────────────────────▼─────────────────────────────────────┐
   │ CERTIFICADOS DE WORKLOAD · efêmeros · por serviço            │
   └──────────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────┐
   │ KEK (HSM) ──envolve──▶ DEK ──cifra──▶ segredos e material    │
   │  rotação da KEK recifra as DEK, NÃO os dados (ADR-0216)      │
   └──────────────────────────────────────────────────────────────┘
```

Regras normativas:

1. Material privado **nunca** é gravado em claro (`./Database.md` §0).
2. Material entregue ao solicitante **não é recuperável** (invariante I5) — perdeu,
   reemita.
3. Chave `retired` **não** é destruída enquanto houver artefato válido dependente.
4. Um dump do banco `security` **não** concede acesso a nada: os segredos estão
   envolvidos por DEK, que estão envolvidas por uma KEK que não está no backup.

---

## 4. Isolamento Multi-Tenant

RLS por `tenant_id` em `principal`, `credential`, `secret`, `revocation` e
`federation_trust` (`./Database.md`). `tenant` divergente do contexto autenticado →
`AIOS-SEC-0003`.

Uma consequência importante: a **federação é por tenant**. Um IdP registrado por
`acme` não autentica princípios de `globex`, e a suspensão da confiança afeta apenas o
tenant que a registrou.

---

## 5. Superfície de Ataque

| Superfície | Exposição | Controle |
|------------|-----------|----------|
| `/oauth2/*`, `/.well-known/*` | **Pública** (via `004-API`) | Rate-limit no Gateway; contenção por princípio; mensagens não-oráculo; nenhuma operação de escrita além de token. |
| `/v1/security/*` | Interna | mTLS + PEP/PDP + auditoria integral. |
| gRPC `aios.security.v1` | Malha interna | mTLS + PEP. |
| `aios-security-ca` (gRPC) | **Rede isolada** | Alcançável apenas pelo `security-svc`. |
| HSM / KMS | Rede segregada | PKCS#11/TLS; credencial dedicada. |
| Schema `security` no PostgreSQL | Interna | *Role* exclusiva; nenhum outro módulo tem acesso. |
| Endpoint de IdP externo (federação) | Saída para terceiro | `issuer` registrado, `jwks_uri` validado, escopo limitado. |
| CA raiz | **Nenhuma** | Offline. |

A superfície pública é deliberadamente mínima: apenas os endpoints OIDC padronizados,
sem nenhuma operação administrativa alcançável a partir da borda.

---

## 6. Threat Model — STRIDE

| Ameaça | Vetor no 021-Security | Mitigação | Verificação |
|--------|------------------------|-----------|-------------|
| **S**poofing | Contêiner comprometido pede a credencial de outro serviço (ex.: a *role* de banco de outro schema) fingindo ser ele. | **Atestação obrigatória** antes da emissão (`AIOS-SEC-0006`), comparando serviço declarado × observado; `purpose` verificado; TTL ≤ 24 h limita o dano. | T-ATT-01 |
| **T**ampering | Alteração de perfil de sandbox para afrouxar isolamento; adulteração de token; mudança de `attributes` do princípio para ganhar privilégio. | Perfis **assinados** e imutáveis por versão, verificados pelo `../007-Agent-Runtime/` (NFR-015); JWT assinado; mutação de `attributes` sob capability e auditada; OCC em tudo. | T-PROF-02, T-LEAK-01 |
| **R**epudiation | Negar ter emitido, lido segredo ou revogado credencial. | `issued_by`, `fingerprint`, motivo obrigatório na revogação (`ck_reason_when_revoked`), trilha de **toda leitura de segredo** em `../025-Audit/`. | T-PEP-01 |
| **I**nformation disclosure | Vazamento de material em log/evento/métrica/resposta; **enumeração de contas** por mensagem ou tempo de resposta; dump de banco. | Nenhum material em claro (envelope); varredura contínua (FR-018); erros de AuthN **não-oráculo** com tempo constante; métricas sem `principal` como label; backup inútil sem a KEK. | T-LEAK-01, T-OIDC-02 |
| **D**enial of service | Força bruta contra um princípio; inundação de emissões; ataque ao KMS derrubando o AIOS. | Contenção **por princípio** (nunca global — do contrário, atacar uma conta negaria serviço a todas); `max_active_per_principal`; **validação local no `004` mantém o sistema operando mesmo com o 021 fora**. | T-THR-01 |
| **E**levation of privilege | Identidade federada ganhando escopo interno; operador criando princípio e emitindo-lhe credencial privilegiada sem revisão; abuso da CA. | `max_scope_expansion = 0` por default (ADR-0218); **separação de funções** entre administrar princípios e administrar chaves; aprovação dupla para KEK/raiz; toda emissão pelo PDP. | T-FED-02, T-PEP-01 |

---

## 7. Controles Específicos

### 7.1 Contra enumeração de contas
`AIOS-AUTHN-0001` é idêntico para "princípio inexistente" e "credencial incorreta", com
**tempo de resposta constante** (a verificação de senha é executada mesmo quando o
princípio não existe, contra um *hash* fictício). O motivo real vai apenas para
auditoria e para o `ThreatSignalDetector`.

### 7.2 Contra credenciais de longa vida
`expires_at NOT NULL` no schema, teto de TTL por tipo (`AIOS-SEC-0004`) e métrica
contínua `aios_sec_credentials_without_expiry` — que **deve ser sempre zero**. Não é
auditoria anual: uma credencial eterna criada hoje aparece hoje.

### 7.3 Contra uso indevido de material vazado
TTL curto (mediano ≤ 24 h) + revogação propagada em ≤ 30 s + `fingerprint` em toda
trilha, permitindo identificar **qual** credencial vazou sem que a trilha contenha o
material.

### 7.4 Contra escalada por configuração
Chaves críticas são **não-recarregáveis** (`sec.attestation.required`,
`sec.key.hsm_enabled`, `sec.key.kek_rotation_days`): alterá-las exige reinício, que é
um evento visível. Toda mudança de configuração do módulo é auditada, inclusive as
recarregáveis (`./Configuration.md` §3).

### 7.5 Contra o próprio operador
Separação de funções, aprovação dupla para material de raiz e auditoria integral. Nenhum
operador único consegue, sozinho, criar um princípio, conceder-lhe privilégio e emitir
credencial sem deixar trilha revisável.

---

## 8. LGPD / GDPR

| Requisito legal | Implementação |
|-----------------|---------------|
| **Minimização** | O módulo guarda **identificadores e credenciais**, não conteúdo pessoal de negócio. `display_name` e `external_ref` são o mínimo para operar identidade. |
| **Classificação** | `security.principal` é a única tabela `pii` do módulo e declara `legal_basis` (execução de contrato + obrigação legal de registro de acesso), conforme CV-08 do `../005-Database/`. |
| **Retenção** | Credenciais: 1 ano após `expires_at`. Princípios: enquanto ativos + 5 anos por obrigação legal. Trilha de acesso a segredo: conforme `../025-Audit/`. |
| **Direito ao esquecimento** | O módulo é o **gatilho**: desabilita o princípio (revogação em cascata) e emite `security.rtbf.requested` para `../010-Memory/`, `../005-Database/` e `../025-Audit/` (UC-015). O expurgo do dado de negócio é dos módulos donos. |
| **Limite do esquecimento** | Registros exigidos por obrigação legal (quem acessou o quê e quando) **NÃO DEVEM** ser apagados; são tokenizados e o conflito é registrado, nunca resolvido em silêncio. |
| **Trilha de acesso a segredo** | Toda leitura registra quem, quando, qual caminho e qual lease — **nunca** o valor lido. |
| **Segregação** | RLS por tenant; chaves de tenant sob KEK distintas quando o contrato exigir separação criptográfica. |
| **Transferência internacional** | IdP em outra jurisdição fica registrado em `federation_trust` com escopo explícito, permitindo avaliação de adequação por tenant. |
| **RTBF não se aplica a workload** | Pedido de esquecimento para princípio `service`/`agent` é recusado (UC-015 E2): não há titular pessoal. |

---

## 9. Auditoria

Toda operação deste módulo é auditada em `../025-Audit/`, com no mínimo: `trace_id`,
identidade do chamador, capability avaliada, decisão do PDP, recurso alvo, resultado e
duração. Adicionalmente, e de forma específica:

- **Emissão**: `principal_urn`, `kind`, `purpose`, `fingerprint`, TTL concedido,
  resultado da atestação.
- **Revogação**: motivo obrigatório, autor, horário efetivo, tempo de propagação.
- **Leitura de segredo**: caminho, versão, lease — **nunca** o valor.
- **Operações de chave**: papel, algoritmo, aprovadores (duplos quando aplicável).
- **Publicação de perfil**: nome, versão, diferença em relação à versão anterior.
- **Mudança de configuração**: chave, valor anterior, valor novo, autor.
- **Falhas de atestação e anomalias de autenticação**: origem, divergência observada.

A trilha é a única forma de responder, depois de um incidente, a "que credenciais esse
princípio tinha e quando as perdeu" — e é por isso que a revogação sem motivo é
impedida por *constraint* no banco, não apenas por convenção.

---

## 10. Referências

- Brief: `./_DESIGN_BRIEF.md` §12
- Contratos de segurança e privacidade: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7
- Política: `../022-Policy/` · Auditoria: `../025-Audit/` · Runtime: `../007-Agent-Runtime/`
- Modelo físico: `./Database.md` · Testes de segurança: `./Testing.md` §8
- Cerimônias: `../029-Operations/`
