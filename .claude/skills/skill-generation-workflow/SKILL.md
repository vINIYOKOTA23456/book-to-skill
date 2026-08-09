---
name: skill-generation-workflow
description: "Complete step-by-step workflow for converting books, documents, and knowledge bases into agent skills following the book-to-skill pipeline. Use this skill when you want to understand the full 10-step process (Steps 0-10), know what happens at each stage, understand decision points (analyze-only vs full conversion vs update/fold-in), or need guidance on structuring extracted knowledge into a skill. Covers validation, extraction, analysis, chapter summarization, supporting files, and final assembly."
---

<!-- argument-hint: [step-number, mode-name, decision-point, or complete-workflow-guide] -->

# Skill Generation Workflow — Full Pipeline

The book-to-skill converter transforms documents into structured agent skills through a 10-step process. This skill guides you through each step, explains decision points, and shows what outputs are produced.

---

## Four Modes of Operation

Choose your path based on what you need:

### Mode 1: Full Conversion (Default)
**When:** User provides document path(s), no special instructions
**Steps:** 0–10 (all steps)
**Output:** Complete skill with SKILL.md, chapters/, glossary, patterns, cheatsheet
**Cost:** ~$0.50–$2 per book (one-time)

### Mode 2: Analyze Only
**When:** User says "analyze", "just extract", or "I want to review before generating"
**Steps:** 0–3 only
**Output:** Structured extraction report (frameworks, principles, techniques found)
**Cost:** ~$0.10–$0.30 per book (review phase)

### Mode 3: Generate from Prior Analysis
**When:** User has existing analysis notes or previously ran analyze-only
**Steps:** Skip 0–3, use provided analysis as input, run 4–10
**Output:** Skill files from the provided analysis
**Cost:** ~$0.40–$1.70 per book (generation phase)

### Mode 4: Update / Fold-in (Existing Skill)
**When:** User provides new source(s) and indicates they want to update an existing skill
**Steps:** 0, 1, 1.5, 2, then skip 3–7, jump to **Update/Fold-in Workflow**
**Output:** Updated existing skill with new chapters, merged glossaries/patterns
**Cost:** Proportional to new content added (not re-processing the whole skill)

---

## Step 0 — Out-of-Scope Check

**What happens:** Verify the user provided at least one valid input.

**Decision tree:**
- No arguments provided → Stop. Respond with usage message
- Arguments provided → Parse them, proceed to Step 1

**Parsing rules:**
- If the last argument is not a file/folder/glob that exists, and it looks like a skill slug (e.g. `lowercase-hyphens`), treat it as `SKILL_NAME`
- All other arguments are `INPUT_PATHS`
- If any input path is an existing skill directory (contains `SKILL.md` + `chapters/` subfolder) → Flag as **Update/Fold-in** operation (Mode 4)

**Output:** Parsed `INPUT_PATHS` and `SKILL_NAME` (if provided)

---

## Step 1 — Validate Input

**What happens:** Verify at least one supported file is found.

**Supported formats:** `.pdf`, `.epub`, `.docx`, `.txt`, `.md`, `.markdown`, `.rst`, `.adoc`, `.html`, `.htm`, `.rtf`, `.mobi`, `.azw`, `.azw3`

**For directories and globs:** Expand to find matching files.

**Decision:**
- At least one file found → Proceed to Step 1.5
- No files found → Stop with error message

**Output:** Resolved list of files to process

---

## Step 1.5 — Identify Content Type

**What happens:** Ask the user about content structure.

**User is asked:**
> "What kind of content do these sources have?
> 1. **Technical** — code blocks, tables, formulas, diagrams
> 2. **Text-heavy** — mostly prose, few tables
> 3. **Not sure** — use fast method"

**Answer determines `BOOK_TYPE`:**
- Option 1 → `BOOK_TYPE=technical` → Use Docling
- Option 2 → `BOOK_TYPE=text` → Use pdftotext/pypdf/pdfminer
- Option 3 → `BOOK_TYPE=text` → Default to fast method

**Output:** `BOOK_TYPE` setting for Step 2

---

## Step 2 — Extract Text

**What happens:** Run extraction script based on `BOOK_TYPE`.

**Command:** (simplified)
```bash
python3 scripts/extract.py INPUT_PATHS --mode <BOOK_TYPE> --install-missing ask
```

**Outputs created in temp work directory:**
1. `full_text.txt` — all sources merged with clear source markers
2. `metadata.json` — size stats (pages, words, tokens) + per-source breakdown

