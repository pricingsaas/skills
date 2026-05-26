---
name: ask-willingness-to-pay-expert
description: |
  Ask any pricing strategy question and get an expert answer grounded in the WillingnessToPay.com Strategy Library — books and video case studies on pricing methodology. Produces a co-branded shareable HTML report with the five-pillar WTP reasoning scaffold, concrete next steps, and deep-linked citations. Best for methodology questions ("how do I structure my plans", "how should I run a WTP study") rather than market data questions. Requires the PricingSaaS MCP (pulse.pricingsaas.com/mcp).
---

# Ask Willingness to Pay Expert

Co-branded skill: a user asks a pricing-strategy question, the skill consults the **WillingnessToPay.com Strategy Library** (book chapters from *The Pricing Roadmap* and *Price Psychology*, plus video case-study chunks) via the PricingSaaS MCP, applies the five-pillar WTP answer architecture, and delivers a shareable expert-answer HTML report co-branded **PricingSaaS Pulse × WillingnessToPay.com**.

**Different from `pulse-pricing-benchmark`:** that skill answers "what should we charge?" with market data + a recommendation. This skill answers any pricing-strategy *methodology* question with the expert library + reasoning scaffold. No competitor cohort, no benchmark table, no `get_company_history` calls — pure WTP-grounded advice. If a question needs market data to answer, route to `pulse-pricing-benchmark` instead (or run both).

The report blends:

1. **The question** — restated verbatim so the user sees what was answered
2. **A short answer** — 1 paragraph plain-English summary the user can paste into a doc
3. **Five-pillar reasoning scaffold** — each pillar applied to the user's specific question, citing book pages or YouTube deep-links from the KB
4. **Concrete next steps** — validation moves and interview tactics drawn from the four-method ladder
5. **Sources** — every citation deep-linked: book page numbers + YouTube `&t=` timestamps

Final deliverable: a self-contained HTML file uploaded to `share.pricingsaas.com`, with the share URL returned to the user as the primary output.

---

## Phase 0: Confirm PricingSaaS MCP is installed

Before any research, verify the MCP is reachable:

```
PricingSaaS MCP:get_status()
```

If the call fails or the tool isn't available, stop and tell the user:

> "This skill requires the **PricingSaaS MCP** to be installed. Visit [pulse.pricingsaas.com/mcp](https://pulse.pricingsaas.com/mcp) for setup instructions, then re-run."

Do not attempt a degraded workflow or fabricate citations. The MCP is required because the WillingnessToPay.com Strategy Library is accessed exclusively via `search_pricing_knowledge`.

**WTP access gating:** the WTP Strategy Library is enriched into `search_pricing_knowledge` only for user emails on `pricingsaas.com` or `willingtopay.com` domains. If `get_status` shows a user outside those domains, fall back to whatever generic chunks `search_pricing_knowledge` returns and note "WTP-specific content unavailable for this user — answer is based on the public PricingSaaS knowledge base only" in the report's Data Limitations block.

If `get_status` returns successfully, note credit balance — this skill consumes ~0 credits (all KB queries + upload are free; no `get_company_history` calls).

---

## Phase 1: Clarify scope (only if missing from the original prompt)

**Skip this phase entirely if the user's original message already states the question.** Only invoke `AskUserQuestion` when the question is genuinely ambiguous (e.g., a one-word topic like "freemium").

When asking, keep it to a single round, max 2 questions:

1. **The question.** The exact strategy question to answer. If the user gave a topic only, ask them to phrase it as a question.
2. **Context (optional).** Their product / segment / current pricing if relevant. Many WTP questions are general-purpose and don't need context — don't force it.

---

## Phase 2: Knowledge base search — high-recall, multi-query (parallel, single turn)

The WTP KB returns 1500–3000 char chunks with `similarity` scores. Most pricing questions need 6–10 *good* chunks (sim ≥ 0.55) to ground a complete answer across the five pillars. To reach that, fan out 4–6 reformulated queries in parallel, then dedupe.

**Reformulation rules** (sourced from `project_wtp_api`):

- Avoid the bare phrase "willingness to pay" — it returns weak company-name hits. Use "how to measure customer willingness to pay" instead.
- Phrase around methodology, not the term. ("How do I run customer interviews to find WTP" beats "WTP interviews".)
- Mix outcome queries (pull videos / case studies) with theory queries (pull book chunks). Strategy/theory queries → books; outcome / "how did X price" queries → videos.

