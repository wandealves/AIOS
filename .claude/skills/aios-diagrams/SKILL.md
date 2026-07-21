---
name: aios-diagrams
description: Use ao criar ou revisar QUALQUER diagrama da documentação do AIOS — C4 (Contexto/Contêiner/Componente), sequência, máquina de estados, componentes, implantação (deployment), fluxo e relações de dados/ER —, todos SEMPRE em ASCII (nunca Mermaid, PlantUML ou imagens), para portabilidade em Git/PR/terminal. Contém as convenções de estilo (caracteres de caixa e setas, largura máxima, rótulos de mensagem) e um template ASCII copiável e válido para cada tipo, no padrão já usado em `docs/006-Kernel/_DESIGN_BRIEF.md` e `docs/001-Architecture/Architecture.md`, além de um checklist de qualidade. Acione antes de desenhar, editar ou revisar um diagrama em qualquer documento do projeto.
---

# aios-diagrams — Convenções de Diagramas ASCII do AIOS

Todo diagrama da documentação do AIOS é **ASCII puro**, dentro de um bloco de código
cercado (```` ``` ````), para permanecer legível em Git diff, revisão de PR e
terminal. **Nunca** use Mermaid, PlantUML, SVG ou imagens. Adote a notação **C4** e
**UML** adaptadas para ASCII. Esta skill fixa o estilo e fornece um **template
copiável** por tipo, no padrão de ouro de `docs/001-Architecture/Architecture.md` e
`docs/006-Kernel/_DESIGN_BRIEF.md`.

## 1. Regras de estilo gerais (valem para todos os tipos)

- **Largura máxima: ~100 colunas.** Diagramas mais largos quebram em terminais e no
  visualizador de PR. Se não couber, decomponha em dois diagramas ou reduza rótulos.
- **Caracteres de caixa (box-drawing) canônicos:**
  - Cantos: `┌ ┐ └ ┘` — Cruzamentos/junções: `┬ ┴ ├ ┤ ┼`
  - Linhas: `─` (horizontal) e `│` (vertical). **Não** misture com `-`/`|` ASCII
    simples dentro da mesma caixa (quebra o alinhamento visual).
- **Setas (direção do fluxo):** `▶` (direita), `◀` (esquerda), `▲` (cima), `▼` (baixo).
  Combine com a linha: `──▶`, `◀──`, `──┬──`, e para vertical use `│` terminando em
  `▼`/`▲`. Fluxo bidirecional: `◀──▶` ou `◀─▶`.
- **Rótulos nas arestas:** curtos, sobre ou ao lado da linha — ex.: `──REST/SDK──▶`,
  `─ consulta PDP ─▶`, `◀── allow/deny ──`. Prefira verbo ou protocolo.
- **Caixas:** nome do componente em **PascalCase** (ex.: `SyscallGateway`), com uma
  linha opcional de descrição curta entre parênteses. Mantenha as bordas alinhadas
  verticalmente (mesma coluna de `│`).
- **Alinhamento:** conte colunas. Uma caixa desalinhada por 1 caractere é um defeito.
  Use espaços (nunca tabs) para posicionar.
- **Legenda:** quando usar símbolos não óbvios (ex.: `─▶` síncrono vs `┄▶` assíncrono),
  adicione uma linha `Legenda:` logo abaixo do bloco.
- **Idioma:** rótulos em pt-BR; termos técnicos consagrados em inglês são aceitos
  (gateway, scheduler, broker, stream).
- **Identificadores consistentes com o resto do doc:** componentes em PascalCase;
  eventos `aios.<tenant>.<dominio>.<entidade>.<acao>`; métricas
  `aios_<subsistema>_<nome>_<unidade>`. Um componente tem o **mesmo nome** no texto,
  nas tabelas e no diagrama.

---

## 2. C4 — Nível 1: Contexto do Sistema

Mostra o AIOS como uma caixa central e os **atores/sistemas externos** ao redor, com
o protocolo em cada aresta. Um retângulo grande no centro; atores à esquerda,
sistemas externos à direita.

**Regras específicas:** rotule **toda** aresta com o protocolo (REST/SDK, gRPC, A2A,
MCP, OIDC, HTTPS). Atores humanos e sistemas externos ficam FORA da caixa central.

```
                         ┌───────────────────────────────────────────┐
     ┌──────────┐        │                                           │
     │  Dev de  │──REST/ │                                           │        ┌─────────────────┐
     │ Agentes  │  SDK──▶│                                           │──────▶ │  Provedores LLM  │
     └──────────┘        │                  A I O S                  │ HTTPS  │ (OpenAI, local)  │
     ┌──────────┐        │    Artificial Intelligence Operating      │        └─────────────────┘
     │   SRE /  │──gRPC─▶│              System                       │        ┌─────────────────┐
     │ Operador │        │                                           │──MCP──▶│ Ferramentas /    │
     └──────────┘        │   (control plane + data plane +           │        │ Sistemas Externos│
     ┌──────────┐        │    persistência + observabilidade)        │        └─────────────────┘
     │ Agente   │──A2A──▶│                                           │        ┌─────────────────┐
     │ Externo  │        │                                           │──OIDC─▶│  IdP Corporativo │
     └──────────┘        └───────────────────────────────────────────┘◀──────│ (Keycloak/Entra) │
                                                                              └─────────────────┘
Legenda: ──▶ chamada/integração externa · protocolo rotulado em cada aresta.
```

---

## 3. C4 — Nível 2: Contêineres

Zoom no interior do AIOS: os **contêineres executáveis** (Gateway, Control Plane,
Runtime, Barramento, armazenamento) e como se comunicam. Use um retângulo externo
"AIOS" contendo os contêineres; rotule as arestas com o protocolo/tecnologia.

**Regras específicas:** agrupe serviços afins em um sub-retângulo (ex.: "CONTROL
PLANE SERVICES"); marque a tecnologia de cada contêiner (`.NET 10`, `Python`, `NATS`);
mostre o barramento como faixa horizontal e o armazenamento na base.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                     AIOS                                            │
│  ┌────────────────────┐        ┌──────────────────────────────────────────────┐    │
│  │ Web Console (React)│──HTTPS▶│         API Gateway (YARP, .NET 10)          │    │
│  └────────────────────┘        │   AuthN/AuthZ · rate-limit · versão          │    │
│                                └───────────────────────┬──────────────────────┘    │
│                                             REST/gRPC   │                           │
│   ┌─────────────────────────────────────────────────────▼──────────────────────┐   │
│   │              CONTROL PLANE SERVICES (.NET 10)  — stateless                  │   │
│   │  ┌────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌───────────┐   │   │
│   │  │ Kernel │ │Scheduler │ │ Registry │ │ Policy │ │ Memory │ │ModelRouter│   │   │
│   │  └────────┘ └──────────┘ └──────────┘ └────────┘ └────────┘ └───────────┘   │   │
│   └───────────────────────────────────┬────────────────────────────────────────┘   │
│                                        │ pub/sub · request/reply                    │
│   ┌────────────────────────────────────▼───────────────────────────────────────┐   │
│   │                   COMMUNICATION BUS — NATS (JetStream)                       │   │
│   └────────────────────────────────────┬───────────────────────────────────────┘   │
│                                         │                                           │
│   ┌───────────────┐ ┌──────────────────▼──┐ ┌───────────┐ ┌──────────────────────┐ │
│   │  PostgreSQL   │ │        Redis         │ │   MinIO   │ │  Observability Stack │ │
│   │ +pgvector+AGE │ │  cache/estado quente │ │  blobs    │ │ Prometheus·Grafana   │ │
│   └───────────────┘ └─────────────────────┘ └───────────┘ └──────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
Legenda: cada contêiner marca sua tecnologia; barramento e storage na base.
```

---

## 4. C4 — Nível 3: Componentes (e diagrama de componentes com fluxo)

Zoom em **um** serviço: seus componentes internos (PascalCase), as setas de fluxo
entre eles e as saídas para dependências externas (outros módulos, banco, NATS).

**Regras específicas:** um retângulo nomeado com o serviço e o módulo (`KERNEL SERVICE
(006)`); fluxo principal de cima para baixo ou esquerda→direita; dependências externas
saem por arestas rotuladas com o alvo (`──▶ 022-Policy (PDP)`, `──▶ NATS aios.<tenant>...`).
Este é também o template para um **diagrama de componentes** genérico.

```
┌──────────────────────── KERNEL SERVICE (006 · .NET 10) ─────────────────────────┐
│                                                                                  │
│  ┌────────────────┐   ┌────────────────────┐   ┌───────────────────────────┐     │
│  │ SyscallGateway │──▶│ CapabilityEnforcer │──▶│ AcbStore                  │     │
│  │ (valida/despa.)│   │       (PEP)        │   │ (PG + Redis · OCC version) │     │
│  └───────┬────────┘   └─────────┬──────────┘   └─────────────┬─────────────┘     │
│          │                      │ consulta PDP               │                   │
│          │                      ▼                            │                   │
│          │             ┌────────────────┐                    │                   │
│          │             │  PolicyClient  │──────▶ 022-Policy (PDP)                 │
│          │             │  (CB + cache)  │◀──── allow/deny ──                      │
│          ▼             └────────────────┘                    ▼                   │
│  ┌──────────────────┐                          ┌───────────────────────────┐     │
│  │ ResourceQuota    │◀──────── cotas ─────────▶│ LifecycleCoordinator      │     │
│  │ Manager          │                          │ (FSM ACB · sagas)         │     │
│  └───────┬──────────┘                          └─────────────┬─────────────┘     │
│          │ emite eventos                                     │ pede slot         │
│          ▼                                                   ▼                   │
│    ┌─────────────┐                                    009-Scheduler             │
│    │ EventEmitter│──▶ NATS aios.<tenant>.agent.lifecycle.*                       │
│    │ (Outbox)    │                                                              │
│    └─────────────┘                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
Legenda: PascalCase = componente interno · aresta rotulada com alvo/protocolo.
```

---

## 5. Máquina de Estados (UML statechart em ASCII)

Estados em caixas; transições como setas rotuladas com **gatilho** e, entre
colchetes, a **guarda** — `gatilho (T-NN) [guarda]`. Estado inicial vem de `∅ ──▶`;
estados terminais são anotados `(terminal)`. Numere as transições (`T-01`, `T-02`…)
para casar com a tabela de transições do documento.

**Regras específicas:** transições `self-loop` (ex.: `checkpoint`) apontam de volta ao
próprio estado com rótulo `(self)`; sempre exista uma tabela de transições ao lado do
diagrama descrevendo gatilho + guarda de cada `T-NN`.

```
              spawn (T-01)
   ∅ ──────────────────────▶ ┌─────────┐
                             │ Pending │
                             └────┬────┘
          admite (T-02)          │          reject/timeout (T-03)
      ┌───────────────────────────┼──────────────────────────┐
      ▼                           │                          ▼
 ┌──────────┐  boot ok (T-04)     │                     ┌────────┐
 │ Admitted │──────────────────┐  │                     │ Failed │◀── boot fail (T-05)
 └────┬─────┘                  ▼  │                     └────────┘   (terminal)
      │                 ┌───────────┐                         ▲
 resume (T-09)          │  Running  │─── checkpoint (T-13,self)│
      │                 └──┬────┬───┘                          │ runtime fail (T-12)
      │        suspend(T-06)│    │ kill (T-10)                 │
      │                     ▼    └───────────────┐            │
      │               ┌───────────┐         ┌─────────────┐   │
      └───────────────│ Suspended │         │ Terminating │───┘
              idle(T-08)└────┬──────┘        └──────┬──────┘
                             ▼   resume(T-07)       │ drain done (T-11)
                      ┌───────────┐                 ▼
                      │Hibernated │──resume(T-09)─▶ ┌────────────┐
                      └───────────┘                 │ Terminated │  (terminal)
                                                     └────────────┘
Transições: T-01 spawn [cap agent:spawn ∧ cota ok] · T-06 suspend [cap agent:suspend]
            T-10 kill [cap agent:kill ∧ owner|admin] · … (ver tabela do documento).
```

---

## 6. Diagrama de Sequência (UML sequence em ASCII)

Participantes/atores nomeados no topo, cada um com uma **lifeline** vertical (`│`).
Mensagens são setas horizontais entre lifelines, rotuladas com a operação. O tempo
corre de cima para baixo.

**Regras específicas:**
- **Nomeie todos os participantes** no cabeçalho (Cliente, Gateway, Kernel, …).
- **Síncrona (request):** `├──── msg ────▶│`. **Retorno/resposta:** `│◀─── resp ────┤`.
- **Assíncrona (evento/publish):** use seta tracejada `┄┄▶` e declare na legenda —
  ex.: publicação em NATS não bloqueia o chamador.
- **Timeout / deadline:** anote sobre a mensagem entre colchetes — `[timeout 200ms]`
  ou `[deadline p99≤250ms]` — e, se houver caminho de erro, mostre a resposta de erro
  (`◀── 504 AIOS-KERNEL-0004 ──`).
- Mantenha as lifelines na **mesma coluna** do topo ao fim.

```
Cliente    Gateway    Kernel     Policy    Scheduler   Runtime    Memory
  │  POST     │          │          │          │          │          │
  │ /tasks    │          │          │          │          │          │
  ├──────────▶│ authN/Z  │          │          │          │          │
  │           ├─────────▶│ spawn?   │          │          │          │
  │           │          ├─────────▶│ decide   │ [timeout 25ms]       │
  │           │          │◀─────────┤ allow    │          │          │
  │           │          ├─ cotas ok · admite ─▶│ [deadline p99≤250ms]│
  │           │          │          │          ├─ place ─▶│ boot     │
  │           │          │          │          │          ├─ recupera contexto ─▶│
  │           │          │          │          │          │◀── contexto ─────────┤
  │           │          │          │          │          ├─ loop ReAct (tools) ─┐
  │           │          │          │          │          │◀─────────────────────┘
  │◀── 202 Accepted (task_id, stream) ────────────────────┤                      │
  │           │          │  agent.lifecycle.spawned  ┄┄┄┄┄┄┄┄▶ NATS (assíncrono)  │
  └── SSE/stream de eventos ◀──── NATS ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
Legenda: ──▶ síncrono (request/reply) · ┄┄▶ assíncrono (publish NATS, não bloqueia)
         [..] timeout/deadline · resposta de erro usa código AIOS-<DOMINIO>-<NNNN>.
```

---

## 7. Relações de Dados / ER (entidade-relacionamento em ASCII)

Entidades em caixas; cardinalidade nas arestas com a notação `1` e `*` (um-para-muitos
`1───<  *`). Mostre as chaves estrangeiras/ponteiros como arestas rotuladas para o
módulo dono do dado referenciado.

**Regras específicas:** use `1` no lado "um" e `*` (ou `<`/`>` como "pé de galinha"
simplificado) no lado "muitos"; ponteiros para outros módulos rotulam o alvo
(`memory_ptr → 010`); relações reflexivas (árvore) são setas que retornam à mesma
entidade.

```
   ┌────────────┐        ┌────────────────────────┐        ┌────────────┐
   │  Tenant    │1──────*│  AgentControlBlock(ACB)│1──────1│   Quota    │
   └────────────┘        └───────────┬────────────┘        │(scope=agent)│
        │1                           │  \                    └────────────┘
        │                            │   \──▶ memory_ptr  → 010-Memory
        │                            │        context_ptr → 011-Context
        │                            │        policy_ref  → 022-Policy
        │                   ┌────────┴────────┐
        │                  *│                 │*
        │            ┌──────▼──────┐   ┌───────▼──────┐
        │            │SyscallRecord│   │OutboxMessage │  (event-sourced)
        │            └─────────────┘   └──────────────┘
        └───1 Quota (scope=tenant)

   AgentControlBlock.parent_urn ──▶ AgentControlBlock   (árvore de spawn, reflexiva)
Legenda: 1 = lado "um" · * = lado "muitos" · ──▶ ptr/FK rotula o módulo dono.
```

---

## 8. Implantação (Deployment — Docker/Compose)

Nós/zonas de disponibilidade como retângulos; contêineres empilhados dentro de cada
nó; replicação e balanceamento como arestas rotuladas. Reflete o modelo de `docs/001`
e o detalhe de `028-Deployment`.

**Regras específicas:** nomeie zonas (`AZ-1`, `AZ-2`, quorum); marque réplicas
(`Gateway x2`, `ControlPlane xN`); mostre replicação de estado com aresta
bidirecional rotulada (`◀── streaming rep ──▶`); anote o balanceador e a política de
anti-afinidade fora dos nós.

```
┌──────────────────────── CLUSTER (Docker Compose → K8s no futuro) ─────────────────────┐
│                                                                                        │
│  ZONA A (AZ-1)                       ZONA B (AZ-2)                  ZONA C (AZ-3·quorum)│
│  ┌────────────────┐                  ┌────────────────┐            ┌──────────────────┐│
│  │ Gateway   x2   │                  │ Gateway   x2   │            │ NATS quorum node ││
│  │ ControlPlane xN│                  │ ControlPlane xN│            │ PG witness       ││
│  │ Runtime pool   │                  │ Runtime pool   │            └──────────────────┘│
│  │ NATS node      │                  │ NATS node      │                                │
│  │ PG primary     │◀── streaming rep ─▶ PG standby     │                                │
│  │ Redis primary  │◀── replicação ────▶ Redis replica  │                                │
│  └────────────────┘                  └────────────────┘                                │
│                                                                                        │
│  Balanceador L7 (YARP) ──▶ Gateways · Anti-affinity por AZ · autoscaling do Runtime    │
└────────────────────────────────────────────────────────────────────────────────────────┘
Legenda: xN = nº de réplicas · ◀──▶ replicação de estado · nós stateful têm HA por quorum.
```

---

## 9. Diagrama de Camadas / Fluxo (opcional, quando útil)

Para visões em camadas (L0..L4) ou fluxos lineares, use faixas horizontais separadas
por réguas `───`. Mantenha uma etiqueta de camada à esquerda e o conteúdo à direita.

```
  L3  BORDA       API Gateway (YARP) · AuthN/Z · versão · rate-limit
  ──────────────────────────────────────────────────────────────────
  L2  CONTROLE    Kernel · Scheduler · Registry · Policy · Memory · ModelRouter · …
  ──────────────────────────────────────────────────────────────────
  L1.5 BARRAMENTO NATS (JetStream) — pub/sub · request/reply · streams
  ──────────────────────────────────────────────────────────────────
  L1  DADOS       Agent Runtime (Python) — sandbox · loop cognitivo · MCP host
  ──────────────────────────────────────────────────────────────────
  L0  RECURSOS    PostgreSQL(+pgvector+AGE) · Redis · MinIO · LLMs externos
Regra: uma camada só depende da imediatamente inferior (ou de serviços transversais).
```

---

## 10. Checklist de qualidade do diagrama

Antes de salvar um diagrama, verifique **todos** os itens:

- [ ] **Legível em terminal** e em Git diff (ASCII puro, dentro de bloco ```` ``` ````).
- [ ] **Não excede ~100 colunas** (nenhuma linha ultrapassa; conferido de fato).
- [ ] **Box-drawing consistente** (`┌ ┐ └ ┘ ┬ ┴ ├ ┤ ┼ ─ │`); sem mistura com `-`/`|`.
- [ ] **Bordas e lifelines alinhadas** coluna a coluna (nenhuma caixa "torta").
- [ ] **Setas indicam direção** correta (`▶ ◀ ▲ ▼`) e o sentido do fluxo faz sentido.
- [ ] **Participantes/atores/entidades nomeados** — sem caixa anônima ou "?".
- [ ] **Arestas rotuladas** com protocolo/operação/gatilho conforme o tipo.
- [ ] **Nomes casam com o texto**: componentes PascalCase, eventos
      `aios.<tenant>.<dominio>.<entidade>.<acao>`, métricas `aios_<subsistema>_<nome>_<unidade>`,
      erros `AIOS-<DOMINIO>-<NNNN>`, iguais aos das tabelas do documento.
- [ ] **Legenda presente** quando há símbolos não óbvios (síncrono vs assíncrono,
      cardinalidade, `xN`, terminal, self-loop).
- [ ] **Sequência:** mensagens síncronas (`──▶`) e assíncronas (`┄┄▶`) distinguidas;
      timeouts/deadlines anotados `[..]`; caminho de erro mostrado quando relevante.
- [ ] **Estado:** transições numeradas (`T-NN`) com gatilho+guarda, casando com a
      tabela; estados terminais anotados `(terminal)`.
- [ ] **ER:** cardinalidade `1`/`*` em toda relação; ponteiros/FK apontam o módulo dono.
- [ ] **Implantação:** zonas, réplicas (`xN`), replicação e balanceador explícitos.
- [ ] **Sem redefinir contratos**: o diagrama ilustra; contratos (URN, envelope de
      evento, códigos de erro) pertencem à RFC-0001 e ao Glossário `040` — referencie,
      não os redefina no diagrama.

*Fonte de estilo:* `docs/001-Architecture/Architecture.md` e
`docs/006-Kernel/_DESIGN_BRIEF.md`. Ao revisar um diagrama existente, refine-o para
este padrão sem alterar o significado técnico validado.
