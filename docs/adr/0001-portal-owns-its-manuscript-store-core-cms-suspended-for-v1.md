# The portal owns its manuscript store; core CMS as content hub is suspended for v1

The approved design spec (2026-04-14) makes core CMS the content hub — manuscripts, versioning, and the editor all live there, with the portal as a thin gateway. For the v1 authoring slice we decided the portal owns a real, versioned manuscript store of its own, behind a small storage seam (mediahub-style). Core is a beta monolith midway through its odon-auth migration; coupling v1 to it would put a moving dependency on the critical path while the slice only needs writing and versioning.

**Status:** accepted

**Considered options:**

- Temporary stand-in that mimics core's blueprint semantics for a cheap later swap — rejected: couples the design to a moving target without gaining any of its features.
- Portal-owned store behind a small seam, future open — **chosen**.
- Throwaway prototype storage — rejected: versioned file handling is product infrastructure, not validation glue.

**Consequences:** If manuscripts later move into (or sync with) core CMS, that arrives as a swapped backend or sync behind the same seam, plus a data migration — append-only versioning is chosen partly to keep that migration tractable. The spec's "CMS as universal data hub" rows for manuscripts (§2.2, §3.1) are suspended for v1, not overturned.
