---
Documento: Security
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0071, ADR-0078
RFCs relacionados: RFC-0001, RFC-0071
Depende de: ../021-Security/, ../022-Policy/, ../025-Audit/, ../003-RFC/RFC-0001
---

# 007-Agent-Runtime — Security

> Este documento detalha o modelo de segurança do Agent Runtime: AuthN/AuthZ,
> superfície de ataque, *threat model* STRIDE, gestão de segredos, TLS/mTLS,
> isolamento de sandbox e conformidade LGPD/GDPR. É a elaboração completa de
> `./_DESIGN_BRIEF.md` §12 — nenhuma decisão aqui contradiz o brief; onde o
> brief já normatiza, este documento cita a seção e detalha o controle
> operacional correspondente.

## Índice

1. AuthN/AuthZ
2. Superfície de ataque
3. Threat model (STRIDE)
4. Gestão de segredos
5. TLS/mTLS
6. Isolamento de sandbox — controles detalhados
7. LGPD/GDPR
8. Controles de segurança — resumo executivo
9. Referências

---

## 1. AuthN/AuthZ

### 1.1 Autenticação entre serviços

- Toda comunicação **DEVE** usar **mTLS** interno (`021-Security`,
  RFC-0001 §6): o runtime autentica o Runtime Supervisor e os serviços do
  control plane por certificado, e é autenticado por eles da mesma forma.
- Não há autenticação de usuário final neste módulo — nenhum endpoint é
  alcançável fora da rede de controle interna (ver `./Deployment.md` §6).
- Certificados são rotacionados por `021-Security` sem exigir *restart* do
  processo runtime quando o mecanismo de *hot reload* de TLS estiver
  disponível na *stack* mTLS padrão do AIOS.

### 1.2 Autorização — PEP local (`CapabilityEnforcer`)

- O runtime é **PEP** (Policy Enforcement Point), nunca PDP. Toda ação
  privilegiada (invocação de tool, egress de rede, syscall cognitiva)
  **DEVE** ter uma *capability token* válida, concedida no boot (a partir do
  `AgentSpec`) ou confirmada dinamicamente pelo PDP via Kernel/`022-Policy`.
- Postura **default deny** (Princípio 3, `./Vision.md` §5; ADR-0078): na
  ausência de decisão (cache-miss e PDP indisponível), a ação **É NEGADA**,
  nunca permitida por omissão.
- O `CapabilityEnforcer` cacheia decisões com TTL curto
  (`runtime.pep.decision_cache_ttl_ms`, ver `./Configuration.md`) e invalida
  imediatamente ao consumir `aios.<t>.policy.decision.updated`.

### 1.3 Isolamento de tenant

- `tenant_id` em URNs/subjects/claims é fronteira de isolamento; o runtime
  **NÃO DEVE** aceitar `tenant` divergente do contexto autenticado
  (RFC-0001 §6; ver `./UseCases.md` UC-017).
- Toda comparação de tenant ocorre **antes** de qualquer leitura/mutação —
  nunca depois, evitando janelas de exposição parcial.

---

## 2. Superfície de Ataque

| Vetor | Descrição | Onde é mitigado |
|-------|-----------|--------------------|
| Prompt injection / raciocínio adversarial | O agente é induzido, via conteúdo de contexto/ferramenta, a tentar ações fora do seu escopo autorizado. | `CapabilityEnforcer` (PEP, default deny) — mesmo que o *raciocínio* seja comprometido, a **execução** de qualquer efeito real exige capability válida. |
| Escape de sandbox | Syscall fora da allowlist, escrita fora do FS efêmero, acesso a namespace de outro processo. | `SandboxManager` (`seccomp-bpf`, `cgroups v2`, namespaces, FS read-only+tmpfs). |
| Egress não autorizado | Tentativa de conexão de rede a destino fora da allowlist do tenant/agente. | `sandbox.egress_allowlist` (*default deny* de rede). |
| Exaustão de recursos (DoS local) | Loop com fan-out/recursão descontrolada, consumo excessivo de tokens/custo. | `QuotaGovernor` (`loop.max_steps`, `loop.max_recursion_depth`, `loop.max_tool_fanout`, `budget.*`). |
| Vazamento de PII em telemetria | `thought`/`observation` carregam dado pessoal e são publicados sem redação. | `pii.redaction.enabled` (FR-015), metadados de sensibilidade de `010-Memory`. |
| Reuso indevido de `Idempotency-Key` | Chave reaproveitada com payload divergente para forçar efeito inesperado. | Detecção e rejeição (`AIOS-RUNTIME-0007`). |
| Dupla execução / *split-brain* | Duas instâncias tentando operar a mesma sessão simultaneamente. | *Lease* distribuído Redis por `session_id` (ADR-0079). |
| Checkpoint adulterado | Blob de working memory alterado fora do fluxo normal. | Checksum sha256 obrigatório antes de qualquer `restore()` (I5, `./ClassDiagrams.md`). |
| Man-in-the-middle interno | Interceptação de tráfego entre runtime e serviços de recurso. | mTLS obrigatório em toda comunicação interna. |

