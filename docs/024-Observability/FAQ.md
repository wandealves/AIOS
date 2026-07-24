---
Documento: FAQ
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0243, ADR-0244, ADR-0245, ADR-0246, ADR-0247, ADR-0248, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: Vision.md, API.md, StateMachine.md, Examples.md
---

# 024-Observability — Perguntas Frequentes

## 1. Escopo — o que este módulo faz e não faz

**O 024 é dono das minhas métricas?**
Não. Cada módulo **define e emite** as suas; o 024 **admite, governa, armazena e
serve**. O `owner_module` do descritor é seu — inclusive a responsabilidade de
corrigi-lo quando a cardinalidade estourar.

**Qual a diferença entre 024 e 025-Audit?**
Telemetria é **amostrável, perecível e pode perder dados sob pressão**; a trilha de
auditoria não pode nenhuma das três coisas (ADR-0246). Se você precisa provar que algo
aconteceu, é `../025-Audit/`. Se precisa saber como o sistema está se comportando, é
aqui.

**O 024 responde ao incidente?**
Não. Ele **detecta, dispara e roteia** — com o runbook anexado. A resposta é de
`../029-Operations/`.

**O 024 decide rollback?**
Não. Ele emite `telemetry.budget.exhausted`; quem decide é `../028-Deployment/`
(releases) e `../022-Policy/` (publicações de política).

**Por que a telemetria não trafega pelo NATS?**
Porque o volume de observabilidade competiria com o tráfego de negócio no mesmo
recurso. Métricas, traces e logs usam OTLP direto ao coletor; o barramento carrega
apenas **eventos** do módulo (`./Events.md` §5).

## 2. Ingestão e perda

**Por que a ingestão nunca devolve `429`?**
Porque isso faria o SDK do serviço observado gastar CPU em retry justamente quando o
sistema está sob pressão — transformando a falha do observador em falha do observado
(ADR-0243). Sob saturação, o gateway **descarta e contabiliza**.

**Então eu posso estar perdendo telemetria sem saber?**
Saber é obrigatório: `aios_obs_dropped_total` mede a perda e
`telemetry.ingest.degraded` a anuncia acima de 1%. O *fail-open* torna a perda
silenciosa por construção; essas duas coisas a tornam audível.

**Se eu perder telemetria, perco também os dados de auditoria?**
Não. São subsistemas disjuntos (ADR-0246). A trilha do `../025-Audit/` tem garantias
próprias e não passa por este pipeline.

**Quanto de telemetria posso perder numa queda do gateway?**
Até 1 minuto (NFR-011), limitado pelo buffer em disco do `otel-agent`
(`obs.ingest.disk_buffer_mb`, 2 GiB).

**Posso ligar `obs.ingest.fail_mode = closed`?**
Não em produção. A chave é não-recarregável e existe apenas para diagnóstico em
laboratório: `closed` transformaria uma saturação do coletor em indisponibilidade de
todos os serviços instrumentados.

## 3. Registro de sinais e cardinalidade

**Por que preciso registrar a métrica antes de emitir?**
Porque o registro é o **único ponto do sistema capaz de impedir a explosão de
cardinalidade antes de ela acontecer** (ADR-0241). Depois que a série existe, o custo
já foi pago.

**Minha métrica foi quarentenada. Perdi o histórico?**
Não. A quarentena descarta a ingestão **daquele sinal** enquanto durar; o que já foi
coletado permanece pela retenção normal. A quarentena é reversível (invariante S2):
corrija o descritor e reabilite.

**Por que não posso usar `agent_urn` como label?**
Com 10⁶ agentes, isso cria 10⁶ séries temporais de uma vez — o observador passaria a
custar mais que o observado. A granularidade individual vive em **trace e log**, onde
alta cardinalidade é normal e barata.

**Como faço para investigar um agente específico, então?**
Pelo `trace_id` no log e no trace (`./Examples.md` §8). É um salto, não uma correlação
por horário — desde que você propague `traceparent` (RFC-0001 §5.6).

**Recebi alerta de cardinalidade. Aumento o teto?**
Não como primeira ação. Isso adia o problema e fixa o custo mais alto
permanentemente. Identifique o label ofensor (`offendingLabels` no evento), remova-o do
descritor em nova versão e reabilite (`./Monitoring.md` RB-O-03).

**Por que buckets de histograma precisam ser explícitos?**
Buckets automáticos não caem nas fronteiras dos seus limiares de SLO, e
`histogram_quantile` sobre eles devolve um número que **parece** um p99 mas foi
interpolado entre fronteiras erradas.

## 4. Traces e amostragem

**Por que amostrar traces?**
Porque reter 100% em 10⁶ agentes é inviável em armazenamento. A pergunta certa não é
"quanto amostrar", é **"o quê nunca amostrar fora"**.

**O que nunca é descartado?**
Traces com erro em qualquer span e traces acima de
`obs.trace.tail_keep_slow_ms` — 100% retidos (ADR-0244). O *head sampling* puro
falharia aqui: ele descartaria 90% dos erros, que são os raros e os úteis.

**Vi traces incompletos durante um deploy. É bug?**
Durante rollout do `otel-gateway`, spans do mesmo trace podem cair em réplicas
diferentes. É **esperado e temporário** (`./Deployment.md` §6);
`aios_obs_trace_fragmented_total` sobe e normaliza. Não é motivo de rollback.

## 5. SLO e alertas

**Por que meu SLO não pode ser 100%?**
Porque um SLO de 100% não deixa error budget — e um serviço sem error budget não pode
fazer nenhuma mudança, o que na prática significa que o objetivo será ignorado. O
`CHECK` do banco recusa `objective >= 1`.

