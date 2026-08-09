---
name: skill-quality-rules
description: "Quality standards and token budgets for generating agent skills from books and documents. Use this skill when building chapter summaries, glossaries, pattern lists, or cheatsheets; need to understand token budgets (800–3000 tokens per chapter depending on depth); want guidance on writing practitioner-focused content; or need to avoid common quality pitfalls like copying raw text, padding to hit token targets, or losing author's precision in framework names."
---

<!-- argument-hint: [budget, chapter-structure, glossary-guide, or quality-rule-number] -->

# Skill Generation Quality Rules & Standards

Seven core quality rules guide skill generation. These aren't arbitrary — they ensure skills are densely informative, precisely grounded in the source, and optimized for agent recall and application.

---

## Quality Rule #1: Extract Structure, Not Summaries

### The Principle

A skill captures the author's **frameworks, techniques, and mental models** — not a report of what the book says.

**Wrong:** "Chapter 5 discusses the importance of clear communication in teams."  
**Right:** "**Clear Communication Framework**: establish shared vocabulary, confirm mutual understanding before proceeding, allocate decision authority upfront."

**Wrong:** "The book explains that customer discovery is essential."  
**Right:** "**Customer Discovery Loop**: schedule 5–10 customer interviews per week, focus on *problems* not solutions, ask 'what does this person actually do' before proposing fixes."

### Why This Matters

- Agents use skills to **reason and decide**, not to retrieve paragraphs
- Raw summaries force the agent to re-synthesize at query time (expensive, lossy)
- Structure + naming lets the agent apply frameworks directly

### How to Apply It

When writing a chapter or skill:
1. Identify **named frameworks** (exact author names)
2. Identify **decision rules** (when X, do Y)
3. Identify **mental models** (think of X as Y)
4. Identify **anti-patterns** (what to avoid and why)

Only include sentences that introduce one of these. Cut descriptive passages. If the author spent 2 pages explaining why something works, capture the principle in 1 sentence, not the whole explanation.

---

## Quality Rule #2: Preserve the Author's Precision

### The Principle

Framework names **matter**. They're shortcuts for deep ideas.

**Example:** "The 5 Whys" ≠ "ask why multiple times"
- The *specific number* (5, not 7 or 10) teaches pacing
- The *order* (root-cause analysis depth) is part of the technique
- Miscellaneous "why-asking" loses the methodology

**Example:** "Westrum organizational topology" (names the model + author)  
vs "communication styles matter" (generic, loses precision)

### Why This Matters

Agents use exact names to cross-reference. If the cheatsheet says "Apply Westrum topology to diagnose org health," an agent can jump to the glossary and find the exact model definition. Generic names ("communication styles") don't link to structure.

### How to Apply It

- **Copy exact framework names from the text** (with author if the author names it that way)
- **Preserve exact numbering** ("5 Whys", not "five whys" or "multiple whys")
- **Quote the author's definition verbatim** if it's 1 sentence
- **Attribute clearly** ("The author calls this...", "Known as...")
- **If the author doesn't name something**, you can name it (e.g. "the Build-Measure-Learn cycle" for Lean Startup ideas), but note it's derived

---

## Quality Rule #3: Density Over Completeness

### The Principle

A 1,000-token summary beats a 10,000-token excerpt. Write for re-reading, not first reading.

**Wrong approach:** Copy all 3 pages the author wrote on topic X, then let the agent summarize.  
**Right approach:** Distill to 1 paragraph: "Use X when Y, because Z. Trade-off: slow but certain."

### Why This Matters

- Skills load on-demand. Density keeps per-chapter cost low
- Agents retrieve better from structured summaries than raw text (lost-in-the-middle problem)
- Token budgets are **per-file budgets**, not limits on content — if a chapter is thin, let it be thin

### How to Apply It

- Every sentence should teach something, not repeat
- Cut examples that illustrate the same point twice
- Omit stage-setting and motivation (the author probably spent pages there; capture the essence in 1 sentence)
- Organize by **structure** (frameworks, mental models, anti-patterns) not by reading order
- If you're ever padding to hit a token target, **stop and cut instead**

---

