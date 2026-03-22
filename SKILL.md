---
name: market-research
description: Conduct market research, competitive analysis, investor due diligence, and industry intelligence with source attribution and decision-oriented summaries. Always activate when the user wants market sizing, competitor comparisons, fund research, technology scans, TAM/SAM/SOM estimates, or any research that informs a business decision. Also activate when the user says "research X", "what do you know about Y company", "compare these competitors", "who should I talk to about Z", or "help me think through this market" — even without the word "research".
---

# Market Research

Produce research that supports decisions, not research theater. Every deliverable should make a specific decision easier for a specific person.

## Workflow

When this skill activates:

1. **Ask the decision question first** — before searching anything, ask: "What decision will this research support?" The answer determines mode, depth, and output format. One question; don't stall.
2. **Choose the research mode** from the taxonomy below. Modes can combine.
3. **Build a search plan** — identify what to look for and where before searching. See Signal Sources below.
4. **Search in passes** — broad first, then targeted. Use web search throughout; don't rely on training knowledge for current market facts, funding rounds, or pricing.
5. **Separate fact from inference** — label everything. Never present an estimate as a fact.
6. **Synthesize toward the decision** — findings that don't bear on the decision get cut.
7. **Run the quality gate** before delivering.

**When to ship vs. do another pass:** if the next pass would change the recommendation, keep going. If it would only add more supporting evidence, ship.

---

## Research Standards

1. **Source every important claim** — include URL and access date where possible
2. **Flag stale data explicitly** — anything older than 18 months in a fast-moving market warrants a caveat
3. **Separate fact, inference, and estimate** — use labels: `[fact]`, `[inferred]`, `[estimate]`
4. **Include the bear case** — find the strongest argument against your emerging thesis and address it directly
5. **Translate to a decision** — the last thing you write should be: "Based on this, I would / would not [action], because..."

---

## Research Modes

### Investor / Fund Diligence

**Purpose:** Determine whether this fund is worth pursuing and how to approach them.

**What to find:**
- Fund size, stage focus, and typical check size (Crunchbase, fund website, PitchBook if available)
- Relevant portfolio companies — especially direct overlaps and adjacent bets
- General partner backgrounds and stated thesis (Twitter/X, Substack, podcast appearances)
- Recent investment activity — what they've funded in the last 6–12 months signals current priorities
- Known anti-portfolio signals — sectors or models they've publicly passed on

**Fit assessment:** after gathering, score the fit across three axes:
- Stage fit (are we at the right stage for their fund?)
- Thesis fit (does our category match their current focus?)
- Proof point fit (do we have what they typically need to write a check?)

**Deliverable:** a one-page dossier per fund with a clear fit / no-fit / maybe verdict and a tailored outreach angle.

---

### Competitive Analysis

**Purpose:** Understand the competitive landscape well enough to position accurately and anticipate moves.

**What to find:**
- Product reality (not marketing copy) — find user reviews on G2, Capterra, Reddit, App Store, and Trustpilot for unfiltered product experience
- Pricing structure — look for pricing pages, teardowns, community discussions, and sales Reddit threads
- Distribution model — where do they get customers? Outbound, PLG, marketplace, partnership?
- Funding and investor history — Crunchbase, TechCrunch, their blog
- Team and hiring signals — LinkedIn headcount trends, active job postings reveal strategic priorities
- Technical signals — GitHub (open source activity, stars, issue velocity), StackOverflow tags, developer community

**Positioning gaps:** identify what every competitor is bad at (per real user reviews) that your target customers care about. That's your opening.

**Deliverable:** comparison matrix (you vs. each competitor on 6–8 dimensions that buyers actually care about) plus a narrative on where the gap is and whether it's defensible.

---

### Market Sizing

**Purpose:** Establish a credible range for the addressable market — not a confident number, a reasoned range with explicit assumptions.

**Approach — run both and reconcile:**

**Top-down:**
- Find total industry revenue from analyst reports, public filings, or trade associations
- Apply segmentation percentages to isolate your addressable slice
- Express as a range (reports rarely agree — use the spread as your uncertainty band)

**Bottom-up:**
- Identify your ideal customer profile precisely
- Estimate the number of such customers (use LinkedIn company filters, industry databases, government data)
- Multiply by realistic ACV or annual spend
- Apply a realistic win rate and ramp timeline

**Reconciliation:** if top-down and bottom-up are more than 5× apart, you have an assumption problem — find it before presenting.

**Label every number:** `[analyst estimate, 2024]`, `[our calculation]`, `[assumption: 15% win rate]`

**Deliverable:** TAM / SAM / SOM in a table with a row per assumption, labeled by source and confidence.

---

### Technology / Vendor Research

**Purpose:** Determine whether this technology or vendor is the right fit, and what the risks are.

**What to find:**
- Technical reality — find independent technical assessments, not vendor documentation. Look for engineering blog posts, conference talks, and community discussions
- Adoption signals — GitHub stars, npm/PyPI downloads, Stack Overflow activity, job posting volume using the technology
- Integration complexity — find migration stories and integration war stories on Hacker News, Reddit, and engineering blogs
- Lock-in risk — can you leave? What does egress cost? Is your data portable?
- Security and compliance posture — SOC 2, ISO 27001, GDPR, HIPAA — check their trust page and any known CVEs
- Vendor health — funding, headcount trend, customer references, churn signals

