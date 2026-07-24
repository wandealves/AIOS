---
Documento: Security
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy + CISO
ADRs relacionados: ADR-0008, ADR-0010, ADR-0221, ADR-0224, ADR-0225, ADR-0227, ADR-0228
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §12, 021-Security, 025-Audit, 005-Database
---

# 022-Policy — Segurança

> Este módulo **decide** a segurança de todo o resto. Sua superfície de ataque não é o
> dado que guarda — é a **decisão que emite**: comprometer o 022 não vaza segredo,
> **concede acesso**.

## 1. Superfície de ataque

| Superfície | Exposição | Controle primário |
|-----------|-----------|-------------------|
| `Evaluate` (gRPC/NATS/REST) | Todos os PEPs internos | mTLS + validação de tenant + limite por tenant |
| Autoria e publicação de bundle | Operadores do tenant | Autogoverno (§2.3), aprovação dupla, assinatura |
| Concessão de exceção | Operadores do tenant | Prazo máximo, aprovador distinto, trilha |
| `ExplainDecision` | SRE, auditoria | Capability + escopo de tenant (evita enumeração de política alheia) |
| Bundle-raiz (`scope = platform`) | Arquitetura-chefe + CISO | Imutável pelo fluxo comum; cerimônia |
| Fontes de atributo | Módulos provedores | Registro explícito, criticidade, `sensitive` |

## 2. AuthN / AuthZ

### 2.1 Autenticação

- **Serviços**: **mTLS** com certificado de workload emitido pelo `../021-Security/`
  após atestação (RFC-0001 §6). O 022 **não** autentica ninguém por conta própria.
- **Humanos**: OAuth2/OIDC validado no Gateway (`../004-API/`), com MFA exigido pelo
  `../021-Security/`; claims propagadas como atributos do sujeito.

### 2.2 Autorização das operações de decisão

| Regra | Efeito |
|-------|--------|
| Capability `pol:decision:evaluate` exigida | Sem ela, `403 AIOS-POL-0002` |
| Um PEP só pergunta pelo **seu** tenant | `tenant` divergente ⇒ `403 AIOS-POL-0003` (NFR-016) |
| RLS por `tenant_id` em todas as tabelas multi-tenant | Segunda camada, independente do código (`./Database.md`) |

### 2.3 O problema do autogoverno

O PDP autoriza a si mesmo — o que só é seguro se a política que o governa **não puder
ser alterada pelo mesmo caminho que ela governa**. O desenho:

```
   ┌────────────────────────────────────────────────────────────────┐
   │  BUNDLE-RAIZ  (scope = platform, pol.selfgov.root_bundle_immutable)
   │  · define quem autora, testa, simula, publica, reverte e       │
   │    concede exceção — em QUALQUER tenant                        │
   │  · suas regras `deny` têm PRECEDÊNCIA ABSOLUTA (§6 StateMachine)│
   │  · alteração exige CERIMÔNIA: dois aprovadores distintos,      │
   │    fora do caminho de API comum, registrada em 025-Audit       │
   └───────────────────────────┬────────────────────────────────────┘
                               │ governa
                               ▼
   ┌────────────────────────────────────────────────────────────────┐
   │  BUNDLES DE TENANT (scope = tenant)                            │
   │  · nenhum tenant pode conceder-se o que a plataforma nega      │
   └────────────────────────────────────────────────────────────────┘
```

**Separação de funções (SoD):**

| Operação | Regra | Erro |
|----------|-------|------|
| Publicar bundle | `approved_by ≠ authored_by` (constraint `ck_bundle_approver`) | `AIOS-POL-0018` |
| Conceder waiver | `approved_by ≠ requested_by` (constraint `ck_exc_approver`) | `AIOS-POL-0018` |
| Vincular papel `is_privileged` | Aprovação dupla | `AIOS-POL-0018` |
| Alterar bundle-raiz | Cerimônia com dois aprovadores de plataforma | `AIOS-POL-0002` |

