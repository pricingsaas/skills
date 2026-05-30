---
name: ps-pulse-market-scan
description: |
  Map the competitive landscape around any SaaS company. Give us a company name and we'll position it against three rings of competitors — direct alternatives, adjacent players, and the broader category — producing a branded HTML report with pricing comparisons, model analysis, and positioning observations. Requires the PricingSaaS MCP (pulse.pricingsaas.com/mcp).
topics:
  - saas-pricing-intelligence
  - competitive-pricing
  - pricing-benchmarks
  - pricing-research
  - pricingsaas
  - pricingsaas-pulse
---

# PricingSaaS Pulse — Competitive Landscape

Map the pricing landscape around any SaaS company. Give a company name and get a competitive map across three rings — direct competitors, adjacent players, and broader category — with pricing comparisons, model analysis, velocity trends, and strategic recommendations.

**What it does:**
- Positions the seed company against 12–18 tracked competitors across three competitive rings
- Surfaces pricing model patterns, change velocity, discount structures, and billing metrics
- Delivers a branded HTML report at `share.pricingsaas.com` with ring positioning, price comparison chart, and recommendations

**Requires:** [PricingSaaS MCP](https://pulse.pricingsaas.com/mcp) connected to your agent. Free to install; credits consumed on report delivery.

---

You are running a PricingSaaS **company-anchored competitive landscape** workflow.

## Hard rule — no mid-run input requests

**Once this skill is running, never ask the user a question.** Pre-flight validation has already resolved inputs. Commit to the best available assumption, note it in the executive summary or Data Limitations, and complete the report. If it is genuinely impossible to produce any useful output (company cannot be found anywhere), end with `cannot_proceed` and a one-sentence reason — not a question.

## Non-negotiable execution rule

You must follow these instructions end to end. Do not stop after a preview, summary, partial findings, or chat-only answer. The required final deliverable is a hosted HTML report uploaded to `share.pricingsaas.com` via `upload_report`.

Build a structured JSON spec (schema in `## Mandatory report template`), then fetch the hosted template, fill all tokens, write the HTML file, and call `upload_report` to deliver the URL.

Steps 3b (velocity), 3c (discounts), 3d (metrics) remain MANDATORY because their data populates the spec fields the renderer needs.

If any required step cannot be completed, do not silently substitute a weaker approach. State exactly which step failed, why it failed, and what evidence or output was still produced.

---

## Required user inputs

The user request must include either:

- A **specific seed company** (preferred), e.g. "How does Figma compare?", "Map Jobber's competitive landscape", "Who competes with BrowserStack?"
- A **category-only request** as fallback, e.g. "field service management software pricing landscape"

If only a category is given, automatically pick the most-tracked company in that category (highest number of pricing events in the last 90 days via `search_companies_advanced`) as the seed. Note the choice in the executive summary: "Anchored on [Company] as the most-tracked company in [category]." Do not ask the user for a seed — proceed immediately.

The skill always converges on the same structured output: three competitive rings positioned around the seed (or category median in fallback).

---

## Required tools

Use the PricingSaaS MCP. Start by calling:

```
get_status()
```

If the MCP is unavailable, stop and say:

> This workflow requires the PricingSaaS MCP. Please install or reconnect it, then rerun.

Do not fabricate research without MCP access.

---

## Execution mode

Check the `category` input first to determine the execution mode:

### Watchlist mode (when `category` starts with `[WATCHLIST MODE]`)

The user selected specific watchlists to scope the landscape. The company set is pre-defined — **do not run `search_companies` or `search_companies_advanced`**.

1. **Extract slugs** from the `category` field — everything after "Slugs to fetch (call get_company_details for each):" and before the next period.
2. **Remove the seed company slug** from the list (it is the anchor, not a ring member).
3. **Fetch all remaining companies in parallel** via `get_company_details(slug)` — batches of 6.
4. **Distribute into 3 rings** based on similarity to the seed anchor profile:
   - **Ring 1 (Direct):** Same primary use case and target customer segment; closest price band overlap.
   - **Ring 2 (Adjacent):** Overlapping workflows from a different angle, different primary use case or customer segment.
   - **Ring 3 (Broader category):** Same general space but different model, price tier, or a clear outlier (freemium anchor, enterprise-only, etc.).
   Use price, pricing model, employee count, and positioning language from the fetched details to assign rings. Every watchlist company must appear in exactly one ring.
5. **Skip Steps 2 and 3** — but run the **Discovery Pass** (Step 5a) and **Web Price Augmentation** (Step 5b) below before jumping to Step 4.

#### Step 5a — PricingSaaS discovery pass (watchlist supplement)

After distributing watchlist companies into rings, search for additional relevant companies that belong in the landscape but aren't in the watchlist:

```
search_companies(query="<seed primary product keywords>")
search_companies_advanced(has_license=True, price_min=<ring median * 0.3>, price_max=<ring median * 3>)
```

For each result not already in the rings:
- Fetch details: `get_company_details(slug)`
- If genuinely relevant (same category or adjacent workflow), assign to the appropriate ring
- Mark every discovered company with `[DISCOVERED]` internally; render with a `PS Discovery` label and dashed-border card in the report's data sources grid

Limit discovery additions to **4–6 companies** across all rings combined. Skip anything clearly out-of-scope.

#### Step 5b — Web price augmentation (contact-only companies)

For any company in the rings whose pricing is contact-only (no published price found in PricingSaaS):
1. Run `WebSearch("{{company_name}} pricing cost per month site:reddit.com OR site:g2.com OR costbench.com OR capterra.com")`
2. If a credible estimate is found (review site, analyst benchmark, or cost-comparison resource), add it flagged with `🌐 est.` in the price column
3. Add a footnote in the report: "🌐 est. — pricing not published; estimate sourced from third-party review sites as of {{REPORT_DATE}}. Verify directly with the vendor before use."

Limit: flag at most **3 companies** with web estimates per report. Skip if no credible source is found — do not fabricate estimates.

Proceed to **Step 4** with the full combined company set (watchlist + discovered).

---

### Discovery mode (when `category` is a plain text hint or empty)

Standard workflow — proceed through all steps below.

---

## Required workflow

### Step 1 — Anchor on the seed company

If a seed company is given, fetch it first:

```
get_company_details(slug)
```

Pin the **anchor profile** from the response:

- `logo_url`, primary domain, full company name
- Plan names, monthly + annual prices, headline metric (per seat / per user / flat / usage)
- Has freemium / has trial
- Pricing-page primary segment language
- Employee count band

Every ring is filtered and grouped relative to this anchor.

In category-fallback mode, skip this step and compute a synthetic anchor from the category-median plan / price after Step 2.

#### Step 1b — Seed not in the PricingSaaS database

If `search_companies(seed_name)` returns no match (or `get_company_details` 404s), the seed isn't tracked yet. Do **both** of these before continuing:

1. **Submit the seed for ingestion** — call `add_page(url="<seed-pricing-page-url>")` so the seed is tracked from now on. Mention it in the final delivery line ("I've also submitted {{SEED_NAME}}'s pricing page so it'll be tracked going forward.").
2. **Build the anchor profile from the live pricing page** — use `WebFetch` against the seed's pricing URL and extract the same fields (plan names, monthly + annual prices, headline metric, trial/freemium, positioning language). Mark the anchor as "external — not yet in PricingSaaS DB" internally; do not advertise this in the report itself.

Apply the same protocol to ring members: if a well-known competitor isn't in the DB, submit their pricing page via `add_page` (silent — never surface a "Coverage Gaps" section). For critical-to-the-ring competitors, supplement with `WebFetch` so the ring stays complete in this report; otherwise just submit and skip.

---

### Step 2 — Build the three competitive rings

Goal: 12–20 companies total, distributed deliberately across three rings. Run discovery in parallel.

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

**Ring 2 — Less-direct / adjacent (4–6 companies):** overlapping workflows from a different angle. Loosen attribute filters, shift keywords toward adjacent verbs / outputs.

```
search_companies(query="<adjacent verb / output keyword>")
search_companies_advanced(
  has_license=True,
  price_min=<seed entry price * 0.3>,
  price_max=<seed entry price * 4>
)
```

**Ring 3 — Broader category (4–6 outliers):** wider market context — the company giving the use case away free, the one locking it to enterprise-only, the one with a fundamentally different model. These give the report range and make patterns interesting.

```
search_companies(query="<broad category keyword>")
search_companies_advanced(plan_name="<common plan name in space>")
```

Deduplicate across all three rings — each company appears in exactly one ring; pick the closest fit. **Never place the seed company itself in any ring** (it is the anchor, rendered separately).

---

### Step 3 — Pull pricing details for every ring member

Fetch details for all selected companies in parallel batches of 5–6:

```
get_company_details(slug1)
get_company_details(slug2)
...
```

Per company, extract:

- `logo_url` — **always use this Cloudinary URL from our database first. Only fall back to Brandfetch (`https://cdn.brandfetch.io/domain/{domain}?c=1idOTNPrhjEdtHU2JvP`) if `logo_url` is explicitly null or empty. Never Clearbit.**
- `pricing_page_url` — use this as the "View pricing" link in the table's Data column (direct link to the company's live pricing page)
- Plan names and prices (monthly + annual)
- Pricing metric (per user / per seat / flat / usage-based)
- Freemium / trial availability
- Add-ons
- Employee count