**Extraction is format-specific:**
- PDF: pdftotext (text mode) or Docling (technical mode)
- EPUB: ebooklib or stdlib zipfile
- DOCX: python-docx or stdlib ZIP/XML
- HTML: BeautifulSoup or stdlib
- Text/Markdown/RST/AsciiDoc: direct read
- RTF: striprtf or regex
- MOBI/AZW: Calibre ebook-convert

**Output:** `full_text.txt` (extracted text) + `metadata.json` (stats)

---

## Step 2.5 — Pre-flight Cost Estimate

**What happens:** Show the user extraction stats and cost estimate before proceeding.

**Display:**
```
📖 Sources detected: X source(s)
  - filename.pdf (PDF, 244 pages)
  - filename.epub (EPUB)
  ...

📄 Combined: ~244 pages | ~89,000 words | ~119K tokens

💰 Estimated generation cost:
   Input + prompts: ~155K tokens
   Output (chapters + supporting files): ~42K tokens
   Total: ~197K tokens
   
   Cost: multiply by your model's rates (e.g., Claude Sonnet 5)
   
⏱  Estimated time: ~10–15 minutes

📁 Files to be generated:
   SKILL.md + chapter files + glossary + patterns + cheatsheet

➡  Proceed? (or type "analyze only" to preview first)
```

**Decision:**
- User says "analyze only" → Switch to Mode 2 (Analyze Only)
- User confirms → Proceed to Step 3

**Output:** Cost estimate displayed, user confirmation received

---

## Step 3 — Analyze Book Structure

**What happens:** Read first 8,000 characters of `full_text.txt` to identify structure.

**Identifies:**
- Book title and author(s)
- Chapter structure (looks for "Chapter N", "Part I", numbered headings, ToC)
- Core themes and subject domain
- Approximate chapter count

**For "Analyze Only" mode (Mode 2):**
- Produce extraction report now and stop
- Report structure: frameworks, principles, techniques, anti-patterns, suggested skill name, chapters detected
- User can review before committing to generation

**For full conversion (Mode 1):** Store findings and proceed to Step 4

**Output:** Parsed book structure + (if Mode 2) extraction report

---

## Step 4 — Ask Purpose (Full Conversion only)

**What happens:** Ask how the skill will be used (skipped in Modes 2–3).

**User is asked:**
> "What should this skill help you do?
> 1. Apply the author's frameworks while working
> 2. Think with the author's mental models
> 3. Reference specific chapters and concepts
> 4. All of the above"

**Derive `DEPTH` from answer:**
- Answer is **only** option 3 (reference) → `DEPTH=reference` (lean chapters, 800–1,200 tokens)
- Answer includes option 1, 2, or 4 → `DEPTH=study` (deeper chapters, 1,000–3,000 tokens with worked examples)

**This informs the per-chapter budget in Step 7 (no separate "study vs reference" question is asked).**

**Output:** `DEPTH` setting for Step 7

---

## Step 5 — Determine Skill Name & Location

**What happens:** Choose skill slug and filesystem location.

**Skill name logic:**
- If `SKILL_NAME` was provided in Step 0 → Use it
- Otherwise → Propose two options and let user choose:
  - **By author-concept:** `{author-lastname}-{core-concept}` (e.g. `cialdini-influence`)
  - **By title:** lowercase-hyphens from book title (e.g. `designing-data-intensive-apps`)

**Skill location logic:**
Probe user's filesystem for existing skill homes. Prefer based on **host agent**:

| Host | Personal root (in order) | Project root |
|------|--------------------------|--------------|
| GitHub Copilot CLI | `~/.copilot/skills` → `~/.agents/skills` | `.github/skills` → `.claude/skills` → `.agents/skills` |
| Amp | `~/.agents/skills` → `~/.config/agents/skills` → `~/.config/amp/skills` | `.agents/skills` |
| Claude Code | `~/.claude/skills` | `.claude/skills` |

**Selection rules:**
1. If exactly one candidate root exists on disk → Use it
2. If none exist → Ask user which to create
3. If multiple exist → Ask user to choose (remember for session)
4. If can't identify host → Ask "Which agent? (Copilot CLI, Amp, or Claude Code?)"

**Check for collisions:**
If `$SKILLS_HOME/<skill_name>/` already exists:
- Offer: **Update/Fold-in** (Mode 4), **Overwrite**, or **Rename** (append `-2`)

**Output:** `SKILL_NAME` + `SKILLS_HOME` path

---

## Step 6 — Create Skill Directory Structure

**What happens:** Create the skill's filesystem structure.

**Command:**
```bash
mkdir -p "$SKILLS_HOME/<skill_name>/chapters"
```