**Query fan-out template** — adapt to the user's question:

```
search_pricing_knowledge(query="<user's question phrased as 'how to'>")
search_pricing_knowledge(query="<core concept of question> theory")
search_pricing_knowledge(query="how to validate <topic> with customer interviews")
search_pricing_knowledge(query="how to choose pricing metric for <topic>")
search_pricing_knowledge(query="<topic> case study pricing")
search_pricing_knowledge(query="<topic> common mistakes")
```

Use `max_results=10` (the API caps at 20). Goal: surface 30–60 chunks total across the fan-out before dedupe.

---

## Phase 3: Triage chunks

Dedupe by chunk `id`. Then bucket every result into one of three reliability bands using `similarity`:

| Band | Range | Use |
|---|---|---|
| **Load-bearing** | sim ≥ 0.55 | Direct quotes / citations in the report — anchor each pillar with at least one |
| **Supporting color** | 0.50–0.54 | Optional secondary references, mark as adjacent in the synthesis |
| **Drop** | < 0.50 | Discard — too noisy to ground claims |

Then classify each load-bearing chunk by which of the **five pillars** it best supports (a chunk can map to more than one):

1. **Reframe** — segment-first thinking, "price the customer not the product"
2. **Value-chain metric** — choosing what to meter, value-aligned vs measurable
3. **Buyer-budget allocation** — primary 50–80% rule, secondary budgets, line-item pricing
4. **Validation ladder** — gut → interviews → surveys → actual sales
5. **Interview tactic** — price-then-silence, anchor discovery, what to listen for

If a pillar has zero load-bearing chunks, it's still legal to *cite the pillar generically* (the five pillars are pre-vetted in `pulse-pricing-benchmark/SKILL.md`) — but flag in Data Limitations that this run didn't surface a fresh KB chunk for it.

For each load-bearing chunk, extract:

| Field | Source | Example |
|---|---|---|
| `pillar` | classification above | 2 |
| `source_type` | response field | `Book` / `youtube_video` |
| `title` | response field | "The Pricing Roadmap" |
| `page` | `metadata.page_number` (Book only) | 138 |
| `timestamp` | `metadata.timestamp_start` (Video only) | "7:08" |
| `video_url` | `metadata.video_info.url` (Video only) | https://www.youtube.com/watch?v=… |
| `deep_link` | book → page reference; video → `${video_url}&t=${Math.floor(offset_start/1000)}` | … |
| `similarity` | response field | 0.590 |
| `excerpt` | first 280–500 chars of `content`, ≤500 in report | "Credit card fees and any other domain…" |

**Citation discipline (enforce strictly):**

- Cite only what the API returned. Never invent page numbers, timestamps, or video URLs.
- If `metadata.page_number` is missing on a Book chunk, cite as "*Title* (page TBD)" — don't guess.
- If `metadata.offset_start` is missing on a video chunk, omit `&t=` and link to the video alone.
- The skill is co-branded with WillingnessToPay.com — citations are the brand integrity check. Get them right.

---

## Phase 4: Synthesize the answer using the five-pillar architecture

The answer's spine is fixed; the content is question-specific. Build it in order:

### 4a — Short answer (1 paragraph)

Plain-English, no jargon, no citations. ~3–5 sentences. Open with the segmentation/reframe move when applicable, then state the operative answer. This is the paste-into-a-doc summary.

### 4b — The reframe (Pillar 1)

If the question is a "what should…" or "how much…" question, restate it as a "for whom, for what value, against what alternative" question. Show the user *why* their original phrasing doesn't fully resolve. Cite a Pillar-1 chunk (typically Sjöfors's "How To Price SaaS Products Accurately").

For questions where reframing isn't needed (e.g., "how do I run a pricing interview"), skip 4b and go straight to the relevant pillar.

### 4c — Pillar-by-pillar reasoning

For each pillar that's load-bearing for this question (typically 3–5 of the 5):

1. **State the principle** (1 sentence)
2. **Apply it to the user's specific question** (2–4 sentences)
3. **Quote the supporting KB chunk** (block-quoted, ≤500 chars) with its citation — book page or YouTube deep-link

Keep the pillars in their canonical order (1 → 5) for visual consistency across reports.

### 4d — Concrete next steps

