---
id: ADR-004
type: adr
title: Resolução do Redemption via QR rotativo (TOTP) gerado pelo Partner
status: accepted
created: 2026-06-11
updated: 2026-06-11
owner: silvioubaldino
affects: [api, mobile]
parents: []
related: [ADR-001, ADR-002, REQ-001, PDR-001, PDR-002, GLO]
tags: [redemption, qr, antifraude, mvp]
superseded_by: null
---

# ADR-004: Resolução do `Redemption` via QR rotativo (TOTP) gerado pelo `Partner`

> Append-only: nunca reescreva. Decisão nova = novo ADR que substitui este.

## Contexto
O REQ-001 deixava em aberto o **formato e ciclo de vida do QR** que confirma um
`Redemption` (RNF-05: "QR resolvido server-side"). A escolha precisa atender, ao mesmo
tempo, aos três objetivos de integridade do produto:

1. O `Subscriber` não pode registrar um `Redemption` sem o `Partner` "participar".
2. O `Partner` não pode inflar `Redemption`s sem um `Subscriber` real presente.
3. O `Partner` não pode **omitir** (negar/desencorajar a confirmação de um uso real).

Isso é uma decisão **cross-repo**: `mobile` (apps do `Subscriber` e do `Partner`,
[ADR-001](ADR-001-topologia-cross-repo.md)) e `api` (geração/validação do segredo, registro
do `Redemption`). Avaliamos o espectro de mecanismos — do mais leve (self check-in por
geolocalização) ao mais pesado (check-in do `Subscriber` + confirmação explícita do
`Partner`, que traria o papel `PartnerOperator` para um fluxo de aprovação síncrona).

**Decisão de enquadramento:** no modelo do produto, o desconto acontece no balcão entre
`Subscriber` e `Partner`; o `Redemption` é o **reflexo digital** desse uso, e o produto
**não processa a compra** (PROD-001, PDR-001). Logo, a fraude relevante no MVP **não move
dinheiro** — ela **polui a métrica** (`Savings`/`Redemption`s) que sustenta RNF-06
("confiança no número") e o valor comercial mostrado ao `Partner` (RF-16). Isso permite um
mecanismo mais leve que um anti-fraude de meio de pagamento.

Também avaliamos um fluxo em duas etapas — o `Subscriber` "pede o cupom" (criando um
registro `pending` no servidor) e depois confirma com o `Partner` — como salvaguarda extra
contra omissão. **Decisão: fora do MVP.** Sem essa etapa, **nenhum registro existe** se o
`Partner` desencorajar a confirmação **antes** de qualquer ação no app — esse risco fica
**sem mitigação técnica** no MVP, mitigado apenas por **incentivo/UX** (o `Subscriber`
quer seu `Savings` registrado) e por monitoramento de anomalias (RNF-06). Se essa fraude se
mostrar relevante no piloto, reavaliar via novo ADR (reintroduzindo a etapa `pending`).

## Decisão
O **app do `Partner`** (`PartnerOperator`, RF-15/RF-16) exibe um **QR que muda a cada
30–60s**, derivado de um **segredo compartilhado entre o `Partner` e a `api`** (esquema
TOTP — mesmo princípio de um app autenticador 2FA). O **`Subscriber` escaneia** esse QR no
app dele, escolhe o `Benefit` e confirma — sem etapa prévia de "pedir cupom".

**Conteúdo do QR:** `{ partner_id, totp_code }`. O `totp_code` é calculado **localmente**
pelo app do `Partner` a partir de `(segredo_do_partner, janela_de_tempo_atual)` — **não
exige rede do lado do `Partner`**.

**Fluxo (contrato cross-repo):**
1. App do `Partner` exibe o QR rotativo (cálculo local, offline-friendly).
2. `Subscriber` escaneia → identifica `partner_id` → seleciona o `Benefit` desse `Partner`
   (do `Catalog` que já vê) → se `discount_type = percentage`, informa o valor da compra
   (RN-05).
3. App do `Subscriber` envia `{ partner_id, totp_code, benefit_id, purchase_amount? }` à
   `api`.
