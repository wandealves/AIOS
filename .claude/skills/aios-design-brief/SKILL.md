---
name: aios-design-brief
description: Procedimento canônico para escrever o `_DESIGN_BRIEF.md` (fonte única de verdade interna) de um módulo AIOS no padrão de ouro do 006-Kernel. Use ao criar ou atualizar o design brief de qualquer módulo AIOS (`docs/<NNN>-<Nome>/_DESIGN_BRIEF.md`) antes de gerar os 26 documentos derivados. Cobre o cabeçalho de metadados, a estrutura das ~13 seções seção-a-seção, a regra de faixa de ADR por módulo, como referenciar (sem redefinir) a RFC-0001 e o Glossário 040, trechos-modelo prontos e um checklist de completude.
---

# Skill: Autoria de `_DESIGN_BRIEF.md` de módulo AIOS

Esta skill descreve **como escrever um design brief** de módulo AIOS no padrão de
ouro de `docs/006-Kernel/_DESIGN_BRIEF.md` (e do segundo exemplo,
`docs/009-Scheduler/_DESIGN_BRIEF.md`). O brief é a **fonte única de verdade
interna** do módulo: os 26 documentos obrigatórios (ver
`docs/_templates/MODULE_TEMPLATE.md`) derivam dele e **NÃO PODEM contradizê-lo**.

Idioma: **pt-BR**. Palavras normativas conforme RFC 2119: **DEVE, NÃO DEVE, DEVERIA,
NÃO DEVERIA, PODE**. Diagramas **SEMPRE em ASCII**.

---

## 1. Antes de escrever (preparação)

1. **Identifique o módulo alvo** (número + nome, ex.: `010-Memory`). Um brief por vez.
2. **Leia os exemplos e contratos** (não redefina nada que já exista):
   - `docs/006-Kernel/_DESIGN_BRIEF.md` — padrão de ouro (estrutura exata).
   - `docs/009-Scheduler/_DESIGN_BRIEF.md` — segundo exemplo.
   - `docs/_templates/MODULE_TEMPLATE.md` — os 26 docs que o brief alimenta.
   - `docs/README.md` — os 41 módulos, foco de cada um, stack.
   - `docs/001-Architecture/Architecture.md` — arquitetura de referência.
   - `docs/003-RFC/RFC-0001-Architecture-Baseline.md` — contratos centrais
     (URN §5.1, envelope CloudEvents §5.2–§5.3, erro RFC 7807 §5.4, idempotência
     §5.5, correlação §5.6, segurança §6, privacidade §7, *default deny* §5.8).
   - `docs/040-Glossary/Glossary.md` — vocabulário. **Reutilize os termos existentes.**
3. **Mapeie os donos reais** das não-responsabilidades: para cada coisa que o módulo
   NÃO faz, aponte o módulo numerado que É o dono (ex.: política → `022-Policy`,
   materialização de runtime → `007`/`008`, custo/orçamento → `026-Cost-Optimizer`,
   auditoria imutável → `025-Audit`, inferência LLM → `017-Model-Router`).

---

## 2. Cabeçalho de metadados (bloco `---` no topo)

Todo brief começa com este bloco. Preencha ADRs/RFCs e dependências para o módulo alvo.

```
---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: <NNN>-<Nome>
Status: Draft
Versão: 0.1
Última atualização: <AAAA-MM-DD>
Responsável (RACI-A): Arquiteto do Módulo <NNN>-<Nome>
ADRs relacionados: ADR-0001..ADR-0010 (globais, herdados); ADR-<NNN×10>..ADR-<NNN×10+9> (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-<a propor> (deste módulo)
Depende de: <lista de módulos numerados acoplados>
---
```

Logo após o cabeçalho, inclua o título `# <NNN>-<Nome> — Design Brief (Fonte Única
de Verdade)` e **dois blocos de citação normativos**: (a) autoridade do documento e
regra de não-contradição/não-redefinição de contratos; (b) palavras RFC 2119.
Copie o tom dos exemplos.

---

## 3. Estrutura seção-a-seção (as ~13 seções)

Reproduza estas seções na ordem, adaptadas ao domínio do módulo. (Opcionalmente, uma
seção `## 0. Posição no AIOS e contrato de fronteira` com um diagrama ASCII de fronteira,
como no 009.)