Always end with action. Pull from Pillar 4 (validation ladder) and Pillar 5 (interview tactic). Format as a numbered list of 3–5 specific moves the user can do this week. Always include the **price-then-silence interview** as a named, scripted move when the question touches on validation or pricing decisions.

### 4e — Common mistakes (optional — only if KB surfaces them)

If the fan-out (Phase 2 query "<topic> common mistakes") returned a load-bearing chunk, include a short "What to avoid" subsection with the citation. Don't fabricate this section if no chunk supports it.

---

## Phase 5: Build the co-branded HTML report

The report is **a single self-contained `.html` file** with all CSS inline. No external dependencies except the PricingSaaS logo, the WTP wordmark, and any embedded chart images.

### Template loading — try local first, fall back to hosted

```
# Preferred: local skill bundle (Managed Agents native skills)
read("./template.html")

# Fallback: hosted mirror (ChatGPT, Claude Desktop, other MCP clients)
web_fetch("https://share.pricingsaas.com/templates/ask-willingness-to-pay-expert-v1.html")
```

**Always load the canonical template — never invent a design.** Both files have identical content (kept in lockstep by `scripts/sync-skills-to-anthropic.mjs`). If `read("./template.html")` succeeds, use it; otherwise `web_fetch` the hosted copy. The template is the canonical source of truth for visual design (refreshed 2026-05-08, on-brand redesign with the Q1-2026 trends-report design family + WTP co-brand layer). Inline rendering is a fallback used only if BOTH paths fail.

When inline, you must produce a report that is visually identical to the hosted template — same fonts, same colors, same component shapes. Do not invent a custom design.

### Brand tokens — co-branded palette

```css
:root {
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
  --pos:        #0f7a4a;
  --neg:        #9c1b1b;
  --warn:       #b85d00;

  /* WTP co-brand tokens */
  --wtp-primary: #1a2b4a;
  --wtp-accent:  #f5b800;
  --wtp-paper:   #fffbe8;

  --ff-display: 'Aspekta', ui-sans-serif, system-ui, -apple-system, 'Segoe UI', sans-serif;
  --ff-mono:    'Space Grotesk', ui-monospace, SFMono-Regular, 'SF Mono', Menlo, monospace;
}
```

If WTP confirms exact brand hexes, edit this skill's tokens in one place — the report rebuilds will pick them up.

**Fonts (load via Google Fonts CDN — required for on-brand typography):**

```css
@import url('https://fonts.googleapis.com/css2?family=Aspekta:wght@400;500;600;700;800&display=swap');
@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&display=swap');
```

**Legacy CSS variable names (`--ps-navi`, `--ps-lime`, `--ps-yellow`, etc.) are deprecated.** Do not use them.

### Logos (lockup)

The cover lockup uses the **PricingSaaS favicon image** (NOT a CSS swatch): `.logo-mark` containing `<img class="brand-mark" src="https://res.cloudinary.com/dd6dkaan9/image/upload/v1768670870/branding/favicon-512px-green.png" width="24" height="24">` (transparent PNG, sized 24×24 with `object-fit: contain` — works on the navy cover), an Aspekta-700 wordmark "pricingsaas", and a mono "Pulse" chip (uppercase, 9px, .18em letter-spacing).

**Browser-tab favicon (`<link rel="icon">` in `<head>`):** also use the PricingSaaS green favicon URL above. Do NOT use `https://pulse.pricingsaas.com/favicon.ico` — that path serves the Lovable.dev default icon, not the PricingSaaS brand. Required tag:

```html
<link rel="icon" href="https://res.cloudinary.com/dd6dkaan9/image/upload/v1768670870/branding/favicon-512px-green.png">
```

**Search engine indexing — required `<head>` tag:** All reports are private/client documents and must not be indexed. Include this in every generated report (the template already includes it; add it manually when rendering inline):

```html
<meta name="robots" content="noindex, nofollow">
```

The **WTP wordmark uses the WillingnessToPay.com logo image** rendered ONLY in the cobrand-bar `right` slot: `<img class="wtp-mark-img" src="https://res.cloudinary.com/dd6dkaan9/image/upload/v1749844818/uploads/upload_WTPBlack_1749844818447.png" alt="WillingnessToPay.com" height="22">`, wrapped in `<a href="https://willingnesstopay.com" target="_blank">`. **Important:** the WTP logo PNG has a **white (non-transparent) background**, so it must ONLY appear on the white-paper `.cobrand-bar`. Do NOT place the WTP image on the navy cover, navy footer, or any dark-bg surface — the white box will look wrong. In dark-bg areas (footer), use the gold accent (`var(--wtp-accent)`) as a colour cue only; do not embed the WTP image there.

