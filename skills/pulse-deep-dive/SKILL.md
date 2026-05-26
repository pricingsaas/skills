---
name: pulse-deep-dive
description: |
  Get a competitive pricing or packaging recommendation for your SaaS product. Benchmarks your pricing against 15-20 peers, surfaces price increase trends and AI bundling patterns, and delivers a data-backed recommendation report with NRR impact modeling. Also handles packaging decisions (bundle vs. add-on vs. new tier). Requires the PricingSaaS MCP (pulse.pricingsaas.com/mcp).
---

# Pulse Pricing / Packaging Brief — Competitive Analysis & Packaging Decisions

Answer pricing AND packaging questions for SaaS companies using PricingSaaS MCP data. Produce a professional report — either a price change recommendation backed by competitor benchmarks, or a packaging decision brief (bundle vs. add-on vs. new tier) backed by how peers handle the same feature.

## Mode Detection — Pricing vs. Packaging

Before research, identify which mode applies:

**Pricing mode** (default) — user asks how much to charge, whether to raise prices, or wants competitive benchmarks:
- "should we raise prices", "price increase", "competitive pricing", "what should we charge", "pricing benchmark"

**Packaging mode** — user asks how to package a feature, whether it should be an add-on, which tier it belongs in, or how to roll out a new capability:
- "rolling out X", "how should we package", "bundle or add-on", "which tier", "should this be an add-on", "new feature pricing", "how do we monetize X"

**Both** — some questions are hybrid (e.g., "rolling out a new AI tier AND raising prices"). Run both workflows and produce a unified report.

## Phase 0: Confirm PricingSaaS MCP is installed

Before any research, verify the MCP is reachable:

```
PricingSaaS MCP:get_status()
```

If the call fails or the tool isn't available, stop and tell the user:

> "This skill requires the **PricingSaaS MCP** to be installed. Visit [pulse.pricingsaas.com/mcp](https://pulse.pricingsaas.com/mcp) for setup instructions, then re-run."

Do not attempt a degraded workflow or fabricate data. The MCP is required.

## Workflow

### Phase 1: Clarify Scope

Before research, clarify based on mode:

**Pricing mode:**
1. **Which plan?** Specific plan to analyze (e.g., "Business", "Pro")
2. **Goal?** Monetization growth, competitive repositioning, or premium positioning
3. **Segments?** SMB, mid-market, enterprise, or all three
4. **Comparison scope?** Same category only, or broader SaaS with similar pricing attributes

**Packaging mode:**
1. **What is the feature?** Name and one-line description of what it does
2. **Who is it for?** Which customer segment(s) would use it most
3. **Current plan structure?** What tiers exist today
4. **Launch timeline?** Beta now, GA in N months, or already live?
5. **Monetization goal?** NRR growth, acquisition, upsell, or all three

Skip questions the user already answered.

### Phase 2: Research

#### Step 1: Target company data

```
get_company_details(slug)      # Current plans, prices, metrics
get_company_history(slug)      # Historical pricing changes
```

Extract: current price (monthly + annual), pricing metric, freemium/add-ons/usage presence, last price change date and magnitude, value additions since last change.

#### Step 2: Find comparable companies

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

Select **15-20 companies minimum** across three tiers. Don't stop at 7-8 — a shallow comp set produces artificially wide recommendation ranges. Keep adding until you have dense coverage in each tier:
- **Tier 1 (4-6):** Direct competitors in same product category — these set the most defensible price anchors
- **Tier 2 (5-7):** Adjacent platforms with overlapping use cases or buyer persona
- **Tier 3 (5-7):** Broader SaaS with matching pricing model and motion (e.g. PLG→B2B, content library, per-seat tools) even if different category

For each tier write a one-line "Takeaway" below the table: what the tier reveals about headroom, ratio gaps, or market norms.

#### Step 3: Gather competitor pricing

Call `get_company_details(slug)` and `get_company_history(slug)` in parallel for each company (batches of 4-5):

```
get_company_details(slug1)
get_company_history(slug1)
...
```

Capture per company: equivalent plan name, price (monthly + annual), pricing metric, most recent increase (amount, %, date, justification).

