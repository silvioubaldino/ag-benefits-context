---
id: REQ-001
type: requirements
title: Requisitos do produto
status: draft
created: 2026-06-04
updated: 2026-06-22
owner: silvioubaldino
parents: [PROD-001]
children: [AYD-001, AYD-002, AYD-003]
related: [GLO, PDR-001, PDR-002, ADR-003, ADR-004, ADR-006]
tags: [mvp]
superseded_by: null
---

# Requisitos

> Refina o [PROD-001](product.md). Escopo do MVP: **apps mobile do `Subscriber` e do
> `Partner` (`PartnerOperator`) + `api`**, em **uma `Region` piloto** — possivelmente um
> **único app mobile com áreas de `Subscriber` e `Partner`**, a confirmar por viabilidade
> técnica (decisão local via TDR no `mobile`). `Redemption` confirmado por **QR rotativo
> (TOTP) exibido pelo app do `Partner`** (o `Subscriber` lê e escolhe o `Benefit`) — ver
> [ADR-004](architecture_decisions/ADR-004-resolucao-redemption-qr-totp.md). Termos
> canônicos no [GLO](_meta/glossary.md).

## Funcionais (RF)
| ID | Requisito | Prioridade (MoSCoW) | Critério de aceite |
|----|-----------|---------------------|--------------------|
| RF-01 | `Subscriber` cria conta e autentica | Must | Cadastro + login funcionam; sessão segura; recuperação de acesso |
| RF-02 | `Subscriber` contrata `Subscription` mensal (preço fixo) via gateway de pagamento | Must | Pagamento aprovado → `Subscription` fica `active`; recibo visível. A contratação ocorre na **web** (Asaas), **não** no app mobile — conformidade com as lojas ([ADR-006](architecture_decisions/ADR-006-conformidade-billing-lojas.md)) |
| RF-03 | Ciclo de `SubscriptionStatus` (`active`/`past_due`/`canceled`) conforme pagamento | Must | Falha de cobrança → `past_due`; regularização → `active`; só `active` habilita `Redemption` |
| RF-04 | `Subscriber` cancela a `Subscription` | Must | Cancelamento efetivado; perde acesso ao fim do ciclo pago; estado `canceled` |
| RF-05 | `Subscriber` visualiza o `Catalog` de `Benefit`s da sua `Region` | Must | Lista só `Benefit`s de `Partner`s com contrato vigente na `Region` do `Subscriber` |
| RF-06 | `Subscriber` vê detalhe de um `Partner` e seus `Benefit`s | Must | Mostra descrição, desconto, localização e condições do `Benefit` |
| RF-07 | `Subscriber` busca/filtra `Benefit`s (categoria, proximidade) | Should | Filtro retorna resultados coerentes com `Region`/localização |
| RF-08 | `Subscriber` realiza `Redemption` lendo o QR rotativo exibido pelo app do `Partner` | Must | Leitura do QR identifica o `Partner` no servidor (TOTP válido — ADR-004); `Subscriber` escolhe o `Benefit` daquele `Partner`; em `Benefit` percentual informa o valor da compra (RN-05); registra o `Redemption` já confirmado |
| RF-09 | Sistema valida elegibilidade no `Redemption` | Must | Bloqueia se `Subscription` ≠ `active`, contrato não vigente; mensagem clara |
| RF-10 | Registrar `Redemption` de forma durável e idempotente, com campos de métrica | Must | Persiste `Subscriber`×`Benefit`×`Partner`×`Region`×instante×`Savings`; leitura repetida do QR não duplica indevidamente (ver RN-04) |
| RF-11 | `Subscriber` vê confirmação do `Redemption` com o `Savings` do uso | Must | Tela de sucesso exibe o valor economizado naquele `Redemption` |
| RF-12 | `Subscriber` vê histórico de `Redemption`s e `Savings` acumulado | Must | "Você já economizou R$ X" = soma dos `Savings`; lista de usos com data/`Partner` |
| RF-13 | Administração interna de `Partner`/`PartnershipContract`/`Benefit` e provisionamento do segredo TOTP do `Partner` | Must | Equipe cadastra/edita via processo interno (sem UI dedicada); cada `Partner` recebe um segredo TOTP (não por `Benefit` — ADR-004), reemissível |
| RF-14 | Consulta/exportação interna de métricas agregadas | Should | Negócio obtém `Redemption`s e `Savings` por `Partner`/`Region`/período (consulta/export interna, sem UI) |
| RF-15 | `PartnerOperator` autentica e acessa o app mobile do `Partner` | Must | Login do `PartnerOperator`; acesso restrito ao seu `Partner` (`role` `partner_operator`) |
| RF-16 | `PartnerOperator` vê as métricas do próprio `Partner` no app | Must | App mostra `Redemption`s e `Savings` do próprio `Partner` por período; base do valor comercial da parceria |
| RF-17 | App do `Partner` exibe o QR rotativo (TOTP) para leitura pelo `Subscriber` | Must | QR muda a cada ~30s (ADR-004); funciona sem rede do `Partner` (cálculo local a partir do segredo provisionado) |

