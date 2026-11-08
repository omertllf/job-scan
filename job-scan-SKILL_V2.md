---
name: job-scan
description: >
  Scans Gmail for job alert emails from LinkedIn, Drushim, AllJobs, and Glassdoor
  over the last 30 days. Extracts all job listings, deduplicates them, checks for
  application confirmations and rejection emails, grades each job for fit against
  CV files (local folder first, then Project knowledge, latest version wins),
  recommends which CV version to use per job, and writes a styled HTML report to
  C:\Users\97254\Desktop\Omer\CV\Job and CV\. Does NOT send email — the connected
  Gmail tool supports reading and labeling only, not sending. The report is saved
  locally and the user is notified in-chat when it's ready.
  Trigger this skill whenever the user says anything like "scan jobs", "run job scan",
  "check new jobs", "update my job list", "what new jobs came in", "job search update",
  or "grade my jobs". Also trigger if the user asks to refresh or rebuild their job
  tracker, or wants to know which CV to send for a role.
compatibility: "Requires Gmail MCP (read/label only — no send capability, so email delivery of the report is not possible with this connector) and Desktop Commander MCP (for local CV folder access and HTML file writing). If run as a Cowork scheduled task, note it only executes while the machine is awake and Claude Desktop is open; a skipped run fires on next wake, not necessarily at the scheduled time."
---

# Job Scan Skill

Scans Gmail for job alert emails, extracts and deduplicates listings, checks
application/rejection status, grades each job against CV files in the project,
recommends the best CV version per job, and writes a styled HTML report to the
local `Job and CV` folder.

---

## Step 0 — Load CV Files (Local Folder First, Then Project Knowledge)

CV files can exist in two places. Resolve to a single authoritative version
**per CV label** before grading.

### 0a — Read local folder first

Read CV files from the **parent** CV folder only:

```
C:\Users\97254\Desktop\Omer\CV\
```

