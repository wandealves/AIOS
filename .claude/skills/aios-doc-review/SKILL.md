---
name: aios-doc-review
description: Metodologia operacional para auditar a documentação de um módulo AIOS (`docs/NNN-Nome/`) contra o `_templates/MODULE_TEMPLATE.md`, o `_DESIGN_BRIEF.md` do módulo, a `003-RFC/RFC-0001-Architecture-Baseline.md` e o `040-Glossary/Glossary.md`. Fornece os checklists de completude (26 docs), cabeçalho de metadados, Definition of Done, consistência, referências cruzadas, convenções de nomenclatura e rastreabilidade de requisitos; define os níveis de severidade e o formato exato do relatório de saída. Use ao revisar/auditar/validar a documentação de um módulo AIOS. READ-ONLY — produz relatório de defeitos, não corrige.
---

# Skill: aios-doc-review — Auditoria de Documentação de Módulo AIOS

Metodologia **READ-ONLY** para auditar a documentação de **um** módulo do AIOS
(`docs/NNN-Nome/`). O objetivo é encontrar **defeitos acionáveis** de conformidade e
consistência, e reportá-los num relatório padronizado por severidade. Esta skill **não
corrige** nada — apenas descreve o procedimento e o formato de saída.

## 0. Precedência das fontes

Ordem de autoridade (quem manda quando há conflito):

```
MODULE_TEMPLATE.md  (estrutura/DoD)
        ▲
RFC-0001 (contratos centrais)  ≡  Glossary (termos)   ← não redefinir localmente
        ▲
_DESIGN_BRIEF.md do módulo      ← fonte única de verdade interna do módulo
        ▲
Os 26 documentos do módulo      ← DEVEM derivar do brief; nunca o contradizem
```

Um documento do módulo que contradiz o brief, a RFC-0001, o glossário ou o template é
sempre um defeito **do documento**. Leia estas fontes **antes** de julgar:

- `docs/_templates/MODULE_TEMPLATE.md`
- `docs/NNN-Nome/_DESIGN_BRIEF.md`
- `docs/003-RFC/RFC-0001-Architecture-Baseline.md`
- `docs/040-Glossary/Glossary.md`
- `docs/README.md` e `docs/NNN-Nome/README.md`

## 1. Lista canônica dos 26 documentos obrigatórios

Confira presença (`Glob docs/NNN-Nome/*.md`). Cada ausência é **Bloqueante**.

```
 1. Vision.md                 10. Database.md              19. Testing.md
 2. Architecture.md           11. API.md                   20. Benchmark.md
 3. Requirements.md           12. Events.md                21. FailureRecovery.md
 4. FunctionalRequirements.md 13. Configuration.md         22. Scalability.md
 5. NonFunctionalRequirements 14. Deployment.md            23. Examples.md
    .md                       15. Security.md              24. FAQ.md
 6. UseCases.md               16. Monitoring.md            25. ADR.md
 7. SequenceDiagrams.md       17. Logging.md               26. RFC.md
 8. ClassDiagrams.md          18. Metrics.md
 9. StateMachine.md
```

Arquivos legítimos além dos 26: `README.md` e `_DESIGN_BRIEF.md`. Outros `.md` não
previstos = achado **Baixo** (verificar se deveriam existir).

## 2. Cabeçalho de metadados (todo documento)

Todo `.md` (exceto, opcionalmente, `README.md`) DEVE abrir com o bloco YAML-like
delimitado por `---`, com **todos** estes campos preenchidos e não-vazios:

```
---
Documento: <Nome do Documento>
Módulo: NNN-Nome
Status: Draft | Review | Stable | Deprecated
Versão: MAJOR.MINOR
Última atualização: AAAA-MM-DD
Responsável (RACI-A): <papel/equipe>
ADRs relacionados: ADR-XXXX[, ...]  (ou "—" quando não houver)
RFCs relacionados: RFC-XXXX          (ou "—" quando não houver)
Depende de: <módulos/documentos>
---
```

Checar por documento:
- [ ] Bloco de metadados presente e bem-formado (abre e fecha com `---`).
- [ ] Todos os 9 campos presentes; nenhum vazio.
- [ ] `Módulo` == diretório (`NNN-Nome`).
- [ ] `Status` ∈ {Draft, Review, Stable, Deprecated}.
- [ ] `Versão` no formato `MAJOR.MINOR`.
- [ ] `Última atualização` no formato `AAAA-MM-DD` válido.
- [ ] IDs de ADR/RFC no formato `ADR-NNNN` / `RFC-NNNN`.

## 3. Definition of Done (DoD) por documento

Copiado e adaptado do checklist normativo de `MODULE_TEMPLATE.md`. Aplique a **cada** um
dos 26 documentos:

- [ ] **Cabeçalho de metadados preenchido** (ver §2).
- [ ] **Nenhuma seção vazia.** Um cabeçalho (`##`/`###`) sem conteúdo abaixo é defeito.
- [ ] **`TBD` só é permitido se `Status: Draft`.** `TBD`/`TODO`/`FIXME`/`XXX` em documento
      `Review`/`Stable` = defeito.
- [ ] **≥1 diagrama ASCII quando o esqueleto exige.** Obrigatório em: `Architecture.md`
      (C4 Context→Container→Component), `SequenceDiagrams.md`, `ClassDiagrams.md`,
      `StateMachine.md`, `Deployment.md`. Recomendado em `Database.md` (relações) e
      `Scalability.md`.
- [ ] **≥1 tabela e ≥1 exemplo quando aplicável.** Ex.: `FunctionalRequirements.md` e
      `NonFunctionalRequirements.md` DEVEM ter tabela; `API.md`, `Events.md`, `Examples.md`,
      `Configuration.md` DEVEM ter exemplos.
- [ ] **Termos reutilizam `../040-Glossary/Glossary.md`** — sem redefinir (ver §5).
- [ ] **Decisões apontam para uma ADR** — não decidir localmente. Afirmações do tipo
      "decidimos/escolhemos X" sem link para `../002-ADR/ADR-NNNN` = defeito.
- [ ] **Requisitos numerados e rastreáveis** (ver §7).
- [ ] **Riscos e alternativas explicitados** onde aplicável (Architecture, FailureRecovery,
      Scalability, Security, ADR).
- [ ] **Referências cruzadas por caminho relativo válidas** (ver §6).

## 4. Consistência com brief / RFC-0001 / glossário

Detectar **contradições factuais** — o achado mais grave desta auditoria. Compare os 26
documentos com o `_DESIGN_BRIEF.md` e a RFC-0001. Sinais de alerta:

- **Verbos de syscall, componentes (PascalCase), estados da FSM, transições, códigos de
  erro, subjects de evento, chaves de configuração, valores de SLO/NFR, campos de tabela**
  que **diferem** do que o brief define. Ex.: brief define 11 verbos e 8 estados de ACB;
  qualquer documento que liste um conjunto diferente é contradição.
- **Contratos centrais redefinidos** (violação da RFC-0001): reformular o formato de URN,
  do envelope de evento (CloudEvents), do envelope de erro (RFC 7807), das regras de
  idempotência (`Idempotency-Key` ≥24h) ou dos cabeçalhos de correlação. O documento DEVE
  **referenciar** a RFC-0001, não reescrevê-la com regra própria divergente.
- **Faixas reservadas** desrespeitadas: domínios/faixas de código de erro e faixas de ADR
  do módulo declaradas no brief (ex.: Kernel usa `KERNEL/CAP/SYSCALL` e `QUOTA 0001–0020`;
  ADRs `ADR-0060..0069`). Uso fora da faixa = defeito.

Classifique contradição com o brief ou com a RFC-0001 como **Bloqueante** ou **Alto**
conforme o impacto (contrato central ou de interface → Bloqueante; detalhe secundário →
Alto).

## 5. Redefinição de termos do glossário

O `040-Glossary/Glossary.md` é a fonte única terminológica. Um documento **NÃO DEVE**
redefinir um termo já glossado. Heurística de detecção:

1. Extraia os termos canônicos do glossário (linhas `**Termo** ... —`).
2. No documento auditado, procure padrões de **definição local** do mesmo termo:
   `**Termo** é/são/significa ...`, "Definimos Termo como ...", "Para fins deste documento,
   Termo é ...".
3. Se o documento **redefine** (dá sentido próprio, especialmente divergente) em vez de
   **referenciar** (`ver Glossário`, link para `../040-Glossary/Glossary.md`), reporte
   como redefinição indevida.

Uso do termo é permitido e esperado; o defeito é **criar uma segunda definição**. Termo
técnico novo que deveria estar no glossário e não está = achado **Médio** (recomendar
adição ao glossário em vez de definição local).

## 6. Validade de referências cruzadas

1. Extraia todos os links markdown relativos: `Grep` por `](\.\.?/` (padrões `](../` e
   `](./`).
2. Para cada alvo, resolva o caminho relativo a partir do arquivo que contém o link e
   confirme existência com `Glob`/`Read`. Alvo inexistente = **link quebrado** (Alto se
   for referência normativa a brief/RFC/glossário/template; Médio caso contrário).
3. Se o link tem âncora (`arquivo.md#secao`), verifique se o cabeçalho correspondente
   existe no alvo. Âncora ausente = **Baixo**.
4. Reporte também links **absolutos** para dentro do repositório (deveriam ser relativos,
   conforme convenção) = **Baixo**.

