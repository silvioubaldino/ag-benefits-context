---
id: AYD-007
type: design
title: Redemption — registro, confirmação com Savings e histórico
status: approved
created: 2026-07-30
updated: 2026-07-31
owner: silvioubaldino
affects: [api, mobile]
parents: [REQ-001]
children: [SPEC-007@api, SPEC-007@mobile]
related: [AYD-003, AYD-004, AYD-005, AYD-006, ADR-001, ADR-002, ADR-004, PDR-001, PDR-002, GLO]
tags: [mvp, redemption, savings, qr, totp, antifraude, keystone]
superseded_by: null
---

# AYD-007: `Redemption` — registro, confirmação com `Savings` e histórico

> Análise & Design cross-repo. Decide QUAIS repos a feature toca, o PAPEL de cada um
> e os CONTRATOS entre eles. Daqui nascem N SPECs (uma por repo afetado).

## Objetivo

Atende **RF-08** (o `Subscriber` realiza o `Redemption` lendo o QR rotativo exibido pelo app do
`Partner`), **RF-09** (validação de elegibilidade), **RF-10** (registro durável e idempotente
com os campos de métrica), **RF-11** (confirmação exibindo o `Savings` daquele uso) e **RF-12**
(histórico de `Redemption`s + `Savings` acumulado). Resultado esperado: no balcão, o
`Subscriber` escaneia o QR do `Partner`, escolhe um `Benefit` daquele `Partner`, informa o valor
da compra quando o desconto é percentual, confirma — e a `api` valida, registra o `Redemption`
**já confirmado**, **congela** o `Savings` (RN-03/RN-05) e devolve a confirmação; depois, o
`Subscriber` consulta seu histórico e o "você já economizou R$ X".

Este é o **keystone do MVP** (ROAD "M4 — Resgate funciona") e o **fato central de métricas**
do produto (GLO: `Redemption`). Todas as features anteriores convergem aqui: o AYD-004 criou a
oferta e o segredo TOTP, o AYD-005 a expôs no `Catalog`, o AYD-006 entregou o segredo ao app do
`Partner` e **produz** o QR, o AYD-003 dá o `SubscriptionStatus` que faz o gate (RN-01). Este AYD
**consome** o QR e **materializa o evento**.

**Escopo:** (1) o **endpoint de registro do `Redemption`** — validação do `totp_code` (recálculo
server-side, ADR-004), elegibilidade (RN-01/RN-02), limites antifraude (RN-04, PDR-002),
idempotência (RNF-02) e cálculo/congelamento do `Savings` (RN-03/RN-05, PDR-001); (2) a
**entidade `Redemption`** e seus campos de métrica/auditoria (RNF-06); (3) os endpoints de
**histórico e `Savings` acumulado** (RF-12); (4) o **fluxo do app do `Subscriber`**: leitura do
QR pela câmera, escolha do `Benefit`, entrada do valor da compra, tela de confirmação (RF-11) e
tela de histórico.

**Fora de escopo:** métricas do próprio `Partner` no app e as agregadas api-only (RF-16/RF-14)
→ **AYD de métricas** (ROAD "Later"); exibição do QR e auth do `PartnerOperator` (AYD-006); CRUD da oferta (AYD-004);
busca/filtro do `Catalog` (RF-07); `web` (não afetado — ADR-006).

## Repos afetados e papéis

| Repo | Papel nesta feature | SPEC gerada |
|------|---------------------|-------------|
| api | **Dona do domínio e do fato** (ADR-001). Recalcula o `totp_code` do `Partner` na janela atual (±1, PDR-002), valida elegibilidade (RN-01/RN-02) e limites (RN-04), resolve idempotência (RNF-02), **calcula e congela** o `Savings` (RN-03/RN-05 + teto do `Benefit`), persiste o `Redemption` com os campos de métrica (`Subscriber`×`Benefit`×`Partner`×`Region`×instante×`Savings`) e expõe a confirmação e o histórico/acumulado. É quem fixa a taxonomia de erros de elegibilidade. | SPEC-007@api |
| mobile | **Cliente** (área `subscriber`). Lê o QR pela câmera in-app, faz o **parse** do payload do AYD-006, apresenta os `Benefit`s daquele `Partner` (a partir do `Catalog` — AYD-005), coleta o `purchase_amount` quando `discount_type = percentage` (RN-05), envia o registro, renderiza a confirmação com o `Savings` (RF-11) e a tela de histórico + acumulado (RF-12). Reaproveita o gate de `SubscriptionStatus` (AYD-003/`redeemable` do AYD-005) para a affordance; a **checagem forte é da `api`**. **Consome** o contrato; não o redefine. | SPEC-007@mobile |

