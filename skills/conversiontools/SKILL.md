---
name: conversiontools
description: Convert files between 140+ formats using the ConversionTools MCP server. Use when the user needs to convert documents (Word, PDF, Excel, PowerPoint), data formats (JSON, CSV, XML, YAML), images (PNG, JPG, WebP, AVIF, HEIC, SVG), audio (MP3, WAV, FLAC), video (MOV, MKV, AVI to MP4), e-books (EPUB, MOBI), OCR text extraction, AI-powered data extraction from PDFs and images, subtitle conversion, or website screenshots.
compatibility: Requires connection to the ConversionTools MCP server. Works with Claude Code, Claude Desktop, and any MCP-compatible agent.
metadata:
  author: conversiontools
  version: "1.0"
  website: https://conversiontools.io
---

# ConversionTools File Conversion

Convert files between 140+ formats directly from your agent using the ConversionTools MCP server.

## Setup

This skill automatically connects to the ConversionTools MCP server when the plugin is enabled. No manual MCP configuration required.

### Plugin Install (Recommended)

```bash
claude plugin install conversiontools
```

Restart Claude Code after installation. The MCP server connection is configured automatically.

### Manual MCP Setup (Without Plugin)

If you prefer to configure the MCP server manually:

```bash
claude mcp add --transport http conversiontools https://mcp.conversiontools.io/mcp
```

On first use, you will be prompted to authenticate via OAuth in your browser. Free accounts get 100 conversions per month (10 per day). Paid plans available at [conversiontools.io/pricing](https://conversiontools.io/pricing).

## Available Tools

All tools use the `conversiontools` MCP server prefix: `conversiontools:tool_name`.

### `conversiontools:convert_file` — Convert a file

The primary tool. Provide input and output paths — the converter is auto-detected from file extensions.

Parameters:
- `input_path` (required): Path to the input file
- `output_path` (required): Path for the converted output file
- `file_content` (optional): Base64-encoded file content for files up to 5MB
- `file_id` (optional): File ID from a previous `conversiontools:request_upload_url` call
- `converter` (optional): Explicit converter type like `convert.pdf_to_excel`. Auto-detected if omitted.
- `options` (optional): Converter-specific options

### `conversiontools:request_upload_url` — Upload large files

Get a signed URL for uploading files larger than 5MB.

Parameters:
- `filename` (required): Name of the file to upload

Flow for large files:
1. Call `conversiontools:request_upload_url` with the filename
2. Upload file to the returned URL using PUT
3. Call `conversiontools:convert_file` with the returned `file_id`

### `conversiontools:list_converters` — Browse available converters

Parameters:
- `category` (optional): `documents`, `data`, `images`, `pdf`, `audio`, `video`, `ebook`, `ocr`, `ai`, `subtitles`, `web`
- `input_format` (optional): Filter by input format, e.g. `pdf`
- `output_format` (optional): Filter by output format, e.g. `csv`

### `conversiontools:find_converter` — Find the right converter

Parameters:
- `input_format` (required): e.g. `pdf`
- `output_format` (required): e.g. `excel`

### `conversiontools:get_converter_info` — Get converter details

Parameters:
- `converter` (required): Converter type, e.g. `convert.pdf_to_excel`

### `conversiontools:auth_status` — Check login status

### `conversiontools:auth_login` — Trigger OAuth login

### `conversiontools:auth_logout` — Clear credentials

## How to Convert Files

1. Look at the input file extension and the desired output format
2. Call `conversiontools:convert_file` with `input_path` and `output_path` — the converter is auto-detected
3. If auto-detection fails, use `conversiontools:find_converter` to look up the right converter, then pass it explicitly

Example — convert a PDF to Excel:

```
conversiontools:convert_file({
  input_path: "report.pdf",
  output_path: "report.xlsx"
})
```

Example — convert with explicit converter:

```
conversiontools:convert_file({
  input_path: "data.json",
  output_path: "data.csv",
  converter: "convert.json_to_csv"
})
```

## Supported Conversions

### Documents
- Word (docx, doc) → PDF, HTML, Text
- PowerPoint (pptx, ppt) → PDF
- Excel (xlsx, xls) → PDF, CSV, HTML, JSON, XML
- Markdown → PDF, HTML, EPUB

### PDF
- PDF → Word, Excel, CSV, Text, HTML, JPG, PNG, EPUB
- JPG/PNG → PDF

### Data Formats
- JSON ↔ CSV, Excel, XML, YAML
- CSV ↔ JSON, XML, Excel
- XML ↔ JSON, CSV, Excel
- YAML → JSON

### Images
- PNG ↔ JPG, WebP, AVIF, SVG
- JPG ↔ PNG, WebP, AVIF
- WebP ↔ PNG, JPG
- AVIF ↔ PNG, JPG
- HEIC → JPG, PNG
- SVG → PNG, JPG
- Remove EXIF metadata

### Audio
- WAV ↔ MP3
- FLAC → MP3, WAV

### Video
- MOV, MKV, AVI → MP4
- MP4 → MP3 (extract audio)

### E-books
- EPUB ↔ MOBI
- EPUB, MOBI → PDF
- Markdown → EPUB

### OCR (Text Recognition)
- PNG, JPG → Text (OCR)
- Scanned PDF → Text (OCR)
- PNG, JPG → Searchable PDF (OCR)

### AI-Powered
- PDF → JSON, CSV, Excel, Markdown (AI extraction)
- Images → JSON (AI extraction)
- Subtitles → Translated subtitles (AI)

### Subtitles
- SRT ↔ VTT
- SRT → CSV, Text

### Web
- Website URL → PDF, JPG, PNG (screenshot)
- HTML → JPG, PNG
- HTML tables → CSV

## Tips

- Auto-detection works well — just provide file paths with correct extensions and skip the `converter` parameter
- Use `conversiontools:list_converters` with filters when unsure what's available: `conversiontools:list_converters({ input_format: "pdf" })` shows all PDF output options
- AI converters (`convert.ai_pdf_to_json`, etc.) use AI to intelligently extract structured data from complex documents — use these when standard converters produce poor results
- OCR converters are for scanned documents and images containing text — use when the source is an image or non-searchable PDF
- For batch conversions, upload a ZIP, 7z, XZ, or RAR archive containing all files — they will all be converted and returned as a `.result.zip` file. Alternatively, call `conversiontools:convert_file` once per file.
