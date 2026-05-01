---
name: lead-gen-checkpoint
description: "Defines the file-based state schema shared by /lead-gen-orchestrator and all Chrome drivers (LinkedIn, Reddit, Instagram). Specifies the format and operations for ./leads-processed.jsonl, ./leads-skipped.jsonl, ./leads-failed.jsonl, and ./runs/*.md so that crash-resume, dedup, and audit logging stay consistent across skills. Use as the single source of truth for state — every skill that reads or writes lead state must follow this contract. Do NOT use as a runtime scheduler (use /schedule), as a database (it's append-only JSONL), or for storing ICP (that's ./icp.md)."
argument-hint: "[no args — this skill defines a contract, it's referenced by other skills]"
license: MIT
version: 0.1.0
tags: [sales, prospecting, state, checkpoint, contract]
---
# Lead-Gen Checkpoint Contract

Defines the file-based state shared by `/lead-gen-orchestrator` and the Chrome drivers. This skill is a **contract** — it does not perform actions itself. Other skills reference it to know exactly what to read and write.

When any skill in this repo touches lead state, it must conform to this contract. If you change a schema here, update the consuming skills.

## Why files, not a database

- Skills are prompts running inside Claude — no shared process, no shared memory
- Append-only JSONL is crash-safe and trivially diffable
- The user can `grep` / `jq` it directly to debug a run
- Migrating to Postgres later is a one-script job; over-engineering now is not worth it

If state grows past ~100k records (~50MB JSONL), revisit. Until then, files are the right answer.

## File layout

All paths are **relative to the user's working directory** (where they invoke the orchestrator), not the skills repo.

```
./icp.md                      # ICP definition (human-edited, written once by /sales-prospect-list)
./leads-processed.jsonl       # successfully written to Attio
./leads-skipped.jsonl         # rejected pre-write (ICP miss, no identity, dedup hit)
./leads-failed.jsonl          # transient errors (Hunter timeout, Apollo 500) — re-tryable
./runs/
  └── 2026-05-01-0930.md      # per-run human-readable summary
```

Never put these inside the skills repo. They're per-project user data.

## Schema: `./leads-processed.jsonl`

One JSON object per line, append-only. Written by `/lead-gen-orchestrator` after a successful Attio upsert.

```json
{
  "url": "https://linkedin.com/in/janedoe",
  "attio_id": "rec_abc123",
  "email": "jane@acme.com",
  "email_confidence": 92,
  "name": "Jane Doe",
  "company": "Acme Inc",
  "company_domain": "acme.com",
  "source": "linkedin",
  "run_id": "2026-05-01-0930",
  "extracted_at": "2026-05-01T09:32:14Z",
  "written_at": "2026-05-01T09:32:18Z"
}
```

**Required fields:** `url`, `attio_id`, `source`, `run_id`, `written_at`.
**Optional fields:** everything else. Null when unknown — never omit the key.

### Operations

| Operation | Implementation |
|---|---|
| `is_processed(url)` | `grep -F "\"url\": \"<url>\"" ./leads-processed.jsonl` — true if any match |
| `record_processed(record)` | Append one JSON line + newline. Never rewrite or compact. |
| `count_for_run(run_id)` | `grep -c "\"run_id\": \"<run_id>\"" ./leads-processed.jsonl` |

The orchestrator must call `is_processed(url)` before extraction starts — extracting a duplicate burns rate-limit budget.

## Schema: `./leads-skipped.jsonl`

Append-only. Written when a candidate is rejected before write.

```json
{
  "url": "https://linkedin.com/in/johnsmith",
  "reason": "icp_miss" | "no_identity_bridge" | "dedup_hit" | "below_karma_threshold" | "below_follower_threshold" | "bot_account" | "deleted_account",
  "details": "industry=consulting, ICP=saas",
  "source": "linkedin",
  "run_id": "2026-05-01-0930",
  "skipped_at": "2026-05-01T09:33:01Z",
  "attio_id": null
}
```

`attio_id` populated **only** when reason is `dedup_hit` (points to the existing record).

### Operations

| Operation | Implementation |
|---|---|
| `record_skipped(record)` | Append one JSON line |
| `skip_reasons_for_run(run_id)` | `jq` aggregate by reason — used in run summary |

Skipped URLs are NOT considered processed — if the ICP changes, a previously-skipped URL can be re-evaluated. Don't merge skipped into processed.

## Schema: `./leads-failed.jsonl`

Append-only. Written when a transient error occurs mid-pipeline. These ARE re-tryable on the next run.

```json
{
  "url": "https://linkedin.com/in/sarahlee",
  "error": "hunter_timeout" | "apollo_500" | "attio_429" | "chrome_load_timeout" | "unknown",
  "step": "extract" | "enrich_hunter" | "enrich_apollo" | "dedup" | "write",
  "error_message": "raw error text for debugging",
  "source": "linkedin",
  "run_id": "2026-05-01-0930",
  "failed_at": "2026-05-01T09:34:22Z",
  "retry_count": 0
}
```