**Result:**
```
$SKILLS_HOME/<skill_name>/
├── SKILL.md
├── chapters/
│   ├── ch01-<slug>.md
│   ├── ch02-<slug>.md
│   └── ...
├── glossary.md
├── patterns.md
└── cheatsheet.md
```

**Output:** Empty skill directory ready for files

---

## Step 7 — Generate Chapter Summaries

**What happens:** Create one `ch<NN>-<slug>.md` file per chapter.

**Token budget (adaptive by `BOOK_TYPE` + `DEPTH`):**

| | `DEPTH=reference` | `DEPTH=study` |
|---|---|---|
| `BOOK_TYPE=text` | 800–1,200 | 1,000–1,800 |
| `BOOK_TYPE=technical` | 1,200–1,800 | 2,000–3,000 |

**Chapter structure** (template):

```markdown
# Chapter N: <Full Title>

## Core Idea
<1–2 sentences: the single most important thing>

## Frameworks Introduced
- **<Name>**: exact formulation
  - When to use: specific situation
  - How: steps or criteria

## Key Concepts
<5–10 most important terms>

## Mental Models
<2–4 frameworks. Write as "Use X when Y">

## Anti-patterns
- **<What to avoid>**: why it fails

## Code Examples *(technical books only — omit if BOOK_TYPE=text)*
<language>
key example from chapter
</language>

## Reference Tables *(technical books only)*
<markdown tables copied/reconstructed from chapter>

## Worked Example *(DEPTH=study only — omit for DEPTH=reference)*
<One concrete example: sample document, dialogue, template, decision end-to-end>

## Key Takeaways
1. <Actionable insight>
2. <Actionable insight>
3. <Actionable insight>

## Connects To
- **Ch N**: why this chapter relates
- **<Concept>**: external concept it connects with
```

