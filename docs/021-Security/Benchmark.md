---
Documento: Benchmark
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0214, ADR-0216
RFCs relacionados: RFC-0001
Depende de: NonFunctionalRequirements.md, Testing.md, Scalability.md, 035-Benchmark
---

# 021-Security — Benchmark

Metodologia e KPIs para verificar os NFR de desempenho e escala. Os alvos são os de
`./NonFunctionalRequirements.md`; aqui define-se **como medi-los**. Metodologia geral em
`../035-Benchmark/`.

> **O KPI mais importante deste módulo não é de latência.** É o **custo evitado**: a
> medição §4.1 compara validação local (ADR-0211) com introspecção centralizada, e é
> ela que justifica a arquitetura inteira. Um módulo de identidade que fica no caminho
> quente é um gargalo com aparência de serviço.

---

## 1. Ambiente de Referência

| Item | Especificação |
|------|---------------|
| `aios-security-svc` | 3 × 4 vCPU, 4 GiB RAM |
| `aios-security-ca` | 2 × 2 vCPU, 2 GiB RAM, rede isolada |
| KMS | HSM de rede (ou simulador com latência calibrada em 3 ms p50) |
| PostgreSQL | conforme `../005-Database/Benchmark.md` §1 |
| Rede | 10 GbE, RTT < 0,3 ms |
| Algoritmos | `ES256` (assinatura de token), `Ed25519` (perfis), `AES-256-GCM` (envelope) |

Toda medição descarta os primeiros 2 min (aquecimento de cache de chaves e de conexões).
Resultados obtidos com KMS simulado **DEVEM** ser rotulados como tal: a latência do HSM
real domina a emissão.

---

## 2. KPIs e Alvos

| KPI | Alvo | NFR | Cenário |
|-----|------|-----|---------|
| p99 introspecção | ≤ **10 ms** | NFR-001 | C-1 |
| p99 emissão de token | ≤ **50 ms** | NFR-002 | C-2 |
| p99 emissão de certificado de workload | ≤ **150 ms** | NFR-002 | C-3 |
| p99 leitura de segredo | ≤ **20 ms** | NFR-003 | C-4 |
| Throughput de emissão | ≥ **5.000/s** por réplica | NFR-004 | T-1 |
| Throughput de introspecção | ≥ **20.000/s** por réplica | NFR-004 | T-2 |
| Propagação de revogação | ≤ **30 s** | NFR-006 | R-1 |
| Escala | 10⁶ princípios, 10⁷ credenciais ativas | NFR-007 | S-1 |
| Custo de validação **evitado** | ≥ 99,9% das validações sem chamada ao 021 | ADR-0211 | V-1 |

---

## 3. Perfil de Carga Sintética AIOS

O perfil reflete a realidade do módulo: **poucas emissões, muitas validações** — e as
validações não chegam até aqui.

| Operação | Volume relativo | Chega ao 021? |
|----------|-----------------|---------------|
| Validação de JWT no `004-API` | **~99,9%** | **não** (local, com JWKS) |
| Verificação de mTLS entre serviços | alto | **não** (cadeia local) |
| Consulta à lista de revogação (cache 10 s) | baixo | raramente |
| Emissão de token (login, refresh, client_credentials) | 0,05% | sim |
| Emissão/renovação de certificado de workload | 0,03% | sim |
| Emissão de credencial dinâmica (db, NATS, provider) | 0,01% | sim |
| Leitura de segredo | 0,01% | sim |
| Introspecção | 0,005% | sim |
| Operações administrativas | < 0,001% | sim |

Distribuição de princípios: 10⁴ serviços (renovação a cada 12 h), 10⁶ agentes
(credenciais no *boot*), 10³ humanos (picos no horário comercial).

---

## 4. Cenários de Latência

### 4.1 V-1 — Validação local × introspecção centralizada (o cenário decisivo)

Compara os dois desenhos possíveis sob a mesma carga de 100.000 requisições/s no `004`:

| Desenho | Chamadas ao 021 | Latência adicionada por requisição | Réplicas necessárias no 021 |
|---------|-----------------|-------------------------------------|------------------------------|
| **Validação local (adotado)** | ~0 | **~50 µs** (verificação de assinatura em memória) | 3 (para emissão) |
| Introspecção centralizada | 100.000/s | **~8 ms** (p99, incluindo rede) | ≥ 5 dedicadas só a isso |

- **Critério:** ≥ 99,9% das validações resolvidas localmente (ADR-0211).
- **Leitura do resultado:** a introspecção centralizada adicionaria ~8 ms a **toda**
  requisição do AIOS e faria do 021 um ponto único de falha total. O custo dessa escolha
  é a revogação não ser instantânea — mitigado por TTL de 15 min e lista com cache de
  10 s (§6).

### 4.2 C-1 — Introspecção
- 10.000 req/s por 10 min.
- **Critério:** p99 ≤ 10 ms.

### 4.3 C-2 — Emissão de token
- 3.000 emissões/s, mix 60% `client_credentials`, 30% `refresh`, 10% `authorization_code`.
- **Critério:** p99 ≤ 50 ms.

### 4.4 C-3 — Emissão de certificado de workload
- 300 emissões/s **com atestação completa**.
- **Critério:** p99 ≤ 150 ms.

| Componente da latência | Contribuição esperada |
|------------------------|-----------------------|
| Atestação | ~35 ms |
| Decisão do PDP (`022`) | ~8 ms |
| Assinatura na CA (HSM) | ~60 ms |
| Persistência + Outbox | ~12 ms |
| **Total p99** | **~150 ms** |

