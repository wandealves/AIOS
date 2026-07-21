---
name: aios-module
description: Procedimento operacional para gerar os 26 documentos obrigatórios de um módulo AIOS (NNN-Nome) a partir do seu _DESIGN_BRIEF.md. Use ao documentar/preencher um módulo do repositório de documentação AIOS — traz os esqueletos dos 26 docs, o cabeçalho de metadados exato, o Definition of Done por documento, a ordem de escrita, as regras de consistência (Glossário 040, RFC-0001, brief) e a convenção de nomes de arquivo. Invocada pelo subagente `module-doc-writer`.
---

# Skill: aios-module — Gerar os 26 documentos de um módulo AIOS

Este é o **procedimento operacional** para produzir a documentação completa de **um**
módulo `NNN-Nome` do AIOS. O AIOS é um repositório de **documentação técnica em pt-BR**
(não é código). Cada módulo DEVE ter os **26 documentos obrigatórios** definidos em
`docs/_templates/MODULE_TEMPLATE.md`, todos **derivados** do `_DESIGN_BRIEF.md` do
módulo, que é a **fonte única de verdade**.

> Contexto imprescindível a ler antes de escrever qualquer documento:
> - `docs/_templates/MODULE_TEMPLATE.md` — normativo: 26 docs, cabeçalho, convenções, DoD.
> - `docs/NNN-Nome/_DESIGN_BRIEF.md` — fonte única de verdade do módulo alvo.
> - `docs/003-RFC/RFC-0001-Architecture-Baseline.md` — contratos centrais (reutilizar, não redefinir).
> - `docs/040-Glossary/Glossary.md` — vocabulário canônico (reutilizar, não redefinir).
> - `docs/006-Kernel/_DESIGN_BRIEF.md` — exemplo do padrão de profundidade esperado.
> - `docs/README.md` e `docs/prompt.md` — índice dos 41 módulos e missão global.

---

## 0. Pré-requisitos (portão de entrada)

Antes de gerar qualquer documento:

1. **O brief existe?** Verifique `docs/NNN-Nome/_DESIGN_BRIEF.md`.
   - **Existe** → prossiga.
   - **Não existe** → NÃO prossiga. Ou peça o brief ao chamador, ou rascunhe um brief
     mínimo (derivado da linha do módulo no `docs/README.md`, da missão em
     `docs/prompt.md` e da RFC-0001) e **confirme com o chamador antes** de gerar os
     26 docs. Um brief não confirmado nunca vira base dos 26 documentos.
2. **Leia o brief inteiro** e extraia: responsabilidades/não-responsabilidades,
   componentes (PascalCase), modelo de dados, máquina de estados, API, eventos, FR/NFR
   (com SLO/SLI), configuração, falhas, escalabilidade, segurança, e as ADRs/RFCs
   "a propor" e "de que depende".
3. **Leia o template**, a **RFC-0001** e o **Glossário 040**. Anote os contratos que
   você vai referenciar (URN, envelope de evento, envelope de erro, idempotência,
   correlação, subjects) para **não redefini-los**.
4. **Escopo:** UM módulo por execução. Se pedirem vários, faça um e reporte o resto.

---

## 1. Cabeçalho de metadados obrigatório (topo de TODO documento)

Todo `.md` gerado DEVE começar com este bloco. Copie ADRs/RFCs/dependências do brief.

```
---
Documento: <Nome do Documento>
Módulo: NNN-Nome
Status: Draft | Review | Stable | Deprecated
Versão: MAJOR.MINOR
Última atualização: AAAA-MM-DD
Responsável (RACI-A): <papel/equipe>
ADRs relacionados: ADR-XXXX, ADR-YYYY
RFCs relacionados: RFC-XXXX
Depende de: <módulos/documentos>
---
```

### Exemplo preenchido (documento `Architecture.md` do módulo 006-Kernel)

```
---
Documento: Architecture
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0001, ADR-0060, ADR-0061, ADR-0063, ADR-0064
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: 001-Architecture, 022-Policy, 009-Scheduler, 008-Agent-Lifecycle, 020-Communication
---
```

Regras do cabeçalho:
- `Status` inicial dos docs derivados é geralmente `Draft` (só em `Draft` é permitido
  algum "TBD"; ainda assim, prefira sempre conteúdo real).
