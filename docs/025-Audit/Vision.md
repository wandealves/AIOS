---
Documento: Vision
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0001, ADR-0008, ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: 000-Vision, 001-Architecture, 021-Security, 022-Policy, 024-Observability
---

# 025-Audit — Vision

## 1. Propósito

O 025-Audit é a **memória probatória** do AIOS: o subsistema que registra o que
aconteceu de forma que possa ser **provado** — inclusive a alguém que não confia no
operador do sistema.

Ele materializa a metade "auditoria" da
`../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md` e o requisito da
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.8: *toda ação privilegiada e toda
decisão de agente DEVEM emitir registro de auditoria imutável*.

Na analogia de sistema operacional
(`../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md`), este módulo é o
*journal* com garantia de integridade — elevado a requisito de conformidade: não basta
que o registro exista; é preciso demonstrar que **não foi alterado desde que foi
escrito**.

## 2. Problema que resolve

| Patologia sem o módulo | Sintoma observável | Custo |
|------------------------|--------------------|-------|
| **Imutabilidade apenas declarada** | Uma tabela com "não altere" no comentário. | Quem tem `UPDATE` reescreve a história; a trilha não prova nada. |
| **Lacuna invisível** | Um produtor para de emitir e ninguém percebe. | O registro que nunca chegou não deixa rastro — o defeito perfeito. |
| **Imutabilidade × direito ao esquecimento** | Ou se apaga (e perde-se a prova) ou não se apaga (e viola-se a LGPD). | Escolha entre não conformidade legal e não conformidade regulatória. |
| **Auditoria dependente do auditado** | O auditor precisa confiar no operador para verificar a trilha. | A prova vale exatamente o quanto se confia em quem a guarda. |
| **Trilha como métrica** | Usar a trilha para painéis e alertas. | Retenção e custo explodem; ou a auditoria degrada para caber. |

## 3. Escopo

### 3.1 Dentro do escopo (in)

- **Ingestão** de registros por evento (JetStream) e por API direta, idempotente, com
  confirmação **somente após durabilidade**.
- **Envelope canônico de auditoria**: quem, o quê, sobre o quê, quando, sob qual
  decisão e correlação.
- **Encadeamento por hash** e **selo periódico** com raiz de Merkle assinada.
- **Verificação de integridade** contínua (amostragem) e completa (sob demanda).
- **Verificação de completude**: lacuna de sequência e classe silenciosa.
- **Retenção legal** por classe, com base legal declarada.
- **Legal hold**: suspensão de retenção e apagamento sobre um escopo.
- **Apagamento criptográfico** (destruição da chave do titular) preservando a cadeia.
- **Comprovantes** verificáveis de apagamento, próprios e recebidos de outros módulos.
- **Arquivamento WORM** das partições seladas.
- **Consulta** com PEP e **exportação de pacote probatório** verificável offline.
- **Detecção de anomalias** de auditoria.

### 3.2 Fora do escopo (out)

| Não faz | Dono real |
|---------|-----------|
| Ser fonte de métrica operacional ou de painel | `../024-Observability/` |
| Decidir autorização | `../022-Policy/` |
| Alertar o plantão ou rotear notificação | `../024-Observability/`, `../029-Operations/` |
| Estabelecer identidade ou custodiar chaves | `../021-Security/` |
| Executar o expurgo do dado de negócio nos módulos donos | `../005-Database/`, `../010-Memory/`, módulo dono |
| Definir a classificação (`data_class`) do conteúdo auditado | módulo dono do dado |
| Interpretar juridicamente a base legal ou decidir se um *hold* é devido | jurídico/DPO do tenant |
| Reprocessar ou corrigir um registro | — (correção é registro de compensação) |
| Investigar incidente ou conduzir a resposta | `../029-Operations/` |
| Servir como barramento ou fila de integração | `../020-Communication/` |

A lista completa e normativa está em `./_DESIGN_BRIEF.md` §1.3.

## 4. Personas atendidas

