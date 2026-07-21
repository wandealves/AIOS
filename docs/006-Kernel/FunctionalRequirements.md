---
Documento: FunctionalRequirements
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `Requirements.md`, `../003-RFC/RFC-0001-Architecture-Baseline.md`
---

# 006-Kernel — Functional Requirements

> Convenção de prioridade: **MoSCoW** (`Must` / `Should` / `Could` / `Won't-this-release`).
> Todo FR é mensurável e verificável por um critério de aceite objetivo. A
> numeração é global e sequencial; não reutilize um ID mesmo se um FR for
> depreciado (marque como `Deprecated` no lugar). Origem aponta para a seção
> do `_DESIGN_BRIEF.md` de onde o requisito deriva.

## A. ABI de Syscalls Cognitivas

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-001 | O Kernel DEVE expor os 11 verbos de syscall cognitiva (`spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`, `invoke_tool`, `route_model`, `get_quota`, `checkpoint`) via REST (`/v1/kernel/...`) e via gRPC (pacote `aios.kernel.v1`), com paridade funcional entre os dois protocolos. | Must | Contrato OpenAPI e `.proto` publicados em `API.md`; suíte de contract test cobre 100% dos verbos em ambos os protocolos; divergência de comportamento entre REST e gRPC para o mesmo verbo é tratada como bug bloqueante. | Brief §5.1 |
| FR-002 | O Kernel DEVE versionar a ABI por caminho REST (`/v1/...`), header `X-AIOS-Api-Version` e pacote gRPC (`aios.kernel.v1`), suportando coexistência de ≥ 2 versões *major* simultâneas conforme RFC-0001 §5.7. Verbo desconhecido ou versão de ABI não suportada DEVE ser rejeitado. | Must | Requisição a verbo inexistente ou versão não suportada retorna `AIOS-SYSCALL-0001` (HTTP 400); teste de compatibilidade confirma que uma versão `N-1` continua funcional após introdução de `N`. | Brief §5.2, §10 (ADR-0060) |
| FR-003 | O `SyscallGateway` DEVE validar o schema e a versão do payload de toda syscall antes de despachar ao handler, rejeitando payloads malformados sem consumir cota ou disparar efeitos colaterais. | Must | Payload inválido (campo obrigatório ausente, tipo incorreto) retorna `AIOS-KERNEL-0010` (HTTP 400) sem escrita em `syscall_log` de execução nem consumo de cota; apenas registro de rejeição é auditado. | Brief §2.1 (SyscallGateway) |

## B. Governança (PEP ↔ PDP)

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-004 | Toda syscall privilegiada DEVE passar pelo `CapabilityEnforcer` (PEP), que monta o `DecisionRequest` (sujeito, ação, recurso, ambiente) e consulta o PDP do `022-Policy` via `PolicyClient` **antes** de qualquer execução ou efeito colateral. | Must | Teste: syscall sem capability concedida retorna `AIOS-CAP-0001` (HTTP 403) e nenhum estado do ACB, cota ou recurso a jusante é alterado; evento `agent.syscall.denied` é emitido e auditado. | Brief §1.2 (R-03), §2.1 (CapabilityEnforcer) |
| FR-005 | Quando o PDP estiver indisponível, o Kernel DEVE aplicar o modo configurado em `kernel.pep.fail_mode`; com `fail_mode=closed` (default), toda syscall privilegiada DEVE ser negada (*default deny*). | Must | Com PDP inacessível e `fail_mode=closed`, toda syscall privilegiada retorna `AIOS-CAP-0003` (HTTP 503, `retriable=true`); nenhuma syscall privilegiada é executada nesse período. | Brief §8, §9, §12.1 |
| FR-006 | O `PolicyClient` DEVERIA cachear decisões de capability por até `kernel.pep.decision_cache_ttl_ms` e DEVE invalidar o cache imediatamente ao consumir `aios.<tenant>.policy.bundle.updated`. | Should | Decisão repetida dentro do TTL não gera nova chamada ao PDP (medido por contador de chamadas); após evento de atualização de bundle, a primeira decisão subsequente sempre consulta o PDP (cache miss forçado). | Brief §2.1 (PolicyClient), §6.2 |
| FR-007 | O Kernel NÃO DEVE aceitar `tenant` (em URN, subject, claim ou payload) divergente do `tenant` do contexto autenticado (`X-AIOS-Tenant` validado no Gateway). | Must | Requisição com `tenant` divergente retorna `AIOS-CAP-0002` (HTTP 403); evento `agent.syscall.denied` emitido com motivo `tenant_mismatch`; nenhum dado do tenant real é exposto na resposta de erro. | Brief §1.2 (R-10), §12.1 |

