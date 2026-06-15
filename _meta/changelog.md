---
id: META-changelog
type: meta
title: Changelog do repo de contexto
status: approved
updated: 2026-06-14
owner: silvioubaldino
---

# Changelog — Contexto

Registro de mudanças nos docs compartilhados (PROD, REQ, AYD, ROAD, decisões).
É aqui que mora a auditoria do "porquê" dos documentos **vivos**.

> **Ordem:** mais recente **no topo**; mais antigo embaixo (ver `conventions.md` §9).
> **Versão:** SemVer. Enquanto não há commit/PR, tudo fica em **[Não lançado]** (sem data/versão).
> No commit/PR, **[Não lançado]** vira **`[dd-MM-yyyy - vX.Y.Z]`** e abre-se um novo **[Não lançado]** acima.

## [Não lançado]

### Adicionado
- **AYD-001** — Onboarding do `Subscriber` (identidade/conta), primeira fatia vertical do
  produto. Atende RF-01; escopo só identidade (sem billing/`Subscription`). Materializa o
  protocolo da ADR-002 em endpoints (`GET/PATCH /me`), com provisionamento implícito e
  idempotente do `Subscriber` por `firebase_uid` no 1º acesso. Afeta `api` e `mobile`
  (gera SPEC-001@api e SPEC-001@mobile); diagrama de sequência cross-repo em Mermaid.
- **ADR-001** — Topologia cross-repo do MVP (C4 containers): apps mobile do `Subscriber` e do
  `Partner` + api + DB + sistemas externos (Firebase, Asaas); admin e métricas de negócio
  cross-`Partner` api-only, métricas do próprio `Partner` no app do `Partner`; diagrama de
  containers em Mermaid; fronteiras de fonte da verdade (identidade no Firebase, billing no
  Asaas, domínio no DB).
- **ADR-002** — Autenticação/autorização via Firebase: protocolo ID token → api verifica →
  resolve `Subscriber` por `firebase_uid`; autorização por `role` (custom claim:
  `subscriber`/`partner_operator`/`admin`, `partner_operator` escopado ao próprio `Partner`);
  admin protegido pelo mesmo mecanismo (sem auth separado no MVP).
- **ADR-003** — Pagamentos e ciclo de `Subscription` via **Asaas**, atrás de abstração
  `PaymentGateway`; recorrência por cartão + Pix Automático (boleto só avulso); webhooks
  dirigem o `SubscriptionStatus` de forma idempotente; mapa de estados gateway → status.
- **ADR-004** — Resolução do `Redemption` via **QR rotativo (TOTP) gerado pelo app do
  `Partner`**: `Subscriber` escaneia, identifica o `Partner` e escolhe o `Benefit`; sem
  etapa prévia de "pedir cupom" no MVP; segredo TOTP é por `Partner` (não por `Benefit`),
  funciona com `Partner` offline.
- **PDR-002** — Antifraude/repetição de `Redemption` (RN-04): máx. 1 `Redemption`/`Benefit`/
  `Subscriber`/24h; teto de 5 `Redemption`s/dia por `Subscriber`; teto de plausibilidade de
  `purchase_amount` por `Benefit`; janela TOTP de 30s com tolerância de 1 janela.
- **RF-17** (REQ-001) — app do `Partner` exibe o QR rotativo (TOTP).

### Removido
- **ADR-001-example.md** e **AYD-001-example.md** — exemplos de scaffold removidos por
  colisão de id com os docs reais (mesmo critério já aplicado ao `PDR-001-example.md`).

### Alterado
- **manifest.md** — grafo de documentos passa a listar ADR-001/002/003/004 e PDR-002; fase
  atual ajustada para "Design (ADRs de fundação aceitos; AYDs a iniciar)"; diagrama de
  relações inclui a camada de fundação arquitetural.
- **REQ-001** — **app do `Partner`** entra no escopo do MVP (correção, não incremento): novos
  RF-15 (`PartnerOperator` autentica/acessa o app) e RF-16 (vê métricas do próprio `Partner`);
  restrições e "Dentro/Fora" ajustados (sai "app do `PartnerOperator` fora"; permanece fora só
  painel web e self-cadastro do `Partner`).
- **REQ-001** — mecanismo de `Redemption` fechado (ADR-004/PDR-002): RF-08 reescrito (QR
  rotativo identifica o `Partner`, `Benefit` escolhido depois pelo `Subscriber`); RF-13
  passa a ser provisionamento de **segredo TOTP por `Partner`** (não mais "QR por
  `Benefit`"); RNF-05 referencia o mecanismo concreto; RN-04 fechado com os limites do
  PDR-002; "Questões em aberto" todas marcadas como decididas.

### Decisões
- **App do `Partner` no MVP:** haverá um **app mobile do `Partner`** (`PartnerOperator`) —
  motivado pelo **valor comercial das métricas** ao `Partner` e ao time de vendas. Correção de
  uma lacuna do desenho anterior (que o deixava fora). **Possivelmente o mesmo app do
  `Subscriber`** com áreas distintas — viabilidade técnica decidida em **TDR local no
  `mobile`**. Ativa o `role: partner_operator` (ADR-002) e um container cliente no ADR-001.
- **Mecanismo do `Redemption` (ADR-004):** QR **rotativo (TOTP)** exibido pelo app do
  `Partner`, lido pelo `Subscriber` — sem etapa de "pedir cupom" no MVP. Enquadramento:
  fraude relevante no MVP é **poluição de métrica**, não extração financeira (produto não
  processa a compra). Risco residual aceito: omissão **pré-ação** do `Partner` fica sem
  mitigação técnica, coberto por incentivo/UX e monitoramento (RNF-06); revisitar via novo
  ADR (reintroduzindo etapa `pending`) se essa fraude aparecer no piloto.
- **Stack de fundação do MVP:** Firebase (auth), Asaas (pagamentos), DB próprio; push
  **adiado** (sem provedor definido); linguagem/framework de cada serviço fica como **TDR no
  repo do serviço**, fora do repo de contexto.
- **Pagamentos:** Pix Automático (BACEN, em operação desde 2025) como recorrência nativa BR
  ao lado de cartão; boleto recorrente descartado (não há débito automático). Gateway local
  Asaas escolhido; Stripe (já com Pix recorrente em 2026) e Iugu descartados para o MVP, mas
  reavaliáveis via abstração `PaymentGateway`.

### Propagação
- ADR-001/002/003/004 e PDR-002 em `accepted`; serão referenciados (`related`) pelos
  próximos AYDs a partir do AYD-001. Sem `children` ativos ainda (AYDs a iniciar).
- Pendência de compliance LGPD (transferência internacional via Firebase; PII no Asaas)
  registrada nos ADRs como candidata a PDR/nota de compliance.

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
