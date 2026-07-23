---
Documento: NonFunctionalRequirements
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0214, ADR-0216, ADR-0217
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: _DESIGN_BRIEF.md §7.2, 024-Observability, 027-Cluster, 035-Benchmark
---

# 021-Security — Requisitos Não-Funcionais

Metas derivadas de `./_DESIGN_BRIEF.md` §7.2. **Os valores numéricos são normativos** e
aparecem idênticos em `./Monitoring.md`, `./Benchmark.md`, `./FailureRecovery.md`,
`./Scalability.md` e `./Configuration.md`.

---

## 1. Tabela de Requisitos Não-Funcionais

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (validação) | p99 de **introspecção** ≤ **10 ms**. A validação local por JWKS no `../004-API/` **não** consulta este módulo. | `aios_sec_introspect_latency_ms`. |
| NFR-002 | Desempenho (emissão) | p99 de emissão de token ≤ **50 ms**; de certificado de workload ≤ **150 ms**. | `aios_sec_issue_latency_ms{kind}`. |
| NFR-003 | Desempenho (segredo) | p99 de leitura de segredo com *lease* ≤ **20 ms**. | `aios_sec_secret_read_latency_ms`. |
| NFR-004 | Throughput | ≥ **5.000 emissões/s** e ≥ **20.000 introspecções/s** por réplica. | `aios_sec_issue_total` (taxa); cenários T-1/T-2 de `./Benchmark.md`. |
| NFR-005 | Disponibilidade | ≥ **99,99%** — o módulo é dependência de *bootstrap* de todo o sistema. | Uptime probe; *error budget* de 4,4 min/mês. |
| NFR-006 | Propagação de revogação | Revogação efetiva em **≤ 30 s** em todos os verificadores. | Tempo entre `token.revoked` e a primeira recusa observada. |
| NFR-007 | Escalabilidade | ≥ **10⁶** princípios e ≥ **10⁷** credenciais ativas, com custo sub-linear. | Teste de escala em `./Scalability.md` §7. |
| NFR-008 | Confidencialidade | **0** ocorrências de material secreto em log, evento, métrica ou resposta de API. | Varredura automatizada contínua (FR-018). |
| NFR-009 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min**; CA raiz recuperável de custódia *offline*. | DR drill; cerimônia de recuperação de raiz. |
| NFR-010 | Vida curta de credencial | TTL mediano de credencial ativa ≤ **24 h**; **0** credenciais sem expiração. | `aios_sec_credential_ttl_hours` (histograma). |
| NFR-011 | Rotação de chaves | **100%** das chaves rotacionadas dentro do período configurado. | `aios_sec_key_age_days` comparado à política. |
| NFR-012 | Observabilidade | **100%** das operações com trace OTel e auditoria correlacionada, **sem** material sensível. | Cobertura de spans + varredura de conteúdo. |
| NFR-013 | Idempotência | Repetições de mutação produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key`. |
| NFR-014 | Resistência a força bruta | Contenção ativa em ≤ **5** tentativas falhas por princípio por minuto. | `aios_sec_authn_failures_total`; cenário de contenção. |
| NFR-015 | Integridade de perfil | **100%** dos perfis de sandbox consumidos com assinatura verificada pelo `../007-Agent-Runtime/`. | Teste de contrato entre módulos. |

---

## 2. Orçamento de Erro e Priorização

| SLO | Orçamento mensal | Política ao esgotar |
|-----|------------------|---------------------|
| Disponibilidade 99,99% (NFR-005) | **4,4 min** | Congelamento de qualquer mudança no módulo; incidente de revisão obrigatória. |
| Confidencialidade (NFR-008) | **0 violações** | Incidente P1; rotação imediata do material exposto; expurgo do artefato que vazou. |
| Vida curta (NFR-010) | **0 credenciais sem expiração** | Incidente P1; a credencial é revogada, não "regularizada". |
| Propagação ≤ 30 s (NFR-006) | 1% das revogações fora da meta | Investigação do caminho de propagação (Outbox → NATS → cache). |
| Integridade de perfil (NFR-015) | **0 violações** | Incidente P1: perfil não verificado significa isolamento não garantido. |

O orçamento de disponibilidade deste módulo é **cinco vezes menor** que o dos demais
(4,4 min contra 21,9 min) porque sua indisponibilidade impede o *bootstrap* de todo o
sistema. A contrapartida arquitetural que torna essa meta alcançável é justamente
NFR-001: como a validação não passa por aqui, uma janela de indisponibilidade do 021
degrada a emissão, mas não interrompe o que já está autenticado.

Regra de priorização sob conflito: **confidencialidade > integridade > disponibilidade
> latência**. Em segurança, servir errado é pior que não servir.

---

## 3. Atributos por Categoria

### 3.1 Desempenho
NFR-001 a NFR-004. A decisão arquitetural determinante é ADR-0211: tokens são JWT
assinados, validados **localmente**. Sem isso, 10⁶ agentes gerariam uma chamada ao 021
por requisição, e nenhuma meta de latência do AIOS seria alcançável.

### 3.2 Disponibilidade e Recuperação
NFR-005 e NFR-009. Três réplicas *stateless*, estado no PostgreSQL e material no
KMS/HSM. A CA raiz não é um serviço: é custódia *offline*, e sua recuperação é
cerimônia documentada, não procedimento automatizado.

### 3.3 Segurança
NFR-008, NFR-010, NFR-011, NFR-014 e NFR-015 são as metas que definem o módulo.
NFR-010 merece destaque: "TTL mediano ≤ 24 h" e "**0** credenciais sem expiração" são
medidas contínuas, não auditorias anuais — uma credencial eterna criada hoje aparece na
métrica hoje.

### 3.4 Escalabilidade
NFR-007. 10⁶ princípios geram ~10⁷ credenciais ativas. Sustentado por validação local,
TTL curto (o conjunto ativo é pequeno em relação ao histórico) e particionamento de
`security.credential` por `expires_at`.

### 3.5 Observabilidade
NFR-012 tem uma exigência dupla e incomum: cobertura **total** de spans e auditoria,
**e** ausência total de material sensível neles. As duas metas puxam em direções
opostas, e é por isso que ambas são medidas.

---

## 4. Verificação — exemplo de consulta de SLI

```promql
# NFR-002 — p99 de emissão de certificado de workload
histogram_quantile(0.99,
  sum by (le) (rate(aios_sec_issue_latency_ms_bucket{kind="workload_cert"}[15m]))
) < 150

