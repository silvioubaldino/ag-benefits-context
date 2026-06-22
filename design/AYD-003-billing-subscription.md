---
id: AYD-003
type: design
title: Billing — contratação e ciclo da Subscription (web + Asaas)
status: draft
created: 2026-06-22
updated: 2026-06-22
owner: silvioubaldino
affects: [api, web, mobile]
parents: [REQ-001]
children: [SPEC-003@api, SPEC-003@web, SPEC-003@mobile]
related: [ADR-001, ADR-002, ADR-003, ADR-006, PDR-001, GLO]
tags: [mvp, billing, subscription, asaas, web, webhooks]
superseded_by: null
---

# AYD-003: Billing — contratação e ciclo da `Subscription` (web + Asaas)

> Análise & Design cross-repo. Decide QUAIS repos a feature toca, o PAPEL de cada um
> e os CONTRATOS entre eles. Daqui nascem N SPECs (uma por repo afetado).

## Objetivo

Atende **RF-02** (contratar `Subscription` mensal de preço fixo), **RF-03** (ciclo de
`SubscriptionStatus` dirigido por pagamento) e **RF-04** (cancelar). Resultado esperado: uma
pessoa com conta (`Subscriber` provisionado no AYD-001) **assina na web** via **Asaas**, a api
passa a refletir o `SubscriptionStatus` (`active`/`past_due`/`canceled`) dirigido por
**webhook**, e o **mobile lê** esse status para **liberar/bloquear** o `Redemption` (RN-01).

Materializa, em endpoints, o que já foi decidido em arquitetura:
- [ADR-003](../architecture_decisions/ADR-003-pagamentos-ciclo-subscription.md) — Asaas atrás
  da porta `PaymentGateway`; api dona do ciclo; Asaas fonte da verdade; mapa de estados.
- [ADR-006](../architecture_decisions/ADR-006-conformidade-billing-lojas.md) — **a venda mora
  na web**; o app mobile **não vende nem direciona** a contratação (conformidade com as
  lojas); o mobile é superfície grátis que **só lê** o status.

**Escopo:** contratação, leitura de status/recibo, cancelamento e o ciclo dirigido por
webhook — **na web e na api**; **leitura + gate** no mobile. **Cartão** é o método do MVP;
**Pix Automático** entra como seguimento (o contrato já carrega `billing_type`). Ver "questões
em aberto".

## Repos afetados e papéis

| Repo | Papel nesta feature | SPEC gerada |
|------|---------------------|-------------|
| web | **Superfície de venda** (ADR-006): `landing → criação de conta (Firebase) → contratação (inicia `POST /subscription`) → checkout Asaas → recibo/status → redireciona para as lojas`. Também o **cancelamento** (RF-04). Responsivo (mobile-first). | SPEC-003@web |
| api | **Dona do ciclo de `Subscription`.** Garante o *customer* e cria a assinatura recorrente no Asaas atrás da porta `PaymentGateway`; recebe e valida o **webhook**, processa **idempotente** e **transiciona** o `SubscriptionStatus` (mapa do ADR-003); expõe status/recibo e cancelamento; `GET /me` passa a carregar o `subscription_status` real. | SPEC-003@api |
| mobile | **Não vende, não linka** (ADR-006). **Lê** o `subscription_status` (via `GET /me`, contrato do AYD-001) e faz o **gate do `Redemption`** (oculta/desabilita para `status != active`), com mensagem neutra. | SPEC-003@mobile |

> A coleta de cartão **não** trafega pela nossa web nem pela api: usamos **checkout hospedado
> do Asaas** (URL/link de pagamento) — PCI delegado (RNF-03). Ver "questões em aberto" para a
> alternativa de tokenização na web.

## Contratos (fonte da verdade)

Chamadas autenticadas levam `Authorization: Bearer <Firebase ID token>` (ADR-002, `role:
subscriber`) — a **web** autentica no Firebase igual ao mobile. Preço/plano é **fixo e único**
no MVP (configuração da api), por isso não trafega no payload.

