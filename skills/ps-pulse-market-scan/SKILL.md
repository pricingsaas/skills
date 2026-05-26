---
name: ps-pulse-market-scan
description: |
  Map the competitive landscape around any SaaS company. Give us a company name and we'll position it against three rings of competitors — direct alternatives, adjacent players, and the broader category — producing a branded HTML report with pricing comparisons, model analysis, and positioning observations. Requires the PricingSaaS MCP (pulse.pricingsaas.com/mcp).
---

# Pulse Market Scan — Company-Anchored Competitive Map

Map the competitive landscape **around a specific company**. Three rings — direct competitors, less-direct / adjacent players, broader category — every section positioned relative to the seed company.

This is a positioning skill, not a category survey. The seed company is the anchor: their logo headlines the report, their price is highlighted on every chart, and every pattern callout is framed as "where they sit vs. the rest of the ring." Useful for product leaders defending or challenging a position, pricing strategists locating a price anchor inside a peer set, and GTM teams briefing on a target account's competitive context.

If the user gives only a category (no seed company), ask once whether they have a company to anchor on. If they confirm "no seed," fall back to a category-median anchor and explicitly note that in the executive summary.

## Output contract: structured spec, not HTML

The skill's final deliverable is a JSON spec passed to `publish_market_scan_report(spec)` — the MCP server renders the HTML deterministically from a template. **You do NOT generate HTML for this skill.**

Steps 3b (velocity), 3c (discounts), 3d (metrics) remain MANDATORY — the renderer needs the data fields they produce. The spec schema is detailed in Step 5.

## Phase 0: Confirm PricingSaaS MCP is installed

Before any research, verify the MCP is reachable:

```
PricingSaaS MCP:get_status()
```

If the call fails or the tool isn't available, stop and tell the user:

> "This skill requires the **PricingSaaS MCP** to be installed. Visit [pulse.pricingsaas.com/mcp](https://pulse.pricingsaas.com/mcp) for setup instructions, then re-run."

Do not attempt a degraded workflow or fabricate data. The MCP is required.

## Input

The skill expects one of:

- **A specific company (preferred):** "How does Figma compare?", "Map Jobber's competitive landscape", "Who competes with BrowserStack?"
- **A company + category hint:** "Figma in design tools", "Jobber in field service" — use the hint to scope the rings
- **A category only (fallback):** "design tools pricing landscape", "field service management software" — ask once if a seed exists; if not, anchor on category median

Always converge on the same structured output: three competitive rings positioned around the seed (or category median).

## Workflow

### Step 1: Anchor on the seed company

If a seed company is given:

```
get_company_details(slug)
```

Extract from the seed and pin as the **anchor**:
- `logo_url`, primary domain, full company name
- Plan names, monthly + annual prices, headline metric (per seat / per user / flat / usage)
- Has freemium / has trial
- Pricing-page primary segment language ("teams of all sizes", "for enterprises", "for solo creators")
- Employee count band (proxy for company size / target customer)

This becomes the **anchor profile** — every ring is filtered and grouped relative to it.

If category-only fallback was confirmed, skip this step but compute a synthetic anchor profile from the category-median plan / price after Step 2.

#### Step 1b: Seed not in the PricingSaaS database

If `search_companies(seed_name)` returns no match (or `get_company_details` 404s), the seed isn't tracked yet. Do **both** of these before continuing:

1. **Submit the seed for ingestion** — call `add_page(url="<seed-pricing-page-url>")` so the seed gets tracked going forward. Mention this in the final delivery message ("I've also submitted {{SEED_NAME}}'s pricing page so it'll be tracked from now on.").
2. **Build the anchor profile from the live pricing page** — use `WebFetch` against the seed's pricing URL and extract the same fields you'd pull from `get_company_details` (plan names, monthly + annual prices, headline metric, trial/freemium, pricing-page positioning language). Note the anchor as "external — not yet in PricingSaaS DB" internally; do not advertise this fact in the report.

The same protocol applies to ring members: if a well-known competitor isn't in the DB, submit their pricing page via `add_page` (silent — never surface a "Coverage Gaps" section). If a ring member is critical to the rings (e.g. a flagship direct competitor), pull their entry-level price from `WebFetch` so the ring is still complete in this report; otherwise just submit and skip.

