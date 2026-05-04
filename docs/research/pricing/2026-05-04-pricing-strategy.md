---
title: "Pricing Strategy"
---

# Author Portal — Pricing Strategy

> **Date:** 2026-05-04
> **Status:** Draft — anchor numbers for first-customer conversation, not final pricing
> **Authors:** jiwanovski87
> **Related:** [Author Portal Design Specification](../../superpowers/specs/2026-04-14-author-portal-design.md), [Services Status & Roadmap](../../superpowers/specs/2026-04-30-services-status-and-roadmap-design.md), [Competitor Market Overview](../competitors/market-overview.md)

---

## 1. Purpose

Two customers have expressed interest in the author portal. This document captures the pricing thesis to guide the first conversations with them, anchored against verified competitor pricing as of May 2026.

The exact numbers and the precise transaction unit are deliberately left open to be co-defined with the first design-partner customer. This document fixes the *philosophy*, the *structure*, and the *order of magnitude*.

---

## 2. Pricing Philosophy

> **Transaction-based, value-aligned.** If the system creates value for the customer and is heavily used, knk participates in that value. If the solution underperforms or is barely used, the customer pays less.

Source: company owner's preference. Rationale: skin-in-the-game pricing is honest about product quality, scales with customer success, and ages well as features mature.

**Why this is also a competitive differentiator:**

Every direct comparable in publishing software (Klopotek Cloud, Consonance, Fonto) charges per user. A transaction-based model is genuinely novel in the segment and is a defensible talking point in any sales conversation.

**What is explicitly ruled out:**

- **Per-author licensing** — publishing houses can have thousands of authors on their backlist, most of whom barely log in. Per-author pricing creates an awkward conversation, incentivizes the publisher to gate access, and doesn't reflect actual value delivered.

---

## 3. Proposed Structure

A hybrid: **low monthly base + transactional component**.

| Component | Suggested range | Purpose |
|---|---|---|
| **One-time setup fee** | €5,000 – €15,000 | Covers ERP-adapter configuration, branding setup, user onboarding, kickoff workshops. Lower end for design-partner customers |
| **Monthly base** | €750 – €1,000 | Infrastructure, support, account management. Predictable revenue floor for knk; low enough not to scare small publishers |
| **Transactional component** | €5 – €15 per active title per month *(unit TBD with first customer)* | Value alignment. "Active title" candidate definition: at least one author event in the period (login, manuscript edit, message, proof review). Backlist sitting idle costs nothing |

The transaction unit ("active title") is the recommended starting point. Alternatives discussed and the reasons for the recommendation are in §6.

**Hedge for revenue volatility:** pure transaction pricing creates revenue unpredictability for knk and bill-shock risk for the customer. The base fee provides a floor; transactional caps and tiers can be negotiated in larger contracts.

---

## 4. Suggested Magnitude

What the bill looks like at different customer sizes (using a €1,000 base + €10/active title illustrative midpoint):

| Publisher profile | Active titles / month | Base | Transactional | **Monthly total** | Annual |
|---|---|---|---|---|---|
| Small publisher | ~20 | €1,000 | €200 | **€1,200** | €14K |
| Mid-size (≈ both interested customers) | ~100 | €1,000 | €1,000 | **€2,000** | €24K |
| Large trade publisher | ~500 | €1,000 | €5,000 | **€6,000** | €72K |
| Very large (Springer-size) | 2,000+ | negotiated | tier-discounted | **€10K – €20K+** | €120K – €240K+ |

---

## 5. Competitor Pricing Anchors (May 2026)

Cost for a comparable mid-size publisher (~50 active editorial users):

