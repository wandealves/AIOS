---
Documento: FailureRecovery
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit + SRE
ADRs relacionados: ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250
Depende de: _DESIGN_BRIEF.md §9, Monitoring.md, StateMachine.md, Deployment.md
---

# 025-Audit — Falhas e Recuperação

## 1. Princípio

Este módulo é ***fail-closed* nas duas pontas** — o único do AIOS com essa
característica. Na **escrita**, recusar (`AIOS-AUD-0005`) é preferível a aceitar sem
durabilidade; na **leitura**, negar é preferível a servir sem autorização.

A consequência aceita é que uma falha do 025 produz **indisponibilidade visível**, não
perda silenciosa. Os produtores mantêm outbox e reenviam: uma queda **atrasa** a
trilha, não a apaga.

## 2. As duas falhas irreversíveis

Todas as demais falhas produzem atraso ou indisponibilidade. Apenas duas produzem dano
que não se recupera:

| Falha irreversível | Por que é irreversível | Invariante que a impede |
|--------------------|------------------------|-------------------------|
| **Perder um registro** | O fato ocorreu e não há como reconstruí-lo; e ninguém percebe. | I9 (confirmação só após durabilidade) + I3 (lacuna detectada) |
| **Não detectar uma adulteração** | A trilha inteira perde valor probatório retroativamente. | I2 (cadeia) + selo assinado + cópia WORM |

Tudo em §3 e §5 existe para proteger essas duas.

## 3. FMEA

| # | Modo de falha | Efeito | Sev. | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---|---------------|--------|------|----------|-----------|-------------|--------------------|
| F-01 | PostgreSQL primário indisponível | Escrita recusada | Alta | Health | `503 AIOS-AUD-0005`; produtores retêm no outbox | Failover; produtores reenviam | Dedupe por `event_id` |
| F-02 | **Réplica síncrona** indisponível | Escrita recusada (RPO 0 em risco) | Alta | Health/readiness | Réplica de aplicação sai do balanceamento | Restaurar replicação; **não** relaxar `synchronous_commit` | Idempotente |
| F-03 | Consumidor JetStream atrasado | Fatos ainda não registrados | Média | `aios_aud_consumer_lag_s` | Janela de cegueira; **dado não perdido** (JetStream retém) | Escalar `audit-ingest`; drenar backlog | At-least-once |
| F-04 | Chave de assinatura indisponível | Selagem para | Média | `aud.seal.signing_failed` | **Ingestão continua**; registros ficam em `Received` | Restabelecer KMS; selagem **retroativa** do intervalo | Selagem idempotente |
| F-05 | MinIO (WORM) indisponível | Arquivamento atrasa | Média | `aios_aud_archive_lag_s` | Cadeia e selos seguem; verificação com 2 fontes | Retomar arquivamento; expurgo por retenção fica bloqueado até lá | Idempotente por intervalo |
| F-06 | **Lacuna de sequência** | Registro possivelmente perdido | **Crítica** | `CompletenessMonitor` | Partição afetada | Investigação P1 (RB-A-03); conclusão registrada como fato | — |
| F-07 | **Quebra de cadeia** | Integridade comprometida | **Crítica** | `IntegrityVerifier` | Escrita da partição congelada | Comparar com selo e WORM; acionar CISO (RB-A-01) | — |
| F-08 | Divergência banco × WORM | Provável adulteração | **Crítica** | Verificação completa | Partição afetada | RB-A-02; a evidência íntegra é a do WORM | — |
| F-09 | *Trigger* de imutabilidade removido | Camada 2 da defesa perdida | **Crítica** | Verificação de *startup* + diária | — | Reinstalar; investigar quem removeu; a cadeia e o WORM ainda protegem | — |
| F-10 | Verificação parada | Adulteração passaria despercebida | Alta | `aios_aud_verify_staleness_s` | — | Restabelecer `audit-verifier`; verificação completa ao voltar | Idempotente |
| F-11 | PDP (022) indisponível | Consulta e governança negadas | Média | Timeout/CB | **Ingestão não é afetada** | Retry após meia-abertura | Retriable |
| F-12 | Classe auditável silenciosa | Fatos não estão sendo registrados | Alta | `class_volume_ratio` | Classe específica | Investigar o produtor (RB-A-07) | — |
| F-13 | *Hold* esquecido em aberto | Dado pessoal preservado sem base atual | Média | Revisão periódica (90 d) | — | Revisão com jurídico; liberação registrada | — |
| F-14 | Apagamento criptográfico falho | Payload ainda legível | Alta | Verificação pós-apagamento | Titular específico | Reexecutar destruição; **P1 de privacidade** | Idempotente |
| F-15 | Chave de titular destruída por engano | Registro ilegível para sempre | Alta | Comprovante inesperado | Titular específico | **Irrecuperável** — mitigação é preventiva: PEP + capability própria + *hold* | — |
| F-16 | Serialização canônica alterada sem nova RFC | Toda a verificação histórica falha | **Crítica** | `worm_mismatch` em massa | — | Reverter o deploy; a serialização é contrato (RFC-0250) | — |
| F-17 | Exportação com escopo excessivo | Vazamento de evidência | Alta | Revisão na aprovação dupla | Exportação recusada | Reemitir com escopo correto; a tentativa fica registrada | — |

