---
id: ROAD-001
type: roadmap
title: Roadmap
status: draft
created: 2025-01-01
updated: 2026-06-22
owner: silvioubaldino
parents: [PROD-001]
children: []
related: [REQ-001, AYD-001, AYD-002, ADR-003, ADR-004, GLO]
tags: [mvp, planning, pilot]
superseded_by: null
---

# Roadmap & Planejamento

> Planejamento do **MVP** (REQ-001) até o **piloto numa `Region`**. Refina o
> [PROD-001](product.md); sequência **dirigida por dependência**, não por data.
> Documento **vivo** (edita in-place; ver `_meta/conventions.md` §6/§7).

## Premissas de estimativa

As estimativas só fazem sentido com a capacidade explícita — mude-as se a capacidade mudar.

- **Capacidade:** 1 dev full-stack (api em Go + mobile em RN/Expo), **meio período**,
  **assistido por Claude Code**. Trabalho em **série** (um AYD por vez).
- **Unidade:** "**sprint = 2 semanas de calendário**" **já no ritmo de meio período** (≈ 5 dias
  úteis efetivos por sprint). Não confundir com sprint full-time.
- **Tamanho (T-shirt):** `S` ≈ 1 semana · `M` ≈ 1 sprint (2 sem) · `L` ≈ 2 sprints (3–4 sem).
- **Datas-alvo são derivadas** do sequenciamento a partir de **22/06/2026** e **são frágeis**
  (sem dados de velocidade ainda). Recalibrar após os 2 primeiros entregáveis.
- **Contratos primeiro:** features sem AYD ainda exigem **criar o AYD** (o esforço abaixo já
  inclui análise/design + as N SPECs + implementação + testes das convenções de cada repo).

## Now / Next / Later

Três temas em ordem de dependência: **(1)** conta + tubulação, **(2)** oferta + o evento
central (`Redemption`), **(3)** métricas e prontidão de piloto.

| Horizonte | Tema | Entregáveis | Requisitos | Repos |
|-----------|------|-------------|------------|-------|
| **Now** | Conta e tubulação do `Subscriber` | **AYD-001** Onboarding (implementar — specs prontas) · **AYD-002** Baseline de observabilidade · **AYD-003** Billing/`Subscription` (criar AYD + impl.: Asaas atrás de `PaymentGateway`, webhooks, ciclo de `SubscriptionStatus`) | RF-01/02/03/04 · RNF-01/06/07 | api, mobile |
| **Next** | Oferta + núcleo de valor (`Redemption`) | **AYD-004** Admin interno + modelo de domínio (`Partner`/`PartnershipContract`/`Benefit`/`Region` + provisionamento do segredo TOTP) · **AYD-005** `Catalog` por `Region` · **AYD-006** App do `Partner` (auth `PartnerOperator` + exibição do QR TOTP) · **AYD-007** `Redemption` + confirmação/`Savings` + histórico | RF-13 · RF-05/06 · RF-15/17 · RF-08/09/10/11/12 | api, mobile |
| **Later** | Métricas, refinamento e prontidão de piloto | **AYD-008** Métricas (do próprio `Partner` no app + agregadas api-only) · Busca/filtro do `Catalog` · Endurecimento LGPD (consentimento — candidato a **PDR**), retenção/custo de telemetria (**TDR**) e calibração de SLO/alertas | RF-16/14 · RF-07 · RNF-04/05/06 | api, mobile |

> `web` fica **fora** do MVP (REQ-001). AYD-003..008 **ainda não existem** — criá-los faz parte
> do entregável. Contrato muda só aqui (AYD/ADR); serviços implementam (regra cross-repo).

## Estimativas por entregável

| # | Entregável | Tamanho | Sprints | Janela alvo | Pré-requisitos |
|---|-----------|:-------:|:-------:|-------------|----------------|
| 1 | AYD-001 Onboarding (impl.) | M | 1 | 22/06 – 04/07 | specs/plans prontos |
| 2 | AYD-002 Observabilidade baseline | M | 1 | 07/07 – 18/07 | #1 (instrumenta `/me`,`/healthz`) |
| 3 | AYD-003 Billing/`Subscription` | L | 2 | 21/07 – 15/08 | #1; ADR-003 (Asaas/`PaymentGateway`) |
| 4 | AYD-004 Admin + modelo de domínio | M–L | 1,5 | 18/08 – 05/09 | ADR-004 (segredo TOTP por `Partner`) |
| 5 | AYD-005 `Catalog` por `Region` | M | 1 | 08/09 – 19/09 | #4 (dados de `Partner`/`Benefit`) |
| 6 | AYD-006 App do `Partner` (auth + QR TOTP) | M | 1 | 22/09 – 03/10 | #4; ADR-002/004; **TDR app único×dois** |
| 7 | AYD-007 `Redemption` + `Savings` + histórico | L | 2 | 06/10 – 31/10 | #3,#5,#6; ADR-004; PDR-001/002 |
| 8 | AYD-008 Métricas (`Partner` + agregadas) | M | 1 | 03/11 – 14/11 | #7 (dados de `Redemption`) |
| 9 | Busca/filtro `Catalog` (RF-07, *Should*) | S | 0,5 | 17/11 – 21/11 | #5 |
| 10 | LGPD/retenção/SLO (endurecimento) | S | 0,5–1 | 24/11 – 05/12 | #2,#3 (transversal) |

