# AI Resume Scorer - Automated Hiring Pipeline

An n8n workflow that automatically scores resumes against a job description using skill-cluster embeddings and AI evaluation. Accepts submissions via Google Form or direct Google Drive upload, deduplicates candidates, validates text extraction, generates a composite 100-point score, maintains a live top-5 leaderboard, and emails shortlisted candidates, all with full audit logging to Google Sheets.

---

## What It Does

Candidates submit a resume through a Google Form (with PDF upload) or drop a file directly into a monitored Google Drive folder. The workflow then:

1. Validates the form response and checks for duplicate email submissions
2. Copies the resume to Google Drive and converts it to plain text
3. Loads and caches the active Job Description (auto-refreshes when the JD changes)
4. Embeds the JD into skill clusters and scores the resume across four dimensions
5. Calls Groq (LLaMA 3.3 70B) to generate a detailed AI evaluation
6. Parses and validates the AI response
7. Writes the result to Google Sheets and updates the top-5 leaderboard
8. Emails shortlisted candidates and aggregates hiring insights

---

## Dual Trigger Architecture

The workflow supports two entry points:

```
Google Form Submission                    Google Drive (direct upload)
        ↓                                           ↓
Extract Form Response               Watch Resumes Folder (trigger)
        ↓                                           ↓
Validate Response                      Check: Already Scored?
        ↓                                           ↓
Check Duplicate Email                       Guard: Already Scored?
        ↓                                           ↓
Copy Resume to Resumes Folder ──────→  IF: New File?
        ↓                                           ↓
Normalise Form → Pipeline           Config: JD Folder ID
        ↓                                           ↓
Send Ack Email                         Search JD Folder → Extract JD Meta
                                                    ↓
                                         IF: JD Changed?
                                        ↙             ↘
                               Cache JD path      Reset Scores + Re-score Flag
                                        ↓
                              Copy Resume → Google Doc
                                        ↓
                              Export Resume as Text
                                        ↓
                              Read & Validate Resume Text
                                        ↓
                              IF: Text Extractable?
                                        ↓
                              Rate Limiter (5/min) → Wait 2s
                                        ↓
                              Build Prompt v2 (Rule + AI)
                                        ↓
                              Groq — Score Resume
                                        ↓
                              Parse & Validate v2
                                        ↓
                              IF: Scored Successfully?
                               ↙                    ↘
                    Insert to Sheet          Flag Error Row
                    Hiring Insights
                    Top-5 Leaderboard
                    IF: In Top 5?
                          ↓
                    Gmail: Shortlist Email
```

---

## Scoring Model

Resumes are scored across four dimensions:

| Dimension | Weight | Method |
|---|---|---|
| Keyword Match | 40 pts | Exact + fuzzy match against JD keywords |
| Skill Cluster Match | 30 pts | Cosine similarity against JD skill clusters |
| Quality & Clarity | 15 pts | AI evaluation of structure, specificity, impact |
| Education & Credentials | 15 pts | Degree level, certifications, relevance |
| **Total** | **100 pts** | |

### Tier Classification

| Score | Tier |
|---|---|
| ≥ 80 | Strong Fit |
| 60–79 | Good Fit |
| 40–59 | Partial Fit |
| < 40 | Poor Fit |

Candidates in the top 5 by final score receive an automatic shortlist email.

---

## Prerequisites

- **n8n** (self-hosted or cloud) — tested on v1.121.3
- **Google account** with:
  - Google Drive (Resumes folder + JD folder)
  - Google Sheets (with `Evaluations`, `Insights`, and `AuditLog` tabs)
  - Gmail
  - Google Forms (linked to a Google Sheet for response collection)