### Operations

| Operation | Implementation |
|---|---|
| `record_failed(record)` | Append one JSON line |
| `is_failed_recently(url, hours=24)` | Check if URL appears in failed.jsonl within window — skip during current run, retry tomorrow |
| `retry_count_for_url(url)` | `grep` count |

If `retry_count >= 3` for a URL, treat it as `leads-skipped.jsonl` with reason `permanent_failure` and stop retrying.

## Schema: `./runs/YYYY-MM-DD-HHMM.md`

Human-readable run summary. Written by `/lead-gen-orchestrator` at the end of every run (success OR stop).

```markdown
# Run 2026-05-01 09:30

**Source:** https://linkedin.com/sales/search/people?...
**ICP:** SaaS founders, 50–500 employees, US/UK
**Stopped:** target_reached | captcha | rate_limit | quota | manual_stop

## Counters
- Processed (written to Attio):  18
- Dedup hits:                    4
- ICP misses:                    7
- Identity-bridge failures:      2
- Enrichment failures:           1
- Total candidates examined:     32

## Per-source
- linkedin: 18 processed, 4 dedup, 5 icp_miss

## Failures detail
- jane@acme.com — hunter_timeout (retry next run)

## Stop reason
target_reached at 18/25 — user requested stop.

## Next action
Resume tomorrow with same source URL. ICP looks good — 56% pass rate.
```

The orchestrator generates this; consuming skills do not write to it.

## Run ID format

`run_id` is always `YYYY-MM-DD-HHMM` (UTC). Generate once at orchestrator start. Every record from that run carries the same `run_id`. This lets the user reconstruct the run from the JSONL files alone.

## Concurrency

**Single-writer assumption** — only one orchestrator run touches these files at a time. Append-only writes are safe under POSIX, but concurrent runs will produce out-of-order records and break `/schedule`-based daily runs.

If the user runs `/schedule` for daily lead-gen, ensure runs don't overlap (default schedule cadence ≥ 1 hour apart). If they need parallel runs, partition by `run_id` directory:

```
./runs/2026-05-01-0930/
  ├── leads-processed.jsonl
  ├── leads-skipped.jsonl
  └── leads-failed.jsonl
```

This is opt-in — default is flat files at the project root.

## Backup / rotation

JSONL files grow append-only. At ~50MB they slow `grep`-based dedup noticeably. When that happens:

1. `mv leads-processed.jsonl leads-processed-archive-YYYY-MM-DD.jsonl`
2. Reseed `leads-processed.jsonl` from a one-time Attio dump (`/sales-attio` `list-records` → JSONL)
3. Or migrate to SQLite — schema fits in one table

Don't compact silently. The user should know when their dedup history rolls over.

## Atomicity rules

- **Append-only:** never rewrite or compact a JSONL file mid-run
- **Write last:** `record_processed` is the LAST step of each lead loop. If anything earlier fails, write `record_failed` instead — never both
- **No deletes:** if the user wants to "remove" a lead, mark it `deleted: true` in a NEW append. Don't modify existing lines
- **JSON one-line:** never pretty-print into JSONL — one record per line, period

## Reading from these files

Consuming skills should use simple `grep` / `jq` patterns, not load the whole file into memory.

```bash
# Is this URL already processed?
grep -F "\"url\": \"<url>\"" ./leads-processed.jsonl

# Count successes for a run
grep -c "\"run_id\": \"<run_id>\"" ./leads-processed.jsonl

# Get all skip reasons for a run
jq -r 'select(.run_id == "<run_id>") | .reason' ./leads-skipped.jsonl | sort | uniq -c

# Find URLs to retry from yesterday's failures
jq -r 'select(.run_id | startswith("2026-04-30")) | .url' ./leads-failed.jsonl
```

## Validation

Before each run, the orchestrator MUST:

1. Verify `./icp.md` exists. If not, stop and tell user to run `/sales-prospect-list` first.
2. Touch `./leads-processed.jsonl`, `./leads-skipped.jsonl`, `./leads-failed.jsonl` (create empty if missing).
3. Create `./runs/` directory if missing.
4. Write a header line to the new run summary file before the loop starts (so a crashed run still leaves a trail).

## Anti-patterns (do NOT do these)

- Modifying existing JSONL lines — append-only or nothing
- Storing the ICP in any of these files — that's `./icp.md`'s job
- Storing API keys here — environment variables only
- Running multiple orchestrator instances pointed at the same files concurrently
- Using these as a queue — they're a log, not a queue. The orchestrator decides what to do next from the source, not from the failed.jsonl
- Pretty-printing JSON across multiple lines — breaks the line-based ops
- Putting these inside the skills repo — they're per-project user data, not skill code
- Compacting / rewriting / "cleaning up" without a backup
