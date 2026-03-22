# Market Research — Output Templates

Ready-to-fill templates for each research mode. Copy the relevant template, delete unused sections, and fill in from your research.

---

## Investor / Fund Dossier

```markdown
# [Fund Name] — Investor Dossier

**Verdict:** ✅ Strong fit / ⚠️ Partial fit / ❌ Poor fit / ❓ Insufficient data

## Fund Overview
- **Fund size:** $Xm (Fund N, [year])
- **Stage focus:** Pre-seed / Seed / Series A / Growth
- **Check size:** $Xk–$Xm typical
- **Geography:** [regions]
- **Sector focus:** [stated thesis in their own words]

## Relevant Portfolio
| Company | Stage at investment | Why relevant |
|---------|-------------------|--------------|
| [co] | [stage] | [overlap or signal] |

## Recent Activity (last 12 months)
- [Investment 1, date, stage]
- [Investment 2, date, stage]
*Signal: [what this activity suggests about current priorities]*

## Key Partners
| Partner | Background | Public thesis |
|---------|-----------|---------------|
| [name] | [background] | [link or summary] |

## Fit Assessment
| Dimension | Assessment | Evidence |
|-----------|-----------|---------|
| Stage fit | [✅/⚠️/❌] | [reason] |
| Thesis fit | [✅/⚠️/❌] | [reason] |
| Proof point fit | [✅/⚠️/❌] | [what they need vs. what we have] |

## Red Flags / Mismatches
- [Any obvious conflicts, competing portfolio companies, public statements against our model]

## Outreach Angle
[Specific reason to reach out now, tailored to their stated thesis, ideally referencing a specific portfolio company or public statement]

## Sources
1. [URL] — [publication] — [date accessed]
```

---

## Competitive Analysis Matrix

```markdown
# Competitive Landscape: [Category]

**Bottom Line:** [1–2 sentences on where the gap is and whether it's exploitable]

## Players
| | [Us] | [Competitor A] | [Competitor B] | [Competitor C] |
|--|------|----------------|----------------|----------------|
| **Pricing** | | | | |
| **Primary customer** | | | | |
| **Distribution** | | | | |
| **Key strength** | | | | |
| **Key weakness** | | | | |
| **Funding / revenue** | | | | |
| **Tech approach** | | | | |
| **[Dimension buyers care about]** | | | | |

*Sources: G2, Capterra, [direct pricing pages], [funding: Crunchbase]*

## What Customers Actually Complain About
Sorted by frequency from real reviews:

**[Competitor A]:**
- [Real complaint 1 from G2/Capterra] [source]
- [Real complaint 2] [source]

**[Competitor B]:**
- [Real complaint 1] [source]

## Positioning Gap
[Competitors are weak at X. Our target customers care about X. We are strong at X.]
[Evidence that the gap is real and not already addressed by a v2 or upcoming release]

## Threat Assessment
| Competitor | Threat level | Most likely move | Our response |
|-----------|-------------|-----------------|--------------|
| [A] | High/Med/Low | [what they might do] | [our counter] |

## Sources
1. [URL] — G2 reviews — [date accessed]
2. [URL] — Crunchbase funding history — [date accessed]
```

---

## Market Sizing Table

```markdown
# Market Sizing: [Market Name]

**Range:** $Xm–$Xm TAM / $Xm–$Xm SAM / $Xm–$Xm SOM (Year X)

## Top-Down Estimate
| Source | Market estimate | Year | Notes |
|--------|----------------|------|-------|
| [Analyst/report] | $Xbn | 202X | [caveats] |
| [Analyst/report] | $Xbn | 202X | [caveats] |
| **Midpoint used:** | **$Xbn** | | |

Segment applied to reach SAM: [X% of total market = our addressable segment]
Rationale: [why this segmentation is correct]

## Bottom-Up Estimate
| Assumption | Value | Source | Confidence |
|-----------|-------|--------|-----------|
| Total ICP companies (global) | X,000 | [LinkedIn filters / SIC data] | Medium |
| % that are serviceable today | X% | [rationale] | Low estimate |
| Average annual contract value | $X,000 | [benchmark / our pricing] | Medium |
| **SAM (bottom-up)** | **$Xm** | | |

## Reconciliation
Top-down SAM: $Xm | Bottom-up SAM: $Xm | **Ratio: Xm**

[If >5× apart: explain the gap and which is more likely correct]
[If close: treat as validation; use the midpoint]

## SOM (Year 1–3 Realistic)
| Year | Assumption | Revenue |
|------|-----------|---------|
| Y1 | X customers × $X ACV | $Xm |
| Y2 | X customers × $X ACV | $Xm |
| Y3 | X customers × $X ACV | $Xm |

Key assumptions: [win rate, sales cycle length, team capacity]

## Growth Rate
Market CAGR: X% [source, year] — implies $Xbn by 20XX

## Sources
1. [URL] — [analyst firm] — [year] — [Tier 2]
```

