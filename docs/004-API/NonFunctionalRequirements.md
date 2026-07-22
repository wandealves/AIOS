---
Documento: NonFunctionalRequirements
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0044, ADR-0047
RFCs relacionados: RFC-0001, RFC-0040
Depende de: ./_DESIGN_BRIEF.md §7.2, ./Monitoring.md, ./Benchmark.md
---

# 004-API — Requisitos Não-Funcionais

> Reproduzido e detalhado a partir de `./_DESIGN_BRIEF.md` §7.2. As metas
> numéricas abaixo são **idênticas** em `./Monitoring.md`, `./Benchmark.md`,
> `./FailureRecovery.md`, `./Configuration.md` e `./Scalability.md` — qualquer
> divergência entre um desses documentos e esta tabela é defeito do outro
> documento, nunca deste.

## 1. Tabela consolidada por atributo de qualidade

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|------------------------------|
| NFR-001 | Performance (latência) | Overhead de borda do Gateway: p99 ≤ **5 ms**, p50 ≤ **1,5 ms** (sem contar o tempo do upstream). | Histograma `aios_api_gateway_overhead_latency_ms`; ver `./Metrics.md`. |
| NFR-002 | Performance (throughput) | ≥ **150.000 req/s** por réplica (proxy stateless, I/O-bound). | Load test sustentado — `./Benchmark.md`. |
| NFR-003 | Disponibilidade | ≥ **99,99%** do endpoint de borda (ponto único de entrada norte-sul). | Uptime probe multi-região; error budget mensal — `./Monitoring.md`. |
| NFR-004 | Escalabilidade (conexões) | ≥ **100.000** streams SSE concorrentes por réplica; escala horizontal sem limite teórico (stateless). | Teste de escala — `./Scalability.md`. |
| NFR-005 | Durabilidade | Perda de evento de registro de contrato = **0** sob *crash* (Outbox + JetStream). | Chaos test: *kill* entre commit e publish — `./FailureRecovery.md`. |
| NFR-006 | Recuperação (RTO/RPO) | RTO ≤ **5 min** por réplica (stateless, recarrega *snapshot*); RTO ≤ **10 min** para o `ContractRegistry` (PostgreSQL); RPO ≤ **5 min**. | *DR drill* — `./FailureRecovery.md`. |
| NFR-007 | Precisão de rate limit | *Overshoot* de limite `hard` ≤ **1%** sob concorrência. | Teste concorrente com token-bucket — `./Testing.md`. |
| NFR-008 | Segurança | **100%** dos tokens inválidos/expirados rejeitados antes de alcançar o upstream; **0** *bypass* de PEP de rota. | Pentest + auditoria de cobertura — `./Security.md`. |
| NFR-009 | Observabilidade | **100%** das requisições de borda com trace OTel + correlação completa. | Inspeção de spans; cobertura de campos — `./Monitoring.md`, `./Logging.md`. |
| NFR-010 | Idempotência | Repetições de mutação produzem efeito único em **100%** dos casos. | Teste de *replay*/duplicação — `./Testing.md`. |
| NFR-011 | Consistência de contrato agregado | `/v1/api/openapi.json` reflete novo contrato upstream em ≤ **60 s**. | Métrica `aios_api_contract_sync_lag_s` — `./Metrics.md`. |
| NFR-012 | Isolamento de falha (bulkhead) | Indisponibilidade de 1 upstream não aumenta o p99 de outras rotas em mais de **5%**. | Chaos test de falha isolada — `./FailureRecovery.md`. |

## 2. Detalhamento por categoria

### 2.1 Performance (NFR-001, NFR-002)

O Gateway é um proxy *I/O-bound*: seu orçamento de latência **não** inclui o
tempo de processamento do upstream — apenas o custo do próprio pipeline
(AuthN, AuthZ grossa com cache, rate limit, validação, roteamento). A meta de
p99 ≤ 5 ms DEVE ser mantida mesmo com `RouteAuthorizer` e `AuthNFilter` ativos
(caminho quente com cache de decisão/JWKS — ver `./Architecture.md` §7). O
throughput de 150.000 req/s por réplica é dimensionado para o caminho
*pass-through* mais simples (leitura, sem validação de schema pesada); ver
metodologia de carga em `./Benchmark.md`.

### 2.2 Disponibilidade (NFR-003)

99,99% de disponibilidade mensal corresponde a um *error budget* de
≈ 4,3 min/mês. Por ser ponto único de entrada, o Gateway DEVE degradar
graciosamente (§9 do brief) em vez de falhar completamente sob
indisponibilidade parcial de dependências (`021`, `022`).

### 2.3 Escalabilidade (NFR-004)

O limite de 100.000 *streams* SSE por réplica é uma restrição operacional de
memória de conexão, não teórica — a arquitetura *stateless* do
`SseStreamRelay` permite adicionar réplicas sem limite superior definido
(ver `./Scalability.md` §"SSE / conexões longas").

### 2.4 Segurança e Idempotência (NFR-008, NFR-010)

Estas metas são **absolutas** (100%, 0 *bypass*), refletindo o papel do
Gateway como fronteira de confiança única — qualquer regressão nessas metas
é tratada como incidente de segurança, não como degradação de qualidade.

## 3. Rastreabilidade

Todo `NFR-NNN` DEVE ser verificável por um SLI mensurável (coluna 3) e
DEVE reaparecer, com o mesmo valor numérico, em `./Monitoring.md` (alerta
correspondente), `./Benchmark.md` (meta de carga) e `./FailureRecovery.md`
(quando aplicável a RTO/RPO). Ver matriz consolidada em `./Requirements.md`.

*Fim de `NonFunctionalRequirements.md`.*
