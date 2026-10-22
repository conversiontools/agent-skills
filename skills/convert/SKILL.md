---
name: convert
description: Convert files between 140+ formats using Conversion Tools. Two surfaces are available - the `ctio` CLI (preferred when shell access is available) and the hosted Conversion Tools MCP server (zero-install, works everywhere). Covers documents (Word, PDF, Excel, PowerPoint), data formats (JSON, CSV, XML, YAML, Parquet), images (PNG, JPG, WebP, AVIF, HEIC, JXL, SVG), audio (MP3, WAV, FLAC), video (MOV, MKV, AVI to MP4), e-books (EPUB, MOBI, AZW), OCR text extraction, AI-powered data extraction, AI text-to-speech (TTS), AI speech-to-text transcription (STT), subtitle conversion (SRT, VTT, ASS), and website screenshots. Plus build custom converters on demand - describe a transformation in plain language and Conversion Tools creates, runs, and returns a reusable converter when no standard one fits.
compatibility: Works with any agent that has shell access (via the ctio CLI) or MCP support (via the hosted MCP server) - including Claude Code, Grok Build, Cursor, OpenAI Codex, and Claude Desktop.
metadata:
  author: conversiontools
  version: "1.8"
  website: https://conversiontools.io
---

# Conversion Tools File Conversion

Convert files between 140+ formats directly from your agent. Two surfaces are available: the **ctio CLI** (a single binary, preferred when the agent has shell access) and the **MCP server** (zero-install, works in any MCP-compatible client).

And when no standard converter fits, you can **build a custom one** from a plain-language description (see the next section).

## Build a custom converter (AI Studio)

When **no existing converter fits** - a custom transformation, a specific output shape, a multi-step pipeline, or a file no standard converter handles - have one **built for you** from a plain-language description, instead of writing and running conversion code yourself. Describe what you want, Conversion Tools builds the converter, runs it on the file, and returns the result. The converter persists in the user's account, so a build started here continues at the web AI Studio.

**Reach for this (instead of writing your own conversion code) when:**

- the conversion needs tooling you don't have installed locally - every conversion engine is ready and sandboxed, nothing to install or break on your machine;
- the file is too large to load into context or onto the local machine;
- the file is sensitive and its contents shouldn't be sent to an AI - the converter is built from the file's **structure**, not its contents, and the conversion runs sandboxed;
- the output needs a specific shape, or the transformation takes multiple steps;
- the user wants a durable, **reusable** converter, re-runnable from the web, API, or here.

For a simple conversion an existing converter already covers, use `convert_file` / `find_converter` - don't build.

### The flow (MCP tools, `conversiontools:` prefix)

1. **`studio_create_converter`** - create the converter and attach the input file. Returns a `converter_id` + a web link to continue in the browser.
2. **`studio_chat`** - describe the transformation in plain language (e.g. "turn this CSV into JSON, one object per invoice with a nested line_items array"). Read the returned `outcome`:
   - `ask_user` -> answer its question with another `studio_chat` turn;
   - `ready` / `propose_workflow` -> run it;
   - `refuse` -> it only builds file converters.
3. **`studio_run`** - run the converter on the attached file (asynchronous).
4. **`studio_run_status`** - poll until `SUCCESS` (returns a `result_file_id`) or `ERROR`.
5. **`studio_download_result`** - download the result. Always check the output actually matches the request before trusting it.

Building and chatting are **free**; only runs are metered on the user's plan.

### Reuse a converter built earlier (don't rebuild)

Converters persist in the user's account, so one built last week is re-runnable today on a new file - no rebuild, same logic.