## Quality Rule #4: Practitioner Voice

### The Principle

Write as if the reader is about to act, not about to read a book report.

**Wrong:** "The author explains that code reviews improve team knowledge sharing."  
**Right:** "Use code reviews when: onboarding new team members, sharing architectural knowledge, catching edge cases. Expect 10–15 min per review."

**Wrong:** "Chapter 7 discusses pricing strategy."  
**Right:** "**Value-Based Pricing Decision Tree**: If customer has high switching cost → premium tier. If commodity → cost-based. If unique value prop but low awareness → penetration pricing first."

### Why This Matters

Agents consulting a skill are in the middle of work. They need **immediately actionable guidance**, not passive information.

### How to Apply It

- Start with "Use X when...", not "The book discusses X"
- Include specific numbers (thresholds, ratios, timelines) from the text
- Structure as decision rules: "When [situation], [action], because [tradeoff]"
- Show the "how" (steps, sequence, checklist) alongside the "what"

---

## Quality Rule #5: Front-Load SKILL.md (Compaction Truncates from END)

### The Principle

The SKILL.md body is limited to 4,000 tokens. **The most important content goes FIRST.**

Agents see SKILL.md before chapter files. If SKILL.md is truncated (and long ones are), the content that matters most must be visible.

**Example organization:**

```markdown
## Core Frameworks & Mental Models
(~2,000 tokens of the author's top 5–7 named frameworks)
(Agents use these for most queries)

## Chapter Index
(tells agent where to find deep dives)

## Topic Index
(alphabetical navigation)

## Supporting Files
(glossary, patterns, cheatsheet live here, loaded on-demand)
```

If you have 10 frameworks and space for 6, **include the 6 that the agent will use most often for decisions**.

### Why This Matters

- Short context windows truncate long documents
- On-demand chapter files stay loaded, so deep content isn't lost
- Front-loading ensures the **resident core** is always available

### How to Apply It

- Count tokens as you write SKILL.md
- Put the top frameworks in the body
- Put the rest in cheatsheet.md (loaded separately)
- Use the Chapter Index + Topic Index to guide agents to chapter files
- Test: if SKILL.md were cut to 2,000 tokens, would the reader still have what they need for most queries? If not, re-prioritize.

---

## Quality Rule #6: Chapter Files Are On-Demand (Don't Count Against Budget Until Loaded)

### The Principle

A skill has one resident SKILL.md (~4K tokens) + many chapter files (~1–3K tokens each). Agents load chapters only when they ask about a specific topic.

**You never pay for chapters you don't read.**

This is the core economic insight behind book-to-skill. A 400-page book is ~200K tokens. In raw form, it burns 200K tokens on **every query**. In skill form:
- First query: load SKILL.md (~4K) + relevant chapter (~1K) = ~5K tokens
- Second query: same chapter still in history (free), or a different chapter (~5K again)

### Why This Matters

Token budgets are **per-file**, not per-skill. A "thin" chapter (600 tokens) is fine — it only costs 600 tokens when the agent asks about it.

### How to Apply It

- Don't pad chapters to hit a token target
- If a chapter is thin (say, 500 tokens), that's okay — let it be 500 tokens
- If a chapter has 2,000 tokens of actual content and your budget is 1,800, go over — density is the point
- Distribute complexity across chapters: deep topics get their own chapter; thin topics can share or merge

**Example:**
- Chapter 3: Frameworks A, B, C (2,100 tokens) — over budget, but dense, so it's fine
- Chapter 4: Technique X (600 tokens) — under budget, thin, but fine — it's one technique
- Combined skill size is still 4K + 2.1K + 0.6K = 6.7K tokens, loaded in pieces

---

## Quality Rule #7: Never Copy Raw Book Text

### The Principle

**Synthesize everything.** A skill is not an excerpt archive — it's a synthesis of the author's thinking.

**Wrong:** Copy a 3-paragraph explanation the author wrote and paste it.  
**Right:** Distill to its essence: "When X, the author recommends Y because Z."

**Wrong:** Include a full example dialogue or script from the book.  
**Right:** "The author uses this dialogue pattern: [reconstruct the structure in 3–4 lines]."

