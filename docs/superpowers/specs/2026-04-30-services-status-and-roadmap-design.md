---
title: "Services Status & Roadmap"
---

# Author Portal — Services Status & Roadmap

> **Date:** 2026-04-30
> **Status:** Draft
> **Authors:** jiwanovski87
> **Repo:** knkCS/author-portal
> **Supersedes:** none — companion to [2026-04-14-author-portal-design.md](2026-04-14-author-portal-design.md)

---

## 1. Purpose

Two customers have expressed interest in starting an author-portal project. This document answers the question **"How long do we need to have the author portal in place?"** by:

1. Summarising the current state of every service in the knk ecosystem the portal depends on
2. Establishing the v1 scope and the philosophy for closing prerequisite gaps
3. Proposing a sequencing approach that fits a 1–2 engineer team using Claude Code
4. Naming the risks and producing a defensible pilot-launch estimate

Detailed architecture, data model, and feature decisions remain in [2026-04-14-author-portal-design.md](2026-04-14-author-portal-design.md). This document is the *roadmap* layered on top of that spec.

---

## 2. Customer Context

| Customer | ERP (publishing) | ERP (financials) | v1 strategy |
|---|---|---|---|
| Customer 1 | knk on Business Central | Business Central | BC sync adapter (titles, editions, production, contacts) |
| Customer 2 | Klopotek | SAP | CMS-native pilot (manual title metadata in core); Klopotek adapter deferred to Phase 2 |

**v1 scope decision:** Publishing process only. Royalties, contracts, and financials are deferred. This means SAP integration is out of scope for v1, and the BC adapter only covers ~4 entity classes instead of the full 8 from the original spec.

---

## 3. Current Status of Dependencies

A survey of the knk ecosystem (April 2026) found the following:

