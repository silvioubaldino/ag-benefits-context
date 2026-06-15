---
id: ADR-004
type: adr
title: Resolução do Redemption via QR TOTP (Partner gera, Subscriber confirma)
status: accepted
created: 2026-06-15
updated: 2026-06-15
owner: silvioubaldino
affects: [api, mobile]
parents: []
related: [ADR-001, ADR-002, REQ-001, PDR-001, GLO]
tags: [redemption, qr, totp, antifraude, mvp]
superseded_by: null
---

# ADR-004: Resolução do `Redemption` via QR TOTP (`Partner` gera, `Subscriber` confirma)

> Append-only: nunca reescreva. Decisão nova = novo ADR que substitui este.

## Contexto
O `Redemption` é o fato central do produto (PROD-001): precisa ser **confiável** ("confiança
no número", RNF-06) e **registrar `Subscriber` × `Benefit` × `Partner` × `Region` × instante
× `Savings`** (RF-10) sem onerar o balcão (RNF-01). O REQ-001 deixou em aberto o **formato e o
ciclo de vida do QR**, e dois objetivos anti-fraude em tensão: (a) o `Subscriber` não deve
"resgatar" sem o `Partner`; (b) o `Partner` não deve inflar nem **omitir** `Redemption`s.

Com o **app do `Partner`** agora no MVP (ADR-001, RF-15/RF-16), há uma superfície do `Partner`
no balcão. A decisão de QR estático (adesivo) tem o furo conhecido de **foto/compartilhamento**
(quebra RNF-05). Decisão cross-repo (app do `Partner` gera, app do `Subscriber` lê, api valida)
→ ADR, sobre a topologia do [ADR-001](ADR-001-topologia-cross-repo.md) e a auth do
[ADR-002](ADR-002-autenticacao-autorizacao.md).

## Decisão
O `Redemption` é capturado por um **QR baseado em TOTP**: o **app do `Partner` gera** o QR
(rotativo no tempo) e o **app do `Subscriber` lê para confirmar**; a **api valida server-side**.
A "caneta" fica do lado do `Subscriber` (quem completa o registro), e a presença do `Partner` é
provada pelo código TOTP vivo.

**Contrato / responsabilidades:**

| Quem | Faz |
|------|-----|
| **api** | Dona do **segredo TOTP por `Partner`** (provisionado no onboarding, **server-side**, nunca no QR). Emite o payload do QR para o app do `Partner` autenticado; **valida** o código TOTP na janela, marca como **usado (single-use)** contra replay, valida elegibilidade e **grava** o `Redemption`. |
| **App do `Partner`** | Autenticado como `partner_operator` (ADR-002), **renderiza e atualiza** o QR TOTP do seu `Partner`. Não detém PII do `Subscriber` nem decide o registro. |
| **App do `Subscriber`** | **Lê** o QR, escolhe o `Benefit` elegível do `Partner` e (em `percentage`) informa o `purchase_amount` (RN-05); **posta** o `Redemption` à api com seu token (ADR-002) + sinal de geo. |

**Granularidade:** o segredo/QR TOTP é **por `Partner`** (prova "este `Partner`, agora") — não
por `Benefit`. O `Benefit` é **escolhido pelo `Subscriber`** no ato, dentre os elegíveis do
`Partner` (resolve a dúvida "por `Benefit`?" do REQ a favor de "por `Partner`").

**Protocolo (passo a passo):**
1. App do `Partner` (autenticado) exibe o **QR TOTP** vigente do seu `Partner` (rotação por
   janela curta, ex.: ~30s — período exato é detalhe de SPEC).
2. `Subscriber` lê o QR, seleciona o `Benefit` e informa `purchase_amount` quando aplicável.
3. App do `Subscriber` envia `{payload TOTP, partner, benefit, purchase_amount?, geo}` +
   ID token (ADR-002).
4. api valida em ordem: **TOTP** (na janela, **não usado**) → **RN-01** (`Subscription`
   `active`) → **RN-02** (`PartnershipContract` vigente) → **RN-04** (limite de repetição) →
   calcula `Savings` (**PDR-001**) → **grava** o `Redemption` (durável e idempotente, RF-10/
   RNF-02) e marca o código TOTP como usado.
5. api devolve a confirmação com o `Savings` (RF-11).

**Por que fecha os dois objetivos anti-fraude:**

| Vetor | Contenção |
|-------|-----------|
| `Subscriber` resgata sem o `Partner` | QR só existe vivo no app do `Partner`; TOTP rotativo + single-use **mata foto/compartilhamento** e resgate remoto. |
| `Partner` **omite** `Redemption` | O registro nasce **server-side no scan do `Subscriber`**; o `Partner` não segura a caneta. |
| `Partner` **infla** sem `Subscriber` | Cada `Redemption` exige scan autenticado de um `Subscriber` real. |
| Residual | **Conluio** `Partner`↔`Subscriber` — sem dinheiro a extrair no MVP (não somos trilho de pagamento); contido por **RN-04** (PDR de antifraude, a definir). |

**Geolocalização:** sinal de **corroboração e auditoria** (RNF-06), **não** portão duro —
GPS de balcão erra e tem peso LGPD/consentimento (RNF-04). Não bloquear `Redemption` legítimo
por GPS impreciso.

## Alternativas consideradas
| Opção | Prós | Contras | Por que (não) escolhida |
|-------|------|---------|-------------------------|
| **`Partner` gera QR TOTP, `Subscriber` lê** (esta) | Prova presença em tempo real; `Partner` não omite; sem scanner no `Partner`; segredo fica server-side | Exige conectividade no balcão; conluio residual | **Escolhida** |
| QR **estático** do `Partner` (adesivo) | Mínimo esforço | Foto/compartilhamento → resgate remoto (fura RNF-05) | Descartada |
| `Subscriber` exibe código, **`Partner` lê** | Autorização explícita do `Partner` | `Partner` segura a caneta → pode **omitir**; precisa scanner no balcão | Descartada |
| Self check-in (geo) sem QR | UX máxima | Integridade mínima; geo spoofável | Descartada |

## Consequências / trade-offs
- **Positivas:** integridade alta do `Redemption` com ônus mínimo ao `Partner` (só exibir o
  QR no app que ele já abre pelas métricas); segredo TOTP **nunca** trafega no QR nem fica no
  device; resolve a tensão dos dois objetivos anti-fraude num único scan.
- **Negativas:** depende de **conectividade** do `Partner` no balcão (mitigável; derivação
  TOTP offline no device é evolução futura de SPEC); replay dentro da janela exige a api
  rastrear códigos usados (single-use); conluio permanece como residual aceito no MVP.
- **Impacto (IDs/repos afetados):** define o contrato de captura para `api` (segredo/validação
  TOTP + gravação idempotente + ordem de validação) e `mobile` (app do `Partner` gera o QR;
  app do `Subscriber` lê e posta). **Resolve** a questão em aberto do REQ-001 ("Resolução do
  QR"). **Não** decide a **política** de repetição/limites (RN-04) — fica no **PDR de
  antifraude** (a definir). O **fluxo ponta a ponta** (sequência cross-repo) será detalhado no
  **AYD do `Redemption`** (a iniciar), conforme `conventions.md` §10.
