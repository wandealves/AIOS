---
Documento: ClassDiagrams
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0222, ADR-0223, ADR-0225
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: Architecture.md, StateMachine.md, API.md
---

# 022-Policy — Diagramas de Classe e Contratos

> As interfaces abaixo são **conceituais** (este repositório é de documentação, não de
> implementação) e servem para fixar as fronteiras testáveis do módulo, referenciadas
> em `./Testing.md`. Nomes de componentes são idênticos aos de `./Architecture.md` §3.1.

## 1. Visão estrutural do caminho de decisão

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │                        DecisionGateway                              │
   │  + Evaluate(req: DecisionRequest): DecisionResponse                 │
   │  + EvaluateBatch(reqs: DecisionRequest[]): DecisionResponse[]       │
   │  - ValidateTenant(req, authContext): void                           │
   │  - ApplyRateLimit(tenantId): void                                   │
   └───────────────────────────────┬─────────────────────────────────────┘
                                   │ ◆ (composição)
                                   ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │                        DecisionEngine                               │
   │  + Decide(req: DecisionRequest, bundle: CompiledBundle): Decision   │
   │  - Combine(matches: RuleMatch[]): InternalEffect                    │
   │  - Collapse(effect: InternalEffect): ExternalEffect                 │
   │  invariante: Decide é PURA dado (req, bundle, attributes)           │
   └──┬────────┬─────────┬──────────┬──────────────┬────────────────────┘
      │        │         │          │              │
      ▼        ▼         ▼          ▼              ▼
  ┌────────┐ ┌──────┐ ┌──────┐ ┌───────────┐ ┌──────────────────┐
  │Action  │ │Rbac  │ │Abac  │ │Attribute  │ │ObligationComposer│
  │Normal. │ │Resol.│ │Eval. │ │Resolver   │ │                  │
  └────────┘ └──────┘ └──────┘ └───────────┘ └──────────────────┘
                                   │
                                   ▼ (dependência)
                          ┌──────────────────┐
                          │  BundleRuntime   │  ← troca atômica de versão
                          └──────────────────┘
```

## 2. Interfaces do caminho quente

```
interface IDecisionGateway {
  DecisionResponse   Evaluate(DecisionRequest req, AuthContext ctx);
  DecisionResponse[] EvaluateBatch(DecisionRequest[] reqs, AuthContext ctx);
}

interface IDecisionEngine {                    // PURO: sem I/O próprio
  Decision Decide(DecisionRequest req, CompiledBundle bundle, AttributeSet attrs);
}

interface IActionNormalizer {
  CanonicalAction Normalize(string rawAction);  // → <dominio>:<objeto>:<verbo>
}

interface IRbacResolver {
  EffectiveRoles  Resolve(SubjectRef subject, CompiledBundle bundle);
  Permission[]    PermissionsOf(EffectiveRoles roles);
}

interface IAbacEvaluator {
  RuleMatch[] Evaluate(Rule[] candidates, AttributeSet attrs, RequestContext ctx);
}

interface IAttributeResolver {
  AttributeSet Resolve(AttributeKey[] required, RequestContext ctx);
  // required não resolvido ⇒ AttributeSet.Indeterminate(key)
}

interface IObligationComposer {
  Obligation[] Compose(RuleMatch[] matches, int maxPerDecision);
  int          TtlFor(RuleMatch[] matches, ExternalEffect effect);
}

interface IBundleRuntime {
  CompiledBundle ActiveFor(string tenantId, BundleScope scope);   // null ⇒ deny (I5)
  void           Swap(CompiledBundle next);                        // atômica
  int            ActiveVersion(string tenantId, BundleScope scope);
}
```

### 2.1 Contrato de dados (RFC-0220)

```
record DecisionRequest {
  SubjectRef  subject;      // urn + roles/claims propagadas
  string      action;       // forma canônica ou curta (normalizada)
  string      resource;     // URN do recurso (RFC-0001 §5.1)
  Environment environment;  // tenant, hora, origem, prioridade, refs de cota
  Attribute[] attributes;   // atributos fornecidos pelo PEP (opcional)
}