## 7. Rastreabilidade de requisitos

- **Numeração:** `FR-NNN` e `NFR-NNN` com zero-padding consistente, sem **lacunas** nem
  **duplicatas**. Reporte série descontínua (ex.: FR-001, FR-002, FR-004) e IDs repetidos.
- **Verificabilidade:** cada `FR`/`NFR` tem critério de aceite objetivo; NFRs têm meta
  numérica (SLO/SLI) e método de verificação. Requisito vago ("deve ser rápido") = defeito.
- **Matriz Requisito → UseCase → Teste:** deve existir (em `Requirements.md` e/ou nos docs
  de requisitos) e ser coerente. Todo `FR` deveria mapear a ≥1 `UC-NNN` e a ≥1 caso de
  teste (`Testing.md`). Requisito órfão (sem UC ou sem teste) = **Médio**.
- **Integridade referencial:** todo `UC-NNN` citado na matriz existe em `UseCases.md`;
  todo `FR`/`NFR` citado em `UseCases.md`/`Testing.md` existe nos docs de requisitos.
  Referência a ID inexistente = **Alto**.

## 8. Convenções de nomenclatura (verificáveis por Grep)

| Elemento | Padrão exigido | Exemplo válido | Achado se divergir |
|----------|----------------|----------------|--------------------|
| Evento (subject) | `aios.<tenant>.<dominio>.<entidade>.<acao>` | `aios.acme.agent.lifecycle.spawned` | Alto |
| Métrica | `aios_<subsistema>_<nome>_<unidade>` | `aios_kernel_spawn_latency_ms` | Alto |
| Código de erro | `AIOS-<DOMINIO>-<NNNN>` | `AIOS-KERNEL-0002` | Alto |
| URN | `urn:aios:<tenant>:<tipo>:<id>` (`<id>`=ULID) | `urn:aios:acme:agent:01J9...` | Alto |
| Componente | `PascalCase` | `CapabilityEnforcer` | Médio |
| Idioma | pt-BR (termos consagrados em inglês OK) | — | Baixo |
| Palavras normativas | RFC 2119 (DEVE/NÃO DEVE/DEVERIA/PODE) | — | Baixo |

Grep sugerido: `aios\.[a-z-]+\.` (subjects), `aios_[a-z_]+` (métricas),
`AIOS-[A-Z]+-[0-9]{4}` (erros), `urn:aios:` (URNs). Sinalize ocorrências que **não**
casam com o padrão (ex.: subject sem `<tenant>`, métrica sem unidade, erro com 3 dígitos).

## 9. Níveis de severidade

| Severidade | Critério | Exemplos típicos |
|------------|----------|------------------|
| **Bloqueante** | Impede merge; quebra a integridade do módulo. | Documento obrigatório ausente; contradição com contrato central da RFC-0001; contradição direta com o `_DESIGN_BRIEF.md`; cabeçalho de metadados totalmente ausente. |
| **Alto** | Defeito sério de conformidade/consistência que engana o leitor/implementador. | Campo de metadados ausente/inválido; link normativo quebrado; convenção de evento/métrica/erro/URN violada; requisito referenciando ID inexistente; redefinição de contrato secundário. |
| **Médio** | Lacuna de qualidade que reduz confiabilidade, sem quebrar contrato. | Seção sem exemplo/tabela exigidos; requisito órfão (sem UC/teste); termo técnico ausente do glossário; ausência de diagrama recomendado. |
| **Baixo** | Melhoria/estilo; baixo risco. | Âncora inexistente; link absoluto onde caberia relativo; inconsistência menor de formatação; arquivo extra não previsto. |

`TBD` em `Status: Draft` **não** é defeito; em `Review`/`Stable` é no mínimo **Alto**.

## 10. Formato EXATO do relatório de saída

Produza **um único relatório** em Markdown com esta estrutura (sem editar arquivos):

```
# Relatório de Auditoria — Módulo NNN-Nome

- Data da auditoria: AAAA-MM-DD
- Auditor: doc-consistency-reviewer (READ-ONLY)
- Fontes normativas consultadas: MODULE_TEMPLATE.md · _DESIGN_BRIEF.md · RFC-0001 · Glossary.md
- Veredito: APROVADO | REPROVADO   (REPROVADO se houver ≥1 Bloqueante ou ≥1 Alto)

## Placar de conformidade

| Eixo | Resultado |
|------|-----------|
| Completude (26 docs)        | X/26 presentes |
| Cabeçalhos de metadados     | X/26 conformes |
| DoD por documento           | X achados |
| Consistência (brief/RFC/glo)| X achados |
| Referências cruzadas        | X quebradas / Y verificadas |
| Convenções de nomenclatura  | X violações |
| Rastreabilidade             | X achados |
| TOTAL de achados            | Bloq: X · Alto: Y · Médio: Z · Baixo: W |

## Bloqueante

### [B-01] <título curto do achado>
- Arquivo: `docs/NNN-Nome/<arquivo>.md` — seção "<seção>" (linha ~NN)
- Regra violada: <regra + fonte, ex.: "MODULE_TEMPLATE §26 docs" / "RFC-0001 §5.2">
- Evidência: `<trecho ofensor curto>`
- Correção sugerida: <ação concreta e acionável>

## Alto
### [A-01] ...  (mesmo formato)

## Médio
### [M-01] ...

## Baixo
### [L-01] ...

## Limitações da auditoria
<fontes que não puderam ser lidas, verificações não realizadas, dúvidas classificadas
como Baixo>
```

