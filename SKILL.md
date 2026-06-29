---
name: job-scan
description: >
  Scans Gmail for job alert emails from LinkedIn, Glassdoor, Indeed, AllJobs,
  Drushim, and other job boards. Extracts and deduplicates all job listings,
  checks application confirmation and rejection emails, grades each job for fit
  against the user's CV files, recommends the best CV version per role, and
  writes a styled HTML report to a local folder. Also creates a Gmail draft
  summary email.
  Trigger whenever the user says "scan jobs", "run job scan", "check new jobs",
  "update my job list", "what new jobs came in", "job search update", "grade my
  jobs", "which CV should I send", or asks to refresh their job tracker.
compatibility: "Requires Gmail MCP and a file-access tool (Desktop Commander MCP or equivalent) for reading local CV files and writing the HTML report."
---

# Job Scan Skill

Scans Gmail for job alert emails, extracts and deduplicates listings, checks
application and rejection status, grades each job against the user's CV files,
recommends the best CV version per role, and writes a styled HTML report to a
local folder of the user's choice.

---

## Step 0 — Configure User Settings

Before doing anything else, establish the following configuration. If any value
is unknown, ask the user once before proceeding.

| Setting | Description |
|---|---|
| `cv_folder` | Local path to the folder containing the user's CV files |
| `output_folder` | Local path where the HTML report should be saved |
| `report_email` | Email address to send the draft report to |
| `target_role` | The user's target job title / seniority (e.g. "Senior PM", "VP Engineering") |
| `target_domains` | Industries or domains the user is targeting (e.g. "SaaS, FinTech, AI") |
| `target_geography` | Location preference (e.g. "Israel", "Remote", "New York") |
| `scan_days` | How many days back to scan (default: 30) |

If the user has provided any of these in the conversation, use them directly.
If this is a recurring run and settings were established previously, reuse them.

### Load CV files

Read all CV/resume files from `cv_folder`. Supported formats: `.docx`, `.pdf`, `.txt`.
For each file found, extract:
- **Label** — short version name derived from the filename (e.g. "CV_Pd", "Resume_v3")
- **Last modified date** — from filesystem metadata
- **Target role framing** — what seniority/title does this CV lead with?
- **Top keywords** — domains, technologies, methodologies most prominent
- **Differentiators** — what makes this version distinct from others?

If no CV files are found, skip Steps 0 and 4, note it in the report, and
continue with the rest of the scan.

---

## Step 1 — Search Gmail for Job Alert Emails

Run the following Gmail searches using `search_threads`. Adapt based on which
job boards the user is subscribed to. Collect all matching thread IDs.

### Common alert sender patterns (adjust for the user's actual sources)

```
from:jobalerts-noreply@linkedin.com newer_than:{scan_days}d
from:jobs-listings@linkedin.com newer_than:{scan_days}d
from:linkedin@e.linkedin.com subject:jobs newer_than:{scan_days}d
from:noreply@glassdoor.com newer_than:{scan_days}d
from:alerts@glassdoor.com newer_than:{scan_days}d
from:alert@indeed.com newer_than:{scan_days}d
from:jobalert@indeed.com newer_than:{scan_days}d
from:Alljobs@alljob.co.il newer_than:{scan_days}d
from:noreply@drushim.co.il newer_than:{scan_days}d
```

If zero results come back from a source, flag it — the sender domain may have
changed. Do not stop the scan; continue with sources that did return results.

Also search LinkedIn trash for missed alert emails:
```
in:trash from:jobalerts-noreply@linkedin.com newer_than:{scan_days}d
in:trash from:jobs-listings@linkedin.com newer_than:{scan_days}d
```

Fetch the full body of each thread using `get_thread`. For each job listing
found in the email body, parse:

| Field | Notes |
|---|---|
| `title` | Job title as written |
| `company` | Company name |
| `location` | City / remote / hybrid |
| `source` | LinkedIn / Glassdoor / Indeed / AllJobs / Drushim / other |
| `url` | Direct job URL if present |
| `date_found` | Date of the alert email |
| `description_snippet` | First 300 characters of job description if available |
| `from_trash` | true if recovered from Gmail trash |

---

## Step 2 — Deduplicate

A duplicate is any two rows where **both** `title` and `company` match
(case-insensitive, ignore punctuation). When deduplicating:
- Keep the row with the earliest `date_found`
- Prefer the URL with the most detail
- Merge the `source` field to show all sources (e.g. "LinkedIn, Glassdoor")
- Merge `description_snippet` — keep the longer one
- If one came from trash and one from inbox, keep the inbox version but flag the trash one as "also in trash"