---

## Technology / Vendor Assessment

```markdown
# Technology Assessment: [Technology / Vendor]

**Verdict:** ✅ Adopt / ⚠️ Evaluate further / ❌ Pass / 🔨 Build instead

## What It Does
[2–3 sentences: what problem it solves, how it works at a high level, what it is not]

## Adoption Signal
| Signal | Value | Source |
|--------|-------|--------|
| GitHub stars | X,XXX | [link, date] |
| Downloads / month | X,XXX | [npm/PyPI, date] |
| Active job postings using it | XXX | [LinkedIn, date] |
| Stack Overflow questions (30d) | XXX | [date] |
| Notable production users | [list] | [source] |

## Trade-offs
| Aspect | Assessment | Evidence |
|--------|-----------|---------|
| Performance | [✅/⚠️/❌] | [benchmark or real-world report] |
| Scalability | [✅/⚠️/❌] | [evidence] |
| Developer experience | [✅/⚠️/❌] | [community sentiment source] |
| Documentation quality | [✅/⚠️/❌] | [direct assessment] |
| Community / support | [✅/⚠️/❌] | [GitHub issues, Discord, etc.] |

## Integration Complexity
**Estimate:** X–X engineer-weeks to integrate
**Evidence:** [migration post, community discussion, or direct assessment]
**Key risks:** [what's hard, what breaks, what requires custom work]

## Lock-in Assessment
- **Data portability:** [can you export? in what format?]
- **Switching cost:** [what leaving looks like]
- **Vendor health:** [funding, headcount trend, customer base]
- **Open source:** [license, fork risk]

## Compliance / Security
- SOC 2: [✅ Type II / ⚠️ Type I / ❌ None / ❓ Unknown]
- GDPR: [✅ Compliant / ⚠️ Partial / ❓ Unknown]
- Known CVEs: [link or "none found"]
- Data residency: [regions / controls available]

## Build vs. Buy Signal
[If integration > 2 sprints or lock-in risk is high: make the build case explicit]
Build cost estimate: X engineer-months
Maintenance cost: ongoing
Recommendation on build vs. buy: [explicit]

## Sources
1. [URL] — [type] — [date]
```

---

## Thesis Pressure-Test

```markdown
# Thesis Pressure-Test: [Thesis Statement]

**Verdict:** ✅ Thesis holds / ⚠️ Holds with caveats / ❌ Thesis has a fatal flaw

## The Thesis (As Stated)
[Exact statement of what is being tested: "X is a good market to enter / investment to make / product to build because Y"]

## Who Tried This Before
| Company | What they tried | What happened | Why it failed |
|---------|----------------|---------------|---------------|
| [co] | [description] | [outcome] | [root cause — not "bad execution"] |

## Structural Headwinds
[Forces that would make this hard regardless of execution quality]
- [Headwind 1]: [evidence it exists] — [how material]
- [Headwind 2]: [evidence] — [how material]

## Key Assumptions Required for the Thesis to Work
| Assumption | Likelihood | Evidence | Kill switch? |
|-----------|-----------|---------|-------------|
| [Assumption 1] | High/Med/Low | [source] | [what would invalidate] |
| [Assumption 2] | High/Med/Low | [source] | [what would invalidate] |

## The Bear Case (Steelmanned)
[Write the strongest possible version of why this is wrong. Don't strawman it.]

[Bear case paragraph — 100–200 words, the most compelling version of the argument against]

## Rebuttal
[Address the bear case directly. If you can't, say so.]

## Competitive Response Risk
[Who could copy this if it works, and how quickly?]
[What moat exists against that response?]

## Verdict
[Thesis holds / has a fatal flaw / holds conditionally on X being true]
[Explicit: "I would / would not [action] because [1–2 reasons]"]
[If "would not": what would need to change to revisit]

## Sources
1. [URL] — [type] — [date]
```