record DecisionResponse {
  ExternalEffect effect;          // allow | deny  (nunca indeterminate)
  string         reasonCode;      // conjunto estável (_DESIGN_BRIEF §4.7)
  string[]       matchedRules;    // rule_key das regras casadas
  Obligation[]   obligations;     // o PEP DEVE cumprir todas
  int            decisionTtlMs;   // deny ≤ allow
  int            bundleVersion;
  string         decisionId;      // ULID
  string         evaluatedAt;     // RFC 3339 UTC
}

record Obligation { string kind; Map<string,string> parameters; }
```

**Invariantes de contrato:**

- (C1) `effect ∈ {allow, deny}` — `indeterminate` e `not_applicable` são colapsados
  antes de sair do módulo (ADR-0224).
- (C2) `effect = allow ⇒ matchedRules ≠ []` (NFR-008: não existe `allow` sem regra).
- (C3) `effect = deny ⇒ reasonCode ≠ null` (NFR-007).
- (C4) `decisionTtlMs ≤ pol.decision.default_ttl_ms`; para `deny`,
  `≤ pol.decision.deny_ttl_ms`.
- (C5) `obligations.length ≤ pol.obligation.max_per_decision`.
- (C6) Um PEP que não saiba cumprir uma obrigação **DEVE** tratar a decisão como
  `deny` (ADR-0225).

## 3. Estruturas do ciclo de vida do bundle

```
   ┌──────────────────────┐        compila         ┌─────────────────────┐
   │   BundleCompiler     │───────────────────────▶│   CompiledBundle    │
   │ + Compile(src):      │                        │ - ruleIndex         │
   │     CompiledBundle   │◆──── usa ─────┐        │ - roleGraph         │
   │ + Limits: BundleLimits│              │        │ - compiledDigest    │
   └──────────┬───────────┘              │        │ - signature         │
              │                  ┌───────▼──────┐ └─────────────────────┘
              │                  │ConflictAnalyz│           │
              │                  │+ Analyze():  │           │ carregado por
              │                  │  Conflict[]  │           ▼
              │                  └──────────────┘  ┌─────────────────────┐
              ▼                                    │   BundleRuntime     │
   ┌──────────────────────┐   persiste/versiona    └─────────────────────┘
   │   BundleRegistry     │◀───────────────────────────────┐
   │ + Save(bundle)       │                                 │ ativa
   │ + Transition(urn,to) │        ┌────────────────────────┴──────────┐
   │ + Verify(signature)  │        │        BundleDistributor          │
   └──────────┬───────────┘        │ + Activate(urn): void             │
              │                     │ + Propagate(version): void        │
              │                     └───────────────────────────────────┘
              ▼
   ┌──────────────────────┐   ┌──────────────────────┐
   │   PolicyTestRunner   │   │   SimulationEngine   │
   │ + Run(bundle): Report│   │ + Simulate(cand,base,│
   │ + Coverage(): decimal│   │     window): Diff    │
   └──────────────────────┘   └──────────────────────┘
```

```
interface IBundleRegistry {
  BundleRef  Save(BundleSource src, string authoredBy);
  void       Transition(string bundleUrn, BundleState target, long expectedVersion);
  bool       VerifySignature(CompiledBundle bundle);
  CompiledBundle Load(string tenantId, int bundleVersion);   // reprodutibilidade (I6)
}

interface IBundleCompiler {
  CompileResult Compile(BundleSource src, BundleLimits limits);
  // CompileResult = { CompiledBundle? artifact, Conflict[] conflicts, Error[] errors }
}

interface IConflictAnalyzer {
  Conflict[] Analyze(CompiledBundle bundle);  // contradição, sombreamento, inalcançável
}

interface IPolicyTestRunner {
  TestReport Run(CompiledBundle candidate, PolicyTest[] cases);
  decimal    Coverage(TestReport report);     // gate: ≥ pol.test.min_coverage
}

interface ISimulationEngine {
  SimulationDiff Simulate(CompiledBundle candidate, CompiledBundle baseline,
                          TimeWindow window, int maxRequests);
}

interface IExceptionManager {
  ExceptionRef Grant(ExceptionRequest req, string approvedBy);  // approvedBy ≠ requestedBy
  void         Revoke(string exceptionUrn, string revokedBy);
  Exception[]  ActiveFor(SubjectRef subject, CanonicalAction action);
}

interface IExplainService {
  Explanation Explain(string decisionId);     // reproduz sobre a bundle_version exata
}

