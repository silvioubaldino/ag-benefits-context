---
id: AYD-006
type: design
title: App do Partner — auth do PartnerOperator + exibição do QR rotativo (TOTP)
status: approved
created: 2026-07-28
updated: 2026-08-17
owner: silvioubaldino
affects: [api, mobile]
parents: [REQ-001]
children: [SPEC-006@api, SPEC-006@mobile]
related: [AYD-004, AYD-009, ADR-001, ADR-002, ADR-004, PDR-002, GLO]
tags: [mvp, partner, partner_operator, totp, qr, auth]
superseded_by: null
---

# AYD-006: App do `Partner` — auth do `PartnerOperator` + QR rotativo (TOTP)

> Análise & Design cross-repo. Decide QUAIS repos a feature toca, o PAPEL de cada um
> e os CONTRATOS entre eles. Daqui nascem N SPECs (uma por repo afetado).

> ### ⚠️ Auth revisada por [AYD-009](AYD-009-simplificacao-modelo-oferta-identidade.md) (17/08/2026)
> **O QR rotativo (TOTP) e o app do `Partner` seguem valendo integralmente** — a decisão do
> ADR-004 foi reafirmada. O que mudou é só **como a identidade do operador é persistida**:
> - **`PartnerOperator` deixou de ser entidade.** A tabela `partner_operators` e os endpoints
>   `POST/GET /admin/partners/:id/operators` e `PATCH /admin/operators/:id` foram removidos.
> - **A identidade virou claim:** `operator_id` → **`partner_id`**, gravada por
>   `PATCH /admin/users/:firebase_uid/role` (AYD-008 estendido). A pessoa operadora é o usuário
>   Firebase; não há mais cópia local dela.
> - **`GET /partner/me` perdeu o bloco `operator`** — passa a devolver só `{ partner: {...} }`.
>
> **Inalterado:** `GET /partner/me/totp-secret`, o cálculo local do código TOTP, a rotação de
> ~30s, a tolerância de ±1 janela e a reemissão do segredo.

## Objetivo

Atende **RF-15** (o `PartnerOperator` autentica e acessa o app mobile do `Partner`, com
acesso restrito ao seu `Partner`) e **RF-17** (o app do `Partner` exibe o **QR rotativo
(TOTP)** para leitura pelo `Subscriber`, mudando a cada ~30s e funcionando **sem rede** do
lado do `Partner`). Resultado esperado: uma pessoa do `Partner` (`role: partner_operator`)
faz login via Firebase (ADR-002), o app **recebe uma única vez** o **segredo TOTP** daquele
`Partner` (gerado no AYD-004), guarda-o em armazenamento seguro do dispositivo e, a partir
daí, **calcula localmente** o QR rotativo — pronto para o `Subscriber` escanear no
`Redemption` (AYD-007).

Esta é a feature que **fecha o lado do `Partner`** do fluxo de resgate: o AYD-004 fez o
segredo **nascer** (e ser reemissível), o AYD-005 expôs a oferta ao `Subscriber`, e este AYD
**entrega o segredo ao app** e **materializa a exibição do QR**. É **pré-requisito do
keystone `Redemption`** (AYD-007): sem QR exibido, o `Subscriber` não tem o que escanear.

**Escopo:** (1) o **vínculo `PartnerOperator ↔ Partner`** — cadastro interno da pessoa
operadora e resolução da identidade escopada ao seu `Partner` (o AYD-004 modelou o `Partner`,
não a pessoa); (2) o **endpoint de contexto + entrega do segredo** ao app autenticado; (3) os
**parâmetros TOTP** e o **formato do payload do QR** (fonte da verdade compartilhada entre o
app do `Partner`, que **produz** o QR, o app do `Subscriber`, que o **consome** — AYD-007 —,
e a `api`, que **recalcula** o `totp_code` no `Redemption` — ADR-004); (4) a **área `partner`
do app mobile** (login + tela do QR).

**Fora de escopo:** o registro do `Redemption` em si (AYD-007); **as métricas do próprio
`Partner` no app (RF-16)** → AYD-008 (ROAD "Later"); qualquer CRUD da oferta (admin, AYD-004);
`web` (não afetado).

## Repos afetados e papéis

| Repo | Papel nesta feature | SPEC gerada |
|------|---------------------|-------------|
| api | **Dono do domínio e da identidade** (ADR-001/002). Modela o `PartnerOperator` (vínculo `firebase_uid → Partner`), expõe o cadastro **interno** (`role: admin`) da pessoa operadora, resolve a identidade escopada ao `Partner` para requisições `role: partner_operator`, e **entrega o segredo TOTP** + parâmetros ao app autenticado (ADR-004). É quem fixa os parâmetros TOTP e o formato do payload do QR. | SPEC-006@api |
| mobile | **Cliente** (área `partner`, TDR-002@mobile — app único com áreas por `role`). Faz login no Firebase como `partner_operator`, busca o contexto + o segredo (**uma vez**), persiste o segredo em **secure storage**, calcula o `totp_code` **localmente** por janela e renderiza o **QR rotativo** (RF-17). **Consome** o contrato; não o redefine. | SPEC-006@mobile |

