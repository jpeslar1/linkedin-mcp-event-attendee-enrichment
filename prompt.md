# Event Attendee Enrichment — Run on Event Day

A Claude Code prompt that takes an event attendee list (conference, trade show, webinar, Luma event, Eventbrite event, On24/BigMarker/Hopin/Zoom export), resolves every attendee **live on LinkedIn** via the LinkedIn MCP (Zevari), ICP-scores them, tiers them, and enrolls them in an event-specific outbound sequence with hooks that reference the event by name.

The reason this prompt exists: every CSV enrichment tool (Apollo, Clay, Lead411, ZoomInfo) returns whatever the vendor cached, which is 30-90 days stale. Event attendee lists are also stale by the time the event happens — people change titles in the 2-6 weeks between registration and event day. Stale-against-stale gets you a "personalized" email with a title that hasn't been true in months. Zevari resolves live, so the title in your outbound is right.

---

## Overview

You are running event attendee enrichment for `[EVENT_NAME]` (`[EVENT_DATE]`, slug `[EVENT_SLUG]`).

For each attendee on the registration list:

1. **Normalize** the record (trim whitespace, lowercase emails, clean company strings, dedupe)
2. **Resolve live to LinkedIn** via Zevari (`linkedin_get_profile`, `inbox_enrich_sender`, or `linkedin_search_profiles` — depending on what data the registration row has)
3. **Pull live company intelligence** on the attendee's confirmed current company
4. **ICP-score** against the rules in `profile_get`
5. **Bucket into tiers** — TIER_1 (perfect ICP, AE-assigned) / TIER_2 (event drip) / TIER_3 (general nurture) / DQ
6. **Write an event-referencing hook** via `agents_personalize` + `library_context_get`
7. **Enroll** in Instantly campaign `[INSTANTLY_EVENT_CAMPAIGN_ID]` with event-specific custom variables
8. **Save** to a Zevari list `event-[EVENT_SLUG]-[YYYY-MM-DD]` for audit
9. **Create or merge** the contact in Pipedrive, tagged with the event
10. **Slack summary** at the end with TIER_1 names and hooks (for the AE)

This is intentionally NOT batched in 10s like the cold-outbound prompt — a full attendee enrichment runs on the day of the event in one pass. If the list is >150 attendees, expect to hit LinkedIn's ~40/hour cap and pace accordingly (Step 2 explains how).

---

## ERROR & APPROVAL NOTIFICATIONS

If anything stops the task or needs approval, send a notification to `[ALERT_CHANNEL]`. The bash sandbox blocks `hooks.slack.com` — use Chrome:

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on a `chrome://` URL, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Event Enrichment — Action Needed* — [EVENT_NAME]\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — live profile resolution, name+company search, email-to-LinkedIn enrichment, company intelligence, ICP scoring, hook personalization, target save
- **Instantly:** `mcp__[INSTANTLY_MCP_ID]` — event campaign enrollment
- **Pipedrive:** `mcp__pipedrive__` — contact creation + event tagging
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook
- **WebSearch:** event context (keynote topics, vendor sessions, sponsors) for hook variety

Pick exactly ONE source for the attendee list:
- `[ATTENDEE_CSV_PATH]` — file path to a CSV export (Eventbrite, Luma, Hopin, On24, BigMarker, Zoom, badge-scanner export, manual list)
- `[LUMA_EVENT_ID]` — Luma event ID + API
- `[EVENTBRITE_EVENT_ID]` — Eventbrite event ID + API

See `connectors.md`.

---

## STEP 1 — LOAD ATTENDEE LIST

Use whichever source is configured. If multiple are filled in, prefer in this order: CSV path → Luma → Eventbrite.

### Step 1a — Source loaders

**If CSV (`[ATTENDEE_CSV_PATH]`):**
Read the file. Detect columns. Common Luma/Eventbrite/On24/Hopin/BigMarker/Zoom column patterns:
- `name`, `first_name`, `last_name`, `full_name`
- `email`, `Email Address`
- `company`, `Company`, `Organization`, `Company Name`
- `title`, `Job Title`, `Role`
- `linkedin`, `LinkedIn URL`, `linkedin_url`

If headers are non-standard, ask the user before guessing.

