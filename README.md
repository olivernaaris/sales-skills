# sales-skills

Lead-gen orchestration skills for Claude Code / Claude for Chrome. Drives a complete autonomous loop: **source → research → enrich → dedup → write to Attio**.

Built on top of [`sales-skills/sales`](https://github.com/sales-skills/sales) — vendors 9 of its skills and adds 5 owned skills that compose them into a runnable pipeline.

## What this does

```
Claude for Chrome  ─┬─►  LinkedIn Sales Nav
                    ├─►  Reddit (subreddits / threads)
                    └─►  Instagram (hashtags / profiles)
                              │
                              ▼
                      lead-gen-orchestrator
                              │
                              ├─►  /sales-account-map  (extract structured profile)
                              ├─►  /sales-intent       (score buying signals)
                              ├─►  /sales-enrich       ─►  Hunter, Apollo
                              ├─►  /sales-data-hygiene (dedup vs Attio)
                              └─►  /sales-attio        (idempotent upsert)
```

## Install

```bash
npx skills add olivernaaris/sales-skills
```

Or install just the orchestrator (and let it pull what it composes):

```bash
npx skills add olivernaaris/sales-skills --skill lead-gen-orchestrator
```

## Quick start

```bash
# 1. One-time: define your ICP
/sales-prospect-list

# This interviews you and writes ./icp.md to your project root.
# Or copy templates/icp.md from this repo and fill it in manually.

# 2. Run the orchestrator with a source URL
/lead-gen-orchestrator https://linkedin.com/sales/search/people?... target=25

# Or kick off a daily cron via /schedule:
/schedule daily 09:00 /lead-gen-orchestrator <source-url> target=25
```

The orchestrator reads `./icp.md`, dispatches to the right Chrome driver, and appends results to `./leads-processed.jsonl`. Crash-safe — restart picks up where it left off.

## Required MCPs

This repo assumes you have these connected in Claude:

| MCP | Used by | Purpose |
|---|---|---|
| **Hunter.io** | `sales-hunter` | Email finding + verification |
| **Apollo.io** | `sales-apollo` | People/company enrichment, fallback for Hunter |
| **Attio** | `sales-attio` | CRM writes (the single destination for all leads) |

Without these, the skills will load but the pipeline can't write anywhere.

## Skill catalog

### Owned (this repo, the lead-gen pipeline)

| Skill | What it does |
|---|---|
| `lead-gen-orchestrator` | Top-level loop. Reads ICP, dispatches Chrome drivers, runs enrichment chain, dedups, writes to Attio, logs to `./runs/`. |
| `lead-gen-linkedin-chrome` | Drives Claude for Chrome through LinkedIn Sales Nav. Hard cap 100 profiles/day per account. |
| `lead-gen-reddit-chrome` | Drives Chrome through Reddit subreddits / threads. Bridges username → identity via bio link. |
| `lead-gen-instagram-chrome` | Drives Chrome through IG hashtags / profiles. Linktree resolution as identity bridge. Hard cap 40 profiles/day. |
| `lead-gen-checkpoint` | File-based state contract. Defines `./leads-processed.jsonl`, `./leads-skipped.jsonl`, `./leads-failed.jsonl` schemas. |

### Vendored (from [`sales-skills/sales`](https://github.com/sales-skills/sales))

Pinned in `skills-lock.json` to commit `5065117`.

| Skill | What it does |
|---|---|
| `sales-do` | Router across all skills — natural-language intent → skill match. |
| `sales-prospect-list` | ICP definition framework. Run once to produce `./icp.md`. |
| `sales-account-map` | Per-profile structured extraction rubric. |
| `sales-intent` | Buying-signal scoring (hiring, funding, tool requests, recent role). |
| `sales-enrich` | Enrichment chain strategy — when to call Hunter vs Apollo vs manual. |
| `sales-hunter` | Hunter.io platform skill — Email-Finder, verification, retry patterns. |
| `sales-apollo` | Apollo.io platform skill — people_match, company search, fallback enrichment. |
| `sales-attio` | Attio platform skill — field mapping, idempotent upserts, list assignment. |
| `sales-data-hygiene` | Dedup strategy — search-records before write, merge rules. |

## Project layout

This is what the user's *project directory* (where they invoke skills) looks like, NOT this repo:

```
my-leadgen-project/
├── icp.md                          # written once by /sales-prospect-list
├── leads-processed.jsonl           # successful Attio writes (dedup index)
├── leads-skipped.jsonl             # ICP misses, no-identity, dedup hits
├── leads-failed.jsonl              # transient errors, retryable next run
└── runs/
    └── 2026-05-01-0930.md          # per-run human-readable summary
```

See `lead-gen-checkpoint/SKILL.md` for the file format contract.

## Design choices

- **Skills are prompts, not code.** Composition happens at runtime when Claude invokes skills inside a session. No build step.
- **File-based state.** Append-only JSONL — crash-safe, grep-able, no DB. Migrate to SQLite if you exceed ~50MB processed.jsonl.
- **One ICP per project.** ICP lives in `./icp.md` at the project root, not in this repo. Different projects = different ICPs.
- **Conservative anti-bot.** LinkedIn 100/day, Reddit 200/session, Instagram 40/day. Restricted accounts recover; banned accounts don't.
- **No DM automation.** This pipeline finds and writes leads. Outreach happens via your normal channels — never via scraped-platform DMs.
- **Vendored, not submoduled.** Upstream `sales-skills/sales` skills are copied into this repo and pinned in `skills-lock.json`. Edit them freely; check upstream periodically for breaking changes.

## Anti-patterns

- Running multiple Chrome drivers in one session (LinkedIn + Reddit together) — context drifts, selectors confuse, anti-bot triggers compound. The orchestrator enforces this; don't override.
- Storing API keys in any file in this repo. Use environment variables.
- Modifying existing JSONL lines. Append-only or nothing.
- Putting `./icp.md` or any state file inside this repo. They're per-project user data.
- Skipping `./icp.md` and "just running" the orchestrator. It will refuse — that's intentional.

## Updating vendored skills

When upstream `sales-skills/sales` ships changes you want:

```bash
# Re-pull a specific skill from upstream
cd /tmp && git clone --depth 1 https://github.com/sales-skills/sales sales-upstream
cp -r sales-upstream/skills/sales-attio /path/to/this-repo/skills/

# Update commit + hash in skills-lock.json
```

Then diff against your previous version — vendored skills sometimes drift in ways that break field mappings or change the question flow.

## License

MIT.