```
POST /subscription                         (web, autenticado)
  efeito: garante o customer no Asaas (idempotente) e cria a Subscription
          recorrente; retorna a URL de checkout hospedado p/ o 1º pagamento.
  req:  { billing_type: "credit_card" }    // "pix" reservado p/ fase 2
  res 201: {
    id: string,                            // id de domínio da Subscription
    status: "pending",                     // vira "active" via webhook do 1º pagamento
    billing_type: "credit_card",
    checkout_url: string,                  // página hospedada do Asaas
    current_period_end: string|null        // ISO-8601; null até confirmar
  }
  erros:
    401  token ausente/inválido
    409  já existe Subscription active/pending para o Subscriber (sem duplicar)
    422  billing_type inválido

GET /subscription                          (web, autenticado)
  efeito: estado atual + dados do recibo do último pagamento.
  res 200: {
    id, status, billing_type, current_period_end,
    last_payment: { paid_at: string, amount: number, receipt_url: string } | null
  }
  erros:
    401 ; 404 (sem Subscription)

DELETE /subscription                       (web, autenticado)  — RF-04
  efeito: cancelSubscription no gateway; acesso vale até o fim do ciclo pago.
  res 200: { status: "canceled", access_until: string }   // ISO-8601
  erros:
    401 ; 404 (sem Subscription ativa)

POST /webhooks/asaas                       (Asaas → api; SEM Firebase)
  auth: validação de assinatura/token do webhook (ADR-003); NÃO confiar no corpo cru.
  efeito: idempotente por id de evento do gateway; mapeia evento → SubscriptionStatus
          (mapa do ADR-003) e transiciona o estado local.
  res 200: ack após processar (reentrega não duplica efeito — RNF-02)
  erros:
    401  assinatura inválida
```

**`GET /me` (AYD-001) — campo agora preenchido:**
```
res 200: { ..., subscription_status: "active" | "past_due" | "canceled" | null, ... }
```
- `null` = conta sem `Subscription` (nunca assinou). É o estado em que o **mobile esconde o
  `Redemption`**. É o **único** caminho do mobile para o billing — ele **não** chama
  `POST/GET/DELETE /subscription`.

**Mapa de estados (do ADR-003, aqui materializado):**

| Evento Asaas (webhook) | `SubscriptionStatus` | Efeito no mobile |
|------------------------|----------------------|------------------|
| Pagamento confirmado/recebido | `active` | `Redemption` habilitado (RN-01) |
| Cobrança vencida / falha | `past_due` | `Redemption` bloqueado |
| Regularização | `active` | `Redemption` reabilitado |
| Assinatura cancelada (fim do ciclo) | `canceled` | acesso encerrado |

## Modelo de domínio afetado

Introduz a entidade **`Subscription`** (GLO) como registro próprio, **1:1** com o
`Subscriber` no MVP (a relação `Subscriber → subscription_status` é **derivada** dela):

| Campo | Tipo | Origem | Notas |
|-------|------|--------|-------|
| `id` | id | nosso DB | chave de domínio |
| `subscriber_id` | id (FK) | `Subscriber` | dono da assinatura |
| `status` | enum | webhook (ADR-003) | `SubscriptionStatus`: `active`/`past_due`/`canceled` |
| `billing_type` | enum | `POST /subscription` | `credit_card` (MVP); `pix` (fase 2) |
| `gateway_customer_id` | string | Asaas | *customer* no gateway |
| `gateway_subscription_id` | string | Asaas | assinatura recorrente no gateway |
| `current_period_end` | timestamp | webhook | fim do ciclo pago (base do `access_until`) |
| `created_at` / `updated_at` | timestamp | nosso DB | auditoria |

**`WebhookEvent`** (idempotência/auditoria — RNF-02/RNF-06):

| Campo | Tipo | Notas |
|-------|------|-------|
| `gateway_event_id` | string | **único**; descarta reentrega |
| `type` | string | tipo do evento Asaas |
| `processed_at` | timestamp | trilha de auditoria |

