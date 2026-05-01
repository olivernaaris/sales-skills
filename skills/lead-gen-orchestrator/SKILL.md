---
name: lead-gen-orchestrator
description: "Top-level lead generation loop. Drives Claude for Chrome (or any browser/scraping source) through ICP-matched sources (LinkedIn, Reddit, Instagram, niche directories), enriches each lead via Hunter and Apollo, dedupes against Attio, and upserts the result. Use when you want to run an autonomous lead-gen session — define ICP once, then let it find/enrich/write leads for hours. Do NOT use for one-off lookups (call sales-hunter or sales-attio directly), for outbound sequences (use sales-cadence), or for ICP definition only (use sales-prospect-list)."
argument-hint: "[ICP description, source URL, target lead count]"
license: MIT
version: 0.1.0
tags: [sales, prospecting, automation, orchestration]
---
# Lead Generation Orchestrator

Drive a complete lead-generation loop end-to-end: ICP → source → research → enrich → dedup → write to Attio. Designed to compose the other skills in this repo and run unattended for long sessions.

## Step 0 — Read state

State files (`./leads-processed.jsonl`, `./leads-skipped.jsonl`, `./leads-failed.jsonl`, `./runs/*.md`) follow the schemas defined in **`/lead-gen-checkpoint`**. Read that skill before this one — every operation in this skill is named after a checkpoint operation.

Validation steps (per `/lead-gen-checkpoint` Step "Validation"):

1. Verify `./icp.md` exists. If missing, stop and tell the user to run `/sales-prospect-list` first. Do not guess an ICP.
2. Touch the three JSONL files (create empty if missing).
3. Create `./runs/` directory if missing.
4. Generate `run_id` as `YYYY-MM-DD-HHMM` (UTC) — used on every record from this run.
5. Write a header line to `./runs/<run_id>.md` immediately so a crashed run still leaves a trail.

## Step 1 — Gather minimal context

Ask only what's not already in `icp.md`:

1. **Source for this run?**
   - A) LinkedIn Sales Navigator search URL (use `/lead-gen-linkedin-chrome`)
   - B) Reddit subreddit / thread URL (use `/lead-gen-reddit-chrome`)
   - C) Instagram profile or hashtag (use `/lead-gen-instagram-chrome`)
   - D) Apollo search (use `/sales-apollo` directly, no Chrome)
   - E) Custom URL — I'll figure out the extractor

2. **Target count for this run?** Default: 25. Hard cap: 100/day per source to avoid bans.

3. **Attio list to write into?** Required. Format: list slug or list ID.

4. **Confidence threshold for Hunter email?** Default: 70. Below this, mark as `email_pending` and skip enrichment.

If the user already provided these in the prompt, skip the question and proceed.

## Step 2 — Per-lead loop

For each candidate URL/profile from the source. All file operations use the contract in **`/lead-gen-checkpoint`** — do not invent your own field names or paths.

