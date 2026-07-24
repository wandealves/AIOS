---
Documento: Security
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability + CISO
ADRs relacionados: ADR-0008, ADR-0010, ADR-0242, ADR-0243, ADR-0246, ADR-0248, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §12, 021-Security, 022-Policy, 025-Audit, 005-Database
---

# 024-Observability — Segurança

> A telemetria é um alvo atraente por um motivo simples: ela **atravessa todos os
> módulos**. Um atacante com acesso irrestrito à consulta de logs e traces obtém um
> retrato do sistema inteiro — quem chamou o quê, quando, com que latência e com que
> resultado — sem precisar comprometer nenhum serviço específico.

## 1. Superfície de ataque

| Superfície | Exposição | Controle primário |
|-----------|-----------|-------------------|
| Ingestão OTLP (`:4317`/`:4318`) | Todos os módulos emissores | mTLS + coerência `aios.tenant` × certificado |
| Consulta (métricas/traces/logs) | SRE, donos de serviço, console | PEP + `TenantRouter` + orçamento de consulta |
| Administração (sinais, SLO, regras, retenção) | Plataforma e donos de serviço | PEP + capability por operação |
| Silêncios | SRE de plantão | Prazo obrigatório, justificativa, autor, trilha |
| Armazenamentos (Prometheus/Tempo/Seq) | Rede interna | mTLS, credenciais dinâmicas, sem acesso direto de usuário |
| Chave de tokenização de PII | `../021-Security/` | Em memória, nunca em disco; rotação programada |

## 2. AuthN / AuthZ

### 2.1 Autenticação

- **Emissores**: **mTLS** com certificado de workload emitido pelo `../021-Security/`
  após atestação (RFC-0001 §6). O 024 **não** autentica ninguém por conta própria.
- **Humanos**: OIDC validado no Gateway (`../004-API/`), com MFA exigido pelo `021`;
  claims propagadas.

### 2.2 Autorização

| Regra | Efeito |
|-------|--------|
| Consulta exige capability `obs:query:{metrics,traces,logs}` | Sem ela, negado pelo PDP |
| Administração exige capability por operação (`obs:signal:*`, `obs:slo:*`, `obs:alert:*`, `obs:retention:manage`) | `AIOS-OBS-0003` quando o tenant não confere |
| `aios.tenant` do lote **DEVE** casar com a identidade do certificado | Lote rejeitado; evento de segurança para `../021-Security/` |
| RLS por `tenant_id` nas tabelas de controle | Segunda camada, independente do código (`./Database.md`) |
| Rótulo de tenant obrigatório nas séries | Consulta cruzada negada estruturalmente (NFR-017) |

### 2.3 A assimetria deliberada de `fail_mode`

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ INGESTÃO           fail-OPEN   (obs.ingest.fail_mode = open)     │
   │  · saturação ⇒ descarta e contabiliza                            │
   │  · PDP indisponível ⇒ NÃO afeta a ingestão                       │
   │  racional: negar uma métrica não protege nada e quebra o emissor │
   └──────────────────────────────────────────────────────────────────┘
   ┌──────────────────────────────────────────────────────────────────┐
   │ CONSULTA/ADMIN     fail-CLOSED (obs.policy.fail_mode = closed)   │
   │  · PDP indisponível ⇒ consulta e administração NEGADAS           │
   │  racional: sem poder autorizar, ninguém lê telemetria de ninguém │
   └──────────────────────────────────────────────────────────────────┘