- `Versão` começa em `0.1` para docs novos; incrementa MINOR a cada revisão relevante.
- `Última atualização` = data corrente (formato `AAAA-MM-DD`).
- `ADRs/RFCs relacionados` e `Depende de` DEVEM refletir o brief; não invente IDs.

---

## 2. Os 26 documentos — nome de arquivo e conteúdo esperado

Escreva exatamente estes 26 arquivos em `docs/NNN-Nome/` (nomes canônicos,
`PascalCase.md`, sem numerar o arquivo). Cada resumo abaixo é o esqueleto mínimo —
sempre com prosa técnica + tabelas + diagramas ASCII + exemplos onde indicado.

| # | Arquivo | Conteúdo esperado (derivado do brief) |
|---|---------|----------------------------------------|
| 1 | `Vision.md` | Propósito do módulo, problema que resolve, escopo in/out, personas, princípios de design, **não-objetivos explícitos**, e relação com `../000-Vision/Vision.md`. Baseie-se em "Missão/Responsabilidades/Não-Responsabilidades" do brief. |
| 2 | `Architecture.md` | Visão C4 (Context → Container → Component) em ASCII, decomposição interna, responsabilidade por componente (PascalCase), fronteiras, padrões arquiteturais, tecnologias da stack e **justificativa** de cada escolha, alternativas descartadas (link para ADRs). Reaproveite o diagrama de componentes do brief. |
| 3 | `Requirements.md` | Índice consolidado de requisitos + **matriz de rastreabilidade** Requisito → UseCase → Teste, stakeholders, e glossário local (apenas referenciando o 040). |
| 4 | `FunctionalRequirements.md` | Tabela `FR-NNN` (id, descrição, prioridade MoSCoW, critério de aceite, origem). Cada FR mensurável e verificável. Origem = seção 7.1 do brief. |
| 5 | `NonFunctionalRequirements.md` | Tabela `NFR-NNN` por atributo de qualidade (performance, escalabilidade, disponibilidade, segurança, observabilidade, manutenibilidade) com **metas numéricas (SLO/SLI)** e método de verificação. Metas idênticas às do brief. |
| 6 | `UseCases.md` | Casos de uso `UC-NNN` no formato ator / pré-condições / fluxo principal / fluxos alternativos / exceções / pós-condições. Cobrir os verbos/fluxos principais do módulo. |
| 7 | `SequenceDiagrams.md` | Diagramas de sequência **ASCII** dos fluxos críticos (caminho feliz **e** de falha): participantes, mensagens síncronas/assíncronas, timeouts, correlação. |
| 8 | `ClassDiagrams.md` | Diagramas de classe/estrutura **ASCII**, interfaces públicas, contratos, invariantes, relações (composição/agregação/dependência) entre os componentes do brief. |
| 9 | `StateMachine.md` | Máquina(s) de estado: estados, transições, gatilhos, guardas, ações de entrada/saída, estados terminais, **invariantes**. Reproduza e detalhe a FSM da seção 4 do brief (diagrama ASCII + tabela de transições). |
| 10 | `Database.md` | Modelo físico PostgreSQL: DDL, índices (pgvector/AGE quando aplicável), particionamento, RLS por `tenant_id`, retenção, migrações, chaves e constraints. Baseie-se nas entidades da seção 3 do brief. |
| 11 | `API.md` | Contratos REST (OpenAPI) e gRPC (proto): versionamento, autenticação, códigos de erro `AIOS-<DOMINIO>-<NNNN>`, idempotência (`Idempotency-Key`), paginação, exemplos request/response. Mapeie a superfície da seção 5 do brief. |
| 12 | `Events.md` | Catálogo de eventos NATS: subject (`aios.<tenant>.<dominio>.<entidade>.<acao>`), schema JSON (envelope CloudEvents da RFC-0001), produtor, consumidores, semântica de entrega (at-least-once/exactly-once), ordenação, versionamento de schema. Seção 6 do brief. |
| 13 | `Configuration.md` | Todas as chaves de config: tipo, default, faixa válida, escopo (global/tenant/agente), recarregável em runtime?, variável de ambiente (prefixo do módulo). Seção 8 do brief. |
| 14 | `Deployment.md` | Topologia Docker/Compose, recursos (CPU/RAM/GPU), réplicas, dependências, health/readiness, estratégia de rollout, papel do YARP/gateway. |
| 15 | `Security.md` | AuthN/AuthZ (OAuth2/OIDC, RBAC/ABAC), superfície de ataque, **threat model STRIDE**, segredos, TLS/mTLS, sandbox, LGPD/GDPR, controles. Seção 12 do brief. |
| 16 | `Monitoring.md` | Dashboards, alertas (regras Prometheus), SLO/SLA/SLI, golden signals, runbooks associados. Consistente com os NFRs. |
| 17 | `Logging.md` | Eventos de log estruturado (Serilog → Seq), níveis, campos obrigatórios, correlação (`trace_id`/`span_id`/`tenant_id`), retenção. |
| 18 | `Metrics.md` | Catálogo de métricas OpenTelemetry/Prometheus: nome `aios_<subsistema>_<nome>_<unidade>`, tipo (counter/gauge/histogram), labels, unidade, semântica. Inclua as métricas citadas nos NFRs. |
| 19 | `Testing.md` | Estratégia de testes (unit/integration/contract/e2e/chaos/load), cobertura-alvo, fixtures, ambientes, critérios de gate. Ligue casos de teste aos FR/NFR. |
| 20 | `Benchmark.md` | Metodologia, cargas, ambiente, KPIs, resultados-alvo, comparação com baseline, reprodutibilidade. Alvos coerentes com os NFRs de desempenho/throughput. |
| 21 | `FailureRecovery.md` | Modos de falha (FMEA), detecção, isolamento, recuperação, idempotência, retries/backoff, DLQ, **RTO/RPO**, degradação graciosa. Seção 9 do brief. |
| 22 | `Scalability.md` | Modelo de escala (horizontal/vertical), particionamento/sharding, limites teóricos, concorrência, locks, backpressure, caminho para milhões de agentes. Seção 10 do brief. |
| 23 | `Examples.md` | Exemplos executáveis (CLI/SDK/API), do "hello world" ao avançado, snippets comentados. Use os verbos/endpoints reais do módulo. |
| 24 | `FAQ.md` | Perguntas frequentes, armadilhas comuns, esclarecimentos de escopo (o que o módulo NÃO faz — a partir das Não-Responsabilidades). |
| 25 | `ADR.md` | Índice das ADRs que afetam o módulo (link para `../002-ADR/`), com resumo de 1 linha por decisão e seu status. Use a lista "ADRs a propor" do brief. |
| 26 | `RFC.md` | Índice das RFCs relacionadas (link para `../003-RFC/`), status e impacto no módulo. Use a lista de RFCs do brief. |

