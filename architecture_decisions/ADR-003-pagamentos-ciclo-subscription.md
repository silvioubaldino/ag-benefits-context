---
id: ADR-003
type: adr
title: Pagamentos e ciclo de Subscription via Asaas
status: accepted
created: 2026-06-08
updated: 2026-06-08
owner: silvioubaldino
affects: [api, mobile]
parents: []
related: [ADR-001, ADR-002, PROD-001, REQ-001, PDR-001, GLO]
tags: [pagamentos, asaas, subscription, webhooks, mvp]
superseded_by: null
---

# ADR-003: Pagamentos e ciclo de `Subscription` via Asaas

> Append-only: nunca reescreva. Decisão nova = novo ADR que substitui este.

## Contexto
O RF-02/RF-03/RF-04 exigem contratar, manter e cancelar uma `Subscription` mensal de **preço
fixo**, com o ciclo de `SubscriptionStatus` (`active`/`past_due`/`canceled`) **dirigido pelo
pagamento**. O produto **não é meio de pagamento** (PROD-001) e o RNF-03 proíbe armazenar
dados de cartão (PCI delegado). O mercado é brasileiro: a recorrência relevante é
**cartão** e **Pix Automático** (recorrência via Pix do BACEN, em operação desde 2025);
**boleto não suporta cobrança recorrente automática** (é avulso/assíncrono). Decisão
cross-repo (api expõe/serve, mobile inicia o checkout) → ADR, dentro da topologia do
[ADR-001](ADR-001-topologia-cross-repo.md).

## Decisão
Usamos **Asaas** como gateway de pagamento, integrado **atrás de uma abstração
`PaymentGateway`** na api (porta/anti-corruption layer) para manter o fornecedor
**substituível**. A api é a dona do ciclo de `Subscription`; o Asaas é a **fonte da verdade
de billing**, e nós **espelhamos** o `SubscriptionStatus`.

**Métodos de pagamento:**
- **Cartão** e **Pix Automático** → motores de **recorrência** da `Subscription`.
- **Boleto** → apenas **avulso/opcional** (nunca como motor de recorrência).

**Abstração `PaymentGateway` (porta na api):** a api fala com uma interface de domínio
(`createCustomer`, `createSubscription`, `cancelSubscription`, `handleWebhook`), e Asaas é
**um adaptador**. Nenhuma regra de domínio depende de detalhe do Asaas; trocar de gateway é
implementar outro adaptador.

**Fluxo (contrato cross-repo):**
1. O **mobile** inicia a contratação; a api garante um *customer* no Asaas e cria a
   assinatura recorrente (cartão ou Pix Automático), guardando o `gateway_customer_id` e o
   `gateway_subscription_id` no nosso DB. **Não** trafega nem armazena PAN — tokenização no
   gateway (RNF-03).
2. O **Asaas** notifica eventos de pagamento/assinatura por **webhook** para a api.
3. A api **valida a assinatura/token do webhook**, processa de forma **idempotente**
   (RNF-02) e **transiciona o `SubscriptionStatus`** local conforme o evento.
4. Cancelamento (RF-04): a api chama `cancelSubscription` no gateway; o estado `canceled`
   só vale ao fim do ciclo pago.

**Mapa de estados (contrato) — evento do gateway → `SubscriptionStatus`:**

| Situação no Asaas | `SubscriptionStatus` | Efeito |
|-------------------|----------------------|--------|
| Pagamento confirmado/recebido | `active` | Habilita `Redemption` (RN-01) |
| Cobrança vencida / falha de pagamento | `past_due` | Bloqueia `Redemption` |
| Regularização do pagamento | `active` | Reabilita `Redemption` |
| Assinatura cancelada (fim do ciclo) | `canceled` | Acesso encerrado |

`active` é o **único** estado que habilita `Redemption` (RN-01). O estado local é sempre
**derivado** do gateway; em caso de divergência, **reconciliamos** a partir do Asaas
(RNF-06, "confiança no número").

## Alternativas consideradas
| Opção | Prós | Contras | Por que (não) escolhida |
|-------|------|---------|-------------------------|
| **Asaas** (esta) | Recorrência nativa (cartão + Pix Automático + boleto); taxas/adquirência BR; IP autorizada BACEN; custo/simplicidade | Lock-in (mitigado por abstração); PII no fornecedor | **Escolhida** |
| Iugu | Forte em gestão de assinatura | Sem ganho decisivo sobre Asaas no MVP; custo | Descartada — reavaliável via abstração |
| Stripe | DX/SDK excelentes; passou a ter Pix recorrente (2026) | Mais caro no BR; boleto só avulso | Descartada para o MVP |
| Boleto como recorrência | Familiar ao usuário | **Não** há débito recorrente automático; confirmação lenta; quebra o ciclo do RF-03 | Descartada — boleto fica avulso |

## Consequências / trade-offs
- **Positivas:** recorrência aderente ao BR (cartão + Pix Automático); ciclo de
  `Subscription` dirigido por webhooks com idempotência; fornecedor substituível pela
  abstração; nenhum dado de cartão no nosso storage.
- **Negativas:** dependência do Asaas e da maturidade do Pix Automático; webhooks exigem
  endpoint resiliente, validação de assinatura e reconciliação; latência do Pix (janela de
  captura/retentativa) na ativação inicial.
- **Compliance (LGPD/PCI, RNF-03/RNF-04):** PCI delegado ao gateway; PII de pagamento no
  Asaas — registrar base legal (candidato a PDR/nota de compliance, ver ADR-001).
- **Impacto (IDs/repos afetados):** define o contrato de billing para `api` (porta
  `PaymentGateway` + adaptador Asaas + webhooks + mapa de estados) e `mobile` (início do
  checkout); realiza o cálculo de `Savings` do [PDR-001] no fluxo de `Redemption` (a
  `Subscription` apenas habilita o uso). Endpoints/payloads detalhados vão no AYD de
  billing/`Subscription`, não aqui.
