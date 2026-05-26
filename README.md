# PricingSaaS Pulse Skills

Agent skills for competitive pricing intelligence, powered by [PricingSaaS Pulse](https://pulse.pricingsaas.com).

## Install

```bash
npx skills add pricingsaas/skills -a claude-code
```

Or install a specific skill:

```bash
npx skills add pricingsaas/skills --skill pulse-market-scan -a claude-code
```

## Requirements

All skills require the **PricingSaaS MCP** to be connected to your agent.
Setup instructions: [pulse.pricingsaas.com/mcp](https://pulse.pricingsaas.com/mcp)

## Skills

### `pulse-market-scan`
Map the competitive landscape around any SaaS company. Give us a company name and we'll position it against three rings of competitors — direct alternatives, adjacent players, and the broader category — producing a branded HTML report with pricing comparisons, model analysis, and positioning observations.

### `pulse-deep-dive`
Get a competitive pricing or packaging recommendation for your SaaS product. Benchmarks your pricing against 15-20 peers, surfaces price increase trends and AI bundling patterns, and delivers a data-backed recommendation report with NRR impact modeling. Also handles packaging decisions (bundle vs. add-on vs. new tier).

### `ask-willingness-to-pay-expert`
Ask any pricing strategy question and get an expert answer grounded in the WillingnessToPay.com Strategy Library — books and video case studies on pricing methodology. Produces a co-branded shareable HTML report with the five-pillar WTP reasoning scaffold, concrete next steps, and deep-linked citations.

---

[PricingSaaS](https://pricingsaas.com) · [Pulse](https://pulse.pricingsaas.com) · [WillingnessToPay.com](https://willingtopay.com)
