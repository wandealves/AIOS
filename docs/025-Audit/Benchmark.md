---
Documento: Benchmark
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0251, ADR-0252, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250
Depende de: NonFunctionalRequirements.md, Metrics.md, Testing.md, Scalability.md
---

# 025-Audit — Benchmark

## 1. Objetivo

Medir, de forma reprodutível, se a trilha sustenta as metas de
`./NonFunctionalRequirements.md` — em especial NFR-001 (p99 ≤ 20 ms **com
durabilidade**), NFR-002 (vazão), NFR-004 (selagem), NFR-008 (verificação de 10⁹ em
≤ 4 h), NFR-012 (consulta) e NFR-013 (exportação).

A pergunta que este benchmark responde é específica: **quanto custa a garantia?** O
p99 de 20 ms inclui replicação síncrona confirmada; a comparação relevante não é com
uma escrita assíncrona (que seria trivialmente mais rápida e não daria RPO 0), mas com
o orçamento que o produtor pode pagar sem prejudicar seu próprio SLO.

## 2. Ambiente de referência

| Item | Especificação |
|------|---------------|
| `audit-ingest` medido | 1 réplica isolada (4 vCPU, 6 GiB) |
| PostgreSQL | Primário 8 vCPU / 32 GiB + **réplica síncrona** em outra AZ, `synchronous_commit = remote_apply` |
| RTT primário ↔ réplica | ≤ 1 ms (mesma região, AZ distinta) |
| MinIO | 4 vCPU, 16 GiB, *object lock* habilitado |
| KMS (`../021-Security/`) | Simulado com latência de assinatura de 5 ms |
| Partições de cadeia | 16 (`aud.chain.partitions_per_tenant`) |
| Gerador de carga | k6 + gerador de eventos, nó separado |
| Fixtures | `classes-baseline`, `chain-1M`, `chain-1G` |

Toda medição descarta os primeiros 120 s e reporta mediana de 5 execuções.

## 3. Cargas e KPIs

| # | Carga | Perfil | KPI | Alvo |
|---|-------|--------|-----|------|
| B-01 | Escrita unitária (API) | 5.000 req/s, payload 1 KiB | p99 de `AppendRecord` **durável** | ≤ **20 ms** |
| B-02 | Escrita em lote | Lotes de 500 | registros/s por réplica | ≥ **50.000/s** |
| B-03 | Ingestão por evento | JetStream, 50.000 msg/s | registros/s; lag do consumidor | ≥ 50.000/s; lag < 30 s |
| B-04 | **Custo da durabilidade** | B-01 com `remote_apply` vs. `local` | Δ p99 | quantificar (não é alvo) |
| B-05 | Contenção de `seq` | 1, 4, 16, 64 partições | registros/s por partição | crescimento **quase linear** até 16 |
| B-06 | Selagem | 100.000 registros/min | `seal_lag` p99 | ≤ **60 s** |
| B-07 | Assinatura | 1 selo/s por partição | latência de assinatura | ≤ 50 ms |
| B-08 | Verificação por amostragem | 1.000 registros/ciclo | duração do ciclo | ≤ 30 s |
| B-09 | **Verificação completa** | `chain-1G` (10⁹ registros) | duração | ≤ **4 h** |
| B-10 | Consulta por `subject_ref` | `chain-1G` | p95 | ≤ **2 s** |
| B-11 | Consulta por `decision_ref` | `chain-1G` | p95 | ≤ 2 s |
| B-12 | Exportação | 10⁶ registros + selos + provas | duração; verificável offline | ≤ **30 min** |
| B-13 | Expurgo por retenção | 10⁷ payloads | duração; cadeia íntegra após | ≤ 2 h; verificação passa |
| B-14 | *Crypto-shredding* | Titular com 10⁵ registros | latência ponta a ponta | ≤ 24 h (NFR-011) |
| B-15 | Arquivamento WORM | 1.000 selos | `archive_lag` p99 | ≤ 1 h |

## 4. B-04 — quanto custa o RPO 0

```
   Mesma carga (B-01), variando apenas synchronous_commit:

   (a) local        → commit no primário, sem esperar réplica    → p99 ~4 ms
   (b) remote_write → réplica recebeu (não aplicou)              → p99 ~9 ms
   (c) remote_apply → réplica APLICOU (configuração de produção) → p99 ~14 ms

   Custo do RPO 0:  (c) − (a) ≈ 10 ms
```

Este número é **informativo, não um alvo a otimizar**. Ele existe para que a discussão
sobre "acelerar a auditoria" seja quantificada: reduzir para `local` ganharia 10 ms por
escrita e **eliminaria a garantia** que define o módulo. As opções (a) e (b) são
recusadas no *startup* (`./Configuration.md` §9, V-05); o benchmark as mede apenas
para documentar o trade-off.

## 5. B-05 — contenção de sequência por partição

```
   partições:      1        4       16       64
   registros/s:  ~4.100  ~16.800  ~52.000  ~58.000
                    │        │        │        │
                    └────────┴────────┴────────┴── crescimento quase LINEAR até 16;
                                                    a partir daí o COMMIT SÍNCRONO
                                                    do banco domina (§Scalability §5.1)
```

O joelho da curva em 16 partições é o que justifica o default de
`aud.chain.partitions_per_tenant`. Aumentar além disso troca contenção de `seq` por
complexidade de verificação, sem ganho proporcional — o limite passa a ser a camada 2.

## 6. Resultados-alvo (baseline de referência)