## Não-funcionais (RNF)
| ID | Categoria | Requisito | Alvo |
|----|-----------|-----------|------|
| RNF-01 | Performance | Carregamento do `Catalog` e confirmação de `Redemption` rápidos (uso no balcão) | `Catalog` < 2s em 4G; confirmação de `Redemption` < 3s |
| RNF-02 | Confiabilidade | `Redemption` não pode ser perdido nem duplicado | Registro durável + idempotência por (`Subscriber`,`Benefit`,janela) |
| RNF-03 | Segurança | Autenticação forte; dados de cartão tokenizados no gateway (não armazenar PAN) | Sem dados sensíveis de cartão no nosso storage; PCI delegado ao gateway |
| RNF-04 | Privacidade/LGPD | Base legal e consentimento para dados pessoais, de uso e localização | Consentimento registrado; direito de exclusão/portabilidade atendido |
| RNF-05 | Antifraude | Conter `Redemption`s abusivos/duplicados | QR rotativo (TOTP) resolvido server-side (ADR-004); limites por `Benefit`/`Subscriber`/período (RN-04, PDR-002) |
| RNF-06 | Auditoria/Observabilidade | "Confiança no número": todo `Redemption` auditável e métricas verificáveis | Trilha de auditoria por `Redemption`; reconciliação de `Savings` |
| RNF-07 | Disponibilidade | App utilizável durante horário comercial do piloto | ≥ 99% no piloto |
| RNF-08 | Escalabilidade | Modelo preparado para múltiplas `Region`s mesmo iniciando com uma | `Region` é dimensão de primeira classe no domínio |

## Regras de negócio
- **RN-01:** Só `Subscriber` com `SubscriptionStatus = active` pode realizar `Redemption`.
- **RN-02:** Um `Benefit` só entra no `Catalog` se o `PartnershipContract` do `Partner`
  estiver vigente.
- **RN-03:** Cada `Redemption` grava o `Savings` apurado **no instante do uso** (valor
  congelado, não recalculado depois).
- **RN-04:** Política de repetição e limites antifraude (decidido em
  [PDR-002](product_decisions/PDR-002-antifraude-redemption.md)): no máx. **1 `Redemption`
  do mesmo `Benefit` por `Subscriber` a cada 24h**; **teto de 5 `Redemption`s/dia por
  `Subscriber`** (todos os `Benefit`s); `purchase_amount` sujeito a teto de plausibilidade
  por `Benefit`.
- **RN-05:** **Cálculo do `Savings`** (decidido em [PDR-001](product_decisions/PDR-001-savings-calculation.md)):
  no `Redemption`, o `Subscriber` informa o valor da compra. Cada `Benefit` tem uma regra
  de desconto (`discount_type` ∈ {`percentage`, `fixed_amount`} + `discount_value`):
  `percentage` → `Savings = purchase_amount × discount_value`; `fixed_amount` →
  `Savings = discount_value` (valor da compra opcional). `Savings` é congelado no uso (RN-03)
  e o `purchase_amount` autodeclarado fica sujeito a tetos/plausibilidade (RN-04).

