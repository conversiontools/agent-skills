# `conversiontools/agent-skills`

**[Conversion Tools](https://conversiontools.io) agent skills** - convert 140+ file formats directly from AI agents.

Two surfaces, same backend: the **`ctio` CLI** for terminal-first agents (Claude Code, Cursor), and the **MCP server** for zero-install / web-based agents. The skill teaches Claude to pick the right one.

## Installation

### Claude Code Plugin (Recommended)

Installs the skill (which knows about both `ctio` CLI and MCP) and the MCP server connection automatically:

```bash
claude plugin marketplace add conversiontools/agent-skills
claude plugin install conversiontools@conversiontools-skills
```

Restart Claude Code after installation. The MCP server connects automatically.

### ctio CLI (best for shell-driven workflows)

Single-binary CLI, no Node/Bun runtime required. Pipes nicely with `jq`, handles arbitrarily large files via streaming, snappy 500 ms default polling.

**Windows (scoop):**

```bash
scoop bucket add conversiontools https://github.com/conversiontools/scoop-bucket
scoop install ctio
```

**macOS / Linux:** download from <https://github.com/conversiontools/ctio/releases> - one-line install commands for each platform in the [ctio docs](https://conversiontools.io/docs/ctio).

### Skills CLI (Other Agents)

Installs the skill instructions for Cursor, Gemini CLI, Amp, Codex, and other agents:

```bash
npx skills add conversiontools/agent-skills --skill conversiontools
```

MCP server must be configured separately for non-plugin agents. Connect to: `https://mcp.conversiontools.io/mcp`

---

| Skill | Description |
|-------|-------------|
| [conversiontools](https://github.com/conversiontools/agent-skills/tree/main/skills/conversiontools) | Convert files between 140+ formats using ConversionTools. Two surfaces - the `ctio` CLI (preferred when shell access is available) and the MCP server (zero-install). Supports documents, data formats (incl. Parquet), images (incl. JXL), audio, video, e-books, OCR, AI extraction, text-to-speech (TTS), speech-to-text (STT), subtitles (SRT, VTT, ASS), and website screenshots. |

```bash
claude plugin install conversiontools@conversiontools-skills
```

```bash
npx skills add conversiontools/agent-skills --skill conversiontools
```

---

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

- [ctio CLI docs](https://conversiontools.io/docs/ctio)
- [MCP server docs](https://conversiontools.io/docs/mcp)
- [Pricing](https://conversiontools.io/pricing)
- [Support](https://conversiontools.io/contact)

## License

MIT