### Why This Matters

- **Copyright:** Raw copying may infringe. Synthesis is fair use (your notes, your interpretation)
- **Skill quality:** Synthesized content is denser and more actionable than raw text
- **Agent recall:** Models recall structure better than raw excerpts (especially in long files)

### How to Apply It

- Read a passage, close the source, rewrite from memory
- If you must quote, quote **one sentence max** per framework
- Capture the **idea**, not the wording
- Examples: reconstruct in your own structure, not verbatim
- If an example is too good to lose, note the idea and put it in a "Worked Example" section (still rephrased, not copied)

---

## Token Budgets by Content Type and Depth

### Budget Matrix

| Content Type | Reference Depth | Study Depth |
|---|---|---|
| **Text-heavy** (prose, narrative) | 800–1,200 tokens | 1,000–1,800 tokens |
| **Technical** (code, tables, formulas) | 1,200–1,800 tokens | 2,000–3,000 tokens |

### How Budgets Work

- **Budgets are per-chapter targets, not hard caps.** A dense chapter can run 200 tokens over; a thin one can be 200 under
- **Density wins.** A 900-token chapter with structure beats a 1,200-token chapter with padding
- **Technical chapters get more room** because they need code examples and tables (both add bulk legitimately)
- **Study depth adds "Worked Example" sections** (the biggest cost) plus expanded "How" sections

### Example Chapter Breakdown

**Technical, Study Depth (2,500 tokens budget):**
- Core Idea (100 tokens)
- Frameworks Introduced (400 tokens)
- Key Concepts (300 tokens)
- Mental Models (300 tokens)
- Anti-patterns (200 tokens)
- Code Examples (400 tokens)
- Reference Tables (200 tokens)
- Worked Example (500 tokens — this is what makes it "study" depth)
- Key Takeaways (100 tokens)
- Connects To (100 tokens)
**Total: ~2,400 tokens** ✓ (within budget, dense, honest)

**Text-heavy, Reference Depth (1,000 tokens budget):**
- Core Idea (80 tokens)
- Frameworks Introduced (300 tokens)
- Key Concepts (200 tokens)
- Mental Models (200 tokens)
- Anti-patterns (120 tokens)
- Key Takeaways (80 tokens)
- Connects To (40 tokens)
**Total: ~1,020 tokens** ✓ (within budget, no padding, lean)

---

## Supporting Files: Glossary, Patterns, Cheatsheet

### glossary.md

**What:** Every significant term, alphabetically sorted

**Format:**
```
**Term** — definition in 1 sentence (Ch N)
**Another Term** — definition (Ch 5, Ch 12)
```

**Token budget:** ~1,500 tokens max

**Quality check:**
- No definitions longer than 1 sentence
- All terms are actually used in chapters (no orphans)
- Entries are sorted alphabetically
- Chapter references are accurate

### patterns.md

**What:** All concrete techniques, algorithms, design patterns

**Format:**
```markdown
## Pattern Name

**When to use**: specific situation

**How**: step-by-step or criteria

**Trade-offs**: what you gain/lose
```

**Token budget:** ~2,000 tokens max

**Quality check:**
- Patterns are named and structured consistently
- Each pattern has all 3 sections (When, How, Trade-offs)
- No raw book text; all synthesized
- Organized by category if complex (e.g. "Data Patterns", "Social Patterns")

### cheatsheet.md

**What:** Decision rules, decision trees, trade-off matrices, thresholds, tells & smells — everything that captures the author's judgment

**Priority order (what to include first):**

1. **Decision rules** (biggest impact)
   - "When X, do Y, because Z"
   - These let readers act like the author would

2. **Decision trees/flowcharts** (for complex decisions)
   - "If A: path 1. If B: path 2. If C: path 3"
   - Nested bullets or tables

3. **Trade-off matrices**
   - Options (rows) vs. dimensions (columns)
   - Simple scores or descriptions
   - E.g. "Option A: fast but loose. Option B: slow but thorough."

4. **Thresholds & defaults**
   - Specific numbers from the book
   - E.g. "Keep functions under 20 lines", "Alert when error budget < 10%"