**Por que *multi-burn-rate* em vez de um limiar simples?**
Limiar absoluto avisa quando o budget já acabou (tarde demais); taxa instantânea
dispara a cada oscilação (ruído que leva ao silêncio). A combinação de janela curta
(detecta) com janela longa (confirma) resolve os dois (`./StateMachine.md` §6).

**Por que toda regra precisa de runbook?**
Porque um alerta sem procedimento transfere ao plantonista, às 3h da manhã, o trabalho
de descobrir o que fazer — e o resultado previsível é o alerta ser **silenciado em vez
de tratado** (ADR-0247).

**Meu alerta sumiu sozinho. Foi resolvido?**
Verifique o estado: `Resolved` significa que a condição cessou; `Expired` significa que
**parou de haver dado** (invariante I6). São coisas opostas — o alvo que caiu deixou de
reportar exatamente a métrica que provaria a queda.

**Silenciei o alerta. Ele para de ser avaliado?**
Não. O silêncio suprime a **notificação**, nunca a **observação** (invariante I3). Se
a condição cessar durante o silêncio, a instância vai a `Resolved` normalmente.

**Posso criar um silêncio permanente?**
Não. `expires_at` é obrigatório com teto de 72 h. Silêncio permanente é o alerta
excluído sem que ninguém tenha decidido excluí-lo.

**Silencio o mesmo alerta toda semana. E daí?**
`ObservabilitySilenceChurnHigh` dispara após 3 silêncios em 30 dias. Isso indica uma de
duas coisas: o **limiar está errado** (corrija a regra) ou o **problema é real e está
sendo ignorado** (`./Monitoring.md` RB-O-09).

## 6. Consulta e privacidade

**Por que a consulta é negada quando o PDP cai, mas a ingestão continua?**
Assimetria deliberada (`./Security.md` §2.3): negar uma autorização é seguro; negar uma
métrica não protege nada e ainda quebra quem a emitia.

**Minha consulta foi abortada. Por quê?**
Orçamento de tempo (`obs.query.max_duration_s`) ou de séries varridas
(`max_series_scanned`). Normalmente a causa é um painel agregando ao vivo o que
deveria ser *recording rule*.

**Posso ver a telemetria de outro tenant?**
Não. Rótulo de tenant obrigatório, `TenantRouter` e RLS — três camadas, e a consulta
cruzada é negada em todas (NFR-017).

**Por que meu atributo não aparece no log?**
Se está classificado como `pii`, foi redigido ou tokenizado **na borda**, antes de
qualquer persistência (ADR-0249). Redigir no armazenamento significaria que o dado já
esteve em claro em disco.

**Posso logar o prompt do agente para depurar?**
Não. Conteúdo de negócio (prompt, resposta de LLM, item de memória) não entra em log,
span ou métrica. Use o `trace_id` para correlacionar com o módulo dono do dado.

## 7. Armadilhas comuns

| Armadilha | Consequência | Correção |
|-----------|--------------|----------|
| `agent_urn`/`trace_id` como label de métrica | 10⁶ séries; consulta degrada | Granularidade individual em trace/log |
| Emitir métrica sem registrar | Quarentena silenciosa; painel vazio | Registrar no pipeline de CI do módulo |
| `head_sample_ratio = 1.0` em produção | Volume 10× maior sem ganho | 10% + *tail sampling* de erro e cauda |
| Tratar `dropped_total` como erro a zerar a qualquer custo | Escalar sem necessidade | É comportamento projetado; monitore a **taxa** |
| Painel agregando 30 d ao vivo | Consultas abortadas | *Recording rule* pré-calculada |
| Aumentar `max_series_per_tenant` ao receber alerta | Custo fixado mais alto | Quarentenar e corrigir o descritor |
| Log em `Debug` em produção | O log compete com o pipeline que o transporta | `Debug` só em desenvolvimento |
| Assumir que ausência de alerta = saúde | Falha silenciosa | Verificar `alert.nodata` e o probe sintético |
| Usar o log de aplicação como prova de auditoria | Retenção de 14 d; amostrável | `../025-Audit/` |
| Runbook desatualizado | Instruções que não funcionam no incidente | `ObservabilityRunbookStale` (180 d) |

## 8. Operação

**Como sei que o sistema de alerta está funcionando?**
Pelo **probe sintético** (`./Testing.md` §9): uma vez por minuto, uma condição
artificial dispara e o tempo até a notificação é medido. É a única verificação possível
de uma capacidade cuja falha é, por definição, silenciosa.

**Quem observa o observador?**
O `SelfTelemetry`, *scrapeado* diretamente pelo Prometheus, **sem passar pelo
pipeline** (`./FailureRecovery.md` §6). Se dependesse do pipeline, a única falha que
ele não conseguiria reportar seria a do próprio pipeline.

**O que acontece se o PostgreSQL cair?**
A **ingestão continua** com o registro de sinais em cache; administração e SLO param.
A hierarquia de degradação preserva a observação e sacrifica a mudança da
observabilidade (`./FailureRecovery.md` §5).

**Meu serviço novo não aparece em lugar nenhum. Por onde começo?**
Verifique, nesta ordem: (1) `aios.tenant` e `service.name` nos atributos de recurso;
(2) certificado mTLS válido; (3) os sinais estão **registrados**;
(4) `aios_obs_dropped_total` com `reason="unregistered"`.

## 9. Referências

- Visão e fronteiras: `./Vision.md` · Contratos: `./API.md`
- Comportamento: `./StateMachine.md` · Exemplos: `./Examples.md`
- Operação: `./Monitoring.md`, `./FailureRecovery.md`
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`

*Fim de `FAQ.md`.*