> `web` **não** é afetado (não há superfície web do `Partner` — REQ-001). O cadastro do
> `Partner` e a *geração* do segredo continuam no AYD-004; aqui o segredo é **entregue** e
> **exibido**, e a **pessoa** operadora é vinculada ao `Partner`.

## Contratos (fonte da verdade)

Todos os endpoints exigem `Authorization: Bearer <Firebase ID token>` (ADR-002). Dois grupos:
**interno** (`role: admin`, sob `/admin`) para o cadastro da pessoa operadora, e **do app do
`Partner`** (`role: partner_operator`, sob `/partner`) para contexto e entrega do segredo.

### 1. Cadastro interno do `PartnerOperator` (admin — completa o RF-13/RF-15)

O AYD-004 deixou o **vínculo `PartnerOperator ↔ Partner`** explicitamente para cá. O operador
**não** é self-service: é **pré-registrado** pela operação interna e ligado a exatamente um
`Partner`. O binding é por **e-mail** (a conta Firebase da pessoa); o `firebase_uid` é
resolvido no 1º login (ver §2).

```
POST  /admin/partners/:id/operators            (admin)
  efeito: pré-registra uma pessoa operadora vinculada ao Partner :id. Marca o e-mail como
          autorizado a assumir role: partner_operator DAQUELE Partner. Não cria a conta
          Firebase (feito pelo processo interno que seta o custom claim — ADR-002).
  req:  { email: string, name?: string }
  res 201: {
    id, partner_id, email, name,
    firebase_uid: null,               // preenchido no 1º login (bind — §2)
    active: true, created_at
  }
  erros: 401 ; 403 (role != admin) ; 404 (partner) ; 409 (email já vinculado) ; 422

GET   /admin/partners/:id/operators            (admin)
  res 200: [ { id, partner_id, email, name, firebase_uid, active, created_at } ]
  erros: 401 ; 403 ; 404

PATCH /admin/operators/:id                      (admin)
  req:  { name?, active? }            // active=false revoga o acesso; não troca de Partner
  res 200: { ...operator }
  erros: 401 ; 403 ; 404 ; 422
```

### 2. App do `Partner`: contexto + entrega do segredo (partner_operator)

```
GET  /partner/me                                (partner_operator)
  efeito: resolve firebase_uid -> PartnerOperator -> Partner (bind no 1º acesso, ver notas).
          Retorna o contexto do operador e do seu Partner. NÃO retorna o segredo.
  res 200: {
    operator: { id, name, email },
    partner:  { id, name, region: { id, name } }
  }
  erros: 401 ; 403 (token sem role partner_operator, ou e-mail não pré-registrado) ;
         409 (e-mail pré-registrado em outro firebase_uid — conflito de bind)
```

```
GET  /partner/me/totp-secret                    (partner_operator)
  efeito: entrega o segredo TOTP do Partner do operador autenticado + os parâmetros para
          cálculo local (ADR-004). O app persiste em secure storage e passa a calcular OFFLINE.
  res 200: {
    partner_id: string,
    secret: string,                   // base32 (RFC 4648), sensível — ver notas
    algorithm: "SHA1",                // RFC 6238
    digits: 6,
    period_seconds: 30,               // janela; ADR-004 (~30–60s) → fixado em 30
    issued_at: string                 // ISO-8601; muda a cada reissue (AYD-004)
  }
  erros: 401 ; 403 (role/escopo) ; 404 (Partner sem segredo provisionado)
```

### 3. Payload do QR (produzido pelo mobile/Partner, consumido pelo mobile/Subscriber + api)

O conteúdo é o do ADR-004 (`{ partner_id, totp_code }`), com **encoding fixado** aqui para
que produtor e consumidor concordem. String única embutida no QR:

```
agbenefits://redeem?p=<partner_id>&c=<totp_code>

  p = partner_id (id de domínio do Partner)
  c = totp_code  (6 dígitos, RFC 6238, SHA1, period=30s, sobre o segredo do Partner)
```

O app do `Subscriber` (AYD-007) faz o parse, envia `{ partner_id, totp_code, benefit_id,
purchase_amount? }` à `api`, que **recalcula** o `totp_code` esperado do `partner_id` na
janela atual com **tolerância de ±1 janela** (PDR-002) — a validação, elegibilidade,
idempotência e registro do `Redemption` são **do AYD-007**, não deste contrato.

