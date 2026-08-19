# v1 collaboration is asynchronous; concurrent edits are guarded, not merged

Real-time co-editing stays out of v1 (someone will suggest adding Yjs in six months — this records why not yet): knkeditor has no CRDT/realtime layer, and its Suggestion and Comment marks were designed without CRDT semantics, so concurrent-editing support would put the programme's hardest unsolved problem on the pilot's critical path for a capability the author↔Editor loop — asynchronous by nature — doesn't need. Instead, v1 guards concurrency: a Section-level soft lock with presence ("Editor is editing this Section") plus stale-baseline rejection on autosave (taskhub's proven draft-conflict pattern), so two-party access never becomes last-write-wins data loss.

**Status:** accepted

**Consequences:** When realtime comes (Phase 3), it is a contained migration, not a rewrite: the Draft path moves to CRDT update-stream persistence, while Versions — immutable plain-JSON snapshots — and the diff-viewer are untouched, and per-Section documents (ADR-0003) bound each collab session. messengerhub's realtime spine (outbox → NATS → gateway) is the house pattern to copy. Raise CRDT semantics for suggestion/comment marks with the knkeditor team before starting that work.