Use Desktop Commander to list files in that folder, and record each file's
**last modified date** from filesystem metadata. Do **not** read from the
`Job and CV\` subfolder — those are working copies, not authoritative versions.

### 0b — Check Project knowledge files

Then check the CV files already present as Claude Project knowledge (uploaded
to this Project's context — not a separate search step). For each CV label,
note whether a Project-knowledge version exists.

**Limitation:** Project knowledge files do not expose reliable filesystem
modification dates. "Latest" therefore cannot always be determined by
timestamp alone for this source.

### 0c — Resolve to one version per label

For each CV label:
- If only the local file exists → use it.
- If only the Project-knowledge file exists → use it, and flag in the output
  that no local copy was found (the local folder is the source of truth going
  forward; this is a signal to sync it).
- If both exist → compare local `last modified date` against the Project
  knowledge file. If content is materially different (different keywords,
  different framing, different length), do not silently guess which is
  "latest" — flag the conflict in the output (`⚠️ CV_X version conflict —
  local and Project copies differ, verify manually`) and default to the
  **local file** as authoritative, since it has a verifiable timestamp and the
  Project copy does not.

### Active CV versions (use these only)

| Label | File | Target role framing |
|---|---|---|
| `CV_Pd` | `Omer_Tal_CV_Pd.docx` | Program Director / Portfolio PM / AI Program ownership |
| `CV_PSSup` | `Omer_Tal_CV_PSSup.docx` | Global Delivery / PS Leader / Org Build / P&L / people leadership |
| `CV_VEng` | `Omer_Tal_CV_VEng.docx` | VP Engineering / Software Dev Leadership / CTO-adjacent |
| `CV_VC` | `Omer_Tal_CV_VC.docx` | Financial governance / Delivery operations / PS efficiency / pipeline mgmt |
| `CV_Aidoc` | `Omer_Tal_CV_Aidoc.docx` | **Aidoc-specific only** — use exclusively for Aidoc applications |

**`CV_IC_PM_V0.3` is retired — do not recommend it.** Any role that would have
previously mapped to IC_PM_V0.3 should be mapped to `CV_Pd` instead.

Also check `Archive\` for role-specific CVs (e.g. `CV_AllCloud.docx`) that may
be relevant for specific companies.

For each resolved CV file, extract:
- **Version label** (e.g. "CV_Pd")
- **Source used** ("local" or "project knowledge") and last modified date if local
- **Target role framing** — what seniority/title does this CV lead with?
- **Top 10 keywords** — domains, technologies, methodologies most prominent
- **Differentiators** — what makes this version distinct from others?

If neither source has any CV files, proceed without grading and note it in
the output.

---

## Step 1 — Search Gmail for Job Alert Emails

Run these Gmail searches (last 30 days) using `search_threads`. Collect all
matching thread IDs and snippets.

### Alert email queries

```
from:jobalerts-noreply@linkedin.com newer_than:30d
from:jobs-listings@linkedin.com newer_than:30d
from:linkedin@e.linkedin.com subject:jobs newer_than:30d
from:noreply@drushim.co.il newer_than:30d
from:alerts@drushim.co.il newer_than:30d
from:noreply@alljobs.co.il newer_than:30d
from:Alljobs@alljob.co.il newer_than:30d
from:noreply@glassdoor.com newer_than:30d
from:alerts@glassdoor.com newer_than:30d
```

> **Note:** The confirmed AllJobs sender is `Alljobs@alljob.co.il` (not alljobs.co.il).
> If zero results come back from any source, flag it to the user.

Fetch the full body of each thread using `get_thread`. For each job parse:

| Field | Notes |
|---|---|
| `title` | Job title as written |
| `company` | Company name |
| `location` | City / remote / hybrid |
| `source` | LinkedIn / Drushim / AllJobs / Glassdoor |
| `url` | Direct job URL if present |
| `date_found` | Date of the alert email |
| `description_snippet` | First 200 chars of job description if available |

---

## Step 2 — Deduplicate

A duplicate is any two rows where **both** `title` and `company` match
(case-insensitive, ignore punctuation). When deduplicating:
- Keep the row with the earliest `date_found`
- Prefer the URL with the most detail (LinkedIn > others)
- Merge `source` field to show all sources (e.g. "LinkedIn, Glassdoor")
- Merge `description_snippet` — keep the longer one

---

## Step 3 — Check Application Status

### 3a — Application confirmation emails

```
from:jobs-noreply@linkedin.com subject:"application" newer_than:30d
from:jobalerts-noreply@linkedin.com subject:"applied" newer_than:30d
subject:("application received" OR "application submitted" OR "we received your application" OR "תודה על פנייתך") newer_than:30d
```

Extract: company name + job title from each confirmation email.

### 3b — Rejection emails

Search **both inbox and trash**:

```
subject:("we regret" OR "not moving forward" OR "other candidates" OR "position has been filled" OR "לא התקדמנו" OR "לא נבחרת") newer_than:30d
in:trash subject:("we regret" OR "not moving forward" OR "other candidates" OR "unfortunately" OR "לא נבחרת") newer_than:30d
```

Extract: company name + job title where identifiable.

### 3c — Match to job list

For each job in the deduplicated list, set `status`:
- `Applied` — confirmation email matches this company + role
- `Rejected` — rejection email matches
- `Recruiter Contact` — arrived as LinkedIn Recruiter InMail
- `New` — no match found

Fuzzy-match company name (ignore Ltd/Inc/בע"מ suffixes). Title match: email
subject contains the job title or vice versa. When uncertain, default to `New`.

---

## Step 4 — Grade Job Fit and Recommend CV Version

**Only run this step if CV files were successfully loaded in Step 0.**

For each job, evaluate fit using title, company, location, and description snippet.

### Fit scoring criteria

| Dimension | What to assess |
|---|---|
| Seniority match | Does the role match VP/AVP/Director/Head-of level? |
| Domain match | Delivery, operations, Agentic AI, OTC, SaaS, FinTech, Insurance, Telco? |
| Scope match | Global/multi-country teams, P&L ownership, enterprise clients? |
| Geography | Israel-based or remote-friendly? Flag roles requiring relocation |
| Title alignment | Realistic match, or a stretch/underreach? |

### Fit levels

- **High** — Strong match on seniority + at least 2 domain signals. Prioritize.
- **Med** — Partial match. Seniority fits but domain adjacent, or vice versa.
- **Low** — Weak match. Significant seniority gap, unrelated domain, or relocation required.

### CV version recommendation

Map each job to the CV whose framing most closely mirrors the job's language:
- AI Program / PM / portfolio ownership → `CV_Pd`
- Global delivery / P&L / org-build / people leadership → `CV_PSSup`
- Engineering org / VP Eng / CTO-adjacent → `CV_VEng`
- Financial governance / PS efficiency / delivery ops → `CV_VC`
- Aidoc only → `CV_Aidoc`
- Company-specific (e.g. AllCloud) → check `Archive\` first

Never recommend `CV_IC_PM_V0.3`.

### Fit reasoning

One concise sentence (max 15 words):
- "Strong AI delivery match, global scope, seniority aligned"
- "Good domain fit but title is one level below target"
- "Unrelated domain, retain only if company is strategic target"

---

## Step 5 — Write HTML Report to Local Folder

Write a self-contained HTML file using Desktop Commander `write_file`:

**Path:** `C:\Users\97254\Desktop\Omer\CV\Job and CV\Job_Scan_{YYYY-MM-DD}.html`

The file must follow the exact structure and CSS of the existing reference file:
`C:\Users\97254\Desktop\Omer\CV\Job and CV\Job_Scan_2026-06-25.html`

### Embedded CSS (paste verbatim into `<head>`)

```
body { font-family: Segoe UI, Arial, sans-serif; background: #f0f4f8; margin: 0; padding: 20px; color: #1a1a2e; }
h1 { background: #1F4E79; color: white; padding: 16px 24px; border-radius: 8px; margin-bottom: 6px; font-size: 1.4em; }
.subtitle { color: #555; margin-bottom: 20px; font-size: 0.9em; padding-left: 4px; }
.stats { display: flex; gap: 12px; margin-bottom: 20px; flex-wrap: wrap; }
.stat { background: white; border-radius: 8px; padding: 12px 20px; box-shadow: 0 1px 4px rgba(0,0,0,0.1); text-align: center; min-width: 100px; }
.stat .num { font-size: 1.8em; font-weight: bold; }
.stat .lbl { font-size: 0.75em; color: #666; margin-top: 2px; }
.green .num { color: #2e7d32; } .yellow .num { color: #f57f17; } .red .num { color: #c62828; }
.blue .num { color: #1565c0; } .purple .num { color: #6a1b9a; }
.alert-box { background: #fff3cd; border: 1px solid #ffc107; border-radius: 8px; padding: 12px 16px; margin-bottom: 20px; font-size: 0.9em; }
.alert-box strong { color: #856404; }
section { margin-bottom: 28px; }
h2 { font-size: 1em; font-weight: bold; padding: 8px 14px; border-radius: 6px; margin: 0 0 10px 0; }
.h-high { background: #c6efce; color: #1a5c1a; }
.h-med  { background: #ffeb9c; color: #7d5a00; }
.h-low  { background: #ffc7ce; color: #7d0000; }
.h-track { background: #dde3ea; color: #333; }
.h-cv { background: #375623; color: white; }
table { width: 100%; border-collapse: collapse; background: white; border-radius: 8px; overflow: hidden; box-shadow: 0 1px 4px rgba(0,0,0,0.1); font-size: 0.82em; }
th { background: #2E75B6; color: white; padding: 8px 10px; text-align: left; }
td { padding: 9px 10px; border-bottom: 1px solid #e8e8e8; vertical-align: top; line-height: 1.4; }
tr:last-child td { border-bottom: none; }
tr:hover td { background: #f5f9ff; }
.fit-h { background: #c6efce; color: #1a5c1a; font-weight: bold; padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
.fit-m { background: #ffeb9c; color: #7d5a00; font-weight: bold; padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
.fit-l { background: #ffc7ce; color: #7d0000; font-weight: bold; padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
.status-new  { background: #e3f2fd; color: #0d47a1; padding: 2px 6px; border-radius: 10px; font-size:0.85em; }
.status-appl { background: #BDD7EE; color: #00008b; padding: 2px 6px; border-radius: 10px; font-size:0.85em; }
.status-rej  { background: #e0e0e0; color: #333; padding: 2px 6px; border-radius: 10px; font-size:0.85em; }
.status-rec  { background: #e2efda; color: #2e5e1a; padding: 2px 6px; border-radius: 10px; font-size:0.85em; }
.cv-tag { background: #1F4E79; color: white; padding: 1px 6px; border-radius: 4px; font-size: 0.8em; white-space: nowrap; }
.src-li { background: #0077b5; color:white; padding:1px 5px; border-radius:3px; font-size:0.75em; }
.src-dr { background: #6d28d9; color:white; padding:1px 5px; border-radius:3px; font-size:0.75em; }
.src-aj { background: #ef821b; color:white; padding:1px 5px; border-radius:3px; font-size:0.75em; }
.src-gd { background: #0caa41; color:white; padding:1px 5px; border-radius:3px; font-size:0.75em; }
.star { color: #ffa000; }
.warn { color: #e65100; font-weight: bold; }
.notes { color: #555; font-size:0.85em; }
.rej-pi { background: #ff9999; font-weight: bold; }
```

### Document structure (sections in order)

1. `<h1>📋 Daily Job Scan — {Month DD, YYYY}</h1>`
2. `.subtitle`: sources found + `C:\Users\97254\Desktop\Omer\CV\`
3. `.stats` bar: High Fit, Med Fit, Low Fit, Applied, Rejected, Recruiter counts
4. `.alert-box` (omit entirely if no urgent items)
5. HIGH FIT section (`h2.h-high "🟢 HIGH FIT — Apply Now"`) — full 10-column table
6. MED FIT section (`h2.h-med "🟡 MEDIUM FIT — Review & Decide"`) — full 10-column table
7. LOW FIT section (`h2.h-low "🔴 LOW FIT — Skip (listed for completeness)"`) — 5-column table: Date, Job Title, Company, Source, Reason to Skip
8. Application Tracker (`h2.h-track "📊 Application Tracker — Open & Rejected"`) — 5-column table: Date, Job Title, Company, Outcome, Notes
9. CV Version Key (`h2.h-cv "📄 CV Version Key — Use files from CV\ folder"`) — 4-column table: Label, File (CV\ folder), Use For, Examples Today
10. Footer: `Generated by daily job scan — {Month DD, YYYY}`

### Source badge classes

- LinkedIn → `<span class="src-li">LinkedIn</span>`
- AllJobs → `<span class="src-aj">AllJobs</span>`
- Glassdoor → `<span class="src-gd">Glassdoor</span>`
- Drushim → `<span class="src-dr">Drushim</span>`
- LinkedIn Recruiter → `<span class="src-li">LinkedIn Recruiter</span>`

Append qualifiers as plain text after the badge: "HOT", "Easy Apply", "(trash — missed)"

### Row element classes

- Fit badges: `<span class="fit-h">High</span>`, `<span class="fit-m">Med</span>`, `<span class="fit-l">Low</span>`
- Status badges: `class="status-new"`, `class="status-appl"`, `class="status-rej"`, `class="status-rec"`
- CV tags: `<span class="cv-tag">CV_Pd</span>`
- Post-interview rejections: add `class="rej-pi"` to the outcome `<td>`
- Star: `<span class="star">⭐</span>`, Warning: `<span class="warn">⚠️</span>`

### Alert box rules

Include only for genuinely urgent items:
- Unread recruiter InMails older than 3 days
- HOT jobs posted today
- Jobs recovered from trash that may be time-sensitive
- Source/ATS issues requiring user action

Omit entirely if no urgent items.

---

## Step 6 — Report to User

After writing the HTML file, reply with:

```
Job scan complete — {date}

📋 {N} unique jobs found
🟢 {N} High fit  |  🟡 {N} Med fit  |  🔴 {N} Low fit
✅ {N} applied  |  ❌ {N} rejected  |  🆕 {N} new

Sources: LinkedIn ({n}), Drushim ({n}), AllJobs ({n}), Glassdoor ({n})
CV versions used for grading: CV_Pd, CV_PSSup, CV_VEng, CV_VC

📄 Report saved: C:\Users\97254\Desktop\Omer\CV\Job and CV\Job_Scan_{date}.html

🎯 Top picks (High fit, New):
  • {Job Title} @ {Company} — {Fit Reason} → use {CV version}
  • ...

⚠️ Flags (if any):
  - No emails found from [source] — verify sender domain
  - {N} rejections could not be matched to a specific job
```

List up to 5 top picks (High fit + status New).

---

## Step 7 — No Email Send: Report Stays Local

**Do not attempt to send an email.** The Gmail MCP connector available in
this environment supports search and read/label operations only — it has no
send or draft-with-attachment function. Do not call a send tool, do not
attempt a `create_draft` workaround, and do not tell the user an email was
sent. Treat "the report is saved locally" as the terminal state of this
skill, not an intermediate step before delivery.

After Step 6's chat summary, close with:
> "Report saved locally at `C:\Users\97254\Desktop\Omer\CV\Job and CV\Job_Scan_{date}.html` — open it in a browser to view. This connector can't send email, so it won't land in your inbox automatically."

If a future run finds a Gmail MCP tool with genuine send capability attached
to this account, flag that discovery to the user and ask whether they want
this step re-enabled — do not silently start sending once a send-capable tool
exists.

---

## Edge Cases

- **No emails found at all**: Tell the user and stop. Do not write an empty HTML file.
- **Partial source failure**: Continue with available sources, flag missing ones.
- **No CV files in local folder**: Skip Steps 0 and 4. Leave Fit, Best CV, Fit Reason as "—".
- **Only one CV version**: Still run grading — fit score is useful even without version comparison.
- **Hebrew email bodies**: Parse Hebrew job titles and company names as-is. Do not translate.
- **Recruiter InMail forwarded to email**: Source = "LinkedIn Recruiter", status = "Recruiter Contact". Grade for fit.
- **Same job applied via multiple sources**: Consolidate into one row, set status Applied, keep fit grading.
- **Job description too short to grade**: Mark Fit as "?" and note "insufficient description for grading".
- **Jobs auto-deleted to LinkedIn trash**: Search trash for missed job alert emails — surface in Application Tracker as "MISSED" with warning flag.