> Nenhum dado de cartão/PAN é persistido (RNF-03): guardamos só os **ids do gateway**. O estado
> local é sempre **derivado** do Asaas; divergência → **reconciliação** a partir do gateway
> (RNF-06). Não há novo termo de domínio — `Subscription`/`SubscriptionStatus` já vivem no GLO.

## Fluxo cross-repo

```mermaid
sequenceDiagram
    actor U as Pessoa (futuro Subscriber)
    participant W as web (assinatura)
    participant F as Firebase Auth
    participant A as api
    participant G as Asaas (PaymentGateway)
    participant M as mobile (app)

    U->>W: acessa landing / cria conta
    W->>F: signup/login (SDK)
    F-->>W: ID token
    W->>A: POST /subscription { billing_type } (Bearer)
    A->>G: ensure customer + createSubscription
    G-->>A: gateway ids + checkout_url
    A-->>W: 201 { status: pending, checkout_url }
    W-->>U: redireciona ao checkout hospedado (Asaas)
    U->>G: paga (cartão) na página do Asaas
    G->>A: webhook (pagamento confirmado)
    A->>A: valida assinatura + idempotência; status → active
    A-->>G: 200 ack
    W->>A: GET /subscription
    A-->>W: 200 { status: active, last_payment(receipt) }
    W-->>U: recibo + "baixe o app" (redirect às lojas)

    Note over M,A: depois, no app
    M->>A: GET /me (Bearer)
    A-->>M: 200 { subscription_status: active }
    M-->>U: Redemption habilitado (gate aberto)
```

## Decisões relacionadas

- [ADR-003](../architecture_decisions/ADR-003-pagamentos-ciclo-subscription.md) — porta
  `PaymentGateway` + adapter Asaas, webhooks idempotentes e mapa de estados (fonte direta dos
  contratos acima).
- [ADR-006](../architecture_decisions/ADR-006-conformidade-billing-lojas.md) — venda **só na
  web**; mobile sem venda/steering, só leitura + gate; fallback IAP/RevenueCat (não construído
  aqui).
- [ADR-002](../architecture_decisions/ADR-002-autenticacao-autorizacao.md) — auth Firebase
  (a web usa o mesmo protocolo do mobile).
- [AYD-001](AYD-001-onboarding-subscriber.md) — provisionamento do `Subscriber` e o `GET /me`
  cujo `subscription_status` esta feature passa a preencher.
- [PDR-001](../product_decisions/PDR-001-savings-calculation.md) — `Savings` é do `Redemption`;
  aqui a `Subscription` apenas **habilita** o uso (RN-01).

> Nenhuma decisão **nova** de contrato é introduzida aqui — o AYD materializa o que ADR-003 e
> ADR-006 já decidiram. Contrato novo → cria-se/atualiza-se um ADR.

## Fora de escopo / questões em aberto

- **Pix Automático:** MVP entrega **cartão**; `billing_type` já prevê `pix` para a fase 2
  (validar maturidade no sandbox cedo — risco do ROAD/M2).
- **Coleta de cartão:** assumido **checkout hospedado do Asaas** (sem PAN na nossa web/api).
  Alternativa = tokenização na web com SDK Asaas (UX mais integrada, mais trabalho) →
  confirmar antes da SPEC-003@web.
- **Plano único vs. multi-plano:** MVP = **um plano de preço fixo** (config da api), sem
  entidade de plano; multi-plano/trials/cupons ficam para depois.
- **Recibo:** usamos o `receipt_url` do Asaas no MVP (não emitimos recibo próprio).
- **Reconciliação (RNF-06):** rotina periódica que concilia o estado local a partir do Asaas
  (detalhe de implementação → candidato a **TDR@api**).
- **Exclusão de conta / Sign in with Apple:** passos futuros (LGPD / login Google) — não
  mapeados aqui (ADR-006).
- **Fallback IAP/RevenueCat:** só se as lojas rejeitarem o modelo (ADR-006); não construído.
- **Modelo "conta vs. assinante":** seguimos com `Subscription` como **entidade própria** 1:1
  e `subscription_status` derivado (resolve a questão deixada em aberto no AYD-001).
</content>
