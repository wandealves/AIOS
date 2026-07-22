---
Documento: FAQ
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0075, ADR-0078
RFCs relacionados: RFC-0001
Depende de: ./Vision.md, ./_DESIGN_BRIEF.md
---

# 007-Agent-Runtime — FAQ

> Perguntas frequentes, armadilhas comuns e esclarecimentos de escopo. As
> respostas de escopo ("o módulo NÃO faz X") derivam diretamente das
> Não-Responsabilidades do `_DESIGN_BRIEF.md` §1.3 — nenhuma resposta aqui
> introduz uma decisão nova.

## Índice

1. Escopo e responsabilidade
2. Execução e sandbox
3. Cotas e desempenho
4. Eventos e idempotência
5. Segurança
6. Armadilhas comuns de integração

---

## 1. Escopo e Responsabilidade

**P: O Agent Runtime decide *quando* e *onde* um agente executa?**
R: Não. Admissão, *placement* e preempção são responsabilidade do
`../009-Scheduler/`. O Agent Runtime é uma "caixa opaca" de execução vista
de fora — quando recebe `Boot`, ele executa; a decisão de *se* e *quando*
chamar `Boot` é sempre de outro módulo. Ver `_DESIGN_BRIEF.md` N-01.

**P: Quem cria/destrói os processos runtime (gerencia o *pool*)?**
R: O **Runtime Supervisor** (.NET 10, coordenado por `../008-Agent-Lifecycle/`),
um processo separado do control plane. Este módulo especifica apenas o
**processo runtime Python** e o **contrato** (`aios.runtime.v1`) que ele
expõe ao Supervisor. Ver `_DESIGN_BRIEF.md` N-02.

**P: O Agent Runtime escolhe qual modelo de LLM usar?**
R: Não. Toda inferência passa por `../017-Model-Router/`, que decide o
modelo/endpoint (por custo, qualidade, disponibilidade). O runtime apenas
**usa** a decisão recebida e, se configurado, aplica fallback quando o
Router sinaliza indisponibilidade. Ver `_DESIGN_BRIEF.md` N-03, FR-003.

**P: O Agent Runtime registra ou versiona ferramentas?**
R: Não. `../015-Tool-Manager/` é o dono do catálogo, versionamento e
permissões globais de ferramentas. O `McpHost` deste módulo apenas
**consome** o catálogo permitido (recarregado via
`aios.<t>.tool.registry.updated`) e medeia a invocação. Ver
`_DESIGN_BRIEF.md` N-04.

**P: Onde fica armazenada a memória/contexto/plano do agente?**
R: Nunca neste módulo de forma durável. `010-Memory`, `011-Context` e
`012-Planning` são as fontes de verdade; o runtime é produtor/consumidor
efêmero via `MemoryClient`/`ContextClient`/`PlanningClient`. Ver
`_DESIGN_BRIEF.md` N-05, R-05.

**P: O Agent Runtime pode orquestrar vários agentes ao mesmo tempo (um
"maestro")?**
R: Não. Cada processo runtime executa **exatamente um** agente. Fluxos
determinísticos entre múltiplos agentes (sagas, workflows) pertencem a
`../014-Workflow/`. Ver `_DESIGN_BRIEF.md` N-09, `./Vision.md` §7.

---

## 2. Execução e Sandbox

**P: Por que um processo por agente, e não threads compartilhadas?**
R: Isolamento de falhas exige limites de **processo**, não de thread — um
bug de raciocínio, uma ferramenta que trava, ou uma tentativa de *prompt
injection* precisam de uma fronteira que o sistema operacional aplique
(namespaces, `seccomp`, `cgroups`). Threads compartilhadas eliminariam essa
propriedade. Ver ADR-0070 e `./Vision.md` §2.1.

**P: O que acontece se um agente tentar acessar a rede sem permissão?**
R: A conexão é bloqueada pelo `SandboxManager` (*default deny* de egress);
a tentativa é registrada como `AIOS-RUNTIME-0011` (violação de política de
rede) ou, se for uma syscall fora do perfil `seccomp`, como
`AIOS-RUNTIME-0015` (violação de sandbox) seguida de `kill` imediato. Ver
`./Security.md` §6.

**P: Um agente comprometido pode afetar outro agente no mesmo nó físico?**
R: Não, por construção. `cgroups v2` isola CPU/RAM/PIDs por processo;
namespaces isolam visão de processo/rede/FS; a única exclusão distribuída
entre sessões é o *lease* de execução (que impede dupla execução, não
compartilhamento de recurso). Ver `./Scalability.md` §6.

**P: Por que `seccomp-bpf`/`cgroups`/namespaces e não uma microVM (Firecracker/gVisor)?**
R: Overhead de boot de microVM é incompatível com a meta de cold start
≤ 250 ms (NFR-001) para o caso comum. Namespaces+seccomp+cgroups é o padrão
default; microVM permanece candidato para perfis de isolamento máximo por
tenant regulado. Ver ADR-0071 e `./Architecture.md` §11.

