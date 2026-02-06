# `conversiontools/agent-skills`

**[Conversion Tools](https://conversiontools.io) agent skills** — convert 140+ file formats directly from AI agents.

## Installation

### Claude Code Plugin (Recommended)

Installs both the skill and the MCP server connection automatically:

```bash
claude plugin marketplace add conversiontools/agent-skills
claude plugin install conversiontools@conversiontools-skills
```

Restart Claude Code after installation. The MCP server connects automatically.

### Skills CLI (Other Agents)

Installs the skill instructions for Cursor, Gemini CLI, Amp, Codex, and other agents:

```bash
npx skills add conversiontools/agent-skills --skill conversiontools
```

MCP server must be configured separately for non-plugin agents. Connect to: `https://mcp.conversiontools.io/mcp`

---

| Skill | Description |
|-------|-------------|
| [conversiontools](https://github.com/conversiontools/agent-skills/tree/main/skills/conversiontools) | Convert files between 140+ formats using the ConversionTools MCP server. Supports documents, data formats, images, audio, video, e-books, OCR, AI extraction, subtitles, and website screenshots. |
| | ```claude plugin install conversiontools@conversiontools-skills``` |
| | ```npx skills add conversiontools/agent-skills --skill conversiontools``` |

---

## Supported Conversions

- **Documents**: Word, PowerPoint, Excel ↔ PDF, HTML, Text
- **Data**: JSON, CSV, XML, YAML, Excel conversions
- **Images**: PNG, JPG, WebP, AVIF, HEIC, SVG
- **PDF**: Extract to Word, Excel, CSV, Text, Images
- **Audio/Video**: MP3, WAV, MP4, MOV
- **E-books**: EPUB, MOBI, PDF
- **OCR**: Extract text from images and scanned PDFs
- **AI-powered**: Smart extraction from complex documents
- **Subtitles**: SRT, VTT conversions
- **Web**: Website screenshots to PDF, JPG, PNG

## Authentication

On first use, you'll be prompted to authenticate via OAuth in your browser. Free accounts get 100 conversions per month (10 per day). Paid plans available at [conversiontools.io/pricing](https://conversiontools.io/pricing).

## Links

- [Documentation](https://conversiontools.io/docs/mcp)
- [Pricing](https://conversiontools.io/pricing)
- [Support](https://conversiontools.io/contact)

## License

MIT
