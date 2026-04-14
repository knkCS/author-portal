# Author Portal — Design Specification

> **Date:** 2026-04-14
> **Status:** Approved
> **Authors:** jiwanovski87
> **Repo:** knkCS/author-portal

---

## 1. Executive Summary

A SaaS author portal for publishing houses to provide integrated collaboration with their authors. The portal serves as a single pane of glass for the author-publisher relationship: manuscript collaboration, production tracking, royalty reporting, contract visibility, and communication.

**Target market:** Mid-to-large trade publishers, starting with knk ERP customers and expanding to other publishing ERPs.

**Competitive positioning:** No standalone SaaS author portal exists for trade publishers. Legacy ERP add-ons (Klopotek, Vista) are expensive and UX-dated. Drupal-based custom builds carry 2-5x higher TCO with no publishing-specific advantages. This portal fills the market gap with a purpose-built, modern, affordable solution built on knk's existing microservice ecosystem.

**Approach:** The portal is a purpose-built React frontend + thin Go API gateway that orchestrates the existing knk microservice stack (odon, guardian, core CMS, fieldkit, anker). The core CMS acts as the universal data hub — all ERP data is synced into CMS blueprints, making the portal ERP-agnostic.

---

## 2. Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│              Author Portal Frontend              │
│         React + Anker + Fieldkit                 │
│  (branded per publisher, author-facing UX)       │
└──────────────────────┬──────────────────────────┘
                       │ REST/GraphQL
┌──────────────────────▼──────────────────────────┐
│            Portal API Gateway (Go)               │
│  - Author-specific business logic                │
│  - Notification engine                           │
│  - Messaging                                     │
│  - Publisher branding/config                     │
└──┬───────┬────────┬────────┬────────────────────┘
   │       │        │        │
┌──▼──┐ ┌──▼──┐ ┌───▼───┐ ┌──▼──┐
│odon │ │guard│ │ core  │ │field│
│auth │ │perm │ │ CMS   │ │kit  │
└─────┘ └─────┘ └───┬───┘ └─────┘
                     │
            ┌────────▼────────┐
            │   Sync Layer    │
            │  (ERP Adapters) │
            └──┬─────────┬────┘
           ┌───▼───┐ ┌───▼────┐
           │ knk   │ │Klopotek│
           │ ERP   │ │/ other │
           └───────┘ └────────┘