### Seção 1 — Responsabilidades e Não-Responsabilidades
- **1.1 Missão**: 1–2 parágrafos definindo o papel do módulo (use analogia de SO
  quando couber) e sua fronteira de confiança.
- **1.2 Responsabilidades** (o módulo DEVE): tabela `| # | Responsabilidade |` com IDs
  `R-01`, `R-02`, ...
- **1.3 Não-Responsabilidades** (o módulo NÃO DEVE): tabela
  `| # | Não-Responsabilidade | Dono real |` com IDs `NR-01`/`N-01`, ... **Cada linha
  aponta o módulo dono.**

### Seção 2 — Decomposição em Componentes Internos
- **2.1** Tabela `| Componente | Responsabilidade | Depende de |` com nomes em
  **PascalCase** (ex.: `SyscallGateway`, `AcbStore`, `EventEmitter`).
- **2.2** **Diagrama ASCII** do serviço (`<NOME> SERVICE (<NNN> · .NET 10)`) mostrando
  componentes internos e setas para dependências externas (022, 024, 025, NATS, PG, Redis).

### Seção 3 — Modelo de Dados / Entidades Canônicas
- Nota de alinhamento à RFC-0001 §5.1 (URN, ULID) e RLS por `tenant_id`.
- Uma subseção por entidade com tabela DDL (formato em §5 desta skill).
- Índices por entidade. Diagrama ASCII de relações (cardinalidades).

### Seção 4 — Máquina de Estados Canônica (quando aplicável)
- **4.1** Tabela de estados `| Estado | Descrição | Terminal? |`.
- **4.2** Tabela de transições `| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |`
  + lista de **invariantes** (`I1`, `I2`, ...).
- **4.3** **Diagrama de estados ASCII**.
- Se o módulo **não** tiver FSM relevante (ex.: um roteador stateless), declare
  explicitamente "Este módulo não possui máquina de estados canônica porque ..." e
  descreva o ciclo de vida de requisição, mantendo a numeração das seções.

### Seção 5 — Superfície de API (REST + gRPC)
- Nota: autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7;
  pacote gRPC `aios.<modulo>.v1`; base REST `/v1/<modulo>`.
- Tabela de operações `| Operação | REST | gRPC (rpc) | Idempotente | Notas |`.
- **Catálogo de erros** `AIOS-<DOMINIO>-<NNNN>`: `| Código | HTTP | retriable | Significado |`.
  Reserve o domínio de erro do módulo (ex.: `MEM`, `CTX`, `SCHED`) e sua faixa `0001–0099`.

### Seção 6 — Catálogo de Eventos NATS
- Nota: subjects `aios.<tenant>.<dominio>.<entidade>.<acao>`, envelope CloudEvents
  (RFC-0001 §5.2), entrega at-least-once + dedupe por `event.id`, streams JetStream.
- **6.1 Emitidos** (produtor = este módulo): `| Subject | type | Quando | Stream |`.
- **6.2 Consumidos**: `| Subject assinado | Produtor | Ação do módulo |`.

### Seção 7 — Requisitos Funcionais e Não-Funcionais
- **7.1 FR**: `| ID | Requisito | Prioridade | Critério de aceite |` com `FR-001`... e
  prioridade MoSCoW (Must/Should/Could).
- **7.2 NFR**: `| ID | Atributo | Meta (SLO) | SLI / método |` com **números concretos**:
  latência p99, throughput/réplica, disponibilidade (ex.: 99,95%), escalabilidade (10⁶),
  durabilidade (RPO), recuperação (RTO), consistência, segurança, observabilidade, idempotência.

### Seção 8 — Chaves de Configuração Principais
- `| Chave | Default | Faixa | Escopo | Recarregável | Descrição |`. Prefixo de env var
  `AIOS_<MODULO>_`. Escopos: `global`/`tenant`/`agent`. Marque o que exige migração como
  não-recarregável.

### Seção 9 — Modos de Falha e Recuperação
- `| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |`.
- Metas **RTO ≤ 15 min, RPO ≤ 5 min** (control plane) salvo justificativa. Degradação
  graciosa: preferir **fail-safe / default deny**.