---

### Step 3b — Pricing change velocity (MANDATORY)

This step feeds section `06 / Pricing Change Velocity`. Do not skip — the renderer requires this section's data.

For Ring 1 and Ring 2 companies (closest competitors), pull their change history:

```
get_company_history(slug)  # for each Ring 1 + Ring 2 company, in parallel
```

From each response, extract:
- **Tracking period**: `tracking_since` → today (compute weeks tracked)
- **Total notable changes**: count of quarterly + weekly periods with events
- **Activity level**: High (3+ notable events), Medium (1–2), Static (0), Too new (<3 weeks)
- **Key events**: title, period, date, type (model change / price / promo / product / social proof)
- **Pulse diff link**: `https://pulse.pricingsaas.com/companies/{slug}/diffs/{period}`

For Ring 3 companies, skip history if credits are constrained — note "history not pulled" in the section.

Build a chronological event log of the top 8–12 notable events across all companies, tagged by type. Flag events where the competitor restructured their pricing model (highest signal).

---

### Step 3c — Discounts and annual savings (MANDATORY)

This step feeds section `07 / Discounts & Promos`. Do not skip — the renderer requires this section's data.

From `get_company_details` responses already fetched in Step 3:

**Annual billing savings**: For each plan, compare `monthly_recurring_billed_monthly` and `monthly_recurring_billed_yearly` charges. Compute: `savings_pct = round((monthly - annual) / monthly * 100)`. Classify:
- ≥25% → High (aggressive commitment incentive)
- 15–24% → Mid (industry norm)
- <15% → Low (minimal nudge)
- No annual option → note as absent

