---
name: cns-paper-collector
description: >
  Collect, download, rename, and package academic papers (CNS + sub-journals) from WeChat articles, URLs, DOIs, or titles. Handles the entire pipeline: metadata extraction → PDF download → rename → ZIP. Trigger when the user pastes WeChat links, journal URLs, DOIs, or says 汇总/下载/打包/收集.
---

# CNS Paper Collector

**Self-contained. No dependencies on memory files, local scripts, or custom tools. Works on macOS out of the box. Linux/Windows may need minor path adjustments.**

The user handles the actual downloading (clicking "Download PDF" in Chrome). Claude handles everything else: fetching content, extracting metadata, constructing PDF URLs, opening Chrome tabs, renaming files, and packaging ZIPs.

---

## Assumptions (users MUST verify these before first use)

This skill assumes:

| Assumption | Why | How to verify |
|------------|-----|---------------|
| **scrapling is installed** | WeChat articles are JS-rendered and can't be fetched with curl | Run `scrapling --help` |
| **macOS with Swift** | OCR for journal screenshots uses Vision framework | Run `swift --version` |
| **Google Chrome is in /Applications** | PDF pages open via `open -a "Google Chrome"` | Run `ls /Applications/Google\ Chrome.app` |
| **curl is available** | OA PDFs are downloaded with curl | Run `curl --version` |
| **Crossref API is accessible** | DOI lookups use `api.crossref.org` | Run `curl -s https://api.crossref.org/works/10.1038/s41586-026-10476-w` |

If any assumption fails, see the "Frequently Broken Assumptions" section at the bottom.

---

## Download Priority (MUST follow this order)

| Priority | Method | Applies to |
|----------|--------|------------|
| **1. Cache** | Check `~/Downloads/` — already downloaded? Skip. | Everything |
| **2. curl** | `curl -H "Accept: application/pdf" -H "Referer: <article>"` | Nature OA, Nature Comms, Sci Reports, npj, bioRxiv, Genome Biology |
| **3. Chrome** | `open -a "Google Chrome" -n --args --new-window <urls>` | Nature paywalled (406), Science, Cell, non-standard DOIs |
| **4. Give up** | Don't attempt programmatic download. | Cell.com (Cloudflare impenetrable), Sci-Hub (unreliable) |

---

## Metadata Decision Tree

```
Input received (URLs / titles / WeChat links):
│
├─ DOI embedded in URL (nature.com/articles/xxx, doi.org/xxx)?
│   → Extract directly. Skip to Phase 1.
│
├─ WeChat MP URL?
│   → Fetch with scrapling dynamic browser (NOT static get)
│   → For each article, try in order:
│       1. Scan for DOI in body → Crossref API → authoritative metadata
│       2. No DOI? Download first content image → macOS OCR → extract title/journal/date
│       3. No screenshot? Extract blockquote English title → Crossref search
│       4. Only Chinese title? Ask user for DOI or English title
│   → Detect multi-paper: scan for "上一篇/下一篇" boundaries and multiple DOIs
│     If 2+ papers in one article → collect all → confirm with user
│
└─ Paper title only (no URL)?
    → Batch all titles → single parallel Crossref query → fill gaps
    → NEVER fetch publisher pages for metadata (blocks/redirects)
```

---

## Phase 0: Collect Metadata

### Path A: DOI already known

Extract directly from URL — don't re-fetch anything:

| URL pattern | DOI |
|-------------|-----|
| `nature.com/articles/s41586-xxx` | `10.1038/s41586-xxx` |
| `doi.org/10.xxx/yyy` | `10.xxx/yyy` |
| `cell.com/cell/abstract/S0092-8674(xx)xxx` | Need Crossref lookup |

### Path B: WeChat article

**B1. Fetch with scrapling dynamic browser:**

```bash
scrapling extract fetch "https://mp.weixin.qq.com/s/..." /tmp/article.md \
  --timeout 60000 --wait 5000
```

If fetching multiple articles, stagger launches by 2-3s to avoid WeChat captcha:
```bash
scrapling extract fetch "url1" /tmp/w1.md --timeout 60000 --wait 5000 &
sleep 2
scrapling extract fetch "url2" /tmp/w2.md --timeout 60000 --wait 5000 &
sleep 2
... && wait
```

**B2. Extract metadata from fetched markdown:**

Step 1 — Scan for paper DOI (filter out citation DOIs in reference section):
```python
import re
body = text.split("尾注")[0].split("References")[0]
paper_dois = re.findall(r'10\.\d{4,}/[^\s"\'\)\]>#&，。；：]+', body)
```

Step 2 — No paper DOI? Try OCR on journal screenshot:
- Download the first content image after the title (not the cover/avatar)
- Run macOS Vision OCR (see Quick Reference below for the Swift script)
- Extract: title (longest English sentence), journal ("Nature"/"Science"/"Cell"), date ("Published: DD Mon YYYY")

