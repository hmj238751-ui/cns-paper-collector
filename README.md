# CNS Paper Collector

A Claude Code skill that automates academic paper collection: fetches WeChat articles, extracts paper metadata, opens PDF download pages, renames files, and packages them into a ZIP — all from a single command.

**You send a WeChat link. Claude does the rest. You just click Download.**

---

## What it does

```
WeChat article / DOI / title → Claude extracts metadata → opens PDF in your browser → you click Download → ZIP on your Desktop
```

- Reads WeChat official account articles (mp.weixin.qq.com) via dynamic browser
- Extracts paper titles, DOIs, journals, and dates from article text + journal screenshots (OCR)
- Downloads Open Access PDFs automatically (curl)
- Opens paywalled PDF pages in a new Chrome window for you to download
- Renames PDFs to a standardized format: `YYYY-Journal-Title.pdf`
- Packages everything into `SMOOTH_YYYYMMDD.zip` on your Desktop

**~50 seconds for 4 papers. You only click the download button.**

---

## Prerequisites

| Requirement | How to check | How to install |
|-------------|-------------|----------------|
| **Claude Code** | Run `claude --help` | [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code) |
| **scrapling** | Run `scrapling --help` | `pip install scrapling[all]` |
| **Google Chrome** | macOS: `/Applications/Google Chrome.app` | [google.com/chrome](https://www.google.com/chrome/) |
| **curl** | Run `curl --version` | Pre-installed on macOS/Linux |
| **macOS 13+** (OCR) | Run `swift --version` | Built-in on macOS. Linux users: see [Linux OCR](#linux-ocr) |

---

## Installation

### Method 1: git clone (recommended)

```bash
git clone https://github.com/YOUR_USERNAME/cns-paper-collector.git ~/.claude/skills/cns-paper-collector/
```

Then tell Claude: `请用 cns-paper-collector 这个 skill。`

### Method 2: Direct download (if you can't use git)

```bash
# Download the skill file directly
mkdir -p ~/.claude/skills/cns-paper-collector/
curl -sL -o ~/.claude/skills/cns-paper-collector/SKILL.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/cns-paper-collector/main/SKILL.md
curl -sL -o ~/.claude/skills/cns-paper-collector/ocr_image.swift \
  https://raw.githubusercontent.com/YOUR_USERNAME/cns-paper-collector/main/ocr_image.swift
```

### Method 3: GitHub ZIP download

1. Go to `https://github.com/YOUR_USERNAME/cns-paper-collector`
2. Click the green `Code` button → `Download ZIP`
3. Unzip to `~/.claude/skills/cns-paper-collector/`

### Method 4: One-liner (macOS)

```bash
mkdir -p ~/.claude/skills/cns-paper-collector/ && cd ~/.claude/skills/cns-paper-collector/ && curl -sLO https://raw.githubusercontent.com/YOUR_USERNAME/cns-paper-collector/main/SKILL.md && curl -sLO https://raw.githubusercontent.com/YOUR_USERNAME/cns-paper-collector/main/ocr_image.swift && echo "Done. Tell Claude: /cns-paper-collector"
```

---

## Usage

Start a Claude Code session and send a WeChat article link:

```
claude
> https://mp.weixin.qq.com/s/oB9qCK7bBLEY54JHt7KRDw
```

Or use the slash command:

```
/cns-paper-collector https://mp.weixin.qq.com/s/... https://mp.weixin.qq.com/s/...
```

Claude will:
1. Fetch the article content
2. Extract the paper title, journal, and DOI
3. Open the PDF download page in Chrome
4. Ask you to click "Download PDF"
5. Rename and package everything into a ZIP on your Desktop

---

## Supported publishers

| Publisher | Auto-download | Notes |
|-----------|:---:|-------|
| Nature (flagship) OA | Yes | curl with `Accept: application/pdf` |
| Nature Communications | Yes | Open Access |
| Scientific Reports | Yes | Open Access |
| Nature paywalled | No | Opens in Chrome for institutional login |
| Science | No | Opens in Chrome after Cloudflare |
| **Cell Press** | **No** | Cloudflare is impenetrable — always manual |
| bioRxiv / medRxiv | Yes | No protection |
| Genome Biology | Yes | BMC Open Access |
| Oxford Academic | Sometimes | Try curl, fallback to Chrome |

---

## Skill architecture

```
Input (WeChat/DOI/title)
  │
  ├─ Phase 0: Metadata extraction
  │   ├─ DOI embedded in URL → extract directly
  │   ├─ WeChat article → scrapling dynamic browser → scan DOI / OCR screenshot
  │   └─ Title only → batch Crossref API lookup
  │
  ├─ Phase 1: PDF download
  │   ├─ Cache check → skip if already downloaded
  │   ├─ curl (OA journals) → validate %PDF- header
  │   ├─ Chrome new window (paywalled) → user clicks download
  │   └─ Give up (Cell.com)
  │
  └─ Phase 2: Rename & package
      └─ YYYY-Journal-Title.pdf → SMOOTH_YYYYMMDD.zip → Desktop
```

---

## Linux OCR

The macOS OCR script (`ocr_image.swift`) uses Apple Vision framework. Linux users should install `tesseract` instead:

```bash
sudo apt install tesseract-ocr tesseract-ocr-eng tesseract-ocr-chi-sim
tesseract screenshot.png stdout
```

Tell Claude to use `tesseract` instead of `swift ocr_image.swift` when on Linux.

---

## Windows notes

- Replace `open -a "Google Chrome"` with `start chrome`
- Replace `~/Desktop/` with `%USERPROFILE%\Desktop\`
- Replace `~/Downloads/` with `%USERPROFILE%\Downloads\`
- scrapling may require WSL or Git Bash (not tested on native Windows)
- The OCR script is macOS-only; Windows users need `tesseract` via WSL or skip OCR

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "scrapling: command not found" | `pip install scrapling[all]` |
| WeChat captcha on fetch | Claude staggers requests by 2-3s. If still failing, increase stagger. |
| curl returns 406 | Paper is paywalled. Opens in Chrome instead. |
| Cell paper can't download | Expected. Cell.com blocks all programmatic access. Open in Chrome. |
| OCR not working (Linux) | Install tesseract. See [Linux OCR](#linux-ocr). |
| ZIP not on Desktop | Check custom download paths. The skill uses `~/Downloads/` and `~/Desktop/`. |

---

## Contributing

This skill was originally built and battle-tested over ~30 real-world paper collections across Nature, Science, Cell, and bioRxiv.

Found a publisher pattern that works? A new anti-bot bypass? Open an issue or PR.

---

## License

MIT