---

## 3. Ordem recomendada de escrita

Escreva na ordem em que um documento fornece insumo ao seguinte, minimizando
retrabalho e maximizando consistência:

1. **Fundacionais** → `Vision.md`, `Architecture.md`.
2. **Requisitos** → `FunctionalRequirements.md`, `NonFunctionalRequirements.md`,
   depois `Requirements.md` (consolida e cria a matriz de rastreabilidade dos dois
   anteriores).
3. **Comportamento e contratos** → `UseCases.md`, `StateMachine.md`, `ClassDiagrams.md`,
   `SequenceDiagrams.md`, `Database.md`, `API.md`, `Events.md`.
4. **Operação** → `Configuration.md`, `Deployment.md`, `Security.md`.
5. **Observabilidade** → `Monitoring.md`, `Logging.md`, `Metrics.md`.
6. **Garantia e escala** → `Testing.md`, `Benchmark.md`, `FailureRecovery.md`,
   `Scalability.md`.
7. **Apoio ao usuário** → `Examples.md`, `FAQ.md`.
8. **Governança** → `ADR.md`, `RFC.md`.

Após o grupo 1–2, congele os IDs (`FR-NNN`, `NFR-NNN`, `UC-NNN`) e reutilize-os
literalmente nos demais documentos.

---

## 4. Definition of Done — por documento

Um documento só está pronto quando **todos** os itens abaixo são verdadeiros:

