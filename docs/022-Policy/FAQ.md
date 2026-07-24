---
Documento: FAQ
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0222, ADR-0224, ADR-0225, ADR-0227, ADR-0229
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: Vision.md, API.md, StateMachine.md, Examples.md
---

# 022-Policy — Perguntas Frequentes

## 1. Escopo — o que este módulo faz e não faz

**O 022 bloqueia a ação do agente?**
Não. Ele **decide**; quem impede é o PEP do módulo chamador. Essa separação é o que
permite auditar a autorização independentemente de quem a aplicou. Se o seu PEP
recebeu `deny` e deixou passar, o defeito é do PEP.

**Qual a diferença entre 021, 022 e 025?**
O `../021-Security/` responde **quem é** o solicitante; o 022 responde **se ele pode**;
o `../025-Audit/` registra **que aconteceu**. Confundir as três perguntas é o erro de
desenho mais comum em plataformas com governança — e a razão de serem três módulos.

**O 022 controla orçamento e cota?**
Não. Ele **lê** o orçamento consumido como atributo ABAC (`budget.*`, vindo do
`../026-Cost-Optimizer/`) e pode negar com base nele. Definir o orçamento e contabilizar
o gasto é do `026`; cotas de recurso são do `../006-Kernel/`.

**O 022 aplica sandbox?**
Não. Ele pode **exigir** um perfil por obrigação (`sandbox_profile`), mas quem publica
o perfil é o `../021-Security/` e quem o aplica é o `../007-Agent-Runtime/`.

## 2. Comportamento da decisão

**Por que `deny` vem com HTTP `200`?**
Porque negação é o produto normal do módulo, não uma falha. Se `deny` fosse `4xx`, todo
PEP trataria política restritiva como incidente — e a reação a incidente é sempre
afrouxar. Os códigos `AIOS-POL-*` cobrem falhas de administração e de avaliação
(`./API.md` §3.5).

**Recebi `deny` com `no_matching_rule`. É bug?**
Não: é o *default deny*. Nenhuma regra do bundle ativo cobre esse par ação/recurso para
esse sujeito. Use `ExplainDecision` para ver quais regras foram avaliadas e descartadas
(`./Examples.md` §8).

**O que acontece se o tenant não tiver política publicada?**
Tudo é negado, com `reason_code = no_active_bundle` (invariante I5). Ausência de
política **não** é permissão — se fosse, a forma mais rápida de desligar a segurança do
AIOS seria apagar um registro.

**Por que nunca recebo `indeterminate`?**
Porque `indeterminate` e `not_applicable` são colapsados em `deny` dentro do módulo
(ADR-0224). Expor uma terceira via convidaria cada PEP a inventar seu tratamento — e
algum deles trataria "não sei" como "pode".

**Posso ignorar uma obrigação que meu PEP não conhece?**
Não. Obrigação desconhecida **DEVE** virar `deny` (ADR-0225). Ignorá-la aplicaria uma
decisão que o PDP não tomou: o `allow` estava condicionado àquela obrigação.

**Como funciona a precedência entre regras?**
Bundle ausente ⇒ `deny`; `deny` de escopo `platform` vence tudo; menor `priority`
decide; empate ⇒ `deny` vence `allow`; waiver vigente pode converter `deny` dispensável
em `allow`; nada casou ⇒ `deny` (`./StateMachine.md` §6).

## 3. Desempenho

**O PDP não vira gargalo, estando no caminho de tudo?**
Duas camadas evitam isso: o **cache no PEP** (com TTL emitido pelo próprio PDP) absorve
a maioria das decisões, e o **bundle compilado em memória** faz as restantes custarem
milissegundos, sem I/O de banco (`./Scalability.md` §2).

**Por que o TTL de `deny` é menor que o de `allow`?**
Para que uma correção de política restaure acesso mais rápido do que o removeria.
Inverter isso faria a negação persistir no cache além do necessário exatamente durante
um incidente (`./Configuration.md` §1).

**Posso desligar o cache do PEP?**
Pode, mas multiplica a carga sobre o 022 por ~50× e não melhora a correção: a
invalidação granular (`policy.decision.updated`) já limita a janela de decisão obsoleta
a segundos.

**Por que a decisão não é gravada antes de responder?**
Gravar antes trocaria o p99 do PDP pelo p99 do PostgreSQL. O journal é escrito
assincronamente, fora do caminho de resposta (invariante estrutural S6).

## 4. Ciclo de vida da política

**Posso editar uma regra do bundle em vigor?**
Não. O bundle é imutável a partir de `Validated` (`AIOS-POL-0020`). Alterar exige nova
versão — do contrário, decisões passadas deixariam de ser reproduzíveis (NFR-009).

**Por que preciso simular antes de publicar?**
Porque o diferencial mostra, **antes** da publicação, quantas operações que hoje passam
deixarão de passar e por qual regra. Sem isso, a primeira medição do efeito de uma
política é o incidente.

**Publiquei e a taxa de negação explodiu. O que faço?**
Rollback (`./Examples.md` §6): uma transação e uma propagação de ≤ 10 s. Depois compare
o observado com o diferencial da simulação — a divergência indica tráfego não coberto
pela janela simulada.

