---
id: AYD-005
type: design
title: Catalog por Region (exibição da oferta ao Subscriber)
status: approved
created: 2026-07-14
updated: 2026-07-15
owner: silvioubaldino
affects: [api, mobile]
parents: [REQ-001]
children: [SPEC-005@api, SPEC-005@mobile]
related: [AYD-004, ADR-001, ADR-002, ADR-006, GLO]
tags: [mvp, catalog, benefit, partner, region, subscriber]
superseded_by: null
---

# AYD-005: Catalog por Region

> Análise & Design cross-repo. Decide QUAIS repos a feature toca, o PAPEL de cada um
> e os CONTRATOS entre eles. Daqui nascem N SPECs (uma por repo afetado).

## Objetivo

Atende **RF-05** (o `Subscriber` visualiza o `Catalog` de `Benefit`s da sua `Region`) e
**RF-06** (detalhe de um `Partner` e seus `Benefit`s). Resultado esperado: o `Subscriber`
autenticado abre o app e **enxerga a oferta** da sua `Region` — a lista de `Partner`s com
`PartnershipContract` vigente (RN-02) e os `Benefit`s ativos de cada um —, além de poder
abrir o **detalhe** de um `Partner` (descrição, localização, condições do `Benefit`).

É a feature que **expõe ao consumo** o modelo de domínio criado no **AYD-004** (até aqui a
oferta só existia no lado admin, sem superfície de leitura). Fecha a segunda metade do marco
**M3 — "Oferta no ar"**: com o admin cadastrando (`AYD-004`) e o `Catalog` exibindo (este),
o `Subscriber` finalmente vê `Benefit`s reais. É **pré-requisito** do `Redemption` (AYD-007),
que parte de um `Benefit` escolhido no `Catalog`.

**Escopo:** endpoints de **leitura** do `Catalog` por `Region` e do detalhe de `Partner`
(api), consumidos pela área `subscriber` do app (mobile). **Não** inclui busca/filtro por
categoria/proximidade (RF-07, *Should* → ROAD "Later"), nem o `Redemption` em si (AYD-007),
nem qualquer mutação da oferta (isso é admin — AYD-004).

## Repos afetados e papéis

| Repo | Papel nesta feature | SPEC gerada |
|------|---------------------|-------------|
| api | **Dono do domínio** (ADR-001). Expõe os endpoints de leitura do `Catalog`: resolve a `Region` do `Subscriber` autenticado, filtra `Partner`s ativos com `PartnershipContract` **vigente** (RN-02) e seus `Benefit`s ativos, e monta a resposta. Anota a **redeemability** a partir do `SubscriptionStatus` (não esconde o `Catalog`). | SPEC-005@api |
| mobile | **Cliente** (ADR-006). Renderiza a lista do `Catalog` (RF-05) e a tela de detalhe do `Partner`/`Benefit` (RF-06) na área `subscriber`; reaproveita o gate de `SubscriptionStatus` já existente (AYD-003) para apresentar a affordance de resgate. **Consome** o contrato; não o redefine. | SPEC-005@mobile |

> `web` **não** é afetado: `Catalog`, `Redemption` e histórico seguem **só no mobile**
> (REQ-001; ADR-006 — a `web` é só funil de venda da `Subscription`).

## Contratos (fonte da verdade)

Endpoints **autenticados** como `Subscriber`: `Authorization: Bearer <Firebase ID token>`
(ADR-002). Caminhos sob `/catalog`. Unidade e escala de `discount_value` seguem o que a
**SPEC-005@api** fixar em conjunto com SPEC-004@api / AYD-007 (consistência do cálculo do
`Savings`).

