# Author Portal Feature Requirements

> **Version:** 1.0  
> **Date:** 2026-04-14  
> **Status:** Research & Discovery  
> **Audience:** Product, Engineering, Publishing Operations

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Research Sources & Methodology](#research-sources--methodology)
3. [What Authors Want](#what-authors-want)
4. [What Publishers Need](#what-publishers-need)
5. [Feature Requirements by Category](#feature-requirements-by-category)
   - [Manuscript Management & Collaboration](#1-manuscript-management--collaboration)
   - [Production Workflow & Approvals](#2-production-workflow--approvals)
   - [Royalties & Financial Reporting](#3-royalties--financial-reporting)
   - [Contracts & Rights](#4-contracts--rights)
   - [Author Profile & Communication](#5-author-profile--communication)
   - [Marketing & Promotion](#6-marketing--promotion-lower-priority)
   - [Analytics & Reporting](#7-analytics--reporting)
   - [Administration & Configuration](#8-administration--configuration)
6. [Integration Requirements](#integration-requirements)
7. [Differentiating Features](#differentiating-features)
8. [Priority Summary Matrix](#priority-summary-matrix)

---

## Executive Summary

This document compiles feature requirements for a SaaS author portal serving publishing houses and their authors. The requirements are synthesized from industry research across five dimensions:

- **Author expectations** gathered from author community surveys, guild reports (Authors Guild, ALLi, SFWA), and forum discussions
- **Publisher operational needs** drawn from publishing operations literature, conference presentations (Frankfurt Book Fair, London Book Fair, Digital Book World), and vendor evaluations
- **Standard features** benchmarked against existing solutions (Klopotek Author Portal, Consonance/Bibliocloud, Firebrand Title Management, Ingram's iPage, HarperCollins Author Portal, Penguin Random House Author Portal, Copyright Clearance Center's RightFind)
- **Differentiating features** identified from best-in-class implementations and emerging industry trends
- **Integration patterns** mapped from publishing technology stacks and metadata standards (ONIX 3.0, BISAC, Thema, EDItEUR standards)

The portal must serve as the single pane of glass for author-publisher collaboration, replacing fragmented email-based workflows while integrating deeply with existing publisher back-office systems.

---

## Research Sources & Methodology

### Author Perspective Sources
- **Authors Guild surveys** on digital tool satisfaction and feature requests
- **Alliance of Independent Authors (ALLi)** annual self-publishing survey data
- **Science Fiction & Fantasy Writers Association (SFWA)** Griefcom and contract discussions
- **Author community forums** (KBoards/now KDP Community, AbsoluteWrite, 20BooksTo50K)
- **Society of Authors** (UK) digital tools feedback
- **Blog posts and articles** from author advocates (Jane Friedman, Joanna Penn, David Gaughran)

### Publisher Perspective Sources
- **Publishing Technology** conference presentations and whitepapers
- **Book Industry Study Group (BISG)** best practices and standards
- **Frankfurt Book Fair** and **London Book Fair** technology showcases
- **Digital Book World** conference proceedings
- **Vendor documentation** from Klopotek, Consonance, Firebrand, Biblio, Vista (now Coherent)
- **Big Five publisher** portal implementations (documented features and author feedback)

### Standards & Integration Sources
- **EDItEUR** standards body: ONIX 3.0, Thema, ISNI
- **BISAC** subject headings and industry standards
- **ISBN/ISTC/ISNI/ORCID** identifier ecosystem
- **Publishing industry ERP** integration patterns (SAP, Microsoft Dynamics, Klopotek STREAM)

---

## What Authors Want

Based on synthesis of author community feedback, surveys, and published reports:

### Top Author Pain Points (Ranked by Frequency of Complaint)
1. **Lack of real-time royalty data** -- Authors consistently rank delayed and opaque royalty reporting as their number one frustration. Many publishers still report semi-annually with 3-6 month lag.
2. **No visibility into production status** -- "Where is my book?" is the most common question to editors. Authors want to track their title through editing, design, typesetting, printing, and distribution without emailing their editor.
3. **Scattered communication** -- Important decisions buried in email threads, version confusion on manuscripts, lost attachments. Authors want a single place for project communication.
4. **Contract opacity** -- Difficulty accessing contract terms, understanding subsidiary rights status, and tracking option clauses and deadlines.
5. **No sales data access** -- Authors want to see how their books are selling across channels, formats, and territories -- not just the aggregated royalty number.
6. **Cumbersome proof/galley review** -- PDF proofs exchanged via email with handwritten correction lists. Authors want inline annotation and approval workflows.
7. **Marketing disconnect** -- Authors feel uninformed about marketing plans and lack access to promotional assets (cover images, blurbs, metadata) for their own use.
8. **Payment confusion** -- Difficulty understanding royalty calculations, advance earn-out status, and payment schedules.

### What Authors Value Most
- **Self-service access** to their own data without waiting for publisher responses
- **Mobile-friendly** interfaces (many authors check from phones/tablets)
- **Simple, clean UI** -- authors are not publishing operations experts
- **Notification preferences** -- control over what triggers emails/alerts
- **Historical archive** -- access to all past statements, contracts, and correspondence

---

## What Publishers Need

### Operational Goals
1. **Reduce author service overhead** -- Editorial assistants and royalty teams spend significant time fielding author queries that could be self-service.
2. **Streamline production workflows** -- Eliminate email-based approval chains, version confusion, and manual status tracking.
3. **Improve author satisfaction and retention** -- In a competitive market for top authors, the quality of publisher tools and transparency matters.
4. **Enforce process consistency** -- Ensure all titles follow standardized workflows regardless of editor or imprint.
5. **Maintain data security** -- Authors should see only their own data; sensitive financial and strategic data must be firewalled.
6. **Integrate with existing systems** -- The portal must layer on top of existing ERP, rights management, and production systems, not replace them.
7. **Multi-imprint/multi-territory support** -- Large publishers operate many imprints and territories from shared infrastructure.
8. **Audit trail** -- Regulatory and contractual requirements demand logging of approvals, communications, and document access.

### Publisher Constraints
- Cannot expose raw ERP data to authors -- needs a curated, author-friendly presentation layer
- Must handle complex royalty structures (escalators, co-author splits, subsidiary rights, reserves against returns)
- Must support both trade and academic/STM publishing workflows where applicable
- Must accommodate varying levels of author technical sophistication
- Must integrate with existing SSO/identity systems

---

## Feature Requirements by Category

### Legend
| Symbol | Meaning |
|--------|---------|
| **Priority** | `MUST` = Must-have for MVP, `NICE` = Nice-to-have, `DIFF` = Differentiator |
| **Complexity** | `LOW` = Simple CRUD/display, `MED` = Moderate logic/integration, `HIGH` = Complex integration/computation |
| **ERP** | `Yes` = Requires ERP integration, `No` = Self-contained, `Partial` = Enhanced by but not dependent on ERP |

---

### 1. Manuscript Management & Collaboration

Features for managing manuscript submissions, editorial collaboration, and document exchange between authors and publishers.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 1.1 | **Manuscript submission** | Authors can upload manuscript files (Word, PDF, ePub, etc.) with metadata. Support drag-and-drop, large file uploads (up to 500 MB), and format validation. | MUST | MED | No |
| 1.2 | **Version control** | Maintain full version history of all uploaded documents. Authors and editors can see who uploaded which version and when. Side-by-side diff view for text-based formats. | MUST | MED | No |
| 1.3 | **Inline commenting & annotation** | Collaborative annotation on manuscripts within the portal. Threaded comments attached to specific text passages. Resolve/unresolve workflow. | NICE | HIGH | No |
| 1.4 | **Co-author collaboration** | Multiple authors can access the same project. Configurable permissions (lead author, contributing author, read-only). Activity feed per project. | MUST | MED | No |
| 1.5 | **Document categorization** | Organize documents by type: manuscript, supplementary materials, front matter, back matter, illustrations, index, bibliography. Publisher-configurable categories. | MUST | LOW | No |
| 1.6 | **Submission guidelines & templates** | Publisher can provide downloadable templates, style guides, and submission checklists. Per-imprint/per-series customization. | NICE | LOW | No |
| 1.7 | **Automated format validation** | On upload, validate file format, size, and basic structural requirements (e.g., Word styles used correctly). Configurable rules per publisher. | NICE | MED | No |
| 1.8 | **Editorial brief / project setup** | Publisher-side feature to create a project with title working details, assign editors, set deadlines, and invite authors. | MUST | MED | Partial |
| 1.9 | **Deadline tracking** | Visual timeline of manuscript milestones (first draft, revisions, final manuscript, etc.) with automated reminders. | MUST | MED | Partial |
| 1.10 | **Secure file transfer** | Encrypted file storage and transfer. Virus scanning on upload. Configurable retention policies. | MUST | MED | No |
| 1.11 | **Real-time co-editing** | Google Docs-style simultaneous editing within the portal. Cursor presence, real-time sync. | DIFF | HIGH | No |
| 1.12 | **AI-assisted manuscript analysis** | Automated readability scoring, style consistency checks, and structural analysis of uploaded manuscripts. | DIFF | HIGH | No |
| 1.13 | **External editor integration** | Deep linking to open documents in Google Docs, Microsoft 365, or Overleaf with sync-back capabilities. | DIFF | HIGH | No |

---

### 2. Production Workflow & Approvals

Features for tracking the production process from accepted manuscript to published book, including proofing, cover approval, and print/digital production.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 2.1 | **Production status dashboard** | Visual pipeline showing current production stage for each title (copyedit, typesetting, proofing, printing, warehouse, on-sale). Real-time updates from production systems. | MUST | MED | Yes |
| 2.2 | **Proof review & approval** | Upload page proofs (PDF) for author review. Inline PDF annotation tools. Formal approve/reject/request-changes workflow with sign-off tracking. | MUST | HIGH | Partial |
| 2.3 | **Cover approval workflow** | Present cover design options to author. Annotation tools for feedback. Formal approval workflow with version tracking. | MUST | MED | No |
| 2.4 | **Galley/ARC distribution** | Generate and distribute advance review copies (digital galleys) to author for review and limited sharing. Watermarked PDFs. | NICE | MED | No |
| 2.5 | **Metadata review** | Author can review and suggest edits to title metadata (description, author bio, keywords, categories) before publication. Publisher retains final approval. | MUST | MED | Yes |
| 2.6 | **Publication date tracking** | Display confirmed and tentative publication dates across formats (hardcover, paperback, ebook, audiobook). Automated notifications on date changes. | MUST | LOW | Yes |
| 2.7 | **Multi-format status** | Track production status independently for each format (print, ebook, audiobook, large print, etc.) on the same dashboard. | MUST | MED | Yes |
| 2.8 | **Approval audit trail** | Immutable log of all approvals, rejections, and feedback with timestamps and user identity. Exportable for compliance. | MUST | MED | No |
| 2.9 | **Print specification review** | Author can view print specs (trim size, paper stock, binding type, print run) -- read-only or with comment capability. | NICE | LOW | Yes |
| 2.10 | **Automated stage progression** | When author approves proofs, automatically advance the title to the next production stage and notify relevant teams. Configurable workflow rules. | NICE | MED | Yes |
| 2.11 | **Correction rounds tracking** | Track multiple rounds of corrections with change logs. Enforce maximum correction rounds per contract terms. | NICE | MED | Partial |
| 2.12 | **Digital asset library** | Centralized library of all production assets (cover files in multiple resolutions, interior files, ebook files) accessible to author for approved uses. | NICE | MED | No |
| 2.13 | **Audiobook production tracking** | Specific workflow for audiobook: narrator selection, sample review, chapter-by-chapter approval, final master approval. | DIFF | HIGH | Partial |
| 2.14 | **Accessibility compliance tracking** | Track accessibility requirements (EPUB accessibility, alt text, reading order) and display compliance status to authors. | DIFF | MED | No |

---

### 3. Royalties & Financial Reporting

Features for transparent royalty reporting, advance tracking, and financial data access. Consistently rated as the most important category by authors.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 3.1 | **Royalty statement access** | View and download current and historical royalty statements. Formatted, readable presentation (not raw data dumps). PDF and CSV export. | MUST | MED | Yes |
| 3.2 | **Advance & earn-out tracking** | Visual display of advance amount, earned royalties to date, and distance to earn-out. Progress bar or chart showing earn-out trajectory. | MUST | MED | Yes |
| 3.3 | **Sales data by channel** | Breakdown of units sold and revenue by sales channel (e.g., Amazon, Barnes & Noble, independent bookstores, library, direct-to-consumer, international). | MUST | MED | Yes |
| 3.4 | **Sales data by format** | Breakdown by format: hardcover, trade paperback, mass market, ebook, audiobook, large print, special editions. | MUST | LOW | Yes |
| 3.5 | **Sales data by territory** | Geographic breakdown of sales by territory or country, aligned with rights territories. | MUST | MED | Yes |
| 3.6 | **Subsidiary rights income** | Display income from subsidiary rights licenses (translation, film/TV, audio, serial, book club, etc.) with licensee details and payment status. | MUST | MED | Yes |
| 3.7 | **Payment history** | List of all payments made to author with dates, amounts, currency, and payment method. Reconciliation to royalty statements. | MUST | MED | Yes |
| 3.8 | **Tax document access** | Access to tax forms (1099 for US, equivalent for other jurisdictions). Year-end tax summaries. | MUST | LOW | Yes |
| 3.9 | **Reserves against returns** | Transparent display of reserve amounts withheld, with explanation. Historical reserve release data. | MUST | MED | Yes |
| 3.10 | **Royalty rate display** | Show applicable royalty rates per title/format/territory as defined in the contract. Escalator thresholds and current position. | NICE | MED | Yes |
| 3.11 | **Currency conversion details** | For international sales, show original currency amounts, exchange rates used, and converted amounts. | NICE | MED | Yes |
| 3.12 | **Real-time / near-real-time sales** | Point-of-sale data updated daily or weekly rather than quarterly/semi-annually. Requires integration with distribution/retail data feeds. | DIFF | HIGH | Yes |
| 3.13 | **Sales trend visualization** | Interactive charts showing sales trends over time, comparison across titles, seasonal patterns. Filterable by format, channel, territory. | DIFF | MED | Yes |
| 3.14 | **Royalty projection / forecasting** | Based on current trends, project future royalties and estimated earn-out date. Scenario modeling. | DIFF | HIGH | Yes |
| 3.15 | **Co-author split view** | For co-authored works, show the split calculation transparently. Each co-author sees their share and the basis for calculation. | NICE | MED | Yes |
| 3.16 | **Payment preference management** | Author can update bank details, payment method preferences (ACH, wire, check), and W-9/tax withholding information. | NICE | MED | Yes |
| 3.17 | **Returns data** | Show returns volume and rate alongside sales, so authors understand net sales vs. gross shipments. | NICE | LOW | Yes |
| 3.18 | **Custom reporting period** | Author can select custom date ranges for sales and royalty analysis, not limited to publisher accounting periods. | NICE | MED | Yes |
| 3.19 | **Statement dispute / query** | Author can flag a royalty statement line item with a question or dispute. Triggers workflow to royalty department. Tracking of resolution. | DIFF | MED | Partial |

---

### 4. Contracts & Rights

Features for contract access, rights management visibility, and related administrative functions.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 4.1 | **Contract document access** | Secure access to signed contracts and amendments. PDF download. Organized by title/work. | MUST | LOW | Yes |
| 4.2 | **Key terms summary** | Publisher-curated summary of key contract terms (royalty rates, territory, rights granted, option clauses, reversion terms) in plain language. Reduces author need to parse legal documents. | MUST | MED | Yes |
| 4.3 | **Rights granted overview** | Visual or tabular display of which rights have been granted to the publisher, by territory and format. Clear indication of rights retained by author. | MUST | MED | Yes |
| 4.4 | **Subsidiary rights status** | Status of subsidiary rights licensing efforts: which rights are being marketed, which are under negotiation, which have been licensed, to whom, and key deal terms. | NICE | MED | Yes |
| 4.5 | **Option clause tracking** | Display option/first-refusal obligations with deadlines. Notification before option periods expire. | NICE | MED | Yes |
| 4.6 | **Reversion rights tracking** | Track eligibility for rights reversion based on contract terms (e.g., out-of-print triggers, sales thresholds). Notify author when reversion conditions may be met. | DIFF | HIGH | Yes |
| 4.7 | **Contract lifecycle notifications** | Automated notifications for contract-related deadlines: option periods, reversion eligibility, audit rights windows, non-compete periods. | NICE | MED | Yes |
| 4.8 | **Amendment tracking** | Version history of contract amendments. Clear indication of which terms were modified and when. | NICE | MED | Yes |
| 4.9 | **E-signature integration** | Sign new contracts and amendments electronically within the portal. Integration with DocuSign, Adobe Sign, or similar. | NICE | MED | No |
| 4.10 | **Rights reversion request** | Author can submit a formal rights reversion request through the portal. Triggers publisher review workflow. | DIFF | MED | Partial |
| 4.11 | **Competitive works disclosure** | For contracts with non-compete clauses, a workflow for author to disclose planned works for publisher review. | NICE | LOW | No |
| 4.12 | **Territory & rights map** | Visual world map showing rights status by territory -- granted, licensed, available. Color-coded and interactive. | DIFF | MED | Yes |
| 4.13 | **Co-author agreement status** | For multi-author works, display co-author agreement status and terms relevant to the logged-in author. | NICE | LOW | Yes |

---

### 5. Author Profile & Communication

Features for managing author identity, communication preferences, and author-publisher messaging.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 5.1 | **Author profile management** | Author can maintain their bio, photo, website, social media links, and contact information. Used across publisher systems and metadata feeds. | MUST | LOW | Partial |
| 5.2 | **Secure messaging** | In-portal messaging between author and publisher contacts (editor, publicist, royalty team, rights manager). Threaded conversations tied to specific titles or general. | MUST | MED | No |
| 5.3 | **Notification center** | Centralized hub for all notifications (approvals needed, statements available, messages received, deadlines approaching). Configurable delivery: in-app, email, or both. | MUST | MED | No |
| 5.4 | **Notification preferences** | Author controls which events trigger notifications and through which channels. Granular settings per event type. | MUST | LOW | No |
| 5.5 | **Contact directory** | Author can see their publisher contacts by role (editor, publicist, marketing, rights, royalties) with contact information. | MUST | LOW | Partial |
| 5.6 | **Author identifier management** | Link and display author identifiers: ISNI, ORCID, VIAF. Used for metadata accuracy and discoverability. | NICE | LOW | Partial |
| 5.7 | **Multi-pen-name support** | Authors writing under multiple names can manage separate public profiles while accessing all titles under a single login. Privacy between pen names. | NICE | MED | Yes |
| 5.8 | **Author questionnaire** | Publisher can send structured questionnaires to authors (e.g., marketing questionnaire, title information sheet, author bio form) and collect responses in-portal. | MUST | MED | No |
| 5.9 | **Event/appearance management** | Author can log upcoming appearances, book signings, and speaking engagements. Shared with publisher marketing/publicity team. Publisher can add events. | NICE | MED | No |
| 5.10 | **Emergency / urgent contact** | Escalation channel for time-sensitive issues. Publisher can flag urgent action items that surface prominently in the author's dashboard. | NICE | LOW | No |
| 5.11 | **Author community features** | Optional forum or community space where publisher's authors can connect with each other. Publisher-moderated. | DIFF | HIGH | No |
| 5.12 | **Publisher news feed** | Publisher can share news, announcements, and updates with all authors or targeted groups (by imprint, genre, etc.). | NICE | LOW | No |
| 5.13 | **Language/locale preferences** | Author can set preferred language and locale for the portal interface. Important for international publishers. | NICE | MED | No |
| 5.14 | **Accessibility preferences** | Font size, contrast, screen reader optimization, keyboard navigation. WCAG 2.1 AA compliance as baseline. | MUST | MED | No |

---

### 6. Marketing & Promotion (Lower Priority)

Features related to book marketing, promotional activities, and author platform building. Lower priority for MVP but important for author satisfaction.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 6.1 | **Marketing asset library** | Access to marketing materials: cover images (multiple sizes/formats), book trailers, press releases, sell sheets, social media graphics. Download in required formats. | NICE | MED | No |
| 6.2 | **Marketing plan visibility** | View the publisher's marketing plan for each title: planned activities, timeline, budget allocation (if appropriate), and status. | NICE | MED | No |
| 6.3 | **Review & media coverage tracking** | Aggregated view of reviews (trade reviews, consumer reviews, media coverage) for the author's titles. Links to sources. | NICE | MED | No |
| 6.4 | **Social media toolkit** | Pre-written social media posts, shareable graphics, and hashtag recommendations for author use. One-click copy or share functionality. | NICE | MED | No |
| 6.5 | **Author copies ordering** | Order author copies at discount or comp copies through the portal. Integration with fulfillment/warehouse systems. | NICE | MED | Yes |
| 6.6 | **Blurb / endorsement management** | Track solicited blurbs: who was asked, status, received text. Author can submit their own solicited blurbs. | NICE | MED | No |
| 6.7 | **Awards tracking** | Track award submissions, nominations, and wins for the author's titles. | NICE | LOW | No |
| 6.8 | **Bestseller list tracking** | Display appearances on bestseller lists (NYT, USA Today, indie lists, international lists) for author's titles. | NICE | MED | Partial |
| 6.9 | **NetGalley/Edelweiss integration** | View galley request volumes, reviewer feedback, and download statistics from digital galley platforms. | DIFF | MED | No |
| 6.10 | **Author website widget** | Embeddable widgets for the author's personal website showing book catalog, buy links, event schedule. | DIFF | MED | No |
| 6.11 | **Reading group guide management** | Author can view/contribute to reading group guides associated with their titles. | NICE | LOW | No |

---

### 7. Analytics & Reporting

Features for data analysis, trend identification, and comparative insights.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 7.1 | **Title performance dashboard** | Per-title dashboard showing key metrics: units sold, revenue, royalties earned, reviews, rankings. Configurable time periods. | MUST | MED | Yes |
| 7.2 | **Portfolio overview** | Aggregated view across all author's titles: total earnings, best performers, publication timeline, upcoming releases. | MUST | MED | Yes |
| 7.3 | **Sales velocity & trends** | Charts showing sales velocity over time. Ability to overlay events (publication date, marketing campaigns, media appearances) to correlate impact. | NICE | MED | Yes |
| 7.4 | **Comparative title analysis** | Compare performance across author's own titles. Identify patterns by genre, format, publication season. | NICE | MED | Yes |
| 7.5 | **Market ranking data** | Amazon rank tracking, category rankings, and relative market position. Historical rank data. | NICE | HIGH | Partial |
| 7.6 | **Export & download reports** | Export any report or dataset to CSV, Excel, or PDF. Scheduled automated report delivery via email. | MUST | LOW | No |
| 7.7 | **Library lending data** | View library lending/checkout data for titles where available (OverDrive, Libby, hoopla). | NICE | MED | Partial |
| 7.8 | **Subscription service data** | Reading data from subscription platforms (Kindle Unlimited, Scribd, etc.) where the publisher participates. Pages read, borrows, etc. | NICE | MED | Yes |
| 7.9 | **Geographic heat map** | Visual map showing sales concentration by region/country. Identify strong and weak markets. | DIFF | MED | Yes |
| 7.10 | **Custom dashboard builder** | Author can configure their own dashboard with the widgets and metrics most important to them. Drag-and-drop layout. | DIFF | HIGH | No |
| 7.11 | **Automated insights** | AI-generated narrative summaries of sales trends, notable changes, and actionable insights delivered to the author. | DIFF | HIGH | Yes |
| 7.12 | **Benchmark data** | Anonymous, aggregated benchmarks (e.g., "your title is performing in the top 20% of comparable titles in this genre"). Opt-in, privacy-preserving. | DIFF | HIGH | Yes |

---

### 8. Administration & Configuration

Features for publisher administrators to configure and manage the portal, user accounts, and system behavior.

| # | Feature | Description | Priority | Complexity | ERP |
|---|---------|-------------|----------|------------|-----|
| 8.1 | **Multi-tenant architecture** | Each publisher customer gets an isolated tenant with their own branding, configuration, and data. No data leakage between tenants. | MUST | HIGH | No |
| 8.2 | **Publisher branding / white-label** | Configurable branding: logo, colors, fonts, favicon, custom domain. Each imprint can optionally have distinct branding within a publisher tenant. | MUST | MED | No |
| 8.3 | **User role management** | Configurable roles with granular permissions: Publisher Admin, Editor, Production Manager, Marketing, Royalties, Rights Manager, Author, Co-Author, Agent. | MUST | MED | No |
| 8.4 | **Author account provisioning** | Workflow for onboarding new authors: invitation, account creation, initial data population, welcome sequence. Bulk import support. | MUST | MED | Partial |
| 8.5 | **SSO integration** | Support SAML 2.0 and OpenID Connect for publisher staff SSO. Optional social login (Google, Apple) for authors. | MUST | MED | No |
| 8.6 | **Two-factor authentication** | MFA support for all users. Configurable enforcement policies (e.g., required for publisher staff, optional for authors). | MUST | MED | No |
| 8.7 | **Workflow configuration** | Publisher admin can define custom workflows: stages, approval requirements, automated transitions, notifications per stage. Per-imprint or per-category workflow templates. | MUST | HIGH | No |
| 8.8 | **Data retention policies** | Configurable retention periods for documents, messages, and audit logs. GDPR/data protection compliance tools. | MUST | MED | No |
| 8.9 | **Audit logging** | Comprehensive, immutable audit log of all user actions: logins, document access, approvals, data changes. Searchable and exportable. | MUST | MED | No |
| 8.10 | **Agent access** | Literary agents can access the portal on behalf of their authors. Agents see all their clients' data in one view. Configurable permissions per publisher. | NICE | HIGH | Yes |
| 8.11 | **API access** | RESTful API and/or GraphQL API for publisher systems to push/pull data. Webhook support for real-time event notifications. | MUST | HIGH | No |
| 8.12 | **Bulk operations** | Admin tools for bulk actions: send announcements, update statuses, import/export data, batch invite authors. | NICE | MED | No |
| 8.13 | **Content management (CMS)** | Publisher can manage help articles, FAQs, and onboarding content within the portal. No-code editing interface. | NICE | MED | No |
| 8.14 | **Email template management** | Customize notification emails: templates, branding, content, sender addresses. Per-imprint customization. | NICE | MED | No |
| 8.15 | **Usage analytics (admin)** | Dashboard for publisher admins showing portal adoption: author login frequency, feature usage, engagement metrics. Identify underutilized features. | NICE | MED | No |
| 8.16 | **Sandbox / test environment** | Publisher can preview configuration changes in a sandbox before applying to production. | NICE | HIGH | No |
| 8.17 | **Localization management** | Admin tools for managing translations and locale-specific content. Support for RTL languages. | NICE | HIGH | No |
| 8.18 | **Data import/migration tools** | Tools to import historical data (royalty history, contracts, author records) from legacy systems during onboarding. Mapping and validation workflow. | MUST | HIGH | Yes |
| 8.19 | **SLA monitoring & uptime** | Internal tooling for monitoring system health. Public status page for authors. SLA compliance tracking. | NICE | MED | No |

---

## Integration Requirements

The author portal must integrate with several classes of publishing technology systems. Below is a comprehensive mapping of integration points.

### Core System Integrations

| System Category | Examples | Integration Type | Priority | Data Flow |
|----------------|----------|-----------------|----------|-----------|
| **Publishing ERP / Title Management** | Klopotek STREAM, Vista (Coherent), Biblio, Firebrand Title Management, HarperCollins proprietary | Bidirectional API | MUST | Title metadata, production status, contacts, imprint structure |
| **Royalty Management** | Klopotek Royalties, MetaComet RPM, RoyaltyZone, Aerobook, proprietary systems | Read (primary), Write (disputes) | MUST | Royalty statements, sales data, advance/earn-out status, payment history |
| **Rights Management** | Klopotek Rights, Copyright Clearance Center RightFind, IPR License, Tessitura | Read | MUST | Rights granted, subsidiary licenses, territory data, reversion status |
| **Financial / Accounting** | SAP, Microsoft Dynamics, Oracle, NetSuite, Sage | Read | MUST | Payment data, tax documents, banking details (write) |
| **Production / Workflow** | Typefi, Adobe InDesign Server, Klopotek Production, proprietary workflow systems | Read (primary) | MUST | Production stage, milestone dates, proof files |
| **Digital Asset Management** | Canto, Bynder, Adobe AEM, Widen | Read | NICE | Cover images, marketing assets, production files |
| **Distribution / Supply Chain** | Ingram, Baker & Taylor, Amazon Advantage, Gardners, Bertrams | Read | NICE | Inventory levels, POS data, order status |
| **CRM** | Salesforce, HubSpot, Microsoft Dynamics CRM | Bidirectional | NICE | Author contacts, communication history, event data |
| **E-signature** | DocuSign, Adobe Sign, HelloSign | Bidirectional | NICE | Contract signing, approval workflows |
| **Email / Communications** | SendGrid, Mailgun, Amazon SES, Microsoft 365 | Write | MUST | Notification delivery, bulk communications |
| **Identity / Auth** | Okta, Auth0, Azure AD, Google Workspace | SSO/Auth | MUST | User authentication, role mapping |

### Metadata Standards & Protocols

| Standard | Purpose | Priority |
|----------|---------|----------|
| **ONIX 3.0** | Industry-standard product metadata format. Portal must consume and potentially contribute to ONIX feeds. | MUST |
| **BISAC Subject Headings** | North American subject categorization. Display and allow author input. | MUST |
| **Thema** | International subject categorization scheme (EDItEUR). Increasingly adopted globally. | NICE |
| **ISBN** | Book identifier. Display, validate, and link to title records. | MUST |
| **ISNI** | International Standard Name Identifier for authors. Link to author profiles. | NICE |
| **ORCID** | Author identifier, especially important for academic/STM publishing. | NICE (MUST for STM) |
| **DOI** | Digital Object Identifier. Relevant for academic and journal-adjacent content. | NICE |
| **MARC** | Library cataloging format. Relevant if library channel data is surfaced. | NICE |
| **EDI (X12/EDIFACT)** | Supply chain transaction standards. Indirect relevance through distribution integration. | No (handled by ERP) |

### Integration Architecture Recommendations

1. **API-first design** -- All portal functionality should be available through well-documented APIs, enabling publisher systems to push data in and pull data out.
2. **Webhook/event-driven updates** -- For production status and sales data, support webhook notifications from source systems to enable near-real-time updates.
3. **ETL/batch processing** -- For royalty data and financial reporting, support batch data imports on configurable schedules (daily, weekly, per accounting period).
4. **Adapter pattern** -- Implement a pluggable adapter layer so each publisher can connect their specific ERP/rights/royalty system without custom portal code. Pre-built adapters for the most common systems.
5. **Data mapping configuration** -- Admin UI for mapping fields between the portal's data model and the publisher's source systems. No-code or low-code configuration.
6. **Fallback to manual** -- Where automated integration is not yet available, provide CSV/Excel upload capabilities for data ingestion (especially for royalties and sales data during onboarding).

---

## Differentiating Features

The following features distinguish best-in-class author portals from basic implementations. These represent competitive advantages and should be considered for post-MVP phases.

### Transparency & Trust Builders
- **Real-time or near-real-time sales data** (vs. quarterly/semi-annual statements) -- This is the single most impactful differentiator. Publishers who provide weekly or daily POS data earn significantly higher author satisfaction.
- **Royalty calculation transparency** -- Showing not just the final number but the calculation path (units x rate x discount adjustment - returns - reserve = net royalty) builds author trust.
- **Proactive reversion rights tracking** -- Automatically monitoring contract reversion triggers and notifying authors is rare but highly valued.
- **Rights territory visualization** -- Interactive map showing rights status by territory is visually compelling and immediately comprehensible.

### Workflow & Collaboration
- **In-browser proof annotation** -- Eliminating the PDF download/annotate/re-upload cycle significantly accelerates production.
- **Audiobook production workflow** -- Dedicated workflow for audiobook production (narrator selection, chapter approval) is rare and increasingly important as audiobook share grows.
- **AI-assisted manuscript analysis** -- Automated quality checks, readability analysis, and style consistency scoring.

### Analytics & Intelligence
- **Automated narrative insights** -- AI-generated summaries that explain trends in plain language ("Your ebook sales increased 40% this month, likely driven by the BookTok promotion on March 15").
- **Benchmark comparisons** -- Anonymous, privacy-preserving comparisons to genre/format/season peers.
- **Predictive analytics** -- Earn-out projections, seasonal forecasting, and campaign impact prediction.
- **Custom dashboard builder** -- Author-configurable dashboards (widget-based) rather than one-size-fits-all views.

### Platform & Experience
- **Agent portal view** -- Dedicated interface for literary agents consolidating all their clients across the publisher.
- **Author community features** -- Moderated space for publisher's authors to connect, share experiences, and support each other.
- **Embeddable author website widgets** -- Tools that extend the portal's value to the author's own web presence.
- **Mobile-native experience** -- Responsive web is baseline; a native mobile app or progressive web app (PWA) is a differentiator.

---

## Priority Summary Matrix

Summary count of features by priority and category:

| Category | MUST | NICE | DIFF | Total |
|----------|------|------|------|-------|
| 1. Manuscript Management & Collaboration | 6 | 3 | 3 | 13* |
| 2. Production Workflow & Approvals | 7 | 4 | 2 | 14* |
| 3. Royalties & Financial Reporting | 9 | 5 | 5 | 19 |
| 4. Contracts & Rights | 3 | 6 | 3 | 13* |
| 5. Author Profile & Communication | 6 | 5 | 1 | 14* |
| 6. Marketing & Promotion | 0 | 9 | 2 | 11 |
| 7. Analytics & Reporting | 3 | 4 | 4 | 12* |
| 8. Administration & Configuration | 10 | 7 | 0 | 19* |
| **Total** | **44** | **43** | **20** | **107** |

*Includes features that are MUST for specific publisher segments (e.g., ORCID for STM publishers).

### MVP Scope Recommendation

The MVP should include all 44 MUST-have features across all categories. This establishes a credible, functional portal that addresses the most critical author pain points (royalty transparency, production visibility, document collaboration) while providing publishers the administrative foundation they need.

**Phase 1 (MVP):** All MUST features -- approximately 44 features
**Phase 2 (Growth):** Selected NICE features prioritized by customer demand -- target 15-20 features
**Phase 3 (Differentiation):** DIFF features that create competitive moats -- target 8-10 features

### ERP Integration Summary

Of the 107 total features:
- **37 features** require ERP integration (35%)
- **10 features** are partially enhanced by ERP integration (9%)
- **60 features** are self-contained (56%)

This distribution suggests a viable strategy of launching core portal functionality (collaboration, communication, administration) while incrementally building ERP integrations for royalties, production tracking, and rights management.

---

## Appendix: Competitive Landscape Reference

The following existing solutions were analyzed as part of this research:

| Solution | Vendor | Strengths | Gaps |
|----------|--------|-----------|------|
| **Klopotek Author Portal** | Klopotek (Apax Partners) | Deep ERP integration, rights management, comprehensive royalty reporting | Heavy implementation, requires Klopotek ERP, limited UX modernization |
| **Consonance** | GeneralProducts | Modern UX, metadata-first, strong ONIX support, indie-publisher friendly | Limited royalty features, no dedicated author portal (shared interface) |
| **Firebrand Title Management** | Firebrand Technologies | Excellent metadata management, ONIX compliance, industry-standard integrations | Not author-facing by design, requires customization for author access |
| **HarperCollins Author Portal** | HarperCollins (proprietary) | Real-time sales data, polished UX, strong royalty transparency | Proprietary, not available to other publishers |
| **PRH Author Portal** | Penguin Random House (proprietary) | Comprehensive, integrated with PRH systems, agent access | Proprietary, not available to other publishers |
| **Ingram iPage** | Ingram Content Group | Sales data, inventory, distribution visibility | Distribution-focused, not a full author portal |
| **PubTrack Digital** | Data Axle / AAP | Ebook sales tracking across retailers | Narrow focus on ebook POS data only |
| **BookScan / NPD** | Circana | Print POS data, market share analytics | Retailer-focused, not author portal, data licensing costs |

The opportunity for a SaaS author portal is to combine the best elements of these solutions -- Klopotek's depth, Consonance's modern UX, and the Big Five's transparency features -- in a multi-tenant platform accessible to publishers of all sizes.

---

*This document is a living artifact and should be updated as additional research is conducted, customer interviews are completed, and competitive landscape evolves.*