## 4. Retries, backoff e DLQ

| Operação | Retry | Backoff | DLQ |
|----------|-------|---------|-----|
| Ingestão por evento | JetStream reentrega até `ACK` | Política do stream | `AUDIT_DLQ` após 10 tentativas |
| Ingestão por API | **Do produtor**, com o mesmo `Idempotency-Key` | Exponencial com jitter | Outbox do produtor |
| Selagem | Sim | Exponencial (1 s → 60 s) | Alerta após `seal.interval_s × 3` |
| Arquivamento WORM | Sim | Exponencial | Alerta; expurgo bloqueado até resolver |
| Publicação de outbox | Sim | Exponencial (200 ms → 30 s) | `AUDIT_DLQ` |
| Verificação | Reagendada no próximo ciclo | — | — |
| Apagamento criptográfico | Sim, até `aud.erasure.max_latency_h` | Exponencial | Alerta de conformidade |

> **O 025 nunca "desiste" de um registro.** Uma mensagem em `AUDIT_DLQ` significa que
> um fato ocorrido **não está na trilha** — é tratada como incidente, não como
> mensagem descartável. O reprocessamento da DLQ é manual e deixa registro próprio.

## 5. Degradação graciosa

Ordem de sacrifício, do primeiro ao último:

```
   1. Exportação                ← auditor espera
   2. Consulta                  ← investigação espera
   3. Verificação contínua      ← janela sem detecção de adulteração
   4. Arquivamento WORM         ← 3ª fonte temporariamente indisponível
   5. Selagem                   ← registros ficam em Received, ainda encadeados
   6. INGESTÃO DURÁVEL          ← nunca; o fato seria perdido para sempre
```

Um sistema que não sela **pode selar depois** — o intervalo é recuperado
retroativamente e o `prev_hash` já detecta alteração local nesse meio-tempo. Um sistema
que não **ingere** perde o fato de forma definitiva e silenciosa.

Comparação com os módulos vizinhos, que fazem escolhas opostas pela mesma lógica:

| Módulo | Primeiro a cair | Último a cair | Por quê |
|--------|-----------------|---------------|---------|
| `../022-Policy/` | Simulação e testes | **Avaliação de decisão** | Sem decisão, nada executa |
| `../024-Observability/` | Traces amostrados | **Caminho de alerta** | Sem alerta, ninguém sabe que há problema |
| **025-Audit** | Exportação | **Ingestão durável** | Sem ingestão, o fato desaparece |

## 6. Recuperação de cada camada de integridade

