# Wayfind

An automated job-discovery and screening agent built with n8n, combining live job board APIs, AI verification, and automated reporting.

## What it does

Wayfind runs daily and automatically:
1. **Discovers** new job postings from multiple sources (RemoteOK, Remotive, Himalayas)
2. **Filters** out stale, duplicate, and clearly irrelevant postings using keyword pattern matching
3. **Verifies** each remaining candidate against a specific criteria profile using OpenAI, reading the actual posting text rather than trusting board metadata alone
4. **Deduplicates** against a Supabase database, so previously-checked companies aren't re-processed
5. **Reports** genuine matches via an auto-generated Google Doc and a Slack notification

## Why I built this

Manually screening job boards for a narrow, specific set of criteria (worldwide-remote, junior-friendly, no formal degree requirement, specific tech stack) turned out to be slow and error-prone — job boards routinely mislabel listings as "Remote" when they're actually country-restricted, or bury seniority requirements deep in the posting text. I built Wayfind to automate that screening honestly: every verdict is grounded in the actual scraped text, not a guess.

## Stack

- **n8n** — workflow orchestration
- **RemoteOK, Remotive, Himalayas** — public job board APIs (no auth required)
- **OpenAI (gpt-4o-mini)** — reads each posting and returns a structured verdict with reasoning
- **Supabase** — persistent storage and deduplication
- **Google Docs API** — automated report generation
- **Slack API** — delivery notifications

## Architecture

```
Daily Schedule Trigger
       │
       ▼
Fetch: RemoteOK + Remotive + Himalayas (parallel)
       │
       ▼
Filter Real Jobs (normalize + keyword pre-filter: title, seniority, location)
       │
       ▼
Check Already-Checked (Supabase lookup, skip duplicates)
       │
       ▼
Filter - Red Flag Keywords (regex pre-screen: country, seniority, degree, stack)
       │
       ▼
OpenAI - Verify & Synthesize (reads full posting text, returns structured verdict)
       │
       ▼
Parse Final Verdict
       │
       ▼
Keep Only MATCH / UNCLEAR
       │
       ├──► Supabase - Store Result
       └──► Build Report → Google Docs → Slack Notification
```

## Real engineering challenges solved

- **Boilerplate truncation bug**: cookie-consent banners on scraped pages were pushing real job content past the character limit sent to the AI, causing false negatives. Fixed by detecting content markers (`## Description`, etc.) and truncating from there instead of from the start.
- **Inconsistent API response shapes**: each job board returns a differently-shaped JSON response (bare array vs. nested `{jobs: [...]}` vs. different field names for the same data). Built a normalization layer so all sources feed into one consistent format downstream.
- **Dead link filtering**: postings can close between discovery and verification. Added a live-URL check to drop dead links before they reach the report.
- **Deduplication**: without a "seen before" check, the same postings would be re-processed and re-reported every run. Added a Supabase lookup step to filter out anything already logged.

## Setup

See `DAY1_SETUP.md` through `DAY4_SETUP.md` for the incremental build process, or import the final workflow JSON directly into your own n8n instance and configure your own API credentials for each service (Firecrawl/job board APIs, OpenAI, Supabase, Google, Slack).

## Status

Actively running on a daily schedule. Currently sourcing from 3 job boards, with more planned (We Work Remotely, Working Nomads, Arbeitnow).