5. **Tells & smells** (diagnostic heuristics)
   - "If you see X, you're probably in trouble Y"
   - Fast rules for recognizing situations

**What NOT to include:**
- Bare term→definition (that's the glossary)
- Prose paragraphs (that's the chapters)
- Anything that doesn't help the reader **decide** something

**Token budget:** ~1,200 tokens max

**Quality check:**
- Mostly tables and decision rules; minimal prose
- Everything supports decision-making
- Formatted compactly (one printed page or less when rendered)

---

## Common Pitfalls & How to Avoid Them

### Pitfall #1: Padding to Hit a Token Target

**Symptom:** "The budget is 1,500 tokens but I'm only at 800. I should add more."

**Why it happens:** Confusing token budgets with hard requirements

**Fix:** Token budgets are *targets*, not minimums. **Stop writing when you've said everything.**

### Pitfall #2: Including Too Many Worked Examples

**Symptom:** "The book has 5 great examples. I should include all 5."

**Why it happens:** Wanting to preserve author's best content

**Fix:** Pick **one** worked example per chapter (for study-depth chapters). More than one repeats the same lesson.

### Pitfall #3: Copying Author's Wording Exactly

**Symptom:** Pasting paragraphs from the source into the chapter

**Why it happens:** Fear of losing nuance

**Fix:** Synthesize. Read, close the source, rewrite. If you can't rewrite it, you don't understand it yet.

### Pitfall #4: Losing Author's Precision in Framework Names

**Symptom:** Calling "The OODA Loop" just "decision loop" because it's shorter

**Why it happens:** Trying to simplify or avoid jargon

**Fix:** Keep exact names. Simplify everything else (description, context), but **preserve the name**.

### Pitfall #5: Overstuffed "Mental Models" Section

**Symptom:** 10+ mental models crammed into one chapter

**Why it happens:** Listing every idea the chapter touches

**Fix:** Include only 2–4 **core** mental models per chapter. Others go in the glossary or patterns.

### Pitfall #6: Weak "Connects To" Section

**Symptom:** "Connects To: Chapter 1, Chapter 2, Chapter 3" with no explanation

**Why it happens:** Filling the section without thought

**Fix:** Only list connections you can explain in 1–2 lines. "Why does this chapter relate to X?"

### Pitfall #7: Cheatsheet Full of Descriptive Text

**Symptom:** Cheatsheet reads like chapter summaries

**Why it happens:** Trying to make cheatsheet comprehensive

**Fix:** Cheatsheet is decision-focused only. If it needs explanation, put it in a chapter instead.

---

## Quality Checklist

Before finalizing a skill, review:

- [ ] Every framework name matches the author's exact wording
- [ ] No chapter is padded to hit a token target
- [ ] Every sentence in glossary/patterns/cheatsheet serves a purpose
- [ ] No raw book text pasted (all synthesized)
- [ ] Chapter "Connects To" sections explain why they relate
- [ ] "Worked Example" sections (study-depth only) are reconstructed, not copied
- [ ] "Code Examples" (technical only) preserve exact syntax
- [ ] Practitioner voice used throughout ("Use X when...", not "The book says...")
- [ ] Topic Index in SKILL.md is alphabetical and complete
- [ ] SKILL.md Core Frameworks front-load the most important ideas
- [ ] Token counts are tracked per file (SKILL.md < 4K, chapters within budget, supporting files at limit)

---

## How to Use This Skill

Ask for help with:
- **Understanding a specific quality rule** — "What does 'practitioner voice' mean?"
- **Token budgets** — "Can a technical chapter be 3,500 tokens?"
- **Avoiding copying raw text** — "How do I synthesize an example without losing the idea?"
- **Specific sections** — "What goes in a Worked Example section?"
- **Trade-offs** — "Should I include all the author's frameworks or pick the most important ones?"
- **Common pitfalls** — "How do I avoid padding to hit token targets?"

Example queries:
- `/skill-quality-rules quality-rule-7`
- `/skill-quality-rules practitioner-voice-examples`
- `/skill-quality-rules cheatsheet-section-guide`
- `/skill-quality-rules How do I know if I'm copying too much raw text?`