## Restrições
- **Pagamento:** via gateway terceiro; não armazenamos dados de cartão (PCI delegado).
- **Legais:** LGPD para dados pessoais, de uso e de localização.
- **Sem processamento da compra:** o produto não é meio de pagamento; impacta o `Savings`
  (ver RN-05).
- **Cadastro e métricas de negócio internos:** o cadastro de `Partner`/`PartnershipContract`/
  `Benefit` (RF-13) e as métricas agregadas cross-`Partner` do Negócio (RF-14) seguem
  **internos/api-only**. O `Partner` tem **app mobile** (`PartnerOperator`) para autenticar e
  ver **as métricas do próprio `Partner`** (RF-15/RF-16); **não** há painel web nem
  self-cadastro do `Partner` no MVP.
- **Geografia:** uma única `Region` piloto no MVP.

## Escopo do MVP
- **Dentro:**
  - App **mobile** do `Subscriber`: conta/login **grátis**, `Catalog` por `Region`
    (navegável **sem** `Subscription`), `Redemption` por leitura de QR **gated** por
    `SubscriptionStatus = active` (botão oculto/desabilitado para não-assinante),
    confirmação com `Savings`, histórico + `Savings` acumulado. **Não** vende nem direciona
    a contratação da `Subscription` (conformidade com lojas — [ADR-006](architecture_decisions/ADR-006-conformidade-billing-lojas.md)).
  - App **web** mínimo do `Subscriber` (responsivo): `landing → criação de conta →
    contratação/cancelamento da Subscription (Asaas) → redirecionamento para as lojas`. É a
    superfície de **contratação** e o funil de aquisição (ADR-006).
  - App **mobile** do `Partner` (`PartnerOperator`): login, visualização das **métricas do
    próprio `Partner`** (`Redemption`s/`Savings`) e exibição do **QR rotativo (TOTP)** que
    confirma o `Redemption` (RF-17, ADR-004). Possivelmente **o mesmo app** do `Subscriber`
    com áreas distintas (viabilidade técnica → TDR no `mobile`).
  - **`api`:** identidade, billing/`Subscription`, `Catalog`, registro confiável de
    `Redemption` + `Savings`, armazenamento e consulta interna de métricas.
  - Administração **interna** de `Partner`/`PartnershipContract`/`Benefit` + geração de QR.
  - **1 `Region`** piloto.
- **Fora (por enquanto):**
  - **Painel web** do `Partner` e **self-cadastro** do `Partner` (cadastro segue interno — RF-13).
  - Backoffice interno com UI dedicada (admin segue api-only).
  - **Web** do `Subscriber` **além** da contratação: a `web` no MVP é **mínima** (só assinatura/
    funil — ADR-006); `Catalog`, `Redemption` e histórico seguem **só no mobile**.
  - Outros mecanismos de `Redemption` (ex.: código de cupom) — só QR no MVP.
  - Múltiplas `Region`s / expansão geográfica.
  - Pontos, cashback, pagamento da compra no app.

## Questões em aberto (candidatas a PDR/ADR)
- ✅ **PDR-001 — Cálculo do `Savings`** (RN-05): decidido (valor da compra informado pelo `Subscriber`).
- ✅ **PDR-002 — Antifraude/repetição de `Redemption`** (RN-04): decidido (limites por
  `Benefit`/`Subscriber`/dia + teto de plausibilidade).
- ✅ **ADR-004 — Resolução do `Redemption`**: decidido (QR rotativo TOTP por `Partner`, sem
  etapa de "pedir cupom"; revisitar se omissão pré-ação do `Partner` se mostrar relevante no
  piloto).