# NFR-010 — nenhuma credencial ativa sem expiração (deve ser sempre 0)
aios_sec_credentials_without_expiry > 0

# NFR-010 — TTL mediano das credenciais ativas
histogram_quantile(0.5, sum by (le) (rate(aios_sec_credential_ttl_hours_bucket[24h]))) > 24

# NFR-006 — propagação de revogação
histogram_quantile(0.99, sum by (le) (rate(aios_sec_revocation_propagation_ms_bucket[1h]))) > 30000

# NFR-011 — chave fora do período de rotação
aios_sec_key_age_days{role="signing"} > 90
```

---

## 5. Rastreabilidade

| NFR | Requisitos funcionais relacionados | Documento de verificação |
|-----|-----------------------------------|--------------------------|
| NFR-001, NFR-002, NFR-003, NFR-004 | FR-001, FR-003, FR-004, FR-008 | `./Benchmark.md` §4–§5 |
| NFR-005 | todos | `./Monitoring.md` §4 |
| NFR-006 | FR-006, FR-014 | `./Benchmark.md` §6 |
| NFR-007 | FR-004, FR-005 | `./Scalability.md` §7 |
| NFR-008 | FR-018 | `./Testing.md` §8 |
| NFR-009 | FR-009 | `./FailureRecovery.md` §5 |
| NFR-010 | FR-007, FR-019 | `./Metrics.md`, `./Testing.md` |
| NFR-011 | FR-002, FR-009 | `./Monitoring.md` §3 |
| NFR-012 | FR-017, FR-018 | `./Logging.md` |
| NFR-013 | FR-003..FR-006 | `./Testing.md` §4 |
| NFR-014 | FR-013 | `./Testing.md` §7 |
| NFR-015 | FR-010 | `./Testing.md` §5 (contrato com `007`) |

---

## 6. Referências

- Brief: `./_DESIGN_BRIEF.md` §7.2
- Requisitos funcionais: `./FunctionalRequirements.md`
- Monitoramento: `./Monitoring.md` · Métricas: `./Metrics.md`
