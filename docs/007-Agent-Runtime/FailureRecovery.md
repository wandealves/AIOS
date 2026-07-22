---
Documento: FailureRecovery
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0074, ADR-0076, ADR-0077, ADR-0079
RFCs relacionados: RFC-0001
Depende de: ./_DESIGN_BRIEF.md, ./StateMachine.md, ./Monitoring.md
---

# 007-Agent-Runtime — Failure Recovery

> FMEA (*Failure Mode and Effects Analysis*) completo do Agent Runtime,
> elaborando `./_DESIGN_BRIEF.md` §9. Idempotência conforme RFC-0001 §5.5;
> retries com backoff exponencial + *jitter*; DLQ em JetStream para eventos
> não publicáveis após `events.outbox.max_retries`. Nenhum RTO/RPO abaixo
> diverge do brief/`./NonFunctionalRequirements.md`.

## Índice

1. Princípio de degradação graciosa
2. FMEA — modos de falha, detecção e recuperação
3. Detalhamento por modo de falha crítico
4. DLQ e eventos não publicáveis
5. RTO/RPO — resumo consolidado
6. Referências

---

## 1. Princípio de Degradação Graciosa

Conforme Princípio 7 de `./Vision.md` §5: diante de falha recuperável, o
runtime **DEVE** preferir fallback/replanejamento a abortar a sessão —
falhar é o último recurso, não o primeiro. A ordem de preferência de
resposta a qualquer falha é:

```
1. Retry local (backoff + jitter)         — mais barato, mais rápido
2. Fallback (modelo alternativo, contexto reduzido)
3. Replanejamento (012-Planning)           — muda a abordagem, não o objetivo
4. Suspensão com checkpoint (suspended)     — preserva progresso, libera recurso
5. Encerramento (failed)                    — último recurso
```

---

## 2. FMEA — Modos de Falha, Detecção e Recuperação

| Modo de falha | Detecção | Recuperação | Idempotência/Retry | RTO/RPO |
|-----------------|----------|-------------|------------------------|-----------|
| Falha de boot do sandbox | `RuntimeBootstrapper` retorna erro | Marca instância `dead`; Supervisor recria; `AIOS-RUNTIME-0002`. | `Boot` idempotente por `Idempotency-Key`. | RTO ≤ 250 ms (novo slot quente). |
| Timeout/erro de tool (`015`) | Timeout/erro do `ToolInvoker` | Retry (até `tool.retry.max_attempts`); *circuit breaker*; fallback/replan. | Invocação idempotente por `idempotency_key`. | RPO 0 (passo re-executável). |
| Falha do Model Router (`017`) | Erro/timeout do `ModelRouterClient` | Fallback de modelo (se `model.fallback_enabled`); retry; degradar contexto. | Chamada idempotente; dedup por `model_call_id`. | RPO 0. |
| Cota esgotada | `QuotaGovernor` | Suspende (checkpoint) ou aborta; `quota_exceeded`/`AIOS-RUNTIME-0004`. | — | Retomável via `Resume`. |
| Loop travado/deadlock | Watchdog (`watchdog.stuck_timeout_ms`) | Interrompe passo; tenta replan; se falhar, `kill`; `AIOS-RUNTIME-0008`. | — | RTO ≤ 30 s (resume de checkpoint). |
| Crash do processo runtime | Heartbeat perdido | Supervisor recria; sessão retomada do último checkpoint. | Outbox re-publica eventos pendentes (dedup por `event.id`). | RTO ≤ 30 s; RPO ≤ 1 passo. |
| Violação de sandbox | seccomp/monitor | `kill` imediato; `sandbox_violation` (auditoria `025`); `AIOS-RUNTIME-0015`. | — | Sessão isolada; sem contágio. |
| Perda de conexão NATS | Erro do `EventPublisher` | Bufferiza no outbox local; reconecta; republica. | Dedup por `event.id`. | RPO 0 (outbox durável efêmero, enquanto o processo vive). |
| Checkpoint corrompido | Checksum inválido | Rejeita `resume`; retoma do checkpoint anterior íntegro; `AIOS-RUNTIME-0010`. | Checksums versionados. | RPO ≤ intervalo de checkpoint (`checkpoint.interval_steps`). |
| Falha de `010`/`011` (memória/contexto) | Erro do cliente | Retry; degradar para contexto mínimo; se persistir, falhar sessão. | Leituras idempotentes. | RPO 0. |
| PDP indisponível (`022-Policy`) | Erro/timeout do `CapabilityEnforcer` | Postura *fail-closed*: nega ação não cacheada (`AIOS-RUNTIME-0003`). | — | Sem impacto em ações já autorizadas em cache. |
| *Lease* Redis indisponível | Erro do `CheckpointManager`/`SupervisorChannel` ao adquirir lease | `Resume` recusado até Redis voltar (prevenção de dupla execução > disponibilidade). | — | RTO estendido até recuperação de Redis (aceito como *trade-off* de segurança). |
| MinIO indisponível (blob de checkpoint) | Erro do `CheckpointManager` na gravação do blob | `Suspend` falha com `AIOS-RUNTIME-0010`; sessão permanece ativa até novo `flush_interval` ou é forçada a `terminating` se cota crítica. | Retry de gravação com backoff. | RPO ≤ intervalo de checkpoint anterior bem-sucedido. |

---

## 3. Detalhamento por Modo de Falha Crítico

