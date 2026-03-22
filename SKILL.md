---
name: article-writing
description: Write articles, guides, blog posts, tutorials, newsletter issues, LinkedIn posts, and other long-form content in a distinctive voice derived from supplied examples or brand guidance. Always activate when the user wants polished written content longer than a paragraph — especially when voice consistency, structure, and credibility matter. Also activate when turning notes, transcripts, or bullet-point research into polished prose, and when the user asks to "write", "draft", "rewrite", or "clean up" any piece of long-form content, even if they don't call it an article.
---

# Article Writing

Write long-form content that sounds like a real person or brand, not generic AI output. Every piece should be concrete, earned, and platform-appropriate.

## Workflow

When this skill activates:

1. **Clarify before writing** — identify audience, purpose, platform, and approximate length. One question if anything is ambiguous; don't stall.
2. **Capture the voice** if a specific voice is required — collect examples, extract characteristics, and produce a Voice Profile (see below) before drafting.
3. **Process raw material** if the user provides notes, transcripts, or research — extract the signal, identify the argument, then draft.
4. **Build a skeletal outline** with one clear purpose per section. Show it and get a quick confirm if the piece is substantial (>800 words).
5. **Draft** — start each section with concrete evidence. Expand only where the next sentence earns its place.
6. **Run the quality gate** before delivering. Cut what fails it.
7. **Offer a revision pass** — one targeted question after delivery to catch misses.

---

## Core Rules

These apply to every piece regardless of format or voice:

1. **Lead with the concrete thing** — example, output, number, anecdote, screenshot description, or code block. Explain after, not before.
2. **Short sentences are a default** — pad only for deliberate rhythm.
3. **Use specific numbers** when available and sourced. "37%" beats "most".
4. **Never invent facts** — no biographical claims, company metrics, customer evidence, or quotes unless provided.
5. **Every section must add new information** — if it only restates what came before, cut it.
6. **Earn the opinion** — any claim stronger than neutral must be supported by evidence in the same paragraph.

---

## Voice Capture

### What to Collect

Before writing in a specific voice, ask for one or more of:
- published articles or blog posts
- newsletter issues
- X / LinkedIn posts
- internal memos or docs
- a short style guide or "write like X" brief

Three to five examples are sufficient. More is not better — look for patterns, not every quirk.

### Voice Profile Template

After reviewing examples, produce and share a Voice Profile. This documents the extracted voice and prevents drift across turns:

```
VOICE PROFILE: [Person / Brand Name]

Sentence length:   [short / medium / long / mixed — with pattern if mixed]
Register:          [formal / conversational / sharp / academic / warm]
First person:      [heavy / light / avoided]
Humor:             [none / dry / self-deprecating / frequent]
Opinion tolerance: [neutral / willing to argue / contrarian by default]

Signature patterns:
- [specific device — e.g., "opens with a counterintuitive claim"]
- [e.g., "uses em dashes instead of parentheses"]
- [e.g., "rhetorical questions to end sections"]
- [e.g., "short fragments for emphasis"]

Formatting habits:
- Headers:  [yes/no/minimal]
- Bullets:  [frequent / sparingly / avoided]
- Bold:     [for key terms / for emphasis / avoided]
- Pull quotes or callouts: [yes/no]

Avoid:
- [things this voice never does — e.g., "no hedging language"]
- [e.g., "never opens with a question"]
```

Share the Voice Profile with the user before drafting. Confirm or adjust. Then hold to it.

**Default voice** (when no examples are provided): direct, operator-style — concrete, practical, low on hype, willing to state an opinion plainly.

---

## Working with Raw Material

When the user provides notes, transcripts, bullet points, or research:

1. **Find the argument** — what is the single thing this piece should leave the reader believing or able to do? State it explicitly before drafting.
2. **Strip scaffolding** — remove meta-commentary ("here I want to talk about..."), repeated points, and anything that was a thinking-out-loud note rather than a content decision.
3. **Sequence by importance, not by how the material arrived** — the order notes were taken in is rarely the order they should be read.
4. **Flag gaps** — identify where the raw material is thin and the piece needs either more research or a scoped-down claim.
5. **Preserve the raw specifics** — real names, real numbers, real quotes from the material are more valuable than synthesized generalities.

---

## Structure by Content Type

### Technical Guides and Tutorials

- Open with what the reader gets at the end — the concrete output, not the topic
- Use code, terminal output, or screenshots in every major section
- Each step should produce something observable
- Close with concrete takeaways and the most common failure modes
- Length: 800–2500 words depending on complexity; err toward shorter if the task is well-scoped

### Essays and Opinion Pieces

- Open with tension, contradiction, or a sharp observation — not background
- One argument thread per section; sidebars go in parentheses or footnotes
- The opinion must be stated explicitly, not implied — readers should be able to quote it
- Use examples that earn the opinion, not examples that just illustrate it
- Close by raising the stakes, not summarizing
- Length: 600–1800 words; longer only if the argument genuinely needs the space

### Newsletter Issues

