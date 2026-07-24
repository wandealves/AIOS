---
Documento: Testing
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit + QA
ADRs relacionados: ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0255, ADR-0257, ADR-0258, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Requirements.md
---

# 025-Audit — Estratégia de Testes

## 1. O que é difícil de testar aqui

Este módulo tem duas falhas que **não se manifestam** em teste funcional comum:

1. **Perda de registro** — o teste "escreveu e leu" passa; o que falha é o caso raro
   de crash entre o `ACK` e o commit. Exige teste de crash, não de caminho feliz.
2. **Adulteração não detectada** — nenhum teste de comportamento a revela; exige
   **injeção adversarial** deliberada em cada camada de defesa.

Por isso a estratégia enfatiza três famílias que, em outros módulos, seriam
secundárias: **testes de crash**, **testes adversariais de integridade** e
**verificação offline por terceiro**.

## 2. Pirâmide e cobertura-alvo

| Nível | Escopo | Cobertura-alvo | Ferramenta |
|-------|--------|----------------|-----------|
| Unitário | `ChainAppender`, `SealService`, `IntegrityVerifier`, `PayloadCipher`, `LegalHoldManager`, `RetentionEnforcer` | ≥ 90% de linhas; **100%** dos ramos da FSM | xUnit |
| Integração | Pipeline + PostgreSQL (com réplica síncrona) + MinIO (*object lock*) | ≥ 85% | Testcontainers |
| Contrato | Envelope canônico (RFC-0250), eventos, API | 100% das operações de `./API.md` | Pact + JSON Schema |
| E2E | Fato → cadeia → selo → WORM → verificação → exportação | 100% dos UC-001..UC-015 | Compose + cenários |
| **Crash** | Durabilidade sob falha de nó/banco | RPO 0 | Chaos harness |
| **Adversarial** | Cada camada de `./Security.md` §3 | 5 camadas | Suíte dedicada |
| Carga | NFR-001..NFR-002, NFR-008, NFR-012 | — | k6 + gerador |
| **Verificação offline** | Pacote probatório validado por ferramenta independente | 100% | Validador de referência |

## 3. Testes por fronteira testável

As interfaces de `./ClassDiagrams.md` §2 são os pontos de injeção:

| Fronteira | Duplo de teste | O que se verifica |
|-----------|----------------|-------------------|
| `IAuditIngestGateway` | Banco com falha programável | Recusa (`503`) em vez de aceite sem durabilidade |
| `IChainAppender` | PostgreSQL real | `seq` sem lacuna sob concorrência; `prev_hash` correto |
| `ISealService` | KMS simulado com falha | Selagem retroativa; assinatura verificável |
| `IIntegrityVerifier` | Registros adulterados injetados | Detecção em ≤ 1 ciclo; comparação com WORM |
| `ILegalHoldManager` | *Holds* sobrepostos | Bloqueio de expurgo e apagamento; liberação parcial |
| `ICryptoShredder` | Chave destruível | Payload ilegível; cadeia íntegra |
| `IExportService` | Validador offline | Pacote verificável sem acesso ao AIOS |

## 4. Casos de teste — requisitos funcionais

| ID | Requisito | Cenário | Resultado esperado |
|----|-----------|---------|--------------------|
| T-AUD-001 | FR-001 | 10⁴ eventos, 30% reentregues pelo JetStream | 0 duplicatas; `seq` contínuo; `duplicates_total` reflete as reentregas |
| T-AUD-002 | FR-002 | Banco indisponível durante 100 escritas | **100% `503 AIOS-AUD-0005`**; 0 registros gravados; produtor reenvia com sucesso |
| T-AUD-003 | FR-003 | Fatos das 12 classes registradas | 100% com `actor`, `action`, `resourceUrn`, `outcome`, `traceId` |
| T-AUD-004 | FR-004 | 10⁵ escritas concorrentes em 16 partições | `prev_hash` correto em 100%; nenhuma cadeia com furo |
| T-AUD-005 | FR-005 | Selagem de 1.000 intervalos | Selo em ≤ 60 s; assinatura verificável com a chave pública |
| T-AUD-006 | FR-006 | Alterar `payload_digest` de um registro (via SQL com *trigger* removido) | `chain_break` detectado em ≤ 1 ciclo; WORM diverge do banco |
| T-AUD-007 | FR-007 | Remover 3 registros de uma partição | `sequence_gap` em ≤ 5 min, apontando `expectedSeq`/`observedSeq` |
| T-AUD-008 | FR-008 | Classe `contains_pii=true` sem `legal_basis` | `422 AIOS-AUD-0010` |
| T-AUD-009 | FR-009 | *Hold* com `approved_by = requested_by` | `403 AIOS-AUD-0009`; `CHECK` do banco também rejeita |
| T-AUD-010 | FR-010 | Expurgo e RTBF sobre escopo em *hold* | `423 AIOS-AUD-0008` em ambos; **0** payloads removidos |
| T-AUD-011 | FR-011 | RTBF de um titular com 3.000 registros | Payload indecifrável; `record_hash` e `payload_digest` inalterados |
| T-AUD-012 | FR-012 | Verificação completa **após** T-AUD-011 | Cadeia 100% íntegra com registros `Shredded` |
| T-AUD-013 | FR-013 | Comprovante de apagamento | `receipt_hash` reproduzível; `chain_record_id` existe na trilha |
| T-AUD-014 | FR-014 | Sobrescrever objeto arquivado | **Rejeitado pelo MinIO** (*object lock*) |
| T-AUD-015 | FR-015 | Consulta com tenant divergente | `403 AIOS-AUD-0003`; RLS bloqueia mesmo com bypass de aplicação |
| T-AUD-016 | FR-016 | 50 consultas à trilha | 50 registros `audit.access` gerados, com solicitante e escopo |
| T-AUD-017 | FR-017 | Pacote de 10⁴ registros validado por **ferramenta externa** | Verificação passa **sem** acesso ao AIOS |
| T-AUD-018 | FR-018 | Injetar cada um dos 6 `anomaly.kind` | Evento emitido com `kind` e `severity` corretos |
| T-AUD-019 | FR-019 | `UPDATE` direto em `audit.record` | Exceção do *trigger*; `AIOS-AUD-0016` pela API |
| T-AUD-020 | FR-020 | Consulta e exportação sem capability | Negadas pelo PDP; **as negações aparecem na trilha** |
| T-AUD-021 | FR-021 | Crash entre commit e publicação | Evento publicado após recuperação; nenhum perdido |
| T-AUD-022 | FR-022 | Evento de classe não registrada | `AIOS-AUD-0002`; DLQ; demais eventos do lote seguem |
| T-AUD-023 | FR-023 | Varredura de eventos, métricas e logs | **0** ocorrências de payload auditado fora da trilha |

