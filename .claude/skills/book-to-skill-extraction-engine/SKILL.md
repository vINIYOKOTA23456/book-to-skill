---
name: book-to-skill-extraction-engine
description: "Understanding text extraction from books and documents (PDF, EPUB, DOCX, HTML, Markdown, plain text, RTF, MOBI/AZW). Use this skill when you need to understand how to extract text from different file formats, choose the right extraction tool for your content type (technical vs text-heavy), or troubleshoot extraction quality issues. Covers all parsers used in the book-to-skill pipeline, their trade-offs, and when to use each."
---

<!-- argument-hint: [format, technical-vs-text, extractor-name, or troubleshooting-query] -->

# Text Extraction Engine — Format Guide

The book-to-skill pipeline supports multiple document formats. Each has format-specific extractors, with different speed/quality trade-offs depending on content type (technical vs text-heavy).

---

## Core Concept: Technical vs Text-Heavy

Before extraction, the pipeline asks:

> **Technical** — has code blocks, tables, formulas, diagrams (e.g. programming books, academic papers, architecture guides)
> **Text-heavy** — mostly prose, few or no tables/code (e.g. management, productivity, narrative non-fiction)

This choice determines:
1. Which extractor is used (affects speed and table/code preservation)
2. Per-chapter token budgets
3. Which sections appear in generated skill chapter files

---

## Format-by-Format Guide

### PDF

**Extraction hierarchy** (tries in order, uses first available):

| Tool | Mode | Speed | Table/Code Support | When to Use |
|------|------|-------|-------------------|------------|
| **Docling** | technical | ~1.5s/page | ⭐⭐⭐ Best — preserves markdown tables and code | Technical books with complex formatting |
| **pdftotext** (poppler) | text | ⚡ instant | ❌ None | Text-heavy books, fastest option |
| **pypdf** | text | ⚡ instant | ❌ None | Text-heavy fallback, pure Python |
| **pdfminer.six** | text | ⚡ instant | ❌ None | Text-heavy fallback, most robust |

**User chooses mode (`technical` or `text`) before extraction.** The pipeline then:
- `technical` → runs `docling` for structure-aware extraction
- `text` → tries `pdftotext` first (fastest if installed), falls back to `pypdf`, then `pdfminer`

**Trade-offs:**
- Docling handles tables, code blocks, and formulas as proper markdown — but takes ~1.5s per page (244-page book ≈ 6 minutes)
- pdftotext is instant but strips all structure (tables become plain text, code blocks flatten)
- pypdf and pdfminer are pure Python fallbacks — no system dependencies needed, but extraction quality varies by PDF encoding

**Quality check:** A text-heavy 103-page book extracts the same token count (~27K) with both pdftotext and Docling. If your text-heavy PDF has OCR or scanned pages, pdfminer handles those better than pdftotext.

---

### EPUB

**Extraction hierarchy:**

| Tool | Quality | When to Use |
|------|---------|------------|
| **ebooklib + beautifulsoup4** | ⭐⭐⭐ Best | Preferred — parses EPUB structure cleanly |
| **stdlib `zipfile` + HTML parser** | ⭐⭐ Always available | Zero dependencies, fallback when libraries missing |

**Both work.** ebooklib is cleaner but adds dependencies. stdlib fallback uses only Python standard library.

**Chapter detection:** EPUB chapter segmentation relies on the ebook's internal structure. Well-formed EPUBs (chapters marked with semantic tags like `<nav type="toc">`) extract cleanly. Poorly structured EPUBs (all content in one file) extract as a single chunk — no chapters detected until Step 3 analyzes headings.

---

### DOCX (Word documents)

**Extraction hierarchy:**

| Tool | Quality | When to Use |
|------|---------|------------|
| **python-docx** | ⭐⭐⭐ Best | Preferred — preserves formatting intent |
| **stdlib `zipfile` + XML** | ⭐⭐ Fallback | Zero dependencies, good enough for plain prose |

**Both work.** python-docx parses DOCX's internal document model. Stdlib fallback reads the underlying ZIP/XML directly (DOCX is a zipped XML container).

