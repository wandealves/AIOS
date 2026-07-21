# AIOS — Padrão de Documentação de Módulo

> **Este arquivo é normativo.** Todo módulo (`NNN-Nome`) do repositório de documentação
> DEVE conter os 26 documentos obrigatórios listados abaixo, cada um seguindo o
> esqueleto correspondente. Nenhum documento pode ser apenas uma lista de títulos.
> A ausência de um documento obrigatório é considerada um defeito de documentação
> (bloqueia o merge — ver `033-DeveloperGuide/DocumentationCI.md`).

## Convenções globais

| Convenção | Regra |
|-----------|-------|
| Idioma | Português (pt-BR). Termos técnicos consagrados permanecem em inglês (ex.: *scheduler*, *throughput*, *idempotency*). |
| Palavras-chave normativas | `DEVE`/`MUST`, `NÃO DEVE`/`MUST NOT`, `DEVERIA`/`SHOULD`, `PODE`/`MAY`, conforme RFC 2119 / RFC 8174. |
| Diagramas | Sempre em ASCII (portabilidade máxima em Git, PR e terminal). Notação C4 e UML adaptadas para ASCII. |
| Identificadores | Componentes: `PascalCase`. Eventos: `dominio.entidade.acao` (ex.: `agent.lifecycle.spawned`). Métricas: `aios_<subsistema>_<nome>_<unidade>`. |
| Versionamento do doc | Cabeçalho `Status` + `Versão` + `Última atualização` + `ADRs relacionados`. Ver bloco abaixo. |
| Referência cruzada | Sempre por caminho relativo: `[Kernel](../006-Kernel/Architecture.md)`. |
| Unidades | Latência em `ms`, `p50/p95/p99`; throughput em `req/s` ou `msg/s`; tamanho em `MiB/GiB`; custo em `USD` e `tokens`. |

### Cabeçalho obrigatório de todo documento

```
---
Documento: <Nome do Documento>
Módulo: NNN-Nome
Status: Draft | Review | Stable | Deprecated
Versão: MAJOR.MINOR (SemVer da documentação)
Última atualização: AAAA-MM-DD
Responsável (RACI-A): <papel/equipe>
ADRs relacionados: ADR-XXXX, ADR-YYYY
RFCs relacionados: RFC-XXXX
Depende de: <módulos/documentos>
---
```

---

## Os 26 documentos obrigatórios e seus esqueletos

### 1. `Vision.md`
Propósito do módulo, problema que resolve, escopo (in/out), personas atendidas,
princípios de design, não-objetivos explícitos, relação com a visão global do AIOS
(`../000-Vision/Vision.md`).

### 2. `Architecture.md`
Visão C4 (Context → Container → Component) em ASCII, decomposição interna,
responsabilidades por componente, fronteiras, padrões arquiteturais adotados,
tecnologias (.NET 10 / Python / NATS / PostgreSQL+pgvector+AGE / Redis / MinIO),
justificativa de cada escolha, alternativas descartadas (linkar ADRs).

### 3. `Requirements.md`
Índice consolidado de requisitos, rastreabilidade (matriz Requisito → UseCase →
Teste), stakeholders, glossário local.

### 4. `FunctionalRequirements.md`
Tabela `FR-NNN` (id, descrição, prioridade MoSCoW, critério de aceite, origem).
Cada FR mensurável e verificável.

### 5. `NonFunctionalRequirements.md`
Tabela `NFR-NNN` por atributo de qualidade (performance, escalabilidade,
disponibilidade, segurança, observabilidade, manutenibilidade) com metas
numéricas (SLO/SLI) e método de verificação.

### 6. `UseCases.md`
Casos de uso no formato ator/pré-condições/fluxo principal/fluxos alternativos/
exceções/pós-condições. Numerados `UC-NNN`.

### 7. `SequenceDiagrams.md`
Diagramas de sequência ASCII para os fluxos críticos (feliz + falha), participantes,
mensagens síncronas/assíncronas, timeouts.

