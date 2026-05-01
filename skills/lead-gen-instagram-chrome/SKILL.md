---
name: lead-gen-instagram-chrome
description: "Drives Claude for Chrome through Instagram (hashtag pages, location tags, profile pages, follower lists) to find creators, influencers, DTC founders, and niche-community accounts. Bridges IG handle to a real identity via the bio link (linktree, personal site, or email-in-bio) before handing off to /lead-gen-orchestrator. Use when ICP is creator-economy, DTC, lifestyle, or local-business heavy. Do NOT use for B2B SaaS leads (LinkedIn is far better), for cold-DM outreach (against IG ToS, kills account), or as a standalone scraper without the orchestrator."
argument-hint: "[hashtag URL, profile URL, or follower-list URL, target count]"
license: MIT
version: 0.1.0
tags: [sales, prospecting, instagram, chrome, computer-use, creators]
---
# Instagram Chrome Driver

Drive Claude for Chrome through Instagram to find leads matching a creator/DTC/local-business ICP, then resolve the IG handle to a real identity via the bio link before handing off to `/lead-gen-orchestrator`.

This skill is invoked by the orchestrator — not standalone except for testing.

## Use this only if your ICP needs it

Instagram is the **most fragile** of the three Chrome drivers (LinkedIn, Reddit, Instagram). If your ICP is B2B SaaS, technical buyers, or anything LinkedIn or Apollo can find — skip this skill. Use it only when:

- You're targeting creators, influencers, DTC founders, or e-commerce brands
- The ICP requires Instagram presence as a signal (e.g. "DTC brands with 10k+ followers")
- Your leads can't be found on LinkedIn (consumer/lifestyle/local businesses)

For B2B leads, this skill's ROI is negative.

## ToS reality check (read first)

Instagram's Terms prohibit automated scraping and unauthorized data collection. Anti-bot is the most aggressive of the three platforms — action blocks trigger within ~50 profile views per session. This skill is intentionally **the most conservative**:

- Hard cap **40 profiles per account per day** — going higher gets accounts shadow-banned within hours
- Use the user's own logged-in session (logged-out browsing hits a forced-login modal after ~5 profiles)
- No follower-list pagination beyond ~200 entries (IG cuts you off — don't fight it)
- Never DM scraped users from this skill — kills the account and converts worse than email anyway

If a user wants higher volume, they should use a paid IG-specific tool (Modash, HypeAuditor, Heepsy — all in the upstream `sales-skills/sales` repo) instead of Chrome scraping.

## Step 0 — Pre-flight

Verify in the active Chrome tab:

1. User is logged into Instagram (look for nav avatar)
2. No "We restrict certain activity" or "Try again later" banner
3. No forced-login modal mid-session
4. The target URL is one of:
   - Hashtag: `instagram.com/explore/tags/<tag>/`
   - Profile: `instagram.com/<handle>/`
   - Location: `instagram.com/explore/locations/<id>/<name>/`
   - Follower list: `instagram.com/<handle>/followers/` (only viewable while logged in)

If pre-flight fails → stop, report to user, do NOT proceed.

## Step 1 — Gather context (only if missing)

1. **Source URL?** Required.

2. **Source type?**
   - A) Hashtag — finds posts tagged with a niche keyword, then scrapes posters
   - B) Competitor follower list — finds users who follow a brand similar to the user's customer
   - C) Location tag — finds local businesses / creators in a city
   - D) Single profile — extract one specific profile (testing / one-off)
   - E) Custom URL — IG search results, etc.

3. **Target count?** Default 15. Hard cap 40.

4. **Minimum follower count?** Default 1,000. Filters inactive / personal accounts.

5. **Maximum follower count?** Default 500,000. Above this is "celebrity" — mass DMs to them never work, skip.

6. **Account type filter?**
   - A) Business / Creator accounts only (has contact button)
   - B) Personal accounts only
   - C) Either

## Step 2 — Per-profile extraction loop

For each candidate handle from the source:

