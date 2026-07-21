---
name: aios-adr
description: Procedimento canônico para autorar uma ADR (Architecture Decision Record) do AIOS em pt-BR — convenção de nome de arquivo, regra da faixa de numeração por módulo (ADR-<NNN*10>..<NNN*10+9>), estrutura de seções derivada do TEMPLATE.md, critérios de qualidade (alternativas reais, trade-offs explícitos), registro no índice 002-ADR/README.md e esqueleto preenchível. Use ao criar ou revisar uma ADR.
---

# Skill: Autorar uma ADR do AIOS

Guia operacional para produzir uma Architecture Decision Record em conformidade
com o repositório AIOS. Tudo em **português (pt-BR)**; documentação técnica, não código.

## 1. Passos do procedimento

1. **Ler o template**: `docs/002-ADR/TEMPLATE.md` (fonte da verdade do front-matter
   e da ordem das seções).
2. **Ler exemplos** para calibrar tom/densidade:
   `docs/002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md` e
   `docs/002-ADR/ADR-0009-Model-Router-Multiobjetivo.md`.
3. **Ler o índice** `docs/002-ADR/README.md` — descobrir números já emitidos.
4. **Ler a baseline** `docs/003-RFC/RFC-0001-Architecture-Baseline.md` — a ADR
   **não pode** contradizer seus contratos (URN, envelope CloudEvents, envelope de
   erro `AIOS-<DOMINIO>-<NNNN>`, idempotência, correlação, subjects NATS). Mudar um
   contrato da RFC-0001 é escopo de nova RFC, não de ADR.
5. **Consultar o glossário** `docs/040-Glossary/Glossary.md` — reutilizar termos.
6. **Escolher o número** conforme a faixa do módulo (§2).
7. **Escrever o arquivo** com o esqueleto (§5), preenchendo todas as seções.
8. **Registrar no índice** (§6).
9. **Verificar** o checklist de qualidade (§4).

## 2. Numeração por módulo (regra da faixa)

- ADRs globais **0001–0010** já existem (reservadas).
- Cada módulo `NNN` tem a faixa **`ADR-<NNN*10>` .. `ADR-<NNN*10+9>`** (10 slots).

| Módulo | Faixa de ADR |
|--------|--------------|
| 004-API | ADR-0040 .. ADR-0049 |
| 005-Database | ADR-0050 .. ADR-0059 |
| 006-Kernel | ADR-0060 .. ADR-0069 |
| 009-Scheduler | ADR-0090 .. ADR-0099 |
| 010-Memory | ADR-0100 .. ADR-0109 |
| 017-Model-Router | ADR-0170 .. ADR-0179 |
| 022-Policy | ADR-0220 .. ADR-0229 |

Regras: pegue o **menor número livre** da faixa; **nunca reutilize** um número
(mesmo `Deprecated`/`Superseded`); sempre **4 dígitos** zero-padded.

## 3. Nome de arquivo

```
docs/002-ADR/ADR-NNNN-Titulo-Em-Palavras.md
```

- Título em pt-BR, palavras com inicial maiúscula separadas por hífen.
- Sem acentos/cedilha no nome do arquivo (o título dentro do doc leva acentos).
- Exemplos: `ADR-0061-Escalonamento-Preemptivo-do-Kernel.md`,
  `ADR-0091-Fila-de-Prioridade-Cognitiva.md`.

## 4. Estrutura de seções (derivada do TEMPLATE.md)

Ordem obrigatória, todas presentes:

1. **Contexto** — forças, restrições, estado atual, requisitos. Fatos, não opiniões.
2. **Problema** — a pergunta arquitetural única e específica.
3. **Alternativas** — lista numerada, ≥ 3 opções plausíveis.
4. **Análise** — tabela comparando alternativas × critérios, com evidências.
5. **Escolha** — a decisão, inequívoca ("Adotamos X.").
6. **Consequências** — **Positivas / Negativas / Neutras-estruturais**.
7. **Riscos** — tabela `Risco | Prob. | Impacto | Mitigação`.
8. **Trade-offs** — ganhos vs. renúncias, explícitos.
9. **Referências** — caminhos relativos para Vision/Architecture/RFC/ADRs/módulos.

### Critérios de qualidade (checklist)

- [ ] Front-matter completo (Documento, Módulo `002-ADR`, Status, Versão, Última
      atualização, Responsável RACI-A, Decisores, Módulos afetados).
- [ ] **≥ 3 alternativas reais** — inclusive a descartada por bom motivo; sem espantalhos.
- [ ] Análise em **tabela**, com critérios relevantes ao módulo.
- [ ] Escolha inequívoca e derivada da análise.
- [ ] Consequências honestas, incluindo as **negativas**.
- [ ] Trade-offs explícitos (o que se abre mão).
- [ ] **Não contradiz a RFC-0001**; contratos referenciados, nunca redefinidos.
- [ ] Termos do glossário reutilizados; nada redefinido.
- [ ] Todos os caminhos relativos resolvem.
- [ ] Uma decisão por ADR.
- [ ] Índice `002-ADR/README.md` atualizado.

## 5. Esqueleto preenchível

```markdown
---
Documento: ADR-NNNN — <Título curto e imperativo>
Módulo: 002-ADR
Status: Proposed
Versão: 1.0
Última atualização: AAAA-MM-DD
Responsável (RACI-A): <papel, ex.: Arquitetura-Chefe>
Decisores: <lista, ex.: Steering Committee>
Módulos afetados: <NNN, ...>
---

# ADR-NNNN: <Título>

## Contexto
<Forças em jogo, restrições, estado atual, requisitos. Fatos, não opiniões.>

## Problema
<A pergunta arquitetural específica a ser respondida.>

## Alternativas
1. **<Opção A>** — descrição.
2. **<Opção B>** — descrição.
3. **<Opção C>** — descrição.

## Análise
<Comparação estruturada contra critérios; evidências.>

| Critério | Opção A | Opção B | Opção C |
|----------|---------|---------|---------|
| ... | ... | ... | ... |

## Escolha
Adotamos **<Opção X>**. <Justificativa derivada da análise.>

## Consequências
**Positivas:** ...
**Negativas:** ...
**Neutras/estruturais:** ...

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| ... | ... | ... | ... |

## Trade-offs
<O que ganhamos vs. o que abrimos mão, explicitamente.>

## Referências
`../000-Vision/Vision.md`; `../001-Architecture/Architecture.md`;
`../003-RFC/RFC-0001-Architecture-Baseline.md`; `../<NNN-Modulo>/`.
```

## 6. Registrar no índice

Em `docs/002-ADR/README.md`, na tabela `## Índice`, insira **em ordem numérica**
uma nova linha:

```markdown
| [NNNN](ADR-NNNN-Titulo-Em-Palavras.md) | <Título> | Proposed | <Decisão em 1 linha.> |
```

Mantenha o alinhamento das colunas coerente com as linhas existentes.