```
GET /catalog                               (subscriber)
  efeito: resolve a Region do Subscriber; retorna os Partners ATIVOS dessa Region cujo
          PartnershipContract está VIGENTE hoje (RN-02), cada um com seus Benefits ATIVOS.
  res 200: {
    region: { id, name },
    subscription_status: "pending" | "active" | "past_due" | "canceled",
    redeemable: boolean,                 // true sse subscription_status == active (RN-01)
    partners: [
      {
        id, name,
        description: string | null,
        location: { address?: string, lat?: number, lng?: number } | null,   // RF-06
        benefits: [
          { id, title, description, discount_type, discount_value }
        ]
      }
    ]
  }
  erros: 401 ; 404 (nenhuma Region ativa configurada — piloto ainda sem oferta)
```

```
GET /catalog/partners/:id                  (subscriber)        — RF-06 (detalhe)
  efeito: detalhe de um Partner visível ao Subscriber (mesma Region, ativo, contrato vigente).
  res 200: {
    id, name, description, location,
    redeemable: boolean,
    benefits: [ { id, title, description, discount_type, discount_value } ]
  }
  erros: 401 ; 404 (Partner inexistente, inativo, fora da Region do Subscriber, ou sem
                    contrato vigente — indistinguíveis do cliente por privacidade da oferta)
```

**Notas de contrato:**
- **Visibilidade = Region + vigência + `active` (RN-02).** Só entram: `Region` ativa, `Partner`
  `active`, `PartnershipContract` **vigente hoje** (`starts_at ≤ now ≤ ends_at`, `ends_at` null
  = aberto) e `Benefit` `active`. `Partner` sem nenhum `Benefit` visível **não** aparece na
  lista.
- **Resolução da `Region` (MVP = praça única).** No MVP o produto opera **uma** `Region`
  piloto; o `GET /catalog` resolve a **única `Region` ativa** server-side. O `Subscriber` **não**
  escolhe `Region` e o `Benefit` **não** repete `region_id` (deriva via `Partner`). O vínculo
  `Subscriber → Region` persistido (`Subscriber.region_id`) é um **hook futuro** (RNF-08,
  multi-`Region`) — ver "Questões em aberto"; **fora do escopo** deste contrato.
- **`SubscriptionStatus` anota, não esconde (funil).** O `Catalog` é **navegável por qualquer
  `Subscriber` autenticado**, inclusive `pending`/`past_due`/`canceled` — é superfície de
  **descoberta** (o `mobile` é grátis para navegar; ADR-006/REQ-001). O `SubscriptionStatus`
  **não filtra a visibilidade**; ele deriva o campo `redeemable` (true só em `active`, RN-01),
  que o app usa para apresentar/gate da ação de `Redemption`. A **checagem forte** do resgate
  é do `Redemption` (RN-01, AYD-007), não deste contrato.
- **Antifraude não vaza para o cliente.** `savings_cap` (RN-04/PDR-002) é **configuração
  interna** do `Benefit`; **não** aparece no `Catalog`. O `Subscriber` vê `title`,
  `description`, `discount_type` e `discount_value` (RF-06).
- **Performance (RNF-01):** `Catalog` p95 < 2s (uso no balcão/4G) — carga enxuta e sem N+1 na
  montagem `Partner` + `Benefit`s.
- **Sem paginação no MVP:** a densidade de uma praça piloto cabe numa resposta única;
  paginação/ busca (RF-07) ficam para o "Later".

## Modelo de domínio afetado

Nenhum termo novo. O `Catalog` (GLO) é uma **projeção de leitura** sobre as entidades já
criadas no AYD-004:

| Termo (GLO) | Papel na projeção |
|-------------|-------------------|
| `Catalog` | Conjunto de `Benefit`s visíveis ao `Subscriber`, filtrado por `Region` e `SubscriptionStatus` (aqui: filtra por `Region`/vigência; `SubscriptionStatus` deriva `redeemable`). |
| `Region` | Recorte que define o conjunto visível. MVP: praça única resolvida server-side. |
| `Partner` | Agrupador exibido (nome, descrição, `location`). Só `active` com contrato vigente. |
| `PartnershipContract` | Sua **vigência** decide se os `Benefit`s do `Partner` entram (RN-02). |
| `Benefit` | A oferta exibida (RF-06). Só `active`. |

