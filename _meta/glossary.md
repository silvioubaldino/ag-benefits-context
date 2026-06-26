---
id: GLO
type: glossary
title: Linguagem ubíqua do domínio
status: approved
updated: 2026-06-26
owner: silvioubaldino
related: [PROD-001]
---

# Glossário (Linguagem Ubíqua)

Definições canônicas dos termos do domínio. Toda a documentação e o código — em
**todos os repos** — usam estes termos com este significado. É isto que faz api, web e
mobile "falarem a mesma língua": um termo, uma definição.

> Regra: adicione o termo aqui **antes** de usá-lo em outros docs ou no código.
> **Termo canônico em inglês** (vira código); definição em português. Inclua sinônimos a
> evitar — é onde a ambiguidade costuma virar bug ou contrato confuso.
> (ver `conventions.md` §8 — idioma: docs em PT, código/entidades em EN)

## Atores

| Termo (canônico) | Definição | Sinônimos a evitar |
|------------------|-----------|--------------------|
| **`Subscriber`** | Usuário final pessoa física com uma `Subscription` ativa, que acessa o app e realiza `Redemption`s. (PT: "assinante") | "user", "customer", "member", "usuário" |
| **`Partner`** | Comércio (estabelecimento/marca) com um `PartnershipContract` vigente, que disponibiliza `Benefit`s no app. (PT: "parceiro") | "store", "merchant", "advertiser", "loja" |
| **`PartnerOperator`** | Pessoa do `Partner` que valida/confirma um `Redemption` no ponto de atendimento (quando a modalidade exige). (PT: "operador do parceiro") | "attendant", "cashier", "atendente" |

## Negócio

| Termo (canônico) | Definição | Sinônimos a evitar |
|------------------|-----------|--------------------|
| **`Subscription`** | Vínculo de pagamento recorrente (mensal, preço fixo) que dá ao `Subscriber` o direito de uso do app e de realizar `Redemption`s. Tem um `SubscriptionStatus`. (PT: "assinatura") | "plan" (plano é a configuração; subscription é o vínculo), "mensalidade" |
| **`SubscriptionStatus`** | Estado do vínculo: `pending` (criada, aguardando confirmação do 1º pagamento), `active`, `past_due`, `canceled`. Só `active` habilita `Redemption`. (PT: "status da assinatura") | "situação", "state" genérico |
| **`PartnershipContract`** | Acordo comercial com um `Partner`. Define quais `Benefit`s o `Partner` disponibiliza e a vigência. No MVP, sem cobrança financeira ao `Partner` (troca de engajamento). (PT: "contrato de parceria") | "partnership" (informal), "agreement", "convênio" |
| **`Region`** | Recorte geográfico de operação (cidade/região piloto) onde `Subscriber`s e `Partner`s coexistem. Unidade de densidade do marketplace. (PT: "praça") | "area", "location", "local", "branch" |

## Oferta e uso (núcleo de métricas)

| Termo (canônico) | Definição | Sinônimos a evitar |
|------------------|-----------|--------------------|
| **`Benefit`** | A vantagem/desconto que um `Partner` disponibiliza ao `Subscriber`, definida no `PartnershipContract`. É a **oferta** exibida no app. (PT: "benefício") | **"coupon"/"cupom"** (cupom é apenas um possível mecanismo de resgate — a definir), "promotion", "offer", "product" |
| **`Redemption`** | **Evento** registrado de um `Subscriber` utilizando um `Benefit` em um `Partner`, num instante. É o fato central que gera métricas. Captura, no mínimo: `Subscriber`, `Benefit`, `Partner`, `Region`, instante e `Savings`. (PT: "resgate") | "use", "usage", "check-in", "uso" |
| **`Savings`** | Valor em R$ atribuído a um `Redemption` (quanto o `Subscriber` deixou de pagar). Base do indicador "você já economizou R$ X". (PT: "economia") | "discount" (desconto é a regra; Savings é o valor realizado), "economia" como campo |
| **`Catalog`** | Conjunto de `Benefit`s visíveis a um `Subscriber`, filtrado por `Region` e `SubscriptionStatus`. (PT: "catálogo") | "showcase", "vitrine", "lista de ofertas" |