### Step 2: Build the three competitive rings

Goal: 12–20 companies total, distributed across three rings. Run discovery in parallel.

**Ring 1 — Direct competitors (4–6 companies):** head-on alternatives with the same primary use case and same target customer. Use seed's product description keywords, matched-pricing-model attributes, and overlapping price band.

```
search_companies(query="<seed primary product keywords>")
search_companies_advanced(
  has_freemium=<match seed>,
  has_license=<match seed>,
  price_min=<seed entry price * 0.5>,
  price_max=<seed entry price * 2.5>
)
```

**Ring 2 — Less-direct / adjacent (4–6 companies):** overlapping workflows from a different angle (e.g., for Figma: design-to-handoff tools, prototyping-only tools, dev-mode-adjacent platforms). Loosen attribute filters, shift keywords toward adjacent verbs / outputs.

```
search_companies(query="<adjacent verb / output keyword>")
search_companies_advanced(
  has_license=True,
  price_min=<seed entry price * 0.3>,
  price_max=<seed entry price * 4>
)
```

**Ring 3 — Broader category (4–6 outliers):** wider market context — the company giving the use case away free, the one locking it to enterprise-only, the one with a fundamentally different model (per-site vs. per-seat, usage-based in a per-seat market). These give the report range and make patterns interesting.

```
search_companies(query="<broad category keyword>")
search_companies_advanced(plan_name="<common plan name in space>")
```

Deduplicate across all three rings — a company can only appear in one ring; pick the closest fit. Never place the seed company itself in any ring (it is the anchor, rendered separately).

### Step 3: Pull pricing details for every ring member

Call `get_company_details(slug)` in parallel for each ring member (batches of 5–6):

```
get_company_details(slug1)
get_company_details(slug2)
...
```

Per company, extract:
- `logo_url` (use this in the report header — never Clearbit; use Brandfetch `https://cdn.brandfetch.io/domain/{domain}?c=1idOTNPrhjEdtHU2JvP` as fallback)
- Plan names and prices (monthly + annual)
- Pricing metric (per user / per seat / flat / usage-based)
- Freemium / trial availability
- Add-ons
- Employee count

### Step 3b: Pricing change velocity (MANDATORY)

This step feeds report section `06 / Pricing Change Velocity`. Do not skip.

For Ring 1 and Ring 2 companies (closest competitors), pull change history in parallel:

```
get_company_history(slug)  # for each Ring 1 + Ring 2 company
```

From each response, extract:
- **Tracking period**: `tracking_since` → today (compute weeks tracked)
- **Total notable changes**: count of quarterly + weekly periods with events
- **Activity level**: High (3+ notable events), Medium (1–2), Static (0), Too new (<3 weeks)
- **Key events**: title, period, date, type (model change / price / promo / product / social proof)
- **Pulse diff link**: `https://pulse.pricingsaas.com/companies/{slug}/diffs/{period}`

For Ring 3 companies, skip history if credits are constrained — note "history not pulled" in the section.

Build a chronological event log of the top 8–12 notable events across all companies, tagged by type. Flag events where the competitor restructured their pricing model (highest signal).

### Step 3c: Discounts and annual savings (MANDATORY)

This step feeds report section `07 / Discounts & Promos`. Do not skip.

From `get_company_details` responses already fetched in Step 3:

**Annual billing savings**: For each plan, compare `monthly_recurring_billed_monthly` and `monthly_recurring_billed_yearly` charges. Compute: `savings_pct = round((monthly - annual) / monthly * 100)`. Classify:
- ≥25% → High (aggressive commitment incentive)
- 15–24% → Mid (industry norm)
- <15% → Low (minimal nudge)
- No annual option → note as absent

**Promos**: Extract from `discounts[]` array in the response — note `discount_category` (flash_sale, intro_discount, seasonal, migration_discount, etc.), amount, and duration. If discounts array is empty, note "no published promos."

**Trials**: `has_trial` and `trial_days` fields (or infer from freemium/plan names).

### Step 3d: Pricing metrics (MANDATORY)

This step feeds report section `08 / Pricing Metrics`. Do not skip.

