# ICP Definition

> **What this is:** the per-project ICP file consumed by `/lead-gen-orchestrator` and the Chrome drivers. Copy this template to your project root as `./icp.md`, fill in the bracketed fields, and the orchestrator will read it on every run.
>
> **How to generate this without filling it manually:** run `/sales-prospect-list` — it interviews you and writes a populated version of this file.
>
> **One ICP per project.** If you target multiple ICPs, run them as separate projects (separate working directories) so the dedup state and run history stay isolated.

---

## 1. Firmographic filters (company-level)

These define WHICH companies to target. The orchestrator rejects any candidate whose company doesn't match the **required** filters.

| Filter | Required values | Notes |
|---|---|---|
| **Industry** | [e.g. SaaS, Fintech, B2B Marketing] | Use Apollo / LinkedIn industry taxonomy where possible |
| **Company size (employees)** | [e.g. 50–500] | Hard range — outside this = reject |
| **Annual revenue** | [e.g. $5M–$100M] | Optional; leave blank if you don't filter on this |
| **Geography** | [e.g. US, UK, DACH] | Country codes or region names |
| **Funding stage** | [e.g. Seed, Series A, Series B, bootstrapped] | Optional |
| **Technology signals** | [e.g. uses Salesforce, runs Kubernetes] | Optional but high-value if available |

## 2. Demographic filters (person-level)

These define WHO at the target companies you want to reach.

| Filter | Required values |
|---|---|
| **Job titles** | [e.g. VP Engineering, Head of Growth, CTO, Founder] |
| **Seniority** | [e.g. Director+, VP+, C-suite] |
| **Department** | [e.g. Engineering, Sales, Marketing] |
| **Job function** | [e.g. people-management, IC, ops] |

## 3. Signal filters (intent boosts)

These don't reject leads — they boost priority. Used by `/sales-intent` to score buying intent.

- [ ] **Recent funding** — raised in last [90] days
- [ ] **Recent role change** — started current role in last [90] days
- [ ] **Hiring signals** — open roles for [SDR, RevOps, etc.]
- [ ] **Tool requests** — public posts asking "looking for a tool to do X"
- [ ] **Competitor mentions** — currently using or evaluating [competitor names]
- [ ] **Speaking / conferences** — recent talks at [event names]
- [ ] **Custom signal:** [describe your unique buying-intent pattern]

## 4. Negative filters (always exclude)

- [ ] Existing customers (sync from Attio at run start)
- [ ] Companies in active deals
- [ ] Do-not-contact list — see [./do-not-contact.txt] if you maintain one
- [ ] Competitors of [your company]
- [ ] [Other exclusions specific to your business]

## 5. Source priority

Which Chrome driver should the orchestrator try FIRST for this ICP? List in order. The orchestrator falls through if a source returns < 5 leads matching the ICP.

1. [e.g. linkedin] — Sales Nav search URL: `[https://...]`
2. [e.g. reddit] — subreddit / search URL: `[https://...]`
3. [e.g. apollo] — direct Apollo search (no Chrome): query `[describe]`
4. [e.g. instagram] — only if creator/DTC ICP

If you only have one viable source, leave the others blank — the orchestrator skips empty entries.

## 6. Per-source extraction overrides

Optional. If a Chrome driver should look for ICP-specific fields beyond the defaults, list them here.

```yaml
linkedin:
  extra_fields: []           # e.g. ["years_at_company", "team_size_managed"]
  extra_signals: []          # e.g. ["mentions_AI", "mentions_compliance"]
reddit:
  subreddits_to_prefer: []   # e.g. ["r/SaaS", "r/startups"]
  signal_phrases: []         # e.g. ["looking for a tool", "any recommendations for"]
instagram:
  account_categories: []     # e.g. ["Beauty Cosmetic Store", "Health/Beauty"]
  min_followers: 1000
  max_followers: 500000
```

## 7. Enrichment configuration

Tunes how `/sales-enrich` chains providers.

| Setting | Value |
|---|---|
| **Hunter confidence threshold** | [70] — below this, fall through to Apollo |
| **Apollo fallback enabled** | [yes / no] |
| **Allow `email_pending` leads** | [yes / no] — write to Attio without email if both providers miss |
| **Phone enrichment** | [yes / no] — slower, lower hit rate |

## 8. Attio field mapping

Maps the JSON output of the Chrome drivers to your Attio attributes. **Required** — without this, `/sales-attio` doesn't know where to write.

```yaml
target_list: "[your Attio list slug, e.g. inbound-2026-q2]"

attio_field_mapping:
  # Standard people fields
  name:              name
  email:             email_addresses
  job_title:         job_title
  location:          location

  # Company relationship
  company:           company
  company_domain:    company.domains

  # Custom attributes (must exist in your Attio workspace)
  linkedin_url:      linkedin_url
  twitter:           twitter_handle
  github:            github_handle
  personal_url:      personal_website
  reddit_url:        reddit_url
  instagram_url:     instagram_url
  bio_text:          bio
  signals:           intent_signals
  signal_quote:      intent_signal_quote
  signal_post_url:   intent_signal_url
  source:            lead_source
  run_id:            inbound_run_id
  follower_count:    instagram_followers
  verified:          instagram_verified
```

If a custom attribute doesn't exist in your Attio workspace, either:
1. Create it in Attio first, OR
2. Comment out the line — the field will be dropped on write, not silently misrouted

## 9. Daily run budget

Hard caps that the orchestrator will respect on every run.

| Source | Max profiles per day | Max profiles per run |
|---|---|---|
| linkedin | 100 | 25 |
| reddit | 200 | 50 |
| instagram | 40 | 15 |

Lower these if you're seeing throttle warnings. Don't raise above the defaults — they're tuned to anti-bot tolerances.

## 10. Notes

Free-form notes about this ICP — context, edge cases, things you've learned, signals that worked / didn't.

- [e.g. "ICP_misses on LinkedIn often turn out to be agencies — added 'consulting' to negative filters"]
- [e.g. "Hunter confidence < 80 produces too many bounces for this ICP — raised threshold to 80"]
- [e.g. "Reddit signal 'looking for a tool to' converts at 3x other phrases"]

---

**Last updated:** [YYYY-MM-DD]
**Owner:** [your email or name]
