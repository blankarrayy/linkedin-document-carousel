# linkedin-document-carousel

A Claude Code skill that creates high-converting **LinkedIn PDF document carousels** (1080×1350pt portrait, 4:5) plus companion post copy — and actually ships a QA'd PDF, not just a slide outline.

**Author:** [blankarrayy](http://github.com/blankarrayy/) · **Repo:** [blankarrayy/linkedin-document-carousel](http://github.com/blankarrayy/linkedin-document-carousel)

## What it does

- Turns an article, workflow, thread, or concept into a 7–10 slide swipeable carousel
- Two **formats**: **1080×1350 portrait (4:5, default)** or **1080×1080 square (1:1)**
- Offers **three design directions** the user picks from:
  - **Minimal Professional** (default) — clean consulting-deck look, safe for any industry
  - **Experimental** — dark, loud type, asymmetric; for builders and personal brands
  - **Editorial** — magazine/serif thought-leadership energy
- Writes the companion LinkedIn post (hook, body, CTA question)
- Builds the PDF through a fixed pipeline and QA's it before handing it over:

  ```
  carousel.html (HTML/CSS layout)
      → Playwright headless Chromium  (DOM overflow QA → page.pdf render)
      → PyMuPDF                      (parse back: page geometry, text bbox, font-size asserts)
      → slide-NN.png                 (visual QA — every slide inspected)
      → carousel.pdf
  ```

No reportlab, weasyprint, or hand-drawn PDF text — layout lives in HTML; PyMuPDF only audits the output.

## Requirements

- Python 3.9+
- Two packages (installed by the skill on first use, or pre-install yourself):

```bash
python3 -m pip install pymupdf playwright
python3 -m playwright install chromium   # add --with-deps on Linux
```

## Install

### Option 1 — Clone into your personal skills directory (recommended)

```bash
git clone http://github.com/blankarrayy/linkedin-document-carousel.git \
  ~/.claude/skills/linkedin-document-carousel
```

Restart Claude Code (or start a new session). The skill appears as `linkedin-document-carousel` and triggers automatically when you ask for a LinkedIn document, carousel, or swipeable PDF post.

### Option 2 — Project-level install (team-shared)

From your project root:

```bash
git clone http://github.com/blankarrayy/linkedin-document-carousel.git \
  .claude/skills/linkedin-document-carousel
```

Commit `.claude/skills/` so everyone on the team gets the skill.

### Option 3 — Manual copy

Download or copy this folder to either:

- `~/.claude/skills/linkedin-document-carousel/` (all projects), or
- `<your-project>/.claude/skills/linkedin-document-carousel/` (one project)

Either way, the folder must contain `SKILL.md` at its root.

### Verify

```bash
ls ~/.claude/skills/linkedin-document-carousel/SKILL.md
python3 -c "import fitz, playwright; print('deps ok')"
```

## Usage

Just ask in Claude Code:

> "Make a LinkedIn carousel from this blog post: \<paste or link\>"

> "Turn this thread into a 9-slide LinkedIn document, experimental design"

> "Make a square 1:1 LinkedIn carousel from these release notes, editorial style"

The skill will:

1. Clarify topic, audience, slide count, **format** (portrait 4:5 or square 1:1), and **design direction** (if not specified)
2. Structure the slide arc (cover hook → insights → CTA)
3. Write `carousel.html` + `build_carousel.py`, run the render → parse → QA loop
4. Deliver `carousel.pdf` (in the chosen format), `slide-NN.png` previews, the source files, and the companion post copy

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | The skill definition Claude Code loads |
| `README.md` | This file |

## Related skills

- Text-only feed post → [`linkedin-post-writer`](http://github.com/blankarrayy/linkedin-post-writer)
- De-AI an existing draft → `linkedin-humanizer`
- Repurpose an article/thread into source material first → `linkedin-repurposer`

## License

See the repository for license details.
