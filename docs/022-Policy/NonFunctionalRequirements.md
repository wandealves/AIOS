---
Documento: NonFunctionalRequirements
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0010, ADR-0221, ADR-0222, ADR-0226
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §7.2, 024-Observability, 025-Audit, 005-Database
---

# 022-Policy — Requisitos Não-Funcionais

> Metas idênticas às de `./_DESIGN_BRIEF.md` §7.2. Os mesmos números aparecem em
> `./Monitoring.md`, `./Benchmark.md`, `./FailureRecovery.md`, `./Configuration.md` e
> `./Scalability.md` — divergência numérica entre documentos é defeito.

## 1. Tabela de requisitos não-funcionais

| ID | Atributo | Meta (SLO) | SLI / método de verificação | Teste |
|----|----------|-----------|-----------------------------|-------|
| NFR-001 | Desempenho (decisão) | p99 de `Evaluate` ≤ **5 ms** com atributos em cache; ≤ **15 ms** com resolução remota de atributo. | Histograma `aios_pol_decision_latency_ms{path}`. | T-POL-101 |
| NFR-002 | Throughput | ≥ **50.000 decisões/s** por réplica com bundle compilado em memória. | Benchmark `./Benchmark.md` §3. | T-POL-102 |
| NFR-003 | Disponibilidade | ≥ **99,99%** — o PDP está no caminho de toda ação privilegiada. | Uptime probe; *error budget* mensal. | T-POL-103 |
| NFR-004 | Propagação de política | Bundle ativo em **100%** das réplicas em ≤ **10 s**; invalidação de cache nos PEPs em ≤ **5 s** (p99). | Tempo entre `policy.bundle.updated` e a primeira decisão servida com a nova versão. | T-POL-104 |
| NFR-005 | Escalabilidade | ≥ **10⁶** agentes, ≥ **10⁴** regras por bundle e ≥ **10⁵** vínculos por tenant, com custo de decisão sub-linear. | Teste de escala `./Scalability.md` §5. | T-POL-105 |
| NFR-006 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min**; bundle ativo reconstruível a partir do registro assinado. | DR drill trimestral. | T-POL-106 |
| NFR-007 | Explicabilidade | **100%** das decisões `deny` com `reason_code` e regras casadas. | Auditoria do journal: contagem de `deny` sem motivo = 0. | T-POL-107 |
| NFR-008 | *Default deny* verificável | **0** decisões `allow` sem regra explícita casada. | Varredura contínua: `effect = allow ∧ matched_rules = {}` ⇒ defeito. | T-POL-108 |
| NFR-009 | Determinismo | Mesma requisição + mesma `bundle_version` + mesmos atributos ⇒ **mesma** decisão em **100%** das reproduções. | `ExplainDecision` comparado ao journal em amostragem contínua. | T-POL-109 |
| NFR-010 | Orçamento de avaliação | **100%** das avaliações concluem em ≤ `pol.decision.eval_budget_ms` (**50 ms**) **ou** retornam `deny`. | `aios_pol_eval_timeout_total`; nenhuma avaliação pendente sem resposta. | T-POL-110 |
| NFR-011 | Qualidade da política | Cobertura de regras exercitadas por teste ≥ **90%** antes de qualquer publicação. | `test_coverage` no gate de publicação. | T-POL-111 |
| NFR-012 | Simulação | Reavaliar **10⁶** requisições registradas em ≤ **10 min**, sem impacto no p99 de produção. | Benchmark de simulação com carga concorrente. | T-POL-112 |
| NFR-013 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key` (RFC-0001 §5.5). | T-POL-113 |
| NFR-014 | Observabilidade | **100%** das decisões com trace OTel e auditoria correlacionada em `../025-Audit/`, sem valor sensível. | Cobertura de spans; varredura de conteúdo. | T-POL-114 |
| NFR-015 | Integridade do bundle | **100%** dos bundles ativos com assinatura verificada na carga. | Verificação na ativação e no *startup* da réplica. | T-POL-115 |
| NFR-016 | Isolamento de tenant | **0** decisões que atravessem a fronteira de tenant. | RLS + validação estrutural; `AIOS-POL-0003` monitorado. | T-POL-116 |

## 2. Atributos de qualidade por categoria

| Categoria | NFRs | Comentário |
|-----------|------|-----------|
| Desempenho | NFR-001, NFR-002, NFR-010 | O caminho quente **NÃO DEVE** executar I/O de banco (`./Scalability.md` §2). |
| Escalabilidade | NFR-005, NFR-012 | Escala horizontal sem estado compartilhado. |
| Disponibilidade e recuperação | NFR-003, NFR-006 | Degradação graciosa preserva a avaliação por último (`./FailureRecovery.md` §5). |
| Segurança | NFR-008, NFR-015, NFR-016 | *Default deny* verificável é requisito de segurança, não de conveniência. |
| Observabilidade e auditoria | NFR-007, NFR-009, NFR-014 | ADR-0010: auditoria por construção. |
| Manutenibilidade da política | NFR-011 | O gate de cobertura é o que impede política sem prova de entrar em vigor. |
| Consistência | NFR-004, NFR-009, NFR-013 | Convergência de versão entre réplicas é condição de correção, não otimização. |

## 3. Orçamento de latência da decisão (p99, caminho quente)

```
   total ≤ 5 ms (atributos em cache)
   ├── DecisionGateway  (validação + correlação + limite) ......  0,4 ms
   ├── ActionNormalizer .......................................  0,1 ms
   ├── AttributeResolver (cache local hit) ....................  0,3 ms
   ├── RbacResolver (grafo de papéis em memória) ..............  1,0 ms
   ├── AbacEvaluator (expressões compiladas) ..................  1,7 ms
   ├── ObligationComposer + TTL ...............................  0,3 ms
   └── journal assíncrono (fora do caminho de resposta) .......  0,0 ms
                                                        margem:  1,2 ms

   com resolução remota de atributo: + até 10 ms (timeout rígido de 20 ms
   por fonte, com o orçamento total de avaliação limitado a 50 ms — NFR-010)
```

> O journal **NÃO DEVE** estar no caminho síncrono da resposta: gravar antes de
> responder trocaria o p99 do PDP pelo p99 do PostgreSQL, e o PDP está no caminho de
> toda ação privilegiada do AIOS.

## 4. Riscos e trade-offs

| Risco | Impacto | Mitigação | NFR afetado |
|-------|---------|-----------|-------------|
| Cache no PEP atrasa a revogação | Acesso indevido por até o TTL | TTL curto + `policy.decision.updated` granular | NFR-004 |
| Bundle grande degrada a avaliação | p99 acima do SLO | Limites de complexidade na compilação (`AIOS-POL-0016`) | NFR-001, NFR-005 |
| Fonte de atributo lenta | Decisões negadas por timeout | Timeout por fonte + cache + evento `attribute.degraded` | NFR-010 |
| Simulação concorrendo com produção | p99 degradado | Pool dedicado (`policy-simulator`) | NFR-012 |
| Journal como gargalo de escrita | Perda de amostragem de `allow` | Escrita assíncrona em lote + particionamento por tempo | NFR-007 |

## 5. Referências

- Fonte: `./_DESIGN_BRIEF.md` §7.2
- Métricas citadas: `./Metrics.md` · Alertas: `./Monitoring.md`
- Metodologia de medição: `./Benchmark.md` · Testes: `./Testing.md`
- Rastreabilidade: `./Requirements.md`

*Fim de `NonFunctionalRequirements.md`.*
