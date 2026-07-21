---
Documento: FAQ
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: 000-Vision, 001-Architecture, 022-Policy, 009-Scheduler, 008-Agent-Lifecycle, 020-Communication, 021-Security, 024-Observability, 025-Audit
---

# 006-Kernel — Perguntas Frequentes (FAQ)

> Este documento esclarece dúvidas recorrentes de escopo, fronteira e uso do
> Kernel. Ele **não redefine** contratos já normatizados em
> `../003-RFC/RFC-0001-Architecture-Baseline.md` nem termos já definidos em
> `../040-Glossary/Glossary.md` — apenas os aplica ao contexto deste módulo.
> Para o comportamento formal e autoritativo, consulte `./_DESIGN_BRIEF.md`.

---

## Índice

1. [Conceitos gerais](#1-conceitos-gerais)
2. [Fronteiras de responsabilidade](#2-fronteiras-de-responsabilidade)
3. [ABI de syscalls e versionamento](#3-abi-de-syscalls-e-versionamento)
4. [ACB e ciclo de vida](#4-acb-e-ciclo-de-vida)
5. [Capability enforcement (PEP) e políticas](#5-capability-enforcement-pep-e-políticas)
6. [Cotas de recurso](#6-cotas-de-recurso)
7. [Idempotência e concorrência](#7-idempotência-e-concorrência)
8. [Eventos e integração assíncrona](#8-eventos-e-integração-assíncrona)
9. [Erros e diagnóstico](#9-erros-e-diagnóstico)
10. [Desempenho e escala](#10-desempenho-e-escala)
11. [Segurança e multi-tenant](#11-segurança-e-multi-tenant)
12. [Armadilhas comuns (pitfalls)](#12-armadilhas-comuns-pitfalls)

---

## 1. Conceitos gerais

### 1.1 O que é, em uma frase, o Kernel do AIOS?

É o **núcleo cognitivo**: a fronteira de confiança única pela qual todo
agente solicita qualquer recurso do AIOS, expondo uma ABI de 11 syscalls,
mantendo o ACB de cada agente, aplicando capability enforcement (PEP) e
cotas, e coordenando o ciclo de vida em conjunto com Scheduler e Lifecycle.
Ver `./Vision.md` §1.

### 1.2 Por que "Kernel" e não apenas "API Gateway" ou "Orquestrador"?

Porque ele faz mais do que rotear requisições: mantém **estado autoritativo**
de cada agente (o ACB, análogo ao PCB de um SO clássico), impõe uma **máquina
de estados canônica**, e é o ponto estrutural de **default deny**. Um API
Gateway (YARP, no AIOS) resolve AuthN, roteamento HTTP e rate-limit de borda;
o Kernel resolve semântica de domínio — é uma camada acima, no plano de
controle. Ver `_DESIGN_BRIEF.md` §1.1 e o diagrama de componentes em
`./Architecture.md`.

### 1.3 O Kernel executa o agente?

**Não.** O Kernel nunca executa o loop de raciocínio nem hospeda o sandbox do
agente — isso é `007-Agent-Runtime`. O Kernel apenas mantém e transiciona o
**estado de controle** (ACB) e broqueia o acesso a recursos. Ver
Não-Responsabilidade NR-01 no `_DESIGN_BRIEF.md` §1.3.

### 1.4 Quais são exatamente as 11 syscalls?

`spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`,
`invoke_tool`, `route_model`, `get_quota`, `checkpoint`. Contratos completos
(REST/gRPC, idempotência, exemplos) em `./API.md`; mapeamento verbo→endpoint
no `_DESIGN_BRIEF.md` §5.1.

---

## 2. Fronteiras de responsabilidade

### 2.1 O Kernel decide *onde* um agente executa?

**Não.** O Kernel *pede* admissão/placement ao `009-Scheduler`; quem decide
slot, preempção e prioridade de fila é o Scheduler. O Kernel apenas reage à
decisão (transições T-02/T-03 da FSM). Ver NR-02 no `_DESIGN_BRIEF.md` §1.3
e `./StateMachine.md`.

### 2.2 O Kernel materializa o processo do agente?

**Não.** Materialização (start do runtime), hibernação e snapshot/restore de
estado físico são responsabilidade do `008-Agent-Lifecycle` (e do Runtime
Supervisor). O Kernel apenas solicita essas ações via `LifecycleClient` e
reage às confirmações (transições T-04/T-05/T-11/T-12). Ver NR-03.

### 2.3 O Kernel decide se uma ação é permitida?

**Não, ele *aplica*.** O Kernel é **PEP** (Policy Enforcement Point): monta o
contexto de decisão e consulta o **PDP** do `022-Policy`, que decide segundo
regras RBAC/ABAC. O Kernel nunca hospeda lógica de política — apenas cacheia,
por curto prazo, a última decisão recebida (`capability_set` no ACB é cache,
não autoridade). Ver NR-04 e `./Security.md` (quando disponível) /
`_DESIGN_BRIEF.md` §12.1.

### 2.4 O Kernel armazena memórias, planos ou resultados de ferramentas?

**Não.** Ele encaminha `remember/recall/plan/invoke_tool/route_model` aos
módulos donos (`010`, `011`, `012`, `015`, `017` respectivamente), aplicando
antes capability + cota. O Kernel guarda apenas **ponteiros** (`memory_ptr`,
`context_ptr`) no ACB, nunca o conteúdo em si. Ver NR-05.

### 2.5 Quem persiste a trilha de auditoria — o Kernel ou o Audit?

O Kernel **emite** o registro (evento/telemetria de decisão); quem
**persiste de forma imutável e durável** é o `025-Audit`. Essa separação evita
que o Kernel acumule responsabilidade de armazenamento de longo prazo que não
é seu domínio. Ver NR-06.

### 2.6 O Kernel valida tokens JWT/OIDC?

**Não.** AuthN (validação de token OAuth2/OIDC) acontece no **Gateway
(YARP)**, consultando o `021-Security`. O Kernel **consome claims já
validadas** (tenant, subject, escopos) — ele nunca decodifica ou valida
assinatura de token por conta própria. Ver NR-07.

### 2.7 Quem define quanto custo um tenant pode gastar?

O **orçamento** (o valor limite) é definido pelo `026-Cost-Optimizer`; o
**Kernel aplica** esse limite como uma dimensão de cota (`cost_usd_limit` na
entidade `Quota`). O Kernel não decide política de FinOps, apenas garante que
o limite vigente seja respeitado em tempo real. Ver NR-08.

---

## 3. ABI de syscalls e versionamento

### 3.1 Por que a ABI tem exatamente 11 verbos e não mais?

Porque cada verbo mapeia 1:1 a uma operação de controle que precisa ser
autorizada, cotada e auditada de forma independente. Agrupar verbos (por
exemplo, um único verbo genérico `execute`) tornaria a política de
autorização e a contabilidade de cota ambíguas — a granularidade dos 11
verbos é uma decisão de design proposta em `ADR-0060`.

### 3.2 Posso adicionar um 12º verbo sem passar por RFC?

**Não deveria.** A ABI é um contrato estável consumido por
`007-Agent-Runtime` e pelo SDK (`031-SDK`). Novos verbos alteram a superfície
pública e DEVEM ser propostos via nova RFC (extensão da `RFC-0006`) e
registrados no `ADR.md` do módulo, seguindo o processo de versionamento de
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.7/§9.

### 3.3 REST e gRPC expõem a mesma semântica?

Sim. Ambos os protocolos expõem os mesmos 11 verbos com o mesmo contrato
semântico (mesmos campos, mesmos erros, mesma idempotência) — REST é a
superfície externa (via Gateway), gRPC é a superfície interna (chamada por
outros módulos e pelo Agent Runtime), conforme `_DESIGN_BRIEF.md` §5.
Detalhes em `./API.md`.

### 3.4 Como funciona o versionamento de API do Kernel?

Segue `RFC-0001` §5.7: caminho `/v1/kernel/...` para REST, pacote
`aios.kernel.v1` para gRPC, header `X-AIOS-Api-Version`. Mudanças
incompatíveis incrementam a versão major e coexistem por ≥ 2 majors
(V-R9 da Visão Global).

---

## 4. ACB e ciclo de vida

### 4.1 O que exatamente é um ACB?

É a estrutura de controle autoritativa de um agente gerenciado: identidade
(URN), estado de FSM, prioridade, referência de cota, ponteiros de
memória/contexto, referência de política, capability cache, referência de
runtime e de checkpoint. Tabela completa em `_DESIGN_BRIEF.md` §3.1 e
contrato formal em `./ClassDiagrams.md`.

### 4.2 Quantos estados tem a máquina de estados do ACB?

Oito: `Pending`, `Admitted`, `Running`, `Suspended`, `Hibernated`,
`Terminating`, `Terminated` (terminal), `Failed` (terminal). Diagrama
completo, guardas e invariantes em `./StateMachine.md`.

### 4.3 Qual a diferença entre `Suspended` e `Hibernated`?

`Suspended` é um estado **quente**: o agente está pausado, mas seu estado
ainda reside em memória rápida (Redis) e pode retomar (`resume`) com baixa
latência. `Hibernated` é um estado **frio**: o estado foi movido para
armazenamento durável (MinIO/PostgreSQL) e a RAM foi liberada — retomar exige
materialização sob demanda (transição T-09), com meta de latência p99 < 250 ms.
A transição `Suspended → Hibernated` (T-08) é automática, disparada por
ociosidade além de `kernel.hibernation.idle_ttl_ms`.

### 4.4 Um agente pode pular direto de `Pending` para `Running`?

**Não.** A FSM exige passar por `Admitted` (o Scheduler precisa confirmar
slot antes de qualquer materialização). Pular esse passo violaria a
invariante I1 (`runtime_ref` só é não-nulo em `Running`/`Suspended`) e a
separação de responsabilidade entre admissão (Scheduler) e materialização
(Lifecycle). Ver `_DESIGN_BRIEF.md` §4.2, transições T-02/T-04.

### 4.5 O que acontece se o boot do runtime falhar após admissão?

O ACB transiciona `Admitted → Failed` (T-05), a reserva de slot no Scheduler
é revertida e a cota reservada é liberada — tudo como parte da saga de
`spawn` com compensação (ADR-0064). Nenhum recurso fica "preso" em um agente
que nunca chegou a `Running`.

### 4.6 `checkpoint` muda o estado do ACB?

**Não diretamente.** `checkpoint` (T-13) é uma transição de auto-loop: o
agente permanece em `Running` ou `Suspended`, mas o `checkpoint_ref` é
atualizado com o novo snapshot durável. Ele existe para permitir `resume`
consistente após hibernação ou falha.

### 4.7 O que é o `parent_urn` no ACB?

É o URN do agente que executou `spawn` para criar este agente — permite
reconstruir a **árvore de agentes** (ex.: um agente orquestrador que gera
sub-agentes especializados). Fan-out de `spawn` é limitado por política para
evitar o problema O(N²) de coordenação citado na Visão Global (RSK-03).

---

## 5. Capability enforcement (PEP) e políticas

### 5.1 O que significa "default deny" na prática do Kernel?

Significa que, na ausência de uma decisão explícita `allow` do PDP para a
tupla (sujeito, ação, recurso, ambiente) de uma syscall privilegiada, o
Kernel **nega** a execução — inclusive quando o PDP está indisponível
(`kernel.pep.fail_mode=closed`, o padrão). Não existe caminho de "permitir
por omissão".

### 5.2 O `capability_set` armazenado no ACB é a fonte de verdade?

**Não.** É apenas um **cache** do último *grant* concedido pelo PDP, usado
para otimizações de leitura (por exemplo, checagens de UI). A autoridade
real é sempre a consulta ao PDP em tempo de syscall, via `PolicyClient`.

### 5.3 O que acontece se o PDP estiver fora do ar?

Depende de `kernel.pep.fail_mode`. Com o padrão `closed`, toda syscall
privilegiada é negada com `AIOS-CAP-0003` (503, retriable) até o PDP voltar
ou o circuit breaker entrar em meia-abertura. Com `open` (configuração
explícita, não recomendada em produção), syscalls seriam permitidas sem
decisão fresca — uso restrito a ambientes de teste. Ver
`_DESIGN_BRIEF.md` §9.

### 5.4 Por quanto tempo uma decisão do PDP fica em cache?

Por `kernel.pep.decision_cache_ttl_ms` (padrão 3000 ms). É um TTL
deliberadamente curto: equilibra latência (evita ida ao PDP em toda syscall)
com atualidade (uma revogação de capability se propaga em poucos segundos).

### 5.5 Uma mudança de política do 022-Policy é aplicada instantaneamente?

O Kernel assina o evento `aios.<tenant>.policy.bundle.updated` e invalida o
cache do `CapabilityEnforcer` ao recebê-lo — a propagação é, portanto,
assíncrona e da ordem de latência do barramento (tipicamente sub-segundo),
não instantânea de forma síncrona.

---

## 6. Cotas de recurso

### 6.1 Cotas são aplicadas por agente, por tenant, ou ambos?

Ambos. A entidade `Quota` tem um campo `scope` (`tenant` ou `agent`); um ACB
referencia uma cota de escopo `agent` via `quota_ref`, e existe também uma
cota agregada de escopo `tenant`. As duas dimensões são verificadas.

### 6.2 O que diferencia `enforcement: hard` de `enforcement: soft`?

`hard` **rejeita** a operação que excederia o limite (retorna
`AIOS-QUOTA-000x`, 429). `soft` **permite** a operação mas marca o evento e
emite `agent.quota.warning` — útil para observação antes de endurecer um
limite em produção.

### 6.3 Onde vivem os contadores de consumo de cota — Redis ou PostgreSQL?

Os **contadores de alta frequência** (o quanto já foi consumido na janela
corrente) vivem em **Redis**, como token-buckets atualizados por script Lua
atômico. O **PostgreSQL** guarda os **limites** (a definição da cota), não os
contadores quentes — ele é reconciliado periodicamente a partir do Redis.
Ver `_DESIGN_BRIEF.md` §3.2 e §10.

### 6.4 É possível haver overshoot (consumo acima do limite)?

Sob concorrência extrema, sim, mas a meta é **≤ 1%** (NFR-009), garantida
pela atomicidade do script Lua no Redis. Overshoot maior indica defeito e
deve ser tratado como incidente (`Testing.md` cobre teste de carga
concorrente para essa meta).

### 6.5 `get_quota` retorna dado em tempo real ou pode estar desatualizado?

Retorna o saldo dos contadores quentes do Redis, coerente ±1 janela de
reconciliação com o PostgreSQL (FR-010). Não há atraso perceptível para o
chamador — é uma leitura direta do contador atômico.

---

## 7. Idempotência e concorrência

### 7.1 Toda syscall do Kernel exige `Idempotency-Key`?

Toda **mutação** exige (`spawn`, `kill`, `suspend`, `resume`, `remember`,
`plan`, `invoke_tool`, `route_model`, `checkpoint`). Leituras puras
(`get_quota`, `GetAcb`, e a variante de leitura de `recall`) não exigem chave
de idempotência porque não têm efeito colateral a deduplicar — mas ainda
seguem os cabeçalhos de correlação obrigatórios da RFC-0001 §5.6.

### 7.2 O que acontece se eu reenviar a mesma `Idempotency-Key` com um payload diferente?

O Kernel retorna `AIOS-SYSCALL-0002` (409) — reuso de chave com payload
divergente é tratado como erro de uso da API, não como nova operação. A
chave deve ser única por *intenção* de operação, não reaproveitada entre
operações logicamente distintas.

### 7.3 Por quanto tempo o resultado idempotente fica retido?

No mínimo 24h (piso normativo da RFC-0001 §5.5); o Kernel usa
`kernel.idempotency.retention_h`, com padrão de 48h, configurável até 720h.

### 7.4 Como o Kernel resolve concorrência quando duas requisições tentam transicionar o mesmo ACB ao mesmo tempo?

Via **concorrência otimista (OCC)** sobre o campo `version` do ACB: a
segunda escrita, ao detectar `version` divergente, falha com
`AIOS-KERNEL-0009` (409, retriable) e o cliente (ou o próprio Kernel,
internamente, até `kernel.acb.optimistic_retry_max` vezes) tenta novamente
lendo o estado atualizado. Não há locks pessimistas no caminho quente — ver
Princípio de Design 6 em `./Vision.md` e `_DESIGN_BRIEF.md` §10.

### 7.5 Existe algum caso em que o Kernel usa lock distribuído?

Sim, excepcionalmente — quando uma transição de FSM realmente concorrente
não pode esperar por retry de OCC (por exemplo, uma corrida de `kill`
simultâneo). Nesses casos, um lock Redis de **TTL curto** é usado, mas OCC
continua sendo a estratégia preferida (`_DESIGN_BRIEF.md` §10).

---

## 8. Eventos e integração assíncrona

### 8.1 Que garantia de entrega o Kernel oferece para seus eventos?

**At-least-once** via NATS/JetStream, com o padrão **Outbox transacional**
garantindo que nenhum evento aceito seja perdido mesmo em crash entre commit
e publicação. Consumidores DEVEM deduplicar por `event.id`, conforme
RFC-0001 §5.2/§5.5.

### 8.2 Os eventos do Kernel são ordenados?

Sim, por stream — `KERNEL_LIFECYCLE`, `KERNEL_QUOTA` e `KERNEL_AUDIT`
preservam ordem de publicação dentro do mesmo stream. Não há garantia de
ordem *entre* streams diferentes.

### 8.3 Como um módulo externo sabe que um agente foi hibernado?

Assinando `aios.<tenant>.agent.lifecycle.hibernated`. Ver catálogo completo
em `./Events.md` e `_DESIGN_BRIEF.md` §6.1.

### 8.4 O Kernel consome eventos ou só produz?

Ambos. Ele **produz** eventos de ciclo de vida/cota/decisão e **consome**
eventos de `009-Scheduler` (decisão de admissão/preempção), `008-Agent-Lifecycle`
(runtime iniciado/parado), `022-Policy` (bundle de política atualizado) e
`026-Cost-Optimizer` (orçamento atualizado). Ver `_DESIGN_BRIEF.md` §6.2.

---

## 9. Erros e diagnóstico

### 9.1 Que domínios de código de erro pertencem ao Kernel?

`KERNEL` (0001–0099), `CAP` (0001–0099), `SYSCALL` (0001–0099), e uma
subfaixa de `QUOTA` (`AIOS-QUOTA-0001`..`AIOS-QUOTA-0020`, compartilhada com
`026-Cost-Optimizer`). Catálogo completo em `./API.md` e
`_DESIGN_BRIEF.md` §5.2.

### 9.2 Recebi `AIOS-KERNEL-0002`. O que significa?

Uma transição de estado inválida foi solicitada — por exemplo, tentar
`resume` em um ACB que está em `Terminated` (estado terminal, sem
transições de saída). Consulte a FSM em `./StateMachine.md` para as
transições permitidas a partir do estado atual do ACB.

### 9.3 Recebi `AIOS-CAP-0001`. Como eu debugo?

Significa que o PDP negou a capability necessária para a syscall. Não é um
defeito do Kernel — é o comportamento correto de *default deny*. Debug:
verifique o `policy_ref` vigente do ACB, os *bindings* de RBAC/ABAC do
agente/tenant no `022-Policy`, e consulte o evento correlato
`aios.<tenant>.agent.syscall.denied` (contém `trace_id` para correlação).

### 9.4 Um erro `retriable: true` significa que devo retentar imediatamente?

Não imediatamente — use backoff exponencial com jitter, e para 429
(`AIOS-QUOTA-*`) respeite o campo `retryAfterMs`/header `Retry-After` do
envelope de erro (RFC-0001 §5.4, Apêndice B).

---

## 10. Desempenho e escala

### 10.1 Qual a meta de latência para `spawn`?

p99 ≤ 250 ms no caminho quente (cold agent → running), conforme NFR-001.
Para syscalls de controle simples (`suspend`/`resume`/`kill`/`get_quota`), a
meta é p99 ≤ 20 ms (NFR-002).

### 10.2 O Kernel escala horizontalmente sem coordenação?

Sim — o serviço Kernel é **stateless**: todo estado vive em PostgreSQL/Redis
externalizados. Réplicas escalam por CPU/fila sem afinidade obrigatória;
sharding (`hash(tenant,agent) mod N`) é uma otimização de localidade de
cache, não um requisito de correção. Ver `_DESIGN_BRIEF.md` §10 e
`./Scalability.md` (quando disponível).

### 10.3 Como o Kernel evita que um tenant afete o desempenho de outro?

Via limites de `syscalls_per_sec` por agente/tenant, cotas `hard`,
admissão controlada pelo Scheduler, e rejeição precoce (`429`) preferida a
degradação silenciosa de todos — a estratégia clássica de contenção de
*noisy neighbor* (Princípio 3 da Visão Global).

### 10.4 Quantos ACBs o Kernel suporta?

Meta de ≥ 10⁶ ACBs (majoritariamente `Hibernated`), com crescimento
sub-linear de custo (NFR-006), habilitado pela combinação hot/cold + sharding
+ cache de decisão de política.

---

## 11. Segurança e multi-tenant

### 11.1 O que acontece se um agente tentar acessar recursos de outro tenant?

O Kernel rejeita com `AIOS-CAP-0002` (403) — qualquer `tenant` no payload,
URN ou claim que divirja do contexto autenticado é recusado antes de
qualquer outra lógica. Ver FR-011 e RFC-0001 §6.

### 11.2 A comunicação entre o Kernel e seus clientes internos é criptografada?

Sim, via **mTLS** entre serviços internos (RFC-0001 §6, delegado ao
`021-Security`); o Kernel não implementa sua própria camada de transporte
seguro — reutiliza a malha padronizada da plataforma.

### 11.3 O Kernel armazena PII?

Não deveria — identificadores são URNs opacos e o Kernel aplica minimização
de dados nos payloads de eventos/logs (RFC-0001 §7, `_DESIGN_BRIEF.md`
§12.3). Dados de conteúdo (memória, resultados) pertencem a `010-Memory` e
carregam sua própria base legal.

### 11.4 Como funciona o "direito ao esquecimento" (LGPD/GDPR) para um agente?

Via `kill` seguido de expurgo rastreável do ACB, retendo apenas os metadados
de auditoria exigidos por lei — a operação é coordenada com `025-Audit`. Ver
`_DESIGN_BRIEF.md` §12.3.

---

## 12. Armadilhas comuns (pitfalls)

| Armadilha | Por que é um erro | Como evitar |
|-----------|--------------------|-------------|
| Tratar `capability_set` do ACB como fonte de verdade de autorização. | É apenas cache do último *grant*; pode estar desatualizado até `decision_cache_ttl_ms`. | Sempre trate `AIOS-CAP-0001` em tempo de syscall como a decisão real; não pré-filtre no cliente com base no cache. |
| Reaproveitar a mesma `Idempotency-Key` para operações logicamente diferentes. | Gera `AIOS-SYSCALL-0002` (payload divergente) ou, pior, mascara uma operação nova como repetição de outra. | Gere uma chave nova (ULID/UUID) por *intenção* de mutação, nunca por sessão inteira. |
| Assumir que `suspend` é instantâneo e sempre permitido. | A transição exige que o agente esteja em estado suspensível (sem seção crítica não-idempotente em curso) — ver guarda de T-06. | Verificar o estado do ACB antes de assumir sucesso; tratar `AIOS-KERNEL-0002` como sinal de "tente depois". |
| Chamar `remember`/`recall`/`plan`/`invoke_tool`/`route_model` esperando que o Kernel implemente a lógica de negócio. | O Kernel é broker; a lógica pertence a `010/011/012/015/017`. | Consultar a documentação do módulo dono do recurso para semântica de domínio; o Kernel documenta apenas o envelope de brokering. |
| Ignorar o campo `retryAfterMs` em respostas 429 de cota. | Retentar imediatamente agrava contenção e pode acionar limites de tenant inteiro. | Implementar backoff respeitando `Retry-After`/`retryAfterMs`. |
| Presumir que `Hibernated → Running` tem a mesma latência que `Suspended → Running`. | Hibernação envolve materialização a frio (rehidratar de armazenamento durável); é uma ordem de grandeza mais lenta, ainda que dentro da meta de 250 ms p99. | Não usar hibernação para agentes que precisam retomar em latência de milissegundos consistente — nesse caso, manter em `Suspended`. |
| Configurar `kernel.pep.fail_mode=open` em produção "para não travar o sistema" durante uma indisponibilidade do PDP. | Viola *default deny* estrutural e abre uma janela de execução sem autorização — contraria Princípio 2 da Visão Global e o requisito NFR-010. | Manter `closed` em produção; investir em disponibilidade do PDP (cache, HA) em vez de contornar o enforcement. |
| Assumir que eventos do Kernel chegam exatamente uma vez. | A garantia é *at-least-once*; duplicatas são esperadas sob retry/falha de rede. | Consumidores DEVEM deduplicar por `event.id`, nunca assumir entrega única. |

---

## Referências

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md`
- Visão do módulo: `./Vision.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Exemplos práticos: `./Examples.md`

*Fim de `FAQ.md`.*
