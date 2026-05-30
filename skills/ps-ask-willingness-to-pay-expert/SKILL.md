---
name: ps-ask-willingness-to-pay-expert
description: |
  Ask any pricing strategy question and get an expert answer grounded in the WillingnessToPay.com Strategy Library — books and video case studies on pricing methodology. Produces a co-branded shareable HTML report with the five-pillar WTP reasoning scaffold, concrete next steps, and deep-linked citations. Best for methodology questions ("how do I structure my plans", "how should I run a WTP study") rather than market data questions. Requires the PricingSaaS MCP (pulse.pricingsaas.com/mcp).
topics:
  - willingness-to-pay
  - pricing-research
  - saas-pricing-intelligence
  - packaging-strategy
  - pricingsaas
  - pricingsaas-pulse
---

# Ask WTP Expert — Pricing Strategy Q&A

Get expert pricing strategy advice grounded in the WillingnessToPay.com Strategy Library — books, webinars, and case studies from Ulrik Lehrskov-Schmidt and the WTP team. Ask any pricing methodology question and get a cited, actionable answer.

**What it does:**
- Searches the WillingnessToPay.com Strategy Library (book chapters + video transcripts) for relevant frameworks
- Applies the five-pillar WTP answer architecture: reframe → value metric → buyer budget → validation ladder → interview tactics
- Delivers a co-branded PricingSaaS × WillingnessToPay.com HTML report at `share.pricingsaas.com` with citations, next steps, and sources

**Best for:** methodology questions — "how do I price X?", "should I bundle or add-on?", "how do I validate my price?" Not for "what do competitors charge?" (use Competitive Landscape for that).