### Required sections in order

1. **Cover (full-bleed navy)** — `padding: 56px 64px 48px`, `background: var(--navy)`, white text, ring-motif SVG (6 concentric `<circle>` elements with `rgba(216,227,100,.14)` stroke, top-right offset). `cover-top` row: logo lockup left, mono `cover-meta` block right ("Ask WTP Expert" eyebrow + bold date in mono caps). `cover-body`: lime mono eyebrow ("Expert Answer · WillingnessToPay.com Strategy Library") → Aspekta-800 H1 at 56px line-height 1.04, letter-spacing -.035em (one phrase wrapped in `<span class="lime-under">` for the lime-under highlight) → 17px subtitle white-78%. `cover-footer`: 4-column grid (Source library / Citations / Date / Framework) on a 1px white-16% top border, separated by 1px white-10% verticals.

2. **Co-brand seal (immediately after cover)** — `.cobrand-stripe` 4px high, `display:flex` with two equal halves (left lime, right WTP-accent gold). Then `.cobrand-bar` — white background, 14×64px padding, ink-08 bottom border, three-column flex: PricingSaaS `<span class="ps-mark">` lockup left (Aspekta-700 navy text "pricingsaas" + cream chip "Pulse"), mono uppercase ink-60 "Expert Answer · in collaboration with" center, WTP logo image right (`<img class="wtp-mark-img">` — height 22px, auto width, white-bg PNG sits on this paper-white bar).

3. **Section heads (numbered)** — Every content section opens with `.section-head`: 1px solid navy bottom border, flex with `<h2>` (Aspekta-700 28px ink, letter-spacing -.02em) on the left and `<span class="num">` mono uppercase ink-40 on the right (e.g. "01 / The brief", "02 / Executive", "03 / Pillar 1", … "11 / Honesty"). Margin-top 64px, padding-bottom 14px.

4. **The Question** — `.question-card`: WTP-paper background (`#fffbe8`), 1px WTP-primary border + 6px solid WTP-accent left strip. Mono uppercase WTP-primary label "Your question", Aspekta-500 18px ink question text, mono uppercase ink-60 q-meta line on a 1px WTP-primary-18% top border.

5. **Short Answer** — `.exec`: cream-background card (`#faffcc`) with 1px navy border + 6px solid navy left strip (the "key finding" pattern). Mono navy label + 15.5px ink body with bold-navy emphasis spans. No citations here.

6. **The Reframe (Pillar 1)** — `.reframe` panel: bone background, 4px lime left border, 24×28px padding. Inside: `.reframe-arrow` 3-column grid (original | arrow | reframed) of `.reframe-cell` paper cards each with a top border (3px ink-20 for original, 3px lime for new); large mono navy "→" between. Followed by supporting prose paragraphs and `.kb-quote` callout(s) citing the Pillar-1 chunk.

7. **Pillars 2–5** — Each pillar is its own `<section>` with its own `.section-head` (numbered "04 / Pillar 2", "05 / Pillar 3", etc.), `.section-lede` paragraphs (16px ink-80, max-width 820px), and `.kb-quote` callouts citing supporting KB chunks. KB excerpt blockquotes use `.kb-quote`: paper background, 1px ink-08 border + 3px WTP-accent left border, padded 18×22, with a large gray Aspekta quote-mark `::before` glyph at top-left, hairline-divided `.src` line at the bottom in mono ink-60. Citation format inside `.src`:
   - Book: `<b><em>The Pricing Roadmap</em></b> — Ulrik Lehrskov-Schmidt, p.138` (no link)
   - Video: `<a href="VIDEO_URL&t=OFFSET_SECONDS" target="_blank">Title — WillingnessToPay.com (MM:SS)</a>`

8. **What to avoid (conditional — only if Phase 4e surfaced a chunk)** — `.avoid` panel: `#fff1e0` background, `#f0c08a` border, `--warn` (`#b85d00`) 4px left border. Mono uppercase warn label "⚠ Anti-patterns flagged by the WTP library". `<ul>` with custom 8×1.5px hairline `::before` instead of bullets.

