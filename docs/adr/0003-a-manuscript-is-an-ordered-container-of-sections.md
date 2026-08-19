# A manuscript is an ordered container of sections; the section is the versioned unit

A Manuscript is an ordered container of Sections, and each Section is its own knkeditor (ProseMirror JSON) document — one editor instance edits one Section. The store versions Sections as immutable snapshots; a manuscript-level version is a manifest pinning one snapshot per Section. We chose this over one-document-per-manuscript because TipTap and the client-side diff-viewer degrade at real manuscript lengths (80–150k+ words), because the content is naturally sectioned (trade chapters, commentary §§ — the content-editor mockup is a single §), and because per-section workflows (locking, per-chapter co-authors, review) become possible without a data migration.

**Status:** accepted

**Considered options:**

- One knkeditor document per manuscript, chapters as collapsible headings — rejected: editor and diff performance ceilings would appear at pilot-realistic manuscript sizes, and splitting later is a painful data migration plus editor-UX rework.
- Container of Sections — **chosen**.

**Consequences:** The store needs a container entity, section ordering, and a manifest layer for manuscript-level versions — more schema than the single-document model. Cross-section operations (whole-manuscript search, export, word count) must aggregate over Sections.
