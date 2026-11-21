---
name: linkedin-document-carousel
description: Create high-converting LinkedIn PDF document carousels (1080x1350 portrait 4:5 or 1080x1080 square 1:1) plus companion post copy. Use when the user asks for a LinkedIn document, LinkedIn carousel, swipeable PDF post, slide deck for LinkedIn, or wants to repurpose an article, workflow, thread, or concept into LinkedIn slides with slide-by-slide copy and layout specs. Not for plain text-only feed posts (use linkedin-post-writer) or for auditing an existing draft (use linkedin-humanizer).
---

# LinkedIn Document Carousel Creator

Create swipeable LinkedIn PDF document carousels in one of two formats — **1080×1350 (4:5 portrait, default)** or **1080×1080 (1:1 square)** — plus high-engagement companion post copy. LinkedIn renders uploaded PDFs as native swipeable carousels, which boosts dwell time and saves.

## Format (user picks one)

| Format | Value | Page size in pt | When to use |
|---|---|---|---|
| **Portrait (default)** | 1080×1350, 4:5 | `1080pt × 1350pt` | Maximum vertical real estate on the LinkedIn mobile feed — default unless the user says otherwise |
| **Square** | 1080×1080, 1:1 | `1080pt × 1080pt` | Even grid presence, simpler layouts, quotes/stat slides, when the user asks for square |

The px figures are how LinkedIn users talk about the sizes; author and assert in **pt** (1:1 px↔pt mapping at these dimensions) so the PDF pages come out exact. Record the choice as `data-format="portrait"` or `data-format="square"` on `<body>` alongside `data-design`. All pipeline geometry below is parameterized by this choice — never hardcode 1350 when the format is square.

**Pipeline:** author slides as **HTML/CSS** → render to PDF with **Playwright (headless Chromium)** → parse, QA, and verify the PDF with **PyMuPDF**. Do not generate the PDF with reportlab, weasyprint, fpdf, or direct PyMuPDF text-drawing — layout lives in HTML; PyMuPDF only audits the output.

## Mandatory libraries (exact install)

Two packages, no substitutes:

| Step | Library | Role |
|---|---|---|
| Render HTML → PDF | `playwright` (Python) + Chromium | The only renderer — pixel-faithful CSS, real browser engine |
| Parse / QA / verify PDF | `pymupdf` (imports as `fitz`) | Re-open the PDF, bbox checks, page-geometry asserts, PNG renders for visual QA |

```bash
python3 -m pip install --quiet pymupdf playwright
python3 -m playwright install chromium        # add --with-deps on Linux if system libs are missing
```

Verify both before any work:

```bash
python3 -c "import fitz, playwright; print(fitz.__doc__); print('ok')"
```

If `import fitz` fails: `python3 -m pip install --user pymupdf`, then `python3 -m pip install --force-reinstall pymupdf`. If Chromium is missing: re-run `python3 -m playwright install chromium`. Never swap in another PDF library — fix the install.

## Canonical pipeline

### 1. Author `carousel.html`

One self-contained HTML file, all slides in document order. **First pick the format** (portrait 1080×1350 or square 1080×1080) **and the design direction** (minimal-professional / experimental / editorial), and set both on `<body data-format="…" data-design="…">`; the `<style>` block below is the shared skeleton — override its tokens (page size, palette, fonts, heading sizes, footer style) to match the chosen format and direction. Starting point (portrait shown; for square change `--page-h` to `1080pt` in both `@page` and `.slide`):

```html
<!doctype html>
<html><head><meta charset="utf-8">
<style>
  /* Tokens — set these from the chosen format + design direction */
  :root {
    --page-w: 1080pt;
    --page-h: 1350pt;            /* square format: 1080pt */
    --bg: #FAFAF8; --ink: #111111; --accent: #1F4FD8; --muted: #6B6B6B;
    --font-display: Helvetica, Arial, sans-serif;
    --font-body: Helvetica, Arial, sans-serif;
  }
  @page { size: 1080pt 1350pt; margin: 0; }  /* square: size: 1080pt 1080pt — keep literal, match --page-h */
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { font-family: var(--font-body); background: var(--bg); color: var(--ink); }
  .slide {
    width: var(--page-w); height: var(--page-h);
    page-break-after: always; break-after: page;
    overflow: hidden; position: relative;
    padding: 72pt 72pt 96pt;
    display: flex; flex-direction: column; justify-content: center;
  }
  .slide:last-child { page-break-after: auto; }
  h1 { font-size: 72pt; font-weight: 700; line-height: 1.1; }
  h2 { font-size: 44pt; font-weight: 700; line-height: 1.2; }
  p, li { font-size: 24pt; line-height: 1.4; }   /* body floor: 24pt */
  .footer { position: absolute; left: 72pt; right: 72pt; bottom: 40pt;
            display: flex; justify-content: space-between; font-size: 18pt; }
</style></head>
<body data-format="portrait" data-design="minimal-professional">
  <section class="slide" data-slide="1"> … cover … </section>
  <section class="slide" data-slide="2"> … insight … </section>
  <!-- one <section class="slide"> per slide; N−1 middle slides, CTA last -->
</body></html>
```

