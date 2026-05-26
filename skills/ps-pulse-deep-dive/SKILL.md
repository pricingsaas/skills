---
name: ps-pulse-deep-dive
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

### Phase 3: Publish — single MCP call from a JSON spec

The skill no longer generates HTML. The publish step is one tool call:

```
publish_pricing_brief_report(spec=<the JSON object below>)
```

The MCP server fetches `template.html` (Q1-2026 trends-report design family — fonts, navy/lime/cream tokens, ring motif on cover, card-on-gray wrapper at max-width 1080px, lime-under highlight, cream key-finding card, dark navy final-recommendation panel, tile-format Data Sources grid — all baked in) and substitutes every token deterministically. No model time in the render step. Returns `structuredContent.public_url`.

**Forbidden:** `read`, `write`, `edit`, `upload_report` for this skill. You don't touch HTML.

#### Spec schema

```jsonc
{
  "mode": "pricing" | "packaging" | "both",
  "seed": { "name": "Notion", "slug": "notion", "logo_url": "<cloudinary URL preferred>" },
  "meta": {
    "report_title_h1": "<optional override; default is auto>",
    "report_subtitle": "<optional override>",
    "eyebrow": "<optional override; default 'PRICING BRIEF' or 'PACKAGING BRIEF'>"
  },
  "exec": {
    "question": "<optional — packaging mode shows this prominently>",
    "headline": "Recommendation in one sentence.",
    "body_html": "2–3 sentence rationale. <b>Inline bold</b> allowed.",
    "bullets": ["Aligns with X", "Reduces friction Y", ...]   // optional
  },
  "stats": [
    { "k": "Companies analyzed", "v": "15", "accent": true },
    { "k": "Recommended price", "v": "$0.99/credit" },
    { "k": "Risk level", "v": "Low" },
    { "k": "Launch timeline", "v": "Aug 11" }
  ],

  // ── PRICING MODE — pricing_tiers + price_increase_timeline ──
  "landscape": {
    "lede": "...",
    "pricing_tiers": [
      {
        "label": "Tier 1 — Direct",
        "framing": "Same product category and primary buyer.",
        "companies": [ /* PricingTierRow[] */ ],
        "takeaway": "1-line takeaway about ring deltas / median / outliers."
      },
      { "label": "Tier 2 — Adjacent", ... },
      { "label": "Tier 3 — Model-match", ... }
    ]
  },
  "price_increase_timeline": {
    "lede": "...",
    "rows": [ /* PriceIncreaseRow[] — target 10+ entries sorted by magnitude */ ]
  },
  "nrr_model": {   // optional — only when ARR/NRR data was provided
    "lede": "...",
    "rows": [ /* NrrRow[] */ ],
    "callout": "..."
  },

  // ── PACKAGING MODE — feature_rows + decision_matrix + rollout ──
  "landscape": {
    "feature_lede": "...",
    "feature_rows": [ /* FeatureLandscapeRow[] — 12-18 peers */ ],
    "feature_takeaway": "..."
  },
  "patterns": {
    "lede": "...",
    "distribution": [
      { "label": "Bundled into existing tier", "pct": 32 },
      { "label": "Add-on with credits", "pct": 41, "lead": true },
      { "label": "New tier", "pct": 18 },
      { "label": "Beta-only / free", "pct": 9 }
    ],
    "callout": "Market signal sentence — what the distribution implies for the seed."
  },
  "decision_matrix": {
    "rows": [ /* DecisionRow[] — Signal × (Tier-gate / Add-on / New-tier) */ ],
    "verdict_label": "Add-on with usage-based metering",
    "verdict_winner": "addon"
  },
  "rollout": {
    "lede": "...",
    "phases": [
      { "stage": "Phase 1", "when": "Now → Aug 11", "title": "Beta", "body_html": "..." },
      { "stage": "Phase 2", "when": "Aug 11 launch", "title": "Paid GA", "body_html": "..." },
      { "stage": "Phase 3", "when": "Q4 2026", "title": "Existing customer migration", "body_html": "..." },
      { "stage": "Phase 4", "when": "Q1 2027", "title": "Optimization & expansion", "body_html": "..." }
    ]
  },

  // ── BOTH MODES ──
  "current_state": {   // optional — seed's current plans table
    "title": "Current plan structure",
    "lede": "...",
    "plans": [ /* PlanRow[] */ ]
  },
  "expertise": {
    "lede": "...",
    "frameworks": [
      { "title": "Why usage-based works for compute features", "body_html": "...", "citation": "Tomasz Tunguz, 2024" },
      ...
    ]
  },
  "context_quotes": [   // optional — direct customer quotes from intake
    { "quote": "Our procurement keeps asking why we don't have a usage tier", "attribution": "VP Sales, Acme" }
  ],
  "options": {
    "lede": "...",
    "cards": [
      {
        "number": "01", "title": "Tier-gate behind Business",
        "rows": [
          { "key": "Structure", "value": "..." },
          { "key": "Pricing", "value": "..." },
          { "key": "Upsell motion", "value": "..." }
        ],
        "risk": "medium"
      },
      { "number": "02", "title": "Add-on with credits", "rows": [...], "risk": "low", "recommended": true },
      { "number": "03", "title": "New tier", "rows": [...], "risk": "high" }
    ]
  },
  "secondary_findings": [   // optional — 1-3 inline context subsections
    { "label": "AI MONETIZATION", "title": "Add-on phaseout pattern", "body_html": "..." }
  ],
  "final_recommendation": {   // optional — navy summary panel at bottom
    "title": "Final recommendation",
    "grid": [
      { "k": "Packaging", "v": "Add-on with usage-based metering" },
      { "k": "Price", "v": "$0.99 / 1k executions, pre-purchased in $10 credit packs" },
      { "k": "Availability", "v": "All paid tiers (Plus, Business, Enterprise)" },
      { "k": "Timeline", "v": "Beta now, paid GA Aug 11" }
    ],
    "points": [
      "Aligns with variable cost — every execution incurs compute.",
      "Removes friction vs. tier-gating — no plan upgrade required to try.",
      ...
    ]
  },
  "companies": [   // tile-format Data Sources grid (seed auto-added)
    { "slug": "figma", "name": "Figma", "logo_url": "<URL>", "ring_tag": "DIRECT" },
    { "slug": "airtable", "name": "Airtable", "ring_tag": "ADJACENT" },
    ...
  ]
}
```