### Seção 10 — Escalabilidade e Concorrência
- Tabela `| Dimensão | Estratégia |`: sharding determinístico `hash(tenant_id, agent_id)
  mod N`, estado quente (Redis) vs. fonte da verdade (PostgreSQL), OCC vs. locks,
  backpressure, caminho quente sem I/O bloqueante, e uma linha **"Rumo a milhões"** de
  agentes. Opcional: mini-diagrama ASCII de particionamento.

### Seção 11 — ADRs e RFCs a Propor
- Nota da **faixa de ADR** do módulo (ver §4 desta skill).
- Tabela `| ADR | Título proposto |` na faixa correta.
- Tabela `| RFC | Título proposto | Relação |` com RFC-0001 como **baseline herdada**.

### Seção 12 — Decisões de Segurança
- **12.1 AuthN/AuthZ**: AuthN no Gateway (OAuth2/OIDC), mTLS interno (RFC-0001 §6);
  módulo como **PEP** consultando **PDP do 022** com *default deny*; isolamento
  multi-tenant (rejeitar `tenant` divergente).
- **12.2 STRIDE**: `| Ameaça | Vetor no <módulo> | Mitigação |` cobrindo as **6**
  categorias (Spoofing, Tampering, Repudiation, Information disclosure, Denial of
  service, Elevation of privilege).
- **12.3 LGPD/GDPR**: minimização, retenção, direito ao esquecimento, segregação.

### Seção 13 — Referências
- Links relativos: Vision, Architecture, Glossary, RFC-0001, template, README do módulo
  e módulos acoplados. Encerre com uma linha itálica reafirmando que o brief governa os
  26 documentos e que qualquer conflito é defeito do documento derivado.

---

## 4. Regra de faixa de ADR por módulo (obrigatória)

Cada módulo tem uma faixa reservada de 10 ADRs: **`ADR-<NNN×10>`..`ADR-<NNN×10+9>`**.

| Módulo | Faixa de ADR |
|--------|--------------|
| 006-Kernel | ADR-0060..ADR-0069 |
| 009-Scheduler | ADR-0090..ADR-0099 |
| 010-Memory | ADR-0100..ADR-0109 |
| 017-Model-Router | ADR-0170..ADR-0179 |
| 022-Policy | ADR-0220..ADR-0229 |

Fórmula geral: início = `NNN × 10`, fim = `NNN × 10 + 9` (zero-padded a 4 dígitos).
ADR-0001..ADR-0010 são globais/herdados. Registre os novos em `docs/002-ADR/`.

---

## 5. Trechos-modelo (copie e adapte)

### 5.1 Tabela de Responsabilidades / Não-Responsabilidades

```
### 1.2 Responsabilidades (o <Módulo> DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Expor a superfície de <verbos/endpoints> via REST (externo) e gRPC (interno). |
| R-02 | Manter o <Store autoritativo> como registro da verdade de <entidade>. |
| R-03 | Atuar como PEP: consultar o PDP do 022-Policy antes de toda ação privilegiada. *Default deny*. |

### 1.3 Não-Responsabilidades (o <Módulo> NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Decidir política (ser PDP). O módulo é PEP; a decisão é do 022. | 022-Policy |
| NR-02 | Persistir a trilha de auditoria imutável (o módulo emite; o Audit persiste). | 025-Audit |
| NR-03 | Definir orçamento/custo por tenant (o módulo aplica; o orçamento é do 026). | 026-Cost-Optimizer |
```

### 5.2 Tabela DDL de entidade de dados

```
### 3.1 Entidade `<Entidade>` — tabela `<schema>.<tabela>`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:<tipo>:<ULID>` (RFC-0001 §5.1). |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento multi-tenant. |
| `state` | `text` | NOT NULL, CHECK ∈ <Enum> | Estado da FSM (§4). |
| `version` | `bigint` | NOT NULL default 0 | Concorrência otimista (OCC). |
| `created_at` | `timestamptz` | NOT NULL | Criação (RFC 3339 UTC). |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, <chave natural>)`; `btree(tenant_id, state)`.
```

### 5.3 Linha de tabela de erro e de evento

```
| `AIOS-<DOMINIO>-0001` | 403 | não | Capability negada pelo PDP (*default deny*). |
| `AIOS-<DOMINIO>-0002` | 429 | sim  | Cota/limite excedido. |
```

