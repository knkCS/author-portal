# v1 ships as one hub-template repo: authorhub

The portal is built as a single new repo in the knkcs org, `knkcs/authorhub` (Go module `github.com/knkcs/authorhub`), on the established hub template (messengerhub is the reference skeleton): one Go service — Connect-RPC, Ent/Atlas/Postgres, koanf, `knkcs/commons` — with the npm-workspace layout (`packages/authorhub-ui` feature library, thin `web/` shell embedded via `go:embed`), odon OIDC relying-party wiring, Helm chart, shared `go-service-ci`, release-please. This supersedes the approved spec's §9 two-repo structure (`author-portal-web` CDN-served SPA + `author-portal-backend`).

**Status:** accepted — supersedes design spec 2026-04-14 §9 (and the §8 CDN-hosting row)

**Considered options:**

- Two repos per the spec — rejected: two repos' CI/release/deploy ceremony plus a CORS/session boundary for a 1–2 engineer pilot, and no existing knkCS repo demonstrates that shape; we'd invent instead of copy. Runtime white-label theming (spec §6.3) needs nothing from a CDN — branding is fetched per hostname at startup either way.
- Code inside the knkCS/author-portal docs repo — rejected: mixes program-level docs (pricing, market research, ERP roadmap) into a service repo and breaks the repo-equals-deployable-service convention.

**Consequences:** knkcs/author-portal stays the program-level home (research, specs, cross-service decisions); authorhub gets its own CONTEXT.md, ADRs, and agent-skills setup when created, and implementation tickets land there. v1 has no sync worker (no ERP in scope); if ERP sync returns, it comes back as a second entrypoint in the same repo, which the template supports.
