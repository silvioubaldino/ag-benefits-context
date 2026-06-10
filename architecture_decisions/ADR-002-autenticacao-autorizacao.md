---
id: ADR-002
type: adr
title: Autenticação e autorização via Firebase
status: accepted
created: 2026-06-08
updated: 2026-06-08
owner: silvioubaldino
affects: [api, mobile]
parents: []
related: [ADR-001, PROD-001, REQ-001, GLO]
tags: [auth, firebase, seguranca, mvp]
superseded_by: null
---

# ADR-002: Autenticação e autorização via Firebase

> Append-only: nunca reescreva. Decisão nova = novo ADR que substitui este.

## Contexto
O RF-01 exige cadastro, login, sessão segura e recuperação de acesso; o RNF-03 exige
autenticação forte sem assumirmos o peso de guardar credenciais. Precisamos de um protocolo
de auth que **atravessa repos** (mobile ↔ api) e de um modelo de **autorização** que
distinga o `Subscriber`, a **operação interna** (endpoints admin/métricas api-only, RF-13/
RF-14) e o **`PartnerOperator`** (app do `Partner`, RF-15/RF-16). Decisão cross-repo →
ADR. A topologia que a contém está no [ADR-001](ADR-001-topologia-cross-repo.md).

## Decisão
Usamos **Firebase Auth** como *identity provider*. A api **não** mantém senhas; ela
**verifica** o token emitido pelo Firebase e resolve a identidade de domínio.

**Protocolo (contrato cross-repo):**
1. O **mobile** faz signup/login/recuperação direto no Firebase (SDK) e obtém um
   **Firebase ID token** (JWT).
2. O mobile envia o ID token no header `Authorization: Bearer <token>` em toda chamada à api.
3. A **api** verifica o token (Firebase Admin SDK): assinatura, expiração e emissor.
4. A api resolve o `Subscriber` interno pelo **`firebase_uid`** (claim `sub` do token). No
   primeiro acesso autenticado sem `Subscriber` correspondente, a api **provisiona** o
   registro de domínio (`Subscriber`) chaveado pelo `firebase_uid`.

**Fronteira de fonte da verdade:** identidade de autenticação vive no **Firebase**; o
**perfil de domínio** (`Subscriber`) vive no **nosso DB**, ligado por `firebase_uid`. A api
nunca trata o token como perfil — só como prova de identidade.

**Autorização (papéis):** o papel é carregado em **custom claim** `role` no token Firebase:

| `role` | Quem | Acesso |
|--------|------|--------|
| `subscriber` (default) | `Subscriber` do app | Endpoints do app (`Catalog`, `Redemption`, histórico…). Sujeito a RN-01 (`Subscription` `active`). |
| `partner_operator` | `PartnerOperator` (pessoa do `Partner`) | App do `Partner`: métricas do próprio `Partner` (RF-15/RF-16) e participação no `Redemption` (fluxo a definir). Acesso **escopado ao seu `Partner`**. |
| `admin` | Operação interna | Endpoints internos de `Partner`/`PartnershipContract`/`Benefit` e métricas (RF-13/RF-14). |

- O papel `admin` é atribuído por processo interno controlado (set de custom claim), **não**
  por self-service.
- Endpoints internos exigem `role = admin`; o mesmo mecanismo de verificação de token vale
  para todos — **não** há um segundo sistema de auth para o admin no MVP.
- `PartnerOperator` **entra no MVP** (app do `Partner`); o modelo de `role` já o comportava,
  sem trocar o protocolo. Seu acesso é **limitado ao próprio `Partner`** — além do `role`, a
  api aplica **autorização por recurso** (o `PartnerOperator` só lê dados do seu `Partner`).

## Alternativas consideradas
| Opção | Prós | Contras | Por que (não) escolhida |
|-------|------|---------|-------------------------|
| Firebase Auth (esta) | Cobre signup/login/recuperação; tira PCI-adjacent de credenciais; SDK mobile; já no stack | Lock-in; PII fora do BR (LGPD) | **Escolhida** |
| Auth próprio na api (sessões/JWT caseiro) | Controle total; sem fornecedor | Reinventa segurança; custo e risco altos | Descartada |
| Provedor alternativo (Auth0/Cognito) | Recursos equivalentes | Sem ganho sobre Firebase aqui; somaria vendor | Descartada |
| Sistema de auth separado para admin | Isolamento do admin | Duplica mecanismo; complexidade desnecessária no MVP | Descartada — custom claim resolve |

## Consequências / trade-offs
- **Positivas:** segurança de credenciais delegada; um único protocolo de token para app e
  admin; autorização extensível por `role`; api stateless quanto à sessão.
- **Negativas:** dependência do Firebase (lock-in na verificação de token); papel em custom
  claim exige processo para propagar mudança de `role` (token precisa renovar).
- **Compliance (LGPD, RNF-04):** Firebase armazena PII fora do BR → **transferência
  internacional**; registrar base legal/consentimento (candidato a PDR/nota de compliance,
  ver ADR-001).
- **Impacto (IDs/repos afetados):** define o contrato de auth para `api` (verificação +
  resolução de `Subscriber` + checagem de `role`) e `mobile` (obtenção e envio do ID token);
  consumido pelos AYDs que tocam endpoints autenticados. Detalhes de endpoints/payloads vão
  nos AYDs, não aqui.