Rules:

- **Units are pt, not px.** `@page size: 1080pt 1350pt` (or `1080pt 1080pt` square) + `.slide` at the same dimensions makes Chromium emit PDF pages of exactly 1080×1350pt or 1080×1080pt. A px-based layout converts at 96dpi and lands at 810×1012.5pt / 810×810pt — right ratio, wrong assert target. Author in pt so geometry asserts pass. Note `@page { size }` cannot reliably read `var()` in Chromium — keep it a literal matching `--page-w`/`--page-h`.
- **Format drives geometry; direction drives skin.** Change `--page-h`, `@page size`, and the `page.pdf()` / PyMuPDF assert dimensions together when switching formats. Swap `--bg`, `--ink`, `--accent`, fonts, heading sizes, and footer treatment per the direction's spec — but keep the pt geometry, the 24pt body floor, and the `.slide` structure unchanged. Experimental may push cover titles to 80–96pt; editorial uses serifs; minimal stays restrained.
- **Square format is tighter:** 270pt shorter — trim cover titles toward the lower end of each direction's range (e.g. 64pt minimal, 80pt experimental), and keep middle-slide body even leaner (target ≤32 words) so centered layouts don't crowd the footer.
- Body ≥ 24pt, headings ≥ 44pt (cover title per direction: 64–72pt minimal, 80–96pt experimental, 64–80pt editorial). High-contrast backgrounds via the direction's palette.
- Footer: consistent handle left, `01/08`-style slide number right — inside `.footer`, never floating free.
- Keep copy inside the slide: 35–40 words max per body slide (≤32 in square). `overflow: hidden` on `.slide` is a backstop, not a license to overfill.

### 2. Pre-render DOM QA (Playwright, before PDF)

Open the file and check overflow in the real layout engine — catch problems before the PDF exists:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("file:///abs/path/carousel.html", wait_until="networkidle")
    over = page.evaluate("""() => [...document.querySelectorAll('.slide')].map((el, i) => ({
        slide: i + 1,
        overflowY: el.scrollHeight - el.clientHeight,
        words: (el.innerText.trim().split(/\\s+/).length)
    })).filter(s => s.overflowY > 1 || s.words > 40)""")
    assert not over, f"DOM QA failed: {over}"
    # then render (step 3) in the same session
    browser.close()
```

Fix the HTML and re-run until this returns `[]`.

### 3. Render HTML → PDF (Playwright, still in the same session)

```python
    # portrait: 1080×1350pt  |  square: 1080×1080pt — match the chosen format
    page.pdf(
        path="carousel.pdf",
        width="1080pt", height="1350pt",   # square: height="1080pt"
        print_background=True,        # required — solid slide backgrounds
        prefer_css_page_size=True,    # honor @page size above
        margin={"top": "0", "bottom": "0", "left": "0", "right": "0"},
    )
```

Keep steps 2–3 in one script (e.g. `build_carousel.py`) so a copy edit → re-QA → re-render is one command.

### 4. Parse back + programmatic QA (PyMuPDF — mandatory)

Reopen the rendered file; never trust the renderer blindly:

```python
import fitz

FORMATS = {"portrait": (1080, 1350), "square": (1080, 1080)}
W, H = FORMATS[fmt]                      # fmt = "portrait" or "square"
doc = fitz.open("carousel.pdf")
assert len(doc) == slide_count
MARGIN = 36  # pt
for i, page in enumerate(doc, 1):
    assert page.rect == fitz.Rect(0, 0, W, H), f"slide {i}: wrong page size {page.rect} (expected {W}×{H} for {fmt})"
    for block in page.get_text("dict")["blocks"]:
        for line in block.get("lines", []):
            for span in line["spans"]:
                x0, y0, x1, y1 = span["bbox"]
                assert x0 >= MARGIN and x1 <= W - MARGIN, f"slide {i}: x overflow {span['text']!r}"
                assert y0 >= MARGIN and y1 <= H - MARGIN, f"slide {i}: y overflow {span['text']!r}"
                assert span["size"] >= 17, f"slide {i}: font too small ({span['size']}pt): {span['text']!r}"
