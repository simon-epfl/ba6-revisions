# AGENTS.md — Usable Security & Privacy Cheatsheet

Working guide for an agent editing the exam cheatsheet in this directory.

## What this is

A single HTML file (`cheatsheet_compressed_1_page.html`) that compiles to a **2-page A4 PDF** summarising the course material for the USEC exam. The PDF is the deliverable; the HTML is the source.

Source material lives in `lectures/` (PDFs of the course slides). Past exams in `exams/`, tutorials in `tutorials/`.

## Files

- `cheatsheet_compressed_1_page.html` — the only file you edit
- `cheatsheet_compressed_1_page.pdf` — generated output, overwrite freely
- `cheatsheet_original_6_pages.html` — earlier snapshot, don't touch
- `lectures/lecture-*.pdf` — source material. Read with the Read tool (it handles PDFs). For PDFs > 10 pages, pass `pages: "1-5"` etc.

## Rendering

### Generate the PDF
```bash
weasyprint cheatsheet_compressed_1_page.html cheatsheet_compressed_1_page.pdf
```

### Read the rendered PDF visually (IMPORTANT)
Claude cannot visually parse the PDF directly — you must rasterise first. **Always use `pdftoppm` to verify layout.** The Read tool's PDF text output is ambiguous for multi-column layouts; only the rasterised image reveals actual column usage, gaps, and cutoffs.

```bash
# Both pages at readable resolution
pdftoppm -r 200 cheatsheet_compressed_1_page.pdf /tmp/cshi -png

# Single page (faster)
pdftoppm -r 200 cheatsheet_compressed_1_page.pdf /tmp/cshi -png -f 1 -l 1
```

Then `Read` the PNG: `/tmp/cshi-1.png`, `/tmp/cshi-2.png`.

### Check page count
```bash
pdfinfo cheatsheet_compressed_1_page.pdf | grep Pages
```
**Must be exactly 2.** If it's 3+, content is overflowing — either trim or shrink the font.

## Hard constraints

1. **Exactly 2 A4 pages.** Overflow to page 3 is a regression.
2. **Content must not be visually cut off** at the bottom of any column. The `.page` div has `overflow: hidden`, so hidden content appears as silently-truncated text. Always verify via `pdftoppm`.
3. **No emojis** unless explicitly requested.

## Layout architecture

Each page is a `<div class="page">` containing `<section>` cards. CSS multi-column (`column-count: 3`) flows cards through 3 columns per page.

Key CSS rules (in the `<style>` block):

- `column-fill: balance` — distributes content evenly across the 3 columns instead of filling column 1 then 2 then 3 sequentially. Critical — with `auto`, column 3 ends up empty if total content ≤ 2 columns.
- `section { /* no break-inside: avoid */ }` — sections are allowed to split across columns so all space gets used.
- `table, tr, h2, h3, h4 { break-inside: avoid; }` — atomic units that must stay together.
- `h2, h3, h4 { break-after: avoid; }` — a header never sits alone at the bottom of a column.

### Font sizing — single source of truth

All fonts are `rem` (relative to `html { font-size: Xpt }`). To resize the whole sheet, change the `html` `font-size` only. Ratios currently:

| Element | Size |
|---|---|
| `html` | base (e.g. 5pt) |
| `.ptitle` | 1.21rem |
| `h2` | 1.08rem |
| `h3` | 1.02rem |
| `h4` | 0.97rem |
| `.smallnote`, `table` | 0.92rem |
| `.vs` | 0.86rem |

Do not reintroduce absolute `pt` values for individual elements — it breaks the scaling invariant.

### Section colour themes

Each `<h2>` gets a colour class that signals the topic family. Keep them semantically meaningful:

- `blue` — authentication, crypto, technical auth
- `purple` — privacy concepts, consent, policies
- `green` — warnings, communication frameworks (HitL, NEAT, SPRUCE)
- `red` — ethics, GDPR, law
- `teal` — research methods, study design
- `orange` — attacks, phishing, IoT, AI, access control
- `slate` — meta sections (case studies, exam heuristics)

## Page split

- **Page 1** — Authentication · Privacy · Warnings · Consent & Policies (blue/purple/green heavy)
- **Page 2** — Study Design · Ethics · GDPR · Phishing · IoT · AI · Access Control (teal/red/orange heavy)

If you move a section between pages, also update the `<div class="ptitle">` text and consider recolouring the `h2` to match the new page's palette.

## Workflow for content changes

1. Edit `cheatsheet.html`.
2. Run `weasyprint`.
3. Check page count with `pdfinfo`.
4. **Rasterise with `pdftoppm -r 200`** and `Read` the PNG(s).
5. Verify: exactly 2 pages, all 3 columns used on each page, no visible cutoffs, content ends cleanly.
6. If overflowing → trim content, shrink font, or move sections between pages.
7. If underfilled (column 3 mostly empty on a page) → either add high-value content from `lectures/` or verify `column-fill: balance` is set.

## Adding exam-valuable content

Source is `lectures/lecture-*.pdf`. When asked to add content, skim the relevant lecture (use `Read` with a `pages` range for long PDFs) and extract:
- Named frameworks, acronyms, taxonomies (exam loves these)
- Specific studies with authors + key finding
- Definitions and useful distinctions
- Exam-answer templates

Skip content already well-covered. For an overview of what's already there, grep for `<h2` in the HTML.

## Common pitfalls

- **"Page 1 only shows 2 columns filled"** — `column-fill: auto` was reintroduced, or there isn't enough content. Confirm `column-fill: balance` is set.
- **"Text looks way too small compared to the rest"** — someone reintroduced absolute `pt` font sizes. All element font-sizes must be `rem`.
- **"Content seems to be missing"** — `overflow: hidden` silently truncates. Rasterise and read the PNG to see what's actually on the page.
- **`pdftoppm` not found** — the poppler binary lives at `/opt/homebrew/Cellar/poppler/*/bin/pdftoppm`. Symlink to `/opt/homebrew/bin/pdftoppm` if the PATH doesn't already resolve it.
- **Tables split across columns** — ensure `table, tr { break-inside: avoid; }` is present.

## Things the user has previously confirmed

- Rem-based font scaling with a single `html` knob is the right abstraction.
- Sections should flow across column boundaries — no `break-inside: avoid` on `section`.
- Always rasterise with `pdftoppm` before reporting layout results back to the user.
- Bigger-than-minimum font is preferred; current base is 5pt.
