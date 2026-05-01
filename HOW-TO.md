# How to Run These Skills

Step-by-step runbook for using `sales-skills` with **Claude Code** and **Claude Cowork**. For the project overview, see [README.md](./README.md).

## Contents

- [Setup once: push the repo to GitHub](#setup-once-push-the-repo-to-github)
- [Running with Claude Code](#running-with-claude-code)
- [Running with Claude Cowork](#running-with-claude-cowork)
- [Caveats: Code vs Cowork](#caveats-code-vs-cowork)
- [Recommended split: use both](#recommended-split-use-both-for-what-each-is-good-at)
- [First-run checklist](#first-run-checklist)

---

## Setup once: push the repo to GitHub

Skills can't be installed from a local-only repo. First push it:

```bash
cd /Users/olivernaaris/git-projects/sales-skills
git add . && git commit -m "feat: initial release"
gh repo create olivernaaris/sales-skills --public --source=. --push
```

Tag a release for stable consumption:

```bash
git tag v0.1.0 && git push --tags
```

---

## Running with Claude Code

### 1. Install the skills into Claude Code

In the project directory where you want to run lead-gen:

```bash
cd /path/to/my-leadgen-project

# Install only what you need (recommended — installing all 14 bloats context):
npx skills add olivernaaris/sales-skills --skill lead-gen-orchestrator
npx skills add olivernaaris/sales-skills --skill lead-gen-linkedin-chrome
npx skills add olivernaaris/sales-skills --skill lead-gen-reddit-chrome
npx skills add olivernaaris/sales-skills --skill lead-gen-instagram-chrome
npx skills add olivernaaris/sales-skills --skill lead-gen-checkpoint
npx skills add olivernaaris/sales-skills --skill sales-prospect-list
npx skills add olivernaaris/sales-skills --skill sales-account-map
npx skills add olivernaaris/sales-skills --skill sales-intent
npx skills add olivernaaris/sales-skills --skill sales-enrich
npx skills add olivernaaris/sales-skills --skill sales-hunter
npx skills add olivernaaris/sales-skills --skill sales-apollo
npx skills add olivernaaris/sales-skills --skill sales-attio
npx skills add olivernaaris/sales-skills --skill sales-data-hygiene
```

Or all at once (if you want the `/sales-do` router too):

```bash
npx skills add olivernaaris/sales-skills --skill '*' -a claude-code -y
```

This drops the skills into `.claude/skills/` so Claude Code sees them as `/lead-gen-orchestrator`, `/sales-prospect-list`, etc.

### 2. Wire up your MCPs

The skills assume Hunter, Apollo, and Attio MCPs are connected. Edit `~/.claude.json` (or your project's `.mcp.json`):

```json
{
  "mcpServers": {
    "hunter": { "url": "https://hunter-remote-mcp.example/sse", "transport": "sse" },
    "apollo": { "url": "https://mcp.apollo.io/sse", "transport": "sse" },
    "attio":  { "url": "https://mcp.attio.com/sse",  "transport": "sse" }
  }
}
```

Verify in Claude Code with `/mcp` — should list all three connected.

### 3. One-time: define the ICP

```bash
cd /path/to/my-leadgen-project
claude
```

Inside Claude Code:

```
/sales-prospect-list
```

Answer the questions. It writes `./icp.md` to your project root.

Or skip the interview and edit by hand:

```bash
cp ~/.claude/skills/templates/icp.md ./icp.md
```

(Adjust the path to wherever `npx skills add` placed the templates folder.)

### 4. Run a lead-gen session

```
/lead-gen-orchestrator https://www.linkedin.com/sales/search/people?... target=25
```

What happens:

1. Reads `./icp.md`
2. Calls `/lead-gen-linkedin-chrome` → drives Claude for Chrome through the search URL
3. Each lead → `/sales-account-map` → `/sales-intent` → `/sales-enrich` (Hunter then Apollo) → `/sales-data-hygiene` → `/sales-attio` (upsert)
4. Appends to `./leads-processed.jsonl`
5. Stops at target=25 or first anti-bot signal
6. Writes `./runs/2026-05-01-1430.md` summary

### 5. Schedule it for daily runs

```
/schedule daily 09:00 /lead-gen-orchestrator <linkedin-search-url> target=25
```

Cron-driven, runs unattended, dedup state survives across runs.

### Resume after a crash

```
/lead-gen-orchestrator <same-source-url> target=25
```

The orchestrator reads `./leads-processed.jsonl` → skips already-processed URLs → continues where it stopped.

---

## Running with Claude Cowork

Cowork does **not** support `npx skills add`. You bring skill content into Cowork manually — there's no `/lead-gen-orchestrator` slash command unless you paste it in.

### Recommended setup: one Cowork Project per ICP

#### 1. Create a Cowork Project

Name it after your ICP, e.g. "Lead Gen — SaaS Founders".

#### 2. Paste skills into Custom Instructions

From your local repo, concatenate the skills you need:

```bash
cat skills/lead-gen-orchestrator/SKILL.md \
    skills/lead-gen-checkpoint/SKILL.md \
    skills/lead-gen-linkedin-chrome/SKILL.md \
    skills/sales-account-map/SKILL.md \
    skills/sales-intent/SKILL.md \
    skills/sales-enrich/SKILL.md \
    skills/sales-hunter/SKILL.md \
    skills/sales-attio/SKILL.md \
    skills/sales-data-hygiene/SKILL.md
```

Copy the combined output into the Project's custom instructions field.

> **Limit:** ~6–8 skills max. Cowork has a custom-instructions size limit and skills past that bloat context. Pick the Chrome driver for the source you're running this session — don't paste all three.

#### 3. Upload `icp.md` as a Project file

That's how Cowork "reads" it.

#### 4. Connect Hunter + Apollo + Attio MCPs

In Cowork: Settings → Connectors. (You already have these.)

#### 5. Enable Claude for Chrome

In the Cowork Project. This is what makes Cowork distinct from Code — it drives your actual browser.

#### 6. Start a session

```
Run the lead-gen-orchestrator workflow on this LinkedIn Sales Nav search:
https://www.linkedin.com/sales/search/people?...
Target 25 leads. Use Claude for Chrome.
```

Cowork loads the pasted skills as system context, follows the orchestrator's loop, drives Chrome through LinkedIn, calls Hunter/Apollo/Attio MCPs.

#### 7. Manage state files manually

State files (`leads-processed.jsonl`, etc.) live in the Project's file storage:

- Upload before the session
- Download after the session to inspect
- Re-upload before the next session

This is the awkward part of Cowork vs Code. There's no automatic file persistence between Cowork sessions.

---

## Caveats: Code vs Cowork

|  | Claude Code | Claude Cowork |
|---|---|---|
| Install skills | `npx skills add` | Paste into Project custom instructions manually |
| Run skill | `/lead-gen-orchestrator <url>` | Type natural-language "run lead-gen-orchestrator..." |
| Persistent state | Local `./leads-*.jsonl` files | Upload/download to Cowork Project files |
| Schedule | `/schedule` | No native scheduler — start each session manually |
| Drive a real browser | Via Computer Use | **Native — Cowork's main strength** |
| Run for hours unattended | `/schedule` daily cron | One long session (degrades over time) |

---

## Recommended split: use both, for what each is good at

The original goal was "AI agent runs for hours, drives Chrome, finds leads on LinkedIn/Reddit/Instagram." Honest answer: neither tool does this in one shot. The pragmatic split:

### Cowork (Chrome-driven, ~30 min sessions)

- Run a lead-gen session interactively
- Watch Chrome do the LinkedIn / Reddit / IG browsing
- Cowork handles the human-loop parts (CAPTCHAs, restriction banners → you intervene)
- Output: appends leads to your Attio + downloads updated `leads-processed.jsonl`

### Claude Code (background, scheduled)

- `/schedule` runs Apollo-only or Reddit-API enrichment passes overnight (no browser needed)
- Generates run summaries, audit logs
- Maintains `./leads-*.jsonl` in your local repo / git
- Owns the durable state

### Same Attio = shared truth

Both write to the same Attio list and dedup against the same records. Wherever you run the orchestrator, the state converges in Attio.

---

## First-run checklist

Before your first real run:

- [ ] Repo pushed to GitHub
- [ ] `npx skills add` succeeds in your project directory (Code) OR skills pasted into Cowork Project (Cowork)
- [ ] Hunter / Apollo / Attio MCPs connected and verified (`/mcp` in Code, Settings → Connectors in Cowork)
- [ ] `./icp.md` exists and filled in
- [ ] Attio custom attributes from `icp.md` `attio_field_mapping` actually exist in your workspace (otherwise writes silently drop fields)
- [ ] Test list created in Attio (e.g. `lead-gen-test`) for the first few runs
- [ ] First run: `target=5`, source = Reddit (lowest anti-bot risk) to validate the whole chain end-to-end before pointing it at LinkedIn

### Why Reddit first

Reddit is the lowest-friction source — public data, lenient pacing, no account-ban risk. If 5 leads land in Attio with `email` and `signal_quote` populated, the loop works. *Then* try LinkedIn with `target=10`. *Then* schedule daily.

If you start with LinkedIn and burn through anti-bot tolerances debugging, you've cost yourself a 24–72h cooldown on a Sales Nav account before the first successful run.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/lead-gen-orchestrator` refuses to start | `./icp.md` missing | Run `/sales-prospect-list` or copy the template |
| Attio writes succeed but custom fields are blank | Custom attribute doesn't exist in workspace | Create the attribute in Attio, OR remove the line from `icp.md` `attio_field_mapping` |
| Hunter `Email-Finder` returns nothing | Confidence threshold too high, or company_domain wrong | Lower threshold in `icp.md`; verify domain extraction in Chrome driver output |
| LinkedIn driver stops with "restriction" stop_reason | Account flagged | Halt LinkedIn for 72h minimum. Don't retry. |
| Reddit driver returns 0 leads | Subreddit too small, signal phrase too narrow, or karma threshold too high | Loosen filters in `icp.md` per-source overrides |
| `leads-processed.jsonl` has duplicate URLs | Concurrent runs (forbidden) | Stop overlapping runs. Schedule cadence ≥ 1 hour apart |
| Run summary shows 100% ICP misses | ICP filters mismatch source URL | Either the source URL targets the wrong segment, or your ICP is too tight — adjust in `icp.md` |
