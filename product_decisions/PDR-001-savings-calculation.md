---
id: PDR-001
type: pdr
title: Cálculo do Savings a partir do valor de compra informado pelo Subscriber
status: accepted
created: 2026-06-04
updated: 2026-06-04
owner: silvioubaldino
parents: []
related: [PROD-001, REQ-001, GLO]
tags: [savings, redemption, metrics, mvp]
superseded_by: null
---

# PDR-001: Cálculo do `Savings` a partir do valor informado pelo `Subscriber`

> Append-only: nunca reescreva. Decisão nova = novo PDR que substitui este.

## Contexto / problema de produto
O produto promete mostrar a economia real — "você já economizou R$ X" (`Savings`) — como
prova de valor e alavanca de retenção (input do North Star, MRR). Porém o ag-benefits
**não é meio de pagamento** e **não processa a compra** (ver anti-escopo do [PROD-001](../product.md)),
então não temos o valor gasto de forma automática. Sem resolver a origem do `Savings`,
não é possível fechar o contrato do `Redemption` (REQ-001, RN-05). Decisão necessária
agora porque ela molda o fluxo e o payload do `Redemption` (AYD a seguir).

## Decisão
No ato do `Redemption` (após o QR resolver `Partner`+`Benefit` no servidor), o
`Subscriber` **informa o valor da compra**; o sistema calcula e **congela** o `Savings`.

- Cada `Benefit` carrega uma **regra de desconto**: `discount_type` ∈
  {`percentage`, `fixed_amount`} + `discount_value`.
- Apuração do `Savings`:
  - `percentage` → `Savings = purchase_amount × discount_value` (exige o valor da compra).
  - `fixed_amount` → `Savings = discount_value` (valor da compra é **opcional**, coletado
    só para métricas de ticket, se houver).
- O `Savings` é gravado no `Redemption` no instante do uso e **não é recalculado** depois
  (REQ-001, RN-03).
- O `purchase_amount` é **autodeclarado**. Para preservar o princípio "confiança no número":
  - aplica-se **teto de `Savings` por `Benefit`** e validação de plausibilidade do valor
    informado (limites detalhados no PDR de antifraude — RN-04);
  - na comunicação ao `Subscriber`, o agregado é apresentado como **economia realizada
    pelos seus usos** (baseada nos valores que ele declarou), não como valor auditado de
    terceiros.

## Trade-offs
- **Ganhamos:** cobertura dos dois tipos de oferta mais comuns (percentual e valor fixo)
  com um único fluxo; número de economia "vivo" e personalizado por uso.
- **Abrimos mão de:** zero-fricção no balcão — descontos percentuais exigem 1 input do
  `Subscriber` (mitigado: para `fixed_amount` o valor é opcional).
- **Risco:** valor autodeclarado pode ser inflado/errado → exige tetos e plausibilidade
  (antifraude) e assume-se imprecisão residual no agregado.

## Impacto em métricas
- Habilita o KR "`Savings` médio/`Subscriber` ≥ preço da `Subscription`" (PROD-001, Obj.3).
- `Savings` é métrica de **input** do North Star (MRR via retenção); sendo autodeclarado,
  sua exatidão depende dos guarda-fias antifraude — monitorar distribuição de
  `purchase_amount`/`Savings` para detectar abuso.

## Alternativas descartadas
- **`Savings` fixo declarado por `Benefit` (sem valor de compra):** simples e sempre
  consistente, mas **não cobre descontos percentuais**, comuns no varejo local.
- **Faixa/valor médio estimado por `Benefit`:** sem fricção, mas número aproximado —
  conflita com "confiança no número" e enfraquece o argumento de valor.
- **Híbrido como dois fluxos distintos de UX:** redundante; a regra de desconto por
  `Benefit` já unifica `percentage` e `fixed_amount` num só fluxo.