**Promos**: Extract from `discounts[]` array in the response — note `discount_category` (flash_sale, intro_discount, seasonal, migration_discount, etc.), amount, and duration. If discounts array is empty, note "no published promos."

**Trials**: `has_trial` and `trial_days` fields (or infer from freemium/plan names).

---

### Step 3d — Pricing metrics (MANDATORY)

This step feeds section `08 / Pricing Metrics`. Do not skip — the renderer requires this section's data.

From `get_company_details` responses already fetched in Step 3:

For each plan's `charges[]`, extract:
- `charge_type`: `flat` (fixed fee) or `metered` (scales with usage)
- `pricing_metric_id` / metric name: per seat / per user / per property / per reservation / per event / etc.
- `usage_type`: `subscription` (recurring) or `pay_as_you_go`
- `billing`: `monthly_recurring_billed_monthly` / `monthly_recurring_billed_yearly` / `usage` / `pricing_not_disclosed`

Classify each company's headline metric:
- **Per property / per unit** — `metered` + property/unit metric
- **Per user / per seat** — `metered` + user/seat metric
- **Per reservation / % of bookings** — `metered` + reservation metric or `usage` billing
- **Flat subscription** — `flat` + `subscription`, no metric
- **Flat minimum** — `flat` but pricing page language says "minimum package" or "starting from"
- **Volume bands** — `flat` but plan names indicate listing-count brackets (e.g. "2–14 listings")
- **Contact only** — `pricing_not_disclosed` across all plans
- **Hybrid** — mix of models across tiers (e.g. usage on entry, flat on higher tiers)

