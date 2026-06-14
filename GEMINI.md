# Conversion Tools Gemini Extension

This extension converts files between 140+ formats using Conversion Tools, via the hosted MCP server (`https://mcp.conversiontools.io/mcp`).

## Authentication

On first use you will be prompted to authenticate via OAuth - a browser window opens to log in to your Conversion Tools account. Free accounts get 100 conversions per month (10 per day); paid plans are at https://conversiontools.io/pricing.

## Usage

Ask Gemini to convert a file and it will pick the right converter automatically:

- "Convert data.json to Excel"
- "Extract the table from invoice.pdf to CSV"
- "Transcribe meeting.mp3 to text"

For files over 5 MB, the agent uses `request_upload_url` first, then `convert_file` with the returned `file_id`.

## More

If the agent also has shell access, the `ctio` CLI is faster for batch and large-file workflows. See https://conversiontools.io/docs/agents for every Conversion Tools agent integration (Claude Code, Cursor, Codex, Grok, Gemini, and the ctio CLI).