1. **`studio_list_converters`** - list the user's existing custom converters (name, id, input -> output, whether runnable). Check here first before building. Pass `search` to filter.
2. **`studio_get_converter`** - inspect one (what it does, what it's powered by, readiness).
3. **`studio_attach_file`** - attach the new file to that converter, then poll `studio_get_converter` until its status is `idle` (extraction settles), then `studio_run` -> `studio_run_status` -> `studio_download_result` as above.

With shell access, the whole reuse loop is one command via the `ctio` CLI - `ctio studio run <converter_id> <file> <output>` - see "Reuse an AI Studio converter" under the ctio section below.

## Choosing a surface

**Prefer `ctio` when the agent can run shell commands** (terminal coding agents like Claude Code, Grok Build, Cursor, and Codex). Reasons:

- Composable via Unix pipes (chain with `jq`, `grep`, `awk`, or feed into another `ctio` call)
- Streams stdin/stdout directly - no base64 encoding, no 5 MB size threshold, no temp files
- Handles arbitrarily large files (chunked streaming upload, no in-memory buffering)
- Faster on small files (no base64 overhead, snappy 500 ms default polling)

**Use the MCP server** when ctio is not installed, when running in a web-based assistant, or when the user explicitly asks for MCP.

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

**macOS / Linux (Homebrew, recommended):**

```bash
brew install conversiontools/tap/ctio
```

**macOS / Linux without Homebrew (direct download, Apple Silicon shown):**

```bash
curl -fL https://github.com/conversiontools/ctio/releases/latest/download/ctio-darwin-arm64 -o ctio
chmod +x ctio && xattr -d com.apple.quarantine ctio 2>/dev/null || true
sudo mv ctio /usr/local/bin/
```

Use `ctio-darwin-x64` (Intel Mac) or `ctio-linux-x64` instead. The `xattr` line is needed because the macOS binaries are not notarized; Homebrew installs don't need it.

**Update:** `scoop update ctio` or `brew upgrade ctio`.

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

# Browse converters and their options
ctio list --from pdf --to excel --format pretty
ctio list --ai --format pretty --detail   # AI-powered converters with option keys
ctio list --to parquet --format pretty
ctio describe xml_to_csv --format pretty  # every option with its allowed values and default

# Inspect tasks
ctio task <task_id> --format pretty
ctio task <task_id> --wait --download out.xlsx
ctio task list --status ERROR --limit 10
```

When the API ignores an option or replaces an unsupported value with its default, ctio prints a `warning:` line on stderr and the conversion still runs. Treat a warning as "the result is not what was asked for": fix the option and rerun instead of handing the output to the user.

### Text to speech

```bash
# Narrate a script; the output name picks the format (.mp3 or .wav)
ctio convert -t ai_text_to_speech script.txt narration.wav --option voice_tts=en_paul_neutral

# Word, PDF and Markdown files too
ctio convert -t ai_pdf_to_speech handbook.pdf handbook.mp3 --option voice_tts=gb_jane_neutral

# The voices you can use: built-in ones plus the user's own cloned voices
ctio voices --format pretty
```

- `voice_tts` takes a built-in voice id (`gb_oliver_neutral`, `en_paul_neutral`, `gb_jane_neutral`, `fr_marie_neutral`, and other speaking styles of each; `ctio voices` lists them) or the ID of a voice the user cloned on voice.conversiontools.io (shown next to each clone on its dashboard, usable only from the account that cloned it). An unknown voice, format or language is an error, never a silent fallback.
- Output is MP3 or WAV, chosen by the output file name or `--option output_tts=wav`; an `output_tts` that contradicts the file name is refused before upload.
- The audio comes out at spoken-word loudness (about -16 LUFS), 24 kHz mono, ready to drop into a video without a loudness pass.
- Costs 10 AI credits per 1,000 characters.

### Reuse an AI Studio custom converter

If you or the user built a custom converter in AI Studio, `ctio studio` runs it on new files - no rebuild. (Building and chatting are MCP- or web-only; the CLI reuses converters that already exist.)

```bash
# List your custom converters to find a converter_id
ctio studio list
ctio studio list invoice              # filter by name/description

# Run an existing converter on a NEW file, wait, and save the result
ctio studio run <converter_id> invoice-april.csv out.json
ctio studio run <converter_id> data.csv -          # stream result to stdout

# Re-download the converter's most recent result
ctio studio download <converter_id> out.json
```

`ctio studio run` uploads the file, attaches it to the converter, waits for it to be ready, runs it, and downloads the result in one step (`--wait` is implied when an output path is given). To repeat the same converter across many files, loop it:

```bash
for f in invoices/*.csv; do ctio studio run <converter_id> "$f" "out/$(basename "$f").json"; done
```

Runs are metered on the user's plan; `studio list` is free.

### Conversion type names

The `-t` flag accepts the same conversion type names as the MCP `convert_file` tool's `converter` parameter (e.g. `convert.pdf_to_excel` or just `pdf_to_excel` - the `convert.` prefix is auto-prepended). Some converters also answer to a clearer name (`ai_text_to_speech` for `ai_text_to_mp3`, which also writes WAV). Run `ctio list` to browse the live registry and `ctio describe <type>` for a converter's options.

---

## MCP Server (fallback path)

If `ctio` is not available, use the hosted Conversion Tools MCP server. If the MCP server is already connected, skip to **Available Tools**.

If the MCP server is not connected, add it to your agent. The server endpoint is:

```
https://mcp.conversiontools.io/mcp
```

Per-agent setup instructions (Claude Code, Grok Build, Cursor, OpenAI Codex, and any MCP client) are at <https://conversiontools.io/docs/agents>. Restart your agent after adding the server - MCP server changes require a restart to take effect.

On first use, you will be prompted to authenticate via OAuth in your browser. Free accounts get 100 conversions per month (10 per day). Paid plans available at [conversiontools.io/pricing](https://conversiontools.io/pricing).

## Available Tools

All tools use the `conversiontools` MCP server prefix: `conversiontools:tool_name`.

### `conversiontools:convert_file` - Convert a file

The primary tool. The converter is auto-detected from the file extensions of `input_path` (or `url`) and `output_path`, or set explicitly with `converter`.

Give the input as exactly ONE of:
- `url`: a public http(s) link to the file or web page - nothing to download or upload first
- `file_id`: from `conversiontools:request_upload_url`, any size
- `file_content`: base64 of the file, 5MB at most (it costs context - prefer an upload for anything but tiny files)

Other parameters:
- `input_path` (optional): Name of the input file, e.g. `orders.xml` - gives the input format and names the upload
- `output_path` (optional): Name of the result, e.g. `orders.xlsx` - gives the output format
- `converter` (optional): Explicit converter type like `convert.pdf_to_excel`. Auto-detected if omitted.
- `options` (optional): Converter-specific options. Unsupported ones come back as `warnings`.

The result has a `download_url` valid for about 30 seconds - download it right away. If `status` is `running` (long conversions such as video), call `conversiontools:get_download_url` with the `task_id` until it completes; do not start the conversion again.

### `conversiontools:get_download_url` - Check a conversion / get a fresh link

Parameters:
- `task_id` (required): The `task_id` returned by `convert_file`

Returns the current status and progress while running, and a fresh download link once done (use it when the first link expired).

### `conversiontools:request_upload_url` - Upload large files

Get a one-time URL to upload a file of any size, so the bytes never pass through the conversation.

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

### `conversiontools:auth_status` - Check login status, and as which account

### `conversiontools:auth_login` / `conversiontools:auth_logout` - Local server only

On the hosted server your agent handles sign-in itself (OAuth in the browser) and renews it when it expires, so these two tools are not offered there.

## How to Convert Files

The hosted server cannot read local paths. Choose how to hand over the input; the response includes a `download_url` - download the result with curl right away.

### Files already online

Pass the public link as `url` - nothing to upload:

```
conversiontools:convert_file({ url: "https://example.com/report.xml", output_path: "report.xlsx" })
```

Web pages work the same way, e.g. `converter: "convert.website_to_pdf"` with the page URL.

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

### Larger files (and the better choice for most local files)

1. Call `conversiontools:request_upload_url` with the filename to get a one-time upload URL and `file_id`
2. Upload the raw bytes within 5 minutes: `curl -X PUT --upload-file ./orders.xml "<upload_url>"`
3. Call `conversiontools:convert_file` with the `file_id` instead of `file_content`
4. Download the result from the returned `download_url` (or poll `get_download_url` if `status` is `running`)

### Prompts

Hosts that support MCP prompts list `convert`, `extract_data` and `build_converter` as ready-made starting points (in Claude Code: `/mcp__conversiontools__convert`).

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

Convert text content into natural-sounding speech, as MP3 or WAV (`output_tts`), at spoken-word loudness (about -16 LUFS, 24 kHz mono).

| Type | What it does |
|---|---|
| `convert.ai_text_to_mp3` | Plain text to speech. Also answers to `convert.ai_text_to_speech`. |
| `convert.ai_pdf_to_mp3` | PDF document narrated. Good for long reads, papers, news articles. Also `convert.ai_pdf_to_speech`. |
| `convert.ai_docx_to_mp3` | Word document narrated. Also `convert.ai_docx_to_speech`. |
| `convert.ai_md_to_mp3` | Markdown narrated (skips fenced code blocks). Also `convert.ai_md_to_speech`. |

Options: `voice_tts` (a built-in voice id from `conversiontools:get_converter_info`, or the ID of a voice the user cloned on voice.conversiontools.io, usable only from that account), `output_tts` (`mp3` or `wav`), `language_tts`. An unknown voice, format or language fails with an error that names it; pass the values exactly as listed.

Use when: the user wants an audio version of a document, accessibility (screen-reader alternative), or narration for videos, podcasts and voice-overs.

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