```

### 2.2 Key Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Frontend | React SPA with anker components | Consistent with knk ecosystem, enables white-labeling via runtime theming |
| API layer | Go service (combined repo, two entrypoints) | Thin orchestration layer, same language as backend stack |
| Auth | odon via OAuth2/PKCE | Already handles invitations, MFA, organizations, GDPR |
| Permissions | guardian (Zanzibar-pattern) | Maps perfectly to author-title-editor relational permissions |
| Content/manuscripts | core CMS | Content versioning, TipTap editor, workflows, media library |
| Forms | fieldkit | Author questionnaires, profile forms, metadata input |
| ERP integration | Adapter pattern via sync layer | knk BC adapter first, pluggable for Klopotek/others later |
| Multi-tenancy | odon organizations + guardian namespacing | Each publisher = one odon organization with isolated data |
| Data hub | core CMS | All ERP data synced into CMS blueprints; portal has one data source |

### 2.3 What's New vs. What Exists

| Component | Status |
|---|---|
| Portal Frontend (React SPA) | **New** — the main deliverable |
| Portal Backend (Go, API + sync) | **New** — thin orchestration + ERP adapters |
| knk BC Adapter | **New** — bidirectional sync with Business Central |
| odon, guardian, core CMS, fieldkit, anker | **Existing** — consumed as services/packages |

---

## 3. Data Model & Ownership

### 3.1 CMS as Universal Data Hub

The portal doesn't own most data — it orchestrates data from existing systems. The core CMS acts as the single data source for the portal. ERP data is synced into CMS blueprints.

| Domain | Source of Truth | Lives in CMS as | Sync |
|---|---|---|---|
| Users, auth, sessions | odon | — | — |
| Permissions | guardian | — | — |
| Titles, ISBNs, editions | ERP | `title` blueprint | ERP → CMS |
| Contracts, rights | ERP | `contract`, `rights-territory` blueprints | ERP → CMS |
| Royalties, sales, payments | ERP | `royalty-statement`, `sales-data` blueprints | ERP → CMS (batch) |
| Production milestones | ERP or CMS (configurable) | `production-milestone` blueprint | Configurable |
| Manuscripts, versions | CMS-native | `manuscript` blueprint | Portal-native |
| Proofs, cover approvals | CMS-native | `proof`, `cover-review` blueprints | Portal-native |
| Media, assets | CMS-native | Media library | Portal-native |
| Workflows, tasks | CMS-native or ERP (configurable) | Workflow engine | Configurable |
| Author profiles | CMS-native | `author-profile` blueprint | CMS → ERP (optional writeback) |
| Messages | Portal API | — | Portal-native |
| Notifications | Portal API | — | Portal-native |
| Branding config | Portal API | — | Portal-native |

### 3.2 Configurable Source of Truth

The source of truth is not fixed — it depends on how deeply each customer uses their ERP. The sync layer supports per-entity, per-tenant configuration.

| Entity | Mode A: ERP is source | Mode B: CMS is source | Mode C: Bidirectional |
|---|---|---|---|
| Titles & editions | ERP → CMS (read) | Unlikely | — |
| Contracts & rights | ERP → CMS (read) | Unlikely | — |
| Royalties & sales | ERP → CMS (read) | Unlikely | — |
| Production milestones | ERP → CMS (read) | CMS-native, no sync | ERP ↔ CMS (merge) |
| Task workflows | ERP drives stages | CMS workflow engine | CMS tasks linked to ERP milestones |
| Author profiles | ERP contacts → CMS | CMS-native, writeback to ERP | CMS ↔ ERP (conflict resolution) |
| Manuscripts | — | Always CMS | — |
| Messages, notifications | — | Always Portal API | — |

Tenant configuration model:

```go
type EntitySyncConfig struct {
    Entity        string    // e.g., "production-milestone"
    SourceOfTruth string    // "erp", "cms", "bidirectional"
    SyncDirection string    // "erp_to_cms", "cms_to_erp", "both"
    ConflictRule  string    // "erp_wins", "cms_wins", "latest_wins", "manual"
    SyncFrequency string    // "realtime", "hourly", "daily", "manual"
}
```

**Principle:** Titles, contracts, royalties — ERP is always the source (financial/legal data of record). Manuscripts, messages — CMS/portal is always the source. Production workflows and author profiles are the configurable zone.

### 3.3 Guardian Permission Model

Zanzibar-style relation tuples map cleanly to publishing relationships:

```
# Relationship tuples
title:978-3-123#author@user:author_jane
title:978-3-123#editor@user:editor_bob
title:978-3-123#production@user:prod_team_alice
imprint:fiction#member@user:editor_bob
publisher:acme#admin@user:admin_chris