```
1. Open instagram.com/<handle>/ in same tab (never new tabs — looks bot-like on IG)
2. Wait 6–12s for full render (IG lazy-loads bio + posts)
3. Check filters:
   - follower_count in [min, max] range → else skip
   - account_type matches filter → else skip
   - has_bio → if empty, skip (no identity bridge possible)
4. Extract from profile header:
   - handle              → @<handle>
   - display_name        → name shown above bio (often real name for creators)
   - bio_text            → full bio block
   - bio_link            → linktree / personal site / direct URL in bio (THE identity bridge)
   - email_in_bio        → if email regex matches in bio_text (rare but happens — instant identity)
   - verified            → blue checkmark present?
   - account_type        → "personal" | "business" | "creator"
   - category            → IG category label if business/creator (e.g. "Beauty Cosmetic Store")
   - follower_count, following_count, post_count
   - external_email      → if "Email" contact button exists (business/creator only) — click reveals email
5. Detect signals (see below)
6. Hand control to orchestrator with output JSON
7. Wait 90–180s (randomized) before next profile — STRICT, IG anti-bot is aggressive
```

### Signals to detect (feed `/sales-intent`)

- **Recent product launch** — bio mentions "🎉 New launch" / "Out now" → `signal: "recent_launch"`
- **Hiring** — bio mentions "Hiring" / "Now hiring" → `signal: "hiring"`
- **Brand collaboration interest** — bio mentions "DM for collabs" / "Open to brand deals" → `signal: "open_to_collabs"`
- **Press / media** — "As seen in [Forbes/etc]" → `signal: "press_<outlet>"`
- **Funding / scale** — "VC-backed" / "Y Combinator" → `signal: "funded"`
- **Location** — emoji flags or city in bio → `signal: "location_<city>"`
- **High-engagement** — recent post likes ÷ followers > 5% → `signal: "high_engagement"`

## Step 3 — Bridge IG handle → real identity

This is the critical step. IG handles alone are not actionable for outreach.

```
1. If email_in_bio found → identity is solved, skip rest of bridging.
2. If bio_link is a linktree / link.bio / beacons.ai / similar:
   a. Open the linktree page in same tab
   b. Wait 5–10s
   c. Extract all listed URLs
   d. Pick most lead-worthy:
      - Personal/business website (top priority — domain → company)
      - Direct email link (mailto:)
      - LinkedIn URL (gold — feeds /sales-apollo cleanly)
      - Twitter/X
      - Substack / newsletter
   e. Capture {linktree_url, resolved_personal_url, resolved_email, resolved_linkedin}
3. If bio_link is a direct URL (no linktree):
   a. Capture as personal_url, resolve domain
4. If business/creator account has "Email" contact button:
   a. Click to reveal — captures business email directly
   b. This is the highest-quality identity signal
5. If NONE of the above → identity is dead-end. Append to ./leads-skipped.jsonl with reason "no_identity_bridge" and continue.
```

Linktree resolution typically converts ~70% of bio_links into a usable identity. Direct URLs convert ~95%. Empty bios convert 0% (skip them at Step 2.3).

## Step 4 — Output to orchestrator

For each lead with usable identity, output one JSON line:

```json
{
  "source": "instagram",
  "handle": "...",
  "name": "...",                    // display_name from profile
  "company": "...",                 // resolved from personal_url or category, else null
  "company_domain": "...",          // resolved from personal_url, else null
  "email": "...",                   // from email_in_bio, contact button, or linktree mailto
  "personal_url": "...",
  "linkedin_url": "...",            // from linktree if present
  "twitter": "...",                 // from linktree if present
  "linktree_url": "...",
  "profile_url": "instagram.com/...",
  "bio_text": "full bio",
  "category": "...",                // IG category label
  "account_type": "business" | "creator" | "personal",
  "verified": true | false,
  "follower_count": 12345,
  "following_count": 567,
  "post_count": 89,
  "signals": ["recent_launch", "open_to_collabs", "..."],
  "extracted_at": "ISO8601"
}
```

The orchestrator will:
- If `email` already populated (from email_in_bio / contact button) → skip Hunter, go straight to dedup + write
- Otherwise use `company_domain` for `/sales-hunter` Email-Finder
- Fall back to `/sales-apollo` people_match by name + domain
- Dedup against Attio
- Upsert with `bio_text` + signals as conversation context

## Step 5 — Anti-bot stop conditions

Stop **immediately** and report if any of these appear:

| Signal | Meaning | Action |
|---|---|---|
| "We restrict certain activity" banner | Action block started | Stop, no IG for 24h minimum |
| "Try again later" page | Rate-limited | Stop, retry in 4h |
| Forced-login modal mid-session | Session degraded | Stop, log out and back in tomorrow |
| CAPTCHA / "confirm you're human" | Anti-bot triggered | Stop, do NOT solve |
| "Suspicious login attempt" email to user | Account flagged | Stop, halt 72h |
| Profile pages won't load (>30s) | Throttled | Stop, halt 4h |
| Follower-list cuts off after <50 entries | Account-level throttle | Stop, switch source |
| Posts grid shows blank squares | Lazy-load failure / throttle | Stop, halt 1h |

**Never:**
- Solve CAPTCHAs
- Refresh through "restrict activity" banners — escalates to permanent shadow-ban
- Log in/out repeatedly to "reset" — fastest path to ban
- Run this skill after a stop event the same day
- Use the IG mobile site as a workaround — same anti-bot, worse selectors

## Step 6 — Output JSON to orchestrator

After each successful lead OR stop event:

```json
{
  "status": "ok" | "stop",
  "stop_reason": null | "action_block" | "rate_limit" | "captcha" | "login_modal" | "no_more_results",
  "lead": { ... lead JSON ... } | null,
  "page_url": "current source URL",
  "next_action": "continue" | "halt_4h" | "halt_24h" | "halt_72h"
}
```

The orchestrator uses `next_action` to skip IG for the rest of the run / day / week.

## Field mapping reference (for `/sales-attio`)

| Output field | Attio attribute |
|---|---|
| `name` | `name` |
| `company` (if known) | `company` (relationship) |
| `company_domain` | `company.domains[0]` |
| `email` | `email_addresses[0]` |
| `personal_url` | `personal_website` (custom attr) |
| `linkedin_url` | `linkedin_url` (custom attr) |
| `linktree_url` | `linktree_url` (custom attr) |
| `profile_url` | `instagram_url` (custom attr) |
| `bio_text` | `bio` (custom long-text) |
| `category` | `instagram_category` (custom select) |
| `follower_count` | `instagram_followers` (custom number) |
| `verified` | `instagram_verified` (custom checkbox) |
| `signals[]` | `intent_signals` (custom multi-select) |
| `source: "instagram"` | `lead_source` |

## What this skill does NOT do

- **Instagram DMs** — against ToS for unsolicited outreach, kills the account, converts worse than email
- **Story scraping** — unreliable, anti-bot-heavy, low ROI
- **Reel transcript extraction** — out of scope, IG actively blocks
- **Engagement-rate calculation across many posts** — too slow, hits rate limits; use Modash / HypeAuditor for this
- **Follower count of follower lists ("followers of followers")** — recursive, instant ban
- **Multi-account rotation** — single account, single session, single browser
- **Login automation** — user logs in manually before this skill runs

## Why a Chrome driver for Instagram at all?

Honest tradeoff: dedicated IG influencer-discovery tools (Modash, HypeAuditor, Heepsy, GRIN, Aspire — all skills in the upstream repo) are *significantly* better than this Chrome driver for any volume of IG prospecting:

- They have full follower databases (no scraping needed)
- They calculate engagement rates correctly
- They detect fake followers
- They don't risk your IG account

Use this Chrome skill when:
- The user is doing low-volume manual sourcing (< 40 leads/day)
- They don't want to pay for a dedicated tool
- They're already using Chrome for LinkedIn / Reddit and want one consistent loop

For anything above ~200 IG leads/month, switch to **`/sales-modash`** or **`/sales-hypeauditor`** in the upstream repo.

## Anti-patterns (do NOT do these)

- Scraping >40 profiles/day per IG account
- Running this skill while logged out — forced-login modal hits within 5 profiles
- Skipping the bio_link / linktree resolution — IG handle alone is not a usable lead
- Skipping the follower-count filter — wastes Hunter quota on dead accounts
- DMing scraped users from IG — instant ToS violation, account-killer
- Solving "confirm you're human" challenges programmatically — escalation to permanent ban
- Running IG + LinkedIn + Reddit in one Chrome session (orchestrator already enforces this — don't override)
- Scraping celebrity / >500k follower accounts — they ignore cold DMs and email
- Running at 3am — IG flags off-hours scraping faster; pattern this 9am–9pm in user's timezone
