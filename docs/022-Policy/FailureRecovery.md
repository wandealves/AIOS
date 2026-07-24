---
Documento: FailureRecovery
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy + SRE
ADRs relacionados: ADR-0008, ADR-0221, ADR-0224, ADR-0226
RFCs relacionados: RFC-0001
Depende de: _DESIGN_BRIEF.md §9, Monitoring.md, StateMachine.md, Deployment.md
---

# 022-Policy — Falhas e Recuperação

## 1. Princípio

Toda falha deste módulo resolve-se para **`deny`**. Não há modo degradado permissivo:
falta de política, falta de atributo, falta de tempo e falta do próprio PDP produzem
negação (RFC-0001 §5.8, ADR-0008). A consequência aceita é que uma falha do 022
**para o sistema** — e é por isso que o SLO de disponibilidade é 99,99% (NFR-003) e
que a avaliação é a última capacidade a ser sacrificada (§5).

## 2. FMEA

| # | Modo de falha | Efeito | Sev. | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---|---------------|--------|------|----------|-----------|-------------|--------------------|
| F-01 | PDP indisponível (visto pelo PEP) | Ações privilegiadas negadas | Alta | Timeout/CB no `PolicyClient` do chamador | O PEP aplica seu `fail_mode` (default `closed`) | Réplicas voltam com bundle assinado carregado do registro | Avaliação é leitura pura: retry sempre seguro |
| F-02 | PostgreSQL (`policy`) indisponível | Autoria, publicação e journal param | Média | Health/readiness | **Avaliação continua** com o bundle em memória | Failover do `../005-Database/`; journal represado é drenado | Idempotente |
| F-03 | Fonte de atributo `required` indisponível | Regras dependentes negam | Média | Timeout no `AttributeResolver` | Apenas as regras que usam aquele atributo | `deny` com `attribute_unavailable`; evento `attribute.degraded`; retomada automática | Sem retry cego dentro do orçamento |
| F-04 | Bundle ativo corrompido / assinatura inválida | Réplica não serve | Alta | Verificação na carga (NFR-015) | Réplica recusa ficar *ready* e sai do balanceamento | Recarrega do registro; se persistir, rollback para versão íntegra | Carga idempotente |
| F-05 | Publicação que quebra produção | Salto de `deny`; operações legítimas bloqueadas | Alta | `PolicyDenyRateSpike` | Confinado ao tenant | **Rollback (T-11/T-12)**; a simulação obrigatória previne a recorrência | Rollback idempotente por `Idempotency-Key` |
| F-06 | Falha de propagação a uma réplica | Réplica decide com política velha | Alta | `PolicyBundleVersionSkew` | Réplica retirada do balanceamento antes de decidir com versão defasada | Repropagação forçada; alerta se exceder `propagation_target_s` | Dedupe por `event.id` |
| F-07 | Explosão de complexidade de política | Latência acima do SLO | Média | `aios_pol_compile_duration_ms`, latência p99 | Bundle **rejeitado na compilação** (`AIOS-POL-0016`), nunca em produção | Refatoração da política em nova versão | — |
| F-08 | Estouro do orçamento de avaliação | Decisões individuais negadas | Média | `aios_pol_eval_timeout_total` | Decisão individual, não a réplica | `deny` (`eval_timeout`); investigar a regra custosa | Retry do chamador é seguro |
| F-09 | Tempestade de decisões (tenant abusivo) | Saturação de CPU | Média | `aios_pol_rate_limited_total` | Limite **por tenant**, nunca global | `AIOS-POL-0010` com `Retry-After`; cache do PEP absorve o restante | Retriable |
| F-10 | Waiver esquecido em aberto | Acesso mais amplo que o pretendido | Média | Varredura de `expires_at`; `PolicyExceptionChurnHigh` | — | Expiração automática; revisão semanal | Determinístico |
| F-11 | Perda do journal de decisões | Explicabilidade recente degradada | Média | Gap de partição; `aios_pol_journal_dropped_total` | Escrita assíncrona já isolou o caminho quente | Reconstrução parcial pela trilha do `../025-Audit/`; simulação usa janela menor | — |
| F-12 | Divergência de versão entre réplicas | Duas políticas simultâneas | **Crítica** | `aios_pol_bundle_version_skew > 0` | Réplica divergente retirada | Repropagação forçada; alarme P1 — viola I1 na prática | — |
| F-13 | Ausência de bundle ativo | Tudo negado no tenant | **Crítica** | `PolicyNoActiveBundle` | Confinado ao tenant | Ativar a última versão íntegra (T-12) ou publicar bootstrap (T-09) | Idempotente |
| F-14 | NATS/JetStream indisponível | Invalidação de cache atrasada | Média | `aios_pol_outbox_pending` | Decisão continua; caches dos PEPs envelhecem até o TTL | Drenagem do outbox ao restabelecer | Dedupe por `event.id` |
| F-15 | Redis indisponível | Mais chamadas a fontes externas; latência sobe | Baixa | `attribute_cache_hit_ratio` cai | Cache local por réplica assume | Reconexão automática | Idempotente |

## 3. Detecção e isolamento por dependência