| Persona | Necessidade | Como o módulo atende |
|---------|-------------|----------------------|
| **Auditor externo** | Verificar a trilha **sem confiar** no operador do AIOS. | Pacote probatório com selos assinados, provas de Merkle e instruções de verificação offline (FR-017). |
| **DPO / jurídico** | Conciliar retenção legal, *legal hold* e direito ao esquecimento. | Retenção por classe com base legal; *hold* com `case_ref`; apagamento criptográfico com comprovante. |
| **Investigador de incidente** (`../029-Operations/`) | Reconstruir a cronologia de uma ação privilegiada. | Consulta por `subject_ref`, `decision_ref` e `trace_id`; correlação com o journal do `../022-Policy/`. |
| **Módulos produtores** | Registrar o fato sem risco de perdê-lo. | At-least-once + dedupe; recusa explícita (`AIOS-AUD-0005`) em vez de aceite sem durabilidade. |
| **Segurança / CISO** | Detectar tentativa de adulteração da evidência. | Cadeia + selo assinado + WORM + verificação contínua; `chain_break` é incidente máximo. |
| **`../005-Database/`** | Saber quando suspender expurgo. | Evento `audit.legalhold.applied`. |
| **Titular de dados** | Exercer o direito ao esquecimento. | Apagamento criptográfico com comprovante verificável. |

## 5. Princípios de design

1. **Imutabilidade é verificável, não declarada.** Cadeia de hash + selo de Merkle
   assinado + cópia WORM: adulterar exige comprometer três sistemas distintos
   (ADR-0251).
2. **A lacuna é o defeito mais perigoso.** Um registro adulterado quebra o hash e é
   detectado; um registro que nunca chegou não deixa rastro. Completude é **medida**,
   não presumida (ADR-0256).
3. **Apagar conteúdo, nunca a prova de existência.** O apagamento criptográfico
   destrói a chave; `seq`, hashes e `payload_digest` permanecem (ADR-0253).
4. **Legal hold sobrepõe tudo.** Nem retenção nem direito ao esquecimento removem
   registro sob *hold* (ADR-0254).
5. **Confirmar só após durabilidade.** `503` é a resposta correta quando não se pode
   garantir a escrita — o produtor tem outbox e reenvia (ADR-0255).
6. **Append-only sem exceção.** Correção é **novo** registro de compensação; não
   existe capability de alterar (ADR-0258).
7. **A consulta à trilha é fato auditável.** Quem consultou o quê é exatamente o que
   um auditor precisa saber (ADR-0257).
8. **A cadeia é particionada.** Uma sequência global única seria um ponto de
   serialização absoluto (ADR-0252).

## 6. Não-objetivos explícitos

- **Não** é observabilidade: nada aqui é amostrável, e a retenção é legal, não
  operacional (ADR-0246, definido em `../024-Observability/`).
- **Não** oferece "modo rápido" que aceita sem durabilidade: `aud.ingest.require_durable_ack`
  é não-recarregável e `true`.
- **Não** expõe operação de alteração ou exclusão de registro — a capability não
  existe no modelo.
- **Não** guarda conteúdo de negócio além do necessário para provar a ação.
- **Não** substitui o journal de decisões do `../022-Policy/` (retenção curta,
  explicabilidade) nem os comprovantes de expurgo dos módulos donos: **registra** os
  dois.
- **Não** decide se um *hold* é devido nem interpreta base legal: aplica o que foi
  decidido, com autor e aprovador registrados.

## 7. Relação com a visão global do AIOS

`../000-Vision/Vision.md` coloca **governabilidade** — "política, auditoria e
compliance como estrutura" — entre os princípios do sistema. O 025 é onde essa
promessa se torna verificável por terceiros: sem ele, "auditoria" seria um registro
que o próprio auditado guarda e pode reescrever.

```
   000-Vision ──── "política, auditoria e compliance como estrutura"
        │
        ├── ADR-0008 ──▶ toda ação privilegiada passa pelo PDP
        ├── ADR-0010 ──▶ auditoria POR CONSTRUÇÃO
        │                       │
        │              RFC-0001 §5.8 ──▶ registro imutável obrigatório
        │                       │
        └──────────────▶  025-AUDIT (este módulo)
                                │
        021 (quem é) ── 022 (se pode) ──┼── 024 (como está indo)
                                │
                        auditor externo (verifica sem confiar)
```

## 8. Critérios de sucesso do módulo

| Critério | Meta | Verificação |
|----------|------|-------------|
| Nenhum registro confirmado é perdido | RPO = 0 | NFR-006 |
| Adulteração sempre detectada | ≤ 1 ciclo de verificação | NFR-007 |
| Completude verificável | lacuna detectada em ≤ 5 min | NFR-005 |
| Cadeia sobrevive ao apagamento | 100% verificável após RTBF | FR-012 |
| Nenhum expurgo sob *legal hold* | 0 ocorrências | NFR-010 |
| Auditor verifica sem acesso ao AIOS | pacote validável offline | FR-017 |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md` (*Event Sourcing*)
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- Arquitetura do módulo: `./Architecture.md` · Requisitos: `./Requirements.md`

*Fim de `Vision.md`.*
