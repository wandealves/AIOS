---
Documento: Vision
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0006, ADR-0008, ADR-0010 (globais); ADR-0040, ADR-0041, ADR-0042, ADR-0048 (deste módulo, a propor)
RFCs relacionados: RFC-0001, RFC-0040, RFC-0041
Depende de: 000-Vision, 001-Architecture, 021-Security, 022-Policy, 020-Communication
---

# 004-API — Visão do Módulo

> "O API Gateway está para o AIOS assim como a chamada de sistema pública e o
> *ambassador* de borda estão para um sistema operacional distribuído: a única
> porta pela qual o mundo externo entra, e o lugar onde o contrato — não a
> confiança — é a moeda de troca." — analogia condutora deste módulo (ver
> `../040-Glossary/Glossary.md`, termos **Control Plane**, **mTLS**, **PEP/PDP**).

---

## 1. Propósito do Módulo

O **004-API** é o **API Gateway** do AIOS: o **único ponto de entrada
norte-sul** para todo tráfego que se origina fora da malha interna de
serviços — CLI (`030-CLI`), SDK (`031-SDK`), Web Console (`032-WebConsole`),
integrações de terceiros e agentes externos via A2A. Ele corresponde à camada
**L3 BORDA** do Modelo de Camadas descrito em
`../001-Architecture/Architecture.md` §6 e é implementado sobre **YARP
(.NET 10)**.

O módulo tem duas responsabilidades indissociáveis, como estabelecido no
`_DESIGN_BRIEF.md` §0:

1. **Plano de execução do Gateway** — terminar REST/gRPC no *edge*, autenticar,
   autorizar grosseiramente por rota, aplicar rate limiting e idempotência de
   borda, validar schema, versionar e traduzir erro, e então rotear/agregar
   para o serviço do plano de controle dono do recurso.
2. **Registro de contratos (registry)** — manter, conforme RFC-0001 §8, os
   catálogos centrais e versionados que todo módulo consulta e para os quais
   contribui: tipos de recurso URN, domínios de subject NATS, catálogo de
   códigos de erro `AIOS-<DOMINIO>-<NNNN>` e versões de `dataschema` de evento.

O Gateway **decide se a chamada entra**; os serviços do plano de controle
**decidem o que fazer com ela**. Ele nunca executa lógica de domínio, nunca é
o PDP, e nunca gerencia identidade federada — ver `_DESIGN_BRIEF.md` §1.3.

---

## 2. Problema que o Módulo Resolve

A Visão Global (`../000-Vision/Vision.md` §2) identifica limitações estruturais
dos agentes de IA atuais. O 004-API é a resposta direta a três delas no ponto
de entrada do sistema:

| Limitação da Visão | Como o API Gateway endereça |
|---------------------|-------------------------------|
| **P8 — Governança fraca** | O Gateway é o primeiro **PEP** que qualquer requisição externa encontra: sem AuthN válida e sem decisão `allow` do PDP (`022-Policy`) para a rota, a requisição nunca alcança um módulo de domínio. Isso torna "nenhuma chamada externa sem governança" uma propriedade estrutural, não uma convenção que cada equipe de API teria que lembrar de aplicar. |
| **P7 — Baixa observabilidade** | Por ser passagem obrigatória de todo tráfego norte-sul, o Gateway é o ponto natural de instrumentação de borda: 100% das requisições produzem trace OTel, métrica `aios_api_*` e sinal de segurança correlacionado (NFR-009 do brief). |
| **P4 — Ausência de gerenciamento de recursos** | O Gateway aplica rate limiting e quotas de borda por tenant/rota antes que a carga alcance o plano de controle — a primeira linha de defesa contra *noisy neighbors* externos. |

Adicionalmente, o módulo resolve um problema de **engenharia de plataforma**
que nenhum módulo de domínio pode resolver isoladamente: como manter **um único
contrato de fronteira consistente** (URN, subjects, códigos de erro, schemas de
evento) quando dezenas de módulos evoluem de forma independente. Sem um
registro central governado, cada módulo inventaria seu próprio esquema de
identificação e erro, e a RFC-0001 §8 se tornaria inaplicável na prática. O
004-API é o **guardião operacional** desses registros.

### 2.1 Por que um Gateway único, e não um por domínio (BFF)

Uma alternativa de design seria um *Backend-For-Frontend* por consumidor
(CLI, SDK, Console) ou um gateway por domínio de módulo. Essa alternativa foi
descartada (ver ADR-0040, a propor) porque:

- **Fragmentaria o enforcement de AuthN/AuthZ de borda**, exigindo que cada
  BFF reimplemente validação de JWT e consulta ao PDP — risco de divergência
  de postura de segurança.
- **Fragmentaria os registros centrais de contrato** (RFC-0001 §8), que
  precisam de um único dono operacional para permanecerem consistentes.
- **Multiplicaria a superfície de ataque** norte-sul, com N pontos de entrada
  em vez de um único ponto auditável e enrijecido.

O Gateway único **centraliza** essas preocupações transversais e **delega**
toda lógica de negócio aos serviços de destino — ele é *front door*, nunca
executor (ver Não-Responsabilidade NR-01 do `_DESIGN_BRIEF.md`).

---

## 3. Escopo

### 3.1 Escopo — Dentro (o API Gateway É responsável por)

| Área | Descrição |
|------|-----------|
| Terminação de protocolo | REST e gRPC no *edge*, incluindo gRPC-Web para o Web Console. |
| AuthN de borda | Validação de token OAuth2/OIDC (JWT) via JWKS de `021-Security`. |
| AuthZ grossa de rota | Consulta ao PDP (`022-Policy`) por rota/role, com *default deny*. |
| Rate limiting/quotas de borda | Token-bucket por tenant e por rota. |
| Correlação | Geração/propagação de `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` (RFC-0001 §5.6). |
| Idempotência de borda | Cache de resultado por `Idempotency-Key` em mutações. |
| Validação de schema | OpenAPI (REST) e descritor proto (gRPC) antes de rotear. |
| Tradução de erro | Normalização no envelope RFC-0001 §5.4, mesmo para falha de upstream não conforme. |
| Versionamento de API | `/vN` + header, FSM de `ApiVersion` (coexistência ≥ 2 majors). |
| Agregação de contrato | Publicação do OpenAPI/proto unificado dos módulos upstream. |
| Registro de contratos | Tipos URN, domínios de subject, catálogo de erros, `dataschema` (RFC-0001 §8). |
| Streaming | SSE/long-poll multiplexando consumo de NATS/JetStream por tenant. |
| Isolamento de upstream | Circuit breaker/bulkhead por serviço de destino. |

### 3.2 Escopo — Fora (o API Gateway NÃO é responsável por)

Reproduzido do `_DESIGN_BRIEF.md` §1.3 (não redefinido aqui, resumido para
orientar o leitor deste documento de Visão):

