---
Documento: StateMachine
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0042, ADR-0043
RFCs relacionados: RFC-0001 (§5.7), RFC-0040
Depende de: ./Database.md, ./API.md, ./Events.md
---

# 004-API — Máquina de Estados

> Reproduz e detalha `./_DESIGN_BRIEF.md` §4 (fonte única de verdade). O
> Gateway em si é majoritariamente um **pipeline de requisição sem estado**
> (Recebida → Autenticada → Autorizada → Limitada → Validada → Roteada →
> Traduzida) — cada etapa é um filtro YARP idempotente e sem memória entre
> requisições (ver `./SequenceDiagrams.md` UC-001). A **única entidade com
> máquina de estados canônica persistente** neste módulo é `ApiVersion`.

## 1. Pipeline de requisição (não é uma FSM de entidade, mas é normativo)

```
 Recebida ──▶ Autenticada ──▶ Autorizada ──▶ Limitada ──▶ Validada ──▶ Roteada ──▶ Traduzida
  (EdgeRouter)  (AuthNFilter)  (RouteAuth-   (RateLimiter) (SchemaVal-  (EdgeRouter   (ErrorTranslator,
                                orizer/PEP)                idator)      + CB Mgr)      sempre no erro)
```

Cada etapa PODE terminar o pipeline com um erro terminal (ver
`./API.md` §"Catálogo de erros"), sempre normalizado pelo `ErrorTranslator`
antes de retornar ao cliente. Nenhuma etapa persiste estado além do que já é
externalizado em Redis/PostgreSQL pelos componentes correspondentes
(`RateLimiter`, `IdempotencyRelay`).

## 2. Máquina de estados canônica — `ApiVersion`

### 2.1 Estados (`ApiVersionState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Draft` | Versão major em desenvolvimento; ainda não roteável para clientes externos. | não |
| `Published` | Versão ativa, roteável, coberta por SLO; contrato publicado e testado. | não |
| `Deprecated` | Versão ainda roteável, mas sinalizada como obsoleta (`Deprecation`/`Sunset` headers); nova major já `Published`. | não |
| `Retired` | Versão desativada; requisições retornam `410 Gone` (`AIOS-API-0011`). | **sim** |

### 2.2 Diagrama de estados (ASCII)

```
                registrar (T-01)
      ∅ ──────────────────────────▶ ┌─────────┐
                                     │  Draft  │
                                     └────┬────┘
                 publish (T-02)           │  cancel (T-03, sem rotas ativas)
              ┌──────────────────────────┤────────────────────┐
              ▼                          │                    ▼
        ┌───────────┐                    │              ┌───────────┐
        │ Published │◀── undeprecate ────┼──────────────│  Retired  │ (terminal)
        └─────┬─────┘   (T-06, exc.)     │              └───────────┘
              │ nova major M+1 Published (T-04)                ▲
              ▼                                                 │ retire (T-05):
        ┌───────────┐   janela+tráfego OK + ≥2 majors novas ────┘  janela cumprida ∧
        │Deprecated │──────────────────────────────────────────────  tráfego residual baixo
        └───────────┘
```

### 2.3 Tabela de transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação de entrada |
|---|-----------|---------|-------------------------------|-------------------|
| T-01 | ∅ → `Draft` | Registro de nova major (admin/PR, UC-009 correlato) | Módulo dono aprovado (ADR/RFC do módulo referenciado). | Cria linha `api.version`; `state=Draft`. |
| T-02 | `Draft` → `Published` | `publish` (UC-010) | Contrato OpenAPI/proto publicado ∧ contract tests passam ∧ ≥ 1 `RouteDefinition` ativa referenciando a major. | `published_at=now()`; emite `aios._platform.api.version.published`. |
| T-03 | `Draft` → `Retired` | `cancel` | Nenhuma rota ativa referencia a major (versão abandonada antes do lançamento). | `retired_at=now()`. |
| T-04 | `Published` → `Deprecated` | Nova major `M+1` atinge `Published` (UC-011) | Versão `M` NÃO é a única `Published`/`Deprecated` do módulo (invariante I1). | `deprecated_at=now()`; emite `aios._platform.api.version.deprecated`. |
| T-05 | `Deprecated` → `Retired` | `retire` (UC-012) | `now - deprecated_at ≥ deprecation_window_days` ∧ tráfego observado abaixo de `api.version.retirement_traffic_threshold` ∧ ≥ 2 majors mais novas em `Published`/`Deprecated`. | `retired_at=now()`; emite `aios._platform.api.version.retired`. |
| T-06 | `Deprecated` → `Published` | `undeprecate` (excepcional, administrativo) | Reversão antes de `Retired`, com justificativa auditada. | `deprecated_at=NULL`; emite evento de reversão (auditado). |

### 2.4 Invariantes

- **I1 — Coexistência mínima.** Uma versão major **NÃO DEVE** transicionar
  para `Retired` se for a única versão em `Published`/`Deprecated` do módulo
  — garante a coexistência mínima de `api.version.coexistence_majors_min`
  (default 2) majors exigida pela RFC-0001 §5.7.
- **I2 — `Retired` é terminal e bloqueante.** Toda requisição para uma major
  em `Retired` **DEVE** retornar `410 Gone` (`AIOS-API-0011`), sem tentar
  alcançar o upstream.
- **I3 — Aviso obrigatório em `Deprecated`.** Toda requisição para uma major
  em `Deprecated` **DEVE** incluir os cabeçalhos `Deprecation: true` e
  `Sunset: <data estimada>`.
- **I4 — Transição atômica e auditável.** Toda transição **DEVE** usar OCC
  (campo `version`, ver `./Database.md`) e **DEVE** emitir exatamente um
  evento `aios._platform.api.version.<acao>` (ver `./Events.md`).

### 2.5 Estados terminais e retenção

`Retired` é o único estado terminal. Não há transição de saída. Registros em
`Retired` **DEVEM** ser retidos indefinidamente no `ContractRegistry` para
fins de auditoria histórica (uma major retirada não é apagada, apenas deixa
de ser roteável) — ver `./Database.md` §"Retenção".

### 2.6 Concorrência

Toda transição é protegida por **Optimistic Concurrency Control (OCC)** via
o campo `version` de `api.version` (`./Database.md` §3.2). Duas requisições
concorrentes de transição sobre a mesma `(module_owner, major)` DEVEM
resultar em uma vencedora e uma rejeitada com erro de conflito de versão,
nunca em estado inconsistente. Mutações de registro têm baixa frequência
(administrativas), portanto não há necessidade de locks distribuídos
adicionais além do OCC (ver `./_DESIGN_BRIEF.md` §10).

## 3. Relação com o restante do módulo

- `VersionNegotiator` (`./Architecture.md` §5) é o componente de *runtime*
  que **lê** o estado corrente de `ApiVersion` para decidir se uma
  requisição é roteável, avisada ou rejeitada — ele nunca transiciona o
  estado diretamente; transições ocorrem apenas via a superfície de
  administração (`./API.md` §"RegistryService").
- Toda transição é correlacionada a um evento em `./Events.md` §6.1 e a um
  caso de uso em `./UseCases.md` (UC-009..UC-012).

*Fim de `StateMachine.md`.*
