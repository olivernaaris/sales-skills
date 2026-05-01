---
name: lead-gen-linkedin-chrome
description: "Drives Claude for Chrome through LinkedIn (Sales Navigator, Recruiter, or basic search) to extract lead profiles into the structured format consumed by /lead-gen-orchestrator. Handles pagination, rate-limiting, anti-bot detection, and per-profile field extraction. Use when the orchestrator selects LinkedIn as the source. Do NOT use for outbound messaging (use /sales-cadence), for LinkedIn Ads campaigns, or as a standalone scraper without the orchestrator — the orchestrator owns dedup and writes."
argument-hint: "[Sales Nav search URL or profile list URL, target count]"
license: MIT
version: 0.1.0
tags: [sales, prospecting, linkedin, chrome, computer-use]
---
# LinkedIn Chrome Driver

Drive Claude for Chrome through a LinkedIn Sales Navigator (or basic) search and extract structured lead data. This skill is invoked by `/lead-gen-orchestrator` — it should not be run standalone except for testing.

## ToS reality check (read first)

LinkedIn's User Agreement prohibits automated scraping. This skill is intentionally **conservative** to look like normal human browsing:

- Hard cap **100 profiles per account per day** — going higher gets accounts restricted within hours
- Use the user's own logged-in session (Claude for Chrome attaches to their browser)
- Do NOT use multi-account rotation, headless mode, or proxy pools — those escalate to permanent ban
- A restricted account is recoverable; a banned account is not

If the user has a Sales Navigator seat, prefer that — its UI is designed for high-volume prospecting and tolerates faster click-throughs than basic search.

## Step 0 — Pre-flight

Before scraping anything, verify in the active Chrome tab:

1. User is logged into LinkedIn (look for nav avatar)
2. If using Sales Nav, the URL contains `/sales/` and the seat is active
3. The search results page has loaded fully (results count visible)
4. No "We've restricted your account" or "Verify it's you" banner

If any check fails → stop, report to user, do NOT proceed.

## Step 1 — Gather context (only if missing from invocation)

1. **Search URL provided?** Required. Should be a Sales Nav search URL with filters already applied (industry, title, company size, geography). If missing, ask the user for one.

2. **Target count for this run?** Default 25. Hard cap 100. If user requests > 100, refuse and explain the ban risk.

3. **Sales Nav vs basic LinkedIn?**
   - A) Sales Navigator (preferred — better data, tolerates faster pacing)
   - B) Basic LinkedIn search (slower pacing required)
   - C) LinkedIn Recruiter (similar to Sales Nav)

## Step 2 — Per-profile extraction loop

For each result on the search page:

```
1. Open profile in same tab (avoid opening tabs — looks bot-like)
2. Wait for profile to fully render (4–8s randomized)
3. Extract:
   - name              → from H1 in profile header
   - title             → first line under name
   - company           → company name in current experience
   - company_url       → href of company link in current experience
   - company_domain    → resolve via "Visit company website" button if present, else null
   - location          → location line in profile header
   - profile_url       → canonical URL (strip tracking params)
   - signals[]         → list of detected signals (see below)
4. Output one JSON line in this exact shape:
   {"source": "linkedin", "name": "...", "title": "...", "company": "...",
    "company_domain": "...", "profile_url": "...", "location": "...",
    "signals": [...], "extracted_at": "ISO8601"}
5. Hand control back to /lead-gen-orchestrator for enrichment + dedup + write
6. Wait 60–120s (randomized) before next profile — STRICT, do not shorten
```

### Signals to detect (feed `/sales-intent`)

Scan the profile and "Activity" tab for:

- **Recent job change** — "Started [role] at [company]" within 90 days → `signal: "new_role_<days>d"`
- **Hiring signals** — "We're hiring" posts, open job links → `signal: "hiring_<role>"`
- **Funding announcements** — recent post about Series A/B/C → `signal: "funding_<round>"`
- **Speaking / events** — recent conference talks → `signal: "speaker_<event>"`
- **Tech mentions** — keywords matching the user's ICP intent topics → `signal: "tech_<keyword>"`
- **Mutual connections** — count if visible → `signal: "mutuals_<count>"`

If no signals found, output `signals: []` — the orchestrator can still proceed, signals just boost priority.

## Step 3 — Pagination

After processing all profiles on a results page:

1. Wait 90–180s before clicking "Next page" — page transitions are the highest-risk action
2. Click LinkedIn's native pagination only — never modify the URL `?page=N` parameter directly (looks bot-like)
3. Verify new page loaded (URL changed, results changed) before continuing
4. If pagination button is disabled / missing → no more results, stop cleanly

## Step 4 — Anti-bot detection (stop conditions)

Stop **immediately** and report to the orchestrator if any of these appear at any time:

| Signal | Meaning | Action |
|---|---|---|
| CAPTCHA challenge | LinkedIn flagged the session | Stop, do NOT solve, flag for user |
| "Verify it's you" page | Login challenge | Stop, user must verify manually |
| "We've restricted your account" | Soft ban started | Stop, no LinkedIn for 24–72h |
| "Unusual activity" warning | Pre-restriction warning | Stop, slow down future runs |
| Search results suddenly empty | Throttling | Stop, retry tomorrow |
| Profile page won't load (>30s) | Rate-limited | Stop, wait an hour minimum |
| HTTP 999 in any request | LinkedIn anti-bot HTTP code | Stop immediately |

Never:
- Solve a CAPTCHA programmatically
- Refresh through a restriction banner
- Retry a failed page load more than once
- "Catch up" by reducing wait times after a slow start

## Step 5 — Output to orchestrator

After each successful profile extraction OR a stop event, return to `/lead-gen-orchestrator` with:

```json
{
  "status": "ok" | "stop",
  "stop_reason": null | "captcha" | "restriction" | "verify" | "throttle" | "no_more_results",
  "lead": { ... profile JSON ... } | null,
  "page_url": "current results URL",
  "next_action": "continue" | "halt_24h" | "halt_indefinite"
}
```

The orchestrator uses `next_action` to decide whether to skip LinkedIn for the rest of the run / day.

## What this skill does NOT do

- **Connection requests / messaging** — out of scope, would massively escalate ban risk
- **Profile photo / image extraction** — not needed for Attio
- **Full work history scraping** — only current role; deeper history wastes tokens and time
- **Email lookup on LinkedIn directly** — leave that to `/sales-hunter` and `/sales-apollo` after extraction
- **Multi-account orchestration** — single account, single session, single browser

## Field mapping reference

The output JSON maps to Attio (via `/sales-attio`) like so:

| Output field | Attio attribute |
|---|---|
| `name` | `name` |
| `title` | `job_title` |
| `company` | `company` (relationship) |
| `company_domain` | `company.domains[0]` |
| `profile_url` | `linkedin_url` (custom attr) |
| `location` | `location` |
| `signals[]` | `intent_signals` (custom multi-select) |
| `source: "linkedin"` | `lead_source` |

The orchestrator handles the actual write — this skill only produces the JSON.

## Anti-patterns (do NOT do these)

- Running headless or with a fresh browser profile per run — looks like a bot
- Scraping >100 profiles/day per account
- Opening profiles in new tabs (humans don't, mostly)
- Skipping pre-flight checks "just this once"
- Modifying URL parameters to skip pages
- Retrying after a restriction banner — wait 72h minimum
- Running this skill at 3am — humans browse during business hours; pattern this run during 9am–6pm in the user's timezone