# Implied permissions
author on title → can view manuscript, royalties, contracts, production status
editor on title → can view/edit manuscript, see production, assign tasks
admin on publisher → can view all titles, manage users, configure branding
```

---

## 4. Business Central Integration

knk ERP runs on Microsoft Dynamics 365 Business Central. The sync layer connects via BC's REST APIs.

### 4.1 API Surface

knk exposes publishing-specific entities as custom API pages via AL extensions:

```
https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{env}/api/knk/publishing/v1.0/
```

| BC Entity | Maps to CMS Blueprint | Sync Direction |
|---|---|---|
| knk Titles | `title` | BC → CMS |
| knk Editions (Items) | `edition` | BC → CMS |
| knk Royalty Contracts | `contract` | BC → CMS |
| knk Royalty Statements | `royalty-statement` | BC → CMS (batch) |
| knk Sales Data | `sales-data` | BC → CMS (batch) |
| knk Production Orders | `production-milestone` | BC → CMS |
| knk Rights / Territories | `rights-territory` | BC → CMS |
| Contacts (Authors) | `author-profile` | Bidirectional |

### 4.2 Authentication

OAuth2 client credentials flow (service-to-service):

- App registration in Azure Entra ID with `Dynamics 365 Business Central / API.ReadWrite.All`
- Corresponding BC entry under Azure Active Directory Applications with appropriate permission set
- Token auto-refresh via `golang.org/x/oauth2/clientcredentials`
- Token endpoint: `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`
- Scope: `https://api.businesscentral.dynamics.com/.default`

### 4.3 Sync Strategy

Three mechanisms used together:

| Mechanism | Use Case | Frequency |
|---|---|---|
| Scheduled poller | Royalties, sales, contracts | Daily / weekly / per accounting period |
| Webhook receiver | Production status, title metadata changes | Event-driven (with polling fallback) |
| On-demand fetch | Author requests refresh on a specific title | Ad-hoc |

Incremental sync uses BC's `lastModifiedDateTime` filter:

```
GET /api/knk/publishing/v1.0/titles?$filter=lastModifiedDateTime gt 2026-04-13T00:00:00Z
```

BC webhooks are best-effort (not guaranteed delivery), so the scheduled poller acts as a reconciliation safety net.

### 4.4 Rate Limit Handling

BC enforces 600 API calls/min per environment per app:

- Exponential backoff on HTTP 429 (using `Retry-After` header)
- Request batching via `$expand` and `$top=1000`
- Sync scheduling spread across the day to avoid spikes

### 4.5 Multi-Tenant Considerations

Each publishing house customer has their own BC environment or tenant:

```go
type BCTenant struct {
    TenantID        string // Azure tenant ID
    EnvironmentName string // BC environment (e.g., "production")
    ClientID        string // App registration per tenant
    ClientSecret    string // Stored encrypted
    APINamespace    string // e.g., "knk/publishing/v1.0"
}
```

### 4.6 ERP Adapter Interface

```go
type ERPAdapter interface {
    SyncTitles(ctx context.Context, since time.Time) error
    SyncContracts(ctx context.Context, since time.Time) error
    SyncRoyalties(ctx context.Context, period Period) error
    SyncProductionStatus(ctx context.Context, titleID string) error
    WriteBackAuthorProfile(ctx context.Context, profile AuthorProfile) error
    WriteBackMilestone(ctx context.Context, milestone Milestone) error
}
```

The `knkBCAdapter` implements this interface. Future ERP adapters (Klopotek, Vista, etc.) implement the same interface — the CMS and portal never change.

### 4.7 Requirements from knk ERP Side

Since knk owns the ERP product:

1. Custom API pages exposed for all publishing entities the portal needs
2. Webhook subscriptions enabled for production status and title changes
3. A dedicated BC permission set for the portal sync app (read-only on most entities, write on author contacts)

---

## 5. MVP Scope and Phasing

### 5.1 Phase 1 — MVP (Foundation)

Goal: A working portal that authors can log into, see their titles, collaborate on manuscripts, and view production status. Enough to demo to customers and onboard a pilot publisher.

| Category | Features (34 must-haves) | Depends on |
|---|---|---|
| Administration | Multi-tenant setup, branding/white-label, user roles, author invitations, SSO via odon, MFA, audit logging | odon, guardian |
| Author Profile & Communication | Profile management, secure messaging, notification center, notification preferences, contact directory, author questionnaire | Portal API, fieldkit |
| Manuscript Management | Manuscript submission, version control, document categorization, co-author collaboration, editorial brief/project setup, deadline tracking, secure file transfer | core CMS |
| Production Workflow | Production status dashboard, proof review & approval, cover approval, metadata review, publication date tracking, multi-format status, approval audit trail | core CMS (+ ERP sync for status) |

