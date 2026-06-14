---
id: MANIFEST
type: meta
title: Manifesto / Índice do produto
status: approved
updated: 2026-06-14
owner: silvioubaldino
---

# Manifesto — Mapa da Documentação

> Ponto de entrada para humanos e IAs. Mantenha sincronizado com os arquivos.

## Estado do produto
- **Produto:** ag-benefits — clube de benefícios local por assinatura
- **Repos:** context (este) · api · web · mobile
- **Fase atual:** Design (PROD e REQ em draft; ADRs de fundação aceitos; AYD-001 em draft)

## Grafo de documentos
| Camada | ID | Documento | Status | Refina | Detalhado por |
|--------|----|-----------|--------|--------|----------------|
| Produto      | PROD-001 | Visão & estratégia | draft   | —        | REQ-001 |
| Requisitos   | REQ-001  | Requisitos (MVP)   | draft   | PROD-001 | AYD-001 |
| Design       | AYD-001  | Onboarding do Subscriber (identidade/conta) | draft | REQ-001 | SPEC-001@api, SPEC-001@mobile |
| Roadmap      | ROAD-001 | Roadmap            | draft   | PROD-001 | — |
| Decisão prod | PDR-001  | Cálculo do Savings | accepted | —       | — |
| Decisão prod | PDR-002  | Antifraude/repetição do Redemption | accepted | — | — |
| Decisão arq  | ADR-001  | Topologia cross-repo (C4) | accepted | — | ADR-002, ADR-003, ADR-004 |
| Decisão arq  | ADR-002  | Autenticação (Firebase)   | accepted | — | — |
| Decisão arq  | ADR-003  | Pagamentos/Subscription (Asaas) | accepted | — | — |
| Decisão arq  | ADR-004  | Resolução do Redemption (QR TOTP) | accepted | — | — |
| Glossário    | GLO      | Linguagem ubíqua   | approved | —       | — |

## Ordem de leitura para a IA
1. `_meta/conventions.md` (regras, ciclo de vida, propagação) →
2. esta tabela →
3. a camada relevante para a tarefa (ver `CLAUDE.md`).

## Diagrama de relações
```
PROD-001
   ├─ REQ-001 ─ AYD-001 (onboarding Subscriber) ─┬─ SPEC-001@api ─ PLAN-001@api
   │                                             └─ SPEC-001@mobile ─ PLAN-001@mobile
   └─ ROAD-001
   (web fora do MVP do Subscriber; AYDs futuros geram outras SPECs)

Fundação arquitetural (referenciada pelos AYDs):
   ADR-001 (topologia C4) ─┬─ ADR-002 (auth/Firebase)
                           ├─ ADR-003 (pagamentos/Asaas)
                           └─ ADR-004 (Redemption: QR TOTP) ─ PDR-002 (antifraude)
(PDR / ADR / GLO referenciados transversalmente por todos)
```