## C. ACB / Ciclo de Vida

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-008 | O Kernel DEVE manter o `AgentControlBlock` (ACB) como estrutura autoritativa por agente, com a FSM canônica (`Pending`, `Admitted`, `Running`, `Suspended`, `Hibernated`, `Terminating`, `Terminated`, `Failed`) e controle de concorrência otimista (`version`). | Must | Transição fora das arestas definidas na FSM (§4 do brief) retorna `AIOS-KERNEL-0002` (HTTP 409); mutação com `version` divergente retorna `AIOS-KERNEL-0009` (HTTP 409) e é reexecutada até `kernel.acb.optimistic_retry_max`. | Brief §3.1, §4 |
| FR-009 | O Kernel DEVE coordenar `spawn` como uma **saga** que solicita admissão ao `009-Scheduler` e, após admissão, materialização ao `008-Agent-Lifecycle`, com **compensação** automática (liberação de slot e de cota) em caso de falha em qualquer etapa. | Must | Falha de admissão (`AIOS-KERNEL-0003`) ou de boot (`AIOS-KERNEL-0004`) resulta em ACB → `Failed`, slot do Scheduler liberado e cota de `agent_count` devolvida — verificado por teste de injeção de falha em cada etapa da saga. | Brief §1.2 (R-05), §2.1 (LifecycleCoordinator), §4.2 (T-01..T-05) |
| FR-010 | O Kernel DEVE suportar `suspend` (voluntário via syscall e involuntário via preempção do Scheduler) e `resume` (retorno a `Running` a partir de `Suspended`), preservando `memory_ptr`, `context_ptr` e `capability_set` do ACB. | Must | Após `suspend`→`resume`, o ACB retorna a `Running` com os mesmos ponteiros de memória/contexto anteriores (comparação de igualdade); latência de `resume` a partir de `Suspended` medida por `aios_kernel_syscall_latency_ms{verb="resume"}`. | Brief §4.2 (T-06, T-07), §6.2 (preemption.requested) |
| FR-011 | O Kernel DEVERIA suportar hibernação automática de agentes ociosos (`Suspended`→`Hibernated`) após `kernel.hibernation.idle_ttl_ms`, e `resume` a partir de `Hibernated` restaurando o estado a partir do último `checkpoint_ref`. | Should | Agente sem atividade além do TTL configurado transita para `Hibernated` (verificável via `syscall_log`/evento); `resume` de um agente `Hibernated` restaura `memory_ptr`/`context_ptr` do checkpoint e materializa o runtime com p99 ≤ 250 ms (alinhado a NFR-001). | Brief §4.2 (T-08, T-09), §2.1 (CheckpointManager) |
| FR-012 | O Kernel DEVE suportar `kill` a partir de `Running`, `Suspended` ou `Hibernated`, transitando para `Terminating` → `Terminated`, executando drenagem, compensações pendentes e liberação de cotas de agente antes de marcar o estado como terminal. | Must | Após `kill`, o ACB atinge `Terminated` com `terminated_at` preenchido e cotas de agente liberadas (verificável via `get_quota` do tenant); evento `agent.lifecycle.terminated` emitido exatamente uma vez. | Brief §4.2 (T-10, T-11), invariante I2 |
| FR-013 | O Kernel DEVE suportar a syscall `checkpoint` a partir de `Running` ou `Suspended`, coletando ponteiros de memória/contexto do ACB, disparando snapshot durável via `008-Agent-Lifecycle`→MinIO e atualizando `checkpoint_ref` no ACB. | Should | Após `checkpoint` bem-sucedido, `checkpoint_ref` do ACB aponta para um snapshot recuperável; um `resume` subsequente a partir desse checkpoint restaura estado equivalente (teste de round-trip). | Brief §2.1 (CheckpointManager), §4.2 (T-13) |
| FR-014 | O Kernel DEVE suportar a criação de sub-agentes via `spawn` aninhado, registrando `parent_urn` no ACB filho e aplicando limite de *fan-out* configurável para conter custo de coordenação O(N²). | Should | `spawn` com `parent_urn` cria árvore rastreável (consulta recursiva por `parent_urn`); exceder o limite de fan-out configurado retorna erro de cota (`AIOS-QUOTA-0001`/`AIOS-QUOTA-0003`, conforme a cota configurada) em vez de criar o sub-agente. | Brief §3.1 (`parent_urn`), §3.5, §10 |