**Notas de contrato:**
- **Bind por e-mail (não self-provision).** Diferente do `Subscriber` (provisionado no 1º
  acesso — ADR-002), o `PartnerOperator` só existe se **pré-registrado** por admin. No 1º
  `GET /partner/me`, a `api` casa o **e-mail verificado** do token com um `PartnerOperator`
  ativo de `firebase_uid` nulo e **grava** o `firebase_uid` (bind único). E-mail não
  pré-registrado com `role: partner_operator` → **403** (nunca cria vínculo implícito).
- **Escopo por recurso (ADR-002).** Além do `role`, a `api` aplica **autorização por
  recurso**: o operador só enxerga o **seu** `Partner`. Não há endpoint que receba
  `partner_id` do cliente — o `Partner` é **derivado da identidade**, nunca informado.
- **Segredo: entrega e superfície.** O `secret` só trafega em `GET /partner/me/totp-secret`,
  **sempre sobre TLS**, para um operador **autenticado e escopado**. O app o guarda em
  **secure storage** (`shared/storage`, mobile) e calcula **offline** (RF-17: sem rede do
  `Partner`). O endpoint é **re-buscável** pelo operador autenticado (novo dispositivo,
  troca de operador, ou **reissue** — AYD-004 invalida o anterior e muda `issued_at`); o app
  detecta `issued_at` diferente e re-provisiona. **Decisão:** entrega re-buscável (a auth já
  é o gate), não "one-time" server-enforced — o "1º login" do ADR-004 é o **momento típico**
  de UX, não uma restrição de uso único. Trade-off registrado em "Questões em aberto".
- **Segredo nunca no admin nem no `GET /partner/me`.** Coerente com AYD-004 (`totp_secret`
  write-only): o valor não aparece em nenhum contrato admin nem no contexto; só no endpoint
  dedicado do próprio operador.
- **Parâmetros TOTP são contrato compartilhado.** `SHA1`/`6 dígitos`/`period=30s`/`±1 janela`
  precisam ser **idênticos** no cálculo do app (produz) e no recálculo da `api` (valida —
  AYD-007). Fixados aqui; mudá-los é PR neste AYD (ou ADR-004).
- **Clock skew (RNF/ADR-004).** O QR depende do relógio do dispositivo do `Partner`. A
  tolerância de ±1 janela (PDR-002) absorve desvio pequeno; desvio grande é risco operacional
  conhecido (ROAD "risco: segredo TOTP e clock skew") — mitigação de UX/monitoramento, não
  contrato novo.
- **Sem PII em telemetria (ADR-002/AYD-002).** Logs/métricas usam o **id de domínio** do
  `PartnerOperator`/`Partner`; nunca `firebase_uid`, e-mail ou o `secret`.

## Modelo de domínio afetado

Introduz **uma entidade** — o `PartnerOperator` (já no GLO como **ator**; aqui ganha
persistência). Nenhum termo novo de domínio.

**`PartnerOperator`** (GLO — pessoa do `Partner` que opera o app; ADR-002 `role`):

| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | id | chave de domínio |
| `partner_id` | id (FK) | `Partner` ao qual o operador é **escopado** (RF-15); imutável |
| `email` | string | e-mail da conta Firebase; chave do **bind** (1º login) |
| `firebase_uid` | string\|null | ponte auth↔domínio (ADR-002); **null até o 1º login**, então gravado |
| `name` | string\|null | exibição |
| `active` | bool | soft-delete / revogação de acesso (não troca de `Partner`) |
| `created_at` / `updated_at` | timestamp | auditoria |

> O **segredo TOTP não é campo do `PartnerOperator`** — ele é **do `Partner`** (`Partner.
> totp_secret`, AYD-004). Vários operadores de um mesmo `Partner` compartilham o mesmo segredo
> e exibem o **mesmo** QR (ADR-004). Este AYD só **lê** `Partner.totp_secret` para entregá-lo.

| Termo (GLO) | Papel nesta feature |
|-------------|---------------------|
| `PartnerOperator` | Ator que autentica (`role: partner_operator`) e exibe o QR; escopado a um `Partner`. |
| `Partner` | Dono do `totp_secret` (AYD-004) entregue aqui; contexto (`name`, `Region`) exibido. |
| `Region` | Exibida no contexto do operador (deriva via `Partner`). |

## Fluxo cross-repo

