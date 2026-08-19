# Section content lives in Postgres, not object storage

Draft and Version payloads (ProseMirror JSON, typically tens to a few hundred KB) are stored as columns in Postgres via Ent, taskhub-spec-version style — not in Azure Blob behind mediahub's presigned-upload pattern. At v1 scale (a prolific pilot is ~15 GB) Postgres handles this comfortably, and it makes a snapshot one transactional insert instead of a presign→PUT→finalize handshake with pending-row garbage and non-transactional deletes, with no Azurite in every dev/test setup.

**Status:** accepted

**Considered options:**

- Object storage (mediahub-style) — rejected for JSON payloads: right for opaque, unbounded binaries (which future file upload will be), buys nothing for structured text.

**Consequences:** The store's seam takes and returns payloads without exposing where they live. When file upload returns (ADR-0002), blob-kind Versions get an object-storage backend behind the same seam; text Versions stay in Postgres. If a pathological manuscript ever exceeds sane row sizes, that's a per-kind backend decision, not a schema rewrite.