### 8. `ClassDiagrams.md`
Diagramas de classe/estrutura ASCII, interfaces públicas, contratos, invariantes,
relações (composição/agregação/dependência).

### 9. `StateMachine.md`
Máquina(s) de estado do módulo: estados, transições, gatilhos, guardas, ações de
entrada/saída, estados terminais, invariantes.

### 10. `Database.md`
Modelo físico (PostgreSQL), DDL, índices (inclui pgvector/AGE quando aplicável),
particionamento, políticas de retenção, migrações, chaves e constraints.

### 11. `API.md`
Contratos REST (OpenAPI) e gRPC (proto), versionamento, autenticação, códigos de
erro, idempotência, paginação, exemplos de request/response.

### 12. `Events.md`
Catálogo de eventos NATS: subject, schema JSON, produtor, consumidores, semântica
de entrega (at-least-once/exactly-once), ordenação, versionamento de schema.

### 13. `Configuration.md`
Todas as chaves de configuração, tipo, default, faixa válida, escopo (global/por
tenant/por agente), recarregável em runtime? variáveis de ambiente.

### 14. `Deployment.md`
Topologia (Docker/Compose), recursos (CPU/RAM/GPU), réplicas, dependências,
health/readiness, estratégia de rollout, YARP/gateway.

### 15. `Security.md`
AuthN/AuthZ (OAuth2/OIDC, RBAC/ABAC), superfície de ataque, threat model (STRIDE),
segredos, TLS/mTLS, sandbox, LGPD/GDPR, controles.

### 16. `Monitoring.md`
Dashboards, alertas (regras Prometheus), SLO/SLA/SLI, golden signals, runbooks
associados.

### 17. `Logging.md`
Eventos de log estruturado (Serilog→Seq), níveis, campos obrigatórios, correlação
(trace_id/span_id/tenant_id), retenção.

### 18. `Metrics.md`
Catálogo de métricas OpenTelemetry/Prometheus: nome, tipo (counter/gauge/histogram),
labels, unidade, semântica.

### 19. `Testing.md`
Estratégia de testes (unit/integration/contract/e2e/chaos/load), cobertura-alvo,
fixtures, ambientes, critérios de qualidade de gate.

### 20. `Benchmark.md`
Metodologia, cargas, ambiente, KPIs, resultados-alvo, comparação com baseline,
reprodutibilidade.

### 21. `FailureRecovery.md`
Modos de falha (FMEA), detecção, isolamento, recuperação, idempotência,
retries/backoff, DLQ, RTO/RPO, degradação graciosa.

### 22. `Scalability.md`
Modelo de escala (horizontal/vertical), particionamento/sharding, limites teóricos,
concorrência, locks, backpressure, caminho para milhões de agentes.

### 23. `Examples.md`
Exemplos executáveis (CLI/SDK/API), do "hello world" ao avançado, snippets
comentados.

### 24. `FAQ.md`
Perguntas frequentes, armadilhas comuns, esclarecimentos de escopo.

### 25. `ADR.md`
Índice das ADRs que afetam o módulo (link para `../002-ADR/`), com resumo de 1 linha
de cada decisão e seu status.

### 26. `RFC.md`
Índice das RFCs relacionadas (link para `../003-RFC/`), status e impacto no módulo.

---

## Checklist de qualidade (Definition of Done por documento)

- [ ] Cabeçalho de metadados preenchido.
- [ ] Nenhuma seção vazia ("TBD" só permitido em `Status: Draft`).
- [ ] Ao menos 1 diagrama ASCII quando o esqueleto o exige.
- [ ] Ao menos 1 tabela e 1 exemplo quando aplicável.
- [ ] Termos reutilizam o `../040-Glossary/Glossary.md` (sem redefinir).
- [ ] Decisões apontam para uma ADR (não decidir localmente).
- [ ] Requisitos numerados e rastreáveis.
- [ ] Riscos e alternativas explicitados.
- [ ] Referências cruzadas por caminho relativo válidas.