**Quality rules:**
- Extract structure, not summaries
- Preserve author's exact naming
- Density over completeness
- Practitioner voice ("Use X when Y")
- Never copy raw passages — always synthesize
- Chapter files are on-demand (don't count against budget until loaded)

**Output:** `chapters/ch01-<slug>.md`, `ch02-<slug>.md`, etc.

---

## Step 8 — Generate Supporting Files

### glossary.md
- Every significant term, alphabetically sorted
- Format: `**Term** — definition (Ch N)`
- Max 1,500 tokens

### patterns.md
- All techniques, algorithms, design patterns from the book
- Format: `## Pattern Name\n**When to use**: ...\n**How**: ...\n**Trade-offs**: ...`
- Max 2,000 tokens

### cheatsheet.md
**Most differentiated layer.** Captures the author's judgment: decisions they'd make and why.

Priority order:
1. **Decision rules** — "When X, do Y, because Z"
2. **Decision trees/flowcharts** — for multi-branch choices
3. **Trade-off matrices** — options scored on dimensions the author cares about
4. **Thresholds & defaults** — specific numbers and rules of thumb
5. **Tells & smells** — fast heuristics for recognizing a situation

Format: Compact tables and decision rules; the content you'd want on one printed page kept beside you while working.
Max 1,200 tokens.

**Output:** `glossary.md`, `patterns.md`, `cheatsheet.md`

---

## Step 9 — Generate Master SKILL.md

**What happens:** Create the main skill file that agents load.

**Critical:** Keep SKILL.md body under 4,000 tokens (compaction truncates from END).

**Structure:**

```markdown
---
name: <skill_name>
description: "Knowledge base from \"<Title>\" by <Author>. Use when applying <author>'s frameworks for <topics>, studying the book, or referencing concepts."
---

# <Full Title>
**Author**: <Author> | **Pages**: ~<N> | **Chapters**: <N> | **Generated**: <YYYY-MM-DD>

## How to Use This Skill
- Without arguments → load core frameworks
- With a topic → ask about specific topic; I find and read the relevant chapter
- With chapter → ask for `ch05`; I load that specific chapter
- Browse → ask "what chapters do you have?"

---

## Core Frameworks & Mental Models
<~2,000 tokens of the most critical frameworks and insights>

---

## Chapter Index
| # | Title | Key Frameworks |
| [ch01](chapters/ch01-<slug>.md) | <Title> | <framework1>, <framework2> |
...

## Topic Index
- **<Term>** → ch<N>[, ch<N>]
...

## Supporting Files
- [glossary.md](glossary.md)
- [patterns.md](patterns.md)
- [cheatsheet.md](cheatsheet.md)

---

## Scope & Limits
This skill covers the book content only. For hands-on implementation, combine with project-specific tools.
```

**Key principles:**
- Front-load most important content (read first 2,000 tokens)
- Chapter index tells agent which file to read for a topic
- Topic index is critical for navigation
- Preserve author's exact framework names and formulations

**Output:** Complete `SKILL.md` ready for use

---

## Step 9.5 — Scan Generated Skill

**What happens:** Security/quality scan before loading.

**Command:**
```bash
python3 tools/scan_generated_skill.py "$SKILLS_HOME/<skill_name>"
```

**Checks for:**
- Accidental raw passages (should be synthesized)
- Malformed markdown
- Missing chapter references
- Other quality issues

**Decision:**
- Scanner passes → Proceed to Step 10
- Scanner fails → Ask human to review findings before proceeding

**Output:** Scan report (pass/fail)

---

## Step 10 — Cleanup & Report

**What happens:** Clean up temporary work files and report success.

**Cleanup:**
```bash
rm -rf "$BOOK_SKILL_WORKDIR"  # /tmp/book_skill_work by default
```

**Report to user:**

```
✅ Skill created: $SKILLS_HOME/<skill_name>/

📚 Book: <Title> — <Author>
📄 Pages: ~<N> | Chapters: <N>

Files generated:
  SKILL.md              (~4K tokens, core frameworks + index)
  chapters/             (~X chapters, ~X tokens each, ~X total)
  glossary.md           (~1.5K tokens)
  patterns.md           (~2K tokens)
  cheatsheet.md         (~1.2K tokens)
  ────────────────────────────────────────
  Total: ~X tokens (loaded on-demand)

Usage:
  /<skill_name>                  → load core frameworks
  /<skill_name> about <topic>    → find and explain topic
  /<skill_name> ch<N>            → dive into chapter N

Reload (if needed):
  GitHub Copilot CLI: /skills reload
  Claude Code:        restart the session
  Amp:                restart the session
```

**Output:** Complete skill, ready to use

---

## Update / Fold-in Workflow (Mode 4)

**When user provides new source(s) + indicates update:**

### 1. Read Existing Skill Structure
- Parse existing SKILL.md (Chapter Index, Topic Index, Core Frameworks)
- List `chapters/` files to find highest chapter number
- Read `glossary.md`, `patterns.md`, `cheatsheet.md`

### 2. Analyze New Content
- Identify updates (new versions of existing chapters) vs additions (new chapters)
- New chapters start numbering after the highest existing chapter

### 3. Generate/Update Chapter Files
- For updated chapters: merge new details into existing files
- For new chapters: create new `ch<NN>-*.md` files

### 4. Merge Supporting Files
- Combine glossaries (alphabetical, de-duplicate, append chapter refs)
- Merge patterns (consistent formatting, keep total under 2,500 tokens)
- Integrate new cheatsheet rules/thresholds

### 5. Re-generate Master SKILL.md
- Update metadata (chapter count, page count, Generated date)
- Fold new mental models into Core Frameworks
- Update Chapter Index (add new chapters)
- Merge Topic Index (append new chapter links)

### 6. Scan & Cleanup
- Run Step 9.5 scan
- Clean up temp files
- Report update summary

**Output:** Updated skill with new chapters seamlessly integrated

---

## Decision Flowchart

```
User provides document path(s)
    ↓
Step 0: Valid input?
    ├─ No  → Error, stop
    └─ Yes → Continue
    ↓
Is this an existing skill folder/slug?
    ├─ Yes → Mode 4 (Update/Fold-in)
    └─ No  → Continue
    ↓
Step 1.5: Ask content type (technical/text)
    ↓
Step 2: Extract text
    ↓
Step 2.5: Show cost estimate
    ↓
User says "proceed" or "analyze only"?
    ├─ "analyze only" → Mode 2: Stop at Step 3 report
    └─ "proceed"      → Mode 1: Continue to Step 10
    ↓
Step 3: Analyze structure
Step 4: Ask purpose (reference vs study)
Step 5: Skill name + location
Steps 6–10: Generate skill files
    ↓
✅ Complete
```

---

## How to Use This Skill

Ask for help with:
- **Understanding a specific step** — "/skill-generation-workflow step 7 what's the token budget?"
- **Choosing a mode** — "Should I analyze-only or do full conversion?"
- **Decision points** — "When is a chapter reference-depth vs study-depth?"
- **Cost estimation** — "How much will it cost to generate a skill from this book?"
- **Update/fold-in workflow** — "How do I add new chapters to an existing skill?"
- **Complete workflow walkthrough** — "/skill-generation-workflow complete-workflow-guide"

Example queries:
- `/skill-generation-workflow step-4-purpose`
- `/skill-generation-workflow update-fold-in`
- `/skill-generation-workflow What's the difference between mode 1 and mode 2?`