> `web` **não** é afetado: `Catalog`, `Redemption` e histórico seguem **só no mobile**
> (REQ-001; ADR-006 — a `web` é só funil de venda da `Subscription`).

## Contratos (fonte da verdade)

Endpoints **autenticados** como `Subscriber`: `Authorization: Bearer <Firebase ID token>`
(ADR-002). Valores monetários (`purchase_amount`, `savings`, `total_savings`) trafegam em
**reais** na borda JSON — mesma escala já usada por `discount_value`/`fixed_amount` no
`Catalog` (AYD-005) e por `last_payment.amount` (AYD-003); a persistência interna em centavos é
detalhe de implementação (SPEC@api).

### 1. Registro do `Redemption` (RF-08/09/10/11)

```
POST /redemptions                                (subscriber)
  efeito: valida o QR + elegibilidade + limites, registra o Redemption JÁ CONFIRMADO
          (ADR-004: sem etapa "pending"), congela o Savings e devolve a confirmação.
  req:  {
    partner_id: string,          // do payload do QR (AYD-006)
    totp_code: string,           // 6 dígitos, do payload do QR
    benefit_id: string,          // Benefit escolhido pelo Subscriber, do Partner escaneado
    purchase_amount?: number     // reais; OBRIGATÓRIO se discount_type == percentage (RN-05)
  }
  res 201: {
    id: string,
    redeemed_at: string,                       // ISO-8601, instante do uso (RN-03)
    savings: number,                           // congelado (RF-11)
    purchase_amount: number | null,            // como declarado pelo Subscriber
    partner: { id, name },
    benefit: { id, title, discount_type, discount_value },
    total_savings: number                      // acumulado do Subscriber JÁ incluindo este uso
  }
  res 200: mesmo corpo, quando é REPLAY idempotente (ver notas) — nenhum novo Redemption
  erros: 401 ; 403 ; 404 ; 409 ; 422
```

**Taxonomia de erros (RF-09 — "mensagem clara").** Corpo uniforme
`{ error: { code, message, details? } }`; o cliente decide a UX pelo `code`, nunca pelo texto:

| HTTP | `code` | Quando | `details` |
|------|--------|--------|-----------|
| 401 | `unauthenticated` | token ausente/inválido (ADR-002) | — |
| 422 | `validation_error` | payload malformado; `purchase_amount` ausente em `percentage`; valor ≤ 0 | `{ field }` |
| 403 | `subscription_inactive` | `SubscriptionStatus != active` (RN-01) | `{ subscription_status }` |
| 422 | `invalid_totp` | `totp_code` não confere na janela atual nem na anterior (ADR-004/PDR-002) | — |
| 404 | `partner_not_found` | `partner_id` inexistente, inativo ou fora da `Region` do `Subscriber` | — |
| 404 | `benefit_not_available` | `Benefit` inexistente, inativo, ou com `PartnershipContract` não vigente (RN-02) | — |
| 422 | `benefit_partner_mismatch` | o `Benefit` não pertence ao `Partner` escaneado | — |
| 409 | `benefit_cooldown` | já houve `Redemption` deste `Benefit` nas últimas 24h (RN-04) | `{ available_at }` |
| 409 | `daily_limit_reached` | teto diário de `Redemption`s do `Subscriber` atingido (RN-04) | `{ limit, resets_at }` |

**Ordem de validação (normativa).** A precedência importa para a UX e para não vazar
informação de oferta: **(1)** auth → **(2)** payload → **(3)** `SubscriptionStatus` (RN-01) →
**(4)** `Partner` + `totp_code` → **(5)** `Benefit` (existência, vigência, pertence ao
`Partner`) → **(6)** **idempotência** → **(7)** limites RN-04 → **(8)** cálculo do `Savings` e
persistência.