interface ISelfGovernanceGuard {
  void Authorize(AdminOperation op, AuthContext ctx);   // contra o bundle-raiz
  void RequireDualApproval(AdminOperation op, string authored, string approved);
}
```

## 4. Relações entre componentes

| Relação | Tipo | Semântica |
|---------|------|-----------|
| `DecisionGateway` ◆── `DecisionEngine` | Composição | O engine não é alcançável fora do gateway; não há rota de decisão sem validação de tenant e limite. |
| `DecisionEngine` ◇── `BundleRuntime` | Agregação | O runtime é compartilhado por todas as avaliações da réplica; sua troca é atômica. |
| `DecisionEngine` ──▶ `AttributeResolver` | Dependência | Única fonte de I/O possível no caminho quente, sob timeout rígido. |
| `BundleCompiler` ◆── `ConflictAnalyzer` | Composição | Análise de conflito é etapa da compilação, não passo opcional posterior. |
| `BundleDistributor` ──▶ `EventEmitter` | Dependência | Ativação e evento na mesma transação (invariante I4). |
| `SimulationEngine` ──▶ `DecisionEngine` | Dependência | Reusa **o mesmo** motor em modo sombra — simular com outro código avaliaria outra política. |
| `SelfGovernanceGuard` ──▶ `BundleRuntime` | Dependência | Consulta o bundle-raiz (`scope = platform`), nunca o do tenant. |
| `ExplainService` ──▶ `BundleRegistry` | Dependência | Carrega a versão exata; sem ela, não há explicação fiel. |

## 5. Invariantes estruturais

| ID | Invariante | Consequência do rompimento |
|----|-----------|----------------------------|
| S1 | `DecisionEngine.Decide` é **puro** dado (`req`, `bundle`, `attrs`). | Sem pureza, `ExplainDecision` não reproduz a decisão (NFR-009 cai). |
| S2 | Nenhum caminho de decisão alcança o banco de dados. | O p99 do PDP passaria a depender do p99 do PostgreSQL (NFR-001 cai). |
| S3 | `BundleRuntime.Swap` é atômica: uma avaliação vê **uma** versão do início ao fim. | Decisão híbrida entre duas políticas — inauditável. |
| S4 | O `SimulationEngine` usa o mesmo `DecisionEngine` da produção. | Simulação deixaria de prever o comportamento real. |
| S5 | `SelfGovernanceGuard` é o único caminho para operações administrativas. | Escalada de privilégio por rota alternativa (`./Security.md` §2.3). |
| S6 | O journal é escrito **fora** do caminho síncrono de resposta. | Latência do PDP passaria a ser latência de escrita. |

## 6. Modelo de domínio (entidades persistidas)

```
   ┌──────────────────┐ 1      * ┌──────────────┐
   │  PolicyBundle    │──────────│  PolicyRule  │
   │ - urn            │          │ - ruleKey    │
   │ - bundleVersion  │          │ - effect     │
   │ - state (FSM)    │          │ - priority   │
   │ - compiledDigest │          │ - condition  │
   │ - signature      │          │ - obligations│
   └───┬──────────┬───┘          └──────────────┘
       │ 1      * │ 1          *
       │          └──────────────┐
       ▼                          ▼
   ┌──────────┐ *      1   ┌──────────────┐
   │   Role   │◀───────────│ RoleBinding  │
   │ - roleKey│  role_key  │ - subjectUrn │
   │ - inherits           │ - scopePattern│
   └──────────┘            │ - notAfter   │
                            └──────────────┘

   ┌──────────────────┐   ┌───────────────────┐   ┌─────────────────┐
   │ PolicyException  │   │  DecisionRecord   │   │ AttributeSource │
   │ - expiresAt (NN) │   │ - decisionId      │   │ - criticality   │
   │ - approvedBy     │   │ - reasonCode      │   │ - sensitive     │
   └──────────────────┘   │ - bundleVersion   │   └─────────────────┘
                           └───────────────────┘
```

Detalhamento físico (tipos, constraints, índices) em `./Database.md`.

## 7. Referências

- Componentes: `./Architecture.md` §3.1 · FSM: `./StateMachine.md`
- Contratos externos: `./API.md` · `./Events.md`
- Fonte única de verdade: `./_DESIGN_BRIEF.md` §2 e §5
- Testes das fronteiras: `./Testing.md` §3

*Fim de `ClassDiagrams.md`.*