9. **What to do this week (Recommendations — dark navy card)** — `.recommendations`: full navy background, white text, with a faint ring SVG bottom-right. Mono lime label, Aspekta-700 24px white H3. `<ol>` with `counter-reset: rec`, each `<li>` has `counter(rec, decimal-leading-zero)` rendered as `::before` in lime ("01" "02" …) at top-left of a 56px-padded item with 3px lime left border on white-4% background. Bold-white emphasis spans. Always include the price-then-silence interview as a named, scripted move.

10. **Sources** — `.sources` panel: bone background, 1px ink-08 border. Two `.group-head` sub-sections ("📚 Books" then "▶ Videos — WillingnessToPay.com"), each followed by a 2-column `.src-grid` of `.src-item` cards. Books use `.src-item.book` (3px navy left border); Videos use the default 3px WTP-accent left border. Each item: Aspekta-600 13px title, mono 11px ink-60 location with deep-links, 13px ink-80 description.

11. **Data Limitations** — `.limitations` callout: `#fff8e6` background, `#f0d585` border, `#d4a017` 4px left border. Mono uppercase label in `#7a5b00`. `<ul>` with custom 8×1.5px hairline `::before` instead of bullets, ink color `#5b4500`. Honest list of: pillars without fresh KB grounding this run, any "page TBD" book chunks, any chunks dropped due to low similarity, and the WTP-domain-gating flag if applicable.

12. **Footer (dark navy)** — `padding: 28px 64px 32px`, navy background, white-70% mono text. Left: 16×16 PricingSaaS favicon `<img class="brand-mark">` (the same image as the cover; transparent PNG works on navy) + "Expert content provided by **WillingnessToPay.com** · Delivered via **PricingSaaS Pulse**". Right: "Generated by `ask-willingness-to-pay-expert` · {date} · {n} citations from the Strategy Library". Do NOT place the WTP white-bg image in this dark-bg footer.

**Print + responsive:** include `@media print { body { background: #fff; } .wrapper { box-shadow: none; max-width: 100%; } .recommendations, .exec, .question-card, .reframe, .avoid, .limitations, .src-item { break-inside: avoid; } }` and a `@media (max-width: 760px)` block that collapses cover padding to `36px 28px 32px`, H1 to 36px, cobrand-bar to `12px 24px`, cover-footer to 2 columns, reframe-arrow to 1 column with arrow rotated 90deg, sources grid to 1 column, footer to column flex.

**Wrapper (card-on-gray layout):** `body { background: #e9eaed; margin: 0; padding: 32px 16px; }` — light gray page. `.wrapper { background: var(--paper); max-width: 920px; margin: 0 auto; border-radius: 16px; overflow: hidden; box-shadow: 0 8px 40px rgba(0,0,0,.14); }`. The cover is INSIDE the wrapper (not full-bleed). `@media (max-width: 640px) { .wrapper { border-radius: 0; box-shadow: none; } body { padding: 0; } }`.

Save to `/mnt/user-data/outputs/<question-slug>-ask-wtp-expert.html` (lowercase, hyphenated, derived from the user's question; max 8 words in slug).

---

## Phase 6: Upload to share.pricingsaas.com

```
PricingSaaS MCP:upload_report(
  filename="<question-slug>-ask-wtp-expert.html",
  file_path="/mnt/user-data/outputs/<question-slug>-ask-wtp-expert.html"
)
```

The MCP returns a presigned PUT URL plus the final public URL. Run the curl command from the response in `bash` to perform the upload, then capture the returned `https://share.pricingsaas.com/...` URL.

If `upload_report` is unavailable, fall back to:

```
upload-file-to-share:upload_file_to_share(
  file_path="/mnt/user-data/outputs/<question-slug>-ask-wtp-expert.html",
  s3_path="reports/{YYYY-MM}/<question-slug>-ask-wtp-expert.html"
)
```

---

## Phase 7: Return the link as the primary output

The **primary output** is the share URL. Format the final response as:

> **Ask WTP Expert: \<short version of question\>** — {n} citations from the Strategy Library
>
> [View report](https://share.pricingsaas.com/...)
>
> _3-line preview of the short answer._

Then call `present_files` so the user can also open the local copy. Do not paste the full report contents into chat — the link is the deliverable.

---
