# job-hunter

On-demand job search pipeline for Phil Gustin. Invoke by saying "run job hunter" in a Cowork session with the skill installed.

## What it does

1. Runs the ATS scraper (`run_parallel.py` / `run_fast.py`)
2. Searches Indeed connector across 8 title queries
3. Scores net-new postings and adds those >= 60 to `data/jobs.json`
4. Tailors up to 5 resumes (MODERATE intensity, no fabrication)
5. Rebuilds the dashboard artifact
6. Auto-sends a summary email to phil.gustin@gmail.com via Gmail

## Requirements

- Claude desktop app running (device bridge needed for on-device scripts)
- Indeed connector active
- Gmail connector active
- Pipeline files at `C:\Users\philg\Documents\Claude\job_hunter\`

## Files

- `SKILL.md` — skill instructions loaded by Claude
- `Job_Search_Context.md` — target companies, title rules, industry watchlists
