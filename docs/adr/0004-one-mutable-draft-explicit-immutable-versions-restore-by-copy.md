# A section has one mutable draft and explicit immutable versions; restore is by copy

Each Section has exactly one Draft that autosave overwrites continuously, and Versions are created only at deliberate moments — an author snapshots with an optional comment, or a workflow event (e.g. submitting to an editor) pins one. Version history is strictly append-only: there is no in-place restore; restoring an old Version copies its content into the Draft as a new starting point. We chose this over version-per-save (history becomes hundreds of near-identical rows that drown the meaningful moments) and over draft+versions with automatic GC'd checkpoints (retention/GC machinery v1 doesn't need — the continuously-saved Draft already provides crash-safety).

**Status:** accepted

**Consequences:** Auto-checkpoints can be added later as Versions with a `system` origin — additive, no schema change. The version list stays human-scale, so the diff-viewer always compares meaningful states. Nobody should later add in-place restore or history rewriting; append-only is the invariant the future core-CMS migration path (ADR-0001) leans on.