**Empty-data fallback:** Optional sections (`current_state`, `patterns`, `decision_matrix`, `nrr_model`, `price_increase_timeline`, `expertise`, `rollout`, `context_quotes`, `secondary_findings`, `final_recommendation`) auto-omit if absent from the spec. Required: `mode`, `seed`, `exec`, `stats`, `landscape`, `options`, `companies`.

**Final response format to the user** (after the tool returns the URL):

> **{Pricing | Packaging} Brief: {SEED_NAME}** — {N} companies analyzed
>
> [View report](https://share.pricingsaas.com/...)
>
> _2-line preview: the recommendation headline + the strongest evidence point._

ALWAYS suggest 3 tailored next steps:

1. **Track competitor pricing changes** — `add_to_watchlist(slugs=[peer slugs])` + `get_pricing_news()`
2. **Map the broader landscape** — run `pulse-market-scan` for a positioning report anchored on this same seed
3. **Validate willingness-to-pay** — run `ask-willingness-to-pay-expert` for a buyer-side WTP anchor

### Phase 4: Verify

Use a subagent to re-query PricingSaaS and spot-check 3-5 key data claims in the report (prices, plan names, last change quarters, peer counts).


### Phase 4: Verify

Use a subagent to re-query PricingSaaS and spot-check 3-5 key data claims in the report.
