---
name: conversiontools
description: Convert files between 140+ formats using ConversionTools. Two surfaces are available - the `ctio` CLI (preferred when shell access is available) and the ConversionTools MCP server (zero-install, works everywhere). Covers documents (Word, PDF, Excel, PowerPoint), data formats (JSON, CSV, XML, YAML, Parquet), images (PNG, JPG, WebP, AVIF, HEIC, JXL, SVG), audio (MP3, WAV, FLAC), video (MOV, MKV, AVI to MP4), e-books (EPUB, MOBI, AZW), OCR text extraction, AI-powered data extraction, AI text-to-speech (TTS), AI speech-to-text transcription (STT), subtitle conversion (SRT, VTT, ASS), and website screenshots.
compatibility: Works with Claude Code (via ctio binary or MCP server), Claude Desktop (MCP), and any MCP-compatible agent.
metadata:
  author: conversiontools
  version: "1.2"
  website: https://conversiontools.io
---

# ConversionTools File Conversion

Convert files between 140+ formats directly from your agent. Two surfaces are available: the **ctio CLI** (a single binary, preferred when the agent has shell access) and the **MCP server** (zero-install, works in any MCP-compatible client).

## Choosing a surface

**Prefer `ctio` when the agent can run shell commands** (Claude Code, terminal agents). Reasons:

- Composable via Unix pipes (chain with `jq`, `grep`, `awk`, or feed into another `ctio` call)
- Streams stdin/stdout directly - no base64 encoding, no 5 MB size threshold, no temp files
- Handles arbitrarily large files (chunked streaming upload, no in-memory buffering)
- Faster on small files (no base64 overhead, snappy 500 ms default polling)

**Use the MCP server** when ctio is not installed, when running in a web-only environment (Claude Desktop, Claude.ai), or when the user explicitly asks for MCP.

Quick capability check:

```bash
which ctio || echo "ctio not installed - use MCP tools below"
```

## ctio CLI

Full docs: <https://conversiontools.io/docs/ctio>. Releases: <https://github.com/conversiontools/ctio/releases>.

### Install (one time)

**Windows (scoop, recommended):**

```bash
scoop bucket add conversiontools https://github.com/conversiontools/scoop-bucket
scoop install ctio
```

**macOS (Apple Silicon):**

```bash
curl -fL https://github.com/conversiontools/ctio/releases/latest/download/ctio-darwin-arm64 -o ctio
chmod +x ctio && xattr -d com.apple.quarantine ctio 2>/dev/null || true
sudo mv ctio /usr/local/bin/
```

**macOS (Intel) / Linux:** same as above with `ctio-darwin-x64` or `ctio-linux-x64`.

### Authenticate (one time)

```bash
ctio auth login
```

Paste an API token from <https://conversiontools.io/profile>. Stored at `~/.config/conversiontools/profiles.toml` (Unix, 0600) or `%APPDATA%\conversiontools\profiles.toml` (Windows). Free tier: 100 conversions/month (10/day). Paid plans at <https://conversiontools.io/pricing>.

### Convert (the verbs)

```bash
# File to file
ctio convert -t json_to_excel data.json out.xlsx

# stdin to file (no temp file, true streaming upload)
cat data.json | ctio convert -t json_to_excel - out.xlsx

# File to stdout (status JSON goes to stderr so the pipe stays clean)
ctio convert -t pdf_to_text invoice.pdf -

# Compose with other tools
ctio convert -t pdf_to_text invoice.pdf - | grep -i total

# Remote URL input (no local file needed)
ctio convert -t website_to_pdf --url https://example.com out.pdf

# Conversion options (repeatable)
ctio convert -t excel_to_xml in.xlsx out.xml --option header=yes --option show_columns=no

# Browse converters
ctio list --from pdf --to excel --format pretty
ctio list --ai --format pretty --detail   # AI-powered converters with option keys
ctio list --to parquet --format pretty

# Inspect tasks
ctio task <task_id> --format pretty
ctio task <task_id> --wait --download out.xlsx
ctio task list --status ERROR --limit 10
```

### Conversion type names

The `-t` flag accepts the same conversion type names as the MCP `convert_file` tool's `converter` parameter (e.g. `convert.pdf_to_excel` or just `pdf_to_excel` - the `convert.` prefix is auto-prepended). Run `ctio list` to browse the live registry.

---

## MCP Server (fallback path)

If `ctio` is not available, use the ConversionTools MCP server. If the MCP server is already connected, skip to **Available Tools**.

If the MCP server is not connected, add it:

```bash
claude mcp add --transport http conversiontools https://mcp.conversiontools.io/mcp
```

Restart Claude Code after adding the server. MCP server changes require a restart to take effect.

