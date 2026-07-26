---
name: parse-integration
description: Extract structured data from documents (invoices, receipts, purchase orders, bills of lading, bank statements, contracts, forms, ID documents, scans, photos of paper) with Parse by Conversion Tools, and build a production integration against its API. Covers getting an API key, defining an extraction schema, submitting a document, polling until the extraction reaches a terminal state, handling failures, replacing polling with webhooks, and exporting results to CSV or XLSX. Use when the user wants document data extraction, PDF-to-JSON with named fields, an OCR-and-structure pipeline, or asks how to integrate a document parsing API into their own code.
compatibility: Works with any agent that has MCP support (the bundled Conversion Tools MCP server exposes the parse_* tools) and with any language for the integration guidance. Examples are Python 3.9+ and Node.js 20+.
metadata:
  author: conversiontools
  version: "1.0"
  website: https://parse.conversiontools.io
---

# Parse: document data extraction

Parse turns documents into structured JSON. You describe the fields you want, send a PDF, scan, or photo, and get those fields back with their values. It handles the layout variation that breaks template-based parsers: the same schema works across suppliers, form revisions, and scan quality.

Two ways to use it, and this skill covers both.

- **Run an extraction right now** with the `parse_*` MCP tools bundled in this plugin.
- **Build an integration** in the user's own codebase against the HTTP API.

## The one rule that decides whether an integration works

`POST /v1/extract` is **asynchronous by default**. It answers `202` with an extraction `id` and `status: "processing"` immediately, and the extracted data is not in that response. You then poll `GET /v1/extractions/{id}` until `status` is `completed` or `failed`.

**Branch on `status`, never on timing.** Every response carries `status`, so one code path handles all of them. Integrations that assume "if the call took a while, the data must be in there" break the first time a document is slow or fast in the wrong direction, and they break silently.

There are two shortcuts, and both still report `status`, so the same branch handles them:

- `wait=N` holds the request open for up to N seconds (max 120) and returns the result inline if it finishes in time. It only applies when the document is uploaded in the same call. If the window expires you get the usual `202` and id, and you poll.
- A cache hit returns `status: "completed"` with `cached: true` instantly. The same document with the same field definitions, already extracted on that account, is served from storage and costs no pages.

---

## Using Parse from this agent (MCP tools)

| Tool | What it does |
|------|--------------|
| `parse_extract` | Submit a document. Returns an extraction id and `status`, **not** the data. |
| `parse_extraction_status` | Poll one extraction. Returns the data once `status` is `completed`. |
| `parse_list_schemas` | List saved field definitions on the account. |
| `parse_create_schema` | Create a reusable field definition. |
| `parse_export` | Turn a completed extraction into CSV or XLSX. |
| `parse_usage` | Pages used, page limit, remaining, reset date. |

The loop is always: `parse_extract`, then `parse_extraction_status` every 2 to 3 seconds until terminal, then optionally `parse_export`. Check `parse_list_schemas` before inventing a new field list - reusing a schema keeps results consistent between runs and lets identical documents hit the cache.

---

## Building an integration

Base URL: `https://api-parse.conversiontools.io`
Auth: `Authorization: Bearer <api key>` on every request.

### Step 1: get an API key

