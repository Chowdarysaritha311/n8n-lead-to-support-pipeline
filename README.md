# 🚀 Lead-to-Support Intelligent Automation Pipeline

> Production-grade n8n workflow for converting inbound leads into a structured, observable, reliable support pipeline.

---

## 📐 Architecture Overview

```
[Webhook POST]
      │
      ├──▶ [ACK 202 Immediately] ─────────────────── (fast response, non-blocking)
      │
      ▼
[🛡️ Validate + Spam Filter]
      │
      ├─ INVALID/SPAM ──▶ [☠️ Dead-Letter Prep] ──▶ [💀 DLQ Sheet]
      │
      ▼
[🔑 Generate Idempotency Hash (SHA-256)]
      │
      ▼
[🔍 Google Sheets Lookup — already processed?]
      │
      ├─ DUPLICATE ──▶ [📋 Log: Duplicate Ignored] ──▶ STOP
      │
      ▼
[🧠 Enrich: Company + Priority Score + Tags]
      │
      ▼
[📊 Log: Observability Entry]
      │
      ▼
[💾 Store to Google Sheets — Leads]
      │
      ▼
[🔀 Route: urgency == "high"?]
      │
      ├─ HIGH ──▶ [📢 Slack Alert] ──▶ [🎫 Jira Ticket] ──▶ [📋 Audit Log]
      │
      └─ NORMAL ──▶ [📧 Confirmation Email] ──▶ [✏️ Update Status] ──▶ [📋 Audit Log]


[⏰ Cron 18:00] ──▶ [📥 Fetch Leads] ──▶ [📊 Compute Analytics] ──▶ [📢 Slack Digest]

[🚨 Error Trigger] ──▶ [💀 DLQ Sheet]
```

---

## 🧠 Advanced Features Implemented (7 of 9)

| Feature | Node | Implementation |
|---|---|---|
| ✅ Idempotency Hashing | `Generate Idempotency Hash` | SHA-256(email\|message\|product) |
| ✅ Structured Logging | `Log: Enrichment Observability`, `Audit Trail Log` | JSON console.log at every stage |
| ✅ Observability per stage | All function nodes | event, timestamp, idempotencyKey logged |
| ✅ Priority Scoring | `Enrich` node | 0–100 score: urgency + keywords + company tier + message depth |
| ✅ Lead Classification Tags | `Enrich` node | HOT_LEAD, HIGH_URGENCY, INFERRED_COMPANY, etc. |
| ✅ Noise/Spam Scoring | `Validate + Spam Filter` | Score-based: keywords(+25), domains(+50), chars(+20), all-caps(+15) |
| ✅ Audit Trail | `Audit Trail Log` | Final routing decision + enrichment outcome per lead |
| ✅ Metrics tracking | `Digest: Compute Analytics` | Daily counts: processed, by urgency, by product, avg score |
| ✅ Configurable env vars | All credential nodes | All keys/IDs via $env.* — never hardcoded |

---

## ⚙️ Setup Instructions

### Prerequisites

- n8n v1.0+ (self-hosted or cloud)
- Google account (Sheets + Gmail)
- Slack workspace with incoming webhooks enabled
- Jira Cloud account (or skip Jira and use Trello alternative)

---

### Step 1 — Google Sheets Setup

Create a Google Sheet with **two tabs**:

#### Tab 1: `Leads`
Columns (exact names, case-sensitive):
```
idempotency_key | name | email | company | message | urgency | product |
priority_score | tags | lead_source | inferred_company | email_domain |
status | processed_at | spam_score | spam_flags
```

#### Tab 2: `DeadLetterQueue`
Columns:
```
timestamp | email | failure_stage | errors | spam_flags | spam_score | is_spam | raw_payload
```

---

### Step 2 — Required Credentials in n8n

Go to **Settings → Credentials** in n8n and add:

| Credential | Type | Used By |
|---|---|---|
| `Google Sheets OAuth2` | Google Sheets OAuth2 API | All Sheets nodes |
| `Gmail OAuth2` | Gmail OAuth2 | Email confirmation node |
| `Slack Webhook` | Slack Incoming Webhook | Alert + Digest nodes |
| `Jira Basic Auth` | HTTP Basic Auth | Jira ticket node |