## 5. Casos de teste — requisitos não-funcionais

| ID | NFR | Método | Critério |
|----|-----|--------|----------|
| T-AUD-101 | NFR-001 | Carga sustentada com réplica síncrona | p99 ≤ 20 ms |
| T-AUD-102 | NFR-002 | Carga por réplica isolada | ≥ 50.000 registros/s |
| T-AUD-103 | NFR-003 | Injeção de falhas + probe | ≥ 99,95% na janela |
| T-AUD-104 | NFR-004 | Carga contínua por 2 h | 100% selados em ≤ 60 s |
| T-AUD-105 | NFR-005 | Produtor parado por 24 h | `class_silent` em ≤ 30 min; lacuna em ≤ 5 min |
| T-AUD-106 | NFR-006 | **Crash do primário no meio de 10.000 escritas** | **100%** dos `201` recebidos estão na trilha; RPO 0 |
| T-AUD-107 | NFR-007 | Adulteração injetada em cada camada | Detecção em ≤ 1 ciclo em todas |
| T-AUD-108 | NFR-008 | Verificação completa de 10⁹ registros | ≤ 4 h |
| T-AUD-109 | NFR-009 | 10⁹ registros, 16 partições | Latência de escrita e consulta dentro do SLO |
| T-AUD-110 | NFR-010 | Expurgo com *holds* ativos | 100% de aderência; **0** expurgo sob *hold* |
| T-AUD-111 | NFR-011 | RTBF ponta a ponta | ≤ 24 h; 0 payload legível após |
| T-AUD-112 | NFR-012 | Consulta por `subject_ref` em 10⁹ registros | p95 ≤ 2 s |
| T-AUD-113 | NFR-013 | Exportação de 10⁶ registros | ≤ 30 min; validável offline |
| T-AUD-114 | NFR-014 | Replay com mesmo `Idempotency-Key` | Efeito único; **mesmo `record_hash`** |
| T-AUD-115 | NFR-015 | Consulta e exportação cross-tenant | Bloqueadas em ambas |
| T-AUD-116 | NFR-016 | Varredura de telemetria | **0** conteúdo auditado |
| T-AUD-117 | NFR-017 | Amostragem de spans | 100% das operações com trace |

## 6. Testes adversariais de integridade

Uma família própria, executada a cada release, cobrindo as cinco camadas de
`./Security.md` §3:

| # | Ataque simulado | Camada testada | Detecção esperada |
|---|-----------------|----------------|-------------------|
| A-01 | `UPDATE` de campo via API | 1 (interface) | Operação **não existe**; `AIOS-AUD-0016` |
| A-02 | `UPDATE` direto no banco | 2 (*trigger*) | Exceção do *trigger*; nenhuma linha alterada |
| A-03 | Remover o *trigger* e alterar um registro | 3 (cadeia) | `chain_break` em ≤ 1 ciclo |
| A-04 | Recomputar a cadeia inteira após alterar | 4 (selo) | Raiz de Merkle diverge do selo assinado |
| A-05 | Forjar um selo com chave própria | 4 (assinatura) | `signed_by` desconhecido; verificação da assinatura falha |
| A-06 | Substituir a cópia WORM | 5 (*object lock*) | Sobrescrita rejeitada pelo armazenamento |
| A-07 | `DELETE` de registro no banco | 2 + 3 | *Trigger* impede; se forçado, `sequence_gap` detecta |
| A-08 | Apagar via RTBF para destruir evidência sob investigação | I5 | `423 AIOS-AUD-0008`; tentativa registrada |
| A-09 | Reduzir `retention.min_days` em runtime | Config não-recarregável | Ignorado; log `Error` + métrica |
| A-10 | Alterar a serialização canônica sem nova RFC | 4 + 5 | `worm_mismatch` em massa; rollback obrigatório |