**Total ≈ 11,5 sprints ≈ 24 semanas** (calendário, meio período) → keystone (`Redemption`)
completo no fim de outubro/2026; **piloto pronto ≈ início de dezembro/2026**. Os *Should*
(itens 8–10) podem deslizar sem bloquear o lançamento do núcleo.

## Marcos

| Marco | Critério de "pronto" | Data alvo | Dependências |
|-------|----------------------|-----------|--------------|
| **M1 — Tubo validado** | `Subscriber` faz signup/login e existe registro de domínio (`GET/PATCH /me`); toda chamada `mobile→api` rastreável (`traceparent`), logs/métricas no Cloud Operations | **18/07/2026** | AYD-001, AYD-002, ADR-001/002 |
| **M2 — Assinante paga** | Contratação aprovada → `SubscriptionStatus = active`; recibo visível; ciclo `active`/`past_due`/`canceled` dirigido por webhook; cancelamento | **15/08/2026** | M1, AYD-003, ADR-003 |
| **M3 — Oferta no ar** | Operação interna cadastra `Partner`/`PartnershipContract`/`Benefit`/`Region` e provisiona segredo TOTP; `Subscriber` enxerga o `Catalog` da sua `Region` | **19/09/2026** | M2, AYD-004, AYD-005 |
| **M4 — Resgate funciona (keystone)** | App do `Partner` exibe QR rotativo; `Subscriber` escaneia, escolhe `Benefit` e confirma; `Savings` congelado (RN-03/05); histórico/"você economizou R$ X"; antifraude (RN-04) e idempotência (RNF-02) | **31/10/2026** | M3, AYD-006, AYD-007, ADR-004, PDR-001/002 |
| **M5 — Piloto observável** | `PartnerOperator` vê métricas do próprio `Partner`; métricas agregadas api-only; dashboards técnico+produto; busca/filtro; LGPD/retenção endurecidos | **05/12/2026** | M4, AYD-008 |

## Metas iniciais do piloto (OKR — a confirmar)

> O PROD-001 deixou as metas dos KRs "a definir no ROAD". **Propostas iniciais** para uma
> `Region` piloto, a calibrar com o comercial — não são decisão fechada.

| Objetivo (PROD-001) | KR | Proposta inicial (a confirmar) |
|---------------------|----|--------------------------------|
| Provar valor na `Region` | `Subscriber`s ativos pagantes ao fim do piloto | **~100** |
| Provar valor na `Region` | Densidade mínima de `Partner`s ativos | **~25** |
| Criar hábito | `Redemption`s por `Subscriber` ativo/mês | **≥ 2** |
| Criar hábito | % de `Subscriber`s com ≥ 1 `Redemption`/mês | **≥ 60%** |
| Reter | `Savings` médio/`Subscriber` ≥ preço da `Subscription` | **"se paga"** |
| Reter | Churn mensal de `Subscriber`s | **≤ 7%** |

## Custos & riscos (resumo)

| Item | Tipo | Estimativa / Mitigação |
|------|------|------------------------|
| **Cold-start two-sided** (densidade por `Region`) | Risco (negócio) | Maior risco do produto. Concentrar aquisição de `Partner` **antes/junto** do M3; meta de densidade mínima (OKR acima); custo zero ao `Partner` reduz fricção da oferta. |
| **Asaas / Pix Automático** (maturidade da recorrência) | Risco | Abstração `PaymentGateway` (ADR-003) deixa o gateway substituível; validar **sandbox cedo** (no M2); cartão como caminho principal, Pix Automático em paralelo. |
| **Fraude por omissão pré-ação do `Partner`** | Risco (aceito) | Sem mitigação técnica no MVP (ADR-004): incentivo/UX (o `Subscriber` quer seu `Savings`) + monitoramento de anomalias (RNF-06). Revisitar via novo ADR se aparecer no piloto. |
| **Segredo TOTP: distribuição e clock skew** | Risco | Entregue no 1º login do `PartnerOperator` (ADR-004); tolerância ±1 janela; plano de reemissão que invalida QRs antigos. |
| **LGPD: transferência internacional** (Firebase/Asaas) + consentimento | Risco / compliance | Pendência aberta em ADR-001/002/003/005 → **PDR de consentimento** no onboarding (base legal/retenção). Tratar no item #10. |
| **App único × dois apps** (`Subscriber` + `Partner`) | Risco (esforço) | Fechar via **TDR no `mobile`** antes do AYD-006; decisão move a estimativa do app do `Partner`. |
| **Free tier de observabilidade** sob volume do piloto | Custo | Confirmar limites (Cloud Logging/Monitoring, Crashlytics/GA4); definir retenção em TDR (AYD-002). Esperado dentro do free tier no piloto. |
| **Custos recorrentes de fornecedores** | Custo | Firebase (plano Blaze p/ Admin SDK/Crashlytics), Cloud Run/Logging/Monitoring, taxas Asaas. Majoritariamente free tier no piloto; taxa Asaas só sobre `Subscription`s reais. |