---

## Step 3 — Check Application Status

### 3a — Application confirmation emails

Search for confirmation emails across inbox and sent:
```
subject:("application received" OR "application submitted" OR "we received your application" OR "thank you for applying" OR "תודה על פנייתך" OR "הגשת מועמדות") newer_than:{scan_days}d
from:jobs-noreply@linkedin.com subject:"application" newer_than:{scan_days}d
```

Extract company name and job title from each.

### 3b — Rejection emails

Search **both inbox and trash**:
```
subject:("we regret" OR "not moving forward" OR "other candidates" OR "position has been filled" OR "unfortunately" OR "לא התקדמנו" OR "לא נבחרת" OR "we have decided") newer_than:{scan_days}d
in:trash subject:("regret" OR "not moving forward" OR "unfortunately" OR "other candidates") newer_than:{scan_days}d
```

Note any post-interview rejections (where subject or body indicates an interview took place before the rejection).

### 3c — Match status to job list

For each job in the deduplicated list, assign `status`:
- `Applied` — a confirmation email matches this company + role
- `Rejected` — a rejection email matches (flag post-interview rejections separately)
- `Recruiter Contact` — arrived via recruiter InMail (`messages-noreply@linkedin.com`)
- `New` — no match found

Fuzzy-match on company name (ignore Ltd/Inc/בע"מ suffixes). For title matching,
the email subject should contain the job title or vice versa. When uncertain,
default to `New` and add a note.

---

## Step 4 — Grade Job Fit and Recommend CV Version

**Only run if CV files were loaded in Step 0.**

Evaluate each job using the user's configured `target_role`, `target_domains`,
and `target_geography` (set in Step 0).

### Fit scoring dimensions

| Dimension | What to assess |
|---|---|
| Seniority match | Does the role's level match the user's target seniority? |
| Domain match | Does the role's industry/domain match `target_domains`? |
| Scope match | Does the role's scope (team size, P&L, geography) match the user's track record? |
| Geography | Is the role in `target_geography` or explicitly remote-friendly? |
| Title alignment | Is the title a realistic match, or a stretch/underreach? |

### Fit levels

- **High** — Strong match on seniority + at least 2 domain signals. Prioritize applying.
- **Med** — Partial match. Seniority fits but domain is adjacent, or domain fits but title is off.
- **Low** — Weak match. Significant seniority gap, unrelated domain, or geography mismatch.

### CV version recommendation

Compare the job's language and emphasis against each CV version's differentiators
(extracted in Step 0). Recommend the CV version whose framing most closely mirrors
the job description. If only one CV exists, recommend it for all High/Med fits.

### Fit reasoning

Write a single sentence (max 15 words) explaining the fit score:
- "Strong AI delivery match, global scope, seniority aligned"
- "Good domain fit but title is one level below target"
- "Unrelated domain — skip unless company is a strategic priority"

---

## Step 5 — Write HTML Report

Write a self-contained HTML file using the file-access tool:

**Filename:** `Job_Scan_{YYYY-MM-DD}.html`
**Path:** `{output_folder}\Job_Scan_{YYYY-MM-DD}.html`

### Embedded CSS (include verbatim in `<head>`)

```css
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
.status-new  { background: #e3f2fd; color: #0d47a1; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.status-appl { background: #BDD7EE; color: #00008b; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.status-rej  { background: #e0e0e0; color: #333; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.status-rec  { background: #e2efda; color: #2e5e1a; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.cv-tag { background: #1F4E79; color: white; padding: 1px 6px; border-radius: 4px; font-size: 0.8em; white-space: nowrap; }
.src-li { background: #0077b5; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-gd { background: #0caa41; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-in { background: #2557a7; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-aj { background: #ef821b; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-dr { background: #6d28d9; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-ot { background: #607d8b; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.star { color: #ffa000; }
.warn { color: #e65100; font-weight: bold; }
.notes { color: #555; font-size: 0.85em; }
.rej-pi { background: #ff9999; font-weight: bold; }
```

### Document structure (sections in order)

1. `<h1>📋 Daily Job Scan — {Month DD, YYYY}</h1>`
2. `.subtitle`: sources scanned + CV folder path
3. `.stats` bar: High Fit, Med Fit, Low Fit, Applied, Rejected, Recruiter
4. `.alert-box` (include only if there are urgent items — see rules below)
5. **HIGH FIT** section (`h2.h-high "🟢 HIGH FIT — Apply Now"`) — 10-column table
6. **MED FIT** section (`h2.h-med "🟡 MEDIUM FIT — Review & Decide"`) — 10-column table
7. **LOW FIT** section (`h2.h-low "🔴 LOW FIT — Skip"`) — 5-column table: Date, Job Title, Company, Source, Reason to Skip
8. **Application Tracker** (`h2.h-track "📊 Application Tracker"`) — 5-column: Date, Job Title, Company, Outcome, Notes
9. **CV Version Key** (`h2.h-cv "📄 CV Version Key"`) — 4-column: Label, File, Use For, Examples Today
10. Footer: `Generated by job-scan — {Month DD, YYYY}`

### Full-table column order (High and Med sections)

Date | Job Title | Company | Location | Source | Fit | Status | Best CV | Fit Reason | Notes

### Source badge classes

- LinkedIn → `<span class="src-li">LinkedIn</span>`
- Glassdoor → `<span class="src-gd">Glassdoor</span>`
- Indeed → `<span class="src-in">Indeed</span>`
- AllJobs → `<span class="src-aj">AllJobs</span>`
- Drushim → `<span class="src-dr">Drushim</span>`
- Other → `<span class="src-ot">{Source Name}</span>`

Append qualifiers as plain text: "HOT", "Easy Apply", "(trash — missed)", "Recruiter"

### Alert box rules

Include only for genuinely urgent items:
- Unanswered recruiter contacts older than 3 days
- HOT jobs posted today that warrant same-day application
- Jobs recovered from trash that may be time-sensitive

Omit entirely if no urgent items exist.

### Sort order within sections

Primary: Status (New before Applied before Rejected) — Secondary: Date descending

---

## Step 6 — Report to User

After writing the HTML file, reply with:

```
Job scan complete — {date}

📋 {N} unique jobs found
🟢 {N} High fit  |  🟡 {N} Med fit  |  🔴 {N} Low fit
✅ {N} applied  |  ❌ {N} rejected  |  🆕 {N} new

Sources: {source list with counts}
CV versions used for grading: {labels}

📄 Report saved: {output_folder}\Job_Scan_{date}.html

🎯 Top picks (High fit, New):
  • {Job Title} @ {Company} — {Fit Reason} → use {CV version}
  • ...

⚠️ Flags (if any):
  - No emails found from [source] — sender domain may have changed
  - {N} rejections could not be matched to a specific job
  - CV folder empty — fit grading skipped
```

List up to 5 top picks (High fit + status New). If none, say so clearly.

---

## Step 7 — Email Draft

Create a Gmail draft. Do **not** send automatically.

- **To:** `{report_email}`
- **Subject:** `Job Scan Report — {today's date}`
- **Body:**

```
Job Scan Report — {date}

SUMMARY
-------
Total unique jobs: {N}
🟢 High fit: {N}  |  🟡 Med fit: {N}  |  🔴 Low fit: {N}
✅ Applied: {N}  |  ❌ Rejected: {N}  |  🆕 New: {N}

Sources: {source breakdown}
CV versions used: {labels}

TOP PICKS (High fit, New)
-------------------------
1. {Job Title} @ {Company} — {Location}
   Fit: {Fit Reason}
   CV to use: {CV label}
   URL: {url}

(up to 5 picks)

FLAGS
-----
{Any warnings}

Full report: {output_folder}\Job_Scan_{date}.html
```

Tell the user: "Draft created — review and send when ready."

---

## Edge Cases

- **No emails found at all**: Stop and tell the user. Do not write an empty report.
- **Partial source failure**: Continue with available sources, flag the missing ones.
- **No CV files**: Skip Steps 0 and 4. Leave Fit, Best CV, Fit Reason as "—".
- **Only one CV version**: Still run fit grading — the score is useful even without version comparison.
- **Hebrew email bodies**: Parse as-is. Do not translate titles or company names.
- **Recruiter InMail**: Arrives as `messages-noreply@linkedin.com`. Source = "LinkedIn Recruiter", status = "Recruiter Contact". Grade for fit if description is available.
- **Same job from multiple sources**: Consolidate into one row, set combined source, keep fit grading.
- **Description too short to grade**: Mark Fit as "?" and note "insufficient description".
- **Jobs auto-deleted to trash**: Surface in Application Tracker with "MISSED" flag and ⚠️.
- **First run / no prior configuration**: Ask the user for `cv_folder`, `output_folder`, `report_email`, `target_role`, `target_domains`, and `target_geography` before scanning. Offer sensible defaults where possible.