```
| `<entidade>.<acao>` | `aios.<entidade>.<acao>` | <quando ocorre> | <STREAM_DURAVEL> |
```

### 5.4 Linha de NFR com SLO numérico

```
| NFR-001 | Desempenho | p99 do caminho quente ≤ 20 ms | histograma `aios_<modulo>_<op>_latency_ms` |
| NFR-002 | Escalabilidade | Suportar ≥ 10⁶ <entidades> com custo sub-linear | teste de escala `Scalability.md` |
| NFR-003 | Disponibilidade | ≥ 99,95% (control plane) | uptime probe / error budget |
```

---

## 6. Como referenciar (sem redefinir) RFC-0001 e o Glossário

- **NÃO** redefina URN, envelope de evento, envelope de erro, idempotência,
  correlação ou subjects. **Referencie** por seção, ex.: "conforme RFC-0001 §5.5" ou
  "envelope CloudEvents (RFC-0001 §5.2)".
- **Reutilize** os termos do Glossário 040 exatamente (ex.: *Kernel Cognitivo*, *ACB*,
  *Syscall Cognitiva*, *tenant*, *shard*). Não crie sinônimos nem redefina termos.
- Ao usar um contrato central, cite a seção da RFC; ao usar um termo, assuma a
  definição do Glossário sem repeti-la.

---

## 7. Checklist de completude (valide antes de finalizar)

- [ ] Cabeçalho de metadados completo; faixa de ADR correta (`NNN×10`..`NNN×10+9`).
- [ ] Dois blocos normativos (autoridade + RFC 2119) após o título.
- [ ] Seção 1 com missão, Responsabilidades (`R-NN`) e Não-Responsabilidades
      (`NR-NN`) **com dono real** em cada linha.
- [ ] Seção 2 com componentes **PascalCase** + **diagrama ASCII**.
- [ ] Seção 3 com tabelas DDL (tipos PG, constraints), `tenant_id`+RLS, URN/ULID,
      índices e diagrama de relações ASCII.
- [ ] Seção 4 com FSM (estados+terminais, transições+guardas, invariantes, diagrama
      ASCII) **ou** justificativa explícita de ausência de FSM.
- [ ] Seção 5 com REST+gRPC (`aios.<modulo>.v1`, `/v1/<modulo>`) e catálogo de erros
      `AIOS-<DOMINIO>-<NNNN>` (HTTP + `retriable`).
- [ ] Seção 6 com eventos emitidos e consumidos, subjects
      `aios.<tenant>.<dominio>.<entidade>.<acao>`, streams JetStream.
- [ ] Seção 7 com FR (MoSCoW + critério de aceite) e NFR com **SLO numéricos** (p99,
      throughput, 99,95%, 10⁶, RPO/RTO).
- [ ] Seção 8 com chaves de config, **defaults**, faixa, escopo, recarregável;
      prefixo `AIOS_<MODULO>_`.
- [ ] Seção 9 com modos de falha e RTO ≤ 15 min / RPO ≤ 5 min; degradação fail-safe.
- [ ] Seção 10 com sharding, estado externalizado, backpressure, "rumo a milhões".
- [ ] Seção 11 com ADRs na faixa correta e RFCs (RFC-0001 herdada).
- [ ] Seção 12 com AuthN/AuthZ (PEP→PDP, mTLS, default deny), **STRIDE (6 categorias)**
      e **LGPD/GDPR**.
- [ ] Seção 13 com referências relativas e nota final de governança.
- [ ] Convenções: pt-BR, RFC 2119, diagramas ASCII, PascalCase, subjects/métricas/erros/
      URN no padrão. Sem placeholders, sem `TODO`, sem números de módulo inexistentes.
- [ ] Nenhum contrato central redefinido; Glossário reutilizado sem sinônimos.

---

## 8. Saída

Grave o arquivo em `docs/<NNN>-<Nome>/_DESIGN_BRIEF.md`. Ao concluir, informe o
caminho, um resumo de 3 linhas do que o brief define e quaisquer pontos que exijam
validação humana (FSM não aplicável, dependências ambíguas, domínio de erro escolhido).