## D. Cotas de Recurso

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-015 | O `ResourceQuotaManager` DEVE reservar, consumir e liberar cotas (tokens, cpu-ms, memória, custo USD, ferramentas, syscalls/s) de forma **atômica** por agente e por tenant, usando token-bucket em Redis (script Lua). | Must | Teste de concorrência com N requisições simultâneas próximas ao limite não produz *double-spend*; overshoot medido ≤ 1% (NFR-009). | Brief §1.2 (R-04), §2.1 (ResourceQuotaManager), §10 |
| FR-016 | O Kernel DEVE expor `get_quota`, retornando o saldo atual (consumido/limite/restante) de tokens, custo, ferramentas e syscalls/s do agente e do tenant, coerente com os contadores quentes do Redis. | Must | Valor retornado por `get_quota` diverge do contador real do Redis em, no máximo, uma janela de reconciliação; teste de integração compara ambos após rajada de consumo. | Brief §5.1 (`get_quota`), §7.1 (FR-010 original) |
| FR-017 | O Kernel DEVE suportar dois modos de *enforcement* de cota por registro de `Quota`: `hard` (rejeita a syscall excedente) e `soft` (permite a execução, marca e alerta). | Must | Cota `hard` excedida retorna `AIOS-QUOTA-0001`/`0002`/`0003` conforme o recurso e HTTP 429; cota `soft` excedida permite a execução e emite `agent.quota.warning` com `enforcement=soft` no payload. | Brief §3.2 (`enforcement`), §6.1 |
| FR-018 | O Kernel DEVE emitir o evento `agent.quota.warning` ao consumo atingir um limiar configurável (ex.: 80%) do limite aplicável, antes de atingir o limite `hard`. | Should | Ao ultrapassar o limiar configurado, evento é publicado com o percentual de consumo no payload; teste verifica que o evento não é reemitido a cada requisição subsequente dentro da mesma janela (deduplicação por janela). | Brief §6.1 (`agent.quota.warning`) |

## E. Idempotência e Eventos

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-019 | Toda syscall de mutação DEVE aceitar o cabeçalho `Idempotency-Key` e o Kernel DEVE persistir o resultado associado por, no mínimo, `kernel.idempotency.retention_h` (≥ 24h), retornando o mesmo resultado em repetições. | Must | Repetir a mesma requisição com a mesma `Idempotency-Key` e o mesmo payload retorna byte-a-byte o mesmo corpo de resposta e HTTP status, sem novo efeito colateral (sem novo evento, sem novo consumo de cota). | Brief §1.2 (R-07), RFC-0001 §5.5 |
| FR-020 | O Kernel DEVE publicar todo evento do catálogo (`Events.md`) via **Outbox transacional**, garantindo atomicidade entre a persistência do estado e a publicação no NATS/JetStream. | Must | Teste de *crash injection* entre o commit da transação e a publicação não resulta em evento perdido (o relay do Outbox reenvia); nenhum evento é publicado sem a mutação correspondente ter sido commitada (sem "evento fantasma"). | Brief §1.2 (R-06, R-07), §2.1 (EventEmitter) |
| FR-021 | O Kernel DEVE detectar e rejeitar o reuso de uma `Idempotency-Key` já registrada para o mesmo tenant com um payload de requisição divergente do original. | Must | Reenvio da mesma chave com payload diferente retorna `AIOS-SYSCALL-0002` (HTTP 409); o resultado original permanece inalterado. | Brief §5.2 (`AIOS-SYSCALL-0002`) |