---

## 3. Cotas e Desempenho

**P: O que acontece quando um agente esgota o orçamento de tokens/custo/tempo?**
R: O `QuotaGovernor` sinaliza esgotamento; se `checkpoint.enabled=true`, a
sessão é **suspensa com checkpoint** (retomável depois, sem perda de
progresso); caso contrário, é encerrada como `failed`. Nunca continua
consumindo além do limite. Ver `./UseCases.md` UC-007, `AIOS-RUNTIME-0004`.

**P: Por que meu agente foi encerrado com `AIOS-RUNTIME-0012`?**
R: O loop atingiu `loop.max_steps`, `loop.max_recursion_depth` ou
`loop.max_tool_fanout` antes de produzir uma resposta final. Isso é uma
proteção contra loops improdutivos (P2 da Visão Global), não um bug — ajuste
os limites via `./Configuration.md` se o caso de uso legitimamente precisa
de mais passos.

**P: Um passo do loop pode ser "lento" por causa do sandbox?**
R: O overhead de sandbox é limitado a ≤ 5% de CPU / ≤ 8 ms por passo
(NFR-002); latência maior que isso normalmente vem da chamada de
LLM/tool (excluída da meta de `aios_runtime_step_latency_ms`, que mede só o
runtime), não do isolamento em si. Ver `./Benchmark.md` §4.

---

## 4. Eventos e Idempotência

**P: Um evento pode ser publicado duas vezes?**
R: Sim — a entrega é *at-least-once* por design (RFC-0001 §5.2). Todo
consumidor **DEVE** deduplicar por `event.id`. O outbox local garante que
nenhuma transição de estado seja "efetivada" sem o evento correspondente
estar gravado localmente antes da publicação, mas não impede reentregas em
retry de rede. Ver `./Events.md` §4.

**P: Reenviei `SubmitTask` com a mesma `Idempotency-Key` — vai criar uma
nova sessão?**
R: Não, se o payload for idêntico — o runtime retorna o mesmo resultado/
stream da submissão original. Se o payload divergir, retorna
`AIOS-RUNTIME-0007` sem processar a nova submissão. Ver `./API.md` §5.

---

## 5. Segurança

**P: O Agent Runtime decide políticas de acesso (RBAC/ABAC)?**
R: Não. É **PEP** (Policy Enforcement Point), nunca **PDP**. Ele aplica
*capabilities* concedidas no boot e consulta o PDP (`../022-Policy/`, via
Kernel) quando exigido — nunca inventa uma decisão própria. Ver
`_DESIGN_BRIEF.md` N-06, `./Security.md` §1.2.

**P: O que acontece se o PDP estiver indisponível?**
R: Postura *fail-closed*: toda decisão não cacheada é **negada**
(`AIOS-RUNTIME-0003`). Decisões já em cache continuam válidas até
expirarem ou serem invalidadas. Ver `./FailureRecovery.md` §2.

---

## 6. Armadilhas Comuns de Integração

| Armadilha | Por que acontece | Como evitar |
|-------------|----------------------|----------------|
| Tentar chamar um provedor de LLM diretamente de uma ferramenta customizada dentro do sandbox. | Viola a regra de fronteira (toda inferência via `017`). | Registrar a necessidade de modelo como uma capability roteada por `017-Model-Router`, nunca embutir uma chave de provedor no sandbox. |
| Assumir que reduzir `watchdog.stuck_timeout_ms` sempre melhora a recuperação. | Um valor muito baixo gera falsos positivos em passos legitimamente longos (ex.: tool com latência alta legítima). | Calibrar por perfil de tenant/ferramenta; ver risco documentado em `./Architecture.md` §10. |
| Esperar que `Suspend` seja instantâneo. | A geração de checkpoint (serialização + gravação em MinIO) tem custo real; `Suspend` só retorna após o checkpoint estar íntegro (guarda G8). | Não tratar `Suspend` como *fire-and-forget* sem aguardar `SuspendReply`. |
| Assumir que `sandbox.egress_allowlist` vazia significa "sem restrição". | É o oposto — lista vazia = *default deny* total de egress. | Adicionar explicitamente cada `host:porta` necessário por tenant/agente. |
| Ignorar `retriable=false` em `TaskFailed` e reenviar a mesma tarefa indefinidamente. | Erros não-retriable (ex.: `AIOS-RUNTIME-0009`, `AgentSpec` inválido) nunca terão sucesso em retry simples. | Corrigir a causa raiz (ex.: `AgentSpec`) antes de reenviar; consultar `./API.md` §4 para a coluna `retriable`. |

---

## Referências

- Não-responsabilidades completas: `./Vision.md` §3.2, `./_DESIGN_BRIEF.md` §1.3.
- Catálogo de erros: `./API.md` §4.
- Modos de falha detalhados: `./FailureRecovery.md`.
- Exemplos práticos: `./Examples.md`.
