---
id: MANIFEST
type: meta
title: Manifesto / Índice do produto
status: approved
updated: 2026-06-22
owner: silvioubaldino
---

# Manifesto — Mapa da Documentação

> Ponto de entrada para humanos e IAs. Mantenha sincronizado com os arquivos.

## Estado do produto
- **Produto:** ag-benefits — clube de benefícios local por assinatura
- **Repos:** context (este) · api · web · mobile
- **Fase atual:** Design (PROD e REQ em draft; ADRs de fundação aceitos; AYD-001 aprovado;
  observabilidade — ADR-005/AYD-002 — em draft; billing/conformidade com lojas — ADR-006 —
  aceito: assinatura na **web**, que reentra no MVP; AYD-003 a iniciar)

## Grafo de documentos
| Camada | ID | Documento | Status | Refina | Detalhado por |
|--------|----|-----------|--------|--------|----------------|
| Produto      | PROD-001 | Visão & estratégia | draft   | —        | REQ-001 |
| Requisitos   | REQ-001  | Requisitos (MVP)   | draft   | PROD-001 | AYD-001 |
| Design       | AYD-001  | Onboarding do Subscriber (identidade/conta) | approved | REQ-001 | SPEC-001@api, SPEC-001@mobile |
| Design       | AYD-002  | Baseline de observabilidade | draft | REQ-001 | SPEC-002@api, SPEC-002@mobile |
| Design       | AYD-003  | Billing — contratação e ciclo da Subscription (web + Asaas) | draft | REQ-001 | SPEC-003@api, SPEC-003@web, SPEC-003@mobile |
| Roadmap      | ROAD-001 | Roadmap            | draft   | PROD-001 | — |
| Decisão prod | PDR-001  | Cálculo do Savings | accepted | —       | — |
| Decisão prod | PDR-002  | Antifraude/repetição do Redemption | accepted | — | — |
| Decisão arq  | ADR-001  | Topologia cross-repo (C4) | accepted | — | ADR-002, ADR-003, ADR-004 |
| Decisão arq  | ADR-002  | Autenticação (Firebase)   | accepted | — | — |
| Decisão arq  | ADR-003  | Pagamentos/Subscription (Asaas) | accepted | — | — |
| Decisão arq  | ADR-004  | Resolução do Redemption (QR TOTP) | accepted | — | — |
| Decisão arq  | ADR-005  | Observabilidade (logs/métricas/traces, stack gratuita) | draft | — | AYD-002 |
| Decisão arq  | ADR-006  | Conformidade de billing com lojas (assinatura na web; fallback IAP) | accepted | — | AYD-003 |
| Glossário    | GLO      | Linguagem ubíqua   | approved | —       | — |

## Ordem de leitura para a IA
1. `_meta/conventions.md` (regras, ciclo de vida, propagação) →
2. esta tabela →
3. a camada relevante para a tarefa (ver `CLAUDE.md`).

## Diagrama de relações
```
PROD-001
   ├─ REQ-001 ─┬─ AYD-001 (onboarding Subscriber) ─┬─ SPEC-001@api ─ PLAN-001@api
   │           │                                   └─ SPEC-001@mobile ─ PLAN-001@mobile
   │           ├─ AYD-002 (observabilidade baseline) ─┬─ SPEC-002@api ─ PLAN-002@api
   │           │                                      └─ SPEC-002@mobile
   │           └─ AYD-003 (billing/Subscription) ─┬─ SPEC-003@api
   │                                              ├─ SPEC-003@web
   │                                              └─ SPEC-003@mobile
   └─ ROAD-001
   (web reentra no MVP só para assinatura — ADR-006; AYDs futuros geram outras SPECs)

Fundação arquitetural (referenciada pelos AYDs):
   ADR-001 (topologia C4) ─┬─ ADR-002 (auth/Firebase)
                           ├─ ADR-003 (pagamentos/Asaas) ─ ADR-006 (billing nas lojas) ─ AYD-003
                           ├─ ADR-004 (Redemption: QR TOTP) ─ PDR-002 (antifraude)
                           └─ ADR-005 (observabilidade) ─ AYD-002
(PDR / ADR / GLO referenciados transversalmente por todos)
```