> **Não introduz coluna nova** no MVP. O único candidato a evolução do modelo é
> `Subscriber.region_id` (multi-`Region`, RNF-08) — deliberadamente **adiado** (ver abaixo)
> para não acoplar este contrato de leitura ao onboarding (AYD-001).

## Fluxo cross-repo

```mermaid
sequenceDiagram
    actor S as Subscriber (mobile)
    participant A as api
    participant DB as DB (domínio)

    S->>A: GET /catalog (Bearer <Firebase ID token>)
    A->>A: resolve Subscriber + Region (MVP: única Region ativa)
    A->>A: deriva redeemable = (subscription_status == active)
    A->>DB: Partners ativos da Region com contrato vigente (RN-02) + Benefits ativos
    DB-->>A: partners[] + benefits[]
    A-->>S: 200 { region, subscription_status, redeemable, partners[...] }

    Note over S: lista renderizada (RF-05); ação de resgate condicionada a redeemable

    S->>A: GET /catalog/partners/:id
    A->>DB: Partner visível (mesma Region, ativo, contrato vigente) + Benefits
    alt visível
        DB-->>A: partner + benefits
        A-->>S: 200 { ...partner, benefits[...] }    // detalhe (RF-06)
    else não visível
        A-->>S: 404
    end
```

## Decisões relacionadas

- [AYD-004](AYD-004-admin-modelo-dominio-oferta.md) — cria o modelo de oferta
  (`Region`/`Partner`/`PartnershipContract`/`Benefit`) que este `Catalog` **lê**. A vigência do
  `PartnershipContract` (RN-02) definida lá é a mesma consumida aqui.
- [ADR-002](../architecture_decisions/ADR-002-autenticacao-autorizacao.md) — leitura
  autenticada como `Subscriber`; mesmo token/identidade do restante do app.
- [ADR-006](../architecture_decisions/ADR-006-conformidade-billing-lojas.md) — o `mobile` é
  **grátis para navegar** o `Catalog`; a venda da `Subscription` fica na `web`. Justifica o
  `Catalog` navegável pré-assinatura (funil), com `redeemable` gated por `SubscriptionStatus`.
- [ADR-001](../architecture_decisions/ADR-001-topologia-cross-repo.md) — a `api` é a fonte da
  verdade; o `Catalog` é projeção de leitura sobre o domínio dela.
- Termos canônicos no [GLO](../_meta/glossary.md).

> Nenhuma decisão **nova** de contrato de fundação é introduzida — o AYD materializa a leitura
> que REQ-001 (RF-05/06, RN-02) e o GLO (`Catalog`) já definiram sobre o modelo do AYD-004.

## Fora de escopo / questões em aberto

- **`Subscriber.region_id` / multi-`Region` (RNF-08):** no MVP a `Region` é resolvida como a
  única praça ativa. Persistir a `Region` por `Subscriber` (e como ela é **atribuída** — default
  no onboarding? seleção explícita?) é **decisão adiada**; quando multi-`Region` entrar, revisita
  AYD-001 (onboarding) + este contrato (resolução deixa de ser "única Region ativa").
- **Busca/filtro por categoria e proximidade (RF-07, *Should*):** ROAD "Later". `location` já é
  exposto para preparar a proximidade, mas taxonomia/categoria do `Benefit` e o endpoint de
  busca ficam para depois.
- **Paginação/ordenar:** fora do MVP (praça única cabe numa resposta). Revisitar se a densidade
  crescer.
- **Cache/ETag do `Catalog`:** otimização candidata a **TDR@api** se o SLO (RNF-01) apertar;
  não é contrato.
- **`Redemption`:** a ação a partir de um `Benefit` do `Catalog` é do **AYD-007**; aqui só se
  **exibe** e se sinaliza `redeemable`.
