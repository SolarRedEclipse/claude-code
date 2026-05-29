---
paths:
  - "*.html"
  - "*.css"
  - "*.js"
---

## Code Style — HTML / CSS / JavaScript

- Use semantic HTML elements: `<main>`, `<section>`, `<button>` — avoid `<div>` soup
- Every interactive element must have an accessible `aria-label` or visible label
- Use **CSS custom properties** (`--var-name`) for colors, spacing, and font sizes
- Use `const` by default; `let` only when reassignment is needed; never `var`
- Prefer `addEventListener` over inline `onclick` HTML attributes
- **No external libraries or CDN imports** unless explicitly approved
- 2-space indentation; single trailing newline; no `console.log` before committing

---

## Ad Placement Strategy

### The Golden Positions (highest RPM, in order)

| Position | Unit Size | Why it works |
|---|---|---|
| Below header / above fold | 728×90 leaderboard | First thing seen — 100% viewability |
| Sidebar top (sticky) | 300×250 rectangle | Stays in view as user scrolls — continuous impressions |
| Mid-content (between sections) | 728×90 or 300×250 | User is engaged and reading — high CTR |
| Sidebar second (below fold) | 300×250 rectangle | Catches users who scrolled past the first |
| Before footer | 728×90 leaderboard | Last impression before user leaves |

### Rules

- **Maximum 3 ads visible at once** — more tanks quality scores on AdSense and Mediavine
- **Never place an ad immediately before or after another ad** — content must separate them
- **Never push the first piece of real content below the fold** with ads — Google penalises this heavily
- **Always add the label** `Advertisement` or `Sponsored` above each slot — legally required and required by AdSense policy
- **Use `min-height` on ad containers** — prevents layout shift (CLS), which hurts Core Web Vitals
- No interstitials, pop-overs, ads inside the hero, or auto-refreshing ads

### For Website Flipping

- Add **3 clearly labelled ad slots** minimum — buyers want to see monetisation infrastructure in place
- Use `div` containers with a dashed border and `Advertisement — 728×90` label as placeholders
- Cluster ad slots where **dwell time is highest**: after the hero, mid-article, and in the sticky sidebar

---

## Website Design Recreation

### Workflow

1. **Generate** a single HTML file using Tailwind CSS via CDN. Include all content inline.
2. **Screenshot** the rendered page using Playwright:

```python
python3 - <<'EOF'
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page(viewport={"width": 1280, "height": 900})
    page.goto("file:///absolute/path/to/your/file.html")
    page.wait_for_timeout(2000)
    page.screenshot(path="screenshot_current.png", full_page=True)
    browser.close()
    print("Screenshot saved.")
EOF
```

3. **Read the screenshot** using the Read tool and visually compare against the reference image.
4. **List every mismatch**: spacing, font sizes/weights, colors (exact hex), alignment, border radii, shadows, image sizing.
5. **Fix** every mismatch. Edit the HTML.
6. **Re-screenshot** and compare again. Repeat until no visible differences — always do at least 2 rounds.
7. Always delete `screenshot_current.png` after the session — never commit it.

### Rules

- Do not add features, sections, or content not in the reference image — match exactly
- Tailwind via CDN is allowed for recreation tasks — the "no frameworks" rule applies to the web app project only
- Placeholder images: `https://picsum.photos/seed/<word>/<width>/<height>`
