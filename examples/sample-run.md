# Sample run

A real-shaped run of the pipeline (sanitized — fictional event, fictional people) from a recent enrichment pass.

## Setup

- Event: **AI in CPG 2026** (fictional industry conference)
- Date: 2026-05-22
- Slug: `ai-in-cpg-2026`
- Source: Luma export → CSV → 247 rows
- All connectors healthy

## Step 1 — load + normalize

- Raw CSV rows: 247
- After whitespace trim + email lowercase: 247 (no exact duplicates, but...)
- After (first+last+normalized-company) dedupe: 241 (caught 6 dupes — same person registered twice with different company strings: "Acme, Inc." vs "Acme Inc")
- Net into Step 2: **241 attendees**

Common dirt patterns spotted:
- 14 rows with trailing whitespace in `name`
- 9 rows with mixed-case emails
- 22 rows with company suffix variants (Inc / Inc. / Inc, / , Inc / LLC)
- 38 rows with free emails (gmail.com / outlook.com / icloud.com) and a self-typed company

## Step 2 — live LinkedIn resolution

Resolution path distribution:

| Path | Description | Count | Resolved |
|---|---|---|---|
| A | LinkedIn URL provided → `linkedin_get_profile` | 87 | 86 |
| B | Email only → `inbox_enrich_sender` → `linkedin_get_profile` | 116 | 92 |
| C | Name + company only → `linkedin_search_profiles` | 38 | 20 |
| — | Unresolved (no confident match) | 49 | 0 |

**Net resolved: 198 / 241 (82%)**

49 unresolved — mostly free-email registrants with very common names ("Sarah Johnson @ healthy snacks co") where the company string didn't match any of the candidate profiles confidently. Saved to `unresolved.csv` for manual pass.

**Rate-limit pacing:** Hit the ~40/hour cap after the first batch. Paced the rest across 4 batches over 4 hours. Total Step 2 wall time: 4.5 hours.

## Step 3 + 4 — company intelligence + ICP scoring

For each of the 198 resolved, pulled `agents_company_intelligence` on the LIVE confirmed company (not the registration form company) and scored via `agents_icp_score`.

**Title drift detected:** 41 of 198 resolved attendees had a LIVE title different from their registration title. Examples:
- Registered as "Director of Marketing @ Snackable" → live: "VP Marketing @ Snackable" (promoted 18 days before event)
- Registered as "Brand Manager @ Goji Goods" → live: "Head of Brand @ Goji Goods" (promoted 6 weeks before event)
- Registered as "Founder @ Crunch Co" → live: "Co-Founder & CEO @ Crunch Co" (no change, registration was short-hand)

41 / 198 = 21% title drift. This is the receipt for the workflow — those 41 emails would have gone out with the wrong title under Apollo CSV enrichment.

## Step 4 — tier breakdown

| Tier | Score range | Count | Action |
|---|---|---|---|
| **TIER_1** — ICP perfect | ≥ 9 | 31 | Enroll + AE Slack ping |
| **TIER_2** — ICP fit | 7-8 | 64 | Enroll in event drip |
| **TIER_3** — ICP loose | 5-6 | 103 | Nurture list, no email |
| **DQ** | < 5 or hard DQ | 49 | Archive |
| **Total resolved** | | 247 | |