Explicitly deferred from MVP:

- Royalties & financial reporting (requires ERP sync to be solid)
- Contracts & rights (requires ERP sync)
- Marketing features (lowest priority)
- Real-time co-editing (differentiator, high complexity)
- Analytics dashboards
- AI features

### 5.2 Phase 2 — ERP Integration (Financial Transparency)

Goal: The #1 author demand — royalty and sales visibility.

- Royalties: statement access, advance/earn-out tracking, sales by channel/format/territory, subsidiary rights income, payment history, tax documents, reserves against returns
- Contracts: document access, key terms summary, rights overview
- Production: automated stage progression, correction rounds tracking
- Analytics: title performance dashboard, portfolio overview, export/download

Requires the BC sync adapter to be production-ready for financial data.

### 5.3 Phase 3 — Differentiation

Goal: Features that make the portal best-in-class.

- Collaboration: real-time co-editing (TipTap + Yjs), inline commenting/annotation
- Analytics: sales trend visualization, geographic heat map, custom dashboards
- Rights: territory map visualization, reversion tracking
- Financial: near-real-time sales data, royalty projections, statement dispute workflow
- Production: audiobook workflow, digital asset library

### 5.4 Feature Count Summary

| Phase | Features | ERP Dependency |
|---|---|---|
| Phase 1 (MVP) | 34 must-haves | Minimal (production status only) |
| Phase 2 (Growth) | ~25 features | Heavy (royalties, contracts, rights) |
| Phase 3 (Differentiation) | ~15 features | Moderate |
| Deferred / backlog | ~33 features | Varies |

---

## 6. Frontend Architecture

### 6.1 Project Structure

```
author-portal-web/
├── src/
│   ├── app/                    # App shell, routing, providers
│   │   ├── routes/             # Route definitions per feature area
│   │   └── theme/              # Publisher branding (loaded from API)
│   ├── features/
│   │   ├── dashboard/          # Author home — overview of all titles
│   │   ├── titles/             # Title list + detail views
│   │   ├── manuscripts/        # Upload, versions, editor (TipTap)
│   │   ├── production/         # Status pipeline, proof review, approvals
│   │   ├── royalties/          # Statements, sales data (Phase 2)
│   │   ├── contracts/          # Contract viewer, rights (Phase 2)
│   │   ├── profile/            # Author profile, preferences
│   │   ├── messages/           # Secure messaging
│   │   └── admin/              # Publisher admin (branding, users, config)
│   ├── shared/
│   │   ├── api/                # API client (generated from OpenAPI via Orval)
│   │   ├── auth/               # odon OAuth2 PKCE flow
│   │   └── components/         # Portal-specific composites using anker
│   └── main.tsx
├── package.json
└── vite.config.ts
```

### 6.2 Tech Stack

| Concern | Choice | Rationale |
|---|---|---|
| Build tool | Vite | Consistent with core CMS |
| UI components | anker (Chakra UI v3) | Shared knk component library |
| Forms | fieldkit | Author questionnaires, profile forms, metadata input |
| State management | Zustand + TanStack Query | Consistent with core CMS |
| API client | Orval (generated from OpenAPI) | Type-safe, consistent with core CMS |
| Rich text | TipTap (from core CMS) | Manuscript editing, inline comments |
| Auth flow | OAuth2 PKCE via odon | Redirect to odon login, receive token |
| Routing | React Router | Standard |
| i18n | i18next | Consistent with core CMS |

### 6.3 White-Labeling

Publisher branding is fetched at runtime and applied to the Chakra/anker theme:

1. Browser hits `portal.publisher.com`
2. App shell calls `GET /api/tenant/branding`
3. Response: `{ logo, primaryColor, accentColor, fontFamily, publisherName }`
4. Inject into Chakra/anker theme provider
5. Render app with publisher's look and feel

