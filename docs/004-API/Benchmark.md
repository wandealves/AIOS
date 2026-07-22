---
Documento: Benchmark
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0044
RFCs relacionados: RFC-0001
Depende de: ./NonFunctionalRequirements.md, ./Deployment.md, ./Scalability.md
---

# 004-API — Benchmark

> Metodologia e metas de carga do Gateway. Todas as metas numéricas são
> **idênticas** às de `./NonFunctionalRequirements.md` (NFR-001, NFR-002,
> NFR-004, NFR-007).

## 1. Metodologia

- **Ferramenta de carga**: gerador HTTP/gRPC capaz de sustentar centenas de
  milhares de req/s com controle preciso de taxa (ex.: k6, Gatling ou
  vegeta), executando fora do cluster do Gateway para não competir por CPU.
- **Topologia sob teste**: 1 réplica isolada do Gateway (para medir custo por
  réplica) e, separadamente, o cluster completo mínimo de 3 réplicas (para
  validar escala horizontal linear).
- **Upstream de teste**: serviço *mock* de latência configurável (para
  isolar o overhead do Gateway do tempo de upstream — necessário para medir
  NFR-001 corretamente) e, em uma segunda rodada, um upstream real
  (`006-Kernel` em ambiente de *staging*) para validar o comportamento
  ponta-a-ponta.
- **Perfis de carga**: (a) *pass-through* simples (GET, sem validação de
  schema pesada); (b) mutação com `Idempotency-Key` e validação de schema;
  (c) SSE de longa duração; (d) mistura realista (80% leitura / 20%
  mutação), representativa do tráfego esperado de CLI/SDK/Console.

## 2. Ambiente de referência

| Componente | Especificação |
|------------|-----------------|
| Réplica do Gateway | 2 vCPU / 1 GiB RAM (conforme `./Deployment.md` §2). |
| Redis | Cluster dedicado, latência de rede < 1 ms até o Gateway. |
| PostgreSQL | Instância dedicada, apenas leitura durante o *benchmark* (registro carregado em *snapshot*). |
| Upstream mock | Latência fixa configurável (0 ms, 10 ms, 50 ms) para isolar overhead. |
| Rede | Mesma zona de disponibilidade (elimina variância de latência entre AZs). |

## 3. KPIs e resultados-alvo

| KPI | Meta | NFR | Perfil de carga |
|-----|------|-----|-------------------|
| Overhead p99 | ≤ 5 ms | NFR-001 | (a) *pass-through* simples |
| Overhead p50 | ≤ 1,5 ms | NFR-001 | (a) *pass-through* simples |
| Throughput sustentado por réplica | ≥ 150.000 req/s | NFR-002 | (a) *pass-through* simples, upstream 0 ms |
| *Streams* SSE concorrentes por réplica | ≥ 100.000 | NFR-004 | (c) SSE de longa duração |
| *Overshoot* de rate limit `hard` | ≤ 1% | NFR-007 | Carga concorrente acima do limite configurado |
| Latência de entrega SSE (publicação→cliente) | p99 ≤ 2000 ms | FR-012 | (c) SSE, sob 10.000 *streams* ativos |

## 4. Comparação com *baseline*

Todo *release* candidato **DEVE** ser comparado ao *baseline* da última
versão `Stable` do módulo. Regressão > 10% em qualquer KPI da tabela acima,
sem justificativa registrada em ADR/PR, **DEVE** bloquear o *release* (ver
critério de *gate* em `./Testing.md` §5).

## 5. Reprodutibilidade

- Todo *benchmark* **DEVE** ser executado a partir de uma definição versionada
  (script + parâmetros de carga) armazenada junto ao repositório de
  implementação do módulo (fora do escopo deste repositório de
  documentação).
- Resultados **DEVEM** registrar: *commit hash*, topologia exata (§2),
  duração do teste (mínimo 10 min em regime estacionário, excluindo *warm-up*
  de 2 min), e percentis completos (p50/p90/p95/p99/p99.9).
- Testes de throughput (NFR-002) **DEVEM** ser executados com e sem
  validação de schema ativa, documentando separadamente o custo de
  `SchemaValidator` (FR-007, Should) sobre o caminho quente.

*Fim de `Benchmark.md`.*
