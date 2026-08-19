# Author Portal

The author-facing SaaS portal for publishing houses. This context currently covers the v1 authoring slice: authors write and revise manuscripts in the portal, and the portal is the manuscripts' system of record for now (see ADR-0001).

## Language

**Manuscript**:
The written work an author creates and revises in the portal — an ordered container of Sections. In v1 it is written in the portal itself, not uploaded (see ADR-0002).
_Avoid_: document, content, text

**Section**:
A self-contained, ordered part of a Manuscript, written and versioned as one unit — a trade book's chapter, a legal commentary's §. Each Section is one editor document (see ADR-0003).
_Avoid_: chapter, part, document