Step 3 — No screenshot? Extract blockquote English title:
```python
bq_titles = re.findall(r'^> ([A-Z][^\n]{40,200})$', text, re.MULTILINE)
```

Step 4 — Only Chinese title? Ask user.

**B3. Batch all missing DOIs via Crossref (single round, parallel queries).**

**B4. Multi-paper detection:** If one WeChat article covers multiple papers, scan for `上一篇`/`下一篇` boundaries and multiple DOIs in body. Collect all → show user → confirm.

### Path C: Title only (no URL)

Batch ALL titles into one parallel Crossref query. Don't fetch publisher pages for metadata — they block programmatic access.

---

## Phase 1: Download PDFs

### Step 1: Check cache

```bash
ls ~/Downloads/ | grep -i "<article-id or keyword>"
# If valid PDF exists (head -c 5 file.pdf == "%PDF-") → skip
```

### Step 2: curl OA articles

```bash
curl -sL -o ~/Downloads/<name>.pdf \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  -H "Accept: application/pdf" \
  -H "Referer: <article-page-url>" \
  "<pdf-url>"

# Validate: head -c 5 ~/Downloads/<name>.pdf should be "%PDF-"
# If 406 or HTML → fall to Step 3
```

### Step 3: Open remaining in Chrome NEW WINDOW

```bash
open -a "Google Chrome" -n --args --new-window \
  "https://www.nature.com/articles/xxx" \
  "https://www.science.org/doi/pdf/10.1126/xxx" \
  ...
```

Tell user: "已在 Chrome 新窗口打开 N 篇文章。请通过 Cloudflare 验证并下载 PDF。完成后告诉我。"

**Wait for user confirmation before Phase 2.**

### PDF URL construction

| Publisher | PDF URL |
|-----------|---------|
| Nature family | `https://www.nature.com/articles/{article-id}.pdf` |
| Science | `https://www.science.org/doi/pdf/{DOI}` |
| bioRxiv | `https://www.biorxiv.org/content/{DOI}.full.pdf` |
| Cell Press | Open article page, user clicks PDF button |
| Genome Biology | `https://genomebiology.biomedcentral.com/counter/pdf/{DOI}` |
| Oxford Academic | Try `https://academic.oup.com/{journal}/article-pdf/{doi-suffix}.pdf` |

---

## Phase 2: Rename & Package

### Find new downloads

```bash
ls -lt ~/Downloads/*.pdf | head -10
```

### Rename

Format: `YYYY-Journal-Title.pdf` (max 120 chars, strip illegal chars `/\:*?"<>|`)

```bash
mkdir -p /tmp/papers_final && rm -f /tmp/papers_final/*.pdf
cp "original.pdf" "/tmp/papers_final/2026-Nature-Paper Title.pdf"
```

### Package

```bash
today=$(date +%Y%m%d)
cd /tmp/papers_final
zip -j "SMOOTH_文章汇总_${today}.zip" *.pdf
cp "SMOOTH_文章汇总_${today}.zip" ~/Desktop/
open ~/Desktop/
```

### Summary

```
✅ N papers → 📦 SMOOTH_文章汇总_YYYYMMDD.zip on Desktop

2026-Nature-Title one.pdf   (X.X MB)
2026-Cell-Title two.pdf     (X.X MB)
```

---

## Quick Reference

### curl download (OA articles)
```bash
curl -sL -o output.pdf \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  -H "Accept: application/pdf" \
  -H "Referer: <article-page-url>" \
  "<pdf-url>"
```

### scrapling commands
```bash
# Fetch WeChat article (dynamic browser required — static get returns JS skeleton)
scrapling extract fetch "<url>" output.md --timeout 60000 --wait 5000

# Search Nature for papers by topic
scrapling extract get "<search-url>" output.html --css-selector article --impersonate chrome

# Stealth fetch — works for Nature, NOT for Cell
scrapling extract stealthy-fetch "<url>" output.md --solve-cloudflare --real-chrome --timeout 90000

# ⚠️ CLI only supports .html/.md/.txt output — CANNOT save .pdf binaries
# ⚠️ --ai-targeted can strip too much content — avoid for search result pages
```

### macOS OCR for journal screenshots

Save this as `ocr_image.swift`:
```swift
import Vision
import CoreImage
import Foundation

guard CommandLine.arguments.count > 1 else { print("Usage: swift ocr_image.swift <image>"); exit(1) }

let path = CommandLine.arguments[1]
guard let img = CIImage(contentsOf: URL(fileURLWithPath: path)) else { exit(1) }

let semaphore = DispatchSemaphore(value: 0)
let request = VNRecognizeTextRequest { request, error in
    defer { semaphore.signal() }
    guard let obs = request.results as? [VNRecognizedTextObservation] else { return }
    for o in obs.prefix(20) {
        if let t = o.topCandidates(1).first { print(t.string) }
    }
}
request.recognitionLevel = .accurate
request.recognitionLanguages = ["en-US", "zh-Hans"]
try? VNImageRequestHandler(ciImage: img).perform([request])
semaphore.wait()
```

