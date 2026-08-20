# Author Portal

## Project Overview
A SaaS author portal solution for publishing houses to provide integrated collaboration with their authors.

## Development Guidelines
- Use clear, descriptive commit messages
- Keep documentation in the `docs/` directory
- Research and specs go in `docs/research/` and `docs/superpowers/specs/` respectively

## Project Structure
```
docs/
  research/
    competitors/     - Competitor analysis and existing solutions
    features/        - Feature requirements and analysis
    market/          - Market research and trends
  superpowers/
    specs/           - Design specifications
```

## Agent skills

### Issue tracker

The live tracker is **knkcs/authorhub** — the single source of truth since 2026-08-20, when this repo's open issues were transferred there. File and work issues on authorhub (`gh ... --repo knkCS/authorhub`); this repo's own issues are the closed v1 tickets, kept as history. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