- **Groq API key** — [Get one free at console.groq.com](https://console.groq.com)

---

## Setup

### 1. Import the Workflow

In n8n: **Workflows → Import from File** → select `AI_Resume_Evaluator.json`

### 2. Set Up Google Drive Folders

Create two folders in Google Drive:
- **Resumes** — where candidate PDFs land (watched by the Drive trigger)
- **JD** — where you place the active Job Description PDF or Doc

Note both folder IDs from their URLs.

### 3. Create Your Google Sheet

Create a Google Sheet with three tabs:

**`Evaluations` tab** — add these headers in row 1:
```
Timestamp | File ID | File Name | Candidate Name | Email | Mobile | Years Exp | Current Role | LinkedIn | Relevance Score | Keyword Score | Cluster Score | Quality Score | Education Score | Final Score | Tier | Score Breakdown | Matched Clusters | Missing Clusters | Status
```

**`Insights` tab** — add these headers:
```
Timestamp | Total Scored | Avg Score | Top Cluster | Tier Counts | Cluster Breakdown
```

**`AuditLog` tab** — add these headers:
```
Timestamp | File ID | File Name | Status | Score | Notes
```

### 4. Configure Credentials

In n8n **Settings → Credentials**, create:

| Credential Name | Type | Used By |
|---|---|---|
| `Google Drive account` | Google Drive OAuth2 | Drive trigger, file copy nodes |
| `Google Sheets account` | Google Sheets OAuth2 | All sheet read/write nodes |
| `Gmail account` | Gmail OAuth2 | Ack, shortlist, and duplicate notice emails |

### 5. Set Your Groq API Key

In the `Groq — Score Resume` node, update the Authorization header:
```
Authorization: Bearer YOUR_GROQ_API_KEY
```

Model used: `llama-3.3-70b-versatile`. You can switch to `llama-3.1-8b-instant` for faster/cheaper runs on high volume.

### 6. Configure Node Settings

Update these values across the workflow:

| Node | Field | Value |
|---|---|---|
| `Watch Resumes Folder` | Folder to Watch | Your Resumes folder ID |
| `Config: JD Folder ID` | jsCode | Set `folderId` to your JD folder ID |
| `Google Form: New Response` | Document ID | Your Google Form response Sheet ID |
| `Check: Already Scored?` | Document ID | Your Evaluations Sheet ID |
| `Check: Duplicate Email?` | Document ID | Your Evaluations Sheet ID |
| `Insert to Google Sheet` | Document ID | Your Evaluations Sheet ID |
| `Write Insights to Sheet` | Document ID | Your Evaluations Sheet ID |
| `Write to Audit Log` | Document ID | Your Evaluations Sheet ID |
| `Gmail: Shortlist Email` | Send To | Your recruiter email |

### 7. Set Up Google Form

Create a Google Form with these fields (exact names matter — they map to column headers):

```
Full Name               (Short answer)
Email Address           (Short answer)
Mobile Number           (Short answer)
Years of Experience     (Short answer / Number)
Current Role / Title    (Short answer)
LinkedIn Profile URL    (Short answer)
Upload Resume           (File upload — PDF only)
```

Link the form to a Google Sheet for response collection. Set that Sheet ID in the `Google Form: New Response` trigger node.

> **Important:** Column names in your response Sheet must exactly match the field names used in `Extract Form Response` node. If your form questions are worded differently, update the keys in that node's code.

### 8. Upload a Job Description

Drop a PDF or Google Doc of the JD into your JD folder. The workflow reads it on first run and caches it. When you replace it with a new file, the cache auto-invalidates and all scores reset for re-evaluation.

### 9. Activate

Toggle the workflow **Active** in n8n.

---

## Nodes Reference

| Node | Type | Purpose |
|---|---|---|
| Google Form: New Response | Google Sheets Trigger | Fires when a new form row appears |
| Extract Form Response | Code | Parses form fields; validates mobile/email types |
| IF: Response Valid? | IF | Guards against malformed submissions |
| Check: Duplicate Email? | Google Sheets | Looks up email in Evaluations sheet |
| Guard: Duplicate Email? | Code | Handles empty-result case from Always Output Data |
| IF: Duplicate Email? | IF | Routes duplicates vs new candidates |
| Send Notice: Duplicate Submission | Gmail | Notifies candidate of duplicate |
| Copy Resume to Resumes Folder | HTTP Request | Copies file to watched folder |
| Normalise Form → Pipeline | Code | Standardises form data shape for pipeline |
| Send Ack: Submission Received | Gmail | Sends receipt confirmation to candidate |
| Watch Resumes Folder | Google Drive Trigger | Fires on new file in Resumes folder |
| Check: Already Scored? | Google Sheets | Looks up file ID in Evaluations sheet |
| Guard: Already Scored? | Code | Handles empty-result edge case |
| IF: New File? | IF | Skips already-scored files |
| Skip (Already Scored) | Code | Returns early with existing score |
| Config: JD Folder ID | Code | Sets JD folder ID from config |
| Search JD Folder | HTTP Request | Lists files in JD folder |
| Extract JD File Meta | Code | Reads file ID, name, modifiedTime |
| Read JD Cache Meta | Code | Checks static data for cached JD info |
| IF: JD Changed? | IF | Compares modifiedTime to detect JD swap |
| JD Change → Reset + Re-score Flag | Code | Resets leaderboard and tier counts |
| Copy JD → Google Doc | HTTP Request | Clones JD as Google Doc |
| Merge JD Meta | Code | Combines JD file ID and temp doc ID |
| Export JD as Text | HTTP Request | Exports JD plain text |
| Read JD Text | Code | Stores JD text + schedules temp doc deletion |
| Cache JD Text | Code | Writes JD to static data cache |
| Embed JD — Skill Clusters | Code | Builds keyword and cluster vectors from JD |
| Copy Resume → Google Doc | HTTP Request | Clones resume as Google Doc |
| Export Resume as Text | HTTP Request | Exports resume plain text |
| Read Resume Text | Code | Reads text + carries candidate metadata forward |
| Validate Resume Text | Code | Checks text quality; populates candidate fields |
| IF: Text Extractable? | IF | Routes unreadable files to error |
| Rate Limiter (5/min) | Code | Throttles AI calls to 5 per minute |
| Wait 2s (API Courtesy) | Wait | Adds 2s delay between AI calls |
| Build Prompt v2 (Rule + AI) | Code | Constructs system + user prompt for scoring |
| Groq — Score Resume | HTTP Request | Calls Groq LLaMA 3.3 70B |
| Parse & Validate v2 | Code | Parses JSON response; validates score fields |
| IF: Scored Successfully? | IF | Routes parse failures to error log |
| Insert to Google Sheet | Google Sheets | Writes scored candidate row |
| Hiring Insights Aggregator | Code | Updates running stats in static data |
| Top-5 Leaderboard Builder | Code | Maintains top-5 ranking in static data |
| IF: In Top 5? | IF | Routes top candidates to shortlist email |
| Gmail: Shortlist Email | Gmail | Sends shortlist notification to recruiter |
| Flag Error Row in Sheet | Google Sheets | Marks failed rows in Evaluations sheet |
| Write to Audit Log | Google Sheets | Logs all events to AuditLog tab |
| Stop and Error | Stop and Error | Halts execution on critical failure |
| Schedule: Refresh JD Cache | Schedule Trigger | Periodic JD cache refresh |

---

## JD Change Behaviour

When you replace the JD file in your JD folder:

- The workflow detects the new `modifiedTime` on next run
- `JD Change → Reset + Re-score Flag` clears all tier counts, leaderboard, and insights from static data
- All future submissions are scored against the new JD
- Previously scored resumes are **not** automatically re-scored — upload them again or run the workflow manually against each file

---

## Rate Limiting

The `Rate Limiter (5/min)` node enforces a maximum of 5 Groq API calls per minute using n8n static data. If you have a paid Groq plan with higher limits, remove this node or increase the threshold in its code.

---

## Troubleshooting

**`$json` red underlined in a Code node**
The node is set to "Run Once for All Items" mode. Replace `$json` with `$input.first().json`.

**`(row['Mobile Number'] || '').trim is not a function`**
Google Sheets returns phone numbers as integers. Fix line 8 in `Extract Form Response`:
```js
const mobile = String(row['Mobile Number'] ?? '').trim();
```

**`No output data returned` on Check: Duplicate Email?**
Enable **Always Output Data** in the node's Settings tab. Then update `Guard: Duplicate Email?` to check `!firstItem?.Email` instead of `items.length === 0`.

**Groq API key invalid**
Ensure the Authorization header value includes the `Bearer ` prefix:
```
Bearer gsk_your_key_here
```

**Candidate metadata empty (name/email/mobile)**
These fields flow from `Normalise Form → Pipeline`. If running from Drive trigger (not form), they will be empty — this is expected behaviour for direct uploads.

**Resume text body is empty**
The `Export Resume as Text` node returns text under a `data` key, not `body`. In `Read Resume Text`, use:
```js
const text = $input.first().json.data || $input.first().json.body || '';
```

---

## Known Limitations

- **No OCR** — scanned/image-only PDFs will not extract text. Only text-layer PDFs work.
- **Google Form file upload** — returns a Drive URL, not a binary. The workflow extracts the file ID from the URL and copies it; ensure the service account has access to the uploaded file.
- **Static leaderboard** — the top-5 is stored in n8n static data and resets on workflow redeploy. For persistence, write the leaderboard to a dedicated Google Sheet tab.
- **One active JD** — the workflow scores all resumes against a single JD at a time. Multi-role scoring requires duplicating the workflow.

---

## License

MIT — free to use, modify, and deploy. Attribution appreciated.
