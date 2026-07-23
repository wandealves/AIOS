---
Documento: FAQ
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0200, ADR-0201, ADR-0202, ADR-0204, ADR-0205, ADR-0206, ADR-0207, ADR-0208
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: Vision.md, Events.md, API.md, Examples.md
---

# 020-Communication — Perguntas Frequentes

---

## Escopo

**O serviço 020 fica entre minha aplicação e o NATS?**
Não. O caminho de mensagem vai **direto** ao broker. O `aios-comm-svc` registra
subjects, declara streams, autoriza sessões A2A e opera a DLQ — mas não é proxy de
tráfego (`./Architecture.md` §1). Se ele cair, você para de declarar coisas; não para de
trocar mensagens.

**Quem define o que significa um evento?**
O módulo produtor. O 020 governa a **forma** (subject, envelope, schema, produtor
autorizado) e as **garantias** (entrega, ordem, isolamento). O significado de
`agent.lifecycle.spawned` pertence ao `../006-Kernel/`.

**Por que não posso publicar em qualquer subject?**
Porque um barramento sem catálogo se torna, em poucos meses, um conjunto de eventos
órfãos que ninguém sabe quem emite nem quem consome. Registro obrigatório
(`AIOS-BUS-0004`) mantém o catálogo verdadeiro — e mantém o `producer_module`
verificável, o que impede forjar evento alheio.

**O barramento guarda meus dados?**
Não. Streams têm `max_age` explícito (default 7 dias) e o módulo **não é fonte da
verdade**. Estado autoritativo vive em `../005-Database/`; trilha imutável, em
`../025-Audit/`.

---

## NATS e escolhas de tecnologia

**Por que NATS e não Kafka?**
O tráfego do AIOS é dominado por mensagens **pequenas, de controle e sensíveis a
latência** — o perfil oposto ao que justifica Kafka. NATS entrega p99 de ~2 ms no core,
tem request/reply nativo, subjects hierárquicos com curinga e contas multi-tenant de
primeira classe, com operação leve (ADR-0004). O preço é um ecossistema menor de
conectores e menos tradição em retenção de longuíssimo prazo — que aqui não é
necessária, porque retenção longa é de `025`/`005`.

**Preciso de JetStream para tudo?**
Não, e usá-lo indiscriminadamente custa caro. Telemetria e sinais efêmeros devem usar
**core pub/sub** (p99 ≤ 2 ms, sem ack). Streams duráveis (p99 ≤ 15 ms com ack R=3) são
para o que **não pode ser perdido**: ciclo de vida, decisões, auditoria. A diferença de
latência entre C-1 e C-2 em `./Benchmark.md` é exatamente o preço da durabilidade.

**Por que 3 nós e não 4?**
Quatro nós não aumentam a tolerância a falhas em relação a três (ambos toleram 1 falha
com quórum) e dobram o custo de replicação. Cluster de quórum usa número **ímpar**.

---

## Namespace e subjects

**Por que não posso ter um subject por agente?**
Com 10⁶ agentes seriam 10⁶ subjects: roteamento degradado, observabilidade ilegível e
catálogo inútil. O subject identifica um **tipo** de evento; a instância vai no campo
`subject` do envelope (`./Examples.md` §4).

**Como faço para um agente receber mensagens direcionadas a ele?**
Duas opções, ambas governadas: canal de **Agent Group** (para difusão) ou **sessão
A2A** (para conversa ponto a ponto). Criar um subject por agente não é uma delas.

**Posso assinar `aios.>`?**
Pode — e receberá apenas o tráfego da **sua conta**. Um curinga amplo não atravessa a
fronteira de tenant, porque a outra conta é invisível ao roteamento (`./Security.md` §3).

**Quero mudar o payload de um evento existente. Como?**
Evolução **aditiva**: adicione campos e registre um novo `dataschema` versionado.
Mudança incompatível exige subject novo e depreciação do antigo, com coexistência de
≥ 2 majors (RFC-0001 §5.7).

---

## Entrega e confiabilidade

**"At-least-once" significa que vou processar duplicado?**
Significa que **pode** haver reentrega. O efeito único vem da combinação: Outbox no
produtor + dedupe por `event.id` na janela do stream + **dedupe no consumidor**. A
terceira parte é sua obrigação (RFC-0001 §5.5) — sem ela, a garantia não se completa.

**Minha mensagem sumiu. Onde procuro?**
Em ordem: (1) o Outbox do produtor — se a publicação foi recusada, ela está lá com
`published = false`; (2) a DLQ (`aios comm dlq list`) — se o consumidor falhou 5 vezes;
(3) o stream, se ainda estiver dentro do `max_age`. O que o barramento **não** faz é
descartar em silêncio.

**Por que minha mensagem foi entregue duas vezes?**
Provavelmente o consumidor não confirmou dentro de `ack_wait` (30 s) e houve reentrega.
Se o processamento é longo, envie `in-progress` para estender o prazo; se falhou, use
`nak` com atraso. E deduplique por `event.id`.

**Posso desligar a DLQ?**
Não deveria. Sem DLQ, uma mensagem envenenada é reentregue para sempre (travando o
consumidor) ou descartada em silêncio. A DLQ transforma isso em uma falha **visível e
diagnosticável**.

**Por que o replay não é automático?**
Porque replay antes da correção reproduz exatamente o mesmo defeito e reabre o mesmo
incidente — agora com o dobro de mensagens. `bus.dlq.auto_replay = false` por default
(ADR-0205) torna a etapa de correção inevitável.

---

## Desempenho e escala

**Meu publish está lento. É o barramento?**
Verifique se você está usando stream durável onde bastaria core pub/sub: o ack com R=3
custa ~13 ms a mais (`./Benchmark.md` §4). Verifique também o tamanho do payload — a
curva de T-3 mostra a degradação a partir de 4 KiB.

