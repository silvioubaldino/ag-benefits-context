# Contexto do Produto (FONTE ÚNICA)

Repositório canônico da **camada compartilhada** do produto.
Consumido em modo **read-only** pelos repos de serviço (api, web, mobile),
que o espelham em `docs/shared/`.

## Comece por
- @manifest.md — mapa de todos os documentos
- @_meta/conventions.md — IDs, frontmatter, ciclo de vida, propagação, refs `ID@repo`
- @_meta/glossary.md — linguagem ubíqua (use SEMPRE estes termos)

## O que mora aqui (compartilhado)
- **PROD** (`product.md`) — visão & estratégia
- **REQ** (`requirements.md`) — requisitos do produto
- **AYD** (`design/`) — Análise & Design por feature (cross-repo: repos afetados + contratos)
- **ROAD** (`roadmap.md`) — roadmap / planejamento
- **PDR** (`product_decisions/`) — decisões de produto
- **ADR** (`architecture_decisions/`) — decisões de arquitetura cross-repo (contratos, protocolos)

## O que NÃO mora aqui
Specs, plans, decisões técnicas locais (TDR), convenções de código e changelog de cada serviço.

## Regra central (cross-repo)
- **1 AYD → N SPECs**, uma por repo afetado. O AYD é a fonte dos contratos.
- IDs globais; referência cross-repo `ID@repo` (ex.: `SPEC-012@api`).
- **Contrato só muda aqui** (no AYD/ADR). Serviços implementam, não redefinem.

## Ciclo de vida
- PROD / REQ / AYD / ROAD = vivos (edita in-place; registra em `_meta/changelog.md`).
- PDR / ADR = append-only (decisão nova substitui a antiga via `superseded_by`).
- Ao mudar algo, marque os `children` afetados (inclusive em outros repos) como `status: review`.

## Outras ferramentas
- Claude Projects: conecte este repo via integração GitHub e use **Sync**.
- NotebookLM / Gemini Gems: rode `scripts/bundle.sh` → `CONTEXT.md` para upload.