Regras do relatório:
- IDs sequenciais por severidade (`B-01`, `A-01`, `M-01`, `L-01`).
- Todo achado cita **arquivo + seção/linha** e **evidência** (trecho curto real do
  arquivo). Sem evidência, não é achado — é dúvida (vai para "Limitações" ou vira Baixo).
- A "Correção sugerida" é **específica** (o que mudar e para qual valor), não genérica.
- Se um eixo estiver 100% conforme, registre-o no placar mesmo sem achados.

## 11. Exemplos de achados bem escritos

**Bloqueante — documento obrigatório ausente**
```
### [B-01] Documento obrigatório ausente: Testing.md
- Arquivo: docs/006-Kernel/ (inventário do módulo)
- Regra violada: MODULE_TEMPLATE.md — "26 documentos obrigatórios" (item 19)
- Evidência: `Glob docs/006-Kernel/*.md` retorna 9 arquivos; Testing.md não consta.
- Correção sugerida: Criar docs/006-Kernel/Testing.md derivado do §7 (NFRs) e §9
  (FailureRecovery) do _DESIGN_BRIEF.md, com estratégia unit/integration/contract/e2e/
  chaos/load e a matriz de rastreabilidade Requisito→Teste.
```

**Alto — contradição de detalhe com o brief**
```
### [A-03] Código de erro divergente do _DESIGN_BRIEF.md
- Arquivo: docs/006-Kernel/API.md — seção "Erros" (linha ~180)
- Regra violada: _DESIGN_BRIEF.md §5.2 (catálogo de erros) — precedência do brief
- Evidência: API.md usa `AIOS-KERNEL-2002` para transição inválida; o brief define
  `AIOS-KERNEL-0002` (HTTP 409, não-retriable).
- Correção sugerida: Renomear para `AIOS-KERNEL-0002` e alinhar HTTP/`retriable` ao
  brief. Códigos do Kernel usam a faixa `0001–0099` (brief §5.2).
```

**Alto — convenção de nomenclatura de evento**
```
### [A-05] Subject de evento sem segmento <tenant>
- Arquivo: docs/006-Kernel/Events.md — tabela "Eventos emitidos" (linha ~40)
- Regra violada: RFC-0001 §5.3 — `aios.<tenant>.<dominio>.<entidade>.<acao>`
- Evidência: `aios.agent.lifecycle.spawned` (falta o segmento <tenant>).
- Correção sugerida: Usar `aios.<tenant>.agent.lifecycle.spawned`; manter o `type`
  CloudEvents como `aios.agent.lifecycle.spawned` (sem tenant), conforme distinção
  subject×type do brief §6.
```

**Médio — requisito órfão na matriz de rastreabilidade**
```
### [M-02] FR-009 sem caso de uso nem teste mapeados
- Arquivo: docs/006-Kernel/Requirements.md — "Matriz Requisito→UseCase→Teste"
- Regra violada: MODULE_TEMPLATE DoD — "Requisitos numerados e rastreáveis"
- Evidência: FR-009 (hibernação/resume) não aparece em nenhuma linha da matriz; sem
  UC-NNN nem teste associados.
- Correção sugerida: Mapear FR-009 a um UC de resume de agente Hibernated (UseCases.md)
  e a um teste de restauração de checkpoint (Testing.md).
```

**Baixo — termo redefinido localmente**
```
### [L-01] Redefinição local de termo do glossário: "ACB"
- Arquivo: docs/006-Kernel/Vision.md — seção "Conceitos" (linha ~55)
- Regra violada: MODULE_TEMPLATE DoD — "Termos reutilizam o Glossary (sem redefinir)"
- Evidência: "Para este módulo, ACB é a estrutura que guarda apenas o estado de execução."
  — diverge do glossário (identidade, estado, cotas, prioridade, ponteiros, política).
- Correção sugerida: Remover a definição local e referenciar `../040-Glossary/Glossary.md`
  (verbete ACB); se o escopo do módulo agrega nuance, propor edição ao glossário via PR.
```
