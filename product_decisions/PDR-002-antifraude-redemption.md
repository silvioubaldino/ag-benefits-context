---
id: PDR-002
type: pdr
title: Antifraude e limites de repetição do Redemption (RN-04)
status: accepted
created: 2026-06-11
updated: 2026-06-11
owner: silvioubaldino
parents: []
related: [PROD-001, REQ-001, ADR-004, PDR-001, GLO]
tags: [antifraude, redemption, mvp]
superseded_by: null
---

# PDR-002: Antifraude e limites de repetição do `Redemption` (RN-04)

> Append-only: nunca reescreva. Decisão nova = novo PDR que substitui este.

## Contexto / problema de produto
O REQ-001 deixava o RN-04 como "default provisório — confirmar em PDR de antifraude" e o
RNF-05 exigia "limites por `Benefit`/`Subscriber`/período". Com o mecanismo definido em
[ADR-004](../architecture_decisions/ADR-004-resolucao-redemption-qr-totp.md) (QR rotativo
TOTP, sem etapa de "pedir cupom"), a fraude relevante no MVP é de **poluição de métrica**
(`Redemption`s/`Savings` que não correspondem a um uso real de valor para o `Partner`), não
de extração financeira. Este PDR fixa os parâmetros que faltavam para fechar RN-04 e a
janela de validação do TOTP.

## Decisão

**1. Janela de validação do TOTP (mecanismo, complementa ADR-004):**
- Código rotaciona a cada **30 segundos**; servidor aceita a janela atual e a anterior
  (tolerância de até ~60s de dessincronia de relógio).
- Cada `(Subscriber, totp_code)` é **single-use** — reenvio do mesmo código pelo mesmo
  `Subscriber` não cria novo `Redemption` (idempotência, RNF-02).

**2. Limite de repetição por `Benefit` (RN-04):**
- **No máximo 1 `Redemption` do mesmo `Benefit` por `Subscriber` a cada 24h** (janela
  móvel, não "por dia de calendário").
- Tentativa adicional dentro da janela é **bloqueada** com mensagem clara ("Você já usou
  este benefício hoje; disponível novamente às HH:MM").
- Este limite é **por `Benefit`**, não por `Partner` — um `Subscriber` pode resgatar
  `Benefit`s diferentes do mesmo `Partner` no mesmo dia.

**3. Limite de volume por `Subscriber` (novo, contenção geral):**
- **Teto diário de `Redemption`s por `Subscriber`** (across todos os `Benefit`s):
  inicialmente **5/dia**. Acima disso, novos `Redemption`s são bloqueados com mensagem
  ("Limite diário de usos atingido; fale com o suporte se isso for um engano").
- Protege contra um `Subscriber` "varrendo" vários `Partner`s/`Benefit`s em sequência sem
  uso real (ex.: passando de loja em loja só para inflar o histórico).

**4. Plausibilidade do `purchase_amount` (liga com PDR-001/RN-05):**
- Cada `Benefit` com `discount_type = percentage` carrega um **teto de `purchase_amount`**
  configurado no cadastro (RF-13), refletindo um valor plausível de ticket para aquele
  `Partner`/categoria.
- Valor informado acima do teto é **limitado ao teto** para fins de cálculo do `Savings`
  (o `Redemption` é registrado normalmente; o `Savings` é congelado sobre o valor
  tetado) — evita que um valor digitado de forma abusiva infle `Savings` sem bloquear o
  uso legítimo.

**5. Monitoramento (RNF-06, sem bloqueio automático):**
- Métricas de `Redemption`s por `Subscriber`/`Partner`/`Benefit`/período ficam disponíveis
  para consulta interna (RF-14), permitindo identificar padrões anômalos (ex.: mesmo
  `Subscriber` e `Partner` sempre nos limites, horários atípicos).
- Não há, no MVP, suspensão automática de `Subscriber`/`Partner` por anomalia — ações
  corretivas são manuais (suporte/operação).

## Trade-offs
- **Ganhamos:** limites simples, sem estado adicional (sem `pending`/`disputed` —
  consistente com a decisão do ADR-004 de não introduzir a etapa "pedir cupom" no MVP);
  parâmetros ajustáveis por configuração (não exigem mudança de contrato).
- **Abrimos mão de:** detecção fina de fraude (ML/heurísticas de comportamento) — fica para
  quando houver volume e dados reais.
- **Risco:** limites iniciais (24h/`Benefit`, 5/dia) são **palpites informados**; podem
  gerar falsos positivos (uso legítimo bloqueado) ou serem frouxos demais. Tratados como
  **parâmetros de configuração**, ajustáveis sem nova decisão formal.

## Impacto em métricas
- Limites de RN-04 protegem a integridade de `Redemption`s/mês (PROD-001, Obj. 2) e do
  `Savings` médio (Obj. 3) contra inflação artificial.
- O teto de `purchase_amount` protege especificamente a credibilidade do `Savings`
  ("confiança no número", RNF-06) sem bloquear o fluxo (RNF-01).

## Alternativas descartadas
- **Sem limite de volume diário (só RN-04 por `Benefit`):** não contém um `Subscriber`
  varrendo muitos `Partner`s/dia — insuficiente para RNF-05.
- **Bloquear (em vez de tetar) `purchase_amount` acima do limite:** prejudica uso legítimo
  de ticket alto e adiciona fricção (re-digitação); tetar preserva o registro do uso e
  apenas limita o `Savings` reconhecido.
- **Suspensão automática por anomalia:** prematuro sem dados reais do piloto; risco de
  bloquear `Subscriber`/`Partner` legítimos.