### 3.1 Falha de Boot do Sandbox

- **Causas comuns:** perfil `seccomp` malformado, `cgroups v2` indisponível
  no nó, falha de criação de namespace (limite de recurso do kernel do
  host).
- **Isolamento:** a falha afeta **apenas** a instância em boot; não impacta
  instâncias `busy`/`idle` já ativas no mesmo nó.
- **Ação operacional:** se a taxa de `AIOS-RUNTIME-0002` subir
  significativamente após um rollout, seguir `./Deployment.md` §5 (parar
  substituição, reverter para versão anterior de imagem).

### 3.2 Crash do Processo Runtime

- **Fluxo completo:** ver `./SequenceDiagrams.md` §6 e `./UseCases.md`
  UC-008.
- **Garantia central:** o *lease* Redis (ADR-0079) garante que, mesmo que o
  processo original "ressuscite" tardiamente, ele não consegue retomar uma
  sessão já assumida por outra instância — elimina o risco de dupla
  execução (RSK-A05 de `./Vision.md` §8).
- **Perda de dados aceitável:** no máximo 1 passo do loop desde o último
  checkpoint confirmado (RPO ≤ 1 passo, NFR-006); o intervalo de checkpoint
  (`checkpoint.interval_steps`, default 5) é o parâmetro de ajuste direto
  desse risco — reduzir o intervalo diminui a janela de perda ao custo de
  maior overhead de I/O em MinIO.

### 3.3 Violação de Sandbox

- **Sem caminho de recuperação automática por design** — toda violação é
  tratada como incidente de segurança, nunca como falha transitória a ser
  reintentada (ver `./Security.md` §3, categoria *Elevation of Privilege*).
- **Efeito de contenção:** a sessão comprometida é encerrada; a instância
  de runtime correspondente **nunca** retorna ao pool quente — é destruída
  pelo Supervisor, eliminando qualquer estado residual potencialmente
  comprometido.

### 3.4 Esgotamento de Cota com Falha de Checkpoint

- Cenário composto: `QuotaGovernor` sinaliza esgotamento **e**
  `CheckpointManager` falha ao gravar o snapshot (ex.: MinIO indisponível
  no momento exato).
- **Recuperação:** a sessão não pode transicionar a `suspended` sem
  checkpoint íntegro (guarda **G8**); nesse caso, o runtime tenta uma
  retentativa de checkpoint dentro de um orçamento curto de tempo adicional
  (não contabilizado contra `budget.max_cost_usd`, apenas contra um
  temporizador de encerramento gracioso); se persistir, a sessão é forçada
  a `terminating→failed` com `AIOS-RUNTIME-0010`, preservando o que já foi
  registrado em `ReActStep`/`ToolInvocation`/`ModelCall` até aquele ponto
  (a trilha de auditoria não se perde, apenas a possibilidade de retomada).

---

## 4. DLQ e Eventos Não Publicáveis

- Eventos que excedem `events.outbox.max_retries` (default 10) sem
  confirmação de publicação são movidos, pelo relay do outbox, para o
  padrão de **Dead Letter Queue** do JetStream (`../020-Communication/`),
  preservando o payload original e o motivo da falha (ex.: subject
  inexistente, tenant sem conta NATS provisionada).
- Um alerta (`RuntimeOutboxBacklog`, `./Monitoring.md` §3) é disparado
  antes de o evento atingir o limite de tentativas, permitindo intervenção
  operacional (ex.: restaurar conectividade) antes da perda efetiva do
  evento.
- Eventos em DLQ **NÃO** representam perda de estado do agente (o estado já
  foi aplicado localmente e refletido em `ReActStep`/checkpoint durável via
  chamada gRPC síncrona a `010`/`025`, quando aplicável) — representam
  apenas atraso/perda de **notificação** a consumidores assíncronos, um
  risco de observabilidade, não de correção funcional.

---

## 5. RTO/RPO — Resumo Consolidado

| Cenário | RTO | RPO |
|---------|-----|-----|
| Boot falho (nova tentativa) | ≤ 250 ms (novo slot quente) | N/A (sessão nunca chegou a existir) |
| Crash de processo com checkpoint recente | ≤ 30 s | ≤ 1 passo |
| Falha de dependência transitória (tool/modelo) | Segundos (retry/backoff) | 0 |
| Violação de sandbox | N/A (sem retomada da sessão comprometida) | N/A |
| Indisponibilidade de MinIO durante `Suspend` | Até a dependência voltar | ≤ intervalo do último checkpoint bem-sucedido |

**Alinhamento com metas globais.** RTO ≤ 15 min / RPO ≤ 5 min do plano de
controle (V-R10 da Visão Global) são **limites superiores**; este módulo,
por ser efêmero e checkpointado, mira metas mais agressivas — todas as
linhas acima são **iguais ou melhores** que o limite global, nunca piores
(`_DESIGN_BRIEF.md` §9).

---

## 6. Referências

- FMEA original (fonte): `./_DESIGN_BRIEF.md` §9.
- Transições de estado associadas a cada falha: `./StateMachine.md`.
- Fluxos de sequência de falha: `./SequenceDiagrams.md` §5–§9.
- Alertas correspondentes: `./Monitoring.md` §3.
- Testes de chaos que exercitam cada modo: `./Testing.md` §6.
- Modelo de escala e *lease* distribuído: `./Scalability.md`.