```

Essa assimetria é o oposto do `../022-Policy/`, que é *fail-closed* em todo o seu
caminho quente — e a diferença tem uma explicação simples: **negar uma autorização é
seguro; negar uma métrica não protege nada e ainda quebra quem a emitia.**

## 3. Integridade e confiança do sinal

| Controle | Como |
|----------|------|
| Origem verificada | Sinal só é aceito do `owner_module` declarado no `SignalDescriptor`, com identidade mTLS correspondente. |
| Sinal declarado | Registro obrigatório (`obs.signal.registration_required`, não-recarregável) — telemetria não registrada é quarentenada. |
| Descritor imutável | A partir de `Registered`; alteração exige nova `descriptor_version` (`AIOS-OBS-0014`). |
| Ledger append-only | `error_budget_ledger` não aceita alteração de lançamento passado. |
| Trilha de governança | Silêncios, mudanças de SLO e de retenção vão para `../025-Audit/`. |

> **Por que a origem importa.** Se qualquer serviço pudesse emitir
> `aios_kernel_syscall_latency_ms`, um atacante poderia injetar valores saudáveis para
> **mascarar** uma degradação real — o alerta nunca dispararia, e a métrica pareceria
> normal enquanto o sistema falhava.

## 4. Proteção de dados sensíveis

| Item | Regra |
|------|-------|
| Conteúdo de negócio | Prompt, resposta de LLM e item de memória **NÃO DEVEM** entrar em log, span ou métrica (`./Vision.md` §3.2). |
| Atributos `pii` | Redigidos ou tokenizados **na borda**, antes de qualquer persistência (ADR-0249). |
| Classificação | `signal_descriptor.data_class` ∈ {`operational`, `pii`, `sensitive`} dirige redação e retenção. |
| Labels de métrica | Identidade (`*_urn`, id de entidade) **proibida** — cardinalidade e privacidade pela mesma regra (ADR-0242). |
| Eventos | Não transportam amostra de telemetria nem valor de atributo sensível (`./Events.md` §1). |
| Tokenização | Chave gerida pelo `../021-Security/`; permite correlacionar sem identificar. |

**Redigir na borda, não no armazenamento:** redigir depois significaria que o dado
esteve em claro em disco — e backups, réplicas e índices já teriam a cópia.

## 5. Threat model STRIDE

| Ameaça | Vetor no 024-Observability | Mitigação | Verificação |
|--------|---------------------------|-----------|-------------|
| **S**poofing | Emissor falso injeta telemetria de outro serviço/tenant, poluindo SLOs e **mascarando um incidente real**. | mTLS com certificado atestado (`../021-Security/`); `aios.tenant` verificado contra a identidade; sinal aceito apenas do `owner_module` declarado. | Teste de contrato negativo; `AIOS-OBS-0003` monitorado. |
| **T**ampering | Alteração de regra de alerta ou de SLO para esconder degradação; silêncio permanente sobre alerta crítico; adulteração do ledger. | Mutações sob PEP + OCC e auditadas em `../025-Audit/`; silêncio com `expires_at` obrigatório e teto de 72 h; SLO alterado cria **nova versão**; ledger append-only. | T-OBS-014; revisão semanal de silêncios. |
| **R**epudiation | Negar ter silenciado, reconhecido ou alterado uma regra durante um incidente. | `created_by`, `acknowledged_by`, `reason` obrigatórios; eventos `telemetry.alert.*` na trilha imutável. | Auditoria cruzada evento × trilha. |
| **I**nformation disclosure | Log ou span carregando prompt, resposta de LLM ou PII; consulta cruzada entre tenants; inferência de comportamento alheio por métricas. | `PiiRedactor` na borda; `data_class` por sinal; proibição de conteúdo de negócio; RLS + `TenantRouter`; labels de identidade proibidos. | T-OBS-007, T-OBS-114; varredura contínua. |
| **D**enial of service | Inundação de telemetria por um tenant; explosão de cardinalidade; consulta que varre o armazenamento inteiro. | Limite de ingestão por tenant; orçamento de cardinalidade com quarentena **do sinal**, nunca bloqueio do tenant; orçamento de consulta (`AIOS-OBS-0010`). **O fail-open garante que o alvo do DoS seja o observador, nunca o observado.** | T-OBS-005; teste de carga por tenant. |
| **E**levation of privilege | Usar a consulta de logs como porta lateral para dados que a API de negócio não exporia; alterar retenção para reter dado que deveria ser expurgado. | Consulta sob PEP com capability **por tipo de sinal**; retenção sob `obs:retention:manage` e validada contra `data_class` (`AIOS-OBS-0015`); nenhum caminho de consulta ignora o `TenantRouter`. | T-OBS-017, T-OBS-018. |

### 5.1 Ameaça específica: o silêncio que vira cegueira

O padrão de abuso mais provável **não** é o ataque externo: é o silêncio renovado
indefinidamente até o alerta deixar de existir na prática. Controles:

- `expires_at` obrigatório com teto de 72 h (`obs.alert.max_silence_h`).
- `reason` e `created_by` obrigatórios — silêncio anônimo não existe.
- Retenção de 400 dias no stream `TELEMETRY_ALERT`, permitindo detectar reincidência.
- Alerta `ObservabilitySilenceChurnHigh` (`./Monitoring.md` §4) quando o mesmo
  `fingerprint` recebe mais de 3 silêncios em 30 dias — sinal de que o alerta está
  **errado** (limiar mau calibrado) ou de que o problema é **real e ignorado**.

## 6. LGPD / GDPR

| Item | Tratamento |
|------|-----------|
| **Minimização** | Telemetria é dado **operacional**: identificador e medida, não conteúdo. |
| **Redação na borda** | Atributos `pii` tokenizados ou removidos antes de persistir (ADR-0249). |
| **Base legal e retenção** | Cada camada declara `legal_basis` quando aplicável (CV-08); telemetria `operational` tem retenção curta por desenho (NFR-012). |
| **Direito ao esquecimento** | UC-015: ao consumir `security.rtbf.requested`, pseudonimiza ou expurga nas três lojas, preservando **agregados sem identificação**; comprovante correlacionado em `../025-Audit/`. |
| **Segregação** | Rótulo de tenant obrigatório; consulta cruzada negada; retenção configurável por tenant. |
| **Transferência internacional** | O armazenamento segue a região do tenant definida em `../027-Cluster/`; sem replicação entre regiões sem política explícita. |
| **Fronteira com auditoria** | O expurgo aqui **não** afeta a trilha do `../025-Audit/`, que tem base legal e retenção próprias (ADR-0246). |

## 7. Controles e verificação contínua

| Controle | Frequência | Evidência |
|----------|-----------|-----------|
| Nenhuma PII em sinal `operational` | Contínua (varredura) | NFR-014, T-OBS-114 |
| Nenhum label de identidade em produção | Contínua | NFR-009, T-OBS-006 |
| 100% dos emissores com mTLS válido | Contínua | Rejeições em `AIOS-OBS-0003` |
| Revisão de silêncios vigentes | Semanal | Relatório de `telemetry.silence` |
| Revisão de runbooks envelhecidos | Mensal | `ObservabilityRunbookStale` (NFR-016) |
| Revisão de capabilities de consulta | Trimestral | `GetEffectivePermissions` no `../022-Policy/` |
| Teste de consulta cross-tenant | A cada release | T-OBS-017, T-OBS-117 |
| Pentest da superfície de consulta | Semestral | Relatório de `../029-Operations/` |

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §12
- Identidade e credenciais: `../021-Security/`
- Autorização: `../022-Policy/` · Trilha imutável: `../025-Audit/`
- Contratos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6 e §7
- Modelo físico: `./Database.md` · Alertas: `./Monitoring.md`

*Fim de `Security.md`.*
