# Cold Emailing AI Agent

An automated cold email workflow built with n8n that reads leads from Google Sheets, scrapes company websites for research, generates fully personalized emails using Claude AI, and sends them via Gmail — with full status tracking and error handling.

---

## Overview

Every day you upload Apollo leads into a Google Sheet with `Status = NEW`. This workflow automatically picks up only those new rows, scrapes each company's website for real context, personalizes an email for each lead based on their job title and actual company research, sends it via Gmail, and updates the sheet with the result.

No manual copy-pasting. No duplicate sends. Every email is unique and grounded in real company information.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          n8n Workflow                                     │
│                                                                           │
│  [1] Manual Trigger                                                       │
│         │                                                                 │
│         ▼                                                                 │
│  [2] Google Sheets ──────── Read all rows from "master" tab              │
│         │                                                                 │
│         ▼                                                                 │
│  [3] Filter ─────────────── Status = NEW + Email not empty               │
│         │                                                                 │
│         ▼                                                                 │
│  [4] Code Node ──────────── Detect title group → map to Google Doc ID    │
│         │                   Extract first name from full name            │
│         ▼                                                                 │
│  [5] Google Docs ────────── Read matching email template doc             │
│         │                                                                 │
│         ▼                                                                 │
│  [6] Code Node ──────────── Extract plain text from doc                  │
│         │                                                                 │
│         ▼                                                                 │
│  [7] HTTP Request ───────── Fetch company website (Company URL column)   │
│         │                   Graceful fallback if site is blocked         │
│         ▼                                                                 │
│  [8] Code Node ──────────── Strip HTML → clean text (4000 char limit)    │
│         │                   Pass as companyResearch to Claude            │
│         ▼                                                                 │
│  [9] Claude API ─────────── Personalize email using:                     │
│  (HTTP Request)             - Structured {{PLACEHOLDER}} template        │
│         │                   - Real company research from website         │
│         │                   - Lead data (name, title, company)           │
│         │                   - Hardcoded proof points + links             │
│         ▼                                                                 │
│  [10] Code Node ─────────── Parse Claude JSON response                   │
│         │                   Bold recruiter labels                        │
│         │                   Convert URLs to HTML links                   │
│         │                   Build clean HTML paragraphs                  │
│         ▼                                                                 │
│  [11] Gmail ─────────────── Send HTML email as "Bhavik"                  │
│         │                                                                 │
│    ┌────┴────┐                                                            │
│    ▼         ▼                                                            │
│ [12] Sheets  [13] Sheets                                                 │
│  Update:      Update:                                                    │
│  Status=Sent  Status=Error                                               │
│  Sent At=now  Error Message                                              │
│  Email Content                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation |
| **Google Sheets** | Lead storage and status tracking |
| **Google Docs** | Email template storage (one doc per audience type) |
| **Claude AI (Haiku)** | Email personalization via Anthropic API |
| **Gmail** | Email sending |
| **HTTP Request** | Company website scraping for real research context |

---

## Google Sheet Structure

Create a sheet named `master` with these exact column headers:

| Column | Description |
|---|---|
| `Name` | Lead's full name |
| `Email` | Lead's email address |
| `Title` | Lead's job title (used for template detection) |
| `Company` | Company name |
| `Company URL` | Company website URL (e.g. `https://openai.com`) — scraped automatically for research |
| `Template Type` | Optional override (`recruiter`, `hiring_manager`, `founder`, `peer_referral`, `generic`) |
| `Email Content` | Auto-filled after send — the exact email body sent |
| `Status` | Set to `NEW` when uploading. Workflow updates to `Sent` or `Error` |
| `Sent At` | Auto-filled timestamp when email is sent |
| `Error Message` | Auto-filled if sending fails |

**Status values:** `NEW` → `Sent` or `Error`

---

## Title → Template Mapping

The workflow automatically detects the right email template based on the lead's job title:

| Title Contains | Template Used | Goal |
|---|---|---|
| Recruiter, Talent, HR | `recruiter` | Introduce Bhavik directly with proof points |
| Manager, Lead, Head, Director, VP | `hiring_manager` | Show technical fit and shipped work |
| Founder, CEO, Co-Founder, Owner | `founder` | Pitch direct operational value |
| Data Scientist, Engineer, Analyst, Researcher | `peer_referral` | Genuine connection + referral ask |
| Everything else | `generic` | Referral ask with honest framing |

> If you fill in the `Template Type` column manually on any row, it overrides the auto-detection.

---

## Email Template System

### How Templates Work
Each template lives in its own Google Doc. The workflow reads the doc, scrapes the company website, then Claude fills in only the `{{PLACEHOLDER}}` markers — everything else is copied exactly as written.

