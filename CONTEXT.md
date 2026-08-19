# Author Portal

The author-facing SaaS portal for publishing houses. This context currently covers the v1 authoring slice: authors write and revise manuscripts in the portal, and the portal is the manuscripts' system of record for now (see ADR-0001).

## Language

**Publisher**:
The tenant — a publishing house whose authors use the portal. Maps to an odon organization; every stored row is scoped to exactly one Publisher.
_Avoid_: tenant, customer, client

**Manuscript**:
The written work an author creates and revises in the portal — an ordered container of Sections. In v1 it is written in the portal itself, not uploaded (see ADR-0002).
_Avoid_: document, content, text

**Section**:
A self-contained, ordered part of a Manuscript, written and versioned as one unit — a trade book's chapter, a legal commentary's §. Each Section is one editor document (see ADR-0003).
_Avoid_: chapter, part, document

**Editor**:
The publisher-side person assigned to a Manuscript who reviews, comments, and makes Suggestions. Always the person — the writing surface is always called "knkeditor", never "the editor".
_Avoid_: reviewer, staff

**Comment**:
A threaded remark anchored to a passage of a Section, exchanged between Author and Editor and resolvable.
_Avoid_: annotation, note

**Suggestion**:
A tracked change proposed in a Section (by an Editor or the Author) that the Author accepts or rejects.
_Avoid_: tracked change, redline

**Draft**:
The single mutable working state of a Section, continuously autosaved as the author writes. Creating a Version copies the Draft; the Draft keeps evolving.
_Avoid_: working copy, WIP

**Version**:
An immutable, numbered snapshot of a Section, created deliberately by an author or a workflow event, with an optional comment (see ADR-0004).
_Avoid_: revision, snapshot, save