- First screen (above the fold) must reward opening — lead with the most interesting thing
- Mix insight with update; avoid diary filler
- Clear section labels for skim navigation
- Each section should stand alone if forwarded or quoted
- Length: 300–900 words for a single-topic issue; multi-section newsletters can be longer with clear navigation

### LinkedIn Posts

- First line must work as a hook with the rest collapsed — most readers never expand
- No "In this post I will tell you about..." — get to the point in line 1
- Short paragraphs; one idea per paragraph; single-line breaks create visual rhythm
- Use a hook → insight → evidence → implication structure
- End with a specific observation, not "what do you think?"
- Length: 150–400 words for most posts; 400–700 for long-form LinkedIn articles

### Launch Posts and Announcements

- Lead with what changed or what the reader can now do — not the company story
- Specific > vague: "ships today" not "coming soon", "saves 3 hours/week" not "saves time"
- One primary call to action; avoid four different links
- Length: 200–500 words

---

## Headlines and Titles

The headline is the highest-leverage line in the piece. Write 3–5 options for every piece and select the best:

**Patterns that perform:**
- Specific number: "7 Things We Learned Rebuilding Our Auth System"
- Direct promise: "How to Cut Your Docker Build Time by 60%"
- Counterintuitive claim: "Why We Stopped Using Microservices"
- Named problem: "The Hidden Cost of Premature Abstraction"
- Before/after: "From 12-Hour Deploys to 40 Minutes"

**Avoid:**
- Questions (weak, often hedging the actual claim)
- "The Ultimate Guide to..." (overused, unearned)
- Titles that describe the article's topic without stating its argument
- Anything that would work equally well for 20 other articles

Always confirm the title is either: a specific claim, a specific promise, or a specific question the reader urgently wants answered.

---

## Platform Formatting

| Platform | Headers | Max line length | Code blocks | Images |
|---|---|---|---|---|
| Company blog | Yes (H2/H3) | ~80 chars | Yes | Yes |
| Medium / Substack | Minimal | Paragraph-width | Yes | Yes |
| LinkedIn | No | Short | No (use backtick sparingly) | Optional |
| Hacker News | None | Plain text | Indent 2 spaces | None |
| GitHub README | Yes (H2/H3) | ~100 chars | Yes | Inline only |
| Email newsletter | None/minimal | ~600px wide | Avoid | Inline |

---

## SEO (When Relevant)

For blog posts and guides where search visibility matters:

1. **Primary keyword** — the phrase readers would search to find this article. State it by the end of the first paragraph, naturally.
2. **Secondary keywords** — 2–3 related phrases; use in H2s and body naturally, not stuffed.
3. **Meta description** — one sentence (150–160 characters), includes primary keyword, describes what the reader gets.
4. **Title tag** — primary keyword near the front if possible without sounding unnatural.
5. **Internal links** — note where other content on the same site should be linked.

SEO should shape word choice, not distort it. A sentence rewritten for a keyword that no longer reads well is a loss.

---

## Banned Patterns

Cut and rewrite any of these without comment:

**Generic openers**
- "In today's rapidly evolving landscape..."
- "As [topic] continues to grow..."
- "Have you ever wondered..."
- "I'm excited to share..."

**Filler transitions**
- "Moreover", "Furthermore", "Additionally", "It is worth noting that"
- "In conclusion", "To summarize", "As we've seen"

**Hype language**
- "game-changer", "revolutionary", "cutting-edge", "best-in-class"
- "seamless", "robust", "holistic", "synergy", "leverage" (as a verb)
- "I believe" / "I think" when followed by a hedge, not an actual belief

**Safety blanket hedging**
- "You might want to consider..."
- "It could be argued that..."
- "There are many factors to consider..."

**Empty credentialism**
- Claiming expertise not backed by provided context
- "As a leader in X" without evidence
- Quotes attributed to unnamed "experts" or "studies"

**Structural tells**
- Three-word section headings that restate the paragraph below them
- Numbered lists where the numbers don't matter
- Bullets that are just the same sentence with commas removed

---

## Quality Gate

Run these before delivering. Fail means rewrite, not minor edit:

- [ ] **Opening**: Does the first sentence earn the second? Cut anything that reads like a warmup.
- [ ] **Headline**: Is it a specific claim, promise, or urgent question — not a topic description?
- [ ] **Voice match**: Read the first paragraph aloud against the Voice Profile. Does it match? If not, identify which rule it breaks.
- [ ] **Evidence**: Every opinion or claim stronger than neutral — is it backed in the same paragraph?
- [ ] **Filler audit**: Ctrl+F for "moreover", "furthermore", "it's worth noting", "game-changer", "seamless". If found, rewrite.
- [ ] **Invented facts**: Any metric, quote, or biographical claim — was it in the provided material? If not, remove it.
- [ ] **Length**: Is the word count appropriate for the platform and scope? Cut aggressively if over.
- [ ] **Platform format**: Headers, line breaks, code blocks — correct for the destination?
- [ ] **Final line**: Does it close with weight, or just trail off? The last sentence should land.