> A separação está no **esquema do banco** (`./Database.md` §2 e §5), não apenas na
> aplicação. Um controle de governança que vive só no código é desligável por um
> *deploy*; um `CHECK` constraint exige uma migração revisada.

### 2.4 Fail-closed

| Condição | Comportamento |
|----------|---------------|
| Fonte de atributo `required` indisponível | `deny` com `attribute_unavailable` |
| Orçamento de avaliação estourado | `deny` com `eval_timeout` |
| Sem bundle ativo | `deny` com `no_active_bundle` (invariante I5) |
| PDP indisponível (visto pelo PEP) | O PEP aplica seu `fail_mode` — default `closed` (RFC-0001 §5.8) |

`pol.attribute.fail_mode = open` existe apenas para desenvolvimento local e **NÃO
DEVERIA** ser habilitado em produção sem exceção documentada em ADR.

## 3. Integridade do artefato de política

| Controle | Como |
|----------|------|
| Assinatura obrigatória | `pol.bundle.signature_required = true` (não-recarregável); ativação sem assinatura válida → `AIOS-POL-0014` |
| Verificação na carga | Toda réplica verifica a assinatura ao carregar; falha ⇒ não fica *ready* (`./Deployment.md` §4) |
| Imutabilidade | Bundle imutável a partir de `Validated` (invariante I2); edição → `AIOS-POL-0020` |
| Reprodutibilidade | `compiled_digest` + retenção do artefato (invariante I6) permitem reexecutar a decisão exata |
| Cadeia de custódia | `authored_by`, `approved_by`, `signed_by`, `activated_at` — todos obrigatórios em `Active` |

## 4. Proteção de dados sensíveis

| Item | Regra |
|------|-------|
| Valores de atributo `sensitive` | **Nunca** em log, métrica, evento, journal ou resposta — apenas `sha256` em `attributes_digest` (FR-023) |
| Conteúdo do recurso avaliado | Nunca transita pelo PDP: o 022 recebe o **URN**, não o dado |
| `policy.decision.denied` | Contém motivo e regra, **não** o atributo que causou a negação (`./Events.md` §1) |
| Métricas | Sem `subject_urn` como label (cardinalidade + privacidade) — `./Metrics.md` §5 |
| `ExplainDecision` | Devolve **quais** atributos foram usados, não seus valores sensíveis |

## 5. Threat model STRIDE

| Ameaça | Vetor no 022-Policy | Mitigação | Verificação |
|--------|---------------------|-----------|-------------|
| **S**poofing | Serviço se passa por PEP de outro módulo/tenant e pergunta em nome de terceiros; ator responde a `policy.decision.request` fingindo ser o PDP. | mTLS com certificado atestado (`../021-Security/`); validação de `tenant` (`AIOS-POL-0003`); no barramento, o subject de resposta é restrito por conta NATS (`../020-Communication/`) de modo que só o PDP responde. | Teste de contrato negativo; `AIOS-POL-0003` monitorado. |
| **T**ampering | Alteração de regra em bundle ativo; injeção de vínculo privilegiado; adulteração do artefato em trânsito ou em disco. | Imutabilidade a partir de `Validated` (`AIOS-POL-0020`); artefato assinado e verificado na carga (NFR-015, `AIOS-POL-0014`); OCC + trilha em `../025-Audit/`. | Teste de manipulação de artefato; réplica recusa carregar. |
| **R**epudiation | Negar ter autorado, aprovado, publicado, revertido ou concedido exceção. | `authored_by`/`approved_by`/`granted_by`/`requested_by` obrigatórios; journal com `decision_id` e `bundle_version`; trilha imutável em `../025-Audit/`. | Auditoria cruzada journal × trilha. |
| **I**nformation disclosure | Vazamento de atributo sensível; uso do `Explain` para enumerar política alheia; inferência da política por sondagem sistemática de decisões. | `sensitive` só como `sha256` (FR-023); `Explain` sob capability e restrito ao tenant; `max_eval_per_s_per_tenant` limita sondagem; métricas sem `subject_urn`. | Varredura contínua de conteúdo; T-POL-023. |
| **D**enial of service | Regra patológica que explode o tempo de avaliação; tempestade de decisões de um tenant; publicação que transforma todo `allow` em `deny`. | Limites de complexidade na **compilação** (`AIOS-POL-0016`), não em runtime; orçamento por decisão (`AIOS-POL-0011`); limite **por tenant** (`AIOS-POL-0010`); gate de simulação expõe o salto de `deny` antes da publicação. | Fuzzing de regra; teste de carga por tenant; T-POL-008. |
| **E**levation of privilege | Autoconcessão de papel privilegiado; waiver eterno; edição do bundle-raiz pelo caminho comum; PEP que ignora obrigação para obter acesso mais amplo. | Aprovação dupla (`AIOS-POL-0018`); `expires_at` obrigatório com teto de 72 h; bundle-raiz imutável com precedência absoluta; obrigação desconhecida ⇒ `deny` no PEP (ADR-0225). | Teste de contrato de obrigação; revisão trimestral de waivers renovados. |