### 2. Histórico e `Savings` acumulado (RF-12)

```
GET /me/redemptions                              (subscriber)
  query: ?limit=20&cursor=<opaque>               // limit default 20, máx 50
  efeito: lista os Redemptions do Subscriber autenticado, do mais recente para o mais antigo,
          com o agregado de Savings.
  res 200: {
    total_savings: number,        // soma de TODOS os Savings do Subscriber ("você já
                                  // economizou R$ X") — NÃO afetado pela paginação
    count: number,                // total de Redemptions do Subscriber (idem)
    items: [
      {
        id, redeemed_at,
        savings: number,
        purchase_amount: number | null,
        partner: { id, name },
        benefit: { id, title, discount_type, discount_value }
      }
    ],
    next_cursor: string | null    // null = fim da lista
  }
  erros: 401
```

> **Não há `GET /redemptions/:id`** no MVP: a confirmação (RF-11) vem na resposta do `POST` e o
> histórico (RF-12) traz os mesmos campos. Endpoint de detalhe é candidato futuro, não contrato.

**Notas de contrato:**

- **Idempotência (RNF-02) — chave e comportamento.** Chave = **(`Subscriber`, `Benefit`, janela
  TOTP casada)**, conforme ADR-004. A "janela casada" é o contador da janela em que o
  `totp_code` validou (atual ou anterior, ±1 — PDR-002), **não** o instante do request: o mesmo
  código reenviado 10s depois casa a **mesma** janela. Reenvio da mesma tripla **não cria** novo
  `Redemption` — a `api` responde **200** com o `Redemption` já existente (replay idempotente,
  não erro), para que retry de rede no balcão seja seguro. A `api` **não persiste o
  `totp_code`**; persiste o **contador da janela** (a idempotência não precisa do valor, e o
  código é derivado de segredo — ADR-004).
- **Leitura conjunta de ADR-004 e PDR-002 quanto ao "single-use".** O PDR-002 §1 diz
  "`(Subscriber, totp_code)` é single-use"; o ADR-004 fixa a chave **incluindo o `Benefit`**.
  Este AYD adota a chave do ADR-004 (mais específica) e lê o PDR-002 como a **garantia de não
  duplicar** — que ela preserva. Consequência: com o **mesmo** scan, o `Subscriber` pode
  registrar `Benefit`s **diferentes** do mesmo `Partner`, o que é coerente com o PDR-002 §2
  ("um `Subscriber` pode resgatar `Benefit`s diferentes do mesmo `Partner` no mesmo dia") e
  segue contido pelo teto diário (RN-04). Se a leitura estrita (um único `Redemption` por
  código) for desejada, é **PDR novo** — ver "Questões em aberto".
- **Cálculo e congelamento do `Savings` (RN-03/RN-05, PDR-001).**
  `percentage` → `savings = purchase_amount × discount_value` (`purchase_amount`
  **obrigatório**); `fixed_amount` → `savings = discount_value` (`purchase_amount` **opcional**,
  guardado só como métrica de ticket). O valor é gravado no instante do uso e **nunca
  recalculado** — mudar o `Benefit` depois não altera `Redemption`s passados.
- **Teto de plausibilidade — `savings_cap` limita o `Savings` (PDR-002 §4).** Quando o `Benefit`
  tem `savings_cap` não nulo: `savings = min(savings_calculado, savings_cap)`. O `Redemption`
  é **registrado normalmente** (o uso não é bloqueado) e o `purchase_amount` é armazenado **como
  declarado** (métrica de ticket); só o `Savings` reconhecido é tetado. **Decisão de contrato:**
  o AYD-004 nomeou o campo `savings_cap` e o PDR-002 §4 descreveu o teto sobre o
  `purchase_amount` — para `percentage` os dois são equivalentes em efeito, e tetar o `Savings`
  é o único que também vale para `fixed_amount`; fica fixado o teto **sobre o `Savings`**.
- **Antifraude não vaza para o cliente.** Coerente com AYD-005: `savings_cap` **não** aparece em
  nenhuma resposta, e a confirmação **não** sinaliza que o valor foi tetado — o `Subscriber` vê
  o `Savings` reconhecido. O tetamento é registrado para auditoria interna (RNF-06).
