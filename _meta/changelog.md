---
id: META-changelog
type: meta
title: Changelog do repo de contexto
status: approved
updated: 2025-01-01
owner: <nome>
---

# Changelog — Contexto

Registro cronológico de mudanças nos docs compartilhados (PROD, REQ, AYD, ROAD, decisões).
É aqui que mora a auditoria do "porquê" dos documentos **vivos**.

## [Não lançado]
- Documentação inicializada a partir do scaffold.

## [2026-06-04]
### Adicionado / Alterado
- **GLO** preenchido com a linguagem ubíqua do ag-benefits (Assinante, Parceiro,
  Assinatura, Contrato de Parceria, Praça, Benefício, Modalidade de Resgate, Resgate,
  Economia, Catálogo). Resolvida a ambiguidade de "cupom" → canonizado **Benefício**;
  "código de cupom" é uma Modalidade de Resgate. (status: approved)
- **PROD-001** preenchido (visão, problema two-sided, personas/JTBD, posicionamento,
  princípios e anti-escopo). North Star = Assinantes ativos pagantes (MRR); Economia e
  Resgates como métricas de input. (status: draft)
- **manifest.md** atualizado: nome do produto, fase atual e grafo de documentos.
### Decisões (registradas como contexto; PDRs formais a criar se necessário)
- North Star = MRR (assinantes ativos pagantes).
- Foco inicial: equilíbrio dos dois lados, concentrado por Praça (cidade/região piloto).
- Assinatura: mensal única, preço fixo.
- MVP sem cobrança ao Parceiro (moeda = engajamento mútuo).
### Propagação
- REQ-001 permanece em draft, a ser preenchido a seguir (refina PROD-001).

### Idioma & nomenclatura (mesma data)
- **conventions.md §8** adicionada: documentação em português, código/entidades em inglês.
  GLO passa a definir o **termo canônico em inglês**; demais docs referenciam esse termo.
- **GLO** traduzido: termos canônicos em inglês (`Subscriber`, `Partner`,
  `PartnerOperator`, `Subscription`, `SubscriptionStatus`, `PartnershipContract`,
  `Region`, `Benefit`, `Redemption`, `Savings`, `Catalog`), definições em português.
- **`RedemptionMethod` removido** do GLO: o mecanismo de resgate (QR/código/app) vira
  decisão de negócio posterior; um `Benefit` terá **um** mecanismo, não vários simultâneos.
- Mantido **`Subscriber`** (vs. `User`) para o ator consumidor.
- **PROD-001** alinhado para referenciar os termos canônicos em inglês.

<!--
## [AAAA-MM-DD]
### Adicionado / Alterado / Removido
### Decisões (PDR/ADR)
### Propagação (docs marcados como review/superseded)
-->
