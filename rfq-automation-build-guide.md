# Building an Email-to-Quote Automation Agent (n8n + Claude)

A complete, reproducible guide to the freight RFQ automation: an inbox is watched for
incoming quote requests, an LLM extracts structured shipment data from the email or
a PDF attachment, the data is logged to a spreadsheet, low-confidence extractions are
routed to a human, and clean ones get an auto-drafted price quote.

This is written the way it was actually built, including the real bug that came up
and how it was diagnosed — useful both as a build guide and as a case study.

---

## 1. Architecture

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

**Design principles baked in from the start:**
- The system never *sends* anything automatically — it creates a Gmail **draft**. A human reviews and sends. This is the single most important trust-building decision in the whole build.
- Every extraction is logged, whether clean or not, so nothing is silently lost.
- Anything the model isn't confident about is routed to a human, not guessed at.

---

## 2. Prerequisites

- An n8n instance (self-hosted or n8n cloud)
- A Gmail account to watch (can be the business inbox or a dedicated alias)
- An Anthropic API key ([console.anthropic.com](https://console.anthropic.com))
- A Google account for logging (Sheets)
- Slack workspace access (optional, for the exception-routing branch)

---

## 3. Build the nodes

### 3.1 Trigger — Gmail Trigger
- Node: **Gmail Trigger**
- Set **Simplify: OFF** — this is required, otherwise attachments aren't available and the email body comes back in a different shape.
- **Download Attachments: ON**
- **Attachment property prefix: `attachment_`** — attachments will show up as `attachment_0`, `attachment_1`, etc.
- Filter: unread only (or whatever label/query fits your inbox)

**Important, learned the hard way:** with Simplify off, Gmail's trigger returns the **raw Gmail API format** — capitalized fields (`From`, `Subject`, `To`), and no flat body-text field at all. The actual message body lives inside `payload`, MIME-encoded and base64-encoded. Don't assume you'll get a clean `subject`/`text`/`from` object — check the real output before writing anything downstream.

### 3.2 Normalize Email (Code node)
Purpose: turn Gmail's raw, nested, base64-encoded payload into flat, usable fields — and prep binary data for the attachment branch.

```javascript
function findPart(parts, mimeType) {
  for (const part of parts) {
    if (part.mimeType === mimeType && part.body && part.body.data) {
      return part.body.data;
    }
    if (part.parts) {
      const found = findPart(part.parts, mimeType);
      if (found) return found;
    }
  }
  return null;
}

function decodeBase64Url(data) {
  if (!data) return '';
  const base64 = data.replace(/-/g, '+').replace(/_/g, '/');
  try {
    return Buffer.from(base64, 'base64').toString('utf-8');
  } catch (e) {
    return '';
  }
}

const payload = $json.payload || {};
let bodyText = '';

if (payload.body && payload.body.data) {
  bodyText = decodeBase64Url(payload.body.data);
} else if (payload.parts) {
  const plainData = findPart(payload.parts, 'text/plain');
  if (plainData) {
    bodyText = decodeBase64Url(plainData);
  } else {
    const htmlData = findPart(payload.parts, 'text/html');
    if (htmlData) {
      bodyText = decodeBase64Url(htmlData).replace(/<[^>]+>/g, ' ');
    }
  }
}

if (!bodyText) {
  bodyText = $json.snippet || '';
}

const binary = $input.item.binary || {};
const attachmentKeys = Object.keys(binary).filter(k => k.startsWith('attachment_'));
const outBinary = { ...binary };
if (attachmentKeys.length > 0) {
  outBinary.data = binary[attachmentKeys[0]];
}

return {
  json: {
    from: $json.From || $json.from || '',
    subject: $json.Subject || $json.subject || '',
    bodyText: bodyText,
    hasAttachments: attachmentKeys.length > 0,
    attachmentCount: attachmentKeys.length
  },
  binary: outBinary
};
```

Mode: **Run Once for Each Item**.

### 3.3 Has Attachment? (IF node)
Condition: `{{ $json.hasAttachments }}` is boolean `true`.
- **True branch** → PDF extraction path
- **False branch** → plain email-body path

### 3.4 Extract Text From PDF (true branch)
Node: **Extract from File**, operation **PDF**, binary property `data`.

**Known limitation:** this only reads a PDF's embedded text layer. Scanned or faxed RFQs (photographed forms, handwritten notes) have no text layer, so this silently returns an empty string rather than erroring. For production use, either add an OCR fallback when the extracted text is empty, or skip this node entirely and send the PDF straight to Claude as a document (Claude reads scanned pages natively — no separate OCR step needed, at a small token-cost premium).

Followed by a **Set** node ("Use PDF Text") that copies `$json.text` into a common field: `documentText`.

### 3.5 Use Email Text (false branch)
**Set** node that copies `$json.bodyText` into the same common field: `documentText`.

Both branches converge on the next node — this is the fan-in point.

### 3.6 Build Extraction Prompt (Code node)
Builds the LLM prompt as a single string, embedding the subject and `documentText`, plus a strict JSON schema the model must follow:

```javascript
const schema = [
  '{',
  '  "is_rfq": boolean,',
  '  "origin": string or null,',
  '  "destination": string or null,',
  '  "commodity": string or null,',
  '  "weight_kg": number or null,',
  '  "volume_cbm": number or null,',
  '  "mode": "air" or "ocean" or "road" or null,',
  '  "incoterm": string or null,',
  '  "requested_pickup_date": string (ISO date) or null,',
  '  "customer_name": string or null,',
  '  "customer_email": string or null,',
  '  "special_requirements": string or null,',
  '  "confidence_notes": string',
  '}'
].join('\n');

const prompt = 'You are extracting structured data from a freight Request for Quote (RFQ) email or document.\n\n' +
  'Subject: ' + ($json.subject || '') + '\n\n' +
  'Content:\n' + ($json.documentText || '') + '\n\n' +
  'Return ONLY valid JSON, no markdown, no commentary, matching exactly this schema:\n' + schema + '\n' +
  'If a field cannot be determined, use null. Do NOT guess or fabricate values. If this is not actually an RFQ (e.g. a confirmation, invoice, or unrelated email), set is_rfq to false and leave other fields null.';

return { json: { ...$json, extractionPrompt: prompt } };
```

**Why a strict null-if-unknown instruction matters:** this single line is what makes the difference between a system you can trust and one that quietly fabricates plausible-looking wrong data.

### 3.7 Claude - Extract RFQ Data (HTTP Request node)
- Method: `POST`, URL: `https://api.anthropic.com/v1/messages`
- Authentication: **Generic Credential Type → Header Auth**
- Credential: header name `x-api-key`, value = your Anthropic API key (the *credential's display name*, e.g. "Anthropic API Key", is separate from the header name — don't put spaces in the header name field itself, that's an invalid HTTP token)
- Headers: `anthropic-version: 2023-06-01`, `content-type: application/json`
- Body (JSON):
```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1500,
  "messages": [{ "role": "user", "content": "{{ $json.extractionPrompt }}" }]
}
```

### 3.8 Validate Extraction (Code node)
Parses Claude's response text as JSON, strips accidental markdown fences, checks required keys are present, and flags anything incomplete or ambiguous for human review rather than letting it flow through silently:

```javascript
const response = $json;
let rawText = response.content?.[0]?.text || '';
rawText = rawText.replace(/```json/g, '').replace(/```/g, '').trim();

let parsed = null;
let validationError = null;
try {
  parsed = JSON.parse(rawText);
} catch (e) {
  validationError = 'Failed to parse JSON from model output: ' + e.message;
}

const requiredKeys = ['is_rfq', 'origin', 'destination', 'commodity', 'weight_kg', 'mode'];
if (parsed) {
  const missingKeys = requiredKeys.filter(k => !(k in parsed));
  if (missingKeys.length > 0) {
    validationError = 'Missing expected keys: ' + missingKeys.join(', ');
  }
}

const needsReview = !parsed || !!validationError || parsed.is_rfq === false ||
  !parsed.origin || !parsed.destination || !parsed.weight_kg;

const sourceFrom = $('Normalize Email').item.json.from;
const sourceSubject = $('Normalize Email').item.json.subject;

return {
  json: {
    ...(parsed || {}),
    _validationError: validationError,
    _needsReview: needsReview,
    _rawModelOutput: rawText,
    _sourceEmailFrom: sourceFrom,
    _sourceSubject: sourceSubject,
    _receivedAt: new Date().toISOString()
  }
};
```

Note the use of `$('Normalize Email').item.json...` — after an HTTP Request node, the incoming JSON is fully replaced by the API response, so the original sender/subject have to be pulled explicitly from that earlier node by name, not assumed to still be on `$json`.

### 3.9 Log to Google Sheet
- Node: **Google Sheets**, operation **Append**
- Point it at a spreadsheet with a tab named `RFQ Log` and these headers in row 1:
  `Received At | From | Subject | Customer Name | Origin | Destination | Commodity | Weight (kg) | Volume (cbm) | Mode | Incoterm | Requested Pickup | Needs Review | Notes`
- Map each column to the corresponding `$json` field from the previous node.

### 3.10 Needs Human Review? (IF node)
Condition: `{{ $json._needsReview }}` is `true`.

### 3.11 Alert Slack - Exception (true branch)
- Node: **Slack**, post a message to a channel (e.g. `#rfq-alerts`)
- Message includes sender, subject, and the reason it was flagged, so a human can act on it directly from Slack.

### 3.12 Apply Pricing Rules (false branch, Code node)
A simple, replaceable rate table — swap in real client pricing here:

```javascript
const data = $json;
const ratePerKgByMode = { air: 4.20, ocean: 0.85, road: 1.10 };
const baseHandlingFee = 75;
const mode = (data.mode || 'road').toLowerCase();
const rate = ratePerKgByMode[mode] || ratePerKgByMode.road;
const weight = Number(data.weight_kg) || 0;
const freightCost = Math.round(weight * rate * 100) / 100;
const totalQuote = Math.round((freightCost + baseHandlingFee) * 100) / 100;

return {
  json: {
    ...data,
    quotedFreightCost: freightCost,
    quotedHandlingFee: baseHandlingFee,
    quotedTotal: totalQuote,
    quoteCurrency: 'USD',
    quoteValidDays: 7
  }
};
```

### 3.13 Draft Quote Email
- Node: **Gmail**, resource **Draft**, operation **Create**
- `sendTo`: `{{ $json._sourceEmailFrom }}`
- Subject and body reference the extracted and priced fields, e.g. `{{ $json.origin }}`, `{{ $json.quotedTotal }}`
- **Deliberately a draft, not a send** — a human reviews before it goes out.

---

## 4. Connect credentials

| Node | Credential type | Notes |
|---|---|---|
| Gmail Trigger / Draft Quote Email | Gmail OAuth2 | Sign in with Google, grant access |
| Claude - Extract RFQ Data | Header Auth | Header name `x-api-key`, value = API key |
| Log to Google Sheet | Google Sheets OAuth2 | Sign in with Google |
| Alert Slack - Exception | Slack OAuth2 or Bot Token | Bot must be invited to the target channel if using a bot token |

---

## 5. Test end to end

1. Send a real test email (clean, complete RFQ) to the watched inbox, unread.
2. On the Gmail Trigger node, click **Test step** to fetch it manually.
3. Click **Test workflow** and let it run through every node.
4. Check each node's output in order — especially that `subject`/`bodyText` are populated after "Normalize Email," and that `_needsReview` is `false` for a clean email.
5. Confirm a new row landed in the Google Sheet and a draft appeared in Gmail Drafts.
6. Repeat with a deliberately incomplete/ambiguous email — confirm it routes to Slack instead of drafting a bad quote.
7. Repeat with a non-RFQ email (e.g. a delivery confirmation) — confirm `is_rfq: false` and it does **not** draft a quote.

---

## 6. Known limitations (be upfront about these in a pitch)

- **One RFQ per email.** An email listing two separate shipments will only be extracted as one record — a real limitation to disclose, not paper over.
- **Scanned/faxed PDFs return empty text** with the current text-extraction node — needs either an OCR fallback or routing PDFs to Claude as documents directly instead of pre-extracting text.
- **Pricing logic is a placeholder.** Real rates, fuel surcharges, and margin rules need to replace the flat per-kg table before this goes to a real client.
- **No deduplication.** If the same email is somehow processed twice, it'll create two log rows and two drafts — worth adding a message-ID check before production use.

---

## 7. What this demonstrates (for a pitch or portfolio writeup)

- Email/document ingestion → AI-based structured extraction → human-reviewable structured data → automated downstream action, the exact pattern buyers in freight, AP, insurance, and procurement are asking for.
- A validation layer that distinguishes "confidently extracted" from "needs a human," rather than trusting the model blindly.
- A human-in-the-loop safety net (draft, not send) that a client can trust from day one.
- Debugging real integration quirks (Gmail's raw payload format) rather than only working against clean, idealized sample data.
