# AGENTS.md — olb-whitepaper

## Scope

This repository holds design documents, ADRs, compose profiles and the
acceptance suite. There is no runnable product code here; implementation lives
in `../olb-api`, `../olb-web` and `../olb-scorer`.

## Documentation rules

- Hard-to-reverse decisions get an ADR in `docs/adr/`; do not rewrite a
  published ADR, supersede it.
- The deployment profiles (`basic`, `demo`, reserved `vision` and `sim`) are
  documented as stated intention; do not describe unbuilt features as shipped.
- Conventional Commits; sign-off (DCO).

## Language

Code, identifiers, comments, file and directory names, route segments, API paths, schema fields and
commit messages are English. Product/UI copy is Spanish.
Routes are English, kebab-case nouns: `/games/[id]/score`, `/teams/[slug]`,
`/games/[id]/verify`, `/new`. Never `/juego/[id]/anotar`, `/equipos/[slug]` or `/nuevo`.
Do not add Spanish redirects for old routes; update the links.
