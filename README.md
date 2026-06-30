# Repo de Contexto

Camada compartilhada da documentação do produto (visão, requisitos, design cross-repo,
roadmap e decisões). É a **fonte única da verdade** consumida pelos repos de serviço.

- Como escrever/evoluir docs: ver `_meta/conventions.md`.
- Mapa de tudo: ver `manifest.md`.
- Para IAs: ver `CLAUDE.md`.
- Orquestração da cascata (PROD → REQ → AYD → SPEC@repo) por IA: skill `cascade`
  (`.claude/skills/cascade/`) + subagentes `doc-explorer` e `spec-author` (`.claude/agents/`).

Os repos de serviço (api, web, mobile) espelham este repo em `docs/shared/` (read-only)
via o `sync-context.sh` de cada um. Contratos mudam **apenas aqui**, nos AYD/ADR.
