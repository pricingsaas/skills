---
name: ps-pulse-deep-dive
description: |
  Get a competitive pricing or packaging recommendation for your SaaS product. Benchmarks your pricing against 15-20 peers, surfaces price increase trends and AI bundling patterns, and delivers a data-backed recommendation report with NRR impact modeling. Also handles packaging decisions (bundle vs. add-on vs. new tier). Requires the PricingSaaS MCP (pulse.pricingsaas.com/mcp).
topics:
  - saas-pricing-intelligence
  - competitive-pricing
  - pricing-benchmarks
  - packaging-strategy
  - pricingsaas
  - pricingsaas-pulse
---

# PricingSaaS Pulse — Pricing / Packaging Brief

Get a data-backed pricing or packaging recommendation for your SaaS product, benchmarked against 12–18 real competitors tracked by PricingSaaS.

**What it does:**
- **Pricing mode** — answers "how much should we charge?" with a competitive price comparison, price-increase trend analysis, and a recommended price point
- **Packaging mode** — answers "how should we package this feature?" with a feature landscape table, approach distribution (tier-gate / add-on / bundle), and a decision matrix
- Delivers a branded HTML report at `share.pricingsaas.com` with named recommendation options and an NRR impact model

**Requires:** [PricingSaaS MCP](https://pulse.pricingsaas.com/mcp) connected to your agent. Free to install; credits consumed on report delivery.

---

You are running a PricingSaaS Pricing / Packaging Brief. This skill handles two types of questions — pricing (how much to charge, price increase analysis) and packaging (how to package a feature: bundle vs. add-on vs. tier-gate vs. new tier). Detect the mode first, then follow the matching workflow.

## Non-negotiable execution rule

You must follow these instructions end to end. Do not stop after a preview, summary, partial findings, or chat-only answer. The required final deliverable is a hosted HTML report uploaded to `share.pricingsaas.com` via `upload_report`.

Build a structured JSON spec (schema below), then fetch the hosted template, fill all tokens, write the HTML file, and call `upload_report` to deliver the URL.

If any required step cannot be completed, do not silently substitute a weaker approach. State exactly which step failed, why it failed, and what evidence or output was still produced.

---

## Step 0 — Mode detection

Before any research, classify the request:

| Mode | Signals |
|---|---|
| **Pricing** | "price increase", "what should we charge", "pricing benchmark", "raise prices", "competitive pricing" |
| **Packaging** | "rolling out X", "how should we package", "bundle or add-on", "which tier", "should this be an add-on", "new feature pricing", "how do we monetize X" |
| **Both** | Question involves a new feature AND price changes simultaneously |

Then call:

```
get_status()
```

If the MCP is unavailable, stop and say:

> This workflow requires the PricingSaaS MCP. Please install or reconnect it, then rerun.

Do not fabricate research without MCP access.

---

## Hard rule — no mid-run input requests

**Once this skill is running, never ask the user a question.** Pre-flight validation has already resolved inputs. Use the defaults below for anything unspecified, note assumptions in the Data Limitations section, and complete the report. If genuinely impossible to proceed (company not found anywhere), end with `cannot_proceed` and one sentence — not a question.

This rule applies to every step. If you cannot build a valid JSON spec or upload the report, end with `cannot_proceed` and one sentence explaining why.

## Hard rule — slug normalization

**Never pass a slug with a trailing dot, TLD, or domain extension to any MCP tool.** `search_companies` results sometimes include slugs like `notion.so`, `monday.com`, or `figma.` — these are not valid slugs. Before every `get_company_details`, `get_company_history`, or any other slug-based MCP call, strip all trailing punctuation and domain extensions:

- `notion.so` → `notion`
- `monday.com` → `monday`
- `figma.` → `figma`
- `hubspot` → `hubspot` (already clean)

The slug is always the bare name with no dots, no TLD, no trailing punctuation. If the slug from search results contains a dot, take only the part before the first dot.

**Defaults when context is unspecified:**
- **Which plan?** Analyze all plans and present each tier in the comparison tables.
- **Mode?** Default to "Both" — run pricing and packaging analysis together.
- **Segments?** Analyze SMB, mid-market, and enterprise; note distinctions in the report.
- **Comparison scope?** Same category first; broaden to adjacent SaaS if fewer than 10 peers found.

## Required user inputs

**Pricing mode** — target company (required). Plan, goal, segments, and comparison scope default per the rules above if not specified.

**Packaging mode** — target company + feature description (required). Segment, plan structure, timeline, and monetization goal default per the rules above.

---

## PRICING WORKFLOW

### P-1 — Target company data

```
get_company_details(slug)
get_company_history(slug)
```

Extract: current price (monthly + annual), pricing metric, freemium/add-ons/usage presence, last price change date and magnitude, value additions since last change.

---

### P-2 — Find comparable companies

Run in parallel:

```
search_companies(query="<category keywords>")

search_companies_advanced(
  has_freemium=<match target>,
  has_license=<match target>,
  price_min=<target * 0.5>,
  price_max=<target * 3>,
  currency="USD"
)
```

Select **15-20 companies across three tiers**:

- **Tier 1 (4-6):** Direct competitors in same product category
- **Tier 2 (5-7):** Adjacent platforms with overlapping use cases
- **Tier 3 (5-7):** Broader SaaS with matching pricing model

---

### P-3 — Gather competitor pricing

Fetch details and history in parallel batches of 4–5:

```
get_company_details(slug)
get_company_history(slug, discovery_only=True)  # period scout
get_company_history(slug)                       # full diff (1 credit/diff)
```

Capture per company: equivalent plan name, price (monthly + annual), pricing metric, most recent increase (amount, %, date, justification).

**Price increase timeline (mandatory):** Target 10+ documented increases, sorted by magnitude. Include: company, plan, from price, to price, % change, quarter/date.

---

### P-4 — Enrich with pricing knowledge

```
search_pricing_knowledge(query="<relevant pricing strategy topic>")
search_pricing_knowledge(query="<pricing model> packaging best practices")
```

These findings anchor recommendations and form the Pricing Expertise section.

---

### P-5 — Analyze patterns

Compute:
- Median peer price for equivalent plans (annual billing)
- Median increase size among companies that raised prices
- Increase frequency (how many raised in last 2 years)
- Target's percentile in peer price distribution
- Gap to next pricing tier
- Common justification patterns (AI bundling, feature expansion, tier restructuring)
- **Pro-to-Team / Individual-to-Business price ratio** — below 1.5× is a red flag

**NRR Impact Model (when ARR/NRR data is provided):**
Build a quantitative projection: Blended NRR = (B2C% × B2C_NRR) + (B2B% × B2B_NRR). Project Year 1–3 path to target NRR. Present as a table.

---

### P-6 — Formulate recommendation

Three scenarios anchored to specific peer data points:

| Scenario | Approach | Typical Range |
|---|---|---|
| Conservative | Match median peer increase %; stay below median peer price | +15-25% |
| Recommended | Align price just below median peer cluster; match most comparable peer | +25-35% |
| Aggressive | Align to median peer; close most of the structural gap | +33-50% |

**Rules:**
- Recommended scenario lands just below or at median peer price — not above it
- Name a specific peer for each scenario (e.g. "matches Loom Business at $18/mo")
- If Pro-to-Team ratio < 1.5×, even Conservative is well-supported — say so

**Per-scenario table:**
| Scenario | Annual | Monthly | Increase | Multiplier | Positioning | Risk |
|---|---:|---:|---:|---:|---|---|

---

## PACKAGING WORKFLOW

### PK-1 — Target company baseline

```
get_company_details(slug)
get_company_history(slug)
```

Extract: current plan structure, pricing metric, any AI or usage-based add-ons already present, how existing features are gated across tiers.

---

### PK-2 — Feature landscape scan

Run in parallel:

```
search_companies(query="<feature keyword> <category>")
search_companies(query="<feature name> AI <product category>")
```

For the top 12-18 matches, call `get_company_details(slug)` in parallel batches and extract how each company packages the feature:

| Approach | Definition |
|---|---|
| **Bundled-all** | All tiers include the feature |
| **Tier-gated** | Feature unlocks at a specific tier |
| **Add-on** | Separate line item on top of base plan |
| **New tier** | Distinct plan anchored on this feature |
| **Usage-add-on** | Included but metered beyond a base quota |

Also extract: tier it first appears in, price premium vs. next-lower tier, usage metric if applicable.

---

### PK-3 — Packaging knowledge enrichment

```
search_pricing_knowledge(query="feature packaging add-on vs bundle tier gate")
search_pricing_knowledge(query="<feature type> pricing SaaS packaging")
search_pricing_knowledge(query="AI feature monetization rollout strategy")
```

Pull frameworks on: when to tier-gate vs. add-on, WTP validation for new features, rollout sequencing, value communication.

---

### PK-4 — Packaging decision analysis

Compute:
- Distribution of packaging approaches (% bundled / gated / add-on / new tier / usage)
- Median price premium for tiers where the feature first appears
- Usage-metering patterns (per message, per session, per property, per seat)
- Rollout patterns (beta-free → paid, hard launch, grandfathered)

**Decision signals:**

| Signal | Points toward |
|---|---|
| Feature is table-stakes in category | Bundled-all or Tier-gated |
| Feature is differentiating / premium | Add-on or New tier |
| Feature has variable AI inference cost | Usage-add-on |
| Feature has strong upsell potential | Tier-gated (drives upgrade) |
| Feature is acquisition vehicle | Freemium / lower-tier |
| Enterprise buyers | Add-on (budget flexibility) |
| SMB buyers | Tier-gated (simpler buying) |
| New feature, uncertain WTP | Beta-free → paid after validation |

---

### PK-5 — Packaging recommendation

Present three options, then select Recommended:

| Option | Structure | Pricing | Best when |
|---|---|---|---|
| **Tier-gate** | Feature unlocks at [Plan X] | $Y/mo premium vs. lower tier | Strong upsell; SMB buyers |
| **Add-on** | Optional line item | $Z/mo per [metric] | Enterprise; variable usage |
| **New tier** | Dedicated [Feature Name] plan | $W/mo | Feature is foundational identity shift |

For the Recommended option — provide all five:
1. Packaging structure (which option and why)
2. Price / metric (specific number anchored to a named peer)
3. Rollout sequence (beta → GA → existing customer communication)
4. Upsell motion (what triggers an upgrade, what does the locked state look like)
5. Success metrics (how do you know it's working in 90 days)

---

## Publish step — fetch template → fill tokens → upload_report

After assembling the spec, produce the final HTML report:

**Step 1 — Fetch the template:**
```
web_fetch("https://share.pricingsaas.com/templates/pulse-deep-dive-v1.html")
```

**Step 2 — Fill all `{{TOKEN}}` placeholders** with data from the spec. Every token in the template must be replaced before upload. Do not leave any `{{...}}` unfilled.

**Step 3 — Write the HTML file:**
```
write(file_path="/mnt/user-data/outputs/<seed-slug>-pricing-analysis.html", content=<filled HTML>)
```

**Step 4 — Upload via MCP:**
```
upload_report(
  filename="<seed-slug>-pricing-analysis.html",
  file_path="/mnt/user-data/outputs/<seed-slug>-pricing-analysis.html"
)
```

Run the curl command from the response to perform the upload. Return the public `https://share.pricingsaas.com/...` URL as the primary output.

**Spec constraints:**
- Required keys: `mode`, `seed`, `exec`, `stats`, `landscape`, `options`, `companies`
- `pricing`: include `landscape.pricing_tiers` + `price_increase_timeline`
- `packaging`: include `landscape.feature_rows` + `patterns` + `decision_matrix` + `rollout`
- Sort `pricing_tiers[].companies` ascending by entry price; `price_increase_timeline.rows` descending by `pct_change`
- Omit optional sections (`current_state`, `nrr_model`, `expertise`, etc.) rather than fabricating

---

## Row shapes — exact field names (renderer is strict)

These are the TypeScript interfaces the renderer uses. Field names must match exactly — wrong names render as blank cells.

### PricingTierRow (pricing mode — `landscape.pricing_tiers[].companies`)

```typescript
{
  slug: string;        // e.g. "hubspot"
  name: string;        // e.g. "HubSpot"
  logo_url?: string;   // Cloudinary URL from get_company_details, or Brandfetch fallback
  plan: string;        // comparable plan name, e.g. "Starter"
  monthly: string;     // formatted monthly price, e.g. "$20/seat" — NEVER leave empty
  annual: string;      // formatted annual price, e.g. "$16/seat"  — NEVER leave empty
  metric: string;      // pricing metric, e.g. "Per seat"  ← USE "metric" NOT "model"
  last_change?: string; // optional, e.g. "Q1 2026"
  is_seed?: boolean;   // true for the anchor company row only
}
```

### PriceIncreaseRow (pricing mode — `price_increase_timeline.rows`)

```typescript
{
  slug: string;          // e.g. "freshworks"
  name: string;          // e.g. "Freshworks"
  logo_url?: string;     // optional — renderer falls back to Brandfetch
  plan: string;          // plan that changed, e.g. "Growth"
  from_price: string;    // e.g. "$15/seat"
  to_price: string;      // e.g. "$19/seat"
  pct_change: number;    // INTEGER e.g. 27  ← NOT "27%" — must be a number
  quarter: string;       // e.g. "Q1 2026"  ← USE "quarter" NOT "period" or "date"
  justification?: string; // 1-line reason, e.g. "AI bundling"
}
```

### FeatureLandscapeRow (packaging mode — `landscape.feature_rows`)

```typescript
{
  slug: string;
  name: string;
  logo_url?: string;
  feature: string;        // feature name
  approach: "bundled_all" | "tier_gated" | "addon" | "new_tier" | "usage_addon";
  approach_label: string; // human label, e.g. "Tier-gated"
  metric: string;         // pricing metric or "n/a"
  tier_or_plan: string;   // e.g. "Business" or "Add-on"
}
```

### Logo fallback (apply universally before building spec)

If `get_company_details` returns a non-empty `logo_url`, use it. Otherwise use:
```
https://cdn.brandfetch.io/{domain}?c=1idOTNPrhjEdtHU2JvP
```
where `{domain}` is the company's primary domain (e.g. `freshworks.com`). The renderer will also auto-apply this fallback for any row with missing `logo_url`, but explicit population avoids race conditions.

---

## QA gate — validate spec before outputting

Run this checklist mentally before outputting the final JSON. If any check fails, fix the spec first.

**Pricing mode — required checks:**

| Check | Pass condition | Common failure |
|---|---|---|
| `pricing_tiers` has 3 tiers | All 3 keys present, each with ≥ 3 companies | Stopped at 1–2 tiers |
| Every `PricingTierRow.monthly` non-empty | `"$20/seat"` or similar | Left as `""` or `null` |
| Every `PricingTierRow.annual` non-empty | `"$16/seat"` or `"n/a"` | Left as `""` or `null` |
| `metric` field (not `model`) | `"Per seat"`, `"Flat"` | Used `model` → blank column |
| Seed row has `is_seed: true` | Exactly one row per tier (the anchor) | Missing → no lime highlight |
| `price_increase_timeline.rows` ≥ 5 | More is better; target 10+ | Only 2–3 entries |
| `PriceIncreaseRow.pct_change` is a number | `27` not `"27%"` | String → render as `NaN%` |
| `PriceIncreaseRow.quarter` field | `"Q1 2026"` | Used `period` or `date` → blank column |
| `companies` array (Data Sources grid) | ≥ 10 slugs including seed | Thin grid looks sparse |
| All slugs are bare names | `"notion"` not `"notion."` or `"notion.so"` | Trailing dot → empty pricing rows, broken logos |

**Packaging mode — required checks:**

| Check | Pass condition |
|---|---|
| `landscape.feature_rows` ≥ 8 companies | Enough for a real distribution |
| `approach` is one of the 5 enum values | `"tier_gated"` not `"tier-gated"` or `"Tier-gated"` |
| `patterns.distribution` sums to ~100 | Percentages add up |
| `decision_matrix.rows` filled | All 8+ signal rows present |

## Final response format

After upload, return the share URL as the primary output:

> **{Pricing | Packaging} Brief: \<Seed Name\>** — N companies analyzed
>
> [View report](https://share.pricingsaas.com/...)
>
> _2-line preview: the recommendation headline + the strongest evidence point._

ALWAYS suggest 3 tailored next steps:

1. **Track competitor pricing changes** — `add_to_watchlist(slugs=[peer slugs])` + `get_pricing_news()`
2. **Map the broader landscape** — run `pulse-market-scan` for a positioning report anchored on this same seed
3. **Validate willingness-to-pay** — run `ask-willingness-to-pay-expert` for a buyer-side WTP anchor