**Rollback de política é a mesma coisa que rollback de deploy?**
Não, e confundi-los durante um incidente é o erro operacional mais provável. Um troca a
**imagem** do serviço; o outro troca o **bundle** (`./Deployment.md` §6).

**Por que preciso de um aprovador diferente do autor?**
Separação de funções (`AIOS-POL-0018`), imposta inclusive por `CHECK` constraint no
banco. Um controle de governança que vive só no código é desligável por um deploy.

**Quanto tempo posso reverter para trás?**
`pol.bundle.rollback_retention_versions` versões (default 10). Além disso o bundle vai
a `Archived` e o artefato é liberado — sem ele, nem rollback nem explicação de decisões
daquela época são possíveis (invariante I6).

## 5. Exceções (waivers)

**Posso conceder uma exceção permanente?**
Não. `expires_at` é `NOT NULL` com teto de 72 h. Exceção sem prazo é política nova
aprovada por ninguém (ADR-0227).

**E se eu precisar renovar toda semana?**
O alerta `PolicyExceptionChurnHigh` dispara após 3 concessões em 30 dias para o mesmo
par (sujeito, ação). Isso não é falha do processo: é o sinal de que aquilo deveria ser
uma **regra revisada**, não uma exceção perpétua.

**Waiver contorna qualquer negação?**
Não. Regras marcadas `dispensable: false` e qualquer `deny` de escopo `platform` são
imunes a waiver.

## 6. Atributos

**Uma fonte de atributo caiu. Minhas ações são liberadas?**
O contrário: se a fonte é `required`, a decisão vira `deny` com
`attribute_unavailable`. Se é `optional`, a regra que a exige é descartada e a avaliação
segue com as demais.

**Posso mudar `fail_mode` para `open` durante um incidente?**
**Não deveria.** Isso transforma "não consegui verificar" em "pode", exatamente quando
o sistema está sob estresse. A opção existe para desenvolvimento local
(`./Configuration.md` §9).

**Por que meu atributo não aparece no `Explain`?**
Atributos marcados `sensitive` nunca têm o valor registrado — apenas o `sha256` do
conjunto (`attributes_digest`). O `Explain` mostra **quais** atributos foram usados, não
seus valores (FR-023).

## 7. Armadilhas comuns

| Armadilha | Consequência | Correção |
|-----------|--------------|----------|
| Vincular papel sujeito a sujeito, com 10⁶ agentes | `role_binding` gigante; grafo pesado em memória | Vincular **grupos**; usar ABAC sobre atributo do sujeito |
| Marcar toda fonte de atributo como `required` | Degradação alheia vira negação própria | Reclassificar como `optional` o que não é decisivo |
| `allow_sample_rate = 1.0` em produção | Journal com 10⁵ escritas/s | Manter 0,05; `deny` já é 100% e não é configurável |
| Tratar taxa de `deny` como taxa de erro | Alarme constante; pressão para afrouxar | Monitorar a **variação** (`PolicyDenyRateSpike`), não o valor absoluto |
| Usar o log de aplicação para explicar decisões | Depende de nível de log e de 14 dias de retenção | Usar o journal e `ExplainDecision` (`./Logging.md` §1) |
| Aumentar `eval_budget_ms` para resolver timeout | Esconde a regra custosa e degrada todo mundo | Corrigir a regra em nova versão (`./Monitoring.md` RB-P-07) |
| Publicar com `skip_simulation` fora do bootstrap | Perde-se a única medição prévia do impacto | Só é permitido sem linha de base (T-09) |
| Assumir "sempre a maior `bundleVersion`" no consumidor | Desfaz silenciosamente um rollback | Respeitar `reason = "rollback"` (`./Events.md` §6) |

## 8. Integração

**Quais superfícies existem e qual devo usar?**
gRPC para o caminho quente interno; NATS request-reply para quem já vive no barramento;
REST para autoria, ferramentas e console. Os três falam o mesmo contrato (RFC-0220).

**Como sei quando invalidar meu cache?**
Dois eventos, dois comportamentos: `policy.bundle.updated` invalida **tudo** do tenant
(raro); `policy.decision.updated` invalida **apenas o sujeito afetado** (frequente).
Consumir só o primeiro faria cada concessão de papel derrubar o cache global.

**Preciso enviar `Idempotency-Key` no `Evaluate`?**
Não: avaliação é leitura pura e repetir é sempre seguro. A chave é obrigatória nas
mutações administrativas (RFC-0001 §5.5).

**Meu módulo é novo. Por onde começo?**
Implemente o PEP com cache por TTL, consuma os dois eventos de invalidação, trate
obrigação desconhecida como `deny` e defina `fail_mode = closed`. O restante é o
contrato de `./API.md`.

## 9. Referências

- Visão e fronteiras: `./Vision.md` · Contratos: `./API.md`
- Comportamento: `./StateMachine.md` · Exemplos: `./Examples.md`
- Operação: `./Monitoring.md`, `./FailureRecovery.md`
- Decisão-mãe: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`

*Fim de `FAQ.md`.*
