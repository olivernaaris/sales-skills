---
name: lead-gen-reddit-chrome
description: "Drives Claude for Chrome through Reddit (subreddits, search, threads, user profiles) to extract leads showing buying-intent signals — pain points, tool requests, hiring posts, founder activity. Bridges Reddit usernames to real identities via profile-linked personal sites / Twitter / GitHub before handing off to /lead-gen-orchestrator for enrichment. Use when the ICP includes founders, indie hackers, or community-driven personas where Reddit is where they actually post. Do NOT use for cold-DM outreach (against Reddit ToS), for Reddit Ads (use /sales-reddit-ads), or as a standalone scraper without the orchestrator."
argument-hint: "[subreddit URL, search URL, or thread URL, target count]"
license: MIT
version: 0.1.0
tags: [sales, prospecting, reddit, chrome, computer-use, intent]
---
# Reddit Chrome Driver

Drive Claude for Chrome through Reddit to find leads exhibiting buying-intent signals, then resolve their Reddit username to a real identity (via personal site / Twitter / GitHub link in their profile) before handing off to `/lead-gen-orchestrator`.

This skill is invoked by the orchestrator — not standalone except for testing.

## ToS reality check (read first)

Reddit's User Agreement allows browsing public content but prohibits unauthorized scraping at scale and **prohibits unsolicited DMs to users found via scraping**. This skill is for **lead identification only**:

- All targets must come from public posts/comments
- Output is read-only signals + identity bridge — never DM the user from this skill
- Outreach to identified leads must go through proper channels (their stated business email, LinkedIn, etc.) — not Reddit DMs
- Hard cap **200 profiles per session** — Reddit throttles aggressive paging
- Prefer `old.reddit.com` over `www.reddit.com` — older UI is faster, more readable, less anti-bot

## Step 0 — Pre-flight

Verify in the active Chrome tab:

1. Reddit loads (logged in or out — both work; logged-in tolerates more pages)
2. The target URL is one of: subreddit `/r/X`, search results, single thread, or user profile
3. No "Reddit is over capacity" or rate-limit page
4. Prefer `old.reddit.com` — if user pasted `www.reddit.com/r/X`, switch to `old.reddit.com/r/X`

## Step 1 — Gather context (only if missing)

1. **Source URL?** Required. Examples:
   - Subreddit: `old.reddit.com/r/SaaS/top/?t=week`
   - Search: `old.reddit.com/search?q="looking+for+a+tool+to"&restrict_sr=on&sr=SaaS`
   - Thread: `old.reddit.com/r/SaaS/comments/.../...`

2. **Signal type to hunt for?**
   - A) Pain point — "I'm struggling with X", "Why is X so hard"
   - B) Tool request — "Looking for a tool to do Y", "What do you use for Z"
   - C) Founder/hiring — "We just raised", "Hiring our first SDR"
   - D) Competitor mentions — users discussing a specific competitor
   - E) Custom — user describes the pattern

3. **Minimum karma threshold?** Default 100. Filters bot accounts and throwaways. Lower for niche/young subreddits.

4. **Account age threshold?** Default 90 days. Filters brand-new accounts (often spam).

5. **Target count?** Default 25. Hard cap 200.

## Step 2 — Find signal-matching posts/comments

```
1. Open the source URL (subreddit / search / thread)
2. For each post visible on the page:
   a. Read post title + body OR top-level comments (depending on source type)
   b. Match against signal pattern from Step 1
   c. If match → capture {post_url, post_title, signal_quote, username, post_score, post_age}
3. For comment-driven sources (threads): scan all top-level + reply comments
4. Wait 5–10s between page loads (randomized) — Reddit is more lenient than LinkedIn
5. Paginate via "next" link at bottom — never modify ?after= URL params manually
```

Skip:
- Posts/comments with score < 1 (downvoted, often spam or troll)
- Authors with `[deleted]` or `[removed]` username
- Bot accounts (look for `Bot` in name, automod patterns, identical comment templates)

## Step 3 — Bridge username → real identity

This is the critical step that makes Reddit leads usable. Reddit usernames alone are not actionable for B2B outreach.

For each matched username, visit `old.reddit.com/u/<username>/about.json` (yes, JSON — fast, low overhead) and the profile page:

```
1. Open old.reddit.com/u/<username>
2. Check profile sidebar / "about" section for:
   - Personal website (most valuable — domain → company)
   - Twitter/X handle
   - GitHub handle
   - Any URL in their bio
3. If profile is empty: scan their last 20 comments for:
   - Self-disclosed company name ("I'm the founder of X", "I work at Y")
   - Self-disclosed name ("Hi, I'm Sarah from...")
   - Linked personal posts ("Wrote about this on my blog: example.com")
4. Capture {personal_url, twitter, github, self_disclosed_company, self_disclosed_name}
5. If NONE found → this lead is dead-end on identity. Append to ./leads-skipped.jsonl with reason "no_identity_bridge" and continue.
```