**Build vs. buy signal:** if integration complexity exceeds two sprints or lock-in risk is high, flag the build option explicitly.

**Deliverable:** a fit assessment with explicit go / no-go / evaluate-further verdict and the key risks if you proceed.

---

### Thesis Pressure-Test

**Purpose:** Before committing (building, funding, entering), find the strongest argument against the thesis and determine whether it's fatal.

**What to find:**
- Who has tried this before and failed — and why (not just "bad execution")
- What structural forces would make this hard regardless of execution quality
- What would have to be true for this to work — and how likely each assumption is
- Who is well-positioned to copy this if it succeeds
- What the market looks like if the strongest competitor wins

**Steelman the bear case:** write the most compelling version of why this is a bad idea. Then address it directly. If you can't address it, that's the answer.

**Deliverable:** a two-column document — bear case and rebuttal — plus a summary verdict on whether the thesis holds.

---

## Source Quality Hierarchy

Not all sources are equal. Apply appropriate skepticism:

| Tier | Sources | Trust Level |
|------|---------|-------------|
| 1 — Primary | SEC filings, official company reports, government data, academic papers | High — verify the filing date and context |
| 2 — Analyst | Gartner, Forrester, IDC, CB Insights, Pitchbook | High for sizing, moderate for predictions |
| 3 — Quality press | FT, WSJ, Bloomberg, Reuters, The Information | High for facts, low for forecasts |
| 4 — Trade press | TechCrunch, VentureBeat, The Verge | Moderate — often based on press releases |
| 5 — Community | G2, Reddit, Hacker News, Twitter/X, Blind | Low reliability, high signal-to-noise on real sentiment |
| 6 — Company-owned | Press releases, blog posts, marketing copy | Always confirm independently; treat as advocacy |

**Cross-tier rule:** any important claim that rests only on Tier 4–6 sources needs a caveat or additional corroboration.

---

## Indirect Signals Worth Searching

These are often more revealing than official sources:

**Job postings** — what a company is hiring for reveals strategic priorities 6–12 months before announcements. A sudden surge in sales engineers signals a go-to-market push. Hiring ML engineers at an infrastructure company signals a product pivot.

**LinkedIn headcount** — growth rate over the past 12 months signals runway and momentum. Decline signals trouble. Filter by function to see where they're investing.

**GitHub** — open source repos reveal technical architecture, activity level signals investment, issue tracker reveals product quality and responsiveness, star/fork velocity signals developer interest.

**G2 / Capterra / App Store reviews** — sorted by recency and lowest ratings. The complaints reveal real product weaknesses that positioning copy never will.

**Patent filings** — leading indicator of R&D direction. Search Google Patents by assignee.

**Hacker News "Ask HN" and discussions** — search `site:news.ycombinator.com [company name]` for unfiltered technical and founder community opinion.

**Conference talks and podcast appearances** — executives reveal strategy and thinking that press releases don't. Search YouTube and Spotify for founder and exec appearances in the past 12 months.

---

## Output Format

Calibrate length to the decision's stakes. A quick competitive scan is different from pre-board due diligence.

### Standard Report Structure

```
## [Research Subject]: [One-Line Verdict]

### Executive Summary
2–4 sentences. The decision, the strongest supporting evidence, the main risk.
No new information — only what's in the report below.

### Key Findings
Findings that bear on the decision. Labeled [fact], [inferred], or [estimate].
Source attributed per claim. Not exhaustive — only decision-relevant.

### [Mode-specific section]
For competitive analysis: comparison matrix
For market sizing: TAM/SAM/SOM table with assumptions
For investor diligence: fund fit assessment
For technology: build/buy/evaluate assessment

### Bear Case
The strongest argument against the thesis or recommendation.
Addressed directly, not dismissed.

### Recommendation
"Based on this research, I would [action] because [1–2 key reasons]."
If confidence is low: "I cannot recommend without [specific missing information]."

### Sources
Numbered list with URL, publication, access date, and tier (1–6).
```

### Length Guidelines

| Scope | Length |
|-------|--------|
| Quick scan (single company, single question) | 400–700 words |
| Standard research memo (one mode, one decision) | 700–1400 words |
| Deep diligence (pre-investment, pre-entry) | 1500–3000 words, split into sections |

---

## Quality Gate

Before delivering, confirm:

- [ ] Every important claim is sourced — URL or explicit label as `[estimate]` or `[inferred]`
- [ ] Data older than 18 months in a fast-moving market is flagged as potentially stale
- [ ] The bear case is present and addressed, not just acknowledged
- [ ] The recommendation follows directly from the findings — no logical gaps
- [ ] Tier 4–6 sources for important claims are either corroborated or caveated
- [ ] The output answers the original decision question — cut findings that don't
- [ ] The final sentence makes the recommendation explicit, not implied