**Preserves:** Headings, bold/italic (for emphasis detection), lists, but **not** tables (currently strips to plain text). For a DOCX with complex tables, consider exporting to PDF and using Docling instead.

---

### HTML

**Extraction hierarchy:**

| Tool | Quality | When to Use |
|------|---------|------------|
| **beautifulsoup4** | ⭐⭐⭐ Best | Preferred — semantic parsing |
| **stdlib `html.parser`** | ⭐⭐ Fallback | Zero dependencies, handles malformed HTML |

**Both work.** beautifulsoup uses lxml or html5lib for robust parsing. Stdlib fallback works on any HTML, even broken tags.

**Extraction logic:** Skips `<script>`, `<style>`, navigation, and boilerplate. Keeps heading hierarchy for chapter detection.

---

### Plain Text, Markdown, reStructuredText, AsciiDoc

**No dependencies needed.** These formats are already readable text.

**Chapter detection:** The pipeline looks for structural headings:
- **Markdown** — `# Title`, `## Chapter`, `### Section`
- **reStructuredText** — underline-style headings (`====`, `----`)
- **AsciiDoc** — `= Title`, `== Chapter`
- **Plain text** — fallback, searches for "Chapter N" patterns

If a text file has no recognizable headings, it's treated as one big chapter.

---

### RTF (Rich Text Format)

**Extraction tool:** `striprtf` (pure Python) or regex fallback

Removes RTF control codes and preserves text and basic structure. RTF is rarely used for books, but older scanned document compilations may be RTF.

---

### MOBI / AZW / AZW3 (Amazon Kindle formats)

**Extraction tool:** Calibre's `ebook-convert` (external app, not pip)

Requires Calibre to be installed: https://calibre-ebook.com/download

These formats have vendor-specific DRM and structure. Calibre handles the conversion. If `ebook-convert` is not found, the pipeline asks the user to install Calibre or convert manually to EPUB/PDF first.

---

## Step-by-Step: Choosing an Extractor

### If your book is **technical** (code, tables, formulas):

**Best path:**
1. Choose `technical` mode
2. Pipeline runs **Docling** (preserves markdown tables and code blocks)
3. Expect ~1.5s/page (worth it for structure preservation)

**If Docling isn't installed:**
- The pipeline prompts: "Docling not found. Install with: `pip install docling`"
- User can install or fall back to `pdftotext` (loses structure)

### If your book is **text-heavy** (prose, narratives):

**Best path:**
1. Choose `text` mode
2. Pipeline tries `pdftotext` first (instant if installed)
3. Falls back to `pypdf` or `pdfminer` if pdftotext unavailable
4. Result: instant extraction, no structure loss (structure was minimal anyway)

**If you don't know:**
- Choose "Not sure" — defaults to `text` mode with fast extraction
- Pipeline warns if the book looks technical but you chose text mode

---

## Metadata & Token Estimation

After extraction, the pipeline creates `metadata.json`:

```json
{
  "total_pages": 244,
  "total_words": ~89000,
  "estimated_tokens": 119000,
  "sources": [
    {
      "filename": "book.pdf",
      "format": "pdf",
      "pages": 244,
      "words": 89000,
      "tokens": 119000
    }
  ]
}
```

**Token estimation formula:** `words / 0.75` ≈ tokens

This is used in **Step 2.5 (Pre-flight cost estimate)** to show the user how much generation will cost before committing.

---

## Troubleshooting Extraction Quality

### Problem: Tables come out as plain text

**Cause:** Using text-mode extraction (pdftotext, pypdf, or pdfminer)

**Solution:** Re-extract with `technical` mode and Docling. Docling converts tables to markdown.

### Problem: Text looks garbled or has random symbols

**Cause:** PDF uses unusual encoding or has OCR artifacts

**Solution:** Try a different PDF extractor:
- `pdfminer.six` is most robust for encoding edge cases
- If PDF is scanned (image-based), none of these work — you'd need OCR (outside this tool's scope)

### Problem: Extraction is too slow

**Cause:** Using Docling on a 500-page technical book (~8+ minutes)

