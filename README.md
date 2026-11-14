# Kanvas Launch Studio: Listing Generation Agent (n8n)

An n8n workflow that turns one structured product brief into a channel-ready marketplace listing, checks it for prohibited claims before any person sees it, and routes it through Legal, merchandiser and CX sign-off.

Built by Kritika Tyagi as the Phase 5 prototype for the NTU Agentic AI Leadership capstone. The case uses **Kanvas & Co.**, a fictional home-décor brand selling on Amazon, Noon, Lazada, Myntra and its own site. The prototype covers **Amazon AE**; the other channels reuse the same pattern with their own rules.

> Status: capstone prototype, not production software. Briefs, people and products are from the course case study.

## What's in this repo

| File | What it is |
|---|---|
| `Kanvas_Launch_Studio_Kritika_v1.3.json` | The full n8n workflow (28 nodes), ready to import |
| `README.md` | This guide |
| `docs/Kanvas_Launch_Studio_PRD_Kritika_Tyagi.pdf` | Product requirements document: problem, goals, features, key logic, launch plan, investment ([PDF](docs/Kanvas_Launch_Studio_PRD_Kritika_Tyagi.pdf) · [Word](docs/Kanvas_Launch_Studio_PRD_Kritika_Tyagi.docx)) |

## How it works

The workflow follows the redesigned launch process: **agents draft and flag; people decide.** Nothing reaches a channel with an open claim flag.

```
Stage 1  Product Brief Form ──► Brief Gate (prevention) ──► refused? ──► "Brief refused" page
                                        │ passes
Stage 2                         Listing Generation Agent (LLM)
                                        │
Stage 3                         Compliance Check (detection guardrail)
                                        ├─ BLOCKED ────────► stop page, routed for re-drafting
                                        ├─ SPEC GAP ───────► stop page, brief sent back to Stage 1
                                        ├─ HOLD (Legal) ───► Legal Review: sign off or strike
                                        └─ clean / brand fixes
Stage 4                         Merchandiser Review: approve · edit · reject (logged)
Stage 5                         Publish (system task, no AI)
Stage 6                         CX Prep Agent (LLM) ──► CX Pack Check (guardrail)
Stage 7                         CX Lead Sign-off ──► Launch Summary (who decided each stage)
```

| Node | Type | Role |
|---|---|---|
| **Brief Gate** | Code | Refuses a brief *before any AI runs* if it carries claim language, lacks Compliance sign-off, relies on unsubstantiated claims, or has missing specs. |
| **Listing Generation Agent** | OpenAI (`gpt-4o-mini`) | Drafts the title, five bullets, description and Arabic key attributes as JSON, using the brand voice, Amazon AE rules and six hard rules. Claims it can't prove go to `removed_claims`, and gaps go to `missing_info`. |
| **Compliance Check** | Code | Scans every draft against the auto-reject list (wellness, medical, sustainability, natural/safety, superlatives), implied-claim patterns, brand vocabulary and format limits. Detects and routes; never rewrites or approves. |
| **Legal Review** | Form | Legal confirms or strikes each held claim. |
| **Merchandiser Review** | Form | A human approves, edits or rejects. Each decision is written to an audit record. |
| **CX Prep Agent** | OpenAI (`gpt-4o-mini`) | Drafts FAQs, macros and known-issue notes from the approved listing. |
| **CX Pack Check** | Code | Runs the same claim checks on customer-facing CX text. |

### What changed in v1.3
- Added the Legal team's seed list of prohibited terms to both the prompt and the Compliance Check. It covers therapeutic, strong wellness, environmental and safety claims, comparatives, and invented facts.
- Added a **prompt-injection block**. The prompt tells the agent to treat brief text as data, not instructions, and the Compliance Check blocks any draft containing phrases like "ignore previous instructions" or "system prompt".
- Tested on 20 mixed-quality sample outputs: 8 of 11 flawed outputs were caught (7 blocked, 1 held for Legal), and 0 of 9 good outputs were blocked.

## Run it

**Requirements:** n8n (self-hosted or n8n Cloud, with Form Trigger and Form nodes v2.3+) and an OpenAI API key.

1. In n8n, choose **+ → New workflow**, then **⋯ → Import from File**, and pick `Kanvas_Launch_Studio_Kritika_v1.3.json`.
2. Open **Listing Generation Agent (LLM)** and **CX Prep Agent (LLM)** and select your own OpenAI credential. No credentials are included in this repo.
3. Save, then click **Execute workflow**. The Stage 1 form opens in a new tab, and each later stage opens as the next form page.

### Test briefs

**Golden brief: Mira Terracotta Planter Set** (expected: passes the gate, drafts, and goes on to the review stages)

| Field | Value |
|---|---|
| SKU | KN-PL-2447 |
| Product name | Mira Terracotta Planter Set (Set of 3) |
| Category / Channel | Planters / Amazon AE |
| Materials | Hand-thrown terracotta, fired at 980°C. Interior food-grade sealant for water retention. Cork pad included. |
| Specs | Small 12×12×14 cm, Medium 16×16×18 cm, Large 22×22×24 cm (±2 cm); set 3.6 kg; Warm Ochre / Soft Charcoal / Pale Sand; made in Jaipur; HS 6914.10 |
| Positioning | Three hand-thrown planters in coordinated sizes; no two identical; real terracotta that develops a patina. |
| Claims on file | Hand-thrown terracotta and food-grade sealant documented; recycled packaging substantiated; **no** broader sustainability, wellness or comparative claims. |

**Edge case: Cedarwood + Eucalyptus Calm Candle** (expected: **refused at the gate**, so the agent never runs)

| Field | Value |
|---|---|
| SKU | KN-CN-2451 |
| Product name | Cedarwood + Eucalyptus Calm Candle |
| Category / Channel | Candles / Amazon AE |
| Positioning | "Clinically proven to reduce stress", "improves sleep quality", "100% natural, eco-friendly soy wax" |
| Claims on file | Verbal supplier confirmation only; claims from supplier marketing; Compliance has **not** signed off. Dimensions not specified. |

If a draft for the golden brief still slips in a word like "sustainable", the Compliance Check holds it for Legal. That is the guardrail working as designed: the prompt reduces errors, and the check makes the review impossible to skip.

## Design principles
- **Prevention, detection, response:** the gate stops bad briefs, the check stops bad drafts, and Legal and the merchandiser decide what is left.
- **People hold the decisions:** Legal rules on claims, the merchandiser on publishing, and the CX Lead on launch readiness.
- **Everything is logged:** every review decision is recorded, and the launch summary shows who decided each stage.
