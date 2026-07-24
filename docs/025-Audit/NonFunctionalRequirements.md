---
Documento: NonFunctionalRequirements
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0252, ADR-0253, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §7.2, 005-Database, 021-Security, 024-Observability
---

# 025-Audit — Requisitos Não-Funcionais

> Metas idênticas às de `./_DESIGN_BRIEF.md` §7.2. Os mesmos números aparecem em
> `./Monitoring.md`, `./Benchmark.md`, `./FailureRecovery.md`, `./Configuration.md` e
> `./Scalability.md` — divergência numérica entre documentos é defeito.

## 1. Tabela de requisitos não-funcionais

| ID | Atributo | Meta (SLO) | SLI / método de verificação | Teste |
|----|----------|-----------|-----------------------------|-------|
| NFR-001 | Desempenho (escrita) | p99 de `AppendRecord` **durável** ≤ **20 ms**. | `aios_aud_append_latency_ms`. | T-AUD-101 |
| NFR-002 | Throughput | ≥ **50.000 registros/s** por réplica. | Benchmark `./Benchmark.md` §3. | T-AUD-102 |
| NFR-003 | Disponibilidade | ≥ **99,95%**. | Uptime probe; *error budget* mensal. | T-AUD-103 |
| NFR-004 | Latência de selagem | **100%** dos registros selados em ≤ **60 s**. | `aios_aud_seal_lag_s`. | T-AUD-104 |
| NFR-005 | Completude | **100%** dos eventos de classes registradas viram registro; lacuna detectada em ≤ **5 min**. | Contagem produtor × trilha; `aios_aud_sequence_gap_total`. | T-AUD-105 |
| NFR-006 | Durabilidade | **RPO = 0** para registro confirmado; **RTO ≤ 15 min**. | Teste de crash; DR drill. | T-AUD-106 |
| NFR-007 | Integridade | **100%** da cadeia verificável; adulteração detectada em ≤ **1 ciclo** (default 15 min). | Injeção controlada; `aios_aud_chain_break_total = 0`. | T-AUD-107 |
| NFR-008 | Verificação completa | Cadeia de **10⁹** registros verificável em ≤ **4 h**. | Benchmark de verificação. | T-AUD-108 |
| NFR-009 | Escalabilidade | ≥ **10⁹** registros por tenant e ≥ **10⁶** agentes produtores. | Teste de escala `./Scalability.md` §5. | T-AUD-109 |
| NFR-010 | Retenção | **100%** de aderência à retenção legal por classe; **0** expurgo sob *legal hold*. | Auditoria de expurgo. | T-AUD-110 |
| NFR-011 | Apagamento criptográfico | Efetivo em ≤ **24 h** do pedido; **0** payload legível após a destruição da chave. | Teste de decifragem pós-apagamento. | T-AUD-111 |
| NFR-012 | Latência de consulta | p95 por `subject_ref`/`decision_ref` ≤ **2 s** em 10⁹ registros. | `aios_aud_query_latency_ms`. | T-AUD-112 |
| NFR-013 | Exportação | Pacote de **10⁶** registros gerado em ≤ **30 min** e verificável offline. | Benchmark de exportação. | T-AUD-113 |
| NFR-014 | Idempotência | Repetições produzem efeito único em **100%** dos casos. | Teste de replay. | T-AUD-114 |
| NFR-015 | Isolamento de tenant | **0** consultas ou exportações que atravessem a fronteira de tenant. | RLS + validação estrutural; `AIOS-AUD-0003`. | T-AUD-115 |
| NFR-016 | Confidencialidade | **0** ocorrências de conteúdo auditado em evento, métrica ou log. | Varredura contínua (FR-023). | T-AUD-116 |
| NFR-017 | Observabilidade | **100%** das operações com trace OTel correlacionado. | Cobertura de spans (`../024-Observability/`). | T-AUD-117 |

## 2. Atributos de qualidade por categoria