**Price increase timeline (mandatory):** Extract every documented increase from the history for all companies. Target 10+ entries in the timeline table sorted by magnitude. Don't stop at 3-4 highlights — comprehensive coverage of increases across the comp set shows the market pattern more convincingly than cherry-picked examples. Include: company, plan name, from price, to price, % change, and quarter/date.

#### Step 4: Enrich with pricing knowledge

Query the knowledge base for frameworks relevant to this analysis:

```
search_pricing_knowledge(query="<relevant pricing strategy topic>")
search_pricing_knowledge(query="<pricing model> packaging best practices")
```

Examples: "per-seat pricing best practices", "freemium conversion optimization", "price increase communication strategies". These findings anchor recommendations in established methodology and form the "Pricing Expertise" section of the report.

#### Step 5: Analyze patterns

Compute:
- Median peer price for equivalent plans (annual billing)
- Median increase size among companies that raised prices
- Increase frequency (how many raised in last 2 years)
- Target's percentile in peer price distribution
- Gap to next pricing tier
- Common justification patterns (AI bundling, feature expansion, tier restructuring)
- **Pro-to-Team / Individual-to-Business price ratio** for the target and all peers — this is one of the strongest structural signals of under- or over-pricing

**NRR Impact Model (when the user provides ARR/NRR data):**
If the customer shares ARR split, NRR by segment, or target NRR — build a quantitative model:
1. Calculate current blended NRR from segment percentages
2. Show the math: `Blended NRR = (B2C% × B2C_NRR) + (B2B% × B2B_NRR)`
3. Model Year 1 impact of the price increase applied to existing customers
4. Project Year 1–3 path to target NRR from organic B2B mix shift + price increase
5. Present as a table: Timeline / B2B ARR / B2B% / B2B NRR / Blended NRR / Notes

This section converts the recommendation from qualitative ("you're underpriced") to quantitative ("here's the NRR point impact and the path to your target").

**Identify secondary finding categories** relevant to this specific company's context:
- If they're adding AI features → add an "AI Pricing Models" section benchmarking how peers handle AI monetization (bundled vs. add-on vs. separate tier)
- If they're considering vertical/industry-specific pricing → add an "Industry Library / Vertical Pricing" section
- If they have usage-based ambitions → add a "Usage-Based Pricing Models" section
- If enterprise compliance is a major factor → add a "De-risking Enterprise Playbook" section

These secondary findings make the report distinctly actionable beyond just the price number.

#### Step 6: Formulate recommendation

Three scenarios — anchored to specific peer data points, not just percentages:

| Scenario | Approach | Typical Range |
|----------|----------|---------------|
| Conservative | Match median peer increase %; stay below median peer price | +15-25% |
| Recommended | Align price to just below the median peer cluster; match most comparable peer | +25-35% |
| Aggressive | Align price to median peer; close most of the structural gap | +33-50% |

**Important calibration rules:**
- The "Recommended" scenario should land the customer **just below or at** the median peer price for their tier — not significantly above it. A recommendation that's still below most peers is far easier to defend than one that overtakes them.
- For each scenario, name a **specific peer company** it aligns with (e.g. "matches Loom Business at $18/mo") — not just a percentage. This makes it concrete.
- If the Pro-to-Team ratio is below 1.5×, the structural gap is severe enough that even Conservative is well-supported. Note this explicitly.
- For each: new price, percentage change, peer alignment, Team/Pro ratio result, and risk level.

**Per-scenario format:**
| Scenario | Annual | Monthly | Increase | Multiplier | Positioning | Risk |
|---|---:|---:|---:|---:|---|---|

---

## Packaging Workflow (run when mode = packaging or hybrid)

### PK-1: Target company baseline

```
get_company_details(slug)
get_company_history(slug)
```

Extract: current plan structure, pricing metric, any AI or usage-based add-ons already present, how existing features are gated across tiers.

### PK-2: Feature landscape — who has this feature and how do they package it?

Run in parallel:

```
search_companies(query="<feature keyword> <category>")
search_companies(query="<feature name> AI <product category>")
```

For the top 12-18 matches, call `get_company_details(slug)` and extract:
- Is this feature in base plans, gated to a higher tier, or sold as an add-on?
- What tier does it first appear in?
- Is it usage-metered, included in a flat plan, or priced separately?
- What is the price delta between tiers where the feature is/isn't present?