```mermaid
sequenceDiagram
    actor Adm as Operação interna (role: admin)
    actor Op as PartnerOperator (mobile, área partner)
    participant FB as Firebase Auth
    participant A as api
    participant DB as DB (domínio)
    participant SS as Secure storage (device)

    Adm->>A: POST /admin/partners/:id/operators { email }
    A->>DB: cria PartnerOperator (partner_id, email, firebase_uid=null)
    A-->>Adm: 201 { id, ... }
    Note over Adm,FB: processo interno cria/seta custom claim role=partner_operator (ADR-002)

    Op->>FB: login (e-mail/senha) → Firebase ID token (role: partner_operator)
    Op->>A: GET /partner/me (Bearer)
    A->>A: resolve firebase_uid; se null, bind por e-mail verificado
    A->>DB: grava firebase_uid no PartnerOperator (bind único)
    A-->>Op: 200 { operator, partner{ id, name, region } }

    Op->>A: GET /partner/me/totp-secret (Bearer)
    A->>DB: lê Partner.totp_secret do Partner do operador
    A-->>Op: 200 { secret, algorithm, digits, period_seconds, issued_at }
    Op->>SS: persiste secret + params (offline daqui em diante)

    loop a cada period (30s)
        Op->>Op: totp_code = TOTP(secret, now) [cálculo LOCAL]
        Op->>Op: renderiza QR "agbenefits://redeem?p=<partner_id>&c=<totp_code>"
    end

    Note over Op: Subscriber escaneia o QR → registra o Redemption (AYD-007)
    Note over A,DB: no reissue (AYD-004) muda issued_at → app re-busca o segredo
```

## Decisões relacionadas

- [AYD-004](AYD-004-admin-modelo-dominio-oferta.md) — **gera** o `Partner.totp_secret` (e a
  reemissão); modela o `Partner`. Adiou para cá a *entrega* do segredo e o *vínculo*
  `PartnerOperator ↔ Partner` — exatamente o que este AYD fecha.
- [ADR-004](../architecture_decisions/ADR-004-resolucao-redemption-qr-totp.md) — mecanismo
  TOTP **por `Partner`**, conteúdo do QR (`partner_id` + `totp_code`), cálculo local
  offline-friendly, tolerância ±1 janela, segredo entregue no 1º login. Este AYD **materializa**
  os parâmetros concretos e o encoding do payload.
- [ADR-002](../architecture_decisions/ADR-002-autenticacao-autorizacao.md) — `role:
  partner_operator` em custom claim + **autorização por recurso** (escopo ao próprio
  `Partner`); mesmo protocolo de token do restante do produto.
- [PDR-002](../product_decisions/PDR-002-antifraude-redemption.md) — janela de tolerância
  TOTP e limites de repetição (consumidos no recálculo do `Redemption`, AYD-007).
- [ADR-001](../architecture_decisions/ADR-001-topologia-cross-repo.md) — a `api` é a fonte da
  verdade do domínio/identidade; o app é cliente.
- Termos canônicos no [GLO](../_meta/glossary.md).

> **Nenhum contrato de fundação novo é introduzido:** o AYD materializa, em endpoints, modelo
> e parâmetros, o que ADR-004 (TOTP/QR), ADR-002 (role/escopo) e AYD-004 (segredo/Partner) já
> decidiram. Se a revisão exigir mudar mecanismo (ex.: entrega one-time server-enforced, ou
> `partner_id` no claim), isso vira atualização do ADR-002/004, não decisão local.

## Fora de escopo / questões em aberto

- **Métricas do próprio `Partner` no app (RF-16):** deliberadamente **fora** — AYD-008 (ROAD
  "Later"). Aqui o app do `Partner` só autentica e exibe o QR.
- **Registro do `Redemption` (RF-08/09/10):** o consumo do QR pelo `Subscriber` e o registro
  são do **AYD-007**; este AYD só define o **payload** e **produz** o QR.
- **Entrega do segredo: re-buscável × one-time.** Decidido **re-buscável** (auth como gate).
  Alternativa "one-time server-enforced" (mais restrita, exige estado de "já entregue" e um
  fluxo explícito de re-provisionamento) fica como candidata a reforço — revisitar via ADR-004
  se a superfície do segredo preocupar no piloto.
- **Criação da conta Firebase do operador e set do custom claim:** processo **interno**
  (ADR-002, "role atribuído por processo controlado"); a mecânica (Admin SDK, convite por
  e-mail, senha inicial) é **detalhe de implementação** → candidata a **TDR@api**. Aqui só se
  fixa que o bind de domínio é por **e-mail verificado**.
- **Multi-operador / multi-dispositivo:** suportado (todos compartilham o segredo do
  `Partner`, mesmo QR — ADR-004). Revogar **um** operador é `active=false`; **rotacionar o
  segredo** (afeta todos) é reissue no admin (AYD-004).
- **Clock skew grande no dispositivo do `Partner`:** risco operacional conhecido (ROAD) —
  mitigação por UX/monitoramento (ex.: aviso de relógio dessincronizado); não é contrato.
- **Deep-link/scheme do QR (`agbenefits://`):** o scheme é **contrato de dados** (ambos os
  apps concordam no parse), **não** necessariamente um deep link registrado no SO — a leitura
  é por câmera in-app no `Subscriber` (AYD-007). Registrar o scheme como deep link real é
  decisão de app → TDR@mobile se/quando fizer sentido.
