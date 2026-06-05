---
id: PROD-001
type: product
title: Visão & Estratégia
status: draft
created: 2026-06-04
updated: 2026-06-04
owner: silvioubaldino
parents: []
children: [REQ-001]
related: [GLO]
tags: [marketplace, beneficios, assinatura]
superseded_by: null
---

# Produto — Visão & Estratégia

> **ag-benefits** — clube de benefícios por assinatura: o `Subscriber` paga uma
> mensalidade fixa e ganha descontos em `Partner`s locais; cada `Redemption` é registrado
> para gerar métricas que sustentam renovação de contratos, novas parcerias e percepção
> de valor.

## Problema
- **`Subscriber`:** quer economizar no dia a dia, mas as vantagens de comércios locais são
  dispersas, pouco confiáveis e difíceis de descobrir. Não há um lugar único que
  concentre descontos reais e mostre **quanto já economizou** (`Savings`).
- **`Partner`:** quer atrair e fidelizar clientes, mas não tem um canal barato de
  divulgação nem **dados de uso** (quem usou, quando, quanto valeu) para decidir ofertas.
- É um problema **two-sided**: sem `Subscriber`s não há valor para o `Partner`; sem
  `Partner`s não há valor para o `Subscriber`. Resolver o cold-start exige densidade por
  **`Region`**.

## Visão (North Star, uma frase)
Ser o clube de benefícios local em que cada real de assinatura se paga em economia real e
mensurável — para o `Subscriber` e para o `Partner`.

## Proposta de valor
- **Para o `Subscriber`:** uma `Subscription` fixa que se paga sozinha — descontos reais
  em `Partner`s perto de você, com a conta clara de **quanto já economizou** (`Savings`).
- **Para o `Partner`:** engajamento e clientes recorrentes sem custo financeiro inicial,
  com **métricas de uso** (`Redemption`s, frequência, perfil) que hoje ele não tem.
- **Diferencial central:** tudo é **orientado a métricas** — todo `Redemption` é registrado
  (`Subscriber` × `Benefit` × `Partner` × `Region` × `Savings`), virando insumo para negócio.

## Personas & Jobs To Be Done
| Persona | Contexto | Job (quando… quero… para…) |
|---------|----------|----------------------------|
| **`Subscriber`** | Consumidor local, sensível a preço, usa o celular para decidir onde consumir | Quando vou gastar perto de mim, quero descontos confiáveis em um só lugar, para economizar de forma visível e previsível |
| **`Partner`** | Comércio local buscando movimento e fidelização, com pouco budget de marketing | Quando preciso atrair/reter clientes, quero um canal de baixo custo com dados de uso, para justificar e ajustar minhas ofertas |
| **`PartnerOperator`** | Atendente/caixa no balcão validando o uso | Quando um `Subscriber` apresenta um `Benefit`, quero confirmar o `Redemption` em segundos, para não travar o atendimento |
| **Negócio (interno)** | Time comercial/produto do ag-benefits | Quando renovo/vendo parcerias, quero métricas claras de `Redemption` e `Savings`, para negociar contratos e provar valor |

## Objetivos (OKRs / métrica North Star)
**North Star:** `Subscriber`s ativos pagantes (MRR).
**Métricas de input (leading):** `Savings` gerado (R$) e `Redemption`s por `Subscriber`/mês
— são o que sustenta a retenção da `Subscription` e, portanto, o MRR.

- **Objetivo 1 — Provar valor na `Region` piloto**
  - KR1: N `Subscriber`s ativos pagantes ao fim do piloto _(meta a definir no ROAD)_
  - KR2: Densidade mínima de `Partner`s ativos na `Region` _(meta a definir)_
- **Objetivo 2 — Criar hábito (engajamento)**
  - KR1: ≥ X `Redemption`s por `Subscriber` ativo/mês _(meta a definir)_
  - KR2: % de `Subscriber`s com ao menos 1 `Redemption` no mês (ativação de uso)
- **Objetivo 3 — Reter (sustentar o MRR)**
  - KR1: `Savings` médio/`Subscriber` ≥ preço da `Subscription` ("se paga")
  - KR2: Churn mensal de `Subscriber`s ≤ X%

## Posicionamento & diferencial
**Posicionamento:** "O clube de benefícios local que se paga — e prova, em reais, quanto
você economizou."

Diferenciais defensáveis:
1. **Camada de métricas como produto** — o `Redemption` é cidadão de primeira classe; os
   dados de uso são o ativo que fideliza `Partner`s e melhora a oferta (efeito de dados).
2. **Densidade por `Region`** — crescer os dois lados em lockstep regionalmente cria um
   marketplace difícil de replicar fora dali.
3. **Custo zero inicial ao `Partner`** — baixa fricção de aquisição do lado da oferta;
   a moeda é engajamento mútuo, não dinheiro.

## Princípios de produto
- **Orientado a métricas > funcionalidades vistosas** — se um fluxo não registra
  `Redemption` confiável, não entra.
- **Densidade local > alcance amplo** — vale mais uma `Region` densa que dez rarefeitas.
- **Simplicidade do balcão** — confirmar um `Redemption` não pode atrapalhar o atendimento.
- **Confiança no número** — "você economizou R$ X" (`Savings`) precisa ser sempre verdadeiro.
- **Contrato único** — regras de oferta e resgate nascem no `PartnershipContract`/AYD;
  serviços implementam, não reinventam (alinhado ao framework cross-repo).

## O que NÃO é (anti-escopo)
- Não é meio de pagamento nem carteira digital (não processa a compra do `Subscriber`).
- Não é cashback/crédito — a vantagem é o desconto no ato, não acúmulo de saldo.
- Não é marketplace de e-commerce — não vende produtos nem faz entrega.
- Não cobra do `Partner` no MVP (sem monetização do lado da oferta por enquanto).
- Não é programa de fidelidade por pontos (modelo de pontos está fora de escopo).
- Não atende B2B2C (assinatura via empresas) — foco em `Subscriber` PF na
  `Region` piloto.