Classify each company's approach:
| Approach | Definition |
|---|---|
| **Bundled-all** | All tiers include the feature |
| **Tier-gated** | Feature unlocks at a specific tier (most common) |
| **Add-on** | Separate line item on top of base plan |
| **New tier** | Company created a distinct plan anchored on this feature |
| **Usage-add-on** | Feature included but metered beyond a base quota |

### PK-3: Enrich with packaging knowledge

```
search_pricing_knowledge(query="feature packaging add-on vs bundle")
search_pricing_knowledge(query="<feature type> pricing SaaS")
search_pricing_knowledge(query="AI feature monetization tier design")
```

Pull frameworks on: when to tier-gate vs. add-on, WTP validation for new features, rollout sequencing, value communication.

### PK-4: Packaging decision analysis

Compute:
- Distribution of packaging approaches (% bundled vs. gated vs. add-on vs. new tier)
- Median price premium for tiers where the feature first appears
- Any usage-metering patterns (per reply, per session, per seat, per property)
- Rollout patterns: do peers launch as beta-free → paid? Hard launch? Grandfathered?

**Decision framework — use these signals to drive recommendation:**

| Signal | Points toward |
|---|---|
| Feature is table-stakes in category | Bundled-all or Tier-gated |
| Feature is differentiating / premium | Add-on or New tier |
| Feature has variable cost (AI inference) | Usage-add-on |
| Feature has strong upsell potential | Tier-gated (drives upgrade) |
| Feature is acquisition vehicle | Freemium / lower-tier |
| Customer segment = enterprise | Add-on (budget flexibility) |
| Customer segment = SMB | Tier-gated (simpler buying) |
| Feature is new, uncertain WTP | Beta-free → paid after validation |

### PK-5: Packaging recommendation

Three options — always present all three, then select one as Recommended:

| Option | Structure | Pricing | Best when |
|---|---|---|---|
| **Tier-gate** | Feature unlocks at [Plan X] | $Y/mo premium vs. lower tier | Strong upsell signal; SMB buyers |
| **Add-on** | Optional line item | $Z/mo per [metric] | Enterprise buyers; variable usage |
| **New tier** | Dedicated [AI/Feature Name] plan | $W/mo | Feature is foundational identity shift |

For the Recommended option:
- Name a specific peer who uses this approach (e.g. "Matches how Guesty handles AI messaging — gated to Business, $X premium")
- State the expected upsell motion (how does this drive upgrades?)
- State the rollout sequence (beta → GA → existing customer communication)
- Name the pricing metric if it's usage-based

---

### Phase 3: Report

Generate a professional report as a single self-contained HTML file.

**Template loading — try local first, fall back to hosted:**
```
# Preferred: local skill bundle (Managed Agents native skills)
read("./template.html")

# Fallback: hosted mirror (ChatGPT, Claude Desktop, other MCP clients)
web_fetch("https://share.pricingsaas.com/templates/pulse-deep-dive-v1.html")
```
Both files have identical content — the sync script keeps them in lockstep. Substitute all `{{TOKEN}}` placeholders with real data. If both paths fail (template not yet published for this skill), build the HTML inline using the brand tokens below.

**Brand tokens (Q1-2026 trends-report family — use exactly; legacy `--ps-navi` / `--ps-lime` / `--ps-yellow` names are deprecated):**

```css
--navy:       #131c3b;
--lime:       #d8e364;
--lime-dark:  #aebb36;
--cream:      #faffcc;
--blue:       #3e5dc2;
--purple:     #9fadf4;
--paper:      #ffffff;
--bone:       #f7f7f7;
--ink:        #131c3b;
--ink-80:     #2a3355;
--ink-60:     #5b6480;
--ink-40:     #8b92a8;
--ink-20:     #c5c9d4;
--ink-08:     #e8e9ed;
--ff-display: 'Aspekta', ui-sans-serif, system-ui, -apple-system, 'Segoe UI', sans-serif;
--ff-mono:    'Space Grotesk', ui-monospace, SFMono-Regular, 'SF Mono', Menlo, monospace;
```

**Fonts (load via Google Fonts CDN):**

```css
@import url('https://fonts.googleapis.com/css2?family=Aspekta:wght@400;500;600;700;800&display=swap');
@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&display=swap');
```

