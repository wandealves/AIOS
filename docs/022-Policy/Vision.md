---
Documento: Vision
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0001, ADR-0008, ADR-0010, ADR-0221, ADR-0224, ADR-0228
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: 000-Vision, 001-Architecture, 021-Security, 025-Audit, 026-Cost-Optimizer
---

# 022-Policy — Vision

## 1. Propósito

O 022-Policy é o **monitor de referência** do AIOS: o único subsistema autorizado a
responder à pergunta que antecede toda ação privilegiada do sistema — *este sujeito
pode executar esta ação sobre este recurso, neste contexto?*

Ele é o **PDP** (*Policy Decision Point*, ver `../040-Glossary/Glossary.md`) de que
fala a `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md` e a
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.8. Todo módulo que executa algo
privilegiado possui um **PEP**; nenhum deles decide. A decisão vive aqui, uma única
vez, sobre um artefato de política versionado, testado e assinado.

Na analogia de sistema operacional que orienta o AIOS
(`../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md`), este módulo é o
subsistema de **controle de acesso obrigatório** — com uma diferença essencial: as
regras não estão compiladas no núcleo, são **dado versionado** que evolui sem
redeploy.

## 2. Problema que resolve

Um AIOS de agentes autônomos executa, sem supervisão humana por chamada, ações que
gastam dinheiro, tocam dados pessoais, invocam ferramentas externas e criam outros
agentes. Sem um ponto de decisão único, três patologias aparecem:

| Patologia | Sintoma observável | Custo |
|-----------|--------------------|-------|
| **Autorização espalhada** | `if (user.IsAdmin)` replicado em dezenas de serviços. | Ninguém consegue enunciar a postura de segurança do sistema, muito menos prová-la a um auditor. |
| **Política como configuração** | Regras editadas em YAML de deploy, sem teste nem histórico. | Uma mudança silenciosa amplia acesso e só se descobre no incidente. |
| **Negação inexplicável** | `403` sem motivo. | A reação operacional a um `deny` inexplicado é sempre a mesma: afrouxar a política. |

O 022 existe para tornar as três impossíveis: **uma** decisão, **um** artefato de
release e **sempre** um motivo.

## 3. Escopo

### 3.1 Dentro do escopo (in)

- Avaliação **RBAC + ABAC** de `DecisionRequest`, sob ***default deny***.
- Resposta com `effect`, `reason_code`, regras casadas, **obrigações** e **TTL** de cache.
- Ciclo de vida do **policy bundle**: autoria → compilação → análise de conflitos →
  teste golden → simulação → assinatura → publicação → supersessão → **rollback**.
- Distribuição do bundle ativo e invalidação de cache nos PEPs
  (`policy.bundle.updated`, `policy.decision.updated`).
- Resolução de **atributos** externos (identidade do `../021-Security/`, orçamento do
  `../026-Cost-Optimizer/`, sinais do `../024-Observability/`).
- **Exceções (waivers)** temporárias com prazo, justificativa e aprovador distinto.
- **Explicabilidade**: reprodução exata de qualquer decisão registrada.
- Autogoverno: autorização das próprias operações administrativas por um bundle-raiz
  de escopo `platform`.

### 3.2 Fora do escopo (out)

| Não faz | Dono real |
|---------|-----------|
| **Aplicar** a decisão (bloquear, cortar, derrubar) | módulo chamador (PEP) |
| Autenticar sujeito, emitir ou validar credencial | `../021-Security/` |
| Persistir a trilha de auditoria imutável | `../025-Audit/` |
| Definir e contabilizar orçamento/custo | `../026-Cost-Optimizer/` |
| Aplicar cotas e rate-limit de recurso | `../006-Kernel/`, `../026-Cost-Optimizer/` |
| Executar isolamento (sandbox, seccomp, cgroups) | `../007-Agent-Runtime/` (aplica), `../021-Security/` (publica) |
| Decidir placement, prioridade ou preempção | `../009-Scheduler/` |
| Classificar dado ou aplicar RLS | `../005-Database/` + módulo dono |
| Detectar ameaça ou anomalia comportamental | `../021-Security/`, `../025-Audit/` |
| Responder a incidente (bloquear sujeito, acionar plantão) | `../029-Operations/` |

A lista completa e normativa está em `./_DESIGN_BRIEF.md` §1.3.

## 4. Personas atendidas

