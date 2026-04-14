# knk ERP Data Model Analysis & CMS Blueprint Specifications

> **Version:** 1.0  
> **Date:** 2026-04-14  
> **Status:** Research & Discovery  
> **Audience:** Engineering (CMS Configuration, Sync Adapter Development), Product  
> **Prerequisites:** [Author Portal Design Specification](../../superpowers/specs/2026-04-14-author-portal-design.md)

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture Recap](#2-architecture-recap)
3. [Blueprint: `title`](#3-blueprint-title)
4. [Blueprint: `edition`](#4-blueprint-edition)
5. [Blueprint: `contract`](#5-blueprint-contract)
6. [Blueprint: `royalty-statement`](#6-blueprint-royalty-statement)
7. [Blueprint: `sales-data`](#7-blueprint-sales-data)
8. [Blueprint: `production-milestone`](#8-blueprint-production-milestone)
9. [Blueprint: `rights-territory`](#9-blueprint-rights-territory)
10. [Blueprint: `author-profile`](#10-blueprint-author-profile)
11. [Blueprint: `manuscript`](#11-blueprint-manuscript)
12. [Blueprint: `proof`](#12-blueprint-proof)
13. [Blueprint: `cover-review`](#13-blueprint-cover-review)
14. [Cross-Blueprint Relationships](#14-cross-blueprint-relationships)
15. [Sync Adapter Implementation Notes](#15-sync-adapter-implementation-notes)
16. [Appendix A: ONIX 3.0 Code List Reference](#appendix-a-onix-30-code-list-reference)
17. [Appendix B: BC API Endpoint Summary](#appendix-b-bc-api-endpoint-summary)
18. [Appendix C: Field Type Reference](#appendix-c-field-type-reference)

---

## 1. Introduction

This document serves two purposes:

1. **ERP Data Analysis** -- what data exists in knk's publishing vertical on Microsoft Dynamics 365 Business Central (BC), how it is structured, and what fields are relevant to the author portal.
2. **CMS Blueprint Definitions** -- the exact content models to configure in knkCMS (core CMS), including field names, types, relationships, and sync mapping from BC.

The portal never reads from the ERP directly. All ERP data flows through the sync layer into CMS blueprints. The CMS is the single data source for the portal frontend.

### Design Principles

- **ERP field names are not user-facing.** The CMS blueprint fields use clean, descriptive names. The sync adapter handles mapping.
- **ONIX 3.0 alignment.** Where BC stores publishing metadata, blueprint field names align with ONIX 3.0 terminology and code lists for interoperability.
- **Denormalization for read performance.** Some BC data that lives in separate tables is flattened into a single blueprint entry where it improves portal query performance.
- **Portal-native fields are clearly marked.** Fields that exist only in the CMS (not synced from ERP) are tagged `[portal-native]` and serve portal-specific workflows (e.g., manuscript collaboration, proof reviews).
- **ERP IDs are always preserved.** Every synced blueprint entry stores the BC system ID and record number for traceability and incremental sync.

### Conventions Used in This Document

| Notation | Meaning |
|---|---|
| `[ERP]` | Field value comes from BC via sync |
| `[portal-native]` | Field exists only in CMS, no ERP equivalent |
| `[bidirectional]` | Field syncs both ways (CMS <-> ERP) |
| `[computed]` | Field derived/calculated during sync, not stored directly in BC |
| **Bold field name** | Required field |
| Regular field name | Optional field |
| `relation:blueprint_name` | Relationship to another blueprint |

---

## 2. Architecture Recap

```
BC (knk ERP)                    knkCMS                        Portal Frontend
┌──────────────┐    sync     ┌──────────────────┐    API    ┌──────────────┐
│ knk Titles   │───────────>│ title blueprint   │────────>│ Title views  │
│ knk Editions │───────────>│ edition blueprint │────────>│ Edition list │
│ knk Contracts│───────────>│ contract blueprint│────────>│ Contract view│
│ knk Royalties│───────────>│ royalty-statement │────────>│ Royalty dash │
│ knk Sales    │───────────>│ sales-data        │────────>│ Sales charts │
│ knk Prod Ord │───────────>│ prod-milestone    │────────>│ Status pipe  │
│ knk Rights   │───────────>│ rights-territory  │────────>│ Rights map   │
│ BC Contacts  │<──────────>│ author-profile    │────────>│ Profile page │
│              │            │ manuscript        │────────>│ MS editor    │
│              │            │ proof             │────────>│ Proof review │
│              │            │ cover-review      │────────>│ Cover approve│
└──────────────┘            └──────────────────┘          └──────────────┘
```

**Sync endpoints:**
```
Base: https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{env}/api/knk/publishing/v1.0/
```

**Incremental sync filter:**
```
?$filter=lastModifiedDateTime gt {lastSyncTimestamp}
```

---

## 3. Blueprint: `title`

The title is the core entity -- a creative work (book) independent of its physical or digital format. In BC, this maps to the knk Title Card, which is the publishing-specific master record above the standard BC Item.

### 3.1 ERP Data Analysis (BC)

In knk's BC vertical, the Title entity holds:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `No.` (Title No.) | Code[20] | Primary key, publisher's internal title number |
| `Working Title` | Text[250] | Working/internal title |
| `Main Title` | Text[250] | Published title (ONIX: DistinctiveTitle) |
| `Subtitle` | Text[250] | Subtitle |
| `Original Title` | Text[250] | Original language title (for translations) |
| `Title Status` | Option | Planning / Active / Out of Print / Cancelled |
| `Publication Date` | Date | First publication date |
| `Announced Date` | Date | Date title was announced/catalogued |
| `Author No.` | Code[20] | Link to primary author contact |
| `Author Name` | Text[250] | Denormalized author display name |
| `Co-Author Nos.` | Code (related table) | Links to additional author contacts |
| `Imprint Code` | Code[20] | Publisher imprint |
| `Publishing Group` | Code[20] | Internal division/group |
| `Main Subject Code` | Code[20] | Primary BISAC or Thema subject code |
| `Additional Subjects` | Related table | Additional subject classifications |
| `BISAC Code` | Code[20] | BISAC subject heading |
| `Thema Code` | Code[20] | Thema subject code |
| `BIC Code` | Code[20] | BIC subject code (UK, legacy) |
| `Series Code` | Code[20] | Series identifier |
| `Series Name` | Text[100] | Series name |
| `Volume No.` | Text[20] | Volume/number within series |
| `Language Code` | Code[10] | ISO 639 language of text |
| `Original Language Code` | Code[10] | Original language (if translation) |
| `Edition Statement` | Text[100] | E.g., "2nd Revised Edition" |
| `Synopsis` | Blob/Text | Marketing description / synopsis |
| `Author Bio` | Blob/Text | Author biography for this title |
| `Keywords` | Text[500] | SEO/search keywords |
| `Age Range From` | Integer | Reading age range start |
| `Age Range To` | Integer | Reading age range end |
| `Audience Code` | Option | General/Children/Young Adult/Academic |
| `Country of Publication` | Code[10] | ISO 3166 country |
| `Internal Notes` | Blob/Text | Publisher-internal notes (NOT for portal) |
| `Created DateTime` | DateTime | Record creation timestamp |
| `Last Modified DateTime` | DateTime | Last modification timestamp |
| `SystemId` | GUID | BC system identifier |

Additionally, knk links titles to:
- **Edition/Item records** (one-to-many: hardcover, paperback, ebook, audiobook)
- **Contracts** (one-to-many: one title can have multiple contracts)
- **Production orders** (one-to-many: per edition)
- **Rights/territory records** (one-to-many)

### 3.2 CMS Blueprint Definition

**Blueprint name:** `title`  
**Description:** A creative work (book). The top-level entity that groups all editions, contracts, royalties, and production data. One title can have many editions (formats).  
**Sync direction:** ERP -> CMS (read-only from portal perspective)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **title_number** | `string` | Yes | `[ERP]` | Publisher's internal title number (BC `No.`) |
| **main_title** | `string` | Yes | `[ERP]` | Published title (ONIX: `DistinctiveTitle`) |
| subtitle | `string` | No | `[ERP]` | Subtitle |
| working_title | `string` | No | `[ERP]` | Internal working title (visible to editors only) |
| original_title | `string` | No | `[ERP]` | Original language title (for translations) |
| **status** | `enum` | Yes | `[ERP]` | `planning`, `active`, `out_of_print`, `cancelled` |
| publication_date | `date` | No | `[ERP]` | First publication date (earliest edition) |
| announced_date | `date` | No | `[ERP]` | Catalogue announcement date |
| synopsis | `richtext` | No | `[ERP]` | Marketing description / back-cover copy |
| author_bio_for_title | `richtext` | No | `[ERP]` | Author biography as used for this specific title |
| **primary_author** | `relation:author-profile` | Yes | `[ERP]` | Primary author (resolved via BC Author No.) |
| contributing_authors | `relation:author-profile[]` | No | `[ERP]` | Co-authors, illustrators, translators, etc. |
| contributor_roles | `json` | No | `[ERP]` | Array of `{author_id, role_code}` using ONIX Contributor Role codes (List 17) |
| imprint | `string` | No | `[ERP]` | Publisher imprint name |
| publishing_group | `string` | No | `[ERP]` | Internal division/group |
| bisac_subject | `string` | No | `[ERP]` | BISAC subject heading code (e.g., `FIC014000`) |
| thema_subject | `string` | No | `[ERP]` | Thema subject code (e.g., `FBA`) |
| additional_subjects | `string[]` | No | `[ERP]` | Additional subject classification codes |
| series_name | `string` | No | `[ERP]` | Series title |
| series_number | `string` | No | `[ERP]` | Volume/number within series |
| language | `string` | No | `[ERP]` | ISO 639-1/639-3 language code of the text |
| original_language | `string` | No | `[ERP]` | Original language code (if translation) |
| edition_statement | `string` | No | `[ERP]` | Free-text edition description |
| audience_code | `enum` | No | `[ERP]` | ONIX List 28: `general`, `children`, `young_adult`, `academic`, `professional` |
| reading_age_from | `number` | No | `[ERP]` | Interest age range start |
| reading_age_to | `number` | No | `[ERP]` | Interest age range end |
| keywords | `string[]` | No | `[ERP]` | Search/SEO keywords |
| country_of_publication | `string` | No | `[ERP]` | ISO 3166-1 alpha-2 country code |
| cover_image_url | `media` | No | `[portal-native]` | Cover image (may be uploaded via portal or synced separately) |
| editions | `relation:edition[]` | No | `[computed]` | Linked edition records |
| contracts | `relation:contract[]` | No | `[computed]` | Linked contract records |
| production_milestones | `relation:production-milestone[]` | No | `[computed]` | Linked production milestones |
| rights_territories | `relation:rights-territory[]` | No | `[computed]` | Linked rights/territory records |
| portal_notes | `richtext` | No | `[portal-native]` | Internal notes added by editors via the portal |
| bc_system_id | `string` | Yes | `[ERP]` | BC SystemId (GUID) for sync reconciliation |
| bc_last_modified | `datetime` | Yes | `[ERP]` | BC lastModifiedDateTime for incremental sync |

### 3.3 Sync Mapping Notes

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `No.` | `title_number` | Direct mapping |
| `Main Title` | `main_title` | Trim whitespace |
| `Subtitle` | `subtitle` | Trim whitespace |
| `Working Title` | `working_title` | Trim whitespace |
| `Original Title` | `original_title` | Trim whitespace |
| `Title Status` | `status` | Map BC Option integer to enum string: `0 -> planning`, `1 -> active`, `2 -> out_of_print`, `3 -> cancelled` |
| `Publication Date` | `publication_date` | BC Date -> ISO 8601 date string (`YYYY-MM-DD`). Use `0001-01-01` as null indicator in BC. |
| `Announced Date` | `announced_date` | Same as above |
| `Synopsis` (Blob) | `synopsis` | Fetch blob text via separate API call if blob field. Convert plain text to basic HTML if needed for richtext. |
| `Author No.` | `primary_author` | Resolve BC Contact No. to CMS `author-profile` entry. Create relationship link. |
| `Co-Author Nos.` | `contributing_authors` | Resolve each BC Contact No. to CMS `author-profile`. Build relationship array. |
| Contributor role from BC contributor table | `contributor_roles` | Map BC role codes to ONIX List 17 codes: `A01` (Author), `A12` (Illustrator), `B06` (Translator), etc. |
| `Imprint Code` | `imprint` | Resolve BC code to imprint name via Imprint setup table, or use code directly if name not available. |
| `BISAC Code` | `bisac_subject` | Direct mapping. Validate against BISAC code list. |
| `Thema Code` | `thema_subject` | Direct mapping. Validate against Thema code list. |
| `Language Code` | `language` | Map BC language code to ISO 639-1. BC may use 3-letter codes; normalize to 2-letter where possible. |
| `Country of Publication` | `country_of_publication` | Map to ISO 3166-1 alpha-2. |
| `SystemId` | `bc_system_id` | Direct mapping (GUID string) |
| `Last Modified DateTime` | `bc_last_modified` | Direct mapping (ISO 8601 datetime) |

**Key considerations:**
- The `editions` relationship is not synced directly from the title API call. It is built during edition sync when each edition references its parent title number.
- The `Internal Notes` field from BC is explicitly **not** synced to the portal for confidentiality reasons.
- If BC has no `Thema Code` but has a `BISAC Code`, the adapter should consider using a BISAC-to-Thema mapping table for international publishers.

---

## 4. Blueprint: `edition`

An edition represents a specific format/manifestation of a title: hardcover, trade paperback, mass-market paperback, ebook, audiobook, large print, etc. In BC, this maps to knk's extended Item table, where each item is linked to a Title.

### 4.1 ERP Data Analysis (BC)

In knk BC, editions are Item records with publishing-specific extensions:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `No.` (Item No.) | Code[20] | Item/SKU number |
| `Description` | Text[100] | Item description (typically format + title) |
| `Title No.` | Code[20] | Link to parent title |
| `ISBN-13` | Code[17] | ISBN-13 with hyphens (e.g., `978-3-16-148410-0`) |
| `ISBN-10` | Code[13] | Legacy ISBN-10 (if applicable) |
| `EAN` | Code[13] | EAN barcode (usually same as ISBN-13 without hyphens) |
| `Format Code` | Option/Code | Physical format (ONIX Product Form) |
| `Binding Code` | Code[20] | Binding type detail |
| `Publication Date` | Date | This edition's publication date |
| `On Sale Date` | Date | Retail on-sale date (may differ from pub date) |
| `Price` (multiple) | Decimal | List price(s) by currency/territory |
| `Currency Code` | Code[10] | ISO 4217 currency |
| `Page Count` | Integer | Number of pages |
| `Trim Width` | Decimal | Physical width (mm) |
| `Trim Height` | Decimal | Physical height (mm) |
| `Spine Width` | Decimal | Spine width (mm) |
| `Weight` | Decimal | Weight (grams) |
| `Duration` | Integer | Audiobook duration (minutes) |
| `Print Run` | Integer | Initial print run quantity |
| `Publisher RRP` | Decimal | Recommended retail price |
| `Status` | Option | Planned / Active / Out of Stock / Out of Print / Cancelled |
| `Inventory` | Decimal | Current stock quantity (standard BC) |
| `Printer/Manufacturer` | Code[20] | Production vendor |
| `Digital File Format` | Code[20] | EPUB, PDF, MOBI, MP3, etc. |
| `DRM Type` | Option | None / Adobe DRM / Social DRM / Watermark |
| `Sample Chapter URL` | Text[250] | Link to sample content |
| `Cover Image` | Blob/Media | Cover image |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 4.2 CMS Blueprint Definition

**Blueprint name:** `edition`  
**Description:** A specific format or manifestation of a title (hardcover, paperback, ebook, audiobook). Each edition has its own ISBN, pricing, physical specs, and publication date. One title has one or more editions.  
**Sync direction:** ERP -> CMS (read-only from portal perspective)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **item_number** | `string` | Yes | `[ERP]` | BC Item No. / SKU |
| **title** | `relation:title` | Yes | `[ERP]` | Parent title |
| **isbn_13** | `string` | Yes | `[ERP]` | ISBN-13 formatted with hyphens (ONIX: `ProductIdentifier` with `ProductIDType` = `15`) |
| isbn_10 | `string` | No | `[ERP]` | Legacy ISBN-10 |
| ean | `string` | No | `[ERP]` | EAN-13 barcode number |
| **format** | `enum` | Yes | `[ERP]` | ONIX Product Form (List 150): `hardback`, `trade_paperback`, `mass_market_paperback`, `ebook_epub`, `ebook_pdf`, `audiobook_downloadable`, `audiobook_cd`, `large_print`, `board_book`, `other` |
| format_detail | `string` | No | `[ERP]` | ONIX Product Form Detail (List 175), free text for additional format info |
| binding | `string` | No | `[ERP]` | Binding description (e.g., "Sewn", "Perfect bound") |
| **edition_status** | `enum` | Yes | `[ERP]` | `planned`, `active`, `out_of_stock`, `out_of_print`, `cancelled` |
| publication_date | `date` | No | `[ERP]` | This edition's publication date |
| on_sale_date | `date` | No | `[ERP]` | Retail availability date |
| list_price | `number` | No | `[ERP]` | Recommended retail price |
| price_currency | `string` | No | `[ERP]` | ISO 4217 currency code (e.g., `EUR`, `USD`, `GBP`) |
| additional_prices | `json` | No | `[ERP]` | Array of `{amount, currency, territory, price_type}` for multi-currency |
| page_count | `number` | No | `[ERP]` | Number of pages (physical formats) |
| trim_width_mm | `number` | No | `[ERP]` | Trim width in millimeters |
| trim_height_mm | `number` | No | `[ERP]` | Trim height in millimeters |
| spine_width_mm | `number` | No | `[ERP]` | Spine width in millimeters |
| weight_grams | `number` | No | `[ERP]` | Weight in grams |
| duration_minutes | `number` | No | `[ERP]` | Audiobook duration in minutes |
| digital_format | `string` | No | `[ERP]` | File format for digital editions: `EPUB2`, `EPUB3`, `PDF`, `MOBI`, `MP3`, `M4B`, `WAV` |
| drm_type | `enum` | No | `[ERP]` | `none`, `adobe_drm`, `social_drm`, `watermark`, `apple_fairplay` |
| print_run | `number` | No | `[ERP]` | Initial/current print run quantity (visible to author depends on publisher config) |
| stock_status | `enum` | No | `[ERP]` | `in_stock`, `low_stock`, `out_of_stock`, `print_on_demand`, `not_yet_available` |
| manufacturer | `string` | No | `[ERP]` | Printer/manufacturer name |
| cover_image | `media` | No | `[ERP]` | Cover image for this specific edition |
| sample_url | `string` | No | `[ERP]` | URL to sample chapter/content |
| description | `string` | No | `[ERP]` | Edition-specific description or marketing text |
| bc_system_id | `string` | Yes | `[ERP]` | BC SystemId (GUID) |
| bc_last_modified | `datetime` | Yes | `[ERP]` | BC lastModifiedDateTime |

### 4.3 Sync Mapping Notes

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `No.` (Item) | `item_number` | Direct mapping |
| `Title No.` | `title` | Resolve to CMS title entry via `title_number`. Create relationship link. If title not yet synced, queue for retry. |
| `ISBN-13` | `isbn_13` | Validate ISBN-13 check digit. Normalize to hyphenated format using ISBN range data (isbn-international.org). If invalid, log warning and store raw value. |
| `ISBN-10` | `isbn_10` | Validate check digit if present. |
| `Format Code` | `format` | Map BC format option/code to ONIX Product Form (List 150) enum values. Mapping table required per knk BC version. |
| `Status` | `edition_status` | Map BC Option integer to enum string. |
| `Price` + `Currency Code` | `list_price` + `price_currency` | Use primary price record. |
| Multiple price records | `additional_prices` | Aggregate all non-primary price records into JSON array. |
| `Page Count` | `page_count` | Direct mapping. Zero = null (don't store 0). |
| `Duration` | `duration_minutes` | Direct mapping. Only relevant for audiobook formats. |
| `Cover Image` (Media/Blob) | `cover_image` | Download blob, upload to CMS media library, store media reference. |
| `Inventory` | `stock_status` | Map quantity to status: `>50 -> in_stock`, `1-50 -> low_stock`, `0 -> out_of_stock`. Thresholds configurable per tenant. |

**Key considerations:**
- ISBN validation should use the Modulus-10 check digit algorithm. Invalid ISBNs should be flagged but still synced (some publishers use internal pseudo-ISBNs for pre-publication).
- Audiobook-specific fields (`duration_minutes`) are only populated when `format` is an audiobook type.
- The `print_run` field is sensitive -- publisher configuration should control whether authors can see this. The CMS stores it; guardian permissions control portal visibility.

---

## 5. Blueprint: `contract`

Contracts represent the legal agreement between a publisher and an author for a specific title or set of titles. In knk BC, this is managed through the Royalty Contract system, which tracks terms, royalty rates, advances, and territory rights.

### 5.1 ERP Data Analysis (BC)

knk BC's Royalty Contract structure:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `Contract No.` | Code[20] | Unique contract identifier |
| `Contract Type` | Option | Author Agreement / Co-Publishing / License / Translation / Subsidiary Rights |
| `Title No.` | Code[20] | Primary title (some contracts cover multiple titles) |
| `Title Nos.` (related table) | Code[20][] | All titles covered by this contract |
| `Author No.` | Code[20] | Contracting author/rights holder contact |
| `Agent No.` | Code[20] | Literary agent contact (if applicable) |
| `Contract Date` | Date | Date contract was signed |
| `Effective Date` | Date | Date contract takes effect |
| `Expiration Date` | Date | Contract end/expiration date |
| `Reversion Date` | Date | Rights reversion date |
| `Contract Status` | Option | Draft / Active / Expired / Terminated / Reverted |
| `Advance Amount` | Decimal | Total advance amount |
| `Advance Currency` | Code[10] | Advance currency |
| `Advance Earned Out` | Boolean | Whether advance has been earned out |
| `Advance Earned Out Date` | Date | Date advance was fully earned out |
| `Royalty Basis` | Option | Net Receipts / Retail Price / Net Sales / Wholesale |
| `Accounting Period` | Option | Monthly / Quarterly / Semi-Annual / Annual |
| `Statement Delay Days` | Integer | Days after period end before statement is produced |
| `Payment Terms` | Code[20] | Payment terms code |
| `Withholding Tax %` | Decimal | Tax withholding percentage |
| Royalty Rate Lines (sub-table): | | |
| `Rate Line No.` | Integer | Line number |
| `Rate Type` | Option | Standard / Escalation / Subsidiary / Digital |
| `Rate %` | Decimal | Royalty percentage |
| `Escalation Threshold` | Integer | Units sold before rate escalates |
| `Format Filter` | Code[20] | Applies to specific formats only |
| `Territory Filter` | Code[20] | Applies to specific territories only |
| `Channel Filter` | Code[20] | Applies to specific channels only |
| `Effective From Date` | Date | Rate effective start date |
| `Effective To Date` | Date | Rate effective end date |
| `Document Attachment` | Media/Blob | Contract document (PDF) |
| `Internal Notes` | Blob/Text | Internal publisher notes |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 5.2 CMS Blueprint Definition

**Blueprint name:** `contract`  
**Description:** A legal agreement between publisher and author governing a title or set of titles. Contains key terms, royalty rate structures, advance information, and contract lifecycle dates. Authors see a summary view; full contract documents are attached.  
**Sync direction:** ERP -> CMS (read-only from portal perspective)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **contract_number** | `string` | Yes | `[ERP]` | Publisher's contract reference number |
| **contract_type** | `enum` | Yes | `[ERP]` | `author_agreement`, `co_publishing`, `license`, `translation`, `subsidiary_rights` |
| **author** | `relation:author-profile` | Yes | `[ERP]` | Contracting author/rights holder |
| agent | `relation:author-profile` | No | `[ERP]` | Literary agent (if applicable, stored as author-profile with agent role) |
| **titles** | `relation:title[]` | Yes | `[ERP]` | Title(s) covered by this contract |
| contract_date | `date` | No | `[ERP]` | Date contract was signed |
| effective_date | `date` | No | `[ERP]` | Date contract takes effect |
| expiration_date | `date` | No | `[ERP]` | Contract end date |
| reversion_date | `date` | No | `[ERP]` | Rights reversion date |
| **contract_status** | `enum` | Yes | `[ERP]` | `draft`, `active`, `expired`, `terminated`, `reverted` |
| advance_amount | `number` | No | `[ERP]` | Total advance amount |
| advance_currency | `string` | No | `[ERP]` | ISO 4217 currency code |
| advance_earned_out | `boolean` | No | `[ERP]` | Whether the advance has been fully earned out |
| advance_earned_out_date | `date` | No | `[ERP]` | Date advance was earned out |
| royalty_basis | `enum` | No | `[ERP]` | `net_receipts`, `retail_price`, `net_sales`, `wholesale` |
| accounting_period | `enum` | No | `[ERP]` | `monthly`, `quarterly`, `semi_annual`, `annual` |
| statement_delay_days | `number` | No | `[ERP]` | Days after period close before statement is available |
| royalty_rates | `json` | No | `[ERP]` | Structured array of rate definitions (see schema below) |
| withholding_tax_percent | `number` | No | `[ERP]` | Tax withholding percentage |
| payment_terms_description | `string` | No | `[ERP]` | Human-readable payment terms |
| contract_document | `media` | No | `[ERP]` | Attached contract PDF |
| key_terms_summary | `richtext` | No | `[portal-native]` | Editor-written plain-language summary of key terms for the author |
| bc_system_id | `string` | Yes | `[ERP]` | BC SystemId (GUID) |
| bc_last_modified | `datetime` | Yes | `[ERP]` | BC lastModifiedDateTime |

**Royalty rates JSON schema:**
```json
{
  "rates": [
    {
      "rate_type": "standard | escalation | subsidiary | digital",
      "rate_percent": 10.0,
      "escalation_threshold_units": 5000,
      "format_filter": "hardback | trade_paperback | ebook_epub | null",
      "territory_filter": "US | UK | WORLD | null",
      "channel_filter": "retail | wholesale | online | null",
      "effective_from": "2026-01-01",
      "effective_to": null
    }
  ]
}
```

### 5.3 Sync Mapping Notes

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `Contract No.` | `contract_number` | Direct mapping |
| `Contract Type` | `contract_type` | Map BC Option to enum string |
| `Author No.` | `author` | Resolve to CMS `author-profile` via BC contact number |
| `Agent No.` | `agent` | Resolve to CMS `author-profile` with agent flag. If no agent, null. |
| `Title No.` / `Title Nos.` | `titles` | Resolve each title number to CMS `title` entry. Build relationship array. |
| `Contract Status` | `contract_status` | Map BC Option to enum string |
| `Advance Amount` + `Advance Currency` | `advance_amount` + `advance_currency` | Direct mapping. Sensitive data -- portal visibility controlled by guardian permissions. |
| Royalty Rate Lines (sub-table) | `royalty_rates` | Iterate all rate lines, build JSON array. Map BC option codes to readable strings. |
| `Document Attachment` | `contract_document` | Download blob, upload to CMS media library. |
| `Internal Notes` | **NOT SYNCED** | Publisher-internal notes are excluded from the portal. |
| `Payment Terms` | `payment_terms_description` | Resolve BC Payment Terms code to description text via the Payment Terms setup table. |

**Key considerations:**
- Contract documents (PDFs) are sensitive. The sync adapter uploads them to the CMS media library with restricted permissions. Guardian enforces that only the contracting author (and their agent, if applicable) can access the document.
- Royalty rates are structured as JSON rather than a separate blueprint because they are always read in the context of a contract and never queried independently.
- The `key_terms_summary` field is portal-native -- editors can write a plain-language summary of key contract terms for the author. This helps authors understand their contract without reading legal text.

---

## 6. Blueprint: `royalty-statement`

Royalty statements are periodic calculations of what an author is owed based on sales, returns, subsidiary income, and advance status. In knk BC, the royalty calculation engine produces statement records per author per accounting period.

### 6.1 ERP Data Analysis (BC)

knk BC's Royalty Statement structure:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `Statement No.` | Code[20] | Unique statement identifier |
| `Contract No.` | Code[20] | Related contract |
| `Author No.` | Code[20] | Author contact |
| `Period Start Date` | Date | Accounting period start |
| `Period End Date` | Date | Accounting period end |
| `Statement Date` | Date | Date statement was generated |
| `Statement Status` | Option | Draft / Final / Paid / Disputed / Credited |
| `Payment Date` | Date | Date payment was made |
| `Payment Reference` | Text[50] | Payment transaction reference |
| Opening Balance Lines: | | |
| `Previous Balance` | Decimal | Carry-forward balance from prior period |
| `Advance Outstanding` | Decimal | Remaining unearned advance |
| Earnings Lines (sub-table): | | |
| `Line No.` | Integer | Line number |
| `Title No.` | Code[20] | Title reference |
| `Edition No.` | Code[20] | Edition/Item reference |
| `Sales Channel` | Code[20] | Channel (retail, wholesale, online, export, etc.) |
| `Territory` | Code[20] | Territory/country |
| `Units Sold` | Integer | Units sold in period |
| `Units Returned` | Integer | Units returned in period |
| `Net Units` | Integer | Net units (sold minus returned) |
| `Gross Revenue` | Decimal | Gross revenue |
| `Net Revenue` | Decimal | Net revenue (after discounts/returns) |
| `Royalty Rate %` | Decimal | Applied royalty rate |
| `Royalty Amount` | Decimal | Calculated royalty for this line |
| `Rate Type` | Option | Standard / Escalation / Digital / Subsidiary |
| Summary Totals: | | |
| `Total Gross Royalty` | Decimal | Sum of all royalty amounts before deductions |
| `Less Advance Recoupment` | Decimal | Amount applied against unearned advance |
| `Less Reserve for Returns` | Decimal | Reserve held back for potential returns |
| `Less Withholding Tax` | Decimal | Tax withheld |
| `Less Agent Commission` | Decimal | Agent's commission deducted |
| `Less Previous Overpayment` | Decimal | Correction for prior overpayment |
| `Net Payable` | Decimal | Final amount payable to author |
| `Currency Code` | Code[10] | Statement currency |
| `Payment Status` | Option | Unpaid / Scheduled / Paid / Held |
| `Minimum Payment Threshold` | Decimal | Minimum amount for payment trigger |
| `Below Threshold` | Boolean | Whether net payable is below minimum |
| `Document Attachment` | Media/Blob | Statement PDF |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 6.2 CMS Blueprint Definition

**Blueprint name:** `royalty-statement`  
**Description:** A periodic royalty calculation for an author. Contains earnings details by title, edition, channel, and territory, along with deductions, advance recoupment, and net payable. This is the primary financial document authors access via the portal.  
**Sync direction:** ERP -> CMS (batch sync, typically after statement run; read-only from portal)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **statement_number** | `string` | Yes | `[ERP]` | Statement reference number |
| **contract** | `relation:contract` | Yes | `[ERP]` | Related contract |
| **author** | `relation:author-profile` | Yes | `[ERP]` | Author receiving this statement |
| **period_start** | `date` | Yes | `[ERP]` | Accounting period start |
| **period_end** | `date` | Yes | `[ERP]` | Accounting period end |
| **statement_date** | `date` | Yes | `[ERP]` | Date statement was generated/finalized |
| **statement_status** | `enum` | Yes | `[ERP]` | `draft`, `final`, `paid`, `disputed`, `credited` |
| **currency** | `string` | Yes | `[ERP]` | ISO 4217 currency code |
| previous_balance | `number` | No | `[ERP]` | Balance carried forward from prior period |
| advance_outstanding | `number` | No | `[ERP]` | Remaining unearned advance at period start |
| earnings_lines | `json` | No | `[ERP]` | Detailed earnings breakdown (see schema below) |
| **total_gross_royalty** | `number` | Yes | `[ERP]` | Sum of all royalty earnings before deductions |
| advance_recoupment | `number` | No | `[ERP]` | Amount applied against unearned advance |
| reserve_for_returns | `number` | No | `[ERP]` | Reserve held for potential returns |
| withholding_tax | `number` | No | `[ERP]` | Tax withheld |
| agent_commission | `number` | No | `[ERP]` | Agent commission deducted |
| previous_overpayment | `number` | No | `[ERP]` | Adjustment for prior overpayment |
| **net_payable** | `number` | Yes | `[ERP]` | Final amount payable to author |
| payment_status | `enum` | No | `[ERP]` | `unpaid`, `scheduled`, `paid`, `held` |
| payment_date | `date` | No | `[ERP]` | Date payment was issued |
| payment_reference | `string` | No | `[ERP]` | Bank transfer / payment reference |
| below_threshold | `boolean` | No | `[ERP]` | Whether payment is below minimum threshold |
| threshold_amount | `number` | No | `[ERP]` | Minimum payment threshold |
| statement_document | `media` | No | `[ERP]` | Statement PDF attachment |
| dispute_notes | `richtext` | No | `[portal-native]` | Author-submitted dispute/query notes |
| dispute_status | `enum` | No | `[portal-native]` | `none`, `submitted`, `under_review`, `resolved` |
| bc_system_id | `string` | Yes | `[ERP]` | BC SystemId (GUID) |
| bc_last_modified | `datetime` | Yes | `[ERP]` | BC lastModifiedDateTime |

**Earnings lines JSON schema:**
```json
{
  "lines": [
    {
      "title_number": "T-001234",
      "title_name": "The Great Novel",
      "edition_isbn": "978-3-16-148410-0",
      "edition_format": "hardback",
      "channel": "retail",
      "territory": "US",
      "units_sold": 1250,
      "units_returned": 45,
      "net_units": 1205,
      "gross_revenue": 29822.50,
      "net_revenue": 25149.75,
      "royalty_rate_percent": 10.0,
      "royalty_amount": 2514.98,
      "rate_type": "standard"
    }
  ]
}
```

### 6.3 Sync Mapping Notes

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `Statement No.` | `statement_number` | Direct mapping |
| `Contract No.` | `contract` | Resolve to CMS `contract` entry |
| `Author No.` | `author` | Resolve to CMS `author-profile` entry |
| `Period Start Date` / `Period End Date` | `period_start` / `period_end` | BC Date -> ISO 8601 |
| `Statement Status` | `statement_status` | Map BC Option to enum. Only sync statements with status `Final` or later; `Draft` statements should not appear in the portal. |
| Earnings Lines (sub-table) | `earnings_lines` | Fetch all sub-table lines via `$expand=royaltyStatementLines`. Build JSON array. Denormalize title name and edition ISBN for display without additional lookups. |
| All monetary fields | Corresponding fields | Direct decimal mapping. All values should use 2 decimal places. BC stores values in local currency by default. |
| `Document Attachment` | `statement_document` | Download and upload to CMS media library. |

**Key considerations:**
- Royalty statements contain the most sensitive financial data in the portal. Sync should only bring over `Final` or `Paid` statements -- never `Draft` statements, which may contain preliminary calculations.
- The `earnings_lines` JSON is denormalized intentionally: it includes `title_name` and `edition_isbn` so the portal can render the statement detail table without additional API calls to the title/edition blueprints.
- The `dispute_notes` and `dispute_status` fields are portal-native. They allow authors to flag a statement for review. The publisher's editorial/finance team sees the dispute in their admin view.
- Batch sync timing matters: statements should be synced after the publisher's royalty run is complete and statements are marked `Final`. This is typically configured per tenant with a cron schedule aligned to their accounting calendar.

---

## 7. Blueprint: `sales-data`

Sales data records provide granular unit and revenue figures by title, edition, channel, territory, and time period. While royalty statements contain aggregated sales, the `sales-data` blueprint provides the detailed breakdown that powers the portal's analytics dashboards and charts.

### 7.1 ERP Data Analysis (BC)

knk BC tracks sales through multiple mechanisms:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `Entry No.` | Integer | Auto-increment ledger entry number |
| `Title No.` | Code[20] | Title reference |
| `Edition No.` (Item No.) | Code[20] | Edition/SKU reference |
| `Posting Date` | Date | Transaction date |
| `Period` | Code[10] | Reporting period (e.g., `2026-Q1`) |
| `Sales Channel` | Code[20] | Distribution channel |
| `Channel Name` | Text[100] | Channel description |
| `Territory Code` | Code[20] | Territory / country / region |
| `Customer No.` | Code[20] | Customer/retailer (may be anonymized for portal) |
| `Customer Name` | Text[100] | Customer name (may be anonymized) |
| `Quantity` | Decimal | Units sold (positive) or returned (negative) |
| `Unit Price` | Decimal | Selling price per unit |
| `Discount %` | Decimal | Discount applied |
| `Net Amount` | Decimal | Net revenue |
| `Currency Code` | Code[10] | Transaction currency |
| `Amount (LCY)` | Decimal | Amount in local currency |
| `Entry Type` | Option | Sale / Return / Free Copy / Review Copy / Export |
| `Format Code` | Code[20] | Product format (for filtering) |
| `Subsidiary Rights Flag` | Boolean | Whether this is a sub-rights transaction |
| `Royalty Relevant` | Boolean | Whether this transaction is royalty-bearing |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 7.2 CMS Blueprint Definition

**Blueprint name:** `sales-data`  
**Description:** Aggregated sales figures by title, edition, channel, territory, and period. Used for sales analytics dashboards. Data is aggregated during sync to reduce volume -- individual transaction-level detail stays in BC.  
**Sync direction:** ERP -> CMS (batch sync; read-only from portal)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **record_id** | `string` | Yes | `[computed]` | Composite key: `{title}_{edition}_{channel}_{territory}_{period}` |
| **title** | `relation:title` | Yes | `[ERP]` | Title reference |
| **edition** | `relation:edition` | No | `[ERP]` | Edition reference (null for title-level aggregates) |
| **period** | `string` | Yes | `[ERP]` | Reporting period identifier (e.g., `2026-Q1`, `2026-03`) |
| **period_start** | `date` | Yes | `[ERP]` | Period start date |
| **period_end** | `date` | Yes | `[ERP]` | Period end date |
| channel | `string` | No | `[ERP]` | Sales channel name (e.g., "Retail", "Wholesale", "Online", "Export") |
| territory | `string` | No | `[ERP]` | Territory / country / region code |
| territory_name | `string` | No | `[ERP]` | Human-readable territory name |
| format | `string` | No | `[ERP]` | Product format (e.g., "Hardback", "Ebook") for format-level breakdowns |
| **units_sold** | `number` | Yes | `[ERP]` | Gross units sold in period |
| units_returned | `number` | No | `[ERP]` | Units returned in period |
| **net_units** | `number` | Yes | `[computed]` | Net units (sold minus returned) |
| gross_revenue | `number` | No | `[ERP]` | Gross revenue before discounts |
| **net_revenue** | `number` | Yes | `[ERP]` | Net revenue after discounts and returns |
| **currency** | `string` | Yes | `[ERP]` | ISO 4217 currency code |
| free_copies | `number` | No | `[ERP]` | Free/review copies distributed (non-revenue) |
| is_subsidiary_rights | `boolean` | No | `[ERP]` | Whether this represents subsidiary rights income |
| is_royalty_bearing | `boolean` | No | `[ERP]` | Whether included in royalty calculations |
| cumulative_units | `number` | No | `[computed]` | Running total of net units sold to date (for escalation tracking) |
| cumulative_revenue | `number` | No | `[computed]` | Running total of net revenue to date |
| bc_last_modified | `datetime` | Yes | `[ERP]` | Latest BC modification timestamp among aggregated entries |

### 7.3 Sync Mapping Notes

| BC Source | Blueprint Field | Transformation |
|---|---|---|
| Item Ledger Entries | Aggregated fields | The sync adapter queries BC ledger entries and **aggregates** them before writing to the CMS. Individual transactions are not synced. |
| `Title No.` | `title` | Resolve to CMS `title` entry |
| `Edition No.` | `edition` | Resolve to CMS `edition` entry |
| `Posting Date` range | `period`, `period_start`, `period_end` | Group entries by configurable period granularity (monthly or quarterly, per tenant setting). |
| `Sales Channel` + `Channel Name` | `channel` | Use channel name for display. Standardize channel codes across BC customers if possible. |
| `Territory Code` | `territory` + `territory_name` | Map BC territory code to ISO 3166 country/region. Use territory setup table for human-readable name. |
| SUM(`Quantity`) WHERE positive | `units_sold` | Aggregate positive quantities |
| SUM(`Quantity`) WHERE negative | `units_returned` | Aggregate negative quantities (store as positive number) |
| `units_sold - units_returned` | `net_units` | Computed during sync |
| SUM(`Net Amount`) | `net_revenue` | Aggregate net amounts |
| GROUP BY key | `record_id` | Build composite key from grouping dimensions |

**Key considerations:**
- Sales data volume can be large. The sync adapter must aggregate at the source (via BC API `$filter` and `$apply` for groupby where supported, or in-adapter aggregation). The CMS stores period-level aggregates, not individual transactions.
- Customer names are **not synced** to the portal. Authors should not see which specific retailers bought their books (contractual/competitive sensitivity). Data is aggregated by channel.
- The `cumulative_units` and `cumulative_revenue` fields are computed by the sync adapter by maintaining a running total across periods. These are important for royalty escalation tracking ("you've sold X copies, Y more until the next royalty tier").
- Period granularity (monthly vs. quarterly) should match the publisher's royalty accounting period to ensure consistency between `sales-data` and `royalty-statement`.

---

## 8. Blueprint: `production-milestone`

Production milestones track the stages a title goes through from manuscript acceptance to publication: editorial, copyediting, typesetting, proofreading, printing, distribution. This is the pipeline view that authors see in the portal.

### 8.1 ERP Data Analysis (BC)

knk BC tracks production through production orders and workflow stages:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `Production Order No.` | Code[20] | Production order identifier |
| `Title No.` | Code[20] | Title reference |
| `Edition No.` | Code[20] | Edition/Item reference |
| `Milestone Code` | Code[20] | Stage identifier |
| `Milestone Description` | Text[100] | Stage name |
| `Sequence No.` | Integer | Order in the production pipeline |
| `Status` | Option | Not Started / In Progress / Completed / Skipped / Blocked |
| `Planned Start Date` | Date | Planned start |
| `Planned End Date` | Date | Planned completion |
| `Actual Start Date` | Date | Actual start |
| `Actual End Date` | Date | Actual completion |
| `Assigned To` | Code[20] | Responsible person/department |
| `Assigned Name` | Text[100] | Name of responsible person |
| `Notes` | Text[250] | Milestone notes |
| `Predecessor Milestone` | Code[20] | Dependency on previous milestone |
| `% Complete` | Integer | Progress percentage (0-100) |
| `Requires Author Action` | Boolean | Whether this stage needs author input |
| `Author Action Type` | Option | None / Review / Approve / Submit / Provide Info |
| `Author Action Due Date` | Date | Deadline for author action |
| `Author Action Completed` | Boolean | Whether author has completed their action |
| `Document Attachment` | Media/Blob | Related document (e.g., style sheet, brief) |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 8.2 CMS Blueprint Definition

**Blueprint name:** `production-milestone`  
**Description:** A stage in the production pipeline for a specific title/edition. Milestones form an ordered sequence (pipeline) that represents progress from manuscript acceptance to publication. Authors see the pipeline with their action items highlighted.  
**Sync direction:** Configurable -- ERP -> CMS (Mode A) or CMS-native (Mode B) or bidirectional (Mode C)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **milestone_id** | `string` | Yes | `[ERP]` or `[portal-native]` | Unique milestone identifier (BC Production Order No. + Milestone Code, or CMS-generated) |
| **title** | `relation:title` | Yes | `[ERP]` or `[portal-native]` | Parent title |
| edition | `relation:edition` | No | `[ERP]` or `[portal-native]` | Specific edition (if milestone is edition-specific) |
| **milestone_name** | `string` | Yes | `[ERP]` or `[portal-native]` | Human-readable stage name (e.g., "Copyediting", "Typesetting", "Proof Review") |
| milestone_code | `string` | No | `[ERP]` | Machine-readable stage code |
| **sequence** | `number` | Yes | `[ERP]` or `[portal-native]` | Order in the pipeline (1, 2, 3...) |
| **status** | `enum` | Yes | `[ERP]` or `[portal-native]` | `not_started`, `in_progress`, `completed`, `skipped`, `blocked` |
| percent_complete | `number` | No | `[ERP]` or `[portal-native]` | Progress percentage (0-100) |
| planned_start | `date` | No | `[ERP]` or `[portal-native]` | Planned start date |
| planned_end | `date` | No | `[ERP]` or `[portal-native]` | Planned completion date |
| actual_start | `date` | No | `[ERP]` or `[portal-native]` | Actual start date |
| actual_end | `date` | No | `[ERP]` or `[portal-native]` | Actual completion date |
| assigned_to | `string` | No | `[ERP]` or `[portal-native]` | Responsible person/department name |
| notes | `richtext` | No | `[ERP]` or `[portal-native]` | Milestone notes/instructions |
| requires_author_action | `boolean` | No | `[ERP]` or `[portal-native]` | Whether this stage needs author input |
| author_action_type | `enum` | No | `[ERP]` or `[portal-native]` | `none`, `review`, `approve`, `submit`, `provide_info` |
| author_action_due_date | `date` | No | `[ERP]` or `[portal-native]` | Deadline for author action |
| author_action_completed | `boolean` | No | `[ERP]` or `[portal-native]` | Whether author has completed their action |
| author_action_completed_date | `date` | No | `[portal-native]` | Date author completed their action |
| predecessor | `relation:production-milestone` | No | `[ERP]` or `[portal-native]` | Dependency on previous milestone |
| attachments | `media[]` | No | `[ERP]` or `[portal-native]` | Related documents (briefs, style sheets, etc.) |
| linked_proof | `relation:proof` | No | `[portal-native]` | Link to a proof review, if this milestone involves proofing |
| linked_cover_review | `relation:cover-review` | No | `[portal-native]` | Link to a cover review, if this milestone involves cover approval |
| bc_system_id | `string` | No | `[ERP]` | BC SystemId (null if CMS-native) |
| bc_last_modified | `datetime` | No | `[ERP]` | BC lastModifiedDateTime (null if CMS-native) |

### 8.3 Sync Mapping Notes

**Mode A (ERP is source):**

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `Production Order No.` + `Milestone Code` | `milestone_id` | Concatenate: `{order_no}_{milestone_code}` |
| `Title No.` | `title` | Resolve to CMS `title` entry |
| `Edition No.` | `edition` | Resolve to CMS `edition` entry |
| `Milestone Description` | `milestone_name` | Direct mapping |
| `Sequence No.` | `sequence` | Direct mapping |
| `Status` | `status` | Map BC Option to enum |
| `Requires Author Action` | `requires_author_action` | Direct boolean |
| `Author Action Type` | `author_action_type` | Map BC Option to enum |
| `Document Attachment` | `attachments` | Download and upload to CMS media library |

**Mode B (CMS is source):** No sync needed. Milestones are created and managed entirely within the portal by editors/production staff.

**Mode C (Bidirectional):**
- ERP pushes milestone status updates to CMS.
- Portal pushes `author_action_completed` and `author_action_completed_date` back to ERP.
- Conflict resolution: configurable per tenant (`erp_wins`, `cms_wins`, `latest_wins`).
- The fields `linked_proof` and `linked_cover_review` are always portal-native and never pushed to ERP.

**Key considerations:**
- The standard milestone pipeline for trade publishing typically includes: Manuscript Accepted -> Editorial Review -> Copyediting -> Author Review of Copyedits -> Typesetting/Design -> First Proofs -> Author Proof Review -> Corrections -> Revised Proofs -> Final Proofs -> Print/Production -> Bound Copies -> Publication -> Distribution.
- The `requires_author_action` flag drives the portal's notification system. When a milestone enters `in_progress` and has `requires_author_action = true`, the author gets a notification with the action type and due date.
- Linking milestones to `proof` and `cover-review` entries enables seamless navigation from the production pipeline to the actual review workflow.

---

## 9. Blueprint: `rights-territory`

Rights and territories define where and in what formats a publisher has the right to sell a title, and what subsidiary rights exist. This is critical for international publishers and for authors who want to understand their rights position.

### 9.1 ERP Data Analysis (BC)

knk BC's rights management:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `Rights Entry No.` | Integer | Primary key |
| `Contract No.` | Code[20] | Related contract |
| `Title No.` | Code[20] | Related title |
| `Right Type` | Option | Publication / Translation / Audio / Film & TV / Dramatic / Serial / Electronic / Merchandising / First Serial / Second Serial / Book Club / Large Print / Reprint |
| `Right Status` | Option | Held / Licensed / Available / Reverted / Expired |
| `Territory Type` | Option | World / World English / Exclusive Territory / Non-Exclusive Territory |
| `Territory Code` | Code[20] | Country or region code (ISO 3166 or custom region) |
| `Territory Description` | Text[100] | Human-readable territory name |
| `Exclusive` | Boolean | Whether rights are exclusive in this territory |
| `Language Code` | Code[10] | Language of the right (for translation rights) |
| `Format Restriction` | Code[20] | If right is limited to specific format(s) |
| `Licensee` | Code[20] | Licensee contact (if right is licensed out) |
| `Licensee Name` | Text[100] | Licensee name |
| `License Start Date` | Date | License effective date |
| `License End Date` | Date | License expiration date |
| `License Fee` | Decimal | Fee/advance for licensed right |
| `License Currency` | Code[10] | Currency of license fee |
| `Royalty Share %` | Decimal | Author's share of sub-rights income |
| `Reversion Date` | Date | Date rights revert to author |
| `Reversion Conditions` | Text[250] | Conditions triggering reversion |
| `Notes` | Text[500] | Additional notes |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 9.2 CMS Blueprint Definition

**Blueprint name:** `rights-territory`  
**Description:** Defines rights held, licensed, or available for a title by territory and right type. Used for the portal's rights overview and territory map visualization (Phase 3). Each record represents one right-type/territory combination.  
**Sync direction:** ERP -> CMS (read-only from portal)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **rights_entry_id** | `string` | Yes | `[ERP]` | Unique identifier (BC Rights Entry No.) |
| **contract** | `relation:contract` | Yes | `[ERP]` | Parent contract |
| **title** | `relation:title` | Yes | `[ERP]` | Related title |
| **right_type** | `enum` | Yes | `[ERP]` | ONIX-aligned right type: `publication`, `translation`, `audio`, `film_tv`, `dramatic`, `serial_first`, `serial_second`, `electronic`, `merchandising`, `book_club`, `large_print`, `reprint` |
| **right_status** | `enum` | Yes | `[ERP]` | `held`, `licensed`, `available`, `reverted`, `expired` |
| **territory_type** | `enum` | Yes | `[ERP]` | `world`, `world_english`, `exclusive_territory`, `non_exclusive_territory` |
| territory_code | `string` | No | `[ERP]` | ISO 3166-1 alpha-2 country code or region code (e.g., `US`, `GB`, `EU`) |
| territory_name | `string` | No | `[ERP]` | Human-readable territory name |
| territory_countries | `string[]` | No | `[computed]` | Expanded list of ISO 3166-1 alpha-2 codes for composite territories (e.g., `EU` -> `[DE, FR, IT, ...]`) |
| is_exclusive | `boolean` | No | `[ERP]` | Whether rights are exclusive in territory |
| language | `string` | No | `[ERP]` | ISO 639-1 language code (relevant for translation rights) |
| format_restriction | `string` | No | `[ERP]` | Format(s) the right is limited to, if any |
| licensee_name | `string` | No | `[ERP]` | Name of licensee (if right is licensed) |
| license_start | `date` | No | `[ERP]` | License start date |
| license_end | `date` | No | `[ERP]` | License end date |
| license_fee | `number` | No | `[ERP]` | License fee/advance (visibility controlled by permissions) |
| license_currency | `string` | No | `[ERP]` | ISO 4217 currency code |
| author_royalty_share | `number` | No | `[ERP]` | Author's share percentage of sub-rights income |
| reversion_date | `date` | No | `[ERP]` | Date rights revert to author |
| reversion_conditions | `string` | No | `[ERP]` | Conditions triggering reversion (e.g., "out of print for 12 months") |
| notes | `string` | No | `[ERP]` | Additional notes |
| bc_system_id | `string` | Yes | `[ERP]` | BC SystemId (GUID) |
| bc_last_modified | `datetime` | Yes | `[ERP]` | BC lastModifiedDateTime |

### 9.3 Sync Mapping Notes

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `Rights Entry No.` | `rights_entry_id` | Convert integer to string |
| `Contract No.` | `contract` | Resolve to CMS `contract` entry |
| `Title No.` | `title` | Resolve to CMS `title` entry |
| `Right Type` | `right_type` | Map BC Option to ONIX-aligned enum. BC may use publisher-specific codes; maintain a mapping table. |
| `Right Status` | `right_status` | Map BC Option to enum |
| `Territory Type` | `territory_type` | Map BC Option to enum |
| `Territory Code` | `territory_code` | Normalize to ISO 3166-1 alpha-2 where possible. BC may use custom region codes (e.g., `NA` for North America). |
| `Territory Description` | `territory_name` | Direct mapping |
| Custom territory codes | `territory_countries` | Expand composite territory codes to individual country lists for map visualization. E.g., `EU` -> all EU member states, `ANZAC` -> `[AU, NZ]`. Expansion table maintained in sync adapter config. |
| `Licensee` | `licensee_name` | Resolve BC contact code to name. Do not expose contact number to portal. |
| `License Fee` | `license_fee` | Direct mapping. Sensitive -- guardian permissions control visibility (authors may or may not see sub-rights revenue details depending on contract). |
| `Royalty Share %` | `author_royalty_share` | Direct mapping (percentage as decimal, e.g., 50.0 for 50%) |

**Key considerations:**
- Territory data is central to the Phase 3 territory map visualization. The `territory_countries` computed field enables rendering a choropleth map showing where rights are held, licensed, or available.
- Right types should align with ONIX 3.0 Rights Type codes where possible, but knk BC may use a simpler set. The mapping table in the sync adapter should be extensible.
- Reversion tracking is important for authors. The portal should highlight upcoming reversion dates (e.g., "Rights revert in 90 days if out of print").
- License fees and sub-rights income details are sensitive. Some publishers share this with authors; others do not. Guardian permissions must control field-level visibility.

---

## 10. Blueprint: `author-profile`

The author profile represents the author as a person -- their contact details, biography, payment info, and preferences. This is the one blueprint with bidirectional sync: authors can update their profile in the portal, and changes can be written back to BC.

### 10.1 ERP Data Analysis (BC)

In BC, authors are stored as Contact records with publishing-specific extensions:

| BC Table/Field | BC Data Type | Description |
|---|---|---|
| `No.` (Contact No.) | Code[20] | BC Contact number |
| `Name` | Text[100] | Full display name |
| `First Name` | Text[30] | First name |
| `Middle Name` | Text[30] | Middle name(s) |
| `Surname` | Text[30] | Last name |
| `Name Prefix` | Text[30] | Prefix/title (Mr., Dr., Prof.) |
| `Name Suffix` | Text[30] | Suffix (Jr., III) |
| `Pen Name` | Text[100] | Pen name / pseudonym (knk extension) |
| `Additional Pen Names` | Related table | Additional pen names |
| `Author Type` | Option | Author / Illustrator / Translator / Editor / Agent |
| `E-Mail` | Text[80] | Primary email |
| `Phone No.` | Text[30] | Primary phone |
| `Mobile Phone No.` | Text[30] | Mobile phone |
| `Address` | Text[100] | Street address line 1 |
| `Address 2` | Text[50] | Street address line 2 |
| `City` | Text[30] | City |
| `County` | Text[30] | State/Province/County |
| `Post Code` | Text[20] | Postal/ZIP code |
| `Country/Region Code` | Code[10] | ISO 3166 country |
| `Nationality` | Code[10] | Nationality |
| `Date of Birth` | Date | Date of birth |
| `Tax ID / SSN` | Text[30] | Tax identification (encrypted in BC) |
| `VAT Registration No.` | Text[20] | VAT number (EU) |
| `Bank Account No.` | Text[30] | Bank account (encrypted) |
| `IBAN` | Text[34] | IBAN (encrypted) |
| `SWIFT/BIC` | Text[11] | SWIFT code |
| `Bank Name` | Text[100] | Bank name |
| `Payment Method` | Option | Bank Transfer / Check / PayPal / Wire |
| `Agent No.` | Code[20] | Author's literary agent contact |
| `Biography` | Blob/Text | Author biography |
| `Photo` | Media/Blob | Author photo |
| `Website` | Text[250] | Author website URL |
| `Social Media` (related table) | Text[] | Social media handles/URLs |
| `ISNI` | Text[16] | International Standard Name Identifier |
| `ORCID` | Text[19] | ORCID identifier (academic) |
| `Created DateTime` | DateTime | Record creation |
| `Last Modified DateTime` | DateTime | Last modification |
| `SystemId` | GUID | BC system identifier |

### 10.2 CMS Blueprint Definition

**Blueprint name:** `author-profile`  
**Description:** Author master data including contact information, biography, payment details, and identifiers. This blueprint supports bidirectional sync: authors update their profile in the portal, and changes flow back to BC (with conflict resolution). Payment/tax details are synced but access is strictly controlled.  
**Sync direction:** Bidirectional (ERP <-> CMS)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **contact_number** | `string` | Yes | `[ERP]` | BC Contact No. (never editable via portal) |
| **display_name** | `string` | Yes | `[bidirectional]` | Full display name |
| first_name | `string` | No | `[bidirectional]` | First name |
| middle_name | `string` | No | `[bidirectional]` | Middle name(s) |
| last_name | `string` | No | `[bidirectional]` | Last name |
| name_prefix | `string` | No | `[bidirectional]` | Name prefix (Dr., Prof., etc.) |
| name_suffix | `string` | No | `[bidirectional]` | Name suffix (Jr., III, etc.) |
| pen_name | `string` | No | `[bidirectional]` | Primary pen name / pseudonym |
| additional_pen_names | `string[]` | No | `[bidirectional]` | Additional pen names |
| **author_type** | `enum` | Yes | `[ERP]` | `author`, `illustrator`, `translator`, `editor`, `agent` |
| **email** | `string` | Yes | `[bidirectional]` | Primary email address |
| phone | `string` | No | `[bidirectional]` | Primary phone number |
| mobile_phone | `string` | No | `[bidirectional]` | Mobile phone number |
| address_line_1 | `string` | No | `[bidirectional]` | Street address line 1 |
| address_line_2 | `string` | No | `[bidirectional]` | Street address line 2 |
| city | `string` | No | `[bidirectional]` | City |
| state_province | `string` | No | `[bidirectional]` | State/Province/County |
| postal_code | `string` | No | `[bidirectional]` | Postal/ZIP code |
| country | `string` | No | `[bidirectional]` | ISO 3166-1 alpha-2 country code |
| nationality | `string` | No | `[bidirectional]` | Nationality (ISO 3166-1 alpha-2) |
| date_of_birth | `date` | No | `[ERP]` | Date of birth (ERP-only, not editable via portal for legal reasons) |
| tax_id | `string` | No | `[bidirectional]` | Tax identification number (encrypted at rest) |
| vat_number | `string` | No | `[bidirectional]` | VAT registration number (EU) |
| bank_account_number | `string` | No | `[bidirectional]` | Bank account number (encrypted at rest) |
| iban | `string` | No | `[bidirectional]` | IBAN (encrypted at rest) |
| swift_bic | `string` | No | `[bidirectional]` | SWIFT/BIC code |
| bank_name | `string` | No | `[bidirectional]` | Bank name |
| payment_method | `enum` | No | `[bidirectional]` | `bank_transfer`, `check`, `paypal`, `wire` |
| agent | `relation:author-profile` | No | `[ERP]` | Literary agent (self-referential relationship) |
| biography | `richtext` | No | `[bidirectional]` | Author biography |
| photo | `media` | No | `[bidirectional]` | Author photo |
| website | `string` | No | `[bidirectional]` | Author website URL |
| social_media | `json` | No | `[bidirectional]` | Social media profiles: `[{platform, url, handle}]` |
| isni | `string` | No | `[ERP]` | International Standard Name Identifier (16 digits) |
| orcid | `string` | No | `[ERP]` | ORCID identifier (academic authors) |
| notification_preferences | `json` | No | `[portal-native]` | Portal notification settings (email, in-app, per event type) |
| portal_language | `string` | No | `[portal-native]` | Preferred portal UI language (ISO 639-1) |
| portal_timezone | `string` | No | `[portal-native]` | Preferred timezone (IANA timezone, e.g., `Europe/Berlin`) |
| portal_last_login | `datetime` | No | `[portal-native]` | Last portal login timestamp |
| odon_user_id | `string` | No | `[portal-native]` | Linked odon user ID for authentication |
| bc_system_id | `string` | Yes | `[ERP]` | BC SystemId (GUID) |
| bc_last_modified | `datetime` | Yes | `[ERP]` | BC lastModifiedDateTime |
| cms_last_modified | `datetime` | No | `[portal-native]` | CMS-side modification timestamp (for conflict resolution) |

**Social media JSON schema:**
```json
{
  "profiles": [
    {
      "platform": "twitter | instagram | facebook | linkedin | tiktok | goodreads | bookbub | amazon_author | other",
      "url": "https://twitter.com/authorhandle",
      "handle": "@authorhandle"
    }
  ]
}
```

### 10.3 Sync Mapping Notes

**ERP -> CMS direction (initial load and ongoing updates):**

| BC Field | Blueprint Field | Transformation |
|---|---|---|
| `No.` | `contact_number` | Direct mapping |
| `Name` | `display_name` | Trim whitespace. BC may concatenate first/last. |
| `First Name` / `Surname` / etc. | `first_name` / `last_name` / etc. | Direct mapping |
| `Pen Name` | `pen_name` | Direct mapping |
| `Author Type` | `author_type` | Map BC Option to enum |
| `E-Mail` | `email` | Normalize to lowercase, validate format |
| Address fields | Address fields | Direct mapping. Normalize `Country/Region Code` to ISO 3166-1 alpha-2. |
| `Tax ID / SSN` | `tax_id` | **Must remain encrypted.** Sync adapter should encrypt before writing to CMS. CMS must store in encrypted field. |
| `IBAN` / `Bank Account No.` | `iban` / `bank_account_number` | Same encryption requirement. |
| `Biography` (Blob) | `biography` | Fetch blob, convert to richtext |
| `Photo` (Media) | `photo` | Download, upload to CMS media library |
| `Website` | `website` | Validate URL format |
| `ISNI` | `isni` | Validate 16-digit format |
| `ORCID` | `orcid` | Validate format (e.g., `0000-0002-1825-0097`) |

**CMS -> ERP direction (writeback when author updates profile):**

| Blueprint Field | BC Field | Transformation |
|---|---|---|
| `display_name` | `Name` | Direct mapping |
| `first_name` / `last_name` | `First Name` / `Surname` | Direct mapping |
| `email` | `E-Mail` | Direct mapping |
| Address fields | Address fields | Reverse normalization if needed |
| `biography` | `Biography` | Strip HTML to plain text for BC blob, or send as-is if BC supports HTML |
| `iban` / `bank_account_number` | `IBAN` / `Bank Account No.` | Decrypt from CMS, send via secure API call to BC |
| Portal-native fields | N/A | `notification_preferences`, `portal_language`, `portal_timezone`, `odon_user_id` are never written to BC |

**Conflict resolution:**
- Default strategy: `latest_wins` -- the most recently modified version (comparing `bc_last_modified` vs. `cms_last_modified`) takes precedence.
- For financial fields (`tax_id`, `iban`, `bank_account_number`): always use `manual` conflict resolution. Flag for publisher admin review.
- For `display_name`, `biography`, `photo`: `cms_wins` is a sensible default (author's own edits should take priority).

**Key considerations:**
- Payment and tax information requires special handling: encrypted storage, restricted access (only the author and publisher finance team), and audit logging on every access.
- The `odon_user_id` links the CMS author-profile to the odon authentication user. This mapping is established during author invitation/onboarding and is never synced to/from BC.
- The `author_type` field is ERP-only because it reflects the publisher's classification. An author cannot change their own type.
- Social media profiles are increasingly important for marketing coordination. The JSON structure is flexible to add new platforms without schema changes.

---

## 11. Blueprint: `manuscript`

Manuscripts are portal-native -- they do not exist in the ERP. This blueprint uses the core CMS's content versioning and TipTap editor capabilities for manuscript collaboration.

### 11.1 ERP Data Analysis

**None.** Manuscripts are created and managed entirely within the CMS/portal. There is no ERP equivalent.

### 11.2 CMS Blueprint Definition

**Blueprint name:** `manuscript`  
**Description:** A manuscript document associated with a title. Supports version control, co-author collaboration, editorial comments, and workflow transitions. Uses the core CMS's TipTap rich-text editor and content versioning system.  
**Sync direction:** Portal-native (no ERP sync)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **manuscript_id** | `string` | Yes | `[portal-native]` | Auto-generated unique identifier |
| **title** | `relation:title` | Yes | `[portal-native]` | Parent title |
| edition | `relation:edition` | No | `[portal-native]` | Specific edition (if format-specific, e.g., audiobook script) |
| **manuscript_name** | `string` | Yes | `[portal-native]` | Manuscript name/label (e.g., "Final Draft", "Audiobook Script") |
| **document_type** | `enum` | Yes | `[portal-native]` | `full_manuscript`, `partial_manuscript`, `synopsis`, `outline`, `chapter`, `audiobook_script`, `supplementary_material`, `index`, `bibliography`, `foreword`, `afterword` |
| **content** | `richtext` | No | `[portal-native]` | Manuscript text content (TipTap JSON/HTML). Null if file-only submission. |
| file_attachment | `media` | No | `[portal-native]` | Uploaded file (Word, PDF, etc.) for authors who prefer file upload over in-browser editing |
| file_format | `string` | No | `[portal-native]` | Original file format: `docx`, `pdf`, `rtf`, `odt`, `epub`, `txt` |
| word_count | `number` | No | `[portal-native]` | Word count (auto-calculated from content or uploaded file) |
| **version** | `number` | Yes | `[portal-native]` | Version number (auto-incremented by CMS versioning) |
| version_label | `string` | No | `[portal-native]` | Human-readable version label (e.g., "Author Revision 3", "Post-Copyedit") |
| version_notes | `string` | No | `[portal-native]` | Notes describing what changed in this version |
| **status** | `enum` | Yes | `[portal-native]` | `draft`, `submitted`, `in_review`, `revision_requested`, `accepted`, `in_copyedit`, `copyedit_complete`, `final` |
| submitted_by | `relation:author-profile` | No | `[portal-native]` | Author who submitted/uploaded this version |
| submitted_at | `datetime` | No | `[portal-native]` | Submission timestamp |
| reviewed_by | `string` | No | `[portal-native]` | Editor who reviewed (odon user reference) |
| reviewed_at | `datetime` | No | `[portal-native]` | Review timestamp |
| editorial_notes | `richtext` | No | `[portal-native]` | Editor's feedback/notes on this version |
| deadline | `date` | No | `[portal-native]` | Submission or revision deadline |
| is_current | `boolean` | No | `[portal-native]` | Whether this is the current/active version |
| language | `string` | No | `[portal-native]` | ISO 639-1 language code of the manuscript |
| confidentiality | `enum` | No | `[portal-native]` | `standard`, `restricted`, `embargo` -- controls who can access |
| tags | `string[]` | No | `[portal-native]` | Organizational tags |

### 11.3 Notes

- The core CMS's built-in content versioning handles the version history. Each save creates a new version in the CMS. The `version` field reflects the CMS version number.
- For large manuscripts, the `content` richtext field may be very large. The CMS should handle this via streaming/pagination of TipTap document nodes if needed.
- File attachments (Word docs, PDFs) are stored in the CMS media library. The portal may offer a conversion feature (DOCX -> TipTap JSON) for authors who upload files but want to use the in-browser editor.
- The `status` workflow is managed by the CMS workflow engine. Transitions trigger notifications (e.g., when status moves to `revision_requested`, the author gets a notification).
- Guardian permissions control access: only authors assigned to the title and editors assigned to the title can view/edit manuscripts.

---

## 12. Blueprint: `proof`

Proofs are portal-native review documents -- page proofs, galley proofs, or digital proofs that authors review and approve during production.

### 12.1 ERP Data Analysis

**None.** Proof review is a portal-native workflow. The ERP tracks that a "proofing" milestone exists, but the actual proof documents and review workflow live in the CMS.

### 12.2 CMS Blueprint Definition

**Blueprint name:** `proof`  
**Description:** A proof document (page proofs, galley proofs, digital proofs) submitted for author review and approval. Supports multiple correction rounds, annotation, and formal approval workflow.  
**Sync direction:** Portal-native (no ERP sync)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **proof_id** | `string` | Yes | `[portal-native]` | Auto-generated unique identifier |
| **title** | `relation:title` | Yes | `[portal-native]` | Parent title |
| **edition** | `relation:edition` | Yes | `[portal-native]` | Specific edition being proofed |
| milestone | `relation:production-milestone` | No | `[portal-native]` | Linked production milestone |
| **proof_type** | `enum` | Yes | `[portal-native]` | `first_page_proofs`, `revised_proofs`, `final_proofs`, `galley_proofs`, `digital_proofs`, `audio_proofs`, `index_proofs` |
| **round** | `number` | Yes | `[portal-native]` | Correction round number (1, 2, 3...) |
| **proof_document** | `media` | Yes | `[portal-native]` | Proof file (typically PDF) |
| supplementary_files | `media[]` | No | `[portal-native]` | Additional reference files (style guide, previous proofs, etc.) |
| **status** | `enum` | Yes | `[portal-native]` | `awaiting_review`, `in_review`, `corrections_submitted`, `corrections_applied`, `approved`, `approved_with_corrections`, `rejected` |
| assigned_to | `relation:author-profile` | No | `[portal-native]` | Author assigned to review |
| assigned_at | `datetime` | No | `[portal-native]` | Assignment timestamp |
| due_date | `date` | No | `[portal-native]` | Review deadline |
| review_started_at | `datetime` | No | `[portal-native]` | When the author started reviewing |
| review_completed_at | `datetime` | No | `[portal-native]` | When the author completed their review |
| corrections_document | `media` | No | `[portal-native]` | Author's corrections (annotated PDF or separate document) |
| corrections_notes | `richtext` | No | `[portal-native]` | Author's notes/comments on the proof |
| corrections_count | `number` | No | `[portal-native]` | Number of corrections noted |
| approved_by | `string` | No | `[portal-native]` | Who approved (author or editor odon user reference) |
| approved_at | `datetime` | No | `[portal-native]` | Approval timestamp |
| approval_signature | `string` | No | `[portal-native]` | Digital signature/confirmation text (e.g., "I approve these proofs for print") |
| editor_notes | `richtext` | No | `[portal-native]` | Editor's notes to the author about this proof |
| page_range | `string` | No | `[portal-native]` | Page range of the proof (e.g., "1-320" or "ix-xii" for front matter) |
| total_pages | `number` | No | `[portal-native]` | Total page count of the proof document |
| instructions | `richtext` | No | `[portal-native]` | Instructions for the author (what to look for, style guidelines, etc.) |

### 12.3 Notes

- Proof review is one of the most time-sensitive workflows in publishing. The portal must make it easy for authors to download proofs, annotate, and re-upload corrections.
- The `round` field tracks correction iterations. A typical flow: Round 1 (first proofs) -> Author corrections -> Round 2 (revised proofs) -> Final approval. Some publishers allow up to 3 rounds; more rounds incur extra costs.
- The `approval_signature` field serves as a lightweight digital sign-off. It is not a cryptographic signature but rather a confirmation that the author has reviewed and approved. The CMS audit trail provides the verification.
- Audio proofs (`audio_proofs` type) may be large media files (MP3/WAV). The CMS media library must handle large file uploads.
- When a proof transitions to `approved` or `approved_with_corrections`, the linked `production-milestone` (if any) should automatically progress. This is handled by the CMS workflow engine.

---

## 13. Blueprint: `cover-review`

Cover review is a portal-native approval workflow for book cover designs. Authors typically review and provide feedback on cover concepts, with a formal approval step before the cover goes to print.

### 13.1 ERP Data Analysis

**None.** Cover review is a portal-native workflow.

### 13.2 CMS Blueprint Definition

**Blueprint name:** `cover-review`  
**Description:** A cover design submitted for author review and approval. Supports multiple design options, feedback rounds, and formal approval. Linked to the production pipeline.  
**Sync direction:** Portal-native (no ERP sync)

| Field Name | Type | Required | Source | Description |
|---|---|---|---|---|
| **cover_review_id** | `string` | Yes | `[portal-native]` | Auto-generated unique identifier |
| **title** | `relation:title` | Yes | `[portal-native]` | Parent title |
| **edition** | `relation:edition` | Yes | `[portal-native]` | Specific edition (covers often differ between hardcover and paperback) |
| milestone | `relation:production-milestone` | No | `[portal-native]` | Linked production milestone |
| **review_name** | `string` | Yes | `[portal-native]` | Name/label (e.g., "Cover Concept Round 1", "Final Cover") |
| **round** | `number` | Yes | `[portal-native]` | Review round number |
| design_options | `json` | No | `[portal-native]` | Array of design options (see schema below) |
| **primary_cover_image** | `media` | Yes | `[portal-native]` | Primary cover design image (high-res) |
| additional_images | `media[]` | No | `[portal-native]` | Additional views (back cover, spine, full spread, mockup) |
| cover_brief | `richtext` | No | `[portal-native]` | Design brief / notes from the design team |
| **status** | `enum` | Yes | `[portal-native]` | `awaiting_review`, `in_review`, `feedback_submitted`, `revisions_in_progress`, `approved`, `approved_with_changes`, `rejected` |
| assigned_to | `relation:author-profile` | No | `[portal-native]` | Author assigned to review |
| due_date | `date` | No | `[portal-native]` | Review deadline |
| author_feedback | `richtext` | No | `[portal-native]` | Author's feedback on the design |
| selected_option | `number` | No | `[portal-native]` | Index of the selected design option (if multiple presented) |
| approved_by | `string` | No | `[portal-native]` | Who approved (odon user reference) |
| approved_at | `datetime` | No | `[portal-native]` | Approval timestamp |
| designer_name | `string` | No | `[portal-native]` | Cover designer name/credit |
| technical_specs | `json` | No | `[portal-native]` | Technical specifications (see below) |

**Design options JSON schema:**
```json
{
  "options": [
    {
      "option_number": 1,
      "label": "Option A - Illustrated",
      "image": "media_ref_123",
      "description": "Full illustration cover with handwritten title",
      "designer_notes": "Inspired by the forest scenes in chapter 3"
    }
  ]
}
```

**Technical specs JSON schema:**
```json
{
  "trim_width_mm": 135,
  "trim_height_mm": 210,
  "spine_width_mm": 22,
  "bleed_mm": 3,
  "color_profile": "CMYK",
  "resolution_dpi": 300,
  "file_format": "PDF/X-4",
  "finish": "matte lamination"
}
```

### 13.3 Notes

- Cover approval is often the most emotionally important review for authors. The portal should present covers at high quality with good image rendering.
- Multiple design options are common, especially in Round 1. The `design_options` JSON allows presenting 2-3 options for the author to choose from and provide feedback.
- When a cover is approved, the high-res image can be propagated to the `edition.cover_image` field to update the portal's edition display.
- Technical specs are included for reference and to help the design team maintain consistency across editions.

---

## 14. Cross-Blueprint Relationships

The following diagram shows all relationships between blueprints:

```
                                  ┌──────────────┐
                                  │ author-      │
                         ┌───────>│ profile      │<───────┐
                         │        └──────────────┘        │
                         │           │        │           │
                    author│      author│   agent│      author│
                         │           │        │           │
    ┌──────────────┐     │  ┌────────▼───┐    │     ┌─────▼──────────┐
    │ manuscript   │     │  │ contract   │    │     │ royalty-       │
    │              │     │  │            │    │     │ statement      │
    └──────┬───────┘     │  └──┬──────┬──┘    │     └────────────────┘
           │             │     │      │       │
     title │        title│title│  contract    │
           │             │     │      │       │
    ┌──────▼───────┐     │  ┌──▼──────▼──┐    │
    │              │<────┘  │ rights-    │    │
    │    title     │<───────│ territory  │    │
    │              │        └────────────┘    │
    └──┬───┬───┬───┘                          │
       │   │   │                              │
  editions │   │                              │
       │   │   │                              │
    ┌──▼───┘   └───────┐                      │
    │                   │                      │
    │  edition          │ production-          │
    │                   │ milestone            │
    ┌──────────────┐    ┌──────────────┐       │
    │ edition      │    │ production-  │       │
    │              │    │ milestone    │       │
    └──┬───────┬───┘    └──────────────┘       │
       │       │                               │
  proof│  cover-review                         │
       │       │                               │
    ┌──▼───┐ ┌─▼──────────┐                    │
    │proof │ │cover-review │                    │
    └──────┘ └─────────────┘                    │
```

### Relationship Summary Table

| From Blueprint | Relationship | To Blueprint | Cardinality | Description |
|---|---|---|---|---|
| `title` | `primary_author` | `author-profile` | Many-to-one | Every title has one primary author |
| `title` | `contributing_authors` | `author-profile` | Many-to-many | Titles can have multiple contributors |
| `edition` | `title` | `title` | Many-to-one | Each edition belongs to one title |
| `contract` | `author` | `author-profile` | Many-to-one | Each contract has one contracting author |
| `contract` | `agent` | `author-profile` | Many-to-one | Optional agent |
| `contract` | `titles` | `title` | Many-to-many | A contract can cover multiple titles |
| `royalty-statement` | `contract` | `contract` | Many-to-one | Each statement relates to one contract |
| `royalty-statement` | `author` | `author-profile` | Many-to-one | Each statement is for one author |
| `sales-data` | `title` | `title` | Many-to-one | Sales data references a title |
| `sales-data` | `edition` | `edition` | Many-to-one | Optionally references a specific edition |
| `production-milestone` | `title` | `title` | Many-to-one | Milestone belongs to a title |
| `production-milestone` | `edition` | `edition` | Many-to-one | Optionally for a specific edition |
| `production-milestone` | `predecessor` | `production-milestone` | Many-to-one | Milestone dependency chain |
| `production-milestone` | `linked_proof` | `proof` | One-to-one | Links to a proof review |
| `production-milestone` | `linked_cover_review` | `cover-review` | One-to-one | Links to a cover review |
| `rights-territory` | `contract` | `contract` | Many-to-one | Rights relate to a contract |
| `rights-territory` | `title` | `title` | Many-to-one | Rights relate to a title |
| `manuscript` | `title` | `title` | Many-to-one | Manuscript belongs to a title |
| `manuscript` | `submitted_by` | `author-profile` | Many-to-one | Who submitted the manuscript |
| `proof` | `title` | `title` | Many-to-one | Proof belongs to a title |
| `proof` | `edition` | `edition` | Many-to-one | Proof is for a specific edition |
| `proof` | `milestone` | `production-milestone` | Many-to-one | Links to production pipeline |
| `proof` | `assigned_to` | `author-profile` | Many-to-one | Author reviewing the proof |
| `cover-review` | `title` | `title` | Many-to-one | Cover review belongs to a title |
| `cover-review` | `edition` | `edition` | Many-to-one | Cover is for a specific edition |
| `cover-review` | `milestone` | `production-milestone` | Many-to-one | Links to production pipeline |
| `cover-review` | `assigned_to` | `author-profile` | Many-to-one | Author reviewing the cover |
| `author-profile` | `agent` | `author-profile` | Many-to-one | Self-referential: author's agent |

---

## 15. Sync Adapter Implementation Notes

### 15.1 Sync Order (Dependency Resolution)

Blueprints must be synced in dependency order to ensure relationship resolution succeeds:

```
1. author-profile     (no dependencies)
2. title               (depends on: author-profile)
3. edition             (depends on: title)
4. contract            (depends on: author-profile, title)
5. rights-territory    (depends on: contract, title)
6. royalty-statement   (depends on: contract, author-profile)
7. sales-data          (depends on: title, edition)
8. production-milestone (depends on: title, edition)
```

Portal-native blueprints (`manuscript`, `proof`, `cover-review`) are never synced from ERP.

### 15.2 Sync State Tracking

The sync adapter must maintain state to support incremental sync:

```go
type SyncState struct {
    TenantID     string    // Publisher tenant
    Blueprint    string    // Blueprint name (e.g., "title")
    LastSyncAt   time.Time // Timestamp of last successful sync
    LastBCModified time.Time // Latest BC lastModifiedDateTime seen
    EntriesSynced int      // Count of entries in last sync run
    Status       string    // "success", "partial", "failed"
    ErrorLog     string    // Error details if failed
}
```

### 15.3 Error Handling Strategy

| Error Type | Handling |
|---|---|
| BC API unavailable (5xx) | Exponential backoff, retry up to 3 times, then mark sync as failed and alert |
| BC rate limit (429) | Respect `Retry-After` header, pause and retry |
| Relationship resolution fails (referenced entity not found in CMS) | Queue the entry for retry after the dependency sync completes. Log warning. |
| Data validation fails (invalid ISBN, etc.) | Sync the entry with a `_validation_warnings` metadata field. Log warning. Do not block. |
| CMS write fails | Retry once. If still failing, log error with full payload for manual investigation. |
| Conflict (bidirectional sync) | Apply conflict resolution strategy from `EntitySyncConfig`. Log the conflict and both values. |

### 15.4 BC API Query Patterns

**Titles with expanded data:**
```
GET /api/knk/publishing/v1.0/titles
  ?$filter=lastModifiedDateTime gt 2026-04-13T00:00:00Z
  &$expand=editions,subjects
  &$top=1000
  &$orderby=lastModifiedDateTime asc
```

**Royalty statements with line details:**
```
GET /api/knk/publishing/v1.0/royaltyStatements
  ?$filter=periodEndDate ge 2026-01-01 and statementStatus eq 'Final'
  &$expand=royaltyStatementLines
  &$top=100
```

**Sales data aggregation (if BC supports $apply):**
```
GET /api/knk/publishing/v1.0/salesEntries
  ?$apply=filter(postingDate ge 2026-01-01 and postingDate le 2026-03-31)
    /groupby((titleNo,editionNo,salesChannel,territoryCode),
      aggregate(quantity with sum as totalQuantity,
                netAmount with sum as totalNetAmount))
```

If BC does not support `$apply`, the adapter fetches raw entries with `$filter` on date range and aggregates in-memory.

**Pagination:**
BC uses `@odata.nextLink` for pagination. The adapter must follow all pages:
```go
func (a *knkBCAdapter) fetchAll(ctx context.Context, url string) ([]json.RawMessage, error) {
    var all []json.RawMessage
    for url != "" {
        resp, nextLink, err := a.fetchPage(ctx, url)
        if err != nil {
            return nil, err
        }
        all = append(all, resp...)
        url = nextLink
    }
    return all, nil
}
```

### 15.5 Data Transformation Pipeline

Each synced entity goes through a consistent pipeline:

```
BC API Response
    │
    ▼
1. Deserialize (JSON -> Go struct)
    │
    ▼
2. Validate (required fields, data format checks)
    │
    ▼
3. Transform (map BC fields to blueprint fields, normalize codes, format dates)
    │
    ▼
4. Resolve relationships (look up CMS entries for related blueprints)
    │
    ▼
5. Compute derived fields (cumulative totals, status mappings, territory expansion)
    │
    ▼
6. Encrypt sensitive fields (tax IDs, bank details)
    │
    ▼
7. Upsert to CMS (create or update based on bc_system_id match)
    │
    ▼
8. Update sync state
```

### 15.6 Monitoring and Observability

The sync adapter should emit structured logs and metrics:

| Metric | Type | Description |
|---|---|---|
| `sync_entries_total` | Counter | Total entries synced (by blueprint, tenant) |
| `sync_errors_total` | Counter | Total errors (by blueprint, tenant, error_type) |
| `sync_duration_seconds` | Histogram | Duration of sync runs (by blueprint, tenant) |
| `sync_last_success_timestamp` | Gauge | Timestamp of last successful sync (by blueprint, tenant) |
| `bc_api_requests_total` | Counter | Total BC API requests (by endpoint, status_code) |
| `bc_api_latency_seconds` | Histogram | BC API response latency |

### 15.7 Tenant Configuration

Each publisher's sync is independently configured:

```go
type TenantSyncConfig struct {
    TenantID        string
    BCTenant        BCTenant              // BC connection details
    EntityConfigs   []EntitySyncConfig    // Per-entity sync settings
    SyncSchedule    map[string]string     // Blueprint -> cron expression
    FieldVisibility map[string][]string   // Blueprint -> hidden fields for portal
    AggregationPeriod string             // "monthly" or "quarterly" for sales data
    ISBNValidation  bool                  // Whether to validate ISBNs strictly
    TerritoryMapping map[string][]string  // Custom territory code -> country list
}
```

Example configuration for a German trade publisher:

```json
{
  "tenant_id": "pub_fischer",
  "sync_schedule": {
    "title": "0 */2 * * *",
    "edition": "0 */2 * * *",
    "contract": "0 6 * * *",
    "royalty-statement": "0 3 1 * *",
    "sales-data": "0 4 1 * *",
    "production-milestone": "*/15 * * * *",
    "rights-territory": "0 6 * * *",
    "author-profile": "0 */4 * * *"
  },
  "entity_configs": [
    {
      "entity": "production-milestone",
      "source_of_truth": "bidirectional",
      "sync_direction": "both",
      "conflict_rule": "latest_wins",
      "sync_frequency": "realtime"
    },
    {
      "entity": "author-profile",
      "source_of_truth": "bidirectional",
      "sync_direction": "both",
      "conflict_rule": "cms_wins",
      "sync_frequency": "hourly"
    }
  ],
  "aggregation_period": "quarterly",
  "isbn_validation": true
}
```

---

## Appendix A: ONIX 3.0 Code List Reference

The following ONIX 3.0 code lists are referenced in blueprint field definitions:

| Code List | Used In | Blueprint Field | Purpose |
|---|---|---|---|
| **List 5** (Product Identifier Type) | `edition` | `isbn_13`, `isbn_10`, `ean` | Identifier type: `15` = ISBN-13, `02` = ISBN-10, `03` = EAN |
| **List 17** (Contributor Role) | `title` | `contributor_roles` | `A01` = Author, `A12` = Illustrator, `B06` = Translator, `B01` = Editor, `E07` = Narrator |
| **List 28** (Audience Code) | `title` | `audience_code` | `01` = General, `02` = Children, `03` = Young Adult, `04` = Academic/Higher Education |
| **List 91** (Country Code) | Multiple | Territory fields | ISO 3166-1 codes |
| **List 150** (Product Form) | `edition` | `format` | `BB` = Hardback, `BC` = Paperback, `DG` = Ebook EPUB, `AJ` = Downloadable audio |
| **List 175** (Product Form Detail) | `edition` | `format_detail` | Detailed format codes |

Full ONIX 3.0 code lists are maintained by EDItEUR at https://ns.editeur.org/onix/en

### BISAC Subject Heading Reference

BISAC codes follow the pattern: `{CATEGORY}{SUBCATEGORY}{NUMBER}` (e.g., `FIC014000` = Fiction / Historical / General).

Full BISAC list maintained by BISG at https://bisg.org/page/BISACSubjectCodes

### Thema Subject Scheme Reference

Thema is the international equivalent of BISAC, designed for multilingual/global use.

Full Thema list maintained by EDItEUR at https://ns.editeur.org/thema/en

---

## Appendix B: BC API Endpoint Summary

All endpoints use the base URL:
```
https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{env}/api/knk/publishing/v1.0/
```

| Blueprint | BC Endpoint | HTTP Method | Key Query Parameters |
|---|---|---|---|
| `title` | `/titles` | GET | `$filter`, `$expand=editions,subjects`, `$top` |
| `edition` | `/editions` or `/titles({id})/editions` | GET | `$filter`, `$expand=prices`, `$top` |
| `contract` | `/royaltyContracts` | GET | `$filter`, `$expand=royaltyRateLines,titleLinks`, `$top` |
| `royalty-statement` | `/royaltyStatements` | GET | `$filter`, `$expand=royaltyStatementLines`, `$top` |
| `sales-data` | `/salesEntries` | GET | `$filter`, `$apply` (if supported), `$top` |
| `production-milestone` | `/productionOrders({id})/milestones` | GET | `$filter`, `$orderby=sequenceNo` |
| `rights-territory` | `/rightsEntries` | GET | `$filter`, `$expand=territories`, `$top` |
| `author-profile` | `/contacts` (with `$filter=contactType eq 'Author'`) | GET / PATCH | `$filter`, `$select` |

**Webhook subscription (for real-time sync):**
```
POST /api/knk/publishing/v1.0/subscriptions
{
  "notificationUrl": "https://portal-sync.example.com/webhooks/bc",
  "resource": "titles",
  "changeType": "updated",
  "expirationDateTime": "2026-05-14T00:00:00Z",
  "clientState": "tenant_pub_fischer_titles"
}
```

---

## Appendix C: Field Type Reference

The following types are available in the knkCMS blueprint system:

| Type | Description | Storage | Example |
|---|---|---|---|
| `string` | Plain text | VARCHAR | `"978-3-16-148410-0"` |
| `richtext` | HTML/TipTap content | TEXT (HTML/JSON) | `"<p>A novel about...</p>"` |
| `number` | Integer or decimal | NUMERIC | `29.95` |
| `boolean` | True/false | BOOLEAN | `true` |
| `date` | Date without time | DATE (ISO 8601) | `"2026-06-15"` |
| `datetime` | Date with time and timezone | TIMESTAMP | `"2026-06-15T14:30:00Z"` |
| `enum` | Predefined set of values | VARCHAR + validation | `"active"` |
| `string[]` | Array of strings | JSONB | `["FIC014000", "FIC049000"]` |
| `json` | Structured JSON data | JSONB | `{"rates": [...]}` |
| `media` | Reference to CMS media library | UUID (media ref) | `"media_abc123"` |
| `media[]` | Array of media references | JSONB (UUID array) | `["media_abc", "media_def"]` |
| `relation:{blueprint}` | Reference to another blueprint entry | UUID (entry ref) | `"entry_xyz789"` |
| `relation:{blueprint}[]` | Array of references | JSONB (UUID array) | `["entry_a", "entry_b"]` |