**PricingSaaS brand mark — render as `<img class="brand-mark">`, never a CSS swatch:**

```
https://res.cloudinary.com/dd6dkaan9/image/upload/v1768670870/branding/favicon-512px-green.png
```

**Browser-tab favicon (`<link rel="icon">` in `<head>`):** also use the PricingSaaS green favicon URL above. Do NOT use `https://pulse.pricingsaas.com/favicon.ico` — that path serves the Lovable.dev default icon, not the PricingSaaS brand. Required tag:

```html
<link rel="icon" href="https://res.cloudinary.com/dd6dkaan9/image/upload/v1768670870/branding/favicon-512px-green.png">
```

**Search engine indexing — required `<head>` tag:** All reports are private/client documents and must not be indexed. Include this in every generated report:

```html
<meta name="robots" content="noindex, nofollow">
```

Use 24×24 in the cover lockup (next to the Aspekta-700 "pricingsaas" wordmark + mono "Pulse" chip on the navy cover). Use 16×16 in the footer.

**Lime-under highlight** — wrap one phrase of the H1 in `<span class="lime-under">`:

```css
.lime-under { background-image: linear-gradient(transparent 88%, rgba(216,227,100,.8) 88%, rgba(216,227,100,.8) 96%, transparent 96%); background-repeat: no-repeat; }
```

A thin 8% baseline band at 80% opacity — reads as an accent underline, not a highlighter bar.

**Required sections — Pricing mode:**
1. Executive Summary with recommendation (include key stats row: companies analyzed, quarters of history, headline recommendation)
2. Current Pricing Position (target's plans + value additions since last price change)
3. Competitive Landscape — **three separate tables by tier** (Tier 1 direct, Tier 2 adjacent, Tier 3 model-match); each table ends with a one-line "Takeaway"
4. Price Increase Trends — comprehensive timeline table with 10+ documented increases, sorted by magnitude; not just 3 highlights
5. Market Patterns (median increase, AI bundling, de-risking playbook, ratio compression signal)
6. **NRR Impact Model** — quantitative projection table if ARR/NRR data was provided; otherwise omit
7. **Pricing Expertise** — frameworks and expert context from `search_pricing_knowledge` results, with citations
8. Recommendation — three scenarios in table format; each names a specific peer company it aligns with; include numbered rationale (6+ points)
9. **Secondary Findings** — 1-3 contextually relevant sections (AI pricing models, vertical pricing, usage-based, etc.) based on what's unique to this company's roadmap
10. Implementation Guidance — tied to customer's stated launch timeline and any experiments already in flight
11. Appendix / Data Sources (all companies with PricingSaaS links)

**Required sections — Packaging mode (replace or supplement sections 3-8):**
1. Executive Summary — packaging recommendation headline + key stats (companies analyzed, feature landscape coverage)
2. Current Plan Structure — target's tiers today, any existing feature gates or add-ons
3. Feature Landscape — table of 12-18 peers, showing how each packages the feature (bundled/gated/add-on/new tier/usage), tier it appears in, and price premium
4. Packaging Pattern Analysis — distribution of approaches, usage-metering norms, rollout patterns across the comp set
5. **Packaging Decision Matrix** — scored comparison of tier-gate vs. add-on vs. new tier for this specific feature + company context
6. **Pricing Expertise** — frameworks from `search_pricing_knowledge` (feature packaging, WTP validation, rollout sequencing)
7. Recommendation — three packaging options with peer alignment; Recommended option includes: packaging structure, price/metric, rollout sequence, and upsell motion
8. **Rollout Playbook** — beta strategy, existing customer communication, grandfathering or migration offer, success metrics
9. Appendix / Data Sources

**Optional but high-value:** "Contextual Signals of Underpricing" section — pull direct quotes and anecdotes from the customer conversation that validate the case for a price increase (procurement remarks, CS hire feedback, competitor price gap observations). This section shows the customer their own words reflected back through a data lens.

Write HTML to a temp file and upload:

```
upload_report(filename="{company}-pricing-analysis.html", file_path="/tmp/{company}-pricing-analysis.html")
```

Execute the returned `curl` command to complete the upload. Use `file_path` — never base64-encode or pass `file_content`.

### Phase 4: Verify

Use a subagent to re-query PricingSaaS and spot-check 3-5 key data claims in the report.