(31 + 64 + 103 + 49 = 247 because the 49 unresolved are counted as DQ for tier purposes — they didn't make it past Step 2 confidently enough to be tiered. The breakdown over the *198 confirmed-resolved*: 31 / 64 / 103 / 0 with the original 49 unresolveds tracked separately.)

## Step 5 — sample TIER_1 hooks

```
Maya Chen, Snackable.ai
hook: Saw you registered for ai in cpg - hope the snackable.ai session was useful!
companyRef: Snackable
industryRef: food brands

(LIVE title was "VP Marketing" — registration form said "Director of Marketing".
That's 21 days of title drift Apollo CSV enrich would have missed.)

Devon Brooks, Goji Goods
hook: Caught your name on the ai in cpg attendee list - did the keynote hold up?
companyRef: Goji Goods
industryRef: wellness brands

(LIVE title was "Head of Brand" — registration form said "Brand Manager".
Promoted ~6 weeks before the event.)

Priya Nair, Crunch Co
hook: Saw you registered for ai in cpg - was the founders track worth the trip?
companyRef: Crunch Co
industryRef: food brands

(Highest signal of the batch — founder of a Series A snacks brand, fits ICP
perfectly, no current vendor in our category. AE flagged for same-day LinkedIn note.)
```

All 31 TIER_1 hooks followed the rules: event name in first 7 words, lowercase after first letter, no LinkedIn references, brief reaction.

## Step 6-8 — enrollment + audit + CRM

- **Instantly enrollments:** 95 (31 TIER_1 + 64 TIER_2) into `AI in CPG 2026 — Event Follow-up`
- **Zevari audit list:** `event-ai-in-cpg-2026-2026-05-22` (198 resolved attendees — all tiers except DQ)
- **Zevari nurture sub-list:** `event-ai-in-cpg-2026-nurture-2026-05-22` (103 TIER_3)
- **DQ archived:** 49 via `targets_archive`
- **Pipedrive:** 95 contacts created or merged (TIER_1 + TIER_2), all tagged `event-ai-in-cpg-2026`. CRM titles updated to LIVE Zevari titles. Note added to each: `"Attended AI in CPG 2026 on 2026-05-22. Live-confirmed title: [live_title] @ [live_company]"`.

## Step 9 — TIER_1 AE Slack pings

31 TIER_1 attendees got individual pings to `#event-ai-cpg`. Sample format:

```
🎯 TIER_1 from AI in CPG 2026

Priya Nair — Founder & CEO @ Crunch Co
LinkedIn: linkedin.com/in/priya-nair-[id]
ICP score: 10/10
Why TIER_1: Series A snacks brand, 12-50 employees (core ICP range),
no current vendor in our category, founder buyer, recent fundraise (4 wks).
Hook used: Saw you registered for ai in cpg - was the founders track worth the trip?

→ Recommend manual LinkedIn note within 48h on top of the Instantly sequence.
```

## Step 10 — run summary

```
🎫 Event Enrichment Complete — AI in CPG 2026

📅 Event date: 2026-05-22
📥 Attendees loaded: 247
🧹 After normalization/dedupe: 241
🔍 Resolved live on LinkedIn: 198
❓ Unresolved: 49 (saved to unresolved.csv for manual pass)

Tier breakdown:
🎯 TIER_1 (perfect ICP, AE pinged): 31
✅ TIER_2 (drip enrolled): 64
📥 TIER_3 (nurture list): 103
🚫 DQ: 49

📧 Instantly enrollments: 95
📇 Pipedrive contacts created/updated: 95
🏷  Zevari audit list: event-ai-in-cpg-2026-2026-05-22

Sample TIER_1 hooks:
• Maya Chen, Snackable — Saw you registered for ai in cpg - hope the snackable.ai session was useful!
• Devon Brooks, Goji Goods — Caught your name on the ai in cpg attendee list - did the keynote hold up?
• Priya Nair, Crunch Co — Saw you registered for ai in cpg - was the founders track worth the trip?
```

## Step 11 — reflection notes

- **Unresolved rate 20% — acceptable but worth tightening.** Most unresolved are free-email registrants with common names. Suggest marketing add a "LinkedIn URL" question to the registration form for next year's event — would cut unresolved by 50%+.
- **Title drift 21%** — this is the headline metric. 41 emails out of 198 would have shipped with a wrong title under Apollo CSV enrichment. Show the number to whoever questions the workflow.
- **TIER_1 surprise:** Priya Nair @ Crunch Co didn't show up as a target on our pre-event TAM list — her company was below the original revenue threshold. The fresh fundraise signal from `agents_company_intelligence` is what bumped her into TIER_1. Wouldn't have caught this from the registration form alone.
- **Rate cap:** Hit it after the first 40. Next event, start the enrichment 24h earlier so all 240 resolutions fit in a daytime window instead of bleeding into evening.
- **Hook quality:** 0 warm-opener fallbacks. The agenda WebSearch surfaced enough specific session names that every TIER_1/TIER_2 hook got an event-specific reference. Good signal.
- **Total run time:** ~5 hours wall clock (mostly Step 2 LinkedIn pacing), ~25 min of active Claude time.