---

## 3. Threat Model (STRIDE)

| Categoria | Ameaça | Mitigação | Referência |
|-----------|--------|-----------|-------------|
| **S**poofing | Runtime ou Supervisor falso se apresentando na rede interna. | mTLS mútuo; validação de identidade de serviço; claims assinadas. | `_DESIGN_BRIEF.md` §12.1 |
| **T**ampering | Alteração de `AgentSpec`, `Checkpoint` ou payload de evento em trânsito/repouso. | Checksums (sha256) em checkpoints; envelope assinado/mTLS; outbox íntegro (SQLite/WAL local); FS read-only do sandbox. | `_DESIGN_BRIEF.md` §12.2 |
| **R**epudiation | Ação do agente sem trilha auditável. | Eventos imutáveis + `025-Audit`; *provenance* por `ReActStep`; correlação `traceparent`. | `_DESIGN_BRIEF.md` §12.2 |
| **I**nformation disclosure | Vazamento de PII/dado cross-tenant em logs/eventos/erros. | Redação de PII (`pii.redaction.enabled`); RLS por `tenant_id` no control plane; `detail` de erro sem dado sensível; egress *default deny*. | `_DESIGN_BRIEF.md` §12.2, §6 (RFC-0001) |
| **D**enial of service | Agente consome recursos ilimitados (*noisy neighbor*) ou satura um nó. | `cgroups v2` (CPU/RAM/PIDs); cotas locais (tokens/USD/passos/wall-clock); `max_sessions_per_node`; *circuit breaker* em dependências. | `_DESIGN_BRIEF.md` §10, §12.2 |
| **E**levation of privilege | Escape de sandbox, syscall proibida, acesso a rede não permitido, ação sem capability. | `seccomp-bpf` allowlist restritiva; namespaces; egress allowlist; kill em violação + evento `sandbox_violation`; PEP *default deny*; sem `CAP_*` desnecessários no processo. | `_DESIGN_BRIEF.md` §12.2 |

---

## 4. Gestão de Segredos

- Segredos (credenciais de mTLS, tokens de serviço) são **injetados** pelo
  `021-Security`, escopados por tenant, montados em **tmpfs** dentro do
  sandbox — nunca gravados em disco persistente, nunca *logados*.
- O processo runtime **NÃO DEVE** ler segredos de variáveis de ambiente em
  produção (usado apenas em desenvolvimento local, ver `./Configuration.md`
  §1); a via de produção é o volume tmpfs gerido por `021-Security`.
- Rotação de segredos ocorre sem reinício do processo quando o mecanismo de
  *hot reload* de credenciais está disponível; caso contrário, a rotação
  coincide com o ciclo natural de `Drain`/substituição de instância
  (`./Deployment.md` §5).
- Ferramentas invocadas via `015-Tool-Manager` **nunca** recebem segredos do
  runtime diretamente — a autenticação de cada tool com seu provedor externo
  é responsabilidade de `015-Tool-Manager`, mantendo o runtime alheio a
  credenciais de terceiros.

---

## 5. TLS/mTLS

| Canal | Protocolo | Observação |
|-------|-----------|--------------|
| Runtime ↔ Runtime Supervisor (`RuntimeControl`) | gRPC sobre mTLS | Comando (`Boot`/`Suspend`/`Resume`/`Kill`/`Drain`/`GetStatus`). |
| Runtime ↔ Control plane (`AgentExecution`) | gRPC sobre mTLS | `SubmitTask`/`CancelTask`/`StreamSteps`. |
| Runtime ↔ `010`/`011`/`012`/`015`/`017` | gRPC sobre mTLS | Todos os clientes de recurso (`MemoryClient`, `ContextClient`, `PlanningClient`, `ToolInvoker`, `ModelRouterClient`). |
| Runtime ↔ NATS/JetStream | mTLS + contas NATS por tenant | Heartbeat, eventos, comandos consumidos. |
| Runtime ↔ Redis | TLS (quando suportado pela infraestrutura) | *Lease* e cache de decisão. |
| Runtime ↔ MinIO (via control plane) | mTLS/TLS | Blobs de checkpoint — sempre via serviço do control plane, nunca acesso direto a credenciais de object storage pelo runtime. |

---

## 6. Isolamento de Sandbox — Controles Detalhados

