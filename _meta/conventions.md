---
id: META-conventions
type: meta
title: Convenções da documentação
status: approved
updated: 2026-06-17
---

# Convenções da documentação

O "contrato" que mantém os documentos conectados e legíveis por humanos e IAs,
de forma consistente entre o repo de contexto e os repos de serviço.
(Isto define como escrever DOCS. Padrões de CÓDIGO ficam em cada serviço, em `docs/conventions/`.)

## 1. Tipos de documento e IDs

ID = `PREFIXO-NNN`, estável (nunca muda, mesmo se o arquivo for renomeado/movido).

| Prefixo | Tipo | Onde | Escopo |
|---------|------|------|--------|
| PROD | Produto (visão + estratégia) | contexto: `product.md` | compartilhado |
| REQ  | Requisitos | contexto: `requirements.md` | compartilhado |
| AYD  | Análise & Design (por feature) | contexto: `design/` | compartilhado, cross-repo |
| ROAD | Roadmap / planejamento | contexto: `roadmap.md` | compartilhado |
| PDR  | Product Decision Record | contexto: `product_decisions/` | compartilhado |
| ADR  | Architecture Decision Record | contexto: `architecture_decisions/` | compartilhado, cross-repo |
| SPEC | Especificação (parte de um repo) | serviço: `docs/specs/` | local |
| PLAN | Plano de implementação | serviço: `docs/plans/` | local |
| TDR  | Technical Decision Record | serviço: `docs/technical_decisions/` | local |
| GLO  | Glossário (linguagem ubíqua) | contexto: `_meta/glossary.md` | compartilhado |

## 2. Frontmatter padrão (obrigatório em todo doc)

```yaml
---
id: AYD-007
type: design             # product|requirements|design|roadmap|pdr|adr|spec|plan|tdr
title: Upload de mídia
status: draft            # draft | review | approved | superseded | deprecated
created: 2025-01-01
updated: 2025-01-01
owner: <nome>
affects: [api, web, mobile]        # só em AYD/ADR: repos impactados
parents: [REQ-003]                 # o que este doc refina (camada acima)
children: [SPEC-012@api, SPEC-013@web]   # o que refina este doc (pode ser cross-repo)
related: [ADR-002, GLO]            # contexto transversal
tags: [media]
superseded_by: null                # ID que substitui este doc, se houver
---
```

## 3. Referências entre repos

IDs são **globais no produto**. Para apontar um doc de outro repo, use `ID@repo`:
`AYD-007@context`, `SPEC-012@api`, `SPEC-013@web`. (`@context` é o repo de contexto.)

## 4. Ciclo de status

`draft → review → approved → (superseded | deprecated)`
- **approved** = fonte da verdade vigente. **superseded/deprecated** = não usar para decisões.

## 5. Regras de linkagem (a "cola" do grafo)

- Refinamento declarado nos dois lados: `children` no pai, `parents` no filho.
- Toda `SPEC` tem ao menos um `AYD` em `parents` (ex.: `[AYD-007@context]`).
- **1 AYD → N SPECs**, uma por repo afetado. O AYD é a fonte dos contratos.
- **Contrato só muda no AYD/ADR (no repo de contexto).** Serviços implementam, não redefinem.
- Termos de domínio vivem só no `GLO`; os outros docs apenas referenciam.

## 6. Ciclo de vida: vivo vs imutável

| Tipo | Comportamento | Como mudar |
|------|---------------|-----------|
| PROD / REQ / AYD / ROAD | **Vivo** | Edita in-place, atualiza `updated`. |
| PDR / ADR / TDR | **Append-only** | Nunca reescreve. Decisão nova substitui a antiga via `superseded_by`. |
| SPEC | **Congela ao aprovar** | Mutável em draft/review; vira contrato quando `approved`. |
| PLAN | **Efêmero** | Documento de trabalho; após executado, é histórico. |
| GLO | Vivo | Edita in-place. |

Auditoria dos docs vivos mora no **git + changelog**.

## 7. Propagação de mudanças

Ao alterar um doc:
1. Edita (ou cria uma decisão PDR/ADR/TDR, se for o caso) e atualiza `updated`.
2. Registra no `changelog.md` do repo.
3. Percorre os `children` (inclusive em outros repos) e marca os afetados como `status: review`.
4. Revisa cada filho; confirma → `approved`, ou aposenta → `superseded`/`deprecated`.

## 8. Idioma (documentação vs. código)

- **Documentação é escrita em português.** Toda prosa dos docs (PROD, REQ, AYD, ROAD,
  PDR, ADR, SPEC, PLAN, TDR) é em PT-BR.
- **Código é escrito em inglês.** Logo, **nomes de entidades de domínio, campos, enums,
  endpoints e eventos são definidos em inglês** — eles atravessam para o código.
- Por isso o **GLO define o termo canônico em inglês** (o nome que vira código), com a
  definição em português. Os demais docs **referenciam o termo canônico em inglês** ao
  citar entidades/campos/contratos; a explicação ao redor segue em português.
- Em contratos (AYD/ADR): payloads, campos e valores de enum em inglês
  (ex.: `SubscriptionStatus: active | past_due | canceled`).
- **Exceção — changelog:** o `changelog.md` é escrito em **inglês** (segue o padrão
  Keep a Changelog, uma convenção consolidada em inglês). Ver §9.

## 9. Changelog (formato e política)

Vale para o `changelog.md` de cada repo.

- **Idioma: inglês.** O changelog é a exceção à regra de docs em PT-BR (§8): segue o padrão
  [Keep a Changelog](https://keepachangelog.com) + [SemVer](https://semver.org), convenções
  consolidadas em inglês.
- **Ordem cronológica invertida:** o mais **recente fica no topo**; o mais antigo, embaixo.
  Toda alteração nova entra **acima** das anteriores.
- **SemVer:** versões no formato `vMAJOR.MINOR.PATCH`.
- **Trabalho não publicado fica em `## Unreleased`** (sempre o bloco do topo).
  Enquanto não há commit/PR, todas as mudanças se acumulam aí — **sem data e sem versão**.
- **No commit/PR**, o bloco `## Unreleased` é renomeado para
  `## [dd-MM-yyyy - vX.Y.Z]` (ex.: `## [04-06-2026 - v0.0.1]`), e um novo
  `## Unreleased` vazio é aberto acima dele para as próximas mudanças.
- **Uma linha por PR:** cada entrada resume, em **uma única linha**, o que o PR entrega —
  generalista e focada no que foi implementado, **sem detalhes de implementação nem do
  framework de docs**; referencie o PR (ex.: `[PR#02](url)`). Sem subdivisão por categoria.

## 10. Convenções de diagramas

- Padrão: **Mermaid embutido no `.md`** (texto, versionável, editável pela IA, renderiza no
  GitHub). Não usar imagem/PNG/Figma como documento canônico.
- Cada diagrama mora **no documento da camada que ele descreve**:
  - **Contexto/containers (C4 nível 1–2)** → no `ADR`: topologia de serviços e sistemas
    externos (ex.: mobile → Firebase → API → DB). Escopo arquitetural, não por feature.
  - **Sequência cross-repo** → no `AYD`: fluxo ponta a ponta de uma feature; torna o
    contrato visual.
- **Subordinação:** o diagrama ilustra; a fonte da verdade do contrato continua sendo o
  texto/tabela do AYD ou ADR. Se divergirem, o texto vence.
- **Ciclo de vida:** o diagrama herda o do doc-pai (vivo em AYD; congela em ADR).
- **Propagação:** ao alterar contrato ou fluxo num doc, **atualize o Mermaid correspondente
  na mesma edição**.