| KPI | Alvo | Baseline esperado | Limite de regressão |
|-----|------|-------------------|---------------------|
| B-01 p99 (durável) | ≤ 20 ms | ~14 ms | +15% |
| B-02 registros/s | ≥ 50.000 | ~62.000 | −10% |
| B-03 lag do consumidor | < 30 s | ~4 s | +50% |
| B-06 `seal_lag` p99 | ≤ 60 s | ~22 s | +30% |
| B-08 ciclo de amostragem | ≤ 30 s | ~11 s | +30% |
| B-09 verificação completa | ≤ 4 h | ~2 h 20 min | +25% |
| B-10 consulta p95 | ≤ 2 s | ~0,8 s | +30% |
| B-12 exportação | ≤ 30 min | ~18 min | +25% |
| B-13 expurgo | ≤ 2 h | ~50 min | +30% |
| **Cadeia íntegra após B-13** | 100% | 100% | **qualquer falha reprova** |
| **Verificação offline (B-12)** | passa | passa | **qualquer falha reprova** |

Dois KPIs têm tolerância **zero**: integridade da cadeia após o expurgo e verificação
offline do pacote. Ambos correspondem a garantias, não a metas de desempenho.

## 7. Metodologia da verificação completa (B-09)

```
   chain-1G: 10⁹ registros em 16 partições, ~62,5 M por partição
        │
        ├─ 16 verificadores paralelos, um por partição, em RÉPLICA DE LEITURA
        │      │
        │      ├─ streaming: lê em ordem de seq, mantém apenas o prev_hash anterior
        │      ├─ recomputa record_hash (serialização canônica RFC-0250)
        │      ├─ acumula folhas para recomputar a raiz de Merkle por selo
        │      └─ verifica a assinatura de cada selo (chave pública)
        │
        └─ amostragem cruzada com o WORM: 1% dos selos baixados e comparados
              (comparar 100% dobraria a duração sem aumentar materialmente a garantia)
```

A verificação é **O(n) com memória O(1) por partição**: nunca carrega a cadeia inteira.
Essa propriedade é o que a torna viável em 10⁹ registros — e é consequência direta de a
cadeia ser sequencial e o Merkle ser por intervalo, não global.

## 8. Comparação com alternativas

| Arranjo | p99 de escrita | RPO | Detecção de adulteração | Verificação de 10⁹ | Escolha |
|---------|----------------|-----|-------------------------|---------------------|---------|
| **Cadeia particionada + selo + WORM** (adotado) | ~14 ms | **0** | ≤ 1 ciclo | ~2 h 20 min | ADR-0251, ADR-0252 |
| Tabela append-only sem cadeia | ~4 ms | 0 | **nunca** | n/a | Descartado: imutabilidade não verificável |
| Assinatura por registro | ~65 ms | 0 | imediata | ~14 h | Descartado: custo de assinatura inviável a 50.000/s |
| Cadeia global única | ~14 ms | 0 | ≤ 1 ciclo | ~2 h | Descartado: ~4.100 registros/s (B-05) |
| Escrita assíncrona (fila → banco) | ~2 ms | ~30 s | ≤ 1 ciclo | ~2 h 20 min | Descartado: **RPO ≠ 0**, lacuna invisível |
| Âncora em blockchain público por registro | ~vários s | 0 | imediata | n/a | Descartado: latência e custo; mantido como âncora **opcional** por selo |

A terceira linha é a mais instrutiva: assinar cada registro daria detecção imediata em
vez de "≤ 1 ciclo", ao custo de **4,6×** na latência de escrita e **6×** na verificação.
O selo por intervalo entrega quase a mesma garantia com uma assinatura a cada 60 s.

## 9. Reprodutibilidade

| Item | Regra |
|------|-------|
| Fixtures | Versionadas (`chain-1M`, `chain-1G`, `export-golden`) com digest publicado |
| Sementes | Geração de carga com semente fixa |
| Configuração | Perfil de produção (`./Configuration.md` §11), **sempre** com réplica síncrona |
| Vetor canônico | O benchmark inclui o vetor fixo de `record_hash` da RFC-0250 — se mudar, o build para (`./Testing.md` §9) |
| Registro | Cada execução grava: versão do serviço, nº de partições, RTT da réplica, resultados brutos |
| Frequência | A cada release e semanalmente em homologação |
| Publicação | Resultados anexados ao release em `../035-Benchmark/` |

## 10. Interpretação dos resultados

| Observação | Diagnóstico provável | Ação |
|------------|----------------------|------|
| B-01 p99 alto, `commit_share` alto | Latência da réplica síncrona | Verificar rede/AZ; **nunca** relaxar `synchronous_commit` |
| B-01 p99 alto, `commit_share` baixo | CPU (hash + cifragem) | Escalar réplicas de `audit-ingest` |
| B-02 abaixo do alvo com CPU ociosa | Contenção de `seq` | Verificar nº de partições (B-05); aumentar exige migração |
| B-06 `seal_lag` alto | Latência de assinatura no KMS | Ver B-07; verificar `../021-Security/` |
| B-09 acima de 4 h | I/O da réplica de leitura | Paralelizar por partição; verificar se está lendo do primário por engano |
| B-10 lento | Índice de `subject_ref` não usado | Verificar plano; consulta sem janela temporal varre partições |
| B-13 com cadeia quebrada | **Bug grave**: o expurgo removeu linha em vez de payload | Bloqueia release; revisar `RetentionEnforcer` e o *trigger* |
| Vetor canônico divergente | Serialização alterada | **Bloqueia release**: invalidaria todo o histórico |

## 11. Referências

- Metas: `./NonFunctionalRequirements.md` · Métricas: `./Metrics.md`
- Testes: `./Testing.md` · Escala: `./Scalability.md`
- Metodologia global de benchmark: `../035-Benchmark/`

*Fim de `Benchmark.md`.*
