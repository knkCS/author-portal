# A v1 manuscript is written in the portal as a knkeditor document; file upload is deferred

Authors write manuscripts in-portal using the `@knkcms/knkeditor-*` packages (TipTap 3); the canonical manuscript form is the editor's ProseMirror JSON, versioned by the portal's own store. Uploading existing files (Word/PDF — feature requirement 1.1, a MUST) is deferred to a later slice.

**Status:** accepted

**Considered options:**

- Upload-only v1 (mediahub-style blob versions) — rejected: writing in-portal is the differentiating capability and the stated v1 focus.
- Both from day one, one store with text and blob artifact kinds — recommended in the interview, rejected by the decision-maker to keep the first slice small.
- Written-only v1 — **chosen**.

**Consequences:** v1 cannot onboard an author who arrives with a Word manuscript; requirement 1.1 must return in a later slice, so the store's versioning model must not preclude adding blob-kind versions (no schema shape that assumes payloads are always JSON). Adopting knkeditor pins the portal frontend to React 19 + Chakra v3 (anker-compatible) with i18next, and CI needs GitHub Packages auth for the `@knkcms` scope. knkeditor deliberately ships no persistence (`onUpdate`/`onBlur` only) — autosave, dirty state, and version creation are wholly the portal's responsibility. Its client-side diff-viewer takes two JSON documents, so version compare can be driven directly from the store.