### Placeholder Reference

| Placeholder | Filled By | Source |
|---|---|---|
| `{{FIRST_NAME}}` | Claude | First word of `Name` column |
| `{{COMPANY}}` | Claude | `Company` column |
| `{{SUBJECT_TAG}}` | Claude | Generated from company website research |
| `{{COMPANY_SPECIFIC_REASON}}` | Claude | Company website research (generic) |
| `{{HONEST_TAKE}}` | Claude | Company website research (generic) |
| `{{COMPANY_SPECIFIC_PROJECT}}` | Claude | Company website research (hiring manager) |
| `{{TECHNICAL_ANGLE}}` | Claude | Company website research (hiring manager) |
| `{{PROOF_POINT}}` | Hardcoded | "architected a production GenAI system cutting intake-to-routing time by 75%..." |
| `{{RECENT_MOVE}}` | Claude | Company website research (founder) |
| `{{REAL_TAKE}}` | Claude | Company website research (founder) |
| `{{PERSONALIZATION_LINE}}` | Claude | Company website research (founder) |
| `{{COMPANY_HOOK}}` | Claude | Company website research (recruiter) |

### Hardcoded Fixed Values (same across all emails)
- **LinkedIn:** `https://www.linkedin.com/in/jbhavik70/` — renders as `LinkedIn` link
- **Resume:** Google Doc link — renders as `Resume` link
- **Recruiter snapshot:** Build / Lead / Communicate bullets copied word for word

### Company Research Fallback
If a company website blocks scraping (403, JS-only, timeout), the workflow continues and Claude writes a plausible observation using the company name and lead title. Nothing crashes.

---

## Email Template Docs

Create 4 Google Docs and paste each template with `{{PLACEHOLDER}}` markers as shown above.

| Doc | Template Type |
|---|---|
| Email — Generic | `generic` / `peer_referral` |
| Email — Hiring Manager | `hiring_manager` |
| Email — Founder | `founder` |
| Email — Recruiter | `recruiter` |

Paste each Doc's ID into the `DOC_IDS` map in the **Determine Template** node.

---

## Setup Instructions

### 1. Import the Workflow
- In n8n: New Workflow → ⋮ menu → **Import from JSON**
- Upload `cold_email_workflow.json`

### 2. Create Credentials in n8n
Go to **Settings → Credentials → New** and create:

| Credential Type | Used By | Notes |
|---|---|---|
| Google Sheets OAuth2 | Get Sheet Rows, Update nodes | Authorize with your Google account |
| Google Docs OAuth2 | Read Template Doc | Same Google account |
| Gmail OAuth2 | Send Gmail | Account you want to send from |
| HTTP Header Auth | Claude Generate Email | Header Name: `x-api-key` → your Anthropic API key |

### 3. Connect Credentials to Nodes
Open each node in n8n and select the matching credential from the dropdown. Click **Save** after connecting all credentials — this is required for them to persist.

### 4. Paste Google Doc IDs
Open the **Determine Template** node and replace the placeholder IDs in `DOC_IDS`:

```javascript
const DOC_IDS = {
  recruiter:      'YOUR_DOC_ID',
  hiring_manager: 'YOUR_DOC_ID',
  founder:        'YOUR_DOC_ID',
  peer_referral:  'YOUR_DOC_ID',
  generic:        'YOUR_DOC_ID'
};
```

Find a Doc ID from its URL: `docs.google.com/document/d/**THIS_PART**/edit`

---

## Running the Workflow

1. Export leads from Apollo
2. Paste them into the Google Sheet `master` tab
3. Set `Status = NEW` and fill in `Company URL` for each lead
4. Open the workflow in n8n → click **Test Workflow**
5. Check your Gmail Sent folder and the Google Sheet for updated statuses

> The workflow only processes rows with `Status = NEW`. Already sent rows are never touched.

---

## Cost Estimate (Claude Haiku)

| Volume | Estimated Cost |
|---|---|
| 100 emails | ~$0.15 |
| 500 emails | ~$0.75 |
| 1,000 emails | ~$1.50 |

*Slightly higher than base estimate due to company research context added to each prompt.*

---

## Safety Guarantees

- Rows marked `Sent` are **never reprocessed** — the filter blocks them
- Rows with a blank `Email` are **skipped automatically**
- If Gmail fails, the row is marked `Error` with the reason saved — nothing is lost silently
- If a company website blocks scraping, the workflow continues with graceful fallback
- Running the workflow multiple times is completely safe
- Success and error paths are wired directly from Gmail's outputs — no IF node logic
