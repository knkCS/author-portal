# A Publisher's portal tenancy is carried by one odon workspace

A Publisher is an odon organization — that stays true at the level where it matters (licensing grants the `authorhub` audience per organization; members are administered there). But the rows authorhub stores are scoped by the odon **workspace** claim: one designated workspace under the Publisher's organization carries the portal, created at onboarding. Decided when the scaffold's code review surfaced the mismatch as an undocumented deviation (#12): `active_workspace` is the unit the house auth model resolves on every request, and every sibling service — commons' context plumbing, taskhub's entscope, messengerhub's boundary — scopes by it, so org-scoping would make authorhub the one knk service with bespoke authorization machinery and would turn ticket #3's entscope port into a redesign.

**Status:** accepted — resolves author-portal#12

**Considered options:**

- Rework authorhub to scope rows by `org_id` (derived from the claim) — rejected: sole deviator from the sibling pattern, custom scoping machinery, and a workspace switcher rendered meaningless inside the portal, for a distinction the pilot cannot observe (one publisher, one workspace).
- Bless the workspace as carrier — **chosen**.

**Consequences:** Workspaces are never used to model structure *inside* a Publisher — imprints and divisions live in the portal's own domain model (guardian, phase 2), as the original spec planned. If an organization ever runs more than one portal workspace, each is a hard-walled tenancy silo by construction — a deliberate property, not a bug to fix. The entscope port in ticket #3 proceeds unchanged.