- **Janelas dos limites (RN-04).** `benefit_cooldown` = **janela móvel de 24h** desde o último
  `Redemption` daquele `Benefit` por aquele `Subscriber` (PDR-002 §2, explicitamente "não por
  dia de calendário"); `available_at` = `último_redeemed_at + 24h`. `daily_limit_reached` =
  **dia de calendário** na timezone da `Region` (`America/Sao_Paulo` no piloto), teto inicial
  **5**; `resets_at` = próxima meia-noite local. Ambos são **parâmetros de configuração**
  (PDR-002), ajustáveis sem mudança de contrato — por isso `limit` viaja no `details`.
- **`Region` do `Redemption`.** Deriva do `Partner` (que já tem `region_id` — AYD-004) e é
  **denormalizada** no registro (GLO: o `Redemption` captura a `Region`), congelando a praça do
  uso mesmo que o `Partner` mude de `Region` depois.
- **`partner_id` vem do QR, não da escolha do cliente.** O `Benefit` é validado como
  **pertencente** ao `Partner` escaneado (`benefit_partner_mismatch`) — sem isso, um `Benefit`
  de outro `Partner` poderia ser registrado com um QR válido qualquer.
- **Sem etapa `pending` (ADR-004).** O `Redemption` nasce **confirmado**; não há estado
  intermediário, TTL nem `disputed` no MVP.
- **Performance (RNF-01):** confirmação do `Redemption` **p95 < 3s** (uso no balcão/4G).
- **Observabilidade (AYD-002/ADR-005):** contadores `redemption_created_total{region}` e
  `redemption_rejected_total{region,reason}` (`reason` = o `code` do erro — baixa
  cardinalidade). Logs com `subscriber_id` de domínio; **nunca** `firebase_uid`, e-mail,
  `totp_code` ou `totp_secret` (RNF-03/04).
- **Auditoria (RNF-06):** todo `Redemption` é imutável após criado (sem update/delete no
  contrato) e carrega o **snapshot da regra de desconto** aplicada — é o que torna o `Savings`
  reconciliável.

## Modelo de domínio afetado

Introduz **uma entidade** — o `Redemption` (já no GLO; aqui ganha persistência). `Savings` é
**campo** do `Redemption` (GLO), não entidade. Nenhum termo novo de domínio.

**`Redemption`** (GLO — evento de uso de um `Benefit` por um `Subscriber` num `Partner`):

| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | id | chave de domínio |
| `subscriber_id` | id (FK) | quem usou (RF-10) |
| `benefit_id` | id (FK) | `Benefit` usado |
| `partner_id` | id (FK) | **denormalizado** do `Benefit`→`Contract`→`Partner` (métrica por `Partner`, RF-14/16) |
| `region_id` | id (FK) | **denormalizado** do `Partner`; congela a praça do uso (RNF-08) |
| `redeemed_at` | timestamp | **instante do uso** (RN-03); base das janelas RN-04 e do histórico |
| `purchase_amount` | number\|null | valor **autodeclarado** (RN-05); null quando `fixed_amount` sem declaração |
| `savings` | number | **congelado** (RN-03); já com o teto aplicado (PDR-002 §4) |
| `discount_type` | enum (snapshot) | `percentage` \| `fixed_amount` **na hora do uso** — auditoria (RNF-06) |
| `discount_value` | number (snapshot) | regra aplicada **na hora do uso** — auditoria (RNF-06) |
| `savings_capped` | bool | o teto foi aplicado? auditoria interna; **não exposto** ao cliente |
| `totp_window` | int | contador da janela TOTP casada — componente da chave de idempotência |
| `created_at` | timestamp | auditoria |

> **Unicidade (RNF-02):** índice único em (`subscriber_id`, `benefit_id`, `totp_window`) — é o
> que torna a idempotência uma garantia de banco, não só de código. O `Redemption` é
> **append-only**: sem update nem delete no contrato.

| Termo (GLO) | Papel nesta feature |
|-------------|---------------------|
| `Redemption` | O **fato** registrado aqui; fonte de todas as métricas de uso. |
| `Savings` | Valor congelado no `Redemption` (RF-11) e somado no histórico (RF-12). |
| `Benefit` | Origem da regra de desconto (`discount_type`/`discount_value`/`savings_cap` — AYD-004). |
| `Partner` | Identificado pelo QR (AYD-006); dono do `totp_secret` recalculado aqui. |
| `Region` | Denormalizada no `Redemption`; dimensão das métricas (RNF-08). |
| `Subscription` | Seu `SubscriptionStatus` faz o gate forte do resgate (RN-01, AYD-003). |
| `PartnershipContract` | Sua vigência decide se o `Benefit` ainda é resgatável (RN-02). |
| `Catalog` | Origem da escolha do `Benefit` no app após o scan (AYD-005). |

## Fluxo cross-repo

```mermaid
sequenceDiagram
    actor Op as PartnerOperator (mobile, área partner)
    actor S as Subscriber (mobile, área subscriber)
    participant A as api
    participant DB as DB (domínio)

    Note over Op: exibe o QR rotativo (AYD-006)<br/>agbenefits://redeem?p=<partner_id>&c=<totp_code>

    S->>S: câmera in-app lê o QR e faz o parse (partner_id, totp_code)
    S->>A: GET /catalog/partners/:id            (AYD-005 — Benefits daquele Partner)
    A-->>S: 200 { partner, benefits[], redeemable }
    S->>S: escolhe o Benefit; se percentage, informa purchase_amount (RN-05)

    S->>A: POST /redemptions { partner_id, totp_code, benefit_id, purchase_amount? }
    A->>A: 1-2. auth + payload
    A->>DB: 3. SubscriptionStatus do Subscriber (RN-01)
    alt status != active
        A-->>S: 403 subscription_inactive
    else active
        A->>DB: 4. Partner (Region do Subscriber) + totp_secret
        A->>A: recalcula TOTP na janela atual e na anterior (±1 — ADR-004/PDR-002)
        alt código não confere
            A-->>S: 422 invalid_totp
        else confere (janela W)
            A->>DB: 5. Benefit ativo, contrato vigente (RN-02), pertence ao Partner
            A->>DB: 6. busca (subscriber_id, benefit_id, totp_window=W)
            alt já existe
                A-->>S: 200 { ...redemption existente }        // replay idempotente (RNF-02)
            else não existe
                A->>DB: 7. limites RN-04 (24h por Benefit ; teto diário)
                alt limite atingido
                    A-->>S: 409 benefit_cooldown | daily_limit_reached
                else dentro dos limites
                    A->>A: 8. savings = regra do Benefit (RN-05), tetado por savings_cap
                    A->>DB: cria Redemption (savings congelado + snapshot da regra)
                    A-->>S: 201 { id, redeemed_at, savings, partner, benefit, total_savings }
                end
            end
        end
    end

    Note over S: tela de confirmação exibe o Savings do uso (RF-11)

    S->>A: GET /me/redemptions?limit=20
    A->>DB: Redemptions do Subscriber (desc) + soma de Savings
    A-->>S: 200 { total_savings, count, items[], next_cursor }   // "você já economizou R$ X" (RF-12)
```

## Decisões relacionadas

- [ADR-004](../architecture_decisions/ADR-004-resolucao-redemption-qr-totp.md) — **decisão-mãe**
  desta feature: QR rotativo TOTP por `Partner`, sem etapa "pedir cupom", `Redemption` criado
  **já confirmado**, recálculo server-side com ±1 janela e chave de idempotência
  (`Subscriber`, `Benefit`, janela). Este AYD **materializa** o fluxo em endpoints, erros e
  modelo.
- [PDR-001](../product_decisions/PDR-001-savings-calculation.md) — origem do `Savings`
  (`purchase_amount` autodeclarado + regra do `Benefit`) e o congelamento (RN-03).
- [PDR-002](../product_decisions/PDR-002-antifraude-redemption.md) — janela TOTP, limites de
  repetição (24h por `Benefit`, teto diário) e teto de plausibilidade. Aqui viram HTTP + janelas
  explícitas.
- [AYD-006](AYD-006-app-partner-auth-qr-totp.md) — **produz** o QR e fixa o payload
  (`agbenefits://redeem?p=&c=`) e os parâmetros TOTP (SHA1/6/30s) que a `api` **recalcula** aqui.
- [AYD-005](AYD-005-catalog-por-region.md) — origem da escolha do `Benefit` após o scan e do
  flag `redeemable` (gate fraco, de UX); a checagem forte (RN-01) é deste contrato.
- [AYD-004](AYD-004-admin-modelo-dominio-oferta.md) — define `discount_type`/`discount_value`/
  `savings_cap` do `Benefit` e o `totp_secret` do `Partner`, **consumidos** aqui.
- [AYD-003](AYD-003-billing-subscription.md) — dono do `SubscriptionStatus` que habilita o
  resgate (RN-01).
- [ADR-002](../architecture_decisions/ADR-002-autenticacao-autorizacao.md) — identidade do
  `Subscriber` pelo token; o `subscriber_id` **nunca** vem do payload.
- [ADR-001](../architecture_decisions/ADR-001-topologia-cross-repo.md) — a `api` é a fonte da
  verdade do fato; o app é cliente.
- Termos canônicos no [GLO](../_meta/glossary.md).

> **Nenhum contrato de fundação novo é introduzido:** o AYD materializa o que ADR-004 (mecanismo),
> PDR-001 (`Savings`) e PDR-002 (limites) já decidiram. As duas escolhas que este AYD **fixa** por
> não estarem fechadas — o `savings_cap` tetar o **`Savings`** e a chave de idempotência incluir o
> `Benefit` — estão registradas acima e em "Questões em aberto"; endurecê-las é PDR novo, não
> decisão local do serviço.

## Fora de escopo / questões em aberto

- **Métricas (RF-14/RF-16):** as métricas do próprio `Partner` no app e as agregadas api-only
  são do **AYD-009** (ROAD "Later"; o número 008 foi ocupado pela atribuição de `role`
  administrativa). Este AYD só **gera o dado** — e o modelo do `Redemption`
  (com `partner_id`/`region_id` denormalizados) já é desenhado para elas.
- **Idempotência estrita por código (`(Subscriber, totp_code)`):** adotada a chave do ADR-004
  (com `Benefit`). A leitura estrita — um único `Redemption` por código escaneado, obrigando novo
  scan para cada `Benefit` — é mais restritiva e **conflita** com o PDR-002 §2; se o piloto
  mostrar abuso de "vários `Benefit`s por scan", revisitar via **PDR novo**.
- **Teto sobre `Savings` × sobre `purchase_amount`:** fixado sobre o `Savings` (ver notas). Se o
  Negócio quiser também limitar o `purchase_amount` **armazenado** (métrica de ticket), é ajuste
  no PDR-002.
- **Estorno/cancelamento de `Redemption`** (uso registrado por engano, disputa com o `Partner`):
  **fora do MVP** — o `Redemption` é append-only. Ação corretiva é **manual/interna** (PDR-002 §5).
  Um fluxo de `disputed`/estorno é candidato a AYD futuro, junto com a etapa `pending` que o
  ADR-004 adiou.
- **Redemption offline do lado do `Subscriber`** (sem rede no balcão): não suportado — o registro
  exige a `api`. Fila local com retry é **decisão de app** (TDR@mobile), não contrato; se virar
  requisito de produto, revisita o contrato (janela TOTP expira).
- **Clock skew grande no dispositivo do `Partner`:** risco operacional conhecido (ROAD; AYD-006).
  Aqui aparece como `invalid_totp` — a mensagem ao `Subscriber` deve sugerir **nova tentativa com
  o QR atualizado**, sem expor o mecanismo.
- **Paginação do histórico:** cursor opaco, ordenado por `redeemed_at` desc. Filtros (por
  período/`Partner`) e export para o `Subscriber` ficam fora do MVP.
- **`total_savings` como agregado calculado:** somado on-the-fly no MVP (volume do piloto). Se o
  SLO apertar, materializar o acumulado é **TDR@api**, não mudança de contrato.
- **SPECs:** `SPEC-007@api` e `SPEC-007@mobile` ainda **não existem** — são o próximo passo da
  cascata (declaradas em `children` como intenção de design).
</content>