Create an account at [parse.conversiontools.io](https://parse.conversiontools.io), then go to **Dashboard, API Keys** ([direct link](https://parse.conversiontools.io/dashboard/api-keys)) and create a key. Keys look like `pk_live_...`.

Keep it in an environment variable or a secret manager. Never commit it, never log it, never put it in client-side code - the key carries the account's full page allowance.

```bash
export PARSE_API_KEY="pk_live_..."
```

### Step 2: define a schema

A schema is a named list of fields. Save it once and reference it by `schema_id`, or pass an inline `fields` array for a genuine one-off.

Field rules, enforced server-side:

- `name` must be snake_case: lowercase letters, digits, underscores, starting with a letter.
- `type` is one of `string`, `number`, `date`, `boolean`, `array`, `object`.
- An `array` needs `items` (`string`, `number`, `date`, `boolean`, or `object`). An `object`, and an `array` of `object`, needs a nested `fields` list.
- Optional per field: `description`, `required`, `examples`, `aliases`, and `validation` (`regex` or `enum` for strings, `min` or `max` for numbers).
- Limits: 50 fields per level, 200 fields total, 4 levels of nesting.

`description` is the single biggest accuracy lever. "Invoice total including tax, bottom right, may be labelled Amount Due" beats `total` on its own. `aliases` covers the other labels a document might use for the same thing.

```bash
curl -X POST https://api-parse.conversiontools.io/v1/schemas \
  -H "Authorization: Bearer $PARSE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Supplier invoice",
    "description": "Fields we post into the ledger",
    "fields": [
      { "name": "invoice_number", "type": "string", "required": true,
        "description": "Supplier invoice reference, usually top right",
        "aliases": ["Invoice No", "Rechnungsnummer"] },
      { "name": "invoice_date", "type": "date", "required": true },
      { "name": "currency", "type": "string",
        "validation": { "enum": ["EUR", "USD", "GBP"] } },
      { "name": "total_amount", "type": "number",
        "description": "Total including tax", "validation": { "min": 0 } },
      { "name": "line_items", "type": "array", "items": "object",
        "fields": [
          { "name": "description", "type": "string" },
          { "name": "quantity", "type": "number" },
          { "name": "unit_price", "type": "number" },
          { "name": "amount", "type": "number" }
        ] }
    ]
  }'
```

A bad field definition comes back as `400` with a `details` array naming the exact path, for example `fields[0].name`. Fix and resend. Nothing is billed for a rejected request.

Other schema endpoints: `GET /v1/schemas`, `GET /v1/schemas/{id}`, `PUT /v1/schemas/{id}` (full replace), `DELETE /v1/schemas/{id}`.

### Step 3: submit a document

One call, `multipart/form-data`:

```bash
curl -X POST https://api-parse.conversiontools.io/v1/extract \
  -H "Authorization: Bearer $PARSE_API_KEY" \
  -F "file=@invoice.pdf" \
  -F "schema_id=SCHEMA_ID"
```

```json
{ "success": true, "id": "65aa28ae...", "status": "processing" }
```

Form fields on this call: `file` (required), `schema_id` or `fields` (a JSON string), `wait`, `webhook_url`, `no_cache`, `output_format`.

Or split it into upload and extract, which is the better shape when you extract the same file more than once, or want the upload and the extraction on different code paths:

```bash
# 1. upload -> file_id
curl -X POST https://api-parse.conversiontools.io/v1/upload \
  -H "Authorization: Bearer $PARSE_API_KEY" \
  -F "file=@invoice.pdf"

# 2. extract by file_id (JSON body)
curl -X POST https://api-parse.conversiontools.io/v1/extract \
  -H "Authorization: Bearer $PARSE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "file_id": "FILE_ID", "schema_id": "SCHEMA_ID" }'
```

JSON body fields: `file_id` (required), `schema_id` or `fields` (an array), `webhook_url`, `no_cache`, `output_format`. Note that `wait` is **not** available on this path - submit, then poll.

Send `no_cache` only when you actually want to bypass the cache. Any value present is read as "bypass", so a literal `false` still disables caching. Leave the field out instead.

### Step 4: poll until terminal, with bounded retries

```bash
curl https://api-parse.conversiontools.io/v1/extractions/EXTRACTION_ID \
  -H "Authorization: Bearer $PARSE_API_KEY"
```

```json
{
  "success": true,
  "id": "65aa28ae...",
  "filename": "invoice.pdf",
  "status": "completed",
  "data": { "invoice_number": "INV-2026-0042", "total_amount": 1284.5, "line_items": [ ... ] },
  "pages_used": 2,
  "validation": { "status": "passed", "warnings": [], "coercions": [] },
  "error": null,
  "created_at": "2026-07-26T10:00:00.000Z",
  "completed_at": "2026-07-26T10:00:37.000Z"
}
```

Poll every 2 to 3 seconds with a small backoff, and stop on a **wall-clock deadline** rather than a retry count - a long document legitimately takes a minute or more, and a fixed count silently means different things at different intervals. Treat hitting the deadline as its own outcome, not as a failure of the extraction: the extraction keeps running server-side and the id stays valid, so you can requeue the poll later.

`validation` reports how well the result matched the requested fields: `passed`, `partial` (usable, but read `warnings`), `failed` (required fields missing or wrong-typed, treat the data as unreliable), or `unvalidated` (no field definitions were supplied). Surface `partial` and `failed` to a human instead of writing them straight into a ledger.

### Step 5: handle `failed`

```json
{ "status": "failed", "error": "...", "pages_used": 1 }
```

A failed extraction is **not billed** - the reserved page is refunded. Do not blind-retry the same bytes with the same schema: it will fail the same way. Log the id and the error, and route it to a human or to a fallback. Retrying is only sensible after the input or the field definitions change.

Re-sending the same document while an extraction is still running returns the **same id** instead of starting a duplicate, so a retry after a network blip is safe and costs nothing extra.

### Step 6: webhooks instead of polling

Pass `webhook_url` at submit time and Parse posts to it when the extraction reaches a terminal state:

```json
{
  "event": "extraction.completed",
  "id": "65aa28ae...",
  "status": "completed",
  "pages_used": 2,
  "error": null,
  "created_at": "...",
  "completed_at": "..."
}
```

The payload deliberately carries the id and status only, **never the extracted data**. Your handler treats it as a trigger and fetches the result over the authenticated API. That way a misconfigured or hijacked endpoint cannot leak anyone's documents.

Requirements and behaviour:

- The URL must be publicly reachable over http or https. Private and internal addresses are rejected at submit time with `400`.
- Delivery is fire and forget with a short timeout and a few bounded retries. A webhook can be missed, so keep a slow reconciliation poll for anything you have not heard about.
- There is no signature on the payload. Use an unguessable path, treat the URL itself as a secret, and re-fetch through the API before acting on anything. Never trust the body as the source of truth.

### Step 7: export to a spreadsheet

Exports run the extracted JSON through a normal conversion. They are free, repeatable, and never re-run the extraction.

```bash
# start it (idempotent per format - a running export is reused)
curl -X POST https://api-parse.conversiontools.io/v1/extractions/EXTRACTION_ID/export \
  -H "Authorization: Bearer $PARSE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "format": "xlsx" }'

# poll, then download when ready
curl -L -o invoice.xlsx \
  "https://api-parse.conversiontools.io/v1/extractions/EXTRACTION_ID/export?format=xlsx" \
  -H "Authorization: Bearer $PARSE_API_KEY"
```

The `GET` speaks two languages: `application/json` with `{"status": "processing", "progress": N}` while it converts, and the file's own bytes once it is ready. **Check the response Content-Type**, do not assume you got a file. Nested arrays are flattened into rows with the parent values repeated, which is the layout spreadsheet users expect.

You can also pass `output_format: "csv" | "xlsx"` at submit time to start the export automatically as soon as the extraction completes.

### Other endpoints

- `GET /v1/usage` - `plan`, `pages_used`, `pages_limit`, `pages_remaining`, `reset_date`. One page is billed per document page. Failed extractions are refunded, cache hits cost nothing.
- `DELETE /v1/extractions/{id}` - deletes the stored result and its exports. Results are kept until you delete them, so wire this into the user's data-retention policy.

---

## Errors worth handling explicitly

| Status | Meaning | What to do |
|--------|---------|------------|
| `400` | Bad request. `details` names the exact field path. | Fix the schema or parameters and resend. Nothing was billed. |
| `401` | Missing, malformed, or revoked key. | Check the `Authorization` header and the key. Do not retry. |
| `404` | No such extraction or schema **on this account**. Ids are per-account. | Check the id. Do not retry. |
| `409` | Export requested before the extraction completed. | Poll until `completed`, then export. |
| `410` | The stored result is gone. | Re-submit the document with `no_cache: true`. |
| `429` | Monthly page allowance used up. | Stop. Retrying fails the same way. Check `GET /v1/usage`, raise the plan at [Dashboard, Billing](https://parse.conversiontools.io/dashboard/billing). |
| `5xx` | Server-side. | Retry a couple of times with backoff, then give up and report. |

Rule of thumb: retry `5xx` and network errors, never `4xx`.

---

## Reference implementation: Python

Python 3.9+, `pip install requests`.

```python
"""Minimal, production-shaped Parse client: submit, poll until terminal, export."""

import json
import os
import time
from pathlib import Path
from typing import Any, Optional

import requests

BASE_URL = "https://api-parse.conversiontools.io/v1"
API_KEY = os.environ["PARSE_API_KEY"]          # never hardcode the key
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


class ParseError(RuntimeError):
    """An API call was rejected."""

    def __init__(self, status: int, message: str, details: Any = None):
        super().__init__(f"HTTP {status}: {message}")
        self.status = status
        self.details = details


class ExtractionFailed(RuntimeError):
    """The document was processed but produced no usable result."""


class ExtractionPending(RuntimeError):
    """Still processing when our deadline ran out. The id stays valid."""


def _unwrap(response: requests.Response) -> dict:
    if response.ok:
        return response.json()
    try:
        body = response.json()
    except ValueError:
        body = {}
    raise ParseError(
        response.status_code,
        body.get("message") or body.get("error") or response.reason,
        body.get("details"),
    )


def create_schema(name: str, fields: list[dict], description: str = "") -> str:
    payload = {"name": name, "fields": fields, "description": description}
    body = _unwrap(requests.post(f"{BASE_URL}/schemas", headers=HEADERS, json=payload, timeout=30))
    return body["schema"]["id"]


def submit(
    path: str | Path,
    *,
    schema_id: Optional[str] = None,
    fields: Optional[list[dict]] = None,
    webhook_url: Optional[str] = None,
    no_cache: bool = False,
) -> dict:
    """Submit a document. Returns the envelope with `id` and `status`.

    This does NOT return the extracted data - poll wait_for_result() next.
    """
    form: dict[str, str] = {}
    if schema_id:
        form["schema_id"] = schema_id
    elif fields:
        form["fields"] = json.dumps(fields)
    if webhook_url:
        form["webhook_url"] = webhook_url
    if no_cache:
        # Send the field only when bypassing the cache: any value present counts.
        form["no_cache"] = "true"

    path = Path(path)
    with path.open("rb") as handle:
        response = requests.post(
            f"{BASE_URL}/extract",
            headers=HEADERS,
            files={"file": (path.name, handle)},
            data=form,
            timeout=120,
        )
    return _unwrap(response)


def get_extraction(extraction_id: str) -> dict:
    return _unwrap(requests.get(f"{BASE_URL}/extractions/{extraction_id}", headers=HEADERS, timeout=30))


def wait_for_result(extraction_id: str, *, timeout_s: float = 600.0) -> dict:
    """Poll until terminal. Branches on `status`, never on elapsed time."""
    deadline = time.monotonic() + timeout_s
    interval = 2.0

    while time.monotonic() < deadline:
        body = get_extraction(extraction_id)
        status = body.get("status")

        if status == "completed":
            return body
        if status == "failed":
            raise ExtractionFailed(body.get("error") or "extraction failed")
        # "pending" / "processing" -> keep waiting

        time.sleep(interval)
        interval = min(interval * 1.5, 10.0)   # gentle backoff, capped

    raise ExtractionPending(
        f"{extraction_id} still processing after {timeout_s:.0f}s. "
        "It is still running server-side; poll this id again later."
    )


def export(extraction_id: str, fmt: str, out_path: str | Path, *, timeout_s: float = 300.0) -> Path:
    """Export a COMPLETED extraction to csv or xlsx and save it."""
    _unwrap(requests.post(
        f"{BASE_URL}/extractions/{extraction_id}/export",
        headers=HEADERS, json={"format": fmt}, timeout=30,
    ))

    deadline = time.monotonic() + timeout_s
    while time.monotonic() < deadline:
        response = requests.get(
            f"{BASE_URL}/extractions/{extraction_id}/export",
            headers=HEADERS, params={"format": fmt}, timeout=60,
        )
        if not response.ok:
            _unwrap(response)

        # JSON means "still converting" or "failed"; anything else is the file.
        if response.headers.get("content-type", "").startswith("application/json"):
            state = response.json()
            if state.get("status") == "failed":
                raise ExtractionFailed(state.get("error") or "export failed")
            time.sleep(2.0)
            continue

        out_path = Path(out_path)
        out_path.write_bytes(response.content)
        return out_path

    raise ExtractionPending(f"export of {extraction_id} still converting after {timeout_s:.0f}s")


if __name__ == "__main__":
    submitted = submit("invoice.pdf", schema_id=os.environ["PARSE_SCHEMA_ID"])
    print("submitted:", submitted["id"], submitted["status"])

    try:
        # A cache hit already comes back completed - the same branch handles it.
        result = submitted if submitted["status"] == "completed" else wait_for_result(submitted["id"])
    except ExtractionFailed as exc:
        raise SystemExit(f"extraction failed, not retrying: {exc}")

    print("pages billed:", result.get("pages_used"), "cached:", result.get("cached", False))
    print(json.dumps(result["data"], indent=2, ensure_ascii=False))

    if (result.get("validation") or {}).get("status") in {"partial", "failed"}:
        print("needs review:", result["validation"].get("warnings"))

    export(result["id"], "xlsx", "invoice.xlsx")
```

## Reference implementation: Node.js

Node.js 20+, no dependencies (`fetch`, `FormData`, and `Blob` are built in).

```js
// parse-client.mjs - submit, poll until terminal, export.
import { readFile, writeFile } from 'node:fs/promises';
import { basename } from 'node:path';

const BASE_URL = 'https://api-parse.conversiontools.io/v1';
const API_KEY = process.env.PARSE_API_KEY; // never hardcode the key
if (!API_KEY) throw new Error('PARSE_API_KEY is not set');

const authHeaders = { Authorization: `Bearer ${API_KEY}` };

export class ParseError extends Error {
  constructor(status, message, details) {
    super(`HTTP ${status}: ${message}`);
    this.name = 'ParseError';
    this.status = status;
    this.details = details;
  }
}
export class ExtractionFailed extends Error {}
export class ExtractionPending extends Error {}

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function unwrap(response) {
  if (response.ok) return response.json();
  let body = {};
  try {
    body = await response.json();
  } catch {
    /* non-JSON error body */
  }
  throw new ParseError(response.status, body.message || body.error || response.statusText, body.details);
}

export async function createSchema(name, fields, description = '') {
  const body = await unwrap(
    await fetch(`${BASE_URL}/schemas`, {
      method: 'POST',
      headers: { ...authHeaders, 'Content-Type': 'application/json' },
      body: JSON.stringify({ name, fields, description }),
    }),
  );
  return body.schema.id;
}

/** Submit a document. Returns { id, status } - NOT the extracted data. */
export async function submit(filePath, { schemaId, fields, webhookUrl, noCache = false } = {}) {
  const form = new FormData();
  form.append('file', new Blob([await readFile(filePath)]), basename(filePath));
  if (schemaId) form.append('schema_id', schemaId);
  else if (fields) form.append('fields', JSON.stringify(fields));
  if (webhookUrl) form.append('webhook_url', webhookUrl);
  // Send no_cache only when bypassing: any value present counts as "bypass".
  if (noCache) form.append('no_cache', 'true');

  // Do NOT set Content-Type by hand - fetch adds the multipart boundary.
  return unwrap(await fetch(`${BASE_URL}/extract`, { method: 'POST', headers: authHeaders, body: form }));
}

export async function getExtraction(id) {
  return unwrap(await fetch(`${BASE_URL}/extractions/${id}`, { headers: authHeaders }));
}

/** Poll until terminal. Branches on `status`, never on elapsed time. */
export async function waitForResult(id, { timeoutMs = 600_000 } = {}) {
  const deadline = Date.now() + timeoutMs;
  let interval = 2000;

  while (Date.now() < deadline) {
    const body = await getExtraction(id);
    if (body.status === 'completed') return body;
    if (body.status === 'failed') throw new ExtractionFailed(body.error || 'extraction failed');
    // 'pending' / 'processing' -> keep waiting
    await sleep(interval);
    interval = Math.min(interval * 1.5, 10_000);
  }

  throw new ExtractionPending(`${id} still processing; it keeps running server-side, poll this id again later`);
}

/** Export a COMPLETED extraction to csv or xlsx and save it. */
export async function exportTo(id, format, outPath, { timeoutMs = 300_000 } = {}) {
  await unwrap(
    await fetch(`${BASE_URL}/extractions/${id}/export`, {
      method: 'POST',
      headers: { ...authHeaders, 'Content-Type': 'application/json' },
      body: JSON.stringify({ format }),
    }),
  );

  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    const response = await fetch(`${BASE_URL}/extractions/${id}/export?format=${format}`, { headers: authHeaders });
    if (!response.ok) await unwrap(response);

    // JSON means "still converting" or "failed"; anything else is the file.
    if ((response.headers.get('content-type') || '').includes('application/json')) {
      const state = await response.json();
      if (state.status === 'failed') throw new ExtractionFailed(state.error || 'export failed');
      await sleep(2000);
      continue;
    }

    await writeFile(outPath, Buffer.from(await response.arrayBuffer()));
    return outPath;
  }

  throw new ExtractionPending(`export of ${id} still converting`);
}

// --- usage ---------------------------------------------------------------
const submitted = await submit('invoice.pdf', { schemaId: process.env.PARSE_SCHEMA_ID });
console.log('submitted:', submitted.id, submitted.status);

// A cache hit already comes back completed - the same branch handles it.
const result = submitted.status === 'completed' ? submitted : await waitForResult(submitted.id);

console.log('pages billed:', result.pages_used, 'cached:', Boolean(result.cached));
console.log(JSON.stringify(result.data, null, 2));

if (['partial', 'failed'].includes(result.validation?.status)) {
  console.warn('needs review:', result.validation.warnings);
}

await exportTo(result.id, 'xlsx', 'invoice.xlsx');
```

## Webhook receiver

```python
# Flask. The payload is a trigger, not the data - always re-fetch through the API.
from flask import Flask, request, abort

app = Flask(__name__)

@app.post("/hooks/parse/<secret_path>")          # unguessable path; treat the URL as a secret
def parse_webhook(secret_path: str):
    if secret_path != os.environ["PARSE_WEBHOOK_PATH"]:
        abort(404)

    event = request.get_json(silent=True) or {}
    extraction_id = event.get("id")
    if not extraction_id or extraction_id not in our_pending_extractions():
        abort(404)                                # never act on ids we did not submit

    if event.get("status") == "failed":
        record_failure(extraction_id, event.get("error"))
        return "", 204

    result = get_extraction(extraction_id)        # the authenticated fetch is the source of truth
    if result["status"] == "completed":
        store(extraction_id, result["data"])
    return "", 204                                # answer 2xx fast; do the work off the request
```

Keep a slow reconciliation poll for anything you never got a webhook for. Delivery is bounded-retry and can be missed.

---

## Checklist before calling an integration done

- [ ] The key comes from the environment or a secret manager, and never reaches logs, the client, or version control.
- [ ] The code branches on `status`, and no path assumes the data is in the submit response.
- [ ] Polling has a wall-clock deadline and a backoff, and a timeout is handled separately from a failure.
- [ ] `failed` is not blind-retried; it is logged with the id and routed to a human or a fallback.
- [ ] `429` stops the batch instead of hammering the API.
- [ ] `validation` of `partial` or `failed` is surfaced for review rather than written straight through.
- [ ] The export `GET` checks Content-Type instead of assuming a file.
- [ ] If webhooks are used, there is still a reconciliation poll, and the payload is treated as a trigger only.
- [ ] Deletion of stored extractions is wired into the data-retention policy.

## Links

- [Parse docs](https://parse.conversiontools.io/docs)
- [Quickstart](https://parse.conversiontools.io/docs/quickstart)
- [Extraction API reference](https://parse.conversiontools.io/docs/api/extract)
- [Schemas API reference](https://parse.conversiontools.io/docs/api/schemas)
- [Usage API reference](https://parse.conversiontools.io/docs/api/usage)
- [Authentication](https://parse.conversiontools.io/docs/authentication)
- [API keys](https://parse.conversiontools.io/dashboard/api-keys)
