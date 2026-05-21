# conversiontools

Convert files between 140+ formats using ConversionTools. Two surfaces, same backend: the **`ctio` CLI** (preferred when shell access is available) and the **MCP server** (zero-install). The skill teaches Claude to pick the right one based on the environment.

Use when the user needs to convert documents (Word, PDF, Excel, PowerPoint), data formats (JSON, CSV, XML, YAML, Parquet), images (PNG, JPG, WebP, AVIF, HEIC, JXL, SVG), audio, video, e-books, OCR text extraction, AI-powered data extraction, text-to-speech, speech-to-text, subtitle conversion, or website screenshots.

## Installation

### Claude Code / Cowork

```bash
claude plugin marketplace add conversiontools/agent-skills
claude plugin install conversiontools@conversiontools-skills
```

### npx skills

```bash
npx skills add conversiontools/agent-skills --skill conversiontools
```

### ctio CLI (recommended companion for shell-driven workflows)

Full install instructions: <https://conversiontools.io/docs/ctio>. Quick Windows install:

```bash
scoop bucket add conversiontools https://github.com/conversiontools/scoop-bucket
scoop install ctio
```

---

> This plugin is auto-generated from [skills/conversiontools](../../skills/conversiontools).
