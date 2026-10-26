# JobPilot: LinkedIn Job Alert Analyzer & Cover Letter Generator

An n8n workflow that turns LinkedIn job alert emails into scored job matches and draft cover letters, using Google Gemini (free tier) and Google Workspace APIs.

> Personal-use automation. It processes only my own job alert emails and never submits applications. Every cover letter is saved as a Gmail draft for human review.

## Problem
Checking job alerts, reading each posting, comparing it to my resume, and writing a tailored cover letter is repetitive and slow. JobPilot automates the first pass so I only spend time on jobs that fit.

## How it works
1. **Gmail Trigger** picks up unread LinkedIn job alert emails.
2. **Google Drive + Extract from File** load my resume PDF once per run.
3. **Code node** parses the email HTML and extracts unique job IDs and links.
4. **Loop Over Items** processes one job at a time (rate-limit friendly).
5. **HTTP Request + Code node** fetch and clean the job description.
6. **JD Analyzer (Gemini)** returns structured JSON: skills, seniority, keywords.
7. **Resume Analysis (Gemini)** compares the resume to the job and returns a match score, gaps, and resume tweaks.
8. **IF node** only generates a cover letter when the score is 70 or higher.
9. **Cover Letter (Gemini)** writes a tailored letter using only facts from the resume.
10. **Google Sheets** appends or updates a row, matched on job ID to prevent duplicates.
11. **Gmail** creates a draft with the letter and resume tweaks.
12. **Wait node** paces calls for free-tier limits; processed emails are marked as read.

## Architecture
```
Gmail Trigger -> Drive (resume) -> Extract PDF -> Parse Email
  -> Loop -> HTTP -> Clean JD -> JD Analyzer -> Resume Analysis -> Score >= 70?
       yes -> Cover Letter -> Build Row -> Sheets -> Gmail Draft -> Wait -> Loop
       no  ------------------> Build Row -> Sheets ---------------> Wait -> Loop
  -> (done) Unique Email IDs -> Mark as Read
```
[Insert screenshot of the n8n canvas here]

## Tech stack
n8n, Google Gemini API (AI Studio free tier), Gmail API, Google Drive API, Google Sheets API, JavaScript (Code nodes), structured output parsers (JSON schema).

## Key design decisions
- **Structured output parsers** keep LLM responses machine-readable and consistent.
- **Score threshold** skips cover-letter generation for weak matches, saving free-tier quota.
- **Dedupe by job ID** using Sheets append-or-update.
- **Loop + Wait + retry on fail** handle per-minute rate limits.
- **Human in the loop:** drafts only, nothing is sent automatically.
- **Prompt guardrails:** the model may only use facts present in the resume.

## Setup
1. Install n8n locally (`npx n8n` or Docker) and open http://localhost:5678.
2. Import `workflow.json` (Workflow menu, then Import from file).
3. Create credentials: Gmail OAuth2, Google Drive, Google Sheets, Google Gemini API key.
4. Upload your resume PDF to Drive and select it in the **Download Resume** node.
5. Create a Google Sheet with these headers: `jobId, title, company, location, url, matchScore, recommendation, matchingSkills, missingSkills, resumeTweaks, coverLetter, status, processedAt`.
6. Set the LinkedIn sender filter and your own email address in the Gmail nodes.
7. Run once with **Execute workflow**, then activate.

## Sample output
<img width="972" height="813" alt="image" src="https://github.com/user-attachments/assets/419aebe1-5e41-4972-9a93-e12cf1b29873" />

<img width="1534" height="869" alt="image" src="https://github.com/user-attachments/assets/d1c86eac-4210-414f-8409-573df776cb37" />

## Limitations
- LinkedIn may block job page fetches; the workflow falls back to the title and email text.
- Email HTML formats change, so the parser may need updates.
- Free-tier Gemini limits slow down large batches.
- n8n must be running for the trigger to poll.

## Future improvements
Error notifications, a database instead of Sheets, resume-tailoring suggestions per job, multi-source alerts, and a dashboard.