```

(Header/heading spans should be checked at ≥44pt where the design says so; 17pt is the absolute floor for the footer. The HTML spec keeps body at 24pt.)

Any assert failure → edit `carousel.html`, re-run steps 2–4. No skipping.

### 5. Visual QA (PyMuPDF render → Read the PNGs)

```python
for i, page in enumerate(doc, 1):
    pix = page.get_pixmap(matrix=fitz.Matrix(1, 1), alpha=False)  # 1080×1350 px, or 1080×1080 square
    pix.save(f"slide-{i:02d}.png")
```

Read every `slide-*.png` and visually confirm: no clipping, no overlap, no wall-of-text, footer and slide numbers present, contrast holds. Fix and re-loop until clean.

### 6. Finalize

Save a compressed copy if you mutated the doc (`doc.save("carousel.final.pdf", garbage=4, deflate=True)`). Deliver `carousel.pdf` (7–10 slides typical, hard cap 30, well under LinkedIn's 100MB limit) plus the companion post copy below. State the delivered format in your summary (e.g. "1080×1350 portrait" / "1080×1080 square"). Ship `carousel.html` and `build_carousel.py` alongside so the user can re-run after edits.

## Design directions (user picks one)

The pipeline is identical for every direction — only the `<style>` block inside `carousel.html` and the copy tone change. Pick **one** mode per carousel; never blend two modes in one document. Record the choice as a `data-design="…"` attribute on `<body>` so re-runs stay consistent.

### A. `minimal-professional` — Minimal Professional (default)

Corporate-safe, high whitespace, zero decoration. Reads like a well-made consulting deck.

- **Palette:** near-white background `#FAFAF8` (or pure `#FFFFFF`), ink `#111111`, single restrained accent (e.g. `#1F4FD8` or brand color) used only for rules, numbers, and the CTA. One alternate slide may invert (ink background, white text) for the cover or CTA only.
- **Type:** one sans family (Helvetica/Inter). Cover title 64–72pt regular or medium — **not** ultra-black. Body 24pt, generous line-height 1.45. Sentence case headings; no ALL-CAPS except tiny eyebrow labels (16–18pt, letter-spaced).
- **Layout:** single column, strong left alignment, 72pt+ margins. Thin 1–2pt horizontal rules to separate eyebrow from headline. Numbers set large (48pt+) in the accent color. Lots of empty space — if a slide feels full, cut words, don't shrink type.
- **Footer:** hairline rule above; handle left, `01/08` right, 16–18pt, muted gray.
- **Avoid:** gradients, shadows, rounded cards, emoji, dark backgrounds on more than one slide.

### B. `experimental` — Cool / Experimental

Dark-first, loud type, asymmetric. For builders, dev-tools, design-engineering personal brands.

- **Palette:** deep background `#0B0B0F` / `#0E1116`, off-white text `#F5F5F7`, one electric accent (`#7C3AED`, `#22D3EE`, `#C6FF00` — pick one). Optional subtle gradient wash or a single large geometric shape (`border-radius`, rotated rect) per slide.
- **Type:** one sans, extreme contrast — cover title 80–96pt heavy (800/900), body 22–24pt regular. Oversized slide index (`03` at 120pt+, accent color, low opacity, bleeding off an edge). Short punchy lines; mono font (`ui-monospace`, `JetBrains Mono`) for labels, code-ish eyebrows, and the footer.
- **Layout:** break the grid on purpose — title pushed to one edge, a stat occupying the opposite corner, intentional asymmetry. One hero element per slide (a number, a one-word title, a diagram). Stagger reveals of information across middle slides.
- **Footer:** mono, accent slash — `handle / 03—08`.
- **Avoid:** more than one accent color per slide, soft corporate blues, dense paragraphs.

### C. `editorial` — Editorial / Magazine

Print-magazine energy for thought leadership, essays, contrarian takes.

- **Palette:** warm paper `#F6F1E7` or crisp `#FFFFFF`, near-black ink `#1A1A1A`, one ink-wash accent (`#8B1E1E`, `#1A5E3A`, or deep navy). Serif-dominant.
- **Type:** serif for display and body (Georgia / `Source Serif` / `Libre Baskerville`; embed a TTF via `@font-face` if provided). Cover title 64–80pt, with a serif italic subtitle. Body 22–24pt, line-height 1.5. Sans only for eyebrows and footers (small, tracked-out caps). Drop-cap or oversized initial on narrative slides if it fits cleanly.
- **Layout:** magazine moves — eyebrow (category label) → serif headline → rule → deck/subhead → body. Two-column body only when text is short enough to stay ≥24pt. Pull-quote slides: one sentence at 40–48pt italic, centered, with quotation marks. Page furniture like print: running eyebrow top, folio bottom.
- **Footer:** folio style — `handle` left, `· 03 ·` centered or right, small caps.
- **Avoid:** neon accents, heavy geometric shapes, ultra-black weights, tech-startup gradients.

### Choosing well

