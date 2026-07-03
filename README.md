# RFQ to Quote Automation

An n8n workflow that turns incoming freight quote-request emails into structured
data and ready-to-review quotes — no manual retyping required.

Gmail inbox → AI extraction (Claude) → validated structured data (Google Sheets)
→ auto-priced quote draft, with low-confidence extractions routed to a human
instead of guessed at.

**Stack:** n8n · Claude API · Gmail API · Google Sheets · Slack

---

## The problem

Freight brokers and logistics teams receive dozens of Request for Quote (RFQ)
emails a day, each formatted differently by whoever sent it. Someone has to
read each one, pull out the shipment details, calculate a price, and draft a
reply — a slow, repetitive, error-prone process that doesn't scale.

## How it works

```
Gmail Trigger (watches inbox)
        │
Normalize Email (parse raw Gmail payload → clean fields)
        │
Has Attachment? ──true──> Extract Text From PDF ──> Use PDF Text ──┐
        │                                                          │
       false──> Use Email Text ─────────────────────────────────┤
                                                                    ▼
                                              Build Extraction Prompt
                                                                    │
                                              Claude — Extract RFQ Data
                                                                    │
                                              Validate Extraction (parse + flag)
                                                                    │
                                              Log to Google Sheet
                                                                    │
                                    Needs Human Review? ──true──> Alert Slack
                                              │
                                            false
                                              │
                                    Apply Pricing Rules
                                              │
                                    Draft Quote Email (Gmail draft, not sent)
```

1. **Ingestion** — a Gmail inbox is monitored for incoming mail, including PDF attachments.
2. **AI extraction** — Claude parses the email or document and returns structured JSON (origin, destination, commodity, weight, volume, mode, Incoterm, requested dates, customer info) against a strict schema. The model is explicitly instructed to return `null` rather than guess when a field is missing or ambiguous.
3. **Validation & logging** — every extraction is parsed and checked for required fields before use. Every result — clean or not — is logged to a spreadsheet.
4. **Smart routing** — complete, high-confidence records get priced and a quote email auto-drafted. Incomplete or ambiguous ones are flagged and sent to Slack for a human to handle instead of proceeding silently.
5. **Human-in-the-loop by design** — the system never sends anything automatically. It only creates a Gmail **draft**. A person always reviews before it goes out.

## Why this pattern, not just this industry

The core loop — **document/email in → AI extraction → structured data → validated automated action** — isn't specific to freight. The same shape applies directly to invoice/AP processing, insurance claims intake, procurement quote comparison, and order entry. This repo is the freight implementation as a working reference.

---

## Tech stack

| Layer | Tool |
|---|---|
| Orchestration | [n8n](https://n8n.io) (self-hosted or cloud) |
| Extraction | [Claude API](https://console.anthropic.com) (Anthropic) |
| Inbox / drafting | Gmail API |
| Structured data store | Google Sheets |
| Exception alerts | Slack API |

Runs comfortably under **$30/month** (a small VPS for self-hosted n8n + pay-per-token API usage).

---

## Setup

### Prerequisites
- An n8n instance (self-hosted or n8n cloud)
- A Gmail account to watch
- An Anthropic API key
- A Google account for Sheets logging
- A Slack workspace (optional — only needed for the exception-routing branch)

### 1. Import the workflow
Import `rfq-to-quote-automation.json` into n8n via **Menu → Import from File**.

### 2. Create the log sheet
Create a Google Sheet with a tab named `RFQ Log` and these headers in row 1:

```
Received At | From | Subject | Customer Name | Origin | Destination | Commodity | Weight (kg) | Volume (cbm) | Mode | Incoterm | Requested Pickup | Needs Review | Notes
```

### 3. Connect credentials

| Node | Credential type | Notes |
|---|---|---|
| Gmail Trigger / Draft Quote Email | Gmail OAuth2 | Sign in with Google |
| Claude - Extract RFQ Data | Header Auth | Header name `x-api-key`, value = your Anthropic API key |
| Log to Google Sheet | Google Sheets OAuth2 | Sign in with Google, then paste your Sheet ID into the node |
| Alert Slack - Exception | Slack OAuth2 or Bot Token | If using a bot token, invite the bot to the target channel |

### 4. Configure the Gmail Trigger
- **Simplify: OFF** — required, otherwise attachments aren't available.
- **Download Attachments: ON**
- **Attachment property prefix:** `attachment_`

> With Simplify off, Gmail returns its **raw API format** — capitalized fields (`From`, `Subject`) and no flat body-text field. The `Normalize Email` node handles decoding the base64 MIME payload; see [Known limitations](#known-limitations) if you're adapting this for a different trigger version.

### 5. Set your pricing rules
Edit the `Apply Pricing Rules` node — it ships with placeholder rates:

```js
const ratePerKgByMode = { air: 4.20, ocean: 0.85, road: 1.10 };
const baseHandlingFee = 75;
```

Replace with real rates/margins before using this for actual quotes.

---

## Testing

1. Send a real test email (a clean, complete RFQ) to the watched inbox, unread.
2. On the Gmail Trigger node, click **Test step** to fetch it manually.
3. Click **Test workflow** and step through each node's output.
4. Confirm: `subject`/`bodyText` populated after Normalize Email → clean extraction with `_needsReview: false` → new row in the sheet → draft appears in Gmail Drafts.
5. Repeat with an incomplete/ambiguous email → should route to Slack instead of drafting a quote.
6. Repeat with a non-RFQ email (e.g. a delivery confirmation) → should return `is_rfq: false` and not draft anything.

---

## Known limitations

- **One RFQ per email.** An email listing two separate shipments is only extracted as a single record.
- **Scanned/faxed PDFs return empty text.** The PDF node reads embedded text only, not OCR — a scanned document silently yields nothing to extract from. Planned fix: route PDFs to Claude directly as a document input instead of pre-extracting text, since Claude reads scanned pages natively.
- **Placeholder pricing.** The rate table is illustrative, not real pricing logic.
- **No deduplication.** Re-processing the same email would create a duplicate log row and draft; a message-ID check should be added before production use.

---

## Roadmap

- [ ] Route PDFs to Claude as documents (handles scanned/non-text PDFs)
- [ ] Message-ID based deduplication
- [ ] Multi-shipment email splitting
- [ ] Configurable pricing rules per customer/lane
- [ ] CRM sync for won quotes

---

## License

MIT — use, adapt, and extend freely.