Karma + age check before bridging — drops 30–50% of matches but raises lead quality dramatically.

## Step 4 — Output to orchestrator

For each lead with a usable identity bridge, output one JSON line:

```json
{
  "source": "reddit",
  "username": "...",
  "name": "...",                   // from self-disclosure if found, else null
  "company": "...",                // from self-disclosure if found, else null
  "company_domain": "...",         // resolved from personal_url, else null
  "personal_url": "...",
  "twitter": "...",
  "github": "...",
  "profile_url": "old.reddit.com/u/...",
  "signal_type": "pain_point" | "tool_request" | "founder" | "competitor" | "custom",
  "signal_quote": "exact text excerpt that matched",
  "signal_post_url": "permalink to the post/comment",
  "subreddit": "r/...",
  "karma": 1234,
  "account_age_days": 567,
  "extracted_at": "ISO8601"
}
```

Hand off to `/lead-gen-orchestrator`. The orchestrator will:
- Use `personal_url` domain (or `self_disclosed_company`) for `/sales-hunter` Email-Finder
- Fall back to `/sales-apollo` people_match by name + domain
- Dedup against Attio
- Upsert with `signal_quote` populated as the conversation starter

If `company_domain` is null and Hunter/Apollo can't resolve, the orchestrator will tag the record as `email_pending` and the user can review manually.

## Step 5 — Anti-bot stop conditions

Stop and report if any of these appear:

| Signal | Action |
|---|---|
| "Reddit is over capacity" page | Stop, retry in 30 min |
| CAPTCHA / "Are you a robot" page | Stop, do NOT solve |
| HTTP 429 on any request | Stop, wait 1h minimum |
| Profile pages return 404 for valid users | Account-level shadowban — switch IPs / wait 24h |
| Search returns 0 results when it should return many | Throttle — slow down |

Never solve CAPTCHAs. Never speed-paginate. Never use multiple accounts in one session.

## Step 6 — Output JSON to orchestrator

After each successful lead OR a stop event:

```json
{
  "status": "ok" | "stop",
  "stop_reason": null | "captcha" | "rate_limit" | "no_identity" | "no_more_results",
  "lead": { ... lead JSON ... } | null,
  "page_url": "current source URL",
  "next_action": "continue" | "halt_30m" | "halt_24h"
}
```

## Field mapping reference (for `/sales-attio`)

| Output field | Attio attribute |
|---|---|
| `name` (if known) | `name` |
| `company` (if known) | `company` (relationship) |
| `company_domain` | `company.domains[0]` |
| `personal_url` | `personal_website` (custom attr) |
| `twitter` | `twitter_handle` (custom attr) |
| `github` | `github_handle` (custom attr) |
| `profile_url` | `reddit_url` (custom attr) |
| `signal_quote` | `intent_signal_quote` (custom long-text) |
| `signal_post_url` | `intent_signal_url` (custom URL) |
| `signal_type` | `intent_signal_type` (custom select) |
| `source: "reddit"` | `lead_source` |

## What this skill does NOT do

- **Reddit DMs** — against ToS for unsolicited outreach
- **Comment on the lead's post** — never engage from this skill
- **Subreddit moderation actions** — out of scope
- **Reddit Ads** — use `/sales-reddit-ads`
- **API-based scraping** — this is the Chrome driver; for API-based, write a separate `lead-gen-reddit-api` skill (Reddit's PRAW + free tier handles 100 req/min, often a better path than Chrome for Reddit specifically)

## Why a Chrome driver for Reddit at all?

Honest tradeoff: Reddit's official API is faster and more reliable than Chrome scraping for Reddit specifically. Use this Chrome skill when:

- The user's Claude for Chrome session is already authenticated and the orchestrator is driving a multi-source run (LinkedIn + Reddit) — keeping all sources in Chrome simplifies the loop
- The user doesn't want to deal with Reddit OAuth / API key setup
- They need to see signals visually that the API doesn't surface cleanly (vote patterns, awards, sidebar widgets)

Otherwise, prefer an API-based skill.

## Anti-patterns (do NOT do these)

- Scraping `www.reddit.com` instead of `old.reddit.com` — slower, more anti-bot, JS-heavy
- Skipping the karma + age check — pollutes Attio with bot accounts
- Skipping the identity bridge — Reddit username alone is not a usable lead
- Capturing usernames without the `signal_quote` — loses the "why" of the lead, kills outreach quality
- DMing leads via Reddit — ToS violation, kills your account, and converts worse than email
- Running Reddit + LinkedIn + Instagram in one Chrome session — context drift, selector confusion (the orchestrator already enforces this, don't override)