**If Luma (`[LUMA_EVENT_ID]`):**
Pull guests via Luma API (`/event/get-guests`). For each guest, you get `name`, `email`, `answers[]` (custom registration questions — company is usually here), and sometimes `linkedin_url`.

**If Eventbrite (`[EVENTBRITE_EVENT_ID]`):**
Pull attendees via Eventbrite API (`/v3/events/{id}/attendees`). Custom questions for company/LinkedIn live in `answers[]`.

### Step 1b — Normalize EVERY row before resolving

This is the single most common source of bugs in this pipeline. Apply in order:

1. **Trim whitespace** on name, company, email (Luma especially leaves trailing spaces)
2. **Lowercase email** — registration forms accept "John@Acme.com" and "john@acme.com" as different rows
3. **Split `full_name`** into `first_name` / `last_name` if you only have one field
4. **Strip company suffixes** for matching: ", Inc.", " Inc", " LLC", ", LLC", " Ltd", ", Ltd."
   - But KEEP the original company string — you'll use the normalized version only for matching/dedupe
5. **Dedupe** by lowercase-email, then by (first+last+normalized-company) for rows missing email

Log the raw count vs. normalized count. Big delta = bad source data.

---

## STEP 2 — LIVE RESOLUTION VIA ZEVARI

For each normalized attendee, pick a resolution path based on what data you have. In priority order:

### Path A — LinkedIn URL provided
Call `linkedin_get_profile` with the URL.
- If it resolves → save the live profile (current title, current company, current company URL, headline)
- If it 404s or the URL is malformed → fall back to Path C

### Path B — Email only (no LinkedIn URL)
Call `inbox_enrich_sender` with the email. Zevari attempts to resolve the email to a LinkedIn profile.
- If it returns a profile → call `linkedin_get_profile` on the resolved URL for the full data
- If it can't resolve (typical for free-email registrants — gmail.com, outlook.com — with no other signal) → fall back to Path C

### Path C — Name + company only
Call `linkedin_search_profiles` with `name` + company-as-keyword filter.
- If exactly 1 confident match → call `linkedin_get_profile` on that URL
- If 2-3 candidates → pick the one whose current company string matches the registration company best. Log the decision in the audit trail (which candidate, why).
- If 0 confident matches or >3 ambiguous matches → log as **unresolved**, do NOT guess, move on.

### Rate-limit pacing
LinkedIn's safe ceiling via Zevari is ~40 lookups/hour. Plan accordingly:

- For lists ≤ 40 attendees → run straight through
- For 40-150 attendees → run in 40-lookup batches with 60-min pauses between batches, OR split across two days
- For 150+ attendees → tier the work: do Tier-1 candidates first (e.g. attendees with named-account companies your AE flagged), full sweep over 2-3 days

If you hit the cap mid-run, save the remaining queue to a `unresolved-pending-[YYYY-MM-DD].csv` and flag in Slack. Do NOT silently fall back to using the registration title — that's the whole anti-pattern this prompt avoids.

### Unresolved
Track every unresolved attendee in a separate `unresolved.csv` with the raw row + the reason (no URL + no match, ambiguous candidates, rate cap hit, etc.). Surface the count in the Slack summary. A human can do a 5-minute pass to recover the actually-valuable ones.

---

## STEP 3 — COMPANY INTELLIGENCE ON CONFIRMED COMPANIES

For each resolved attendee, call `agents_company_intelligence` on their **live confirmed current company** — NOT the company string from the registration form (which is stale).

This returns: company size, industry, recent news, funding, hiring signals, HQ location.

If `agents_company_intelligence` returns thin data, optionally fall back to `linkedin_get_company` on the company URL.

---

## STEP 4 — ICP SCORE + TIERING

### Step 4a — Load current ICP

Call `profile_get` to load the active ICP criteria, hard disqualifiers, employee thresholds, industry buckets, company type rules. Never hardcode — always use what `profile_get` returns.

### Step 4b — Score each resolved attendee

Call `agents_icp_score` for each resolved attendee using the company intelligence + their live title.

If `agents_icp_score` returns `server_side_ai_unavailable`, apply the ICP criteria manually from `profile_get` data. Do not pause.

### Step 4c — Bucket into tiers

Use this tier rubric (defaults — override if `profile_get` returns explicit tier thresholds):

