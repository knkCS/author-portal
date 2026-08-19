# v1 authenticates against odon from day one; guardian is deferred behind an authz seam

authorhub requires odon OIDC bearer auth from its first commit (relying-party wiring copied from messengerhub), with no auth-disabled mode — reaffirming taskhub's ADR 0002 lesson that stubbed auth touches every handler on the way out. Authorization in v1 is a plain ownership check (an author edits their own Manuscripts) behind a small internal authz seam; guardian arrives behind that same interface when multi-role workflows (editor review, publisher admin) enter, since v1's single relationship doesn't need relation tuples and guardian's previously planned portal schema predates the Manuscript/Section model anyway.

**Status:** accepted

**Considered options:**

- Real odon + real guardian immediately — rejected: schema redesign cost now for roles that don't exist in this slice.
- Stub auth — rejected: contradicts the recorded house lesson; retrofit cost lands on every handler.

**Consequences:** Version attribution (who created which Version, when) rides on real odon identities from the start. Pilot accounts are created manually in odon's admin — a pilot-only debt; author invitations remain the odon roadmap item flagged as critical path in the 2026-04-30 roadmap. When guardian lands, the authz seam is the only code that changes.
