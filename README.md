# CNS Paper Collector

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![GitHub stars](https://img.shields.io/github/stars/hmj238751-ui/cns-paper-collector)](https://github.com/hmj238751-ui/cns-paper-collector/stargazers)
[![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Linux-lightgrey)]()

A Claude Code skill for automated academic paper collection. Fetches WeChat articles, extracts metadata, downloads PDFs, renames files, and packages them into a ZIP — **~50 seconds for 4 papers**.

**You send a WeChat link. Claude handles the pipeline. You just click Download.**

---

## 📊 Workflow

```
  WeChat link / DOI / title
         │
         ▼
  ┌──────────────────────────────────────┐
  │  Phase 0: Metadata Extraction        │
  │  DOI scan → OCR screenshot → Crossref │
  └──────────────────┬───────────────────┘
                     ▼
  ┌──────────────────────────────────────┐
  │  Phase 1: PDF Download               │
  │  Cache → curl (OA) → Chrome (paywall)│
  └──────────────────┬───────────────────┘
                     ▼
  ┌──────────────────────────────────────┐
  │  Phase 2: Rename & Package           │
  │  YYYY-Journal-Title.pdf → ZIP → Desktop│
  └──────────────────────────────────────┘
```

---

## 🎯 Key Features

- **WeChat Article Parsing** — Dynamic browser rendering bypasses JS-only page shells
- **Multi-path Metadata Extraction** — DOI scanning > screenshot OCR > blockquote title > Crossref API
- **Smart Download Strategy** — Caches downloads, auto-fetches OA PDFs via curl, opens paywalled papers in Chrome
- **Anti-bot Awareness** — Publisher-specific strategies for Cloudflare, TLS fingerprinting, and captcha avoidance
- **Standardized Output** — All PDFs renamed to `YYYY-Journal-Title.pdf`, packaged into `SMOOTH_YYYYMMDD.zip`
- **Cross-platform** — macOS (native OCR), Linux (Tesseract), Windows (WSL)

---

## 📋 Prerequisites

| Requirement | Version | Check | Install |
|-------------|---------|-------|---------|
| Claude Code | latest | `claude --help` | [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code) |
| Python | 3.12+ | `python3 --version` | [python.org](https://www.python.org/) or Homebrew |
| scrapling | 0.2.99+ | `scrapling --help` | `pip install scrapling[all]` |
| Google Chrome | latest | `/Applications/Google Chrome.app` | [google.com/chrome](https://www.google.com/chrome/) |
| Playwright | latest | `playwright --version` | `playwright install chromium` |
| curl | any | `curl --version` | Built-in on macOS/Linux |
| Swift (OCR) | 5.9+ | `swift --version` | Built-in on macOS 13+. Linux: [Tesseract](#-linux-ocr) |

---

## 🔧 Environment Configuration

### Step 1: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
# or
brew install claude-code
```

### Step 2: Install scrapling + Playwright

**macOS:**
```bash
# Use Homebrew Python 3.12 (required by scrapling)
/opt/homebrew/opt/python@3.12/bin/python3.12 -m pip install --break-system-packages scrapling[all]

# Install Playwright browser (required for WeChat page rendering)
playwright install chromium

# Verify
scrapling --help
playwright --version
```

**Linux (Debian/Ubuntu):**
```bash
sudo apt install python3.12 python3.12-pip
pip install scrapling[all]
playwright install-deps chromium
playwright install chromium
scrapling --help
```

### Step 3: Install the skill

```bash
git clone https://github.com/hmj238751-ui/cns-paper-collector.git ~/.claude/skills/cns-paper-collector/
```

### Step 4: Verify everything

```bash
# Run these checks before first use
claude --help           && echo "✓ Claude Code"
scrapling --help        && echo "✓ scrapling"
playwright --version    && echo "✓ Playwright"
swift --version         && echo "✓ OCR engine"
curl --version          && echo "✓ curl"
ls ~/.claude/skills/cns-paper-collector/skill.md && echo "✓ Skill installed"
```

### Quick config check (one command)

```bash
claude --help > /dev/null 2>&1 && scrapling --help > /dev/null 2>&1 && echo "✅ Ready" || echo "❌ Missing dependencies — see setup above"
```

---

## 🚀 Quick Start

```bash
# Start Claude Code
claude

# Send a WeChat article — Claude handles the rest
> https://mp.weixin.qq.com/s/oB9qCK7bBLEY54JHt7KRDw

# Or batch multiple articles
> /cns-paper-collector https://mp.weixin.qq.com/s/... https://mp.weixin.qq.com/s/...
```

**What happens next:**
1. Claude fetches the article via scrapling dynamic browser
2. Extracts paper title, journal, DOI, and date
3. Downloads OA PDFs automatically; opens paywalled ones in Chrome
4. You click "Download PDF" for paywalled papers
5. Claude renames and packages everything → `SMOOTH_YYYYMMDD.zip` on your Desktop

---

## 💾 Installation (4 methods)

### Method 1: git clone (recommended)

```bash
git clone https://github.com/hmj238751-ui/cns-paper-collector.git ~/.claude/skills/cns-paper-collector/
```

### Method 2: Direct curl (no git required)

```bash
mkdir -p ~/.claude/skills/cns-paper-collector/
curl -sL -o ~/.claude/skills/cns-paper-collector/SKILL.md \
  https://raw.githubusercontent.com/hmj238751-ui/cns-paper-collector/main/SKILL.md
curl -sL -o ~/.claude/skills/cns-paper-collector/ocr_image.swift \
  https://raw.githubusercontent.com/hmj238751-ui/cns-paper-collector/main/ocr_image.swift
```

### Method 3: GitHub ZIP

Go to the [repo](https://github.com/hmj238751-ui/cns-paper-collector) → `Code` → `Download ZIP` → unzip to `~/.claude/skills/cns-paper-collector/`

### Method 4: One-liner (macOS)

```bash
mkdir -p ~/.claude/skills/cns-paper-collector/ && cd ~/.claude/skills/cns-paper-collector/ && curl -sLO https://raw.githubusercontent.com/hmj238751-ui/cns-paper-collector/main/SKILL.md && curl -sLO https://raw.githubusercontent.com/hmj238751-ui/cns-paper-collector/main/ocr_image.swift && echo "Done"
```

---

## 📖 Supported Publishers

| Publisher | Auto-download | Strategy |
|-----------|:---:|----------|
| Nature (flagship) OA | ✅ | `curl` with `Accept: application/pdf` header |
| Nature Communications | ✅ | Direct PDF URL |
| Scientific Reports | ✅ | Direct PDF URL |
| Nature paywalled | ❌ | Opens in Chrome (institutional login) |
| Science | ❌ | Opens in Chrome (Cloudflare) |
| **Cell Press** | ❌ | Cloudflare is impenetrable — always manual |
| bioRxiv / medRxiv | ✅ | No protection |
| Genome Biology | ✅ | BMC Open Access |
| Oxford Academic | ⚠️ | Try curl, fallback to Chrome |

---

## 🛠️ Platform-specific Notes

### macOS (fully supported)

All features work out of the box. OCR uses Apple Vision framework (built-in).

### Linux

OCR requires Tesseract instead of Swift:
```bash
sudo apt install tesseract-ocr tesseract-ocr-eng tesseract-ocr-chi-sim
tesseract screenshot.png stdout
```

### Windows

scrapling may require WSL or Git Bash. Path adjustments:
- `open -a "Google Chrome"` → `start chrome`
- `~/Desktop/` → `%USERPROFILE%\Desktop\`
- `~/Downloads/` → `%USERPROFILE%\Downloads\`

---

## ❓ Troubleshooting

| Problem | Solution |
|---------|----------|
| `scrapling: command not found` | Run `pip install scrapling[all]` (see [Environment Configuration](#-environment-configuration)) |
| `pip: externally-managed-environment` (macOS) | Add `--break-system-packages` flag |
| `No module named 'scrapling'` | Check Python version: `which python3` — must be 3.12+ |
| `BrowserType.launch: Executable doesn't exist` | Run `playwright install chromium` |
| WeChat captcha on fetch | Claude staggers requests by 2-3s. Increase stagger if still failing |
| curl returns 406 | Paper is paywalled — opens in Chrome instead |
| Cell.com PDF won't download | Expected. Cloudflare blocks all programmatic access. Click manually |
| OCR not working (Linux) | Install Tesseract. See [Platform Notes](#-platform-specific-notes) |
| ZIP not on Desktop | Check custom download paths. Default: `~/Downloads/` and `~/Desktop/` |

---

## 📝 Citation

If you use this skill in your research workflow:

```bibtex
@software{cns_paper_collector,
  author    = {hmj238751-ui},
  title     = {CNS Paper Collector: Claude Code skill for automated paper collection},
  year      = {2026},
  url       = {https://github.com/hmj238751-ui/cns-paper-collector}
}
```

---

## 🤝 Contributing

This skill was battle-tested over 30+ real-world paper collections across Nature, Science, Cell, and bioRxiv.

Found a publisher pattern that works? A new anti-bot bypass? A better OCR approach?

1. Fork the repository
2. Create a feature branch
3. Submit a PR

---

## 📧 Contact & Issues

- **Bug reports**: [GitHub Issues](https://github.com/hmj238751-ui/cns-paper-collector/issues)
- **Questions**: Open a Discussion on the repo

---

## 📄 License & Credits

This skill orchestrates several open-source tools. See [ATTRIBUTION.md](ATTRIBUTION.md) for full copyright details.

| Component | License |
|-----------|---------|
| scrapling | MIT |
| Playwright | Apache 2.0 |
| Apple Vision | Proprietary (macOS SDK) |
| Crossref API | Free (no key required) |
| curl | curl license |

**CNS Paper Collector**: MIT © [hmj238751-ui](https://github.com/hmj238751-ui)
