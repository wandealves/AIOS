---
Documento: RFC Index
Módulo: 003-RFC
Status: Stable
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
---

# AIOS — Requests for Comments (RFC)

RFCs especificam propostas técnicas de forma completa, no estilo IETF (RFC 2119/
8174 para palavras-chave normativas). Enquanto ADRs registram *decisões pontuais*,
RFCs especificam *sistemas/protocolos/contratos inteiros* e passam por comentário
público antes de estabilizarem.

## Ciclo de vida

`Draft` → `Discussion` → `Last Call` → `Accepted` → (`Obsoleted by RFC-XXXX`).

## Estrutura obrigatória (estilo IETF)

1. Abstract
2. Status & terminologia (RFC 2119)
3. Motivação
4. Terminologia
5. Especificação (normativa)
6. Considerações de segurança
7. Considerações de privacidade (LGPD/GDPR)
8. Considerações de IANA/registro (registros do AIOS)
9. Compatibilidade e versionamento
10. Referências (normativas/informativas)
11. Apêndices (exemplos)

## Índice

| RFC | Título | Status |
|-----|--------|--------|
| [0001](RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted |

RFCs futuras (fan-out) especificarão: protocolo de syscalls cognitivas (Kernel),
formato de plano (Planning), envelope de eventos NATS, contrato A2A do AIOS,
esquema de memória hierárquica, contrato do Model Router, entre outros. Cada uma é
linkada do `RFC.md` do módulo correspondente.
