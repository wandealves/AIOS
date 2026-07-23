---
Documento: RFC
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0008, ADR-0210, ADR-0211, ADR-0212, ADR-0214, ADR-0215
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: 003-RFC, _DESIGN_BRIEF.md §11
---

# 021-Security — Índice de RFCs

RFCs são registradas em `../003-RFC/`. Este documento lista as que afetam o módulo e o
impacto de cada uma.

---

## 1. RFCs Consumidas (baseline)

| RFC | Título | Status | Impacto no 021-Security |
|-----|--------|--------|-------------------------|
| **RFC-0001** | Architecture Baseline & Core Contracts | Accepted | **Baseline herdada, não redefinida.** A **§6 (Considerações de Segurança)** é o contrato que este módulo torna executável: mTLS obrigatório entre serviços, tokens validados no Gateway com chaves publicadas por `021`, `tenant` como fronteira de isolamento e erros sem dados sensíveis. Além dela: URN (§5.1) para princípios, credenciais, segredos e chaves; envelope CloudEvents (§5.2–§5.3) para o domínio `security`; envelope de erro (§5.4) para `SEC` e `AUTHN`; idempotência (§5.5) em toda emissão; correlação (§5.6); PEP/PDP com *default deny* (§5.8); minimização e retenção de dados pessoais (§7); registros IANA-like (§8). |

O módulo **não reescreve** nenhum desses contratos. A RFC-0001 §6 diz *que* o mTLS é
obrigatório e *que* os tokens são validados no Gateway; este módulo especifica **como o
material que sustenta essas exigências é emitido, rotacionado e revogado**.

---

## 2. RFCs a Propor por Este Módulo

| RFC | Título | Escopo normativo | Relação com RFC-0001 | Status |
|-----|--------|------------------|----------------------|--------|
| **RFC-0210** | AIOS Identity & Credential Contract | Modelo de princípio e **atributos assertáveis** (a fronteira com o PDP); ciclo de vida unificado de credencial (`CredentialState`, T-01..T-09, invariantes I1..I6); política de TTL por tipo; semântica de rotação por sobreposição; formato e propagação da lista de revogação; contrato de *lease* de segredo. | Especializa §6 para o plano de identidade; consome §5.1, §5.4, §5.5 e §5.8 sem redefini-los. | Proposed |
| **RFC-0211** | AIOS Workload Identity & PKI Profile | Protocolo de atestação (o que é atributo verificável e o que é autodeclaração); formato do certificado de workload (extensões, `purpose`, TTL); hierarquia de CA (raiz *offline* → intermediárias → workload); *cross-signing* na recuperação de raiz; formato do perfil de sandbox assinado consumido pelo `../007-Agent-Runtime/`. | Torna executável a exigência de mTLS interno da §6. | Proposed |

---

## 3. Registros Alimentados por Este Módulo

Conforme RFC-0001 §8, os registros centrais são mantidos por `../004-API/`. O 021
contribui com:

| Registro | Contribuição do 021 | Documento |
|----------|---------------------|-----------|
| Domínios de subject NATS | Domínio **`security`** (ADR-0210) — **já consumido de fato** por `../004-API/`, `../010-Memory/` e `../020-Communication/`. | `./Events.md` |
| Códigos de erro | Domínios **`SEC`** (0001–0099) e **`AUTHN`** (0001–0099). | `./API.md` §5 |
| Versões de `dataschema` | `security.jwks.rotated/1`, `security.ca.rotated/1`, `security.sandboxprofile.published/1`, `security.credential.issued/1`, `security.credential.rotating/1`, `security.token.revoked/1`, `security.credential.expiring/1`, `security.principal.disabled/1`, `security.authn.anomaly/1`, `security.attestation.failed/1`, `security.rtbf.requested/1`, `security.federation.suspended/1`. | `./Events.md` §3 |
| Tipos de recurso URN | `principal`, `credential`, `secret`, `key`, `sandboxprofile`, `federationtrust`, `rtbfrequest` (sob avaliação da arquitetura-chefe). | `./Database.md` |