- User's audience is recruiters/clients/execs → `minimal-professional`.
- User is a builder/designer/creator showing taste → `experimental`.
- User is making an argument, essay, or POV post → `editorial`.
- Brand palette or font file provided → keep the chosen direction's *layout and type character*, but swap in the brand colors/font so the deck stays on-brand (the direction still governs structure and tone).
- User asks for something "like the design of X" → map X onto the closest direction above and say which one you picked.

## When to use

- User asks to create a LinkedIn document, carousel, or swipeable PDF post (portrait or square).
- User wants to turn a blog post, technical architecture, workflow, or thread into LinkedIn slides.
- User asks for slide-by-slide LinkedIn copy and visual layout specifications.

## Steps

### 1. Clarify inputs (if missing)

Confirm the core topic or source material, target audience, slide count (default: 7–10), brand palette/handle, accent color, and brand font file (optional TTF → `@font-face` in `carousel.html`) if not already known.

Also confirm the **format** (see the Format section above). If unspecified, default to **portrait 1080×1350 (4:5)**; ask when in doubt:

- **Portrait (1080×1350, 4:5)** — default; tallest presence in the mobile feed
- **Square (1080×1080, 1:1)** — even grid presence; simpler, punchier layouts

And confirm the **design direction** (see the section above). If the user doesn't specify one, ask with these labels — or default to `minimal-professional` if they say "just make it":

- **Minimal Professional** — safe for any industry, recruiter- and exec-readable
- **Experimental** — for builders, designers, dev-tools, personal brands that want to stand out
- **Editorial** — thought-leadership and long-form essay energy

### 2. Structure the slide arc

- **Slide 1 (Cover Hook):** Bold, high-contrast title (under 10 words) + short subtitle + swipe cue. No dense paragraphs.
- **Slides 2 to N−1 (Core Insights):** One idea per slide. Body under 35–40 words. Numbered steps, short bullets, or clear visual comparisons.
- **Final Slide (Closing CTA):** One-line takeaway + a specific action (comment, save, visit link).

### 3. Build via the canonical pipeline

Write `carousel.html` (chosen format + design direction, pt units, `.slide` sections) and `build_carousel.py` (DOM QA → `page.pdf` → PyMuPDF parse/QA → PNG render). Run them. Only hand over a PDF that passed bbox asserts **and** your visual read of every rendered slide — and confirm every slide reads as *one* consistent direction, not a mix.

If Python/Pip/Playwright cannot run in the active environment, output the slide-by-slide blueprint plus the complete `carousel.html` and `build_carousel.py`, and the exact install lines (`python3 -m pip install pymupdf playwright && python3 -m playwright install chromium`).

### 4. Write companion LinkedIn post copy

- **Hook (lines 1–2):** Under 12 words, stops the scroll, no "I" opener or corporate fluff.
- **Body:** 3–5 short lines on why the document matters, with concrete numbers or outcomes.
- **CTA question:** One specific, easy-to-answer question to drive comments.

## Gotchas

- **Wrong unit → wrong page size:** px-authored slides render at 810×1012.5pt (portrait) or 810×810pt (square). Author everything in pt; assert `page.rect` against `FORMATS[fmt]` in step 4 — a failed assert means the file is wrong, not the assert.
- **Format mismatch mid-pipeline:** Changing the format means updating *all four* together: `@page size`, `.slide` height, `page.pdf(height=…)`, and the PyMuPDF `W, H` assert. Updating only the CSS leaves the render and QA disagreeing.
- **`print_background=True` omitted:** Chromium drops solid backgrounds; slides ship white-on-white. Always pass it.
- **Wall-of-text slides:** Never exceed 40 words per slide. Split across two slides instead of shrinking fonts — the DOM word-count check and bbox QA both enforce this.
- **Skipping parse-back QA:** Rendering without re-opening the PDF in PyMuPDF is how clipped text ships. Steps 4–5 are mandatory, not optional.
- **Image-only workarounds:** Do not screenshot slides and paste images into a PDF to dodge layout issues — fix the HTML. (Text-based PDFs stay selectable and were audited for overflow.)
- **Blending directions:** One mode per carousel. A minimal deck with one neon experimental slide looks broken — if a slide needs emphasis, use the direction's own tools (accent rule, inverted panel, pull quote).
- **Decorating past the copy:** The direction changes look, not length. 35–40 words/slide and the 24pt floor hold in all three modes; experimental's oversized type means *fewer* words, not smaller ones.
- **AI buzzwords:** No "delve", "harness", "unlock", "tapestry", or "game-changer" in slides or post copy. Direct, concrete language only.

## Related skills

- Drafting a text-only feed post from scratch → `linkedin-post-writer`
- Stripping AI tells from carousel captions or the companion post → `linkedin-humanizer`
- Turning an existing article or thread into source material first → `linkedin-repurposer`