| Camada rompida | Ainda protege | Recuperação |
|----------------|---------------|-------------|
| Interface (S1) | *Trigger*, cadeia, selo, WORM | Corrigir o código; investigar como a rota surgiu |
| *Trigger* (F-09) | Cadeia, selo, WORM | Reinstalar; verificação completa da janela sem *trigger* |
| Cadeia (F-07) | Selo assinado, WORM | Comparação com as duas fontes determina o escopo real |
| Selo (chave comprometida) | WORM, cadeia | Rotação no `../021-Security/`; **selos antigos permanecem verificáveis** pela chave pública histórica |
| WORM (bucket comprometido) | Cadeia, selo assinado | *Object lock* em modo *compliance* torna isso improvável; se ocorrer, o selo ainda prova |

Nenhuma camada isolada é suficiente; **nenhuma isolada é fatal**. Essa é a definição
prática de defesa em profundidade (`./Security.md` §3).

## 7. RTO / RPO

| Cenário | RTO | RPO | Procedimento |
|---------|-----|-----|--------------|
| Perda de réplica de aplicação | ~0 (balanceador) | **0** | Réplicas restantes atendem |
| Perda total do `audit-ingest` | ≤ 15 min | **0** | Produtores retêm no outbox; JetStream retém |
| Perda do PostgreSQL primário | ≤ 15 min | **0** | Failover para a réplica síncrona (já aplicada) |
| Perda do `audit-sealer` | ≤ 15 min | 0 | Selagem retroativa ao voltar |
| Perda do MinIO | ≤ 60 min | 0 | Arquivamento retomado; expurgo suspenso enquanto durar |
| Perda de região | ≤ 15 min | **0** | `../027-Cluster/`; réplica síncrona cross-AZ; WORM replicado |

**RPO = 0** é a meta que define o módulo (NFR-006) e só é possível pela combinação de
três decisões: confirmação após durabilidade (I9), replicação síncrona
(`aud.db.synchronous_commit = remote_apply`) e outbox nos produtores. Remover qualquer
uma delas transforma o RPO 0 em promessa sem lastro.

## 8. Testes de caos

| Experimento | Hipótese | Critério de sucesso |
|-------------|----------|---------------------|
| Matar o primário no meio de 10.000 escritas | Nenhum registro confirmado se perde | 100% dos `201` recebidos estão na trilha após failover; `seq` sem lacuna |
| Derrubar a réplica síncrona | Escrita é recusada, não aceita | `503 AIOS-AUD-0005`; **0** registros gravados sem replicação |
| Derrubar o KMS por 30 min | Ingestão continua; selagem adia | 0 perda; selagem retroativa cobre o intervalo |
| Alterar um registro direto no banco | *Trigger* impede | Exceção `AIOS-AUD-0016`; nenhuma linha alterada |
| Remover o *trigger* e alterar | Cadeia e WORM detectam | `chain_break` em ≤ 1 ciclo; WORM diverge do banco |
| Adulterar a cópia WORM | *Object lock* impede | Sobrescrita rejeitada pelo armazenamento |
| Remover 3 registros de uma partição | Lacuna detectada | `sequence_gap` em ≤ 5 min |
| Parar um produtor por 24 h | Classe silenciosa detectada | `class_silent` em ≤ 30 min |
| Solicitar RTBF sob *legal hold* | Recusa registrada | `423 AIOS-AUD-0008`; **0** payloads apagados |
| Verificar cadeia após crypto-shred | Cadeia continua íntegra | 100% de verificação passa com registros `Shredded` |

Os dois últimos experimentos verificam as invariantes I5 e I4 — as que sustentam a
conciliação entre imutabilidade e privacidade. Se algum falhar, o desenho de
`./Security.md` §7 não se sustenta.

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §9
- Alertas e runbooks: `./Monitoring.md` §4 e §5
- FSM e invariantes: `./StateMachine.md` §5
- Defesa em profundidade: `./Security.md` §3
- Implantação e readiness: `./Deployment.md` §4 · Escala: `./Scalability.md`

*Fim de `FailureRecovery.md`.*
