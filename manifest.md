---
id: MANIFEST
type: meta
title: Manifesto / Índice do produto
status: approved
updated: 2026-06-04
owner: silvioubaldino
---

# Manifesto — Mapa da Documentação

> Ponto de entrada para humanos e IAs. Mantenha sincronizado com os arquivos.

## Estado do produto
- **Produto:** ag-benefits — clube de benefícios local por assinatura
- **Repos:** context (este) · api · web · mobile
- **Fase atual:** Requisitos → Design (PROD e REQ em draft; AYDs a iniciar)

## Grafo de documentos
| Camada | ID | Documento | Status | Refina | Detalhado por |
|--------|----|-----------|--------|--------|----------------|
| Produto      | PROD-001 | Visão & estratégia | draft   | —        | REQ-001 |
| Requisitos   | REQ-001  | Requisitos (MVP)   | draft   | PROD-001 | (AYDs a definir) |
| Design       | AYD-001  | (a iniciar)        | —       | REQ-001  | — |
| Roadmap      | ROAD-001 | Roadmap            | draft   | PROD-001 | — |
| Decisão prod | PDR-001  | Cálculo do Savings | accepted | —       | — |
| Glossário    | GLO      | Linguagem ubíqua   | approved | —       | — |

## Ordem de leitura para a IA
1. `_meta/conventions.md` (regras, ciclo de vida, propagação) →
2. esta tabela →
3. a camada relevante para a tarefa (ver `CLAUDE.md`).

## Diagrama de relações
```
PROD-001
   ├─ REQ-001 ─ AYD-001 ─┬─ SPEC-001@api ─ PLAN-001@api
   │                     ├─ SPEC-001@web ─ PLAN-001@web
   │                     └─ SPEC-001@mobile ─ PLAN-001@mobile
   └─ ROAD-001
(PDR / ADR / GLO referenciados transversalmente por todos)
```