Also note any **adjacent / industry stack** companies you know are in this category (channel managers, RMS, adjacent SaaS) and their dominant billing metric — adds market context even without pulling their PricingSaaS data.

---

### Step 3e — Fill coverage gaps (silent)

Compile a list of well-known competitors that returned no results from `search_companies`. For each, use `WebSearch` (parallel batch of 4–6) to find their official pricing page URL. If found, submit:

```
add_page(url="https://example.com/pricing")
```

Skip enterprise-only / contact-us-only sites.

**Do NOT include a "Coverage Gaps" section in the HTML report** — never advertise what's missing.

---

### Step 4 — Position the seed inside each ring

For each ring, compute:

- **Median entry price** (excluding the seed)
- **Seed delta** — seed entry price minus ring median, expressed as `+N% above` / `-N% below` / `at parity`
- **Model match rate** — % of ring using the same headline metric as the seed
- **Freemium incidence** — % of ring with a free tier
- **Seed rank** — position N of M when ring members are sorted by entry price

In category-fallback mode, replace "seed" with "category median."

Identify 3–5 **positioning observations** that surface across the rings:

- Is the seed cheaper / pricier than direct? Closer to or farther from adjacent?
- Does the seed match the dominant model in Ring 1, or break from it?
- Does the seed surface freemium where the direct ring doesn't (or vice versa)?
- Are there outliers in Ring 3 that change how the seed should think about its packaging?

These observations drive the executive summary and pattern callouts.

---

## Publish step — fetch template → fill tokens → upload_report

After assembling the spec:

**Step 1 — Fetch the template:**
```
web_fetch("https://share.pricingsaas.com/templates/pulse-market-scan-v1.html")
```

**Step 2 — Fill all `{{TOKEN}}` placeholders** with data from the spec. Every token must be replaced.

**Step 3 — Write the HTML file:**
```
write(file_path="/mnt/user-data/outputs/<seed-slug>-competitive-landscape.html", content=<filled HTML>)
```

**Step 4 — Upload:**
```
upload_report(
  filename="<seed-slug>-competitive-landscape.html",
  file_path="/mnt/user-data/outputs/<seed-slug>-competitive-landscape.html"
)
```

Run the curl command from the response. Return the `https://share.pricingsaas.com/...` URL as the primary output.

### Spec schema