```
1. If is_processed(url) → skip silently. (See /lead-gen-checkpoint Operations.)
   If is_failed_recently(url, hours=24) → skip; will retry tomorrow.

2. Extract structured fields per /sales-account-map rubric:
   - {name, title, company, company_domain, profile_url, signals[]}
   On extraction error → record_failed({step: "extract"}); continue.

3. Score against ICP from ./icp.md:
   - Hard filters (industry, company_size, geography) → reject if miss
   - Soft signals (recent funding, hiring, intent) → boost priority via /sales-intent
   - If reject → record_skipped({reason: "icp_miss", details: "<which filter>"}); continue.

4. Enrich via /sales-enrich chain:
   a. Hunter Email-Finder (name + company_domain)
   b. If confidence < threshold from icp.md OR no result → Apollo people_match fallback
   c. If still nothing AND icp.md allows email_pending → write with email=null, source signal preserved
      Else → record_skipped({reason: "no_email"}); continue.
   On transient error → record_failed({step: "enrich_hunter" | "enrich_apollo"}); continue.

5. Dedup via /sales-data-hygiene:
   a. Attio search-records by email (primary key)
   b. If hit → record_skipped({reason: "dedup_hit", attio_id: "<id>"}); continue.
   c. If miss but company_domain matches existing record → attach as new contact on existing company

6. Write via /sales-attio upsert-record:
   - Map fields per icp.md attio_field_mapping section
   - Add to target_list from icp.md
   - Tag with source ("linkedin" | "reddit" | "instagram" | "apollo") and run_id
   On 429/5xx → record_failed({step: "write"}); continue.
   On 401/403 → STOP run (auth broken).

7. record_processed({url, attio_id, email, email_confidence, name, company,
                     company_domain, source, run_id, extracted_at, written_at})
   This MUST be the last step — never write_processed before the Attio upsert succeeds.

8. Rate-limit: defer to the active Chrome driver's wait policy
   (LinkedIn 60–120s, Reddit 5–10s, Instagram 90–180s). Do NOT shorten.
```

## Step 3 — Stop conditions

Stop the run immediately and report when ANY of these trigger:

- Target lead count reached
- Source returns CAPTCHA, login wall, or "unusual activity" page (do NOT retry — flag for human)
- 3 consecutive enrichment failures (Hunter + Apollo both miss)
- 5 consecutive ICP misses (signals the source/search is wrong)
- Attio API returns 401/403 (auth broken)
- Hunter quota exceeded (HTTP 429 or quota error)

Never retry CAPTCHAs or anti-bot pages. Never reduce wait times to "catch up." Never skip dedup to hit the target.

## Step 4 — End-of-run report

Write the summary to `./runs/<run_id>.md` per the schema in **`/lead-gen-checkpoint`** ("Schema: `./runs/YYYY-MM-DD-HHMM.md`"). Counters are derived from the JSONL files using the `jq` recipes in that contract — do not maintain in-memory counters in this skill.

Required sections in the run summary:

- Source URL + ICP slug
- Stop reason (target_reached | captcha | rate_limit | quota | manual_stop | auth_broken)
- Counters: processed, dedup_hits, icp_misses, identity_bridge_failures, enrichment_failures, total_examined
- Per-source breakdown
- Failures detail (URL + error + step + retry advice)
- Next action recommendation

## Composition map

This skill is a controller — it does not extract or write anything itself. It dispatches to:

| Step | Skill |
|---|---|
| State schema + operations | `/lead-gen-checkpoint` |
| ICP definition (one-time) | `/sales-prospect-list` |
| Source-specific extraction | `/lead-gen-linkedin-chrome`, `/lead-gen-reddit-chrome`, `/lead-gen-instagram-chrome` |
| Per-profile fields | `/sales-account-map` |
| Buying signals | `/sales-intent` |
| Enrichment chain | `/sales-enrich` → `/sales-hunter` → `/sales-apollo` |
| Dedup | `/sales-data-hygiene` |
| Write | `/sales-attio` |

If a sub-skill is missing or fails, stop the run — never silently skip a step.

## Anti-patterns (do NOT do these)

- Running multiple sources in one session (LinkedIn + Reddit + Instagram together) — context drifts, selector mistakes compound
- Skipping dedup to "save time" — pollutes Attio
- Retrying after CAPTCHA — escalates to permanent ban
- Writing partial records (no email, no title) — lowers Attio data quality
- Hard-coding ICP in this skill — always read from `./icp.md` so it's tunable per project
- Running without checkpoint files — a crash mid-run will re-do everything; create them per `/lead-gen-checkpoint` Step "Validation"
- Inventing your own JSONL field names — every record must conform to the schemas in `/lead-gen-checkpoint`. If a field doesn't fit, the contract needs to change first
- Maintaining in-memory counters — derive run stats from the JSONL via `jq` recipes in `/lead-gen-checkpoint`, so a crashed run still produces correct numbers
