# Conversion Tools Gemini Extension

This extension converts files between 140+ formats using Conversion Tools, via the hosted MCP server (`https://mcp.conversiontools.io/mcp`). It can also **build a custom converter** from a plain-language description when no standard converter fits.

## Authentication

On first use you will be prompted to authenticate via OAuth - a browser window opens to log in to your Conversion Tools account. Free accounts get 100 conversions per month (10 per day); paid plans are at https://conversiontools.io/pricing.

## Usage

Ask Gemini to convert a file and it will pick the right converter automatically:

- "Convert data.json to Excel"
- "Extract the table from invoice.pdf to CSV"
- "Transcribe meeting.mp3 to text"

When no standard converter fits, ask Gemini to **build** one: "Build a converter that turns this CSV into JSON, one object per invoice with a nested line_items array." The `studio_*` tools create the converter, run it, and return the result - it persists in your account to continue on the web. Because it persists, you can **reuse** it later on a new file (`studio_list_converters` to find it, `studio_attach_file` to point it at the new file, then run) - no rebuild.

To pull **named fields out of documents** (invoices, receipts, purchase orders, statements, forms, scans), the `parse_*` tools drive Parse: "Extract invoice number, date, total and line items from invoice.pdf." Submission is asynchronous by design - `parse_extract` returns an extraction id, then `parse_extraction_status` is polled until the extraction is `completed` or `failed`, and `parse_export` turns the result into CSV or XLSX. See https://parse.conversiontools.io/docs.

For files up to 5 MB the agent passes the file base64-encoded as `file_content` - it reads the file directly and encodes it in-process, rather than relying on a shell base64 one-liner (some agent sandboxes block those as "command substitution"). For files over 5 MB it uses `request_upload_url` first, then `convert_file` with the returned `file_id`.

## More

If the agent also has shell access, the `ctio` CLI is faster for batch and large-file workflows. See https://conversiontools.io/docs/agents for every Conversion Tools agent integration (Claude Code, Cursor, Codex, Grok, Gemini, and the ctio CLI).