- [ ] Cabeçalho de metadados preenchido (seção 1) e coerente com o brief.
- [ ] Nenhuma seção do esqueleto vazia (`TBD` só é permitido em `Status: Draft`, e
      ainda assim evite).
- [ ] Ao menos **1 diagrama ASCII** quando o esqueleto exige (Architecture,
      SequenceDiagrams, ClassDiagrams, StateMachine, Deployment, Scalability).
- [ ] Ao menos **1 tabela** e **1 exemplo** quando aplicável.
- [ ] Palavras normativas RFC 2119 (DEVE/NÃO DEVE/DEVERIA/PODE) usadas em todo
      requisito/regra.
- [ ] Termos reutilizam `../040-Glossary/Glossary.md` — **sem redefinir**.
- [ ] Contratos reutilizam a RFC-0001 (URN, evento, erro, idempotência, correlação,
      subjects) — **sem redefinir**.
- [ ] Decisões apontam para uma ADR (não decida localmente); nada contradiz o brief.
- [ ] Requisitos numerados e rastreáveis (`FR/NFR/UC` ligados a testes).
- [ ] Riscos e alternativas explicitados onde couber.
- [ ] Referências cruzadas por **caminho relativo** válido
      (ex.: `[Policy](../022-Policy/API.md)`).
- [ ] Identificadores no padrão: componentes `PascalCase`; eventos
      `dominio.entidade.acao`; métricas `aios_<subsistema>_<nome>_<unidade>`;
      erros `AIOS-<DOMINIO>-<NNNN>`.

---

## 5. Regras de consistência (transversais)

- **Fonte única de verdade:** o `_DESIGN_BRIEF.md` do módulo. Qualquer divergência
  entre um documento e o brief é defeito **do documento**.
- **Não redefinir:** URN, envelope CloudEvents, envelope de erro, idempotência,
  correlação, subjects e demais contratos centrais pertencem à RFC-0001; termos
  pertencem ao Glossário 040. Referencie por caminho relativo; nunca recrie a
  definição.
- **Números idênticos em todo lugar:** metas de SLO/SLI, timeouts, defaults de config,
  limites de cota, RTO/RPO — o valor que aparece no brief é o mesmo em NFR, Monitoring,
  Benchmark, FailureRecovery, Configuration e Scalability.
- **Nomes idênticos em todo lugar:** o nome de um componente, estado da FSM, evento,
  código de erro ou chave de config é escrito exatamente igual em todos os 26 docs.
- **Sem novos termos/ADRs/RFCs:** se um termo novo for necessário, sinalize no
  relatório que ele deveria entrar no Glossário 040; se uma decisão nova aparecer,
  referencie uma ADR "a propor" do brief — não crie a ADR/RFC aqui.
- **Não editar a fundação estável** (`000`, `001`, `002`, `003`, `040`) nem o
  `MODULE_TEMPLATE.md`.
- **Profundidade:** escreva como documentação oficial de um SO. Nunca liste apenas
  títulos; cada seção tem prosa, tabela/diagrama e exemplo. O padrão de referência é
  `docs/006-Kernel/_DESIGN_BRIEF.md`.

---

## 6. Nomenclatura de arquivos e diretório

- Diretório do módulo: `docs/NNN-Nome/` (ex.: `docs/009-Scheduler/`), exatamente como
  no `docs/README.md`.
- Os 26 arquivos usam os **nomes canônicos** da tabela da seção 2, em `PascalCase.md`
  (ex.: `Vision.md`, `NonFunctionalRequirements.md`, `FailureRecovery.md`). **Não**
  prefixe com número nem traduza os nomes de arquivo.
- O brief permanece como `_DESIGN_BRIEF.md` (prefixo `_`); não o sobrescreva.
- Se o módulo tiver um `README.md` próprio (índice), atualize-o para apontar aos 26
  docs por caminho relativo, mas não o conte como um dos 26.

---

## 7. Relatório final

Ao concluir, entregue: (a) o módulo tratado; (b) a lista dos 26 caminhos **absolutos**
escritos; (c) quais (se algum) ficaram pendentes e por quê; (d) inconsistências ou
lacunas encontradas no brief que precisam de decisão do chamador; (e) quaisquer termos
que deveriam entrar no Glossário 040.