**Solution:** Consider:
- Is the book truly technical? If mostly prose with a few code examples, use text mode instead
- Accept the time trade-off (structure preservation is worth it for reference later)
- Use a faster tool and accept losing table/code structure

### Problem: EPUB or DOCX structure is lost

**Cause:** The ebook/document has unusual internal structure

**Solution:**
- Convert to PDF and use Docling
- Or manually segment the extracted text before passing to skill generation

### Problem: "Extractor X not installed" error

**Run the preflight check:**

```bash
python3 scripts/extract.py --check
```

This prints:
- Which extractors are installed per format
- Exact `pip install` or system commands to add missing ones
- No file needed

Then install the missing tool and re-run extraction.

---

## Reference: Extractor Comparison

| Extractor | Format | Speed | Structure | Dependencies | Best For |
|-----------|--------|-------|-----------|--------------|----------|
| Docling | PDF | 1.5s/page | ⭐⭐⭐ tables/code as markdown | `pip install docling` | Technical PDFs |
| pdftotext | PDF | ⚡ instant | ❌ none | system: `poppler-utils` | Text-heavy PDFs |
| pypdf | PDF | ⚡ instant | ❌ none | `pip install pypdf` | Text-heavy, pure Python |
| pdfminer.six | PDF | ⚡ instant | ❌ none | `pip install pdfminer.six` | Robust fallback |
| ebooklib | EPUB | ⚡ instant | ⭐⭐⭐ good | `pip install ebooklib beautifulsoup4` | Preferred |
| zipfile | EPUB | ⚡ instant | ⭐⭐ ok | stdlib | Zero-dep fallback |
| python-docx | DOCX | ⚡ instant | ⭐⭐⭐ good | `pip install python-docx` | Preferred |
| BeautifulSoup | HTML | ⚡ instant | ⭐⭐⭐ good | `pip install beautifulsoup4` | Preferred |
| striprtf | RTF | ⚡ instant | ⭐⭐ ok | `pip install striprtf` | RTF documents |
| ebook-convert | MOBI/AZW | ~2s/file | ⭐⭐⭐ good | Calibre app | Kindle formats |

---

## Architecture

The extraction layer is modular:

```
book_to_skill/
├── parsers/
│   ├── pdf.py       # extract_with_docling, extract_with_pdftotext, etc.
│   ├── epub.py      # extract_with_ebooklib, extract_with_zipfile
│   ├── docx.py      # extract_docx
│   ├── html.py      # extract_html_file
│   ├── rtf.py       # extract_rtf
│   ├── text.py      # read_text_file (plain text, Markdown, RST, AsciiDoc)
│   └── calibre.py   # extract_with_ebook_convert (MOBI/AZW)
├── utils.py         # dispatcher: picks parser based on file extension
├── dependencies.py  # optional-dep probing (what's installed)
└── config.py        # format constants, token estimation
```

Each parser returns a string (extracted text) or `None` (extraction failed or tool not installed).

The **dispatcher** in utils.py tries extractors in order (based on mode: technical or text):

```python
# For PDFs in technical mode:
text = extract_with_docling(file_path)  # try first
if not text:
    text = extract_with_pdftotext(file_path)  # fallback
if not text:
    text = extract_with_pypdf(file_path)  # fallback
if not text:
    text = extract_with_pdfminer(file_path)  # fallback
```

---

## How to Use This Skill

Ask for help with:
- **Choosing an extractor** — "I have a technical PDF with lots of code. Which extractor should I use?"
- **Understanding trade-offs** — "Why is Docling slow? Should I use it anyway?"
- **Troubleshooting extraction** — "My tables turned into plain text. How do I fix this?"
- **Pre-flight setup** — "How do I know which extractors are installed?"
- **Architecture** — "Where is the extraction code in the repo?"

Example queries:
- `/book-to-skill-extraction-engine What's the difference between pdftotext and pypdf?`
- `/book-to-skill-extraction-engine My EPUB lost chapter structure. Why?`
- `/book-to-skill-extraction-engine technical-vs-text`