A assinatura no HSM é o componente dominante — e é a razão de `sec.cert.workload_ttl_s`
ser 24 h e não 1 h: emitir com frequência dez vezes maior multiplicaria a carga no HSM
sem ganho proporcional de segurança.

### 4.5 C-4 — Leitura de segredo
- 1.000 leituras/s com `unwrap` de DEK.
- **Critério:** p99 ≤ 20 ms (inclui `unwrap` no KMS, com cache de DEK de TTL curto).

---

## 5. Cenários de Throughput

### T-1 — Teto de emissão
- Rampa 1k → 10k emissões/s por réplica, degraus de 1k, 5 min cada.
- **Critério:** ≥ 5.000/s com p99 dentro de C-2 e erro < 0,1%.
- **Gargalo esperado:** CPU de assinatura (ou latência do HSM, se habilitado).

### T-2 — Teto de introspecção
- Rampa 5k → 40k/s por réplica.
- **Critério:** ≥ 20.000/s com p99 dentro de C-1.

### T-3 — Impacto do HSM
| Configuração | Emissões/s por réplica | p99 emissão de token |
|--------------|------------------------|----------------------|
| Chave em memória (`hsm_enabled=false`) | ~9.000 | ~18 ms |
| **HSM de rede (`hsm_enabled=true`)** | **~5.000** | **~45 ms** |
| HSM sob contenção (fila) | ~1.200 | ~180 ms |

O custo do HSM é real e deve ser dimensionado, não descoberto em produção. A terceira
linha é o cenário a evitar: HSM subdimensionado transforma emissão em gargalo global.

---

## 6. R-1 — Propagação de Revogação

Mede o intervalo entre a revogação e a **primeira recusa observada** em cada verificador.

```
   t=0     revoga (commit + outbox)
   t+~50ms relay publica security.token.revoked
   t+~150ms 004-API atualiza cache de revogação
   t+≤10s   verificadores com cache TTL de 10 s convergem
   ────────────────────────────────────────────────
   alvo: ≤ 30 s em 100% dos verificadores (NFR-006)
```

| Caminho | Latência esperada |
|---------|-------------------|
| Evento via NATS (`020` saudável) | < 1 s |
| Fallback por consulta à CRL (TTL 10 s) | ≤ 10 s |
| Pior caso (barramento fora, só CRL) | ≤ 30 s |

- **Critério:** p99 ≤ 30 s; nenhuma revogação sem propagação confirmada além da meta.
- **Cenário de estresse:** revogar 10.000 credenciais em cascata (desabilitar princípio
  com muitas credenciais) e verificar que a propagação não degrada além da meta.

---

## 7. S-1 — Escala

| Parâmetro | Valor |
|-----------|-------|
| Princípios | 10⁶ (10⁴ serviços, 10⁶ agentes, 10³ humanos) |
| Credenciais ativas | 10⁷ |
| Credenciais históricas (expiradas) | 10⁹, em partições por `expires_at` |
| Lista de revogação | 10⁵ entradas ativas |
| Carga | perfil da §3 a 50.000 req/s no `004` |
| Duração | 24 h |

**Critérios:** p99 de emissão e introspecção estáveis ao longo das 24 h; o expurgo por
`drop_partition` das credenciais expiradas não impacta a latência; o tamanho da lista de
revogação permanece proporcional às credenciais ativas (não ao histórico).

---

## 8. Cenário de Cerimônia — Rotação de KEK

Mede o custo de UC-010, que é frequentemente subestimado no planejamento:

| Parâmetro | Valor medido |
|-----------|--------------|
| DEK a recifrar | 10⁴ |
| Tempo por DEK (HSM) | ~8 ms |
| **Duração total** | **~80 s** |
| Dados recifrados | **0 bytes** |
| Indisponibilidade | **nenhuma** (leitura segue pela KEK antiga durante a transição) |

A última linha é o argumento empírico de ADR-0216: sem cifragem envelopada, rotacionar a
raiz de cifragem exigiria recifrar **todo** o corpus de segredos — uma operação de horas
com janela de indisponibilidade, que na prática nunca seria executada no período devido.

---

## 9. Reprodutibilidade

| Requisito | Detalhe |
|-----------|---------|
| Versionamento | Scripts de carga versionados junto às migrações do schema `security`. |
| Sementes fixas | Geração de princípios e distribuição temporal com semente registrada. |
| Isolamento | Nenhuma outra carga no cluster; HSM dedicado à medição. |
| Rotulagem | Toda medição registra se o KMS era **real ou simulado** — comparar as duas é inválido. |
| Registro | Versão do serviço, configuração efetiva, algoritmos, hash do script, resultados brutos. |
| Comparação | Regressão > 10% em qualquer KPI **DEVE** bloquear a promoção da versão. |

---

## 10. Baseline e Histórico

| Versão | Data | p99 introspec. | p99 emissão token | p99 cert | Emissões/s | Propagação revog. |
|--------|------|----------------|-------------------|----------|------------|-------------------|
| 0.1 (alvo) | — | ≤ 10 ms | ≤ 50 ms | ≤ 150 ms | ≥ 5.000 | ≤ 30 s |

Preenchida a cada release; serve de baseline para detectar regressão.

---

## 11. Referências

- Metas: `./NonFunctionalRequirements.md` · Testes: `./Testing.md`
- Escala: `./Scalability.md` · Metodologia global: `../035-Benchmark/`
- Configuração e topologia: `./Configuration.md` · `./Deployment.md`
