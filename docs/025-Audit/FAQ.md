---
Documento: FAQ
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0246, ADR-0250..ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: Vision.md, API.md, StateMachine.md, Examples.md
---

# 025-Audit — Perguntas Frequentes

## 1. Escopo — o que este módulo faz e não faz

**Qual a diferença entre 024-Observability e 025-Audit?**
O 024 responde **o que está acontecendo agora e como o sistema se comporta**; o 025
responde **o que aconteceu e pode ser provado**. Telemetria é amostrável, tem retenção
curta e pode perder dados sob pressão; a trilha não pode nenhuma das três coisas
(ADR-0246).

**Posso usar a trilha para montar um painel?**
Não deveria. A trilha é otimizada para prova, não para agregação: consultas são caras,
o payload é cifrado e cada consulta gera um registro. Métricas operacionais são do
`../024-Observability/`.

**O 025 decide alguma coisa?**
Não. Ele **registra** decisões (do `../022-Policy/`), emissões (do `../021-Security/`)
e execuções. Aplicar um *legal hold* é executar uma decisão jurídica tomada fora do
módulo, com autor e aprovador registrados.

**O 025 apaga o dado de negócio?**
Não. Ele apaga **o payload da própria trilha** e **registra o comprovante** dos
expurgos executados por `../005-Database/`, `../010-Memory/` e
`../024-Observability/`.

**O 025 alerta o plantão?**
Não. Ele emite `audit.anomaly.detected`; quem roteia o alerta é o
`../024-Observability/` e quem responde é o `../029-Operations/`.

## 2. Durabilidade e o `503`

**Por que recebi `503 AIOS-AUD-0005`? Isso é um bug?**
Não — é o desenho. A escrita só é confirmada após durabilidade (invariante I9). O
`503` significa "não consegui garantir; reenvie". Seu outbox deve reter o fato e
reenviar com a **mesma** `Idempotency-Key`.

**Não seria melhor aceitar e gravar depois?**
Seria pior. Aceitar sem durabilidade troca uma indisponibilidade **visível** por uma
lacuna **invisível** — e a lacuna é o defeito que este módulo existe para impedir
(`./Vision.md` §5).

**O 024 responde `200` e descarta. Por que o 025 não faz o mesmo?**
Porque as garantias são opostas. Telemetria não pode **atrasar o emissor**; auditoria
não pode **perder o fato**. Cada módulo é fail-open ou fail-closed conforme a garantia
que precisa preservar (`./Security.md` §2.4).

**Posso desligar `require_durable_ack` em desenvolvimento?**
Não. A chave é não-recarregável e `true` em **todos** os ambientes. Um desenvolvedor
que aprende a conviver com perda silenciosa não reconhece o defeito em produção.

**Reenviar cria registro duplicado?**
Não. O dedupe é por `(tenant_id, event_id)` / `Idempotency-Key` e devolve o registro
original com o **mesmo `record_hash`** (invariante I7).

## 3. Imutabilidade e integridade

**Como sei que a trilha não foi alterada?**
Cinco camadas (`./Security.md` §3): interface sem operação de mutação, *trigger* de
banco, cadeia de hash, selo de Merkle assinado com chave fora do módulo e cópia WORM.
Adulterar sem detecção exige comprometer **três sistemas distintos**.

**E se alguém tiver acesso de superusuário ao banco?**
Pode remover o *trigger* e alterar a linha — e a **cadeia detecta**, porque o
`record_hash` deixa de bater; o **selo assinado** detecta, porque a raiz de Merkle
diverge; e a **cópia WORM** detecta, porque não pode ser sobrescrita. A premissa do
modelo é que o operador do banco não é confiável.

**Posso corrigir um registro errado?**
Não. A correção é um **novo registro** com `outcome = compensated` referenciando o
original (ADR-0258). Uma capability de alteração seria uma capability de apagar
evidência.

**Por que a cadeia é particionada? Isso não enfraquece a prova?**
Não. Cada partição é uma cadeia completa e verificável isoladamente. Uma sequência
global única serializaria toda a escrita do tenant (~4.100 registros/s em vez de
~52.000 — `./Benchmark.md` §5). A ordem que importa numa investigação é **causal**
(`trace_id`, `occurred_at`), não total.

**Quanto tempo fico sem prova de selagem?**
No máximo `aud.seal.interval_s` (60 s). Nessa janela o `prev_hash` **já** detecta
alteração local; o selo fecha a janela contra a reescrita completa da cadeia.

## 4. Completude — a lacuna

**Por que a lacuna é pior que a adulteração?**
Um registro adulterado quebra o hash e é detectado. Um registro que **nunca chegou**
não deixa rastro algum. Por isso a sequência é monotônica sem lacuna e há um registro
do que **deveria** chegar (`expected_rate_per_day`).

**Recebi `class_silent`. O que significa?**
O volume de uma classe caiu abaixo de `aud.completeness.silence_ratio` (20%) do
esperado. Isso indica que um produtor **parou de emitir** — não que houve menos
atividade. Confirme com o dono antes de ajustar a expectativa
(`./Monitoring.md` RB-A-07).

**E se a classe legitimamente não tem atividade (fim de semana, feature desligada)?**
Ajuste `expected_rate_per_day` com registro do motivo, ou marque a classe
`deprecated`. **Não** desligue a expectativa — isso removeria a única forma de detectar
o silêncio real.

## 5. Retenção, *legal hold* e esquecimento