**Requires:** [PricingSaaS MCP](https://pulse.pricingsaas.com/mcp) connected to your agent. Free to install; credits consumed on report delivery.

---

You are running a co-branded **PricingSaaS Pulse × WillingnessToPay.com** expert-answer workflow.

## Non-negotiable execution rule

You must follow these instructions end to end. Do not stop after a chat-only answer or a partial summary. The required final deliverable is a hosted HTML report. The publish step is a single MCP call to `publish_wtp_expert_report(spec)` — the server fills an HTML template server-side from the spec you provide and returns the `share.pricingsaas.com` URL.

**You do NOT generate HTML for this skill.** Build a structured JSON spec instead — the renderer guarantees the co-branded PricingSaaS × WillingnessToPay.com layout, CSS, and all section structure. The spec schema lives in `## Mandatory report spec` below.

If any required step cannot be completed, do not silently substitute a weaker approach. State exactly which step failed, why it failed, and what evidence or output was still produced.

---

## Hard rule — no mid-run input requests

**Once this skill is running, never ask the user a question.** Pre-flight validation has already resolved inputs. Proceed immediately with the question as stated. Use the best available interpretation and note any ambiguity in the Data Limitations section. If genuinely impossible to proceed (no recognizable pricing-strategy question in the input), end with `cannot_proceed` and one sentence — not a question.

---

## Required user inputs

The user request must include:

- A **pricing-strategy question** (methodology, not "what do competitors charge").

If the question is clear, proceed immediately — do not ask for more context. If the question is ambiguous, choose the most reasonable interpretation and note it in Data Limitations.

If the user is asking for **competitor benchmark numbers**, do not run this skill — route them to `pulse-pricing-benchmark`.

---

## Required tools

Use the PricingSaaS MCP. Start by calling:

```
get_status()
```

If the MCP is unavailable, stop and say:

> This workflow requires the PricingSaaS MCP. Please install or reconnect it, then rerun.

Do not fabricate citations without MCP access.

**WTP gating note:** The WillingnessToPay.com Strategy Library fires inside `search_pricing_knowledge` only for users with `pricingsaas.com` or `willingtopay.com` email domains. If `get_status` shows a different domain, the run is still valid — the public KB still answers — but flag the limitation in the report's Data Limitations section.

---

## Required workflow

### Phase 1 — Multi-query knowledge base fan-out

Run 4–6 reformulated `search_pricing_knowledge` queries in parallel, all with `max_results=10`. Use these reformulation rules:

- **Avoid the literal phrase "willingness to pay"** — it returns weak company-name hits. Use "how to measure customer willingness to pay" instead.
- **Phrase queries around methodology, not topics.** "How do I run pricing interviews" beats "pricing interviews".
- **Mix outcome and theory queries.** Theory queries (how / why / framework) pull book chunks; outcome queries (case study / what happened when) pull video chunks.

Suggested fan-out template — adapt to the user's question:

```
search_pricing_knowledge(query="<user's question rephrased as 'how to'>", max_results=10)
search_pricing_knowledge(query="<core concept of the question> theory", max_results=10)
search_pricing_knowledge(query="how to validate <topic> with customer interviews", max_results=10)
search_pricing_knowledge(query="how to choose pricing metric for <topic>", max_results=10)
search_pricing_knowledge(query="<topic> case study pricing", max_results=10)
search_pricing_knowledge(query="<topic> common mistakes", max_results=10)
```

Goal: 30–60 chunks across the fan-out before dedupe. Dedupe by chunk `id`.

**Source filter (mandatory — run before Phase 2):** After deduping, discard any chunk whose `content_source` field is not one of:

- `null` — PricingSaaS-owned research (always keep)
- `"wtp"` — WillingnessToPay.com Strategy Library (always keep)
- `"transcript"` — WTP webinar transcripts (always keep)

Any other `content_source` value means partner/sponsored content that must not appear in this co-branded report. Drop it silently — do not cite it, do not reference it, do not include it in the Sources section. If this filter removes a majority of chunks and leaves fewer than 5 load-bearing ones, note "limited KB coverage for this query" in Data Limitations but do not relax the filter.

---

### Phase 2 — Triage chunks

Bucket every chunk by similarity:

| Band | Range | Use |
|---|---|---|
| Load-bearing | sim ≥ 0.55 | Direct citations in the report |
| Supporting color | 0.50–0.54 | Secondary references, mark adjacent |
| Drop | < 0.50 | Discard |

Then classify each load-bearing chunk by which **WTP pillar** it best supports (a chunk can map to more than one):

1. **Reframe** — segment-first thinking, "price the customer not the product"
2. **Value-chain metric** — choosing what to meter, value-aligned vs measurable
3. **Buyer-budget allocation** — primary 50–80% rule, line-item pricing
4. **Validation ladder** — gut → interviews → surveys → actual sales
5. **Interview tactic** — price-then-silence, anchor discovery

If a pillar has zero load-bearing chunks for this run, skip it — do not include it in `spec.pillars`.

For each load-bearing chunk, note: `pillar`, `source_type`, `title`, `page` (Book), `timestamp` (Video), `video_url`, `deep_link`, `similarity`, `excerpt` (≤500 chars).

**Citation discipline:** never invent page numbers, timestamps, or URLs. If `metadata.page_number` is missing, write "page TBD". If `metadata.offset_start` is missing, link to the video without `&t=`.

---

### Phase 3 — Compose the JSON spec

Build the spec object. This is the only output. Call `publish_wtp_expert_report(spec)` when it's ready.

---

## Mandatory report spec

### Required fields

```json
{
  "question": "<verbatim user question>",
  "report_title": "<concise title, e.g. 'Pricing AI features in a SaaS product'>",
  "report_subtitle": "<1-2 sentence description of what the report covers>",
  "short_answer": "<3-5 sentence plain-English answer, HTML b/em allowed>",
  "pillars": [ ... ],
  "next_steps": [ ... ],
  "sources": [ ... ],
  "limitations": [ ... ]
}
```

### Optional fields

```json
{
  "reframe": { ... },
  "mistakes": [ ... ],
  "citation_count": 12
}
```

---

### `pillars` — array of active pillars

One object per active pillar, in canonical order (Pillar 1 → 5). Omit pillars with no load-bearing chunks.

```json
{
  "section_label": "04 / Pillar 2 · Value-chain metric",
  "title": "<section heading — the principle applied to the user's question>",
  "lede": "<application paragraph, 2-4 sentences, b/em allowed>",
  "secondary_lede": "<optional second paragraph>",
  "quotes": [
    {
      "text": "<verbatim excerpt ≤500 chars>",
      "source_type": "book",
      "source_title": "<title>",
      "author": "<author>",
      "page": "p.138"
    },
    {
      "text": "<verbatim excerpt ≤500 chars>",
      "source_type": "video",
      "source_title": "<title>",
      "video_url": "<url with &t= timestamp if available>",
      "timestamp": "03:42"
    }
  ]
}
```

### `reframe` — optional Pillar-1 reframe

```json
{
  "original": "<original question phrasing>",
  "reframed": "<reframed question — who / for what value / vs what alternative>",
  "lede": "<optional 1-2 sentence explanation of why the reframe matters>",
  "quote": { "text": "...", "source_type": "book", "source_title": "...", "author": "...", "page": "p.42" }
}
```

Skip the `reframe` key entirely if reframing is not applicable to this question.

### `next_steps` — 3–5 action items

Always include **price-then-silence interview** as one of the named items.

```json
[
  {
    "headline": "Run 5 price-then-silence interviews this week",
    "detail": "Name your price, stop talking. The silence that follows is your data. Schedule 5 interviews with current or prospective buyers before Friday."
  }
]
```

### `mistakes` — optional anti-patterns

```json
["<anti-pattern 1 — b allowed>", "<anti-pattern 2>"]
```

Omit the key entirely if no chunk supports it.

### `sources` — books first, then videos

```json
[
  {
    "type": "book",
    "title": "The Pricing Roadmap",
    "author": "Ulrik Lehrskov-Schmidt",
    "page": "p.138",
    "description": "<1 sentence on what this source contributes>"
  },
  {
    "type": "video",
    "title": "How to run pricing interviews",
    "video_url": "https://youtube.com/watch?v=xxx&t=222",
    "timestamp": "03:42",
    "description": "<1 sentence>"
  }
]
```

### `limitations` — caveats

```json
[
  "Pillar 3 (Buyer-budget allocation) had no load-bearing chunks for this query.",
  "Page numbers for <Title> unavailable — cited as 'page TBD'."
]
```

---

## Final output

**STOP — do NOT call `publish_wtp_expert_report` or any other MCP tool as your final step.**

Output the completed JSON spec as your **final message**, inside a ```json code fence. The server-side poll worker extracts the spec and calls the renderer automatically.

```json
{
  "question": "...",
  "report_title": "...",
  ...
}
```

That is your entire final output. No prose, no HTML, no tool calls. The poll worker handles rendering and will email the user the share URL when done.