| Controle | Mecanismo | Perfil default (`net-restricted`) |
|-----------|-----------|----------------------------------------|
| Isolamento de processo | Linux namespaces (pid/net/mnt/ipc/uts) | Aplicado a todo processo runtime, sem exceção. |
| Restrição de syscalls | `seccomp-bpf` (allowlist) | `seccomp_profile` do `SandboxProfile` (ver `./Database.md` §2, tabela de referência); violação → `kill` + `AIOS-RUNTIME-0015`. |
| Limites de recurso | `cgroups v2` (CPU/RAM/PIDs) | `sandbox.cpu_millicores=1000`, `sandbox.mem_mib=512`, `sandbox.pids_max=256`. |
| Filesystem | Read-only + tmpfs efêmero | `sandbox.fs_writable_mib=128`; nenhuma escrita persiste além da vida do processo. |
| Rede (egress) | *Default deny* + allowlist | `sandbox.egress_allowlist=[]` por padrão; qualquer destino fora da lista é bloqueado e reportado (`AIOS-RUNTIME-0011`). |
| Rede (ingress) | Nenhuma porta de entrada exposta a não ser os canais gRPC de controle (mTLS) | Superfície de ataque de rede mínima. |
| Capacidades Linux (`CAP_*`) | Nenhuma capacidade elevada concedida ao processo do agente | Princípio de menor privilégio; `CAP_SYS_ADMIN` etc. são exclusivas do processo de infraestrutura que **cria** o sandbox, não do agente em si. |

**Perfis alternativos** (`default`, `no-net`, `custom`) trocam apenas os
parâmetros acima, nunca o mecanismo — todo perfil aplica os cinco controles
da tabela. Perfis de isolamento máximo (ex.: microVM/gVisor) permanecem
candidatos por tenant regulado, avaliados e descartados como *default*
(overhead de boot incompatível com NFR-001) — ver `./Architecture.md` §11.

---

## 7. LGPD/GDPR

- **Minimização.** Payloads de evento/log aplicam redação/tokenização de PII
  (RFC-0001 §7); campos `thought`/`observation` de `ReActStep` são redigidos
  conforme metadados de sensibilidade de `010-Memory` antes de qualquer
  emissão externa (`pii.redaction.enabled`, FR-015).
- **Base legal e retenção.** Itens de memória com dados pessoais carregam
  metadados de base legal/retenção geridos por `010`/`025`; o runtime
  **respeita** esses metadados ao persistir working memory em checkpoint —
  não decide política de retenção por conta própria.
- **Direito ao esquecimento.** Uma operação de expurgo (coordenada com
  `010-Memory`/`025-Audit`) invalida checkpoints/working memory associados
  ao dado pessoal e emite evento auditável de expurgo — rastreável fim-a-fim
  (NFR-013).
- **Localidade de dados.** Perfis de sandbox **PODEM** restringir egress a
  regiões permitidas por tenant (residência de dados), via
  `sandbox.egress_allowlist` combinada com política de rede de
  `021-Security`.

---

## 8. Controles de Segurança — Resumo Executivo

| Controle | Meta/SLO associado | Verificação |
|-----------|-----------------------|----------------|
| Cobertura de PEP | 100% das ações privilegiadas (NFR-008) | Auditoria de cobertura cruzando ações com decisões do `CapabilityEnforcer`. |
| Isolamento de sandbox | 0 escapes confirmados (NFR-008) | *Pentest* periódico + teste de escape automatizado (`./Testing.md`). |
| Isolamento multi-tenant | 0 vazamentos cross-tenant (NFR-012) | Teste de isolamento dedicado (`./UseCases.md` UC-017). |
| Redação de PII | 100% dos payloads com `pii.redaction.enabled=true` (NFR-013) | Auditoria de amostragem de payloads publicados. |
| mTLS interno | 100% dos canais de comunicação | Revisão de configuração de rede/certificados (`021-Security`). |
| Integridade de checkpoint | 0 restaurações a partir de checksum inválido | Teste de round-trip de checkpoint (`./FailureRecovery.md`). |

---

## 9. Referências

- Decisões de segurança (fonte): `./_DESIGN_BRIEF.md` §12.
- Componentes de enforcement: `./Architecture.md` §5 (`SandboxManager`, `CapabilityEnforcer`).
- Modos de falha relacionados: `./FailureRecovery.md`.
- Fluxos de violação de sandbox e negação de capability: `./SequenceDiagrams.md` §7; `./UseCases.md` UC-010, UC-011.
- Contratos centrais de segurança: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6, §7.
- Módulos de segurança/política: `../021-Security/`, `../022-Policy/`, `../025-Audit/`.
- Protocolo de sandboxing MCP a propor: RFC-0071 (`./RFC.md`).