From `get_company_details` responses already fetched in Step 3:

For each plan's `charges[]`, extract:
- `charge_type`: `flat` (fixed fee) or `metered` (scales with usage)
- `pricing_metric_id` / metric name: per seat / per user / per property / per reservation / per event / etc.
- `usage_type`: `subscription` (recurring) or `pay_as_you_go`
- `billing`: `monthly_recurring_billed_monthly` / `monthly_recurring_billed_yearly` / `usage` / `pricing_not_disclosed`

Classify each company's headline metric: Per property / Per user / Per reservation / Flat subscription / Flat minimum / Volume bands / Contact only / Hybrid.

Also note any **adjacent / industry stack** companies you know are in this category (channel managers, RMS, adjacent SaaS) and their dominant billing metric — adds market context even without pulling their PricingSaaS data.

### Step 3e: Fill coverage gaps (silent)

After pulling details, you may know notable competitors that returned zero results from `search_companies`. Run web searches in parallel (batch of 4–6) to find their public pricing page URLs, and submit each one via `add_page(url="...")`. Skip enterprise-only / contact-us-only sites.

**Do NOT include a "Coverage Gaps" section in the HTML report — never advertise what's missing.**

### Step 4: Position the seed inside each ring

For each ring, compute:

- **Median entry price** of the ring (excluding the seed)
- **Seed delta** — the seed's entry price minus the ring median, expressed as `% above` / `% below` / `at parity`
- **Model match rate** — % of ring that uses the same headline metric as the seed (per seat vs. flat vs. usage)
- **Freemium incidence** — % of ring with a free tier
- **Where the seed ranks** — position N of M when ring members are sorted by entry price

If category fallback is in use, replace "seed" with "category median" in all of the above.

Identify 3–5 **positioning observations** that surface across the rings:
- Is the seed cheaper / pricier than direct? Closer to or farther from adjacent?
- Does the seed match the dominant model in Ring 1, or break from it?
- Does the seed surface freemium where the direct ring doesn't (or vice versa)?
- Are there outliers in Ring 3 that change how the seed should think about its packaging?

These observations drive the executive summary and pattern callouts.

### Step 5: Publish the report (single MCP call)

**The skill no longer generates HTML.** Instead, hand off a structured JSON spec and the server fills the template deterministically. This avoids the ~10-minute single-shot rendering pass that v9/v10 hit.

#### 5a. Assemble the spec

Use the data you already pulled in Steps 1–4 to build a JSON object with this shape:

```jsonc
{
  "seed": {
    "name": "Intercom",
    "logo_url": "<cloudinary or brandfetch URL>",
    "entry_price": "$29/seat"
  },
  "meta": {
    "report_subtitle": "<optional override; default is fine>",
    "companies_analyzed": 13,
    "pricing_models_count": 5,
    "freemium_pct": 23,
    "direct_ring_median": "$21/seat",
    "adjacent_ring_median": "$45/seat"
  },
  "exec": {
    "headline": "Intercom prices at $29/seat — 35% above the direct ring's median.",
    "body_html": "Two-three sentence narrative. <b>Inline bold</b> is allowed."
  },
  "rings": {
    "direct":   { "lede": "...", "companies": [ /* CompanyRow[] */ ] },
    "adjacent": { "lede": "...", "companies": [ /* CompanyRow[] */ ] },
    "category": { "lede": "...", "companies": [ /* CompanyRow[] */ ] }
  },
  "patterns": [ "Intercom is...", "Intercom matches...", ... ],
  "price_comparison": {
    "rows": [ /* ChartRow[] — comparable-unit companies only */ ],
    "footnote": "Not on chart: Twilio (usage), Crisp (enterprise min)."
  },
  "velocity": {
    "lede": "...",
    "activity_rows": [ /* VelocityActivityRow[] */ ],
    "event_log":     [ /* VelocityEvent[] — top 8-12 */ ],
    "callout": "1-paragraph insight."
  },
  "discounts": {
    "lede": "...",
    "savings_rows": [ /* DiscountRow[] */ ],
    "no_annual_companies": [ "Company A", "Company B" ],
    "promos": [ /* Promo[] */ ],
    "callout": "1-paragraph insight."
  },
  "metrics": {
    "lede": "The dominant billing unit in this market is per-seat.",
    "direct":   [ /* MetricRow[] for ring companies */ ],
    "adjacent": [ /* MetricRow[] for known adjacent tools */ ],
    "callout": "1-paragraph insight (renders on a dark navy card)."
  },
  "recommendations": [ "Intercom should...", ... ]
}
```

