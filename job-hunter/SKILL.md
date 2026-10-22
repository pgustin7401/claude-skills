---
name: job-hunter
description: Run the daily job search pipeline -- ATS scrape, Indeed search, score, tailor resumes, rebuild dashboard, and auto-send a summary email.
---

# Job Hunter

Run Phil Gustin's daily job search pipeline: scrape ATS boards, search Indeed, score postings, tailor resumes, rebuild the dashboard, and auto-send a summary email to phil.gustin@gmail.com.

---

## Setup

All pipeline files live on the user's device at:
```
$HOME/mnt/Claude/job_hunter/
```

Key paths (all relative to that root):
- `data/jobs.json` — canonical posting store, keyed by URL
- `resumes/base_resume.json` — Phil's base resume (never edit this file)
- `resumes/tailored/` — output directory for tailored JSON + .docx files
- `dashboard/build_dashboard.py` — rebuilds `dashboard/dashboard.html`
- `scoring.py` — scoring logic (SCORE_THRESHOLD = 60)
- `tailoring/prompt.py` — tailoring instructions and `build_prompt(posting, base_resume_json)`
- `tailoring/docx_builder.py` — `build(json_obj, output_path)` builds a .docx

Dashboard artifact: https://claude.ai/artifact/GANBNdDL9FkWT36594yPSS

---

## Pipeline Steps

Run all steps via `mcp__remote-devices__device_bash` (NOT the cloud Bash tool).

### Step 1 — ATS Scraper

```bash
cd $HOME/mnt/Claude/job_hunter
timeout 35 python3 run_parallel.py 2>&1
```

If that produces no output or times out, fall back:
```bash
python3 run_fast.py 2>&1
```

The script writes net-new postings into `data/jobs.json` automatically. Note `new_today` from its output.

### Step 2 — Indeed Connector Search

Use `mcp__Indeed__search_jobs` to run these 8 queries. For each result, check whether the URL already exists as a key in `data/jobs.json`; skip duplicates.

Queries (all with `location="Remote, US"` and `country_code="US"`):
1. `search="VP Revenue Operations"`
2. `search="Head of Revenue Operations"`
3. `search="VP GTM Operations"`
4. `search="Head of GTM Operations"`
5. `search="Director Revenue Operations"`
6. `search="VP Sales Operations"`
7. `search="Senior Director Revenue Operations"`
8. `search="Head of Sales Operations"`

Score each net-new result using the rubric below. Add postings with score >= 60 to `data/jobs.json` with `status="new"`, `source="indeed_connector"`, `first_seen="<today>"`.

### Step 3 — Scoring Rubric

| Dimension | Max pts | Criteria |
|---|---|---|
| title_core | 50 | Title contains "Revenue Operations", "GTM Operations", or "Sales Operations" |
| title_directorial | 30 | Seniority is VP, Head of, Senior Director, or Director |
| geo_match | 20 | Remote or Austin TX |
| stage_match | 15 | Company is SaaS, fintech, healthcare IT, or enterprise software |
| keyword_density | 15 | JD mentions Salesforce, forecasting, pipeline, CRO, or AI/automation |
| priority_company | 10 | Company in priority list (see Job_Search_Context.md) |

Dream company bonus (+25, replaces priority_company): openai, stripe, anthropic, databricks, rippling, ramp, figma, linear, vercel.

SCORE_THRESHOLD = 60. Only add/tailor postings that meet or exceed this threshold.

### Step 4 — Resume Tailoring

**Cap: 5 tailored resumes per run.** Tailor in descending score order. Skip postings that already have `status="tailored"`.

**CRITICAL — NEVER fabricate:**
- No invented metrics, percentages, or dollar amounts
- No invented team sizes, dates, company names, or tools
- Every fact in the tailored resume must exist in the base resume

**MODERATE intensity rules:**
- Mirror the JD's vocabulary in bullets (e.g., if JD says "revenue engine" use that phrase)
- Reorder bullets to lead with most-relevant experience for that role
- Update `header.headline` and `summary` to reflect the role's title/focus
- Preserve all companies, dates, and metrics exactly as in base resume
- ASCII-safe punctuation only (no curly quotes, em-dashes use hyphens)
- Output must be valid JSON in the same shape as `resumes/base_resume.json`

**To tailor:**
1. Read `resumes/base_resume.json`
2. Read the job description (from `mcp__Indeed__get_job_details` or from the posting's stored description)
3. Apply MODERATE rules above, produce tailored JSON
4. Write JSON to `resumes/tailored/<slug>_<job_id>.json`
5. Build .docx:
```bash
cd $HOME/mnt/Claude/job_hunter && python3 -c "
import sys, json; sys.path.insert(0, '.')
from tailoring.docx_builder import build
build(json.load(open('resumes/tailored/<file>.json')), 'resumes/tailored/<file>.docx')
"
```
6. Update `data/jobs.json`: set `status="tailored"`, `tailored_resume_path="resumes/tailored/<file>.docx"`

**If the JD is empty or too short to tailor against:**
Set `status="tailored"`, `tailored_resume_path=null`, and add a `tailoring_note` explaining why. Do not fabricate content to fill gaps.

### Step 5 — Rebuild Dashboard

```bash
cd $HOME/mnt/Claude/job_hunter && python3 dashboard/build_dashboard.py 2>&1
```

Then stage `dashboard/dashboard.html` and republish the artifact:
```
mcp__remote-devices__device_stage_files -> Artifact publish to https://claude.ai/artifact/GANBNdDL9FkWT36594yPSS
```

### Step 6 — Send Summary Email

Use `mcp__Gmail__send_message` to auto-send to phil.gustin@gmail.com.

**Subject:** `Job Hunter -- [N] new postings -- [YYYY-MM-DD]`

**Body:**

```
Job Hunter Daily Summary -- [date]

NEW POSTINGS TODAY: [N total] ([N] from ATS, [N] from Indeed connector)

[For each new posting above threshold, sorted by score desc:]
* [Company] -- [Title] | Score: [N] | [Location]
  [URL]

TAILORED RESUMES BUILT: [N]
[List of company -- title -- filename]

[If any skipped:]
SKIPPED (JD too short): [company -- title -- reason]

PIPELINE TOTAL: [N] postings tracked
Dashboard: https://claude.ai/artifact/GANBNdDL9FkWT36594yPSS
```

---

## Strategy Reference

Full target company list, title rules, and industry watchlists are in:
```
$HOME/mnt/Claude/Projects/Job Search/Job_Search_Context.md
```

Read it at the start of each run to apply current priority companies and title matching rules.

---

## Output Rules

- **Prioritize** remote roles and Austin TX
- **Flag separately** any adjacent titles (Chief of Staff to CRO, VP inside a CRO org)
- **Never modify** `resumes/base_resume.json`
- **Always update** `data/jobs.json` after each change (ATS results, connector results, tailoring status)
- **Set `updated_at`** to today's date in `data/jobs.json` after each run