> A-03 a A-06 são os testes que provam a tese do módulo: **imutabilidade verificável,
> não declarada** (ADR-0251). Se qualquer um passar sem detecção, a trilha não vale
> como prova e o release **não** pode sair.

## 7. Verificação offline (o teste que o auditor faria)

```
   1. Exportar pacote de 10⁴ registros (UC-014)
   2. Copiar para ambiente ISOLADO, sem rede para o AIOS
   3. Executar o validador de referência:
        · recomputa record_hash de cada registro (serialização canônica RFC-0250)
        · confere prev_hash encadeado
        · recomputa a raiz de Merkle pelo caminho fornecido
        · verifica a assinatura do selo com a chave pública do pacote
        · confere a cadeia de selos (prev_seal_hash)
   4. Critério: verificação passa 100% SEM nenhuma chamada ao AIOS
```

O validador de referência é mantido junto com a RFC-0250 e é o mesmo artefato entregue
ao auditor externo. Se ele precisar de qualquer acesso ao AIOS para concluir, o desenho
falhou — a premissa é que o auditor **não confia** no operador.

## 8. Fixtures e ambientes

| Fixture | Conteúdo |
|---------|----------|
| `classes-baseline` | 12 classes auditáveis registradas (uma por produtor típico) |
| `chain-1M` | 10⁶ registros encadeados e selados, em 16 partições |
| `chain-1G` | 10⁹ registros para NFR-008/NFR-009 e T-AUD-112 |
| `chain-tampered` | Cópia de `chain-1M` com adulteração conhecida em posição conhecida |
| `holds-overlapping` | *Holds* sobrepostos, com e sem prazo |
| `subjects-rtbf` | Titulares com registros distribuídos por várias classes e partições |
| `export-golden` | Pacote probatório de referência, com resultado de verificação conhecido |

| Ambiente | Uso | Diferenças de produção |
|----------|-----|------------------------|
| Local (Compose) | Unidade/integração | 1 partição, `seal_interval_s=10`, `object_lock_days=1`, PDP `open` |
| CI efêmero | Todos os níveis exceto carga longa | **Réplica síncrona obrigatória**; `require_durable_ack=true` |
| Homologação | E2E, crash, adversarial, carga | **Idêntico** a produção (`./Configuration.md` §11) |

> `aud.ingest.require_durable_ack = true` em **todos** os ambientes, inclusive local
> (`./Configuration.md` §11). Um desenvolvedor que aprende a conviver com perda
> silenciosa em desenvolvimento não reconhece o defeito em produção.

## 9. Gates de qualidade (CI)

Um PR **NÃO DEVE** ser mesclado se qualquer item falhar:

- [ ] Unitários ≥ 90%; **100%** dos ramos da FSM.
- [ ] Nenhum teste de contrato quebrado (Pact).
- [ ] **`T-AUD-002` e `T-AUD-106` verdes** — durabilidade: recusar em vez de perder.
- [ ] **A-01 a A-10 verdes** — todas as camadas adversariais detectam.
- [ ] `T-AUD-010` verde — *legal hold* bloqueia apagamento e expurgo.
- [ ] `T-AUD-012` verde — cadeia íntegra após *crypto-shredding*.
- [ ] `T-AUD-017` verde — **verificação offline por ferramenta independente**.
- [ ] `T-AUD-019` verde — `UPDATE` direto rejeitado pelo *trigger*.
- [ ] `T-AUD-023` verde — nenhum conteúdo auditado fora da trilha.
- [ ] Benchmark sem regressão > 10% vs. baseline (`./Benchmark.md`).
- [ ] Migrações validadas: *trigger* de imutabilidade presente após aplicar.
- [ ] **Serialização canônica inalterada** — mudança exige nova versão de RFC-0250.
- [ ] Nenhum `TODO` nos 26 documentos do módulo.

O penúltimo item é um gate incomum e deliberado: um teste de CI compara o
`record_hash` de um vetor fixo com o valor esperado publicado na RFC-0250. Se mudar,
todo o histórico deixa de verificar — e o build precisa parar antes disso.

## 10. Referências

- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Rastreabilidade: `./Requirements.md` §5 · Casos de uso: `./UseCases.md`
- Fronteiras testáveis: `./ClassDiagrams.md` §2 · Caos: `./FailureRecovery.md` §8
- Defesa em profundidade: `./Security.md` §3 · Desempenho: `./Benchmark.md`

*Fim de `Testing.md`.*