Usage: `swift ocr_image.swift screenshot.png`

### Chrome new window
```bash
open -a "Google Chrome" -n --args --new-window <url1> <url2> ...
```

### Crossref API (DOI lookup)
```bash
curl -s "https://api.crossref.org/works/10.1038/s41586-026-10476-w"
```

### Journal name mapping

| Category | Names |
|----------|-------|
| **CNS core** | Cell, Nature, Science |
| **Nature family** | Nat Med, Nat Biotechnol, Nat Methods, Nat Genet, Nat Cell Biol, Nat Chem Biol, Nat Commun, Nat Neurosci, Nat Immunol, Nat Metab, Nat Aging, Nat Cancer, Nat Struct Mol Biol, Nat Plants, Nat Clim Change, Nat Microbiol |
| **Cell family** | Cell Res, Cell Rep, Cell Metab, Cell Host Microbe, Cell Stem Cell, Cancer Cell, Immunity, Neuron, Mol Cell, Dev Cell, Curr Biol, iScience |
| **Science family** | Sci Adv, Sci Transl Med, Sci Immunol, Sci Robot |
| **Other** | bioRxiv, Genome Biology, Nucleic Acids Res, PNAS, eLife, PLoS Biol, PLoS Genet, EMBO J, etc. |

### Publisher anti-bot reference

| Publisher | Level | Programmatic? | Behavior |
|-----------|-------|---------------|----------|
| Nature OA | Light | curl works | `Accept: application/pdf` + `Referer` required |
| Nature paywalled | Medium | Returns 406 or HTML redirect | Open in Chrome for institutional access |
| Science | Medium | Browser required | PDF URL works after Cloudflare pass |
| Cell Press | **Maximum** | **Impossible** | Cloudflare "请稍候…" even with real Chrome |
| bioRxiv | None | curl works | No protection |
| Genome Biology | None | curl works | BMC OA journal |

### Error handling

| Situation | Action |
|-----------|--------|
| Nature .pdf → 406 | Paywalled, open in Chrome |
| Nature .pdf → redirects to HTML | User clicks "Download PDF" on article page |
| Cell article | Open article page in Chrome immediately |
| curl PDF is actually HTML | Check `head -c 5` — if not `%PDF-`, fall back to Chrome |
| WeChat captcha | Re-fetch with longer stagger between requests |
| WeChat markdown is JS skeleton | Use `extract fetch` (dynamic), not `extract get` |
| Multiple papers in one article | Collect all, confirm with user |
| Downloaded file has ` (1)` suffix | User re-downloaded, take the newest timestamp |
| Filename collision | Append ` (2)` before `.pdf` |

---

## Frequently Broken Assumptions (read before sharing)

### scrapling

**Problem**: scrapling is NOT a standard Python package. It requires:
- Homebrew Python 3.12 (macOS) or Python 3.11+ (Linux)
- Chrome/Chromium browser installed
- Playwright browsers: `playwright install chromium`
- Installation: `pip install scrapling[all]`

If scrapling isn't available, WeChat articles can't be auto-fetched. Fallback: ask the user to paste the article text or provide the paper title/DOI manually.

### macOS OCR

**Problem**: The Vision OCR script requires macOS 13+. On Linux, install `tesseract` instead:
```bash
sudo apt install tesseract-ocr tesseract-ocr-eng tesseract-ocr-chi-sim
tesseract screenshot.png stdout
```

### Chrome path

**Problem**: `open -a "Google Chrome"` is macOS-specific. On Linux:
```bash
google-chrome --new-window <url1> <url2>
```
On Windows (Git Bash):
```bash
start chrome --new-window <url1> <url2>
```

### ~/Downloads assumption

**Problem**: This skill assumes PDFs download to `~/Downloads/`. Some users have custom download paths. Ask before Phase 2 if files can't be found.

### curl headers

**Problem**: Some corporate networks block custom User-Agent headers. If curl returns 403 for OA articles, try removing headers entirely:
```bash
curl -sL -o output.pdf "<pdf-url>"
```

---

## Limits of This Skill

This skill CANNOT:
- Download Cell.com PDFs programmatically (Cloudflare is impenetrable)
- Work without scrapling for WeChat articles (these are JS-rendered)
- Download paywalled Nature/Science PDFs without institutional access
- Run fully automated (the user MUST handle Cloudflare and PDF button clicks)
- Work on mobile devices (requires desktop Chrome + shell)
- Access Sci-Hub reliably (mirrors are unstable)

This skill is optimized for: **macOS + Chrome + institutional journal access + WeChat-based paper discovery**.
