---
id: META-changelog
type: meta
title: Changelog do repo de contexto
status: approved
updated: 2026-06-17
owner: silvioubaldino
---

# Changelog — Context

All notable changes to the shared docs (PROD, REQ, AYD, ROAD, decisions) are documented
here. This is where the audit trail of the "why" behind the **living** documents lives.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this
repo adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> Order: most recent on top; one line per PR, referencing it (see `conventions.md` §9).
> While there is no commit/PR, everything stays under `## Unreleased` (no date/version).
> On the commit/PR, `## Unreleased` becomes `## [dd-MM-yyyy - vX.Y.Z]` and a new empty
> `## Unreleased` is opened above it.
>
> _Note: entries below predating this convention remain in PT-BR / category format._

## Unreleased

- **ROAD-001** — MVP roadmap planned: dependency-ordered Now/Next/Later, milestones M1–M5 with target dates, effort sizing (S/M/L + sprints), risks/costs, and initial pilot OKR targets.
- Clarify changelog policy (§9): one line per PR, which may omit SPEC/PLAN additions

### Adicionado
- **ADR-005 + AYD-002** — Baseline de observabilidade (stack gratuita no Google; contrato `traceparent` mobile→api e vocabulário de métricas). Gera SPEC-002@api/@mobile.
- **AYD-001** — Onboarding do `Subscriber` (identidade): `GET/PATCH /me` com provisionamento idempotente por `firebase_uid`. Gera SPEC-001@api/@mobile.
- **ADR-004 + PDR-002 + RF-17** — `Redemption` via QR rotativo (TOTP) do app `Partner`, com limites de antifraude.
- **ADR-001/002/003** — Topologia C4 do MVP, autenticação Firebase (ID token + `role`) e pagamentos/`Subscription` via Asaas (`PaymentGateway`).

### Alterado
- **REQ-001** — app do `Partner` entra no MVP (RF-15/16); mecanismo de `Redemption` fechado por ADR-004/PDR-002.
- **manifest.md** — grafo/diagrama atualizados com a fundação arquitetural e a observabilidade; AYD-001 → `approved`.

### Removido
- Exemplos de scaffold (`ADR-001-example.md`, `AYD-001-example.md`) por colisão de id.

### Propagação
- ADR-005/AYD-002 → SPEC-002@api criada (com PLAN-002 e `conventions/observability.md`); SPEC-002@mobile a criar.
- Pendência LGPD (transferência internacional via Firebase/Asaas) registrada nos ADRs como candidata a PDR.

## [05-06-2026 - v0.0.3]

### Adicionado
- **conventions.md §10** — Convenções de diagramas: padrão Mermaid embutido no `.md`;
  regras de onde cada diagrama mora (C4 nível 1–2 no ADR, sequência cross-repo no AYD);
  subordinação (texto vence), ciclo de vida e propagação herdados do doc-pai.
- **AYD-000-template.md** — bloco `sequenceDiagram` vazio na seção "Fluxo cross-repo".
- **AYD-001-example.md** — diagrama Mermaid de sequência preenchido como exemplo do
  fluxo de upload de mídia.

## [04/06/2026 - v0.0.2]

### Adicionado
- **PDR-001** — Cálculo do `Savings`: o `Subscriber` informa o valor da compra no
  `Redemption`; `Benefit` define `discount_type` (`percentage`|`fixed_amount`) +
  `discount_value` (status: accepted). Removido o `PDR-001-example.md` (colisão de id).
- **REQ-001** preenchido: RF-01..14, RNF-01..08, RN-01..05, restrições e escopo do MVP
  (status: draft).

### Alterado
- **REQ-001** RN-05 deixa de estar "em aberto" e passa a referenciar [PDR-001]; descontos
  percentuais entram no escopo do MVP; RF-08 atualizado (informar valor da compra).

### Decisões
- `Savings` autodeclarado (valor da compra informado pelo `Subscriber`) com tetos/
  plausibilidade a definir no PDR de antifraude (RN-04). Ver [PDR-001].
- **conventions.md §9** — formato e política do changelog (ordem invertida + SemVer).
- **conventions.md §8** — idioma: docs em PT, código/entidades em EN; GLO define o termo
  canônico em inglês.
- **GLO** preenchido com a linguagem ubíqua do ag-benefits (status: approved).
- **PROD-001** preenchido: visão, problema two-sided, personas/JTBD, posicionamento,
  princípios e anti-escopo (status: draft).

## [04/06/2026 - v0.0.1]
### Alterado
- **GLO** traduzido para termos canônicos em inglês (`Subscriber`, `Partner`,
  `PartnerOperator`, `Subscription`, `SubscriptionStatus`, `PartnershipContract`,
  `Region`, `Benefit`, `Redemption`, `Savings`, `Catalog`); definições em português.
  Resolvida a ambiguidade de "cupom" → canonizado **`Benefit`**.
- **PROD-001** alinhado para referenciar os termos canônicos em inglês.
- **manifest.md** atualizado: nome do produto, fase atual e grafo de documentos.

### Removido
- **`RedemptionMethod`** do GLO: o mecanismo de resgate (QR/código/app) vira decisão de
  negócio posterior; um `Benefit` terá **um** mecanismo, não vários simultâneos.

### Decisões (contexto; PDRs/ADRs formais a criar)
- Escopo do MVP: superfície única = **app mobile do `Subscriber` + `api`**; `Redemption`
  por **QR exibido pelo `Partner`** (Subscriber lê); **1 `Region`** piloto.
- Administração de `Partner`/`PartnershipContract`/`Benefit` é **interna** (sem UI no MVP).
- `Savings` **não** deriva da compra (não somos meio de pagamento) → ver RN-05.
- North Star = MRR (assinantes ativos pagantes); `Savings` e `Redemption`s como input.
- Foco inicial: equilíbrio dos dois lados, concentrado por `Region`.
- `Subscription` mensal única, preço fixo. MVP sem cobrança ao `Partner`.
- Mantido **`Subscriber`** (vs. `User`) para o ator consumidor.
- Questões em aberto: **PDR** cálculo de `Savings` (RN-05), **PDR** antifraude de
  `Redemption` (RN-04), **ADR** resolução do QR server-side.

### Propagação
- REQ-001 em draft (refina PROD-001); AYDs a iniciar.

### Inicialização
- Documentação inicializada a partir do scaffold.