| Área excluída | Dono real | Por quê |
|----------------|-----------|---------|
| Lógica de negócio de qualquer módulo | Cada módulo de destino | O Gateway roteia/agrega, não executa domínio. |
| Autorização fina de capacidade/recurso | Módulo de destino + `022-Policy` | O Gateway autoriza a rota; a decisão fina é do PEP local do módulo (ex.: Kernel `006`). |
| Emissão/renovação/revogação de tokens; JWKS | `021-Security` | O Gateway apenas valida com chaves publicadas. |
| Persistência de estado de negócio (ACB, memória, planos) | Módulo dono do dado | O Gateway persiste apenas seus registros de contrato/borda. |
| Trilha de auditoria imutável | `025-Audit` | O Gateway emite sinais; o Audit persiste. |
| *Scheduling*/*placement* de agentes/tarefas | `009-Scheduler` | Fora do escopo de borda. |
| Sandbox/isolamento de execução de agente | `007-Agent-Runtime` / `021-Security` | Fora do escopo de borda. |
| Topologia de deployment, autoscaling, anti-affinity | `028-Deployment` / `027-Cluster` | O Gateway é operado por essa topologia, não a define. |
| Cálculo de custo/orçamento de tenant | `026-Cost-Optimizer` | Fora do escopo de borda. |
| Broker de mensageria em si | `020-Communication` | O Gateway publica/consome subjects, não provê o broker. |
| Fonte da verdade de política RBAC/ABAC | `022-Policy` | O Gateway é PEP de borda; a decisão é do PDP. |

Esta separação segue o **Princípio 1** da Visão Global (Separação Plano de
Controle / Plano de Dados) especializado para a **fronteira norte-sul**: o
Gateway é o "porteiro", não o "gerente do prédio".

---

## 4. Personas Atendidas

| Persona | Como interage com o Gateway | Necessidade satisfeita |
|---------|------------------------------|--------------------------|
| **Desenvolvedor de Agentes** | Via `031-SDK`, que traduz chamadas de alto nível em REST/gRPC via o Gateway. | Um único ponto de entrada estável, versionado e documentado (OpenAPI agregado). |
| **Operador de Plataforma (SRE)** | Consome métricas `aios_api_*`, dashboards de latência/erro de borda; opera `api.gateway.*`. | Visibilidade de saturação de borda e controles operacionais (rate limit, timeouts, circuit breaker). |
| **Arquiteto Enterprise** | Governa os registros centrais (`RouteDefinition`, `ApiVersion`, catálogo de erros/eventos/URN) via papel `api-registry-admin`. | Consistência de contrato entre todos os módulos do AIOS. |
| **CISO / Compliance** | Audita sinais de segurança (`api.request.rejected`, `api.contract.violation`) emitidos ao `025-Audit`. | Prova de que 100% do tráfego externo passou por AuthN/AuthZ de borda. |
| **Integrador Externo / Parceiro** | Consome `GET /v1/api/openapi.json` e a superfície REST versionada. | Contrato público estável, com política de depreciação previsível (≥180 dias). |
| **CLI (030) / Web Console (032)** | Clientes diretos de toda a superfície *pass-through* e das rotas de administração/registro. | Ponto único de entrada, com SSE para eventos de tarefa em tempo real. |

---

## 5. Princípios de Design do API Gateway

Estes princípios especializam, para este módulo, os princípios invioláveis da
Visão Global (`../000-Vision/Vision.md` §6). Toda ADR do módulo
(`ADR-0040`..`ADR-0049`) DEVE ser avaliada contra eles.

1. **Fronteira de confiança única, não contornável.** Nenhum tráfego externo
   NÃO DEVE alcançar um serviço do plano de controle sem passar pelo Gateway;
   upstreams internos NÃO DEVEM ser alcançáveis diretamente da rede pública
   (segmentação de rede, `028-Deployment`).
2. **Front door, nunca executor.** O Gateway NÃO DEVE conter lógica de
   negócio de qualquer módulo — apenas os *cross-cutting concerns* de borda
   (AuthN, AuthZ grossa, rate limit, idempotência, validação de schema,
   correlação, tradução de erro).
3. **Default deny estrutural.** Nenhuma rota `auth_required` DEVE ser roteada
   sem decisão explícita `allow` do PDP; na ausência de decisão
   (`022-Policy` indisponível), o Gateway DEVE **negar com segurança**
   (`api.auth.fail_mode=closed` por padrão).
4. **Stateless por natureza.** O Gateway DEVE ser um proxy sem estado de
   processo — qualquer réplica atende qualquer requisição; todo estado
   sobrevivente reside em Redis/PostgreSQL/NATS.
5. **Idempotência antes de performance.** Toda mutação exposta externamente
   DEVE ser segura para repetir; otimizações de latência NÃO DEVEM
   comprometer essa garantia (RFC-0001 §5.5).
6. **Coexistência de versões como contrato, não exceção.** O Gateway DEVE
   sustentar ≥ 2 majors de API coexistentes por módulo, com depreciação
   anunciada e janela mínima antes de retirada (RFC-0001 §5.7).
7. **Autorização em duas camadas, nunca substituição.** O Gateway é PEP
   grosso de rota; ele NÃO substitui a autorização fina do serviço de
   destino — é uma camada adicional de defesa em profundidade.
8. **Isolamento de falha por upstream.** A indisponibilidade de um módulo
   NÃO DEVE degradar a latência ou disponibilidade das rotas de outros
   módulos (circuit breaker + bulkhead por serviço).
9. **Registros de contrato são metadados de plataforma, não de tenant.** Os
   registros de RFC-0001 §8 NÃO carregam `tenant_id`/RLS; sua escrita é
   restrita a um papel administrativo (`api-registry-admin`), sua leitura é
   pública.
10. **Observabilidade sem exceção.** 100% das requisições de borda DEVEM
    produzir trace, métrica e sinal de segurança quando aplicável — inclusive
    (e principalmente) as rejeitadas.

---

## 6. Relação com a Visão Global do AIOS

O 004-API é o módulo que materializa, na prática, o conceito de "fronteira
norte-sul governada" da Arquitetura Global (`../001-Architecture/Architecture.md`
§4, §6). Três relações merecem destaque:

- **V-R4 (auditoria imutável de 100% das ações privilegiadas)**: por ser
  passagem obrigatória de todo tráfego externo, o Gateway é o primeiro ponto
  de aplicação estrutural dessa meta — toda rejeição de AuthN/AuthZ, toda
  violação de contrato e todo *breach* de rate limit gera um sinal de
  segurança correlacionado (`api.request.rejected`, `api.contract.violation`,
  `api.ratelimit.breached`) consumido pelo `025-Audit`.
- **RFC-0001 §5.7 e §8**: o Gateway é o módulo que **opera** (não define) a
  regra de versionamento de API e os registros centrais criados pela RFC-0001
  — ele é a implementação viva desses contratos.
- **Escala do sistema**: diferente de módulos de execução de agente, a carga
  do Gateway escala com o número de **operadores/integrações** (humanos,
  CLIs, dashboards, SDKs), não linearmente com o número de agentes ativos —
  a carga de execução trafega majoritariamente **dentro** do plano de
  controle (ver `_DESIGN_BRIEF.md` §10 e `../001-Architecture/Architecture.md`
  §11).

---

## 7. Não-Objetivos Explícitos (deste módulo)

Além das Não-Responsabilidades já listadas em §3.2, o módulo declara
explicitamente os seguintes não-objetivos de **produto/design**:

- O Gateway NÃO tem por objetivo se tornar um **motor de transformação de
  payload** de propósito geral; suas transformações se limitam a
  transcodificação REST↔gRPC (gRPC-Web) e normalização de erro.
- O Gateway NÃO tem por objetivo ser o **PDP**; ele consulta o `022-Policy`,
  nunca decide política localmente além do *default deny* estrutural.
- O Gateway NÃO tem por objetivo prover **UI** de qualquer natureza; consumo
  humano de sua administração ocorre via `032-WebConsole` ou `030-CLI`.
- O Gateway NÃO tem por objetivo ser a fonte de verdade de **identidade**
  (isso é `021-Security`) nem de **orçamento de custo** (isso é
  `026-Cost-Optimizer`) — ele apenas aplica os contratos que esses módulos
  definem.
- O Gateway NÃO tem por objetivo, na v1.0, aceitar registro de rota/versão
  fora de um processo de PR + ADR/RFC de módulo — os registros de RFC-0001
  §8 são deliberadamente governados, não *self-service* irrestrito.

---

## 8. Riscos Estratégicos Específicos do Módulo

Complementando os riscos globais da Visão (`../000-Vision/Vision.md` §10), o
004-API carrega riscos próprios por ser o único ponto de entrada externo:

| Risco | Descrição | Mitigação | Referência |
|-------|-----------|-----------|------------|
| RSK-A01 | O Gateway se tornar **alvo primário de DoS** (único ponto de entrada norte-sul). | Rate limiting por tenant/rota, limite de tamanho de corpo, autoscaling (`028`), proteção L7/edge (`027`). | `_DESIGN_BRIEF.md` §12.2 |
| RSK-A02 | *Bypass* do PEP de borda via acesso direto a upstream. | Segmentação de rede: upstreams só aceitam mTLS do Gateway/malha interna. | `_DESIGN_BRIEF.md` §12.1 |
| RSK-A03 | Inconsistência do contrato agregado (`OpenApiAggregator`) atrasado em relação aos módulos. | Meta `aios_api_contract_sync_lag_s` ≤ 60 s (NFR-011); re-sync sob demanda. | `_DESIGN_BRIEF.md` §7.2 |
| RSK-A04 | Indisponibilidade de `021`/`022` travar todo o tráfego externo. | `fail_mode=closed` configurável + cache de decisão/JWKS com TTL curto + circuit breaker. | `_DESIGN_BRIEF.md` §9 |
| RSK-A05 | Registros globais de contrato mutados incorretamente por engano administrativo. | Papel `api-registry-admin` dedicado, OCC (`version`), auditoria imutável de toda mutação. | `_DESIGN_BRIEF.md` §3, ADR-0043 |

---

## 9. Referências

- Visão Global do AIOS: `../000-Vision/Vision.md`
- Arquitetura Global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Design Brief interno (fonte de verdade do módulo): `./_DESIGN_BRIEF.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../021-Security/`, `../022-Policy/`, `../020-Communication/`,
  `../024-Observability/`, `../025-Audit/`, `../027-Cluster/`, `../028-Deployment/`,
  `../030-CLI/`, `../031-SDK/`, `../032-WebConsole/`.
- Módulo de referência de profundidade: `../006-Kernel/_DESIGN_BRIEF.md`.

---

*Fim de `Vision.md`. Próximo documento na cadeia do módulo:* `./Architecture.md`.