| Service | Repo | Maturity | Ready for portal v1? |
|---|---|---|---|
| **anker** (UI library) | `/repo/anker` | v0.0.6, ~25 components, full Storybook, runtime theming, RHF + Zod integration | ✅ Yes — no blockers |
| **odon** (IDP) | `/repo/odon` | Beta, active dev. OAuth2/OIDC/MFA/orgs/multi-tenant subdomains all implemented | ⚠️ Member invitations are roadmap Phase 3. No SDK (Connect-RPC only) |
| **guardian** (permissions) | `/repo/guardian`, `/repo/guardian-go`, `/repo/guardian-ent` | Feature-complete: tuples, subject sets, schemas, batch checks, ent hooks | ⚠️ Author-portal schema needs to be defined (publisher / imprint / title / manuscript / contract) |
| **core** (CMS) | `/repo/core` | Beta, ~3,800 commits, monolithic. odon-auth migration in flight (sub-projects #1–#4) | ⚠️ Auth migration must complete before portal reads from core in earnest. Workspaces already decoupled from auth — usable today |
| **mediahub** | `/repo/mediahub` | Alpha, ~170 commits. Standalone; core consumes via AssetPicker module federation | ✅ Usable as-is. Collection-service extraction deferred to Phase 2 |
| **fieldkit** | `/repo/fieldkit` | Existing | Assumed ready — verify when first form lands |

**What does not yet exist:**

- `author-portal-web` (frontend) — to be built
- `author-portal-backend` (Go API + sync worker) — to be built. [Plan 1](../plans/2026-04-14-foundation-backend.md) is already written
- knk Business Central sync adapter — greenfield
- Standalone Notification Service — notification functionality currently lives inside core; no standalone service exists. For v1 it will be embedded inside portal-backend (see §6)
- Klopotek adapter — Phase 2

---

## 4. Strategic Philosophy

> Build dependencies as proper standalone services first, but parallelize where possible and accept one well-chosen stub.

Two extreme alternatives were considered and rejected:

| Approach | Why rejected |
|---|---|
| **Strict sequential prereqs first** (microservices-first, no parallelism, no stubs) | Adds 2–3 months of zero customer-visible progress. Risk of customers losing patience or signing elsewhere |
| **Stub-everything pilot first** (skip odon invitations, skip guardian schema, skip Notification extraction; use only what exists) | Accumulates debt across 4–5 dimensions that all have to be paid back before customer #3 |

**Chosen approach: interleaved tracks** — finish the cheap prerequisites properly (guardian schema, finish odon invitations), build the BC adapter and portal-backend in parallel with the portal-web shell going against mocks, and stub exactly one thing (the Notification Service) inside portal-backend for v1.

---

## 5. Sequencing & Critical Path

```
            ┌─ guardian schema (1wk) ─────────────────┐
            │                                         ▼
core auth  ─┤                                 ┌──── portal-backend
migration   │                                 │     foundation (3-4wk)
(in flight) │                                 │       │
            │                                 │       ├──► messaging + notifications (stubbed)
            └─ odon Member Invitations ───────┤       ├──► manuscript handlers
               (2-3wk, on odon roadmap)       │       ├──► production handlers
                                              │       └──► proof/cover handlers
                                              │
                          BC sync adapter ────┤
                          (6-8wk, parallel)   │
                                              ▼
                                       portal-backend integrated
                                                             │
   portal-web shell + theme + auth ──► screens on mocks ──► wired against backend
   (2-3wk)                              (4-6wk, parallel)    (4-6wk)
                                                             │
                                                             ▼
                                                      Customer 1 pilot (BC)
                                                      Customer 2 pilot (CMS-native)
```

**Critical path:** odon Member Invitations is the largest external dependency. If it slips, portal pilot launch slips with it. Worst-case fallback is a portal-side invitation flow built as conscious debt, but the right answer is to promote invitations on odon's roadmap.

**Conscious debt taken in v1 (each with a Phase-2 payoff plan):**

1. **Notification Service** embedded in `author-portal-backend` rather than extracted as a standalone microservice
2. **Klopotek adapter** not built — customer 2 enters title metadata directly into core CMS during pilot
3. **SAP integration** not built — financials are out of scope

---

## 6. Service Inventory & Ownership for v1

| # | Service | Status | Owns in v1 | Work needed |
|---|---|---|---|---|
| 1 | **author-portal-web** | Doesn't exist | Author + publisher-admin SPA | Build greenfield (Plan 2) |
| 2 | **author-portal-backend** | Doesn't exist | API gateway, messaging, notifications (embedded), tenant/branding, BC sync worker | Build greenfield (Plans 1 + 3–5) |
| 3 | **odon** | Beta, active | Auth, OAuth2/OIDC/MFA, organizations, member invitations | Add Member Invitations (currently Phase 3 of odon roadmap) |
| 4 | **guardian** | Feature-complete | Permission checks for publisher / imprint / title / manuscript / contract / proof | Define the author-portal schema (~1 week, then idle) |
| 5 | **core CMS** | Beta, in odon-auth migration | Manuscripts, blueprints, workflow engine, content versioning, media library, ERP-synced blueprints | Finish odon auth migration (already in flight). Define ERP-synced blueprints |
| 6 | **anker** | v0.0.6, ready | Shared UI components, runtime theming | None for v1 |
| 7 | **fieldkit** | Existing | Forms (questionnaires, profile metadata) | Verify when first form lands |
| 8 | **mediahub** | Alpha | Asset management, AssetPicker | None for v1; Collection extraction is Phase 2 |
| 9 | **Notification Service** | Doesn't exist | Email + in-app notifications | **Embedded in portal-backend for v1.** Extract as standalone service in Phase 2 |
| 10 | **knkBC adapter** | Doesn't exist | Sync titles / editions / production / contacts from BC into core blueprints | Build greenfield — part of `cmd/sync` in portal-backend for v1 |
| 11 | **Klopotek adapter** | Doesn't exist | Sync from Klopotek (customer 2) | **Deferred to Phase 2.** Customer 2 runs CMS-native during pilot |
| 12 | **SAP integration** | Doesn't exist | Customer 2 financials | **Deferred indefinitely** — re-evaluate when royalties/contracts come back into scope |

**Two work tracks:**

- **Track A — Backend & ecosystem:** owns #2, #3 (invitations), #4 (schema), #5 (blueprints), #9 (stub), #10 (BC adapter)
- **Track B — Frontend:** owns #1, theming, mocked-then-real wiring against #2

---

## 7. Timeline

All estimates are for **Phase 1 publishing-only pilot** (the scope defined in §2). All assume Claude Code as standard tooling.

| Resourcing | Without Claude Code | **With Claude Code** |
|---|---|---|
| 1 engineer | 9–12 months | **5–7 months** |
| 2 engineers | 5–7 months | **3–4 months** |

The Claude Code multiplier (≈ 1.5–2.5×) reflects:

- Mockups for dashboard, title detail, production, messages, content editor, proof review, royalties already exist — frontend work is largely translation, not invention
- Plan 1 (foundation backend) is already written in superpowers format and executable via `subagent-driven-development`
- odon, anker, guardian-go all have clear integration patterns to copy from
- Greenfield Connect-RPC + Ent + React work benefits more from agentic coding than complex codebase modification

**Pilot definition:** Customer 1 live in production with full BC-synced title/production data; customer 2 live in CMS-native mode with manual title entry. Authors can log in, see their titles, collaborate on manuscripts, view production status, exchange messages, and review proofs/covers.

---

## 8. Risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | **odon Member Invitations slips** — portal can't onboard authors without it | Promote it to "blocks author-portal pilot" in odon's roadmap. Worst-case fallback: portal-side invitation service (debt) |
| 2 | **core's odon-auth migration drags** — portal needs core to authenticate against odon for blueprint reads | Coordinate so core auth migration completes before portal-backend Plan 4 (manuscripts) starts |
| 3 | **BC adapter complexity underestimated** — webhooks are best-effort, rate limits, multi-tenant Azure AD app registration per customer | Build the scheduled poller first as the source of truth; add webhooks as optimization |
| 4 | **Customer 2 disengages because they "only see CMS-native mode"** | Frame CMS-native as the pilot configuration; commit Klopotek adapter as a Phase 2 deliverable with a date |
| 5 | **Notification Service stub becomes permanent** — debt accrues with every new notification type | Time-box the stub: extract before customer #3 onboards. Write the extraction plan now so it isn't forgotten |
| 6 | **Two-customer parallelism stresses 1–2 engineers** | Pick a "lead customer" (the BC one — validates the ERP path) and treat customer 2 as fast-follow |

---

## 9. Recommended Next Steps

1. **Share these timelines with both customers** with the v1 scope made explicit (publishing process, no financials)
2. **Coordinate with odon team** on Member Invitations — this is the first thing that has to move
3. **Define the guardian schema** for author-portal types (1 week, isolatable, can start this week)
4. **Write Plans 2–5** for portal-backend and portal-web (Plan 1 already exists)
5. **Start executing Plan 1** (foundation backend) in parallel with the above

---

## 10. Headline Answer

> **Pilot launch (Phase 1 — publishing process, no financials): 3–4 months at 2 engineers, 5–7 months at 1 engineer**, all with Claude Code, on top of finishing one prerequisite in odon (Member Invitations). Customer 1 launches with full BC-synced data; customer 2 launches in CMS-native mode and gets full Klopotek integration in a follow-on phase. Royalties, contracts, and financials are a Phase 2 effort starting after pilot validation.

---

## Reference

- [Original Author Portal Design Specification (2026-04-14)](2026-04-14-author-portal-design.md)
- [Plan 1: Foundation & Backend (2026-04-14)](../plans/2026-04-14-foundation-backend.md)
- [Feature Requirements](../../research/features/requirements.md)
- [Mockups](../../mockups/)