```
   ┌────────────────────────────────────────────────────────────────┐
   │  022-Policy                                                     │
   │                                                                 │
   │   BundleRuntime (memória) ──── independe de TODAS as deps ───┐  │
   │        │                                                      │  │
   │        ▼                                     avaliação segue │  │
   │   AttributeResolver ──[CB + timeout 20 ms]──▶ 021 / 026 / 024 │  │
   │   BundleRegistry    ──[health]─────────────▶ PostgreSQL       │  │
   │   AttributeCache    ──[degrada p/ local]───▶ Redis            │  │
   │   EventEmitter      ──[outbox represa]────▶ NATS              │  │
   └────────────────────────────────────────────────────────────────┘
```

Cada dependência tem *circuit breaker* próprio (bulkhead): a lentidão de uma fonte de
atributo **NÃO DEVE** consumir o orçamento de avaliação de regras que não a usam. Essa
é a diferença entre "uma parte da política degrada" e "o PDP para".

## 4. Retries, backoff e DLQ

| Operação | Retry | Backoff | DLQ |
|----------|-------|---------|-----|
| `Evaluate` (chamador) | Sim, ilimitado do lado do PEP | Exponencial com jitter, dentro do CB | — |
| Resolução de atributo | **Não** dentro da mesma avaliação | — | — |
| Publicação de outbox | Sim | Exponencial (200 ms → 30 s) | `POLICY_DLQ` após 10 tentativas |
| Consumo de evento | Sim | Exponencial | DLQ do stream de origem (`../020-Communication/`) |
| Compilação/simulação | Manual | — | — |

> Não há retry de resolução de atributo **dentro** da avaliação: o orçamento é de
> 50 ms e o timeout por fonte é de 20 ms. Retentar consumiria o orçamento e produziria
> `eval_timeout` — trocando um `deny` explicável (`attribute_unavailable`) por um
> `deny` genérico, mais lento.

Mensagens em `POLICY_DLQ` exigem inspeção humana: um `policy.bundle.updated` não
publicado significa PEPs com cache desatualizado, e o reprocessamento **DEVE** validar
que a versão ainda é a ativa antes de republicar.

## 5. Degradação graciosa

Ordem de sacrifício, do primeiro ao último:

```
   1. Simulação e testes          ← perde-se a capacidade de PROVAR política nova
   2. Autoria e publicação        ← opera-se com a política de ontem
   3. Journal (amostragem de allow) ← perde-se explicabilidade de allow
   4. Journal de deny             ← INCIDENTE: explicabilidade de negação é obrigatória
   5. AVALIAÇÃO DE DECISÃO        ← nunca; o AIOS não executa nada sem ela
```

Um AIOS que não publica política nova opera com a política de ontem. Um AIOS que não
decide **não executa nada** — toda ação privilegiada depende de uma decisão. Por isso
a avaliação é a última a cair, e por isso o `BundleRuntime` em memória não depende de
nenhuma outra dependência para continuar respondendo (§3).

## 6. RTO / RPO

| Cenário | RTO | RPO | Procedimento |
|---------|-----|-----|--------------|
| Perda de réplica | ~0 (balanceador) | 0 | Réplicas restantes atendem; HPA repõe |
| Perda total do serviço | **≤ 15 min** | **≤ 5 min** | Redeploy; réplicas carregam o bundle assinado do registro |
| Perda do PostgreSQL | ≤ 15 min | ≤ 5 min | Failover do `../005-Database/`; avaliação seguiu funcionando durante a queda |
| Política errada publicada | ≤ 15 min | 0 | Rollback (UC-007): transação + propagação ≤ 10 s; o tempo é de decisão humana |
| Perda de região | ≤ 15 min | ≤ 5 min | `../027-Cluster/`; bundle é artefato replicado e assinado, reconstruível |

O RPO de 5 min aplica-se a **configuração de política** (bundles em edição) e ao
journal. O bundle **ativo** tem RPO efetivo de **0**: é artefato assinado, replicado e
carregado em memória por todas as réplicas.

## 7. Testes de caos

| Experimento | Hipótese | Critério de sucesso |
|-------------|----------|---------------------|
| Derrubar PostgreSQL por 10 min | Avaliação continua; administração para | 0 erro em `Evaluate`; `AIOS-POL-0009`/`5xx` só na superfície administrativa |
| Injetar latência de 5 s na fonte `budget` | Só regras que usam `budget.*` negam | Regras independentes seguem com p99 ≤ 5 ms |
| Corromper o artefato de uma réplica | Réplica não fica *ready* | Réplica fora do balanceamento em < 30 s; `PolicySignatureVerificationFailed` |
| Publicar bundle que nega tudo | Alerta dispara; rollback restaura | `PolicyDenyRateSpike` em < 5 min; rollback restaura em ≤ 10 s de propagação |
| Particionar NATS por 15 min | Caches envelhecem até o TTL; nada quebra | Outbox drena sem perda; nenhuma decisão incorreta |
| Matar o líder do `policy-outbox-relay` | Novo líder assume | Backlog drenado; sem eventos duplicados além do dedupe normal |
| Remover o bundle ativo de um tenant | Tudo é negado, nada é permitido | `PolicyNoActiveBundle`; **0** `allow` no período |

O último experimento é o mais importante: ele verifica a invariante I5 na prática.
Se algum `allow` aparecer com o tenant sem bundle, o *default deny* é ficção.

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §9
- Alertas e runbooks: `./Monitoring.md` §4 e §5
- FSM e invariantes: `./StateMachine.md` §5
- Implantação e readiness: `./Deployment.md` §4 · Escala: `./Scalability.md`

*Fim de `FailureRecovery.md`.*