On first use, you will be prompted to authenticate via OAuth in your browser. Free accounts get 100 conversions per month (10 per day). Paid plans available at [conversiontools.io/pricing](https://conversiontools.io/pricing).

## Available Tools

All tools use the `conversiontools` MCP server prefix: `conversiontools:tool_name`.

### `conversiontools:convert_file` - Convert a file

The primary tool. Provide input and output paths - the converter is auto-detected from file extensions.

Parameters:
- `input_path` (required): Path to the input file
- `output_path` (required): Path for the converted output file
- `file_content` (optional): Base64-encoded file content for files up to 5MB
- `file_id` (optional): File ID from a previous `conversiontools:request_upload_url` call
- `converter` (optional): Explicit converter type like `convert.pdf_to_excel`. Auto-detected if omitted.
- `options` (optional): Converter-specific options

### `conversiontools:request_upload_url` - Upload large files

Get a signed URL for uploading files larger than 5MB.

Parameters:
- `filename` (required): Name of the file to upload

Flow for large files:
1. Call `conversiontools:request_upload_url` with the filename
2. Upload file to the returned URL using PUT
3. Call `conversiontools:convert_file` with the returned `file_id`

### `conversiontools:list_converters` - Browse available converters

Use this to discover what conversions are currently supported. New converters are added regularly - always check here for the latest.

Parameters:
- `category` (optional): `documents`, `data`, `images`, `pdf`, `audio`, `video`, `ebook`, `ocr`, `ai`, `subtitles`, `web`
- `input_format` (optional): Filter by input format, e.g. `pdf`
- `output_format` (optional): Filter by output format, e.g. `csv`

### `conversiontools:find_converter` - Find the right converter

Parameters:
- `input_format` (required): e.g. `pdf`
- `output_format` (required): e.g. `excel`

### `conversiontools:get_converter_info` - Get converter details

Parameters:
- `converter` (required): Converter type, e.g. `convert.pdf_to_excel`

### `conversiontools:auth_status` - Check login status

### `conversiontools:auth_login` - Trigger OAuth login

### `conversiontools:auth_logout` - Clear credentials

## How to Convert Files

You must provide the file content - the server cannot read local paths. Choose the method based on file size. The response includes a `download_url` - download the result with curl.

### Small files (up to 5MB)

1. Base64-encode the file
2. Call `conversiontools:convert_file` with `file_content`, `input_path`, and `output_path`
3. Download the result from the returned `download_url`

```bash
# 1. Encode
base64_content=$(base64 -w 0 data.json)
```

```
# 2. Convert
conversiontools:convert_file({
  input_path: "data.json",
  output_path: "data.csv",
  file_content: "<base64_content>"
})
```

```bash
# 3. Download
curl -sL "<download_url>" -o data.csv
```

### Large files (over 5MB)

1. Call `conversiontools:request_upload_url` with the filename to get a signed upload URL and `file_id`
2. Upload the file to the returned URL with `curl -X PUT`
3. Call `conversiontools:convert_file` with the `file_id` instead of `file_content`
4. Download the result from the returned `download_url`

## Supported Categories

205 converters across the categories below (snapshot, regenerated from the live registry). Use `conversiontools:list_converters` with a `category`, `input_format`, or `output_format` filter to discover specific converters - new formats are added regularly.

- **Office documents** (17) - Word (DOCX, DOC), PowerPoint (PPTX, PPT), Excel (XLSX, XLS), ODS, XML to/from PDF, HTML, CSV, JSON, ODS, Excel-XML/XLSX
- **PDF** (15) - PDF to/from Word, Excel, CSV, HTML, Text, JPG, PNG, SVG, TIFF; plus OXPS to PDF, PDF to XPS, Website to PDF
- **Data formats - JSON family** (10) - JSON, JSON Objects, XML, YAML to/from CSV, Excel, JSON, XML, YAML
- **Data formats - CSV** (6) - CSV from BSON, RSS, Sitemap, XML; CSV to JSON, XML
- **Data formats - XML** (2) - Validators and formatters
- **Data formats - JSONL** (4) - JSONL to/from CSV, Excel
- **Data formats - Parquet** (9) - Parquet to/from CSV, Excel, JSON, JSONL, XML
- **Web feeds to Excel** (3) - BSON, RSS, Sitemap to Excel
- **Images** (57) - PNG, JPG, WebP, AVIF, HEIC, JXL (JPEG XL), SVG, BMP, TIFF, GIF mutual conversion; plus output to PAM, PGM, PPM, YUV; plus EXIF removal
- **E-books** (18) - EPUB, MOBI, AZW, AZW3, FB2, FBZ, PDF mutual conversion
- **Audio** (5) - MP3, WAV, FLAC mutual conversion
- **Video** (4) - MP4, MOV, MKV, AVI to MP4; plus audio extraction to MP3
- **Subtitles** (19) - SRT, VTT, ASS, CSV, Excel (XLSX, XLS), Text bidirectional
- **Markdown** (8) - Markdown to/from PDF, DOCX, HTML, EPUB, LaTeX, Text
- **HTML / Web** (8) - URL/HTML to PDF, JPG, PNG, Word, CSV; HTML metadata extraction; HTML table to CSV; website screenshots
- **OCR** (6) - Extract text from JPG, PNG, scanned PDFs; output as Text or searchable PDF
- **AI-powered** (14) - see the expanded section below

### AI-powered converters (expanded)

These are the highest-leverage converters in the catalog. When standard converters produce poor results, when the input is messy or unstructured, or when the user wants natural-language output, reach for these. Type names use the `convert.ai_*` prefix.

#### Document data extraction

Pull structured data out of complex documents and images. AI handles layout drift, tables across page breaks, handwritten fields, and mixed languages where pdf_to_csv / pdf_to_excel struggle.

| Type | What it does |
|---|---|
| `convert.ai_pdf_to_json` | Extract structured data from PDF as JSON. Best for invoices, receipts, forms, statements. |
| `convert.ai_pdf_to_csv` | Same as above but flattened to CSV rows. |
| `convert.ai_pdf_to_excel` | Same as above as an Excel workbook (preserves multi-table structure). |
| `convert.ai_pdf_to_markdown` | Convert PDF to Markdown with formatting preserved (headings, lists, tables). Good for converting reports/papers into LLM-friendly Markdown. |
| `convert.ai_jpg_to_json` | Extract structured data from a JPG (photographed receipts, screenshots of tables, etc.). |
| `convert.ai_png_to_json` | Same for PNG. |

Use when: standard pdf_to_csv/excel returns garbled output, the document has nested tables, OCR alone is not enough, or the user wants typed fields (dates, amounts, IDs) rather than raw text.

#### AI Text-to-Speech (TTS)

Convert text content into natural-sounding MP3. Supports multiple languages and voices.

| Type | What it does |
|---|---|
| `convert.ai_text_to_mp3` | Plain text to MP3 audio. |
| `convert.ai_pdf_to_mp3` | PDF document narrated as MP3. Good for long reads, papers, news articles. |
| `convert.ai_docx_to_mp3` | Word document narrated as MP3. |
| `convert.ai_md_to_mp3` | Markdown narrated as MP3 (skips fenced code blocks). |

Use when: the user wants an audio version of a document, accessibility (screen-reader alternative), or content for podcast/voice-over workflows.

#### AI Speech-to-Text (STT)

Transcribe audio recordings to text. Fast and accurate, multi-language.

| Type | What it does |
|---|---|
| `convert.ai_mp3_to_text` | Transcribe MP3 audio to plain text. |
| `convert.ai_wav_to_text` | Transcribe WAV audio to plain text. |
| `convert.ai_flac_to_text` | Transcribe FLAC audio to plain text. |

Use when: the user has a recording (meeting, interview, voicemail, lecture) and wants the text out. Output is a single text file with the full transcript.

#### AI Subtitle Translation

| Type | What it does |
|---|---|
| `convert.ai_translate_srt` | Translate an SRT subtitle file into another language while preserving timestamps. Specify the target language via the `language` option. |

Use when: user has subtitles in one language and needs them in another. Faster and cheaper than human translation, good enough for most viewing use cases.

## Discovering Converters

When unsure if a specific conversion is supported, use the discovery tools rather than guessing:

```
# Check what PDF can convert to
conversiontools:list_converters({ input_format: "pdf" })

# Check what can produce Parquet
conversiontools:list_converters({ output_format: "parquet" })

# Browse all data converters
conversiontools:list_converters({ category: "data" })

# Find the right converter for a specific pair
conversiontools:find_converter({ input_format: "xlsx", output_format: "parquet" })

# Get full details and options for a converter
conversiontools:get_converter_info({ converter: "convert.csv_to_parquet" })
```

## Tips

- Auto-detection works well - just provide file paths with correct extensions and skip the `converter` parameter
- Use `conversiontools:list_converters` with filters when unsure what's available
- AI converters (`convert.ai_pdf_to_json`, etc.) use AI to intelligently extract structured data from complex documents - use these when standard converters produce poor results
- OCR converters are for scanned documents and images containing text - use when the source is an image or non-searchable PDF
- For batch conversions, upload a ZIP, 7z, XZ, or RAR archive containing all files - they will all be converted and returned as a `.result.zip` file. Alternatively, call `conversiontools:convert_file` once per file.