---

### Step 3 — Environment Variables

Set these in your n8n instance (**Settings → Environment Variables** or `.env` file for self-hosted):

```env
# Google Sheets
GOOGLE_SHEET_ID=your_google_sheet_id_here

# Slack
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# Jira
JIRA_BASE_URL=https://yourcompany.atlassian.net
JIRA_PROJECT_KEY=SUPP

# Optional: customize spam threshold
SPAM_SCORE_THRESHOLD=50
```

> **Finding your Google Sheet ID:** It's the long string in the URL:
> `https://docs.google.com/spreadsheets/d/THIS_IS_THE_ID/edit`

---

### Step 4 — Import the Workflow

1. Open n8n → **Workflows** → **Import from File**
2. Select `workflow.json`
3. Update all credential references (look for `REPLACE_WITH_YOUR_CREDENTIAL_ID`)
4. Set environment variables
5. **Activate** the workflow

---

## 🧪 Testing the Webhook

### Get your webhook URL:
After activating, n8n shows the webhook URL:
```
https://your-n8n-instance.com/webhook/lead-intake
```

### Test with curl:

```bash
# Normal lead
curl -X POST https://your-n8n-instance.com/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d @samples/lead2.json

# High urgency lead
curl -X POST https://your-n8n-instance.com/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d @samples/lead1.json

# Spam lead
curl -X POST https://your-n8n-instance.com/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d @samples/lead4.json
```

Expected response for all valid POSTs:
```json
{
  "status": "accepted",
  "message": "Lead received and queued for processing",
  "timestamp": "2024-01-15T14:30:00.000Z"
}
```

---

## 🔁 Idempotency Demonstration

The system guarantees **exactly-once processing** regardless of how many times the same payload is sent.

### How it works:

```
hash = SHA-256(email.toLowerCase() + "|" + message.trim() + "|" + product.trim())
```

This hash is stored in the `idempotency_key` column of the Leads sheet.

### To demonstrate:

```bash
# Send lead1.json three times in a row
curl -X POST .../webhook/lead-intake -d @samples/lead1.json
curl -X POST .../webhook/lead-intake -d @samples/lead3.json  # identical payload
curl -X POST .../webhook/lead-intake -d @samples/lead9.json  # identical payload

# Then check your Google Sheet — only ONE row will exist for Sarah Chen
# n8n execution log will show:
#   Execution 1: Lead stored
#   Execution 2: "Duplicate Ignored" — terminated at idempotency check
#   Execution 3: "Duplicate Ignored" — terminated at idempotency check
```

**Expected Sheets result:** 1 row for `sarah.chen@stripe.com` with `idempotency_key = abc123...`

**Expected n8n log:**
```json
{"event": "DUPLICATE_IGNORED", "idempotencyKey": "4f2d...", "email": "sarah.chen@stripe.com"}
```

---

## ⏰ Triggering the Daily Digest Manually

The cron runs at 18:00 daily. To test immediately:

1. Open the workflow in n8n editor
2. Click the **⏰ Cron: Daily Digest at 6PM** node
3. Click **"Test step"** → it will trigger the full digest chain
4. Check Slack for the Block Kit digest message

---

## 🔄 Retry Strategy

n8n's built-in retry mechanism is configured for the storage and external API nodes:

- **Max retries:** 2
- **Retry on failure:** enabled per node settings
- **Error escalation:** Unrecoverable errors → Error Trigger → DLQ Sheet

For exponential backoff, configure in n8n's node settings:
- Attempt 1: immediate
- Attempt 2: 5 seconds delay
- Attempt 3: 30 seconds delay

---

## 📸 Screenshot Guide

### Screenshot 1 — High Urgency Flow Success
**What to capture:** n8n execution showing green checkmarks on:
- Webhook → Validate → Hash → Lookup → Enrich → Store → [urgency=high branch] → Slack → Jira → Audit

**Expected Slack message:** Block Kit card with 🚨 header, name/email/company/priority fields, full message, idempotency key in footer.

**Expected Jira:** New issue created with `[HIGH] Lead from Sarah Chen (Stripe) — PaymentSDK` title, priority = Highest, labels = HIGH_URGENCY, HOT_LEAD.

---