- **TIER_1 — ICP perfect** (score ≥ 9, title in target buyer set, company size in core range)
  → Enroll in `[INSTANTLY_EVENT_CAMPAIGN_ID]` AND ping the AE in Slack at end (`[EVENT_CHANNEL]`) with full context
- **TIER_2 — ICP fit** (score 7-8)
  → Enroll in `[INSTANTLY_EVENT_CAMPAIGN_ID]` (event-specific drip)
- **TIER_3 — ICP loose** (score 5-6)
  → Add to a `event-{slug}-nurture-{date}` Zevari list. No email enrollment by default. AE can promote later.
- **DQ** (score < 5 OR hits any hard disqualifier)
  → Archive in Zevari via `targets_archive`, log in Slack count, do not contact

Save the tier on each attendee record for downstream use.

---

## STEP 5 — WRITE EVENT-REFERENCING HOOKS

Call `library_context_get` with `skill_slug='craft-outreach'` and `include_examples=true`. Do not write any hooks until you've read this — formatting rules live there.

For TIER_1 + TIER_2 only (TIER_3 doesn't get a hook, no email goes out by default).

Use `agents_personalize` to draft the hook with the event explicitly referenced.

### Hook rules (event edition)

1. **Reference the event by name in the first 7 words.** This is the highest-converting format for event outbound.
   - ✓ `"Saw you registered for ai in cpg - hope the keynote was useful"`
   - ✓ `"Caught your name on the ai in cpg attendee list"`
   - ✗ `"Hope you had a great week!"` (no event reference)
   - ✗ `"Hey — I noticed you signed up for the upcoming AI conference"` (event name buried)

2. **Lowercase everything after the very first letter** — including event name, brand names, people's names, place names. Reads like a casual text.
   - ✓ `"Saw you registered for ai in cpg - hope the snackable.ai session was useful"`
   - ✗ `"Saw you registered for AI in CPG - hope the Snackable.ai session was useful"`

3. **No LinkedIn references in the hook.** Write as if you noticed the registration through normal industry channels.

4. **Specific session/keynote callouts beat generic.** If you can name a specific session, sponsor talk, or keynote, do it. Use WebSearch on `[EVENT_NAME]` for the agenda if you don't have it.
   - ✓ `"Saw you registered for ai in cpg - was the [vendor] session worth the trip?"`
   - ✓ `"Caught your name on ai in cpg - did the [keynote name] keynote hold up?"`

5. **Reactions: brief.** `congrats!` `hope it was useful!` `worth the trip?` `enjoy the keynote!` — pick one, stop.

6. **No pitch in the hook.** The hook earns the open. The Instantly body sequence pitches.

7. **Warm opener fallback:** `"Hope ai in cpg was useful."` Only if you cannot get the event-specific angle right (e.g. event hasn't happened yet AND you don't know which sessions they care about).

### companyRef rules
- Short recognizable brand name, title case (use the LIVE company from Zevari, NOT the registration form)

### industryRef rules
- Match the buckets your Instantly campaign expects ("food brands", "bev brands", "saas teams", "agency leaders", etc. — pull from `library_context_get`)

---

## STEP 6 — ENROLL IN INSTANTLY (TIER_1 + TIER_2)

Call `add_leads_to_campaign_or_list_bulk` to enroll in `[INSTANTLY_EVENT_CAMPAIGN_ID]` (`"[EVENT_NAME] — Event Follow-up"`).

Required fields:
- `email`, `first_name`, `last_name`, `company_name` (LIVE company from Zevari)
- `custom_variables: {event_name, event_date, hook, companyRef, industryRef}`

Set `skip_if_in_campaign: true`.

The campaign already has the event-specific sequence (typically: D+0 event-referencing hook, D+3 value-add, D+7 soft CTA, D+14 breakup). Do not create or modify the sequence here — just enroll.

---

## STEP 7 — SAVE TO ZEVARI (audit list)

For all resolved attendees (TIER_1, TIER_2, TIER_3 — not DQ):

1. Call `targets_save` for each resolved attendee (with their LIVE title + company + LinkedIn URL)
2. Create a list named `event-[EVENT_SLUG]-[YYYY-MM-DD]` via `targets_create_list`
3. Add all resolved attendees to that list

This is the audit trail. If someone converts six months later, you can trace which event they came from.

Optionally save TIER_3 to a sub-list `event-[EVENT_SLUG]-nurture-[YYYY-MM-DD]` for separate AE workflows.

---

## STEP 8 — PIPEDRIVE CONTACT CREATION + EVENT TAG

For TIER_1 + TIER_2 (TIER_3 optional — depends on your AE process):

1. Search Pipedrive for existing contact via `search_contacts` (by email, then by name)
2. If found → update `job_title` and link organization (creating the org if it doesn't exist). Append a note `"Attended [EVENT_NAME] on [EVENT_DATE]. Live-confirmed title: [live_title] @ [live_company]"`.
3. If not found → create a new contact with the LIVE Zevari title + company, and link to the (created or existing) organization.
4. Add a tag `event-[EVENT_SLUG]` to every contact (so the AE can filter "all attendees from this event" in Pipedrive).

The CRM should reflect the LIVE title, NOT the registration form title. This is non-negotiable — the whole point of the prompt is title accuracy.

---

## STEP 9 — TIER_1 SLACK PING (AE handoff)

TIER_1 attendees deserve a human touch in addition to the automated sequence. Conference leads have a 14-day decay curve and an Instantly email that lands four days later is too slow for the top of the list.

For each TIER_1 attendee, post to `[EVENT_CHANNEL]` (via Chrome `javascript_tool`):

```
<@[YOUR_SLACK_USER_ID]> 🎯 *TIER_1 from [EVENT_NAME]*

[Name] — [LIVE title] @ [LIVE company]
LinkedIn: [URL]
ICP score: [N]/10
Why TIER_1: [one-line rationale from agents_icp_score]
Hook used: [hook]

→ Recommend manual LinkedIn note within 48h on top of the Instantly sequence.
```

One ping per TIER_1 attendee. If there are >10 TIER_1, consolidate into a single ranked summary instead of spamming the channel.

---

## STEP 10 — RUN SUMMARY SLACK

Post to `[EVENT_CHANNEL]` via Chrome `javascript_tool`:

```
<@[YOUR_SLACK_USER_ID]> 🎫 *Event Enrichment Complete — [EVENT_NAME]*

📅 Event date: [EVENT_DATE]
📥 Attendees loaded: [N]
🧹 After normalization/dedupe: [N]
🔍 Resolved live on LinkedIn: [N]
❓ Unresolved: [N] (saved to unresolved.csv for manual pass)

Tier breakdown:
🎯 TIER_1 (perfect ICP, AE pinged): [N]
✅ TIER_2 (drip enrolled): [N]
📥 TIER_3 (nurture list): [N]
🚫 DQ: [N]

📧 Instantly enrollments: [N]
📇 Pipedrive contacts created/updated: [N]
🏷  Zevari audit list: event-[EVENT_SLUG]-[YYYY-MM-DD]

Sample TIER_1 hooks:
• [name, company] — [hook]
• [name, company] — [hook]
• [name, company] — [hook]
```

---

## STEP 11 — REFLECTION

After the summary, briefly note:

- **Unresolved rate** — if > 30% of the list is unresolved, the source data is too thin (probably a free-email-heavy registration). Flag for the marketing team to require company on next event's registration form.
- **TIER_1 surprises** — any attendees who you thought would be TIER_3 but scored as TIER_1 because Zevari surfaced a current title that wasn't on the registration form. These are the wins.
- **Title drift** — count of attendees whose LIVE title differs from the registration title. This is the receipt for "why Zevari over Apollo CSV enrich" — show the number to whoever questions the workflow.
- **Hook quality** — any hooks that fell back to the warm opener. If > 15%, sharpen the event WebSearch for next time (was the agenda public? sponsors named?).
- **Rate-limit notes** — if the cap was hit, plan the next event's enrichment to start earlier (day-before registration close, not day-of).

---

## Schedule

Run on event day (or day-after for post-event registrations). For multi-day conferences, run once on day 1 (catches early registrants), once on day 3 (catches late add-ons), once 48h post-event (catches no-shows who still want the deck — those are often higher-intent than attendees).

Save the prompt as a Claude Code slash command per recurring event series. e.g. `/enrich-event-aicpg` for an annual conference, with the placeholders pre-filled.