| Persona | Necessidade | Como o módulo atende |
|---------|-------------|----------------------|
| **Serviço com PEP** (`004`, `006`, `007`, `009`, `010`, `011`, `015`, `017`, `020`, `021`) | Decidir rápido, com cache e sem virar refém do PDP. | `Evaluate` p99 ≤ 5 ms (NFR-001), TTL por decisão, invalidação granular por evento. |
| **Engenheiro de segurança do tenant** | Escrever, revisar e provar a política antes de ela vigorar. | Autoria versionada, testes golden obrigatórios, simulação com diferencial, aprovação dupla. |
| **Operador de plantão (SRE)** | Entender por que algo foi negado às 3h da manhã. | `reason_code` estável, `ExplainDecision`, journal de decisões, evento `policy.decision.denied`. |
| **Auditor / DPO** | Provar que a autorização é consistente, explicável e reprodutível. | Decisão reproduzível por `bundle_version`; trilha imutável em `../025-Audit/`; explicabilidade obrigatória (LGPD). |
| **Desenvolvedor de agente** | Saber o que seu agente pode fazer, sem tentativa e erro. | `GetEffectivePermissions`; obrigações declaradas na resposta. |

## 5. Princípios de design

1. **Default deny é comportamento, não configuração.** Falta de regra, falta de
   atributo, falta de tempo e falta de bundle produzem `deny`
   (`./_DESIGN_BRIEF.md` §4.5, invariante I5).
2. **Política é artefato de release.** Compilada, versionada, assinada, testada,
   simulada e **reversível** — nunca editada em vigor (ADR-0221, invariante I2).
3. **Toda decisão é explicável e reproduzível.** `reason_code` + regras casadas +
   `bundle_version` bastam para reconstruir a decisão (NFR-009).
4. **O PDP não pode ser o gargalo.** Bundle compilado em memória, zero I/O no caminho
   quente, cache no PEP com TTL emitido pelo próprio PDP (ADR-0222).
5. **Uma só forma de "não sei".** `indeterminate` e `not_applicable` colapsam em
   `deny` antes de sair do módulo (ADR-0224) — expor uma terceira via convidaria cada
   PEP a inventar o próprio tratamento.
6. **O que autoriza também é autorizado.** O módulo se governa por um bundle-raiz
   imutável, com separação de funções e aprovação dupla (ADR-0228).
7. **Exceção tem prazo.** Waiver sem `expires_at` é política nova aprovada por
   ninguém (ADR-0227).

## 6. Não-objetivos explícitos

- **Não** é um *feature store*: o 022 resolve atributos, não é dono de nenhum.
- **Não** é um motor de regras de negócio: decide autorização, não fluxo de aplicação.
- **Não** oferece "modo permissivo" em produção: `pol.attribute.fail_mode = open`
  existe apenas para ambientes de teste e **NÃO DEVERIA** ser habilitado em produção
  sem exceção documentada em ADR.
- **Não** mantém o cache dos PEPs: emite o TTL, não gerencia a memória alheia.
- **Não** substitui a trilha de auditoria: seu journal tem retenção curta (30 dias);
  a prova de longo prazo é do `../025-Audit/` (ADR-0229).
- **Não** entrega `deny` como erro HTTP: negação é resposta `200` com
  `effect = deny`; os códigos `AIOS-POL-*` cobrem falhas de administração e avaliação.

## 7. Relação com a visão global do AIOS

`../000-Vision/Vision.md` define o AIOS como um sistema operacional cognitivo com
governança por construção. O 022 é o ponto onde essa promessa é verificável: sem ele,
"governança" seria uma propriedade declarada; com ele, é uma decisão registrada, com
motivo, versão e assinatura, para cada ação privilegiada que o sistema executou.

```
   000-Vision ──── "governança por construção"
        │
        ├── ADR-0008 ──▶ PEP/PDP com default deny
        │                       │
        │              RFC-0001 §5.8 ──▶ toda ação privilegiada passa pelo PDP
        │                       │
        └──────────────▶  022-POLICY (este módulo)
                                │
                 021 (quem é) ──┼── 025 (o que aconteceu)
                                │
                        PEPs de todos os módulos (o que é impedido)
```

## 8. Critérios de sucesso do módulo

| Critério | Meta | Verificação |
|----------|------|-------------|
| Nenhum `allow` sem regra explícita | 0 ocorrências | NFR-008, varredura do journal |
| Toda negação explicável | 100% com `reason_code` | NFR-007 |
| PDP fora do caminho crítico de latência | p99 ≤ 5 ms | NFR-001 |
| Mudança de política reversível | rollback ≤ 1 operação | FR-010, T-11/T-12 |
| Nenhuma publicação sem prova | 100% com teste + simulação | FR-007, FR-008 |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisão-mãe: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`
- Arquitetura do módulo: `./Architecture.md` · Requisitos: `./Requirements.md`

*Fim de `Vision.md`.*