**Como imutabilidade e LGPD coexistem?**
Por **apagamento criptográfico** (ADR-0253): o payload é cifrado com uma chave por
titular; ao exercer o direito, a **chave é destruída**. O registro permanece na cadeia
com hash íntegro — prova-se que algo aconteceu, sem que o conteúdo pessoal seja
legível.

**O que sobra depois do apagamento?**
`seq`, `prev_hash`, `record_hash`, `payload_digest`, `occurred_at`, `record_class`,
`actor`, `action`, `resource_urn` e `outcome`. Some o **detalhe**, não o **fato**
(`./Database.md` §11.1).

**Por que o `payload_digest` sobrevive?**
Porque sem ele o apagamento seria indistinguível de um registro que sempre foi vazio —
e um adversário poderia alegar que nada havia ali.

**Meu pedido de esquecimento foi recusado com `423`. Por quê?**
Há um *legal hold* vigente sobre o escopo. A obrigação legal de preservação **sobrepõe**
o direito ao esquecimento (invariante I5). A recusa fica registrada na trilha, e você
é informado do `case_ref`.

**Por que o *legal hold* pode não ter prazo, se waiver e silêncio têm?**
Porque um *hold* não expira por decurso de prazo técnico; expira quando o **caso
encerra**. O controle compensatório é a revisão obrigatória a cada 90 dias, mais a
exigência de `case_ref` e `legal_basis` (`./StateMachine.md` §6.1).

**Posso reduzir a retenção para economizar armazenamento?**
Não abaixo de `aud.retention.min_days` — e a chave é **não-recarregável**. O caminho
correto para reduzir volume é o expurgo de **payload** por classe, que mantém a linha e
a cadeia.

## 6. Consulta e exportação

**Por que minha consulta à trilha aparece na trilha?**
Porque quem consultou o quê é exatamente o que um auditor precisa saber (ADR-0257). A
ausência desse registro seria a lacuna mais conveniente de todas.

**A consulta é negada quando o PDP cai. Isso não atrapalha uma investigação?**
Atrapalha, e é deliberado: sem poder autorizar, ninguém lê a trilha. A **ingestão**,
essa sim, continua — o que não pode faltar é o registro do que está acontecendo durante
o incidente.

**Como o auditor externo verifica sem confiar em nós?**
Pelo **pacote probatório** (`./Examples.md` §11): registros em serialização canônica,
selos assinados, provas de Merkle, chaves públicas e o `VERIFY.md`. O validador de
referência roda em ambiente isolado, sem rede para o AIOS.

**O que o auditor vê de um registro apagado?**
Metadados e `payloadDigest`, com o apagamento **explicado no manifesto** e o
comprovante correspondente. Não é um dado faltante; é um apagamento comprovado.

## 7. Armadilhas comuns

| Armadilha | Consequência | Correção |
|-----------|--------------|----------|
| Tratar `503 AIOS-AUD-0005` como erro a ignorar | Fato perdido silenciosamente | Reter no outbox e reenviar com a mesma chave |
| Registrar sem `Idempotency-Key` | Duplicatas na trilha após retry | Sempre enviar a chave |
| Payload gordo (dump completo do objeto) | Disco domina; consulta lenta | Mínimo necessário para provar a ação |
| Usar o log de aplicação do 025 como prova | Retenção de 14 dias; mutável | A prova é `audit.record` (`./Logging.md` §1) |
| Consultar sem `subject_ref`/`decision_ref` | Varredura de partições; `AIOS-AUD-0013` | Usar os dois eixos indexados + janela temporal |
| Reduzir `synchronous_commit` para ganhar vazão | **Destrói o RPO 0** | Particionar o banco; a chave é validada no *startup* |
| Aumentar partições sem migração | Sequência quebra a partir do ponto | Chave não-recarregável; exige migração |
| Ajustar `expected_rate_per_day` para silenciar alerta | Perde a detecção de produtor parado | Investigar o produtor primeiro |
| Fechar anomalia sem registrar a conclusão | Constraint impede — e com razão | `resolution` é obrigatória com `resolved_at` |
| Alterar a serialização canônica | Invalida a verificação de **todo** o histórico | Exige nova versão da RFC-0250 e coexistência |

## 8. Operação e integração

**Meu módulo é novo. Como faço para ser auditado?**
Registre a `record_class` (`./Examples.md` §4) com produtor, subjects, retenção, base
legal e volume esperado. A partir daí seus eventos entram na trilha — e a **ausência**
deles passa a ser detectada.

**Emito por evento ou por API?**
Por **evento**, sempre que o fato já for um evento do barramento (~99% dos casos). A
API direta serve para fatos que não são eventos (cerimônia manual, sistema externo) e
para comprovantes de expurgo.

**O que acontece se o 025 ficar fora por uma hora?**
Nada se perde: os produtores retêm no outbox e o JetStream retém as mensagens. A trilha
fica **atrasada** — o que é visível em `aios_aud_consumer_lag_s` e significa que uma
investigação naquele intervalo veria a trilha incompleta.

**Preciso do 025 no ar para o AIOS funcionar?**
Para operar, não. Para operar **em conformidade**, sim: RFC-0001 §5.8 exige registro
imutável de toda ação privilegiada, e o `../022-Policy/` audita cada decisão. Um
período longo sem trilha é um período sem prova.

## 9. Referências

- Visão e fronteiras: `./Vision.md` · Contratos: `./API.md`
- Comportamento: `./StateMachine.md` · Exemplos: `./Examples.md`
- Operação: `./Monitoring.md`, `./FailureRecovery.md`
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`

*Fim de `FAQ.md`.*