### Screenshot 2 — Normal Flow Success
**What to capture:** n8n execution green path via normal branch:
- Store → [urgency=normal branch] → Email → Update Status → Audit

**Expected Gmail sent:** HTML email to `marcus.williams@acmecorp.com` with reference ID, product name, response time SLA.

**Expected Sheets row:** status column = `acknowledged`

---

### Screenshot 3 — Dead-Letter Failure Case
**What to capture:** Use `lead4.json` (spam) or `lead7.json` (missing email).

n8n execution showing:
- Webhook → Validate → [isValid=false branch] → Dead-Letter Prep → DLQ Sheet

**Expected DLQ Sheet row:**
```
timestamp: 2024-01-15T...
email: winner123@mailinator.com
failure_stage: VALIDATION
errors: []
spam_flags: keyword:buy now, suspicious_domain:mailinator.com
spam_score: 75
is_spam: true
```

---

### Screenshot 4 — Daily Digest Output
**What to capture:** Slack channel showing the Block Kit digest with:
- Header: "📊 Daily Lead Digest — 2024-01-15"
- Stats: Total, Avg Priority, High/Normal split
- Product breakdown table
- Top 5 recent leads list

---

## 📁 File Structure

```
n8n-lead-pipeline/
├── workflow.json              # Main n8n workflow (import this)
├── README.md                  # This file
└── samples/
    ├── lead1.json             # HIGH urgency — enterprise, production outage
    ├── lead2.json             # NORMAL urgency — standard sales inquiry
    ├── lead3.json             # DUPLICATE of lead1 (replay test #1)
    ├── lead4.json             # SPAM — keyword + suspicious domain
    ├── lead5.json             # MISSING COMPANY — domain inference
    ├── lead6.json             # HIGH urgency — personal email, no company
    ├── lead7.json             # VALIDATION FAILURE — missing email field
    ├── lead8.json             # NORMAL — long enterprise inquiry
    ├── lead9.json             # DUPLICATE of lead1 (replay test #2)
    └── lead10.json            # SPAM — repeated chars + all caps + disposable email
```

---

## 🏗️ Node Naming Conventions

All nodes follow a `[EMOJI] CATEGORY: Description` pattern:

- `🔗 Webhook:` — entry triggers
- `✅ ACK:` — acknowledgment responses
- `🛡️ Validate` — validation/security
- `🔑 Generate` — hash/key generation
- `🔍 Check:` — lookups and existence checks
- `🔀 Route:` — conditional branching (IF nodes)
- `🧠 Enrich:` — data transformation
- `📊 Log:` — observability/logging
- `💾 Store:` — persistence operations
- `📢 Slack:` — Slack notifications
- `📧 Email:` — email operations
- `🎫 Jira:` — ticketing
- `✏️ Update` — record mutations
- `📋 Audit` — audit trail
- `☠️ Dead-Letter:` — DLQ preparation
- `💀 Dead-Letter:` / `Error →` — DLQ storage
- `⏰ Cron:` — scheduled triggers
- `📥 Digest:` — digest data operations
- `🚨 Error Handler:` — error triggers

---

## 🔒 Security Considerations

1. **Never commit** `workflow.json` with real credential IDs — replace before sharing
2. **Validate webhook origin** in production using a shared secret header (`X-Webhook-Secret`)
3. **Rate-limit** the webhook endpoint at your reverse proxy level (nginx/Cloudflare)
4. The spam filter provides a first-layer defense but should be complemented with CAPTCHA on the lead form
5. Store sensitive data (API keys) only in n8n's encrypted credential store, never in function node code

---

## 🐛 Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Webhook returns 404 | Workflow not activated | Toggle "Active" switch in n8n |
| Google Sheets lookup fails | Wrong sheet name or tab | Check tab name matches exactly: `Leads` |
| Duplicate check always passes | Column header mismatch | Ensure `idempotency_key` column exists in row 1 |
| Slack message not sent | Webhook URL wrong | Regenerate Slack webhook URL |
| Jira ticket creation 401 | Basic auth wrong | Use email:api_token format for Jira auth |
| Digest shows 0 leads | Date format mismatch | Ensure `processed_at` is stored as ISO 8601 |

---

*Built as a production-grade automation pattern. Not a beginner template.*