### 5.1 Ameaça específica: o waiver que vira política

O padrão de abuso mais provável **não** é o ataque externo: é a exceção renovada
indefinidamente até se tornar a regra de fato. Controles:

- Prazo máximo de 72 h (`pol.exception.max_ttl_h`), sem renovação automática.
- `policy.exception.granted` e `.expired` com retenção de **365 dias**
  (`POLICY_GOVERNANCE`), permitindo detectar renovações repetidas.
- Alerta `PolicyExceptionChurnHigh` (`./Monitoring.md` §4) quando o mesmo par
  (sujeito, ação) recebe mais de 3 waivers em 30 dias — o sinal de que aquilo deveria
  ser uma **regra revisada**, não uma exceção perpétua.

## 6. LGPD / GDPR

| Item | Tratamento |
|------|-----------|
| **Minimização** | O módulo avalia atributos, não conteúdo. O journal guarda URNs, ação, motivo e `attributes_digest` — nunca valores sensíveis. |
| **Base legal e retenção** | Journal é dado operacional de segurança, retenção curta (30 d); a retenção longa exigida por lei é do `../025-Audit/`, que declara a própria base. |
| **Direito ao esquecimento** | Ao consumir `security.principal.disabled`, o módulo expira vínculos e exceções do titular e **pseudonimiza** `subject_urn` no journal, preservando contagens agregadas: o registro de que *uma* decisão ocorreu não pode desaparecer, mas *quem* pode ser dissociado. |
| **Decisão automatizada** | Quando a política nega acesso a um titular com base em atributos pessoais, a explicabilidade obrigatória (FR-003/FR-016) é o que permite atender ao direito de revisão — um `deny` sem motivo seria, além de defeito operacional, problema de conformidade. |
| **Segregação** | RLS por tenant em todas as tabelas; políticas de um tenant são invisíveis a outro, inclusive por `Explain` e por simulação. |
| **Transferência internacional** | O módulo não transfere dado pessoal; atributos federados vêm do `../021-Security/`, que registra a adequação por tenant. |

## 7. Controles e verificação contínua

| Controle | Frequência | Evidência |
|----------|-----------|-----------|
| Nenhum `allow` sem regra casada | Contínua (constraint + varredura) | NFR-008, `ck_allow_has_rule` |
| Nenhum valor sensível em telemetria | Contínua | T-POL-023 |
| 100% dos bundles ativos assinados | Na carga e na ativação | NFR-015 |
| Revisão de waivers vigentes | Semanal | Relatório de `policy.policy_exception` |
| Revisão de papéis `is_privileged` e seus vínculos | Trimestral | `GetEffectivePermissions` por papel |
| Teste de bypass do `SelfGovernanceGuard` | A cada release | T-POL-018 |
| Pentest da superfície de decisão | Semestral | Relatório de `../029-Operations/` |

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §12
- Identidade e credenciais: `../021-Security/`
- Trilha imutável: `../025-Audit/` · Decisão-mãe: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`
- Contratos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6 e §7
- Modelo físico: `./Database.md` · Alertas: `./Monitoring.md`

*Fim de `Security.md`.*