No rebuild needed per publisher — purely runtime theming.

### 6.4 UI Modes

| Mode | Users | Features |
|---|---|---|
| Author view | Authors, co-authors, agents | Dashboard, titles, manuscripts, production status, profile, messages |
| Publisher admin view | Editors, production staff, admins | Everything above + user management, branding config, workflow config, sync settings |

Visibility controlled by guardian permissions — same app, different navigation and capabilities.

---

## 7. Backend Architecture

### 7.1 Project Structure

```
author-portal-backend/
├── cmd/
│   ├── api/                    # API gateway entrypoint
│   └── sync/                   # Sync worker entrypoint
├── internal/
│   ├── api/
│   │   ├── handler/            # HTTP handlers
│   │   │   ├── messaging/      # Author-editor messaging
│   │   │   ├── notification/   # Notification preferences & delivery
│   │   │   ├── branding/       # Tenant branding config
│   │   │   └── proxy/          # Proxy/orchestrate calls to CMS, odon, guardian
│   │   ├── middleware/         # Auth (odon JWT), tenant resolution, permissions
│   │   └── router.go
│   ├── sync/
│   │   ├── engine/             # Sync orchestrator (scheduling, retry, logging)
│   │   ├── adapter/
│   │   │   ├── adapter.go      # ERPAdapter interface
│   │   │   └── knkbc/          # knk Business Central adapter
│   │   └── config/             # Per-tenant EntitySyncConfig
│   ├── domain/                 # Shared domain types
│   ├── storage/ent/            # Ent ORM, PostgreSQL
│   └── clients/                # Service clients (CMS, odon, guardian)
├── openapi/                    # OpenAPI spec
├── go.mod
├── Dockerfile
└── docker-compose.yml
```

### 7.2 Two Entrypoints, One Codebase

| Entrypoint | Runs as | Responsibility |
|---|---|---|
| `cmd/api` | HTTP server | Portal API — frontend requests, proxies to services, messaging & notifications |
| `cmd/sync` | Worker process | ERP sync — scheduled jobs, webhook receiver, writes to CMS |

Both share domain types, storage, and service clients. Deployable as one binary or two.

### 7.3 API Gateway Pattern

The portal API does not duplicate CMS logic:

- `GET /titles` → proxy to CMS (blueprint: title)
- `GET /manuscripts` → proxy to CMS (blueprint: manuscript)
- `GET /production/:id` → proxy to CMS (blueprint: production-milestone)
- `POST /manuscripts` → proxy to CMS (create content)
- `POST /approvals` → proxy to CMS (workflow transition)
- `GET /messages` → portal's own DB
- `POST /messages` → portal's own DB + notification
- `GET /notifications` → portal's own DB
- `PUT /branding` → portal's own DB

The gateway adds value by: simplifying the CMS API for the frontend, enforcing guardian permissions, aggregating data from multiple sources into single responses, and owning messaging/notifications.

### 7.4 Portal Database (Minimal)

| Table | Purpose |
|---|---|
| `tenants` | Publisher config, branding, custom domain, sync settings |
| `messages` | Author-editor conversations, threaded by title |
| `notifications` | Notification log + delivery status |
| `notification_preferences` | Per-user notification settings |
| `sync_jobs` | Sync run history, status, error logs |
| `entity_sync_config` | Per-tenant, per-entity source-of-truth config |

Everything else lives in the CMS, odon, or guardian.

### 7.5 Tech Stack

| Concern | Choice | Rationale |
|---|---|---|
| Language | Go | Consistent with backend stack |
| HTTP | Connect-RPC or net/http | Consistent with odon, guardian |
| ORM | Ent | Consistent with core CMS, odon |
| Database | PostgreSQL | Consistent |
| Cache/queue | Redis | Notification delivery, sync job queuing |
| API spec | OpenAPI 3.1 | Frontend generates typed client via Orval |