## F. Brokering de Recursos

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-022 | O `ResourceBrokerRouter` DEVE encaminhar as syscalls de recurso (`remember`→`010`, `recall`→`010`, `plan`→`012`, `invoke_tool`→`015`, `route_model`→`017`) ao módulo dono **somente após** aprovação de capability (PEP) e reserva de cota bem-sucedida, sem executar a lógica de negócio do módulo dono. | Must | Teste de integração confirma que nenhuma chamada é feita ao módulo dono antes de PEP=`allow` e cota reservada; teste de contrato confirma que o Kernel não duplica/reimplementa lógica de memória, planejamento, ferramenta ou roteamento de modelo. | Brief §1.2 (R-08, NR-05), §2.1 (ResourceBrokerRouter) |
| FR-023 | O `ResourceBrokerRouter` DEVE aplicar *circuit breaker* e *bulkhead* independentes por dependência (`010`, `011`, `012`, `015`, `017`), isolando falhas de um broker das demais syscalls. | Should | Injeção de falha em um broker (ex.: `015-Tool-Manager` fora do ar) resulta em `AIOS-KERNEL-0005` apenas para syscalls que dependem dele, sem degradar `spawn`, `suspend`, `get_quota` ou brokers saudáveis; breaker abre conforme `kernel.broker.circuit_breaker_threshold`. | Brief §2.1 (ResourceBrokerRouter), §9 |

## G. Auditoria e Observabilidade

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-024 | O Kernel DEVE registrar 100% das syscalls (permitidas e negadas) em `kernel.syscall_log` e emitir o correspondente registro de auditoria imutável via `025-Audit`. | Must | Auditoria de cobertura confirma que toda linha em `syscall_log` tem um registro de auditoria correlato por `trace_id`; nenhuma syscall privilegiada é executada sem entrada correspondente. | Brief §1.2 (R-09), §3.3 |
| FR-025 | O Kernel DEVE instrumentar toda syscall com telemetria OpenTelemetry (spans distribuídos, métricas `aios_kernel_*`, logs estruturados correlacionados por `trace_id`/`tenant_id`/`agent_urn`). | Must | Inspeção de spans confirma presença de atributos obrigatórios (`tenant_id`, `agent_urn`, `verb`, `decision`) em 100% das amostras; dashboards de `Monitoring.md` consomem essas métricas sem transformação adicional. | Brief §1.2 (R-09), §2.1 (KernelTelemetry) |

## H. Administração

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-026 | O Kernel DEVE expor a leitura administrativa do ACB (`GetAcb` / `GET /v1/kernel/agents/{urn}`) respeitando capability e isolamento de tenant, sem exigir consulta ao PDP para leituras não-privilegiadas do próprio tenant quando assim configurado. | Must | `GetAcb` retorna 100% dos campos públicos do ACB (exceto internos de sharding); acesso cross-tenant retorna `AIOS-CAP-0002`; leitura é auditada como ação de baixo risco conforme política vigente. | Brief §5.1 (`GetAcb`) |

## Notas de rastreabilidade

- A relação completa `FR-NNN → UC-NNN → Teste` está em `Requirements.md` §5.
- Nenhum FR deste documento contradiz o `_DESIGN_BRIEF.md`; FR-013..FR-026 são
  **elaborações** dos componentes e seções §1–§10 do brief, não requisitos
  novos fora de escopo.
- Alterações a este catálogo (adição, depreciação) DEVEM ser acompanhadas de
  atualização da matriz de rastreabilidade em `Requirements.md`.