> **Ponto que exige ratificação — e que já se acumula.** A RFC-0001 §5.1 enumera os
> tipos de URN vigentes (`agent`, `task`, `memory`, `tool`, `model`, `plan`, `workflow`,
> `policy`, `event`) e a §5.3 enumera os domínios de subject. Este é o **terceiro**
> módulo consecutivo a precisar de valores novos:
>
> | Módulo | Domínio de subject | Tipos de URN novos |
> |--------|--------------------|--------------------|
> | `../005-Database/` | `database` | `migration`, `schemaobject`, `erasurerequest`, `backup` |
> | `../020-Communication/` | `comm` | `subjectdef`, `a2asession`, `groupchannel` |
> | **021-Security** | **`security`** | `principal`, `credential`, `secret`, `key`, `sandboxprofile`, `federationtrust`, `rtbfrequest` |
>
> O caso do `security` é o mais urgente: seus subjects **já são consumidos** por três
> módulos documentados. Recomenda-se que a arquitetura-chefe decida os três domínios e
> os catorze tipos em **uma única revisão** da RFC-0001, em vez de por PRs isolados —
> caso contrário, a enumeração da §5.1 tende a ficar permanentemente defasada em relação
> ao que os módulos efetivamente usam.

---

## 4. Relação com RFCs de Outros Módulos

| RFC | Módulo | Interação |
|-----|--------|-----------|
| RFC-0050 (Data Platform Contract) | `../005-Database/` | Define as convenções do schema `security` e o contrato de `outbox`; o 021 emite as *roles* que operam sob a RLS ali definida. |
| RFC-0051 (Migration Protocol) | `../005-Database/` | Migrações do schema `security` seguem o mesmo pipeline governado. |
| RFC-0200 (Bus Contract) | `../020-Communication/` | Os streams `SEC_*` são declarados sob esse contrato; o 021 emite as contas NKey/JWT que o barramento usa. |
| RFC-0201 (A2A Profile) | `../020-Communication/` | As âncoras de confiança de pares externos vivem em `security.federation_trust`; a revogação da âncora encerra as sessões. |
| RFC-0071 (MCP Host Sandboxing Profile) | `../007-Agent-Runtime/` | **Acoplamento a resolver:** o perfil de sandbox definido em RFC-0211 e o perfil de *sandboxing* de MCP host da RFC-0071 precisam ser o **mesmo artefato**, ou a hierarquia entre eles deve ser explicitada. Recomenda-se decidir isso antes de ambas serem aceitas. |
| RFC-0006 (Cognitive Syscall ABI) | `../006-Kernel/` | O Kernel consome claims já validadas; a revogação de credencial de agente encerra suas syscalls. |

O item da RFC-0071 é uma **dependência aberta** identificada durante a escrita deste
módulo: duas RFCs propostas por módulos diferentes descrevem material de isolamento com
escopos que se sobrepõem.

---

## 5. Processo

1. Uma especificação de contrato/protocolo **NÃO DEVE** ser criada dentro dos 26
   documentos: vira RFC em `../003-RFC/` e é referenciada aqui.
2. RFCs deste módulo usam a numeração alinhada à faixa de ADR (`RFC-0210`, `RFC-0211`, …).
3. Alteração incompatível em RFC publicada exige nova RFC que a *obsolete*, com período
   de coexistência (RFC-0001 §9).
4. **Específico deste módulo:** alterações que afrouxem garantias criptográficas
   (suítes, tamanhos de chave, TTLs máximos) exigem RFC com análise de risco, ainda que
   pareçam ajustes menores — o custo de reverter uma suíte depreciada em produção é alto.

---

## 6. Referências

- Baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§6 em especial)
- Decisões: `./ADR.md` · `../002-ADR/README.md`
- Registros centrais: `../004-API/`
- Brief: `./_DESIGN_BRIEF.md` §11
