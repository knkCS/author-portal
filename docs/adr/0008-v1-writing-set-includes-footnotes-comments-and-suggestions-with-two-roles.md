# The v1 writing set includes footnotes, comments, and suggestions, with two roles

v1 ships knkeditor's core writing capabilities plus footnotes, Comments, and Suggestion mode — and therefore two roles: the Author, and a publisher-side Editor assigned per Manuscript (a plain assignment row; the ADR-0006 authz seam checks "owner or assigned Editor"). Content references are out (their resolution UI is core-CMS-side, and with no CMS connection there is nothing to reference). We took the two-party capabilities into the first slice because the product's pitch is publisher–author collaboration, and the marginal cost over author-only is one assignment entity and one authz clause.

**Status:** accepted

**Considered options:**

- Core writing + footnotes only (interview recommendation) — rejected by the decision-maker: v1 should demo the collaboration loop.
- Comments/suggestions as single-user tools — rejected: collaboration furniture nobody sits on.

**Consequences:** knkeditor stores only anchors for footnotes and comment threads — the bodies are portal-side entities, and footnote bodies must snapshot together with their Section so restored Versions never carry orphaned anchors (comment threads stay live entities referenced by anchors; a snapshot may reference threads that evolved afterwards, which the UI must tolerate). Suggestion marks live inside the document JSON and version with it. Scope fences for v1: no notifications, no approval workflow, no publisher-admin UI — Editor assignment is a pilot-operations action. Editor accounts are created manually in odon, like authors (ADR-0006).