| Solution | What you get | Annual cost | Source |
|---|---|---|---|
| **knk author portal (this proposal)** | **Complete portal: auth, dashboard, manuscripts, production workflow, ERP sync, branded** | **€24K** | This document |
| Fonto Editor only (named user) | Just the manuscript editor | €15.8K | [Fonto pricing 2024](https://www.fontoxml.com/) |
| Fonto Editor + Review (named user) | Editor + review/commenting | €28.3K | Same |
| Fonto full suite (named user) | Editor + Review + Content Quality + History | €50.6K | Same |
| Klopotek Cloud | Full Klopotek ERP suite incl. author portal module | ~€10K (small team), much more for full deployment | [Klopotek Cloud / Capterra](https://www.capterra.com/p/78716/Klopotek/) |
| Consonance | Publishing management with author features (per-user) | ~€11.4K (8 users min @ £75/user/mo) | [Consonance pricing](https://www.consonance.app/pricing/) |
| Tiptap (self-hosted) | Editor + collaboration backend only | ~€0 | [Tiptap pricing](https://tiptap.dev/pricing) |
| Tiptap Cloud Team | Editor + managed cloud | ~€1.8K | Same |
| Custom Drupal build | Author portal built from scratch | €100K – €300K TCO | Internal estimate |

**Key observations:**

1. **knk's proposed €24K/yr for a mid-size publisher delivering the complete portal is the same price range as Fonto's editor component alone.** This is a powerful sales argument.
2. **Knk's pricing is 4–10× cheaper than custom Drupal builds** at the mid-size tier — the Drupal-counter-proposal that motivated this product.
3. **Knk's transaction-based model is the only non-per-user model in the comparison set** — genuine differentiation.
4. **There is significant pricing headroom.** If knk wanted to be more aggressive on margin, mid-size could move toward €3K–€4K/mo without exceeding competitor benchmarks.

---

## 6. Why "Active Titles" Is the Recommended Transaction Unit

Five units were considered:

| Unit | Pros | Cons | Verdict |
|---|---|---|---|
| **Active titles per month** | Familiar publishing concept; aligns with engagement; backlist costs nothing | Requires clear definition of "active" | **Recommended starting point** |
| Active authors per month | Engagement-aligned | Smells like per-author in negotiation; rejected by owner | Rejected |
| Workflow events (per proof, milestone, etc.) | Granular value alignment | Feels arbitrary; customers may game by approving outside the portal | Backup |
| Per published book | Strongest value alignment | Long invoice cycles (9–18 months); high revenue volatility | Future option |
| Hybrid (base + transactional) | Predictable + value-aligned | More complex to explain | **Adopted as overall structure** |

"Active title" definition (proposed, to be ratified with first customer):

> A title is *active in a billing period* if at least one of the following events occurred against it: an author logged in and opened the title; a manuscript version was created or edited; a proof or cover review was submitted; a message was exchanged in the title's thread; a production milestone was advanced.

---

## 7. Cost-to-Serve and Margin Analysis

Approximate annual cost for knk to deliver the portal to a single mid-size customer:

| Cost line | Annual cost |
|---|---|
| Tiptap (editor + collaboration) | €0 (self-hosted) or ~€600–€2,000 (Tiptap Cloud) |
| Hosting (Azure / similar, single-tenant infrastructure share) | €5,000 – €15,000 |
| odon, guardian, core CMS (knk infrastructure, marginal cost per customer) | (sunk cost, mostly amortized) |
| Support and account management (~5% of one FTE) | €5,000 – €10,000 |
| **Total cost to deliver to one mid-size customer** | **€10K – €27K/yr** |

At the proposed mid-size price of €24K/yr, this is **break-even to ~70% gross margin** on a single customer. The picture improves dramatically with each additional customer because most knk-side costs (BC adapter development, odon/guardian/core development, product engineering) are amortized across the customer base.

---

## 8. First-Customer (Design Partner) Strategy

For customers 1 and 2, lead with a discounted "design partner" structure to lock in commitment and gather feedback:

| Component | Standard | Design partner (customers 1 and 2) |
|---|---|---|
| Setup fee | €5K – €15K | €0 – €5K |
| Monthly base | €750 – €1,000 | €500 |
| Per active title | €10/mo | €5/mo |
| Term | 12-month minimum | 24-month commitment |
| Standard price kicks in | — | Year 3 |

Justification to share with the customer: *"You're shaping the product alongside us. We benefit from your input; you benefit from preferential pricing for the first two years."*

---

## 9. Headline Pitch for the Customer Conversation

> "Approximately **€1,500 – €2,500 per month** for a typical mid-size publisher, structured as a low monthly base plus a small per-active-title fee. You only pay for titles your authors actually engage with — backlist sitting idle costs nothing. The exact transaction metric and per-unit price we'll define together as a design partner.
>
> For comparison: just the editor component used in our portal — Fonto, the industry standard for STM publishing — costs about €1,300/month for a 50-user team, and you'd still need to build everything else. We deliver the complete portal at a similar price."

---

## 10. Open Questions and Next Steps

1. **Define "active title" precisely** — with the first customer, agree on the exact set of qualifying events
2. **Set per-unit price** — €5/title (design partner) or €10/title (standard)?
3. **Decide on caps** — should there be a monthly transactional ceiling for very large publishers?
4. **Volume tiers** — at what scale do per-title rates step down?
5. **Phase 2 pricing** — when royalty/financial features are added (Phase 2), does the base fee or per-title rate increase, or is there a separate Phase 2 module fee?
6. **Multi-imprint customers** — does each imprint count as a separate billing entity?
7. **Setup fee structure for non-knk-BC customers** — Klopotek/SAP integration adds engineering effort; should setup fees vary by ERP complexity?

---

## References

- [Author Portal Design Specification](../../superpowers/specs/2026-04-14-author-portal-design.md)
- [Services Status & Roadmap](../../superpowers/specs/2026-04-30-services-status-and-roadmap-design.md)
- [Competitor Market Overview](../competitors/market-overview.md) — full competitor analysis with Fonto and Tiptap entries
- [Fonto pricing](https://www.fontoxml.com/)
- [Tiptap pricing](https://tiptap.dev/pricing)
- [Klopotek Cloud / Capterra](https://www.capterra.com/p/78716/Klopotek/)
- [Consonance pricing](https://www.consonance.app/pricing/)