---

## 8. Deployment & Infrastructure

```
┌─────────────────────────────────────────────────────────┐
│                    Per-Publisher                          │
│  DNS: portal.publishername.com → Load Balancer           │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                  Shared Infrastructure                    │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Portal Web   │  │ Portal API   │  │ Portal Sync   │  │
│  │ (static SPA) │  │ (Go, HTTP)   │  │ (Go, worker)  │  │
│  │ CDN-served   │  │ stateless    │  │ per-tenant    │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
│                                                          │
│  ┌───────┐ ┌────────┐ ┌──────┐ ┌────────┐ ┌───────┐    │
│  │ odon  │ │guardian │ │ core │ │fieldkit│ │ Redis │    │
│  └───────┘ └────────┘ │ CMS  │ └────────┘ └───────┘    │
│                       └──┬───┘                           │
│  ┌───────────┐    ┌──────▼──────┐                       │
│  │ Portal DB │    │   CMS DB    │                       │
│  │ (Postgres)│    │ (Postgres)  │                       │
│  └───────────┘    └─────────────┘                       │
│  ┌──────────────────┐                                   │
│  │ Azure Blob Store │ (media, manuscripts, proofs)       │
│  └──────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

| Concern | Approach |
|---|---|
| Frontend hosting | Static SPA on CDN, branding loaded at runtime |
| API | Stateless Go service, horizontally scalable |
| Sync worker | Separate process, scales per tenant count |
| Multi-tenancy | Shared infrastructure, data isolation via tenant IDs + guardian |
| Custom domains | Reverse proxy maps publisher domains to shared API, tenant resolved from hostname |
| Containerization | Docker, compose for local dev |
| CI/CD | GitHub Actions |

---

## 9. Repo Structure

| Repo | What | Language |
|---|---|---|
| `knkCS/author-portal` | Docs, specs, plans, research | — |
| `knkCS/author-portal-web` | Portal frontend SPA | React + TypeScript |
| `knkCS/author-portal-backend` | Portal API Gateway + sync layer | Go |
| existing repos | odon, guardian, core, fieldkit, anker | Consumed as services/packages |

Backend starts as a single repo with `cmd/api` and `cmd/sync`. Split into separate repos if/when deployment or team ownership requires it.

---

## 10. Arguments Against Drupal

For the customer conversation, the key arguments are:

1. **No publishing data model** — Drupal has no concept of titles, ISBNs, royalty periods, rights territories. Everything must be custom-built.
2. **No ERP connectors** — No Drupal modules exist for any publishing ERP including Business Central. Full custom integration work regardless.
3. **No real-time collaboration** — Drupal's PHP request-response architecture fundamentally doesn't support Google Docs-style editing.
4. **Weak permissions model** — Drupal's RBAC cannot express "author A can see title X because they have a contract" without fragile custom access hooks. Zanzibar-pattern (guardian) handles this natively.
5. **No multi-tenancy** — Drupal was designed as a single-site CMS. SaaS multi-tenancy requires fundamental architectural modifications.
6. **5-year TCO: $500K-$1.5M** (Drupal) vs. **$100K-$400K** (purpose-built SaaS).
7. **Industry trend** — No major publisher uses Drupal for author-facing portals. Springer Nature, Wiley, Elsevier all use purpose-built systems.
8. **Talent scarcity** — Finding developers who know both Drupal and publishing domain is expensive and getting harder.

**Key question for the customer:** "Do you want to pay to build a publishing platform on a general-purpose CMS, or use a platform that was built for publishing from day one?"

---

## 11. Reference Research

Detailed research documents are available in the repository:

- `docs/research/competitors/market-overview.md` — 20+ solutions analyzed, market gaps identified
- `docs/research/competitors/drupal-analysis.md` — Drupal strengths, limitations, TCO analysis
- `docs/research/features/requirements.md` — 107 features across 8 categories with prioritization