```jsonc
{
  "seed": {
    "name": "Intercom",
    "logo_url": "<cloudinary URL preferred; Brandfetch fallback>",
    "entry_price": "$29/seat"
  },
  "meta": {
    "report_subtitle": "<optional>",
    "companies_analyzed": 13,
    "pricing_models_count": 5,
    "freemium_pct": 23,
    "direct_ring_median": "$21/seat",
    "adjacent_ring_median": "$45/seat"
  },
  "exec": {
    "headline": "Intercom prices at $29/seat — 35% above the direct ring's median.",
    "body_html": "2–3 sentences. <b>Inline bold</b> is allowed; other tags are stripped."
  },
  "rings": {
    "direct":   { "lede": "...", "companies": [ /* CompanyRow[] */ ] },
    "adjacent": { "lede": "...", "companies": [ /* CompanyRow[] */ ] },
    "category": { "lede": "...", "companies": [ /* CompanyRow[] */ ] }
  },
  "patterns": ["Intercom is...", "Intercom matches...", ...],
  "price_comparison": {
    "rows": [ /* ChartRow[] — comparable-unit only */ ],
    "footnote": "Not on chart: Twilio (usage), Crisp (enterprise min)."
  },
  "velocity": {
    "lede": "...",
    "activity_rows": [ /* VelocityActivityRow[] */ ],
    "event_log":     [ /* VelocityEvent[] — top 8-12 across cohort */ ],
    "callout": "1-paragraph insight."
  },
  "discounts": {
    "lede": "...",
    "savings_rows": [ /* DiscountRow[] */ ],
    "no_annual_companies": ["Company A", "Company B"],
    "promos": [ /* Promo[] */ ],
    "callout": "1-paragraph insight."
  },
  "metrics": {
    "lede": "The dominant billing unit in this market is per-seat.",
    "direct":   [ /* MetricRow[] */ ],
    "adjacent": [ /* MetricRow[] — known industry-stack tools */ ],
    "callout": "1-paragraph insight."
  },
  "recommendations": ["Intercom should...", ...]
}
```

### Row shapes

- **CompanyRow** — `{ slug, name, logo_url, plan, monthly, annual, model, free_trial, pricing_url, is_seed? }`. `monthly`/`annual` are formatted strings (e.g. `"$29"`, `"Free"`, `"Contact"`). In `rings.direct.companies`, set `is_seed: true` on the seed's row at its natural price-sorted position — the renderer applies the lime left-border highlight.
- **ChartRow** — `{ slug, name, logo_url, price (number), price_label (formatted), is_seed? }`. Include ONLY companies billing on the same unit as the seed (per-seat / per-user / per-agent). Move usage-based / freemium-only / contact-only to `price_comparison.footnote`.
- **VelocityActivityRow** — `{ slug, name, tracked_since, activity_level: "high"|"medium"|"static"|"new", event_count }`. Levels: High (3+ notable events), Medium (1–2), Static (0), Too new (<3 weeks tracked).
- **VelocityEvent** — `{ slug, date ("Mon DD"), title, period (for the diff URL), type: "model"|"price"|"promo"|"product"|"social", highlight? }`. Set `highlight: true` on the most signal-rich event (e.g. a model-restructure).
- **DiscountRow** — `{ slug, name, plan, monthly, annual_per_month, save_pct (number), save_tier: "high"|"mid"|"low" }`. Tier mapping: high ≥25%, mid 15–24%, low <15%.
- **Promo** — `{ slug, name, label, type: "trial"|"switch"|"gap" }`. `gap` means "Not tracked".
- **MetricRow** — `{ slug?, name, logo_url?, metric (e.g. "Per seat"), model_type, evidence (1-line note) }`. Direct rows should include slug+logo; adjacent rows can omit.

### Sorting

- Every ring's `companies` array: ascending by entry price.
- `price_comparison.rows`: ascending by price.
- `velocity.event_log`: chronological (oldest first or newest first — your call, just consistent).

### Empty-data fallback

If a step's data is genuinely empty (no discounts published, no history available, etc.), still pass the section — leave the array empty and use the `callout` field to explain. The renderer produces clean placeholders so no section is ever missing.

### Final response format

After upload, return the share URL as the primary output:

> **Competitive Landscape: \<Seed Name\>** — N companies across 3 rings
>
> [View report](https://share.pricingsaas.com/...)
>
> _2-line preview: where the seed sits in the direct ring + the most useful positioning observation._

ALWAYS suggest 3 tailored next steps:

1. **Pricing history for the seed** — `get_company_history(slug)` + `get_diff_highlight`
2. **Track the direct ring** — `add_to_watchlist(slugs=[direct ring slugs])` + `get_pricing_news()`
3. **Pricing strategy for the seed** — run `pulse-deep-dive` for plan-by-plan recommendations or `pulse-pricing-benchmark` for a WTP anchor