4. A `api`:
   - **recalcula** o `totp_code` esperado para `partner_id` na janela atual (±1 janela de
     tolerância de relógio);
   - valida elegibilidade (RF-09): `SubscriptionStatus = active` (RN-01),
     `PartnershipContract` vigente (RN-02), limites de repetição (RN-04, ver
     [PDR-002](../product_decisions/PDR-002-antifraude-redemption.md));
   - se tudo OK, cria o `Redemption` **já confirmado**, calcula e congela o `Savings`
     (RN-03/RN-05) e responde com a confirmação (RF-11).
   - idempotência (RNF-02): chave `(Subscriber, Benefit, janela TOTP)` — reenvio do mesmo
     `totp_code` pelo mesmo `Subscriber` não duplica.

**Provisionamento do segredo:** o segredo TOTP é **por `Partner`** (não por dispositivo),
gerado pela `api` no cadastro do `Partner` (RF-13) e entregue ao app do `Partner` no
primeiro login do `PartnerOperator` (autenticado via ADR-002). Se o `Partner` tiver mais de
um `PartnerOperator`/dispositivo, todos compartilham o mesmo segredo (mesmo QR exibido em
qualquer um). Reemissão do segredo (ex.: comprometimento) invalida QRs anteriores
imediatamente.

**Granularidade:** o QR identifica o **`Partner`**, não um `Benefit` específico — a escolha
do `Benefit` acontece no app do `Subscriber` **após** o scan, a partir do `Catalog` daquele
`Partner` (RF-06). Isso elimina a necessidade de "um QR por `Benefit`" (a versão anterior do
RF-13).

## Alternativas consideradas
| Opção | Prós | Contras | Por que (não) escolhida |
|-------|------|---------|-------------------------|
| **QR rotativo TOTP por `Partner`, sem etapa "pedir"** (esta) | Zero ônus por venda; funciona com `Partner` offline; mata foto/replay (~30–60s); um único segredo cobre todos os `Benefit`s | Não mitiga "`Partner` desencoraja confirmação **antes** de qualquer ação"; depende do `Subscriber` validar | **Escolhida** — proporcional ao risco (poluição de métrica, sem dinheiro em jogo) |
| QR estático por `Benefit` (versão original do RF-13) | Simples, gerado uma vez | Fotografável/compartilhável e reusável indefinidamente — viola RNF-05 | Descartada |
| Token único por transação (`Partner` "gera" por venda) | Autorização explícita do `Partner` por uso | +1 toque por venda; exige rede do `Partner` para mintar; reabre risco de omissão (basta não gerar) | Descartada para o MVP — candidata a reforço futuro |
| Fluxo em 2 etapas: `pending` (pedido) → confirmação | Cobre omissão **antes** da ação do `Subscriber`; cria rastro para reconciliação | Mais estado/complexidade (TTL, `disputed`, reconciliação) sem dor comprovada ainda | Adiada — revisitar via novo ADR se a fraude por omissão aparecer no piloto |
| Check-in do `Subscriber` + confirmação síncrona do `Partner` (L3) | Máxima integridade (exige conluio) | Traz fluxo de aprovação síncrona ao app do `Partner`; maior fricção operacional | Descartada para o MVP |
| Self check-in por geolocalização (sem `Partner`) | UX máxima, zero ônus | Integridade mínima; `Subscriber` infla sozinho | Descartada |

## Consequências / trade-offs
- **Positivas:** mecanismo simples (1 segredo por `Partner`, sem ciclo de vida de tokens
  por transação); `Partner` não precisa de rede para "participar"; um único QR cobre todos
  os `Benefit`s do `Partner`; remove a necessidade de gerar/gerenciar QRs por `Benefit`
  (RF-13 simplificado).
- **Negativas:** o risco de **omissão pré-ação** do `Partner` fica sem mitigação técnica no
  MVP (mitigado por incentivo/UX e monitoramento — RNF-06); requer distribuição segura do
  segredo ao app do `Partner` e plano de reemissão.
- **Impacto (IDs/repos afetados):** atualiza **RF-08** (fluxo de leitura), **RF-13**
  (passa a ser provisionamento de segredo por `Partner`, não QR por `Benefit`), **RNF-05**
  (mecanismo concreto de resolução server-side) no REQ-001; define o contrato
  `mobile`↔`api` para o fluxo de `Redemption`. Limites de repetição (RN-04) e janela de
  tolerância TOTP detalhados em [PDR-002](../product_decisions/PDR-002-antifraude-redemption.md).