| Categoria | NFRs | Comentário |
|-----------|------|-----------|
| **Durabilidade** | NFR-006 | O requisito que define o módulo: **RPO = 0**. Nenhum outro módulo do AIOS tem essa meta. |
| **Integridade** | NFR-007, NFR-008 | Imutabilidade verificável, não declarada. |
| **Completude** | NFR-005 | A lacuna é o defeito mais perigoso; medi-la é obrigatório. |
| Desempenho | NFR-001, NFR-002, NFR-012 | A escrita paga o preço da durabilidade síncrona por desenho. |
| Disponibilidade | NFR-003 | Indisponibilidade **atrasa** a trilha; não a perde (produtores têm outbox). |
| Conformidade | NFR-010, NFR-011 | Retenção legal e direito ao esquecimento coexistem via *crypto-shredding*. |
| Escalabilidade | NFR-009 | Cadeia particionada; arquivamento do histórico frio. |
| Segurança e privacidade | NFR-015, NFR-016 | Isolamento por tenant; conteúdo nunca fora da trilha. |
| Verificabilidade externa | NFR-013 | O auditor não precisa confiar no AIOS. |

## 3. Por que a escrita pode custar 20 ms

```
   p99 de AppendRecord ≤ 20 ms (durável)
   ├── AuditIngestGateway (mTLS, validação, dedupe) ...........  2 ms
   ├── RecordNormalizer (envelope canônico) ..................  1 ms
   ├── PayloadCipher (envelope com chave do titular) .........  2 ms
   ├── ChainAppender (seq + hash + INSERT) ...................  3 ms
   ├── COMMIT com replicação síncrona confirmada ............. 10 ms  ◀── o custo do RPO 0
   └── margem ................................................  2 ms

   Comparação deliberada:
     024-Observability  → ACK após ENFILEIRAMENTO   (p99 ≤ 50 ms, RPO 1 min)
     025-Audit          → ACK após DURABILIDADE     (p99 ≤ 20 ms, RPO 0)
```

Os dois módulos fazem escolhas opostas pelo mesmo motivo: cada um otimiza a garantia
que o seu domínio exige. Telemetria pode perder e não pode atrasar o emissor;
auditoria não pode perder e **pode** custar milissegundos ao produtor — que, por sua
vez, escreve no seu outbox e segue.

## 4. A meta que não admite degradação

| NFR | Degradação aceitável? | Justificativa |
|-----|-----------------------|---------------|
| NFR-001 (latência) | Sim, temporariamente | Produtor tem outbox; atraso é recuperável |
| NFR-002 (throughput) | Sim | Backlog drena |
| NFR-003 (disponibilidade) | Sim | Indisponibilidade atrasa, não perde |
| **NFR-006 (RPO = 0)** | **Não** | Registro perdido é irrecuperável e indetectável |
| **NFR-007 (integridade)** | **Não** | Adulteração não detectada invalida toda a trilha |
| NFR-010 (0 expurgo sob *hold*) | **Não** | Destruição de prova sob obrigação legal |

## 5. Riscos e trade-offs

| Risco | Impacto | Mitigação | NFR afetado |
|-------|---------|-----------|-------------|
| Durabilidade síncrona limita o throughput | Produtores enfileiram | Lote na API; partições paralelas; escala horizontal | NFR-001, NFR-002 |
| Cadeia particionada não dá ordem total | Reconstrução cronológica exige `occurred_at` | Ordem causal por `trace_id`; cada partição é verificável isoladamente | NFR-007 |
| Janela entre registro e selo | Adulteração ainda não coberta por assinatura | `prev_hash` já detecta alteração local; selo de 60 s fecha a janela | NFR-004, NFR-007 |
| Volume de 10⁹ registros | Consulta e verificação lentas | Índices por `subject_ref`/`decision_ref`; verificação em réplica; arquivamento WORM | NFR-008, NFR-012 |
| Chave de titular perdida por engano | Registro ilegível para sempre | Destruição exige comprovante e passa por PEP; *hold* bloqueia | NFR-011 |
| PDP indisponível | Trilha não pode ser lida | *Fail-closed* deliberado: a ingestão continua; a leitura espera | NFR-003 |

## 6. Referências

- Fonte: `./_DESIGN_BRIEF.md` §7.2
- Métricas citadas: `./Metrics.md` · Alertas: `./Monitoring.md`
- Metodologia de medição: `./Benchmark.md` · Testes: `./Testing.md`
- Rastreabilidade: `./Requirements.md`

*Fim de `NonFunctionalRequirements.md`.*
