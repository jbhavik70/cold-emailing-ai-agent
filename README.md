# Cold Emailing AI Agent

An automated cold email workflow built with n8n that reads leads from Google Sheets, generates personalized emails using Claude AI, and sends them via Gmail — with full status tracking and error handling.

---

## Overview

Every day you upload Apollo leads into a Google Sheet with `Status = NEW`. This workflow automatically picks up only those new rows, personalizes an email for each lead based on their job title, sends it via Gmail, and updates the sheet with the result.

No manual copy-pasting. No duplicate sends. Every email is unique.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        n8n Workflow                                  │
│                                                                      │
│  [1] Manual Trigger                                                  │
│         │                                                            │
│         ▼                                                            │
│  [2] Google Sheets ──── Read all rows from "master" tab             │
│         │                                                            │
│         ▼                                                            │
│  [3] Filter ─────────── Only rows where Status = NEW                │
│         │               Skip rows with empty Email                  │
│         ▼                                                            │
│  [4] Code Node ──────── Detect title group → map to Google Doc ID   │
│         │                                                            │
│         ▼                                                            │
│  [5] Google Docs ─────── Read the matching email template doc       │
│         │                                                            │
│         ▼                                                            │
│  [6] Code Node ──────── Extract plain text from doc                 │
│         │                                                            │
│         ▼                                                            │
│  [7] Claude API ─────── Personalize email (Haiku model)             │
│  (HTTP Request)         Input: Name, Title, Company + template      │
│         │               Output: JSON { subject, body }              │
│         ▼                                                            │
│  [8] Code Node ──────── Parse Claude response → clean HTML body     │
│         │                                                            │
│         ▼                                                            │
│  [9] Gmail ──────────── Send HTML email                             │
│         │                                                            │
│    ┌────┴────┐                                                       │
│    ▼         ▼                                                       │
│ [10] Sheets  [11] Sheets                                            │
│  Update:      Update:                                               │
│  Status=Sent  Status=Error                                          │
│  Sent At=now  Error Message                                         │
│  Email Content                                                       │
└─────────────────────────────────────────────────────────────────────┘
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

---

## Google Sheet Structure

Create a sheet named `master` with these exact column headers:

| Column | Description |
|---|---|
| `Name` | Lead's full name |
| `Email` | Lead's email address |
| `Title` | Lead's job title (used for template detection) |
| `Company` | Company name |
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
| Recruiter, Talent, HR | `recruiter` | Full-time job opportunity |
| Manager, Lead, Head, Director, VP | `hiring_manager` | Explore team fit |
| Founder, CEO, Co-Founder, Owner | `founder` | Pitch AI/ML value |
| Data Scientist, Engineer, Analyst, Researcher | `peer_referral` | Referral + genuine connection |
| Everything else | `generic` | Safe fallback |

> If you fill in the `Template Type` column manually on any row, it overrides the auto-detection.

---

## Email Template Docs

Create 5 Google Docs — one per audience type. Write your base email in each doc using `[Name]` and `[Company]` as placeholders. Claude will replace them during personalization.

| Doc | Template Type |
|---|---|
| Email — Recruiter | `recruiter` |
| Email — Hiring Manager | `hiring_manager` |
| Email — Founder | `founder` |
| Email — Peer Referral | `peer_referral` |
| Email — Generic | `generic` |

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
| HTTP Header Auth | Claude Generate Email | Header: `x-api-key` → your Anthropic API key |

### 3. Connect Credentials to Nodes
Open each node in n8n and select the matching credential from the dropdown.

### 4. Update Google Sheet URL
In nodes **Get Sheet Rows**, **Update — Sent**, and **Update — Error**, paste your Google Sheet URL.

### 5. Paste Google Doc IDs
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

1. Upload your Apollo leads to the Google Sheet with `Status = NEW`
2. Open the workflow in n8n
3. Click **Test Workflow**
4. Check your Gmail Sent folder and the Google Sheet for updated statuses

> The workflow only processes rows with `Status = NEW`. Already sent rows are never touched.

---

## Cost Estimate (Claude Haiku)

| Volume | Estimated Cost |
|---|---|
| 100 emails | ~$0.12 |
| 500 emails | ~$0.60 |
| 1,000 emails | ~$1.20 |

---

## Safety Guarantees

- Rows marked `Sent` are **never reprocessed** — the filter blocks them
- Rows with a blank `Email` are **skipped automatically**
- If Gmail fails, the row is marked `Error` with the reason saved — nothing is lost silently
- Running the workflow multiple times is completely safe