**Recebi `AIOS-BUS-0006` (payload grande). E se eu realmente precisar enviar 10 MiB?**
Envie a **referência**: grave o objeto em MinIO e publique o ponteiro (ADR-0208). Uma
mensagem de 10 MiB não é apenas lenta para você — ela consome a capacidade de todos os
fluxos que compartilham o cluster.

**Recebi `AIOS-BUS-0007` (cota). Aumento a cota?**
Primeiro entenda se é abuso ou crescimento legítimo. Se for legítimo, o ajuste vem do
`../026-Cost-Optimizer/` via `cost.budget.updated` — não por edição manual permanente.
E não repita imediatamente: respeite o `Retry-After`.

**Meu consumidor está com lag enorme. Aumento `max_ack_pending`?**
Quase nunca. `max_ack_pending` limita o **em voo**, não a taxa: aumentá-lo faz o
consumidor lento acumular mais trabalho pendente e ainda arrasta os demais. A ação certa
é escalar instâncias no mesmo `durable` (`./Examples.md` §8) ou corrigir o
processamento.

**Por que existe limite de 64 assinaturas por agente?**
Para tornar a malha ponto a ponto estruturalmente impossível. Com 10⁶ agentes, permitir
assinaturas irrestritas levaria à topologia quadrática que colapsa o sistema
(`./Scalability.md` §1).

---

## Multi-tenancy e segurança

**Como sei que outro tenant não lê meu tráfego?**
Cada tenant tem uma **conta NATS** própria. Não é uma regra de permissão que pode estar
mal escrita — é o roteamento que sequer considera subjects de outra conta. O teste
T-SEC-01 verifica isso com o curinga mais amplo possível.

**Preciso que dois tenants troquem eventos. É possível?**
Sim, por *export/import* declarado entre contas — capability `bus:account:export`, de
sensibilidade **crítica**, com justificativa auditada (UC-010). É exceção documentada,
nunca configuração de rotina.

**Um serviço pode publicar evento fingindo ser outro módulo?**
Não: cada subject tem um `producer_module` declarado e a publicação por outro é recusada
com `AIOS-BUS-0010`, além de alertada como sinal de segurança (`./Security.md` §4).

**O barramento cifra minhas mensagens fim a fim?**
Não por default: cifrar o envelope inteiro quebraria a validação no publish, que é uma
garantia central do módulo. Tudo trafega sob TLS/mTLS e os volumes são cifrados. Se um
caso de uso exigir sigilo perante a própria plataforma, cifre apenas o campo `data`,
mantendo o envelope legível.

**Como apago uma mensagem já publicada (direito ao esquecimento)?**
Não se apaga: o log do JetStream é imutável. As mensagens expiram pelo `max_age` do
stream, e o expurgo do dado autoritativo é feito no `../005-Database/`. É exatamente por
isso que a **minimização de payload é obrigatória**, não recomendada (`./Security.md` §8).

---

## A2A

**Qual a diferença entre A2A e um canal de grupo?**
Grupo é **difusão** (um para muitos, sem estado). A2A é **conversa** (um para um, com
sessão, capacidades negociadas, cota e trilha). Usar A2A para difusão desperdiça
sessões; usar grupo para conversa privada perde a governança.

**Por que preciso negociar capacidades?**
Porque "abrir sessão" em abstrato não diz nada sobre o que os agentes poderão fazer. O
PDP autoriza **cada capacidade individualmente** (T-04); e elas não podem ser ampliadas
depois (invariante I4) — do contrário, a autorização inicial viraria cheque em branco.

**Posso conversar com um agente de outra organização?**
Se `bus.a2a.allow_external_peers` estiver habilitado para o tenant e a identidade do par
for verificável (mTLS + JWT federado). As capacidades concedidas a par externo são um
subconjunto restrito, conforme o perfil da RFC-0201.

**Minha sessão foi para `Suspended`. Por quê?**
Ou a cota de tráfego estourou, ou o PDP revogou uma capacidade em uso
(`policy.bundle.updated`). Mensagens são **recusadas** com `AIOS-A2A-0004`, não
enfileiradas — enfileirar acumularia trabalho que talvez nunca seja permitido, para
entregá-lo em lote no pior momento.

---

## Operação

**O PDP caiu. O barramento para?**
Não. `fail_mode=closed` nega **operações novas** (declarar stream, abrir sessão A2A); o
**tráfego já autorizado continua fluindo**. Um barramento que para porque o serviço de
política caiu transformaria uma falha localizada em queda total (princípio P-09).

**Perdi o quórum de JetStream. Perdi mensagens?**
Não, se os produtores usam Outbox: a publicação durável falha com `AIOS-BUS-0011` e a
mensagem permanece no banco até o quórum voltar. O core pub/sub continua funcionando
durante a janela.

**Posso atualizar os três nós do NATS ao mesmo tempo?**
Não. Um por vez, aguardando o `jsz` reportar todos os streams com réplicas em dia. Essa
espera é a diferença entre uma atualização de rotina e um incidente de perda de quórum
(`./Deployment.md` §5).

**Meu stream está em 95% do `max_bytes`. O que acontece?**
Depende do `discard`: com `old`, mensagens antigas são descartadas (perda silenciosa de
histórico); com `new`, novas publicações falham com `AIOS-BUS-0013`. Streams de auditoria
usam `new`; telemetria usa `old`. Em ambos os casos, investigue o produtor.

---

## Referências

- Visão e escopo: `./Vision.md` · Eventos: `./Events.md`
- Exemplos: `./Examples.md` · API: `./API.md`
- Segurança: `./Security.md` · Escala: `./Scalability.md` · Operação: `../029-Operations/`