**Row shapes:**
- `CompanyRow`: `{ slug, name, logo_url, plan, monthly, annual, model, free_trial, pricing_url, is_seed? }`. In the direct ring set `is_seed: true` on the row for the anchor company so it gets the lime left-border highlight at its natural price-sorted position.
- `ChartRow`: `{ slug, name, logo_url, price (number), price_label (formatted), is_seed? }`. Include ONLY companies billing on the same unit as the seed (per-seat / per-user / per-agent). Move usage-based / freemium-only / contact-only entries to `price_comparison.footnote`.
- `VelocityActivityRow`: `{ slug, name, tracked_since, activity_level: "high"|"medium"|"static"|"new", event_count }`.
- `VelocityEvent`: `{ slug, date (e.g. "May 22"), title, period (for the diff URL), type: "model"|"price"|"promo"|"product"|"social", highlight? }`.
- `DiscountRow`: `{ slug, name, plan, monthly, annual_per_month, save_pct (number), save_tier: "high"|"mid"|"low" }` (high ≥25%, mid 15–24%, low <15%).
- `Promo`: `{ slug, name, label, type: "trial"|"switch"|"gap" }` (gap = "Not tracked").
- `MetricRow`: `{ slug?, name, logo_url?, metric (e.g. "Per seat"), model_type, evidence (1-line note) }`. Direct rows should include `slug` + `logo_url`; adjacent rows can omit them.

**Sorting:** every ring's `companies` array and the `price_comparison.rows` array must be sorted ascending by entry price.

**Empty-data fallback:** if a step's data is genuinely empty (e.g. zero discounts across the cohort), still pass the section — set `savings_rows: []` and `callout: "No published annual discounts across the tracked set."` and the renderer will produce a clean placeholder. NEVER omit a section.

#### 5b. Publish — one MCP call, done

```
publish_market_scan_report(spec=<the JSON object above>)
```

That's it. The MCP server:
- Validates the spec (returns a clear error if required keys are missing).
- Fetches `template.html` from GitHub raw.
- Substitutes every `{{TOKEN}}` deterministically (no model time).
- Uploads to `share.pricingsaas.com` via the same Lambda as `upload_report`.
- Returns the public URL in `structuredContent.public_url`.

**Do NOT** use `read`/`write`/`edit`/`upload_report` for this skill — they are obsolete on this path. The renderer handles all HTML, CSS, brand tokens, favicon, robots tag, and seed-row styling. Your only deliverable is the spec.

### Step 6: Deliver

Format the final reply for the user:

> **Competitive Landscape: {SEED_NAME}** — {N} companies across 3 rings
>
> [View report]({public_url})
>
> _2-line preview: where the seed sits in the direct ring + the most useful positioning observation._

ALWAYS suggest 3 tailored next steps:

1. **Pricing history for the seed** — `get_company_history(slug)` + `get_diff_highlight`
2. **Track the direct ring** — `add_to_watchlist(slugs=[direct ring slugs])` + `get_pricing_news()`
3. **Pricing strategy for the seed** — run `pulse-deep-dive` for plan-by-plan recommendations or `pulse-pricing-benchmark` for a WTP anchor

## Credit cost

`publish_market_scan_report` is 5 credits (same as the legacy `upload_report` path). All discovery / detail / history tools are free.

## Skill boundaries

- **Does:** Position a specific company against three competitive rings; surface ring deltas, model match, freemium incidence; generate a branded landscape report
- **Does not:** Recommend a specific price (use `pulse-pricing-benchmark`)
- **Does not:** Track changes over time (use `pulse-competitor-watch`)
- **Does not:** Plan-by-plan deep-dive on the seed (use `pulse-deep-dive`)
- **Does not:** Single-feature packaging research (use `pulse-feature-research`)
