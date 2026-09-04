# Wayfind

An automated job-discovery and screening agent built with n8n, combining live job board APIs, AI verification, and automated reporting.

## What it does

Wayfind runs daily and automatically:

1. **Discovers** new job postings from 14 sources — general aggregators (RemoteOK, Remotive, Himalayas, Jobicy, Arbeitnow, Jooble, Findwork.dev, Reed, Adzuna), plus higher-signal sources with native filters or verified-employer listings (Working Nomads, We Work Remotely, Startup Jobs, Remote Landers, Hacker News "Who Is Hiring")
2. **Filters** out stale, duplicate, and clearly irrelevant postings — title/seniority keyword matching, a 30-day freshness cutoff, and location checks specific to each source's actual data shape
3. **Verifies** each remaining candidate against a specific criteria profile using OpenAI, reading the actual posting text rather than trusting board metadata alone
4. **Deduplicates** against a Supabase database, so previously-checked companies aren't re-processed
5. **Reports** genuine matches by appending to a single running Google Doc and sending a Slack notification

## Why I built this

Manually screening job boards for a narrow, specific set of criteria (worldwide-remote, junior-friendly, no formal degree requirement, specific tech stack) turned out to be slow and error-prone — job boards routinely mislabel listings as "Remote" when they're actually country-restricted, bury seniority requirements deep in the posting text, or keep listing jobs long after they've closed. I built Wayfind to automate that screening honestly: every verdict is grounded in the actual scraped text, not a guess.

## Stack

- **n8n** — workflow orchestration
- **14 job sources** — a mix of open, no-key APIs (RemoteOK, Remotive, Himalayas, Working Nomads, We Work Remotely, Arbeitnow, Jobicy, Remote Landers, Hacker News) and free-tier APIs requiring a key (Jooble, Findwork.dev, Reed, Adzuna)
- **OpenAI (gpt-4o-mini)** — reads each posting and returns a structured verdict with reasoning
- **Supabase** — persistent storage and deduplication
- **Google Docs API** — automated, append-only report generation
- **Slack API** — delivery notifications

## Architecture

```
Daily Schedule Trigger
       │
       ▼
Fetch: 14 sources in parallel
       │
       ▼
Filter Real Jobs (normalize 14 different response shapes, title/seniority
                   keyword filter, 30-day freshness filter)
       │
       ▼
Skip Already-Checked (Supabase lookup, skip duplicates)
       │
       ▼
Filter - Red Flag Keywords (regex pre-screen: country, seniority, degree, stack)
       │
       ▼
OpenAI - Verify & Synthesize (reads full posting text, returns structured verdict)
       │
       ▼
Parse Final Verdict (matches verdict back to source data by array position,
                      not by name — see below)
       │
       ▼
Supabase - Store Result (every verdict, not just matches)
       │
       ▼
Keep Only MATCH / UNCLEAR
       │
       └──► Build Report → Google Docs (append) → Slack Notification
```

## Real engineering challenges solved

- **Silent reprocessing loop**: only MATCH/UNCLEAR verdicts were being written to Supabase, so every rejected company got re-fetched and re-verdicted on every single run, forever, burning OpenAI calls on the same NO_MATCH result. Fixed by storing every verdict, not just the ones worth reporting.
- **Fourteen different response shapes**: five sources return their listings under a `jobs` key, three under `results`, and the rest each do their own thing. A single check on the outer key wasn't enough to tell them apart, so the normalization layer disambiguates by checking which unique fields actually show up on the first item.
- **Company-name matching broke dedup and apply links**: the pipeline originally re-matched OpenAI's output back to the original job data by comparing company names as text. That broke the moment OpenAI did its job well — correctly extracting "ProductNow" from a posting whose raw source field just said "PowerToFly.com" — which silently nulled the apply link and let the same company slip past deduplication indefinitely. Fixed by matching on array position instead, with the original name carried forward explicitly for dedup to key on.
- **Stale postings inflating the candidate pool**: several sources keep long-closed roles in their index. Added a 30-day freshness filter — but only where I actually verified the real date field against live data first. Three different formats turned up: ISO strings, Unix epoch timestamps in seconds (not milliseconds, which would have silently broken everything), and UK-style DD/MM/YYYY. Two sources (Remotive, Findwork) are left unfiltered rather than guessed, since I never confirmed a real date field for either.
- **Boilerplate truncation bug**: cookie-consent banners on scraped pages were pushing real job content past the character limit sent to the AI, causing false negatives. Fixed by detecting content markers (`## Description`, etc.) and truncating from there instead of from the start.
- **Report files piling up**: the Google Docs node was set to create a new document on every run. Switched to append-only updates against one fixed document.

## Setup

See `DAY1_SETUP.md` through `DAY4_SETUP.md` for the incremental build process, or import the workflow JSON directly into your own n8n instance and configure your own API credentials for each service (job board APIs, OpenAI, Supabase, Google, Slack).

## Status

Actively running on a daily schedule, sourcing from 14 job boards. Known gaps: Remotive and Findwork still need their real date field names confirmed for freshness filtering; some sources return US-timezone-restricted postings mislabeled as fully remote.
