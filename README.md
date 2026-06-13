# `conversiontools/agent-skills`

**[Conversion Tools](https://conversiontools.io) agent skills** - convert 140+ file formats directly from AI agents.

Two surfaces, same backend: the **`ctio` CLI** for terminal-first agents, and the hosted **MCP server** for zero-install / web-based agents. The skill teaches your agent to pick the right one.

One repo, every agent. Full per-agent setup is at <https://conversiontools.io/docs/agents>.

## Installation

### Claude Code

```bash
claude plugin marketplace add conversiontools/agent-skills
claude plugin install conversiontools@conversiontools-skills
```

Restart Claude Code after installation. The MCP server connects automatically.

### Cursor

```bash
/add-plugin conversiontools
```

Or add the repo as a plugin source, then install `conversiontools` from the plugins browser.

### OpenAI Codex

```bash
codex plugin marketplace add conversiontools/agent-skills
```

Start Codex, open the plugins browser with `/plugins`, and install the `conversiontools` plugin.

### Grok Build

Once listed in the Grok marketplace, run `/marketplace` inside Grok Build and install `conversiontools`. In the meantime, add the MCP server with `grok mcp add conversiontools --url https://mcp.conversiontools.io/mcp`.

### ctio CLI (best for shell-driven workflows)

Single-binary CLI, no Node/Bun runtime required. Pipes nicely with `jq`, handles arbitrarily large files via streaming.

**Windows (scoop):**

```bash
scoop bucket add conversiontools https://github.com/conversiontools/scoop-bucket
scoop install ctio
```

**macOS / Linux:** download from <https://github.com/conversiontools/ctio/releases> - one-line install commands for each platform are in the [ctio docs](https://conversiontools.io/docs/ctio).

### Any MCP client

Connect to the hosted MCP server:

```
https://mcp.conversiontools.io/mcp
```

---

| Skill | Description |
|-------|-------------|
| [convert](https://github.com/conversiontools/agent-skills/tree/main/skills/convert) | Convert files between 140+ formats using Conversion Tools. Two surfaces - the `ctio` CLI (preferred when shell access is available) and the hosted MCP server (zero-install). Supports documents, data formats (incl. Parquet), images (incl. JXL), audio, video, e-books, OCR, AI extraction, text-to-speech (TTS), speech-to-text (STT), subtitles (SRT, VTT, ASS), and website screenshots. |

## Supported Conversions

- **Documents**: Word, PowerPoint, Excel, Markdown, ODS to/from PDF, HTML, Text, LaTeX
- **Data**: JSON, CSV, XML, YAML, Parquet, JSONL, BSON, Excel - with validators and formatters
- **Images**: PNG, JPG, WebP, AVIF, HEIC, JXL (JPEG XL), SVG, BMP, TIFF, GIF
- **PDF**: Extract to Word, Excel, CSV, Text, HTML, Images (JPG, PNG, SVG, TIFF), EPUB
- **Audio**: MP3, WAV, FLAC - plus AI text-to-speech and speech-to-text
- **Video**: MP4, MOV, MKV, AVI - plus audio extraction
- **E-books**: EPUB, MOBI, AZW, AZW3, FB2, FBZ, PDF
- **OCR**: Extract text from images and scanned PDFs - output as Text or searchable PDF
- **AI-powered**: Smart extraction from complex documents, subtitle translation, TTS, STT
- **Subtitles**: SRT, VTT, ASS - bidirectional conversions
- **Web**: Website screenshots to PDF, JPG, PNG

## Authentication

For the MCP server, you'll be prompted to authenticate via OAuth in your browser on first use. For the `ctio` CLI, run `ctio auth login` once and paste an API token from [conversiontools.io/profile](https://conversiontools.io/profile). Free accounts get 100 conversions per month (10 per day). Paid plans available at [conversiontools.io/pricing](https://conversiontools.io/pricing).

## Links

- [All agent integrations](https://conversiontools.io/docs/agents)
- [ctio CLI docs](https://conversiontools.io/docs/ctio)
- [MCP server docs](https://conversiontools.io/docs/mcp)
- [Pricing](https://conversiontools.io/pricing)
- [Support](https://conversiontools.io/contact)

## License

MIT
