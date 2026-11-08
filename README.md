# linkedin-mcp-event-attendee-enrichment

> **Part of the [AI SDR Kit](https://github.com/zevarihq/ai-sdr-kit), the open-source AI SDR harness.** Install the kit to give your agent the whole outbound playbook (setup SOP and every LinkedIn and email sequence), then run this play on top of it.

A Claude Code prompt for working an event attendee list — conference, trade show, webinar, Luma meetup, Eventbrite event, LinkedIn Live audience — and resolving every attendee **live on LinkedIn** so the title in your outbound email is right on the day of the event, not the title the registration form caught six weeks ago.

Sharing it because every "event lead enrichment" tool I've tried gets you a stale title and a fake-confident match.

## The problem with CSV enrichment tools

Apollo CSV enrichment, Clay, Lead411, ZoomInfo bulk — they all match attendees the same way:

- Email match → return whatever Apollo cached for that email (often 30-90 days stale)
- Company name match → "Acme" matches three different Acmes, pick whichever Apollo guesses
- Name + company fuzzy match → confidence scores you can't audit

Event attendee lists are notoriously dirty. Registrations happen 2-6 weeks before the event. People change titles in that window. Companies get acquired and rebranded. The registration form captures "Acme" but the attendee actually works at "Acme Holdings, Inc., a Brand of XYZ." Apollo doesn't know about that change yet — your "personalized" event email goes out with a title that hasn't been true since March.

## What this does instead

The prompt loads an attendee list (CSV path, Luma event ID, or Eventbrite event ID), normalizes the data, then for each attendee:

1. Resolves them to a LinkedIn profile **live** via the LinkedIn MCP ([Zevari](https://zevari.ai))
   - LinkedIn URL provided → direct profile lookup
   - Email only → Zevari resolves email to LinkedIn, then profile lookup
   - Name + company only → live name+company search, confirm match, profile lookup
   - No confident match → log unresolved, skip
2. Pulls company intelligence on the **confirmed current company** (not the one on the registration form)
3. Scores each attendee against your ICP rules (Zevari)
4. Buckets into tiers: TIER_1 (perfect ICP, AE-assigned) / TIER_2 (drip) / TIER_3 (nurture) / DQ
5. Writes a hook that **references the event by name** in the first 7 words
6. Enrolls in your event-specific Instantly campaign with `{event_name, event_date, hook, companyRef, industryRef}`
7. Creates/merges contacts in Pipedrive and tags them with the event
8. Saves to a Zevari list `event-{event-slug}-YYYY-MM-DD` for audit
9. Slack summary with TIER_1 names and hooks for the AE

## Why Zevari (and not just Apollo or Clay)

The whole point of the prompt: event attendee lists are stale by the time the event happens. If you enrich them against a stale source, you compound the problem.

- **Apollo CSV enrichment / Clay / Lead411 / ZoomInfo** = good for bulk hygiene, but the title and company they return are whatever the vendor cached, which is 30-90 days behind LinkedIn
- **[Zevari](https://zevari.ai)** = hits LinkedIn directly through an MCP server that throttles to safe rate limits, returns today's title and today's company, and can resolve name+company live (which Apollo can't do confidently for common names)

The single biggest unlock is the name+company live resolution. Half the entries on a typical Luma export have no LinkedIn URL. Apollo guesses. Zevari searches and lets you pick the right match.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one — his README is honest about ToS risk. For event work where the title accuracy is the whole point I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Live LinkedIn resolution + ICP + hook personalization | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The load-bearing piece — accurate titles on event day, name+company live search, ICP scoring |
| Event-specific email enrollment | [Instantly](https://instantly.ai) | Campaign + sequence already configured; the prompt just enrolls with event custom variables |
| Contact creation + event tagging | [Pipedrive](https://pipedrive.com) | Tracks attribution per-event so you can measure pipeline by conference |
| Source list (optional) | [Luma](https://lu.ma) / [Eventbrite](https://eventbrite.com) / On24 / BigMarker / Hopin / Zoom | Whatever exported the attendee CSV; the prompt handles any of them |
| Notifications | Slack | TIER_1 pings + run summary |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |
| WebSearch | Claude Code built-in | Event context (keynote, sessions, sponsors) for hook variety |

## How to use this

1. Clone the repo
2. Open `prompt.md`
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[EVENT_NAME]` + `[EVENT_DATE]` + `[EVENT_SLUG]` — the event you're working
   - `[ATTENDEE_CSV_PATH]` OR `[LUMA_EVENT_ID]` OR `[EVENTBRITE_EVENT_ID]` — pick one source
   - `[INSTANTLY_EVENT_CAMPAIGN_ID]` — your event-specific campaign UUID
   - `[ZEVARI_MCP_ID]`, `[INSTANTLY_MCP_ID]` — your MCP server IDs
   - `[SLACK_WEBHOOK_URL]`, `[EVENT_CHANNEL]`, `[ALERT_CHANNEL]`, `[YOUR_SLACK_USER_ID]` — Slack
4. Paste into Claude Code (I save it as a slash command per recurring event series)
5. Run it the morning of the event, or the morning after registration closes

See `connectors.md` for full setup.

## What gets generated (sample)

See `examples/sample-run.md` for a real run on a 247-row Luma export from a fictional "AI in CPG 2026" conference — 198 resolved live, 31 TIER_1, 64 TIER_2, 103 TIER_3, 49 DQ. Three sample TIER_1 hooks included.

## Things I learned building this

- **Attendee lists are dirty in predictable ways.** Trailing whitespace in names. Company string mismatches ("Acme" vs "Acme, Inc." vs "Acme Inc"). Email-only entries with no LinkedIn URL. Mixed-case emails. Normalize FIRST, resolve SECOND. If you resolve against unnormalized data the dedupe step misses the duplicates.
- **The single biggest unlock is live name+company resolution.** Most attendee rows don't have a LinkedIn URL. Apollo's name+company match is a confidence guess; Zevari's `linkedin_search_profiles` lets you actually see the candidates and pick the right one. For common names ("John Smith @ Acme") this is the difference between sending an email to the right person vs. an embarrassing one to a stranger.
- **Event-specific hooks need the event name in the first 7 words.** "Saw you registered for [event]" is the format that gets the open. Generic "congrats on the move" hooks don't work for events — the recipient is in event-mode that week and pattern-matches on event mentions.
- **TIER_1 attendees deserve a human AE Slack ping, not an automated drip.** Conference leads have a 14-day decay curve. By day 15 the interest is gone and they've forgotten the event. Your AE has to move fast; an Instantly sequence that lands four days later is too slow for the top of the list.
- **Always update the CRM with the LIVE title from Zevari, not the title from the event registration form.** Registration forms are 2-6 weeks stale. If you trust the form, you'll pitch a "Director of Marketing" who's been "VP Brand" for a month and the email will sound out of date.
- **Resolve unresolveds separately.** ~20% of any attendee list won't resolve cleanly (free-email registrants with common names, no company on the form). Don't junk them — save the unresolved list and have a human do a 5-minute pass to confirm matches. That recovers another ~5-10% of TIER_1.

## Adapting to your event stack

This works with any attendee source that gives you (name, company, email, optional LinkedIn URL). Luma and Eventbrite have direct API integrations. On24, BigMarker, Hopin, Zoom registrations, and physical-conference badge scans all export to CSV — point the prompt at the file path.

Swap Instantly for any cold email engine your campaign already runs in. Swap Pipedrive for HubSpot, Attio, Salesforce — the contact-create-and-tag pattern is the same.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Real-time webhook → live ICP score → HOT/WARM/COLD routing
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — Classify every Gmail thread by LinkedIn-confirmed sender intent
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-webinar-followup](https://github.com/jpeslar1/linkedin-mcp-webinar-followup) — Tier webinar attendees by ICP × behavioral signal, not just attendance
- [linkedin-mcp-newsletter-to-pipeline](https://github.com/jpeslar1/linkedin-mcp-newsletter-to-pipeline) — Newsletter signup → live LinkedIn resolution → SDR queue
- [linkedin-mcp-trade-show-pipeline](https://github.com/jpeslar1/linkedin-mcp-trade-show-pipeline) — 3-phase trade show pipeline with live LinkedIn enrichment
- [linkedin-mcp-icp-discovery](https://github.com/jpeslar1/linkedin-mcp-icp-discovery) — Seed best customers → behavioral lookalike scoring → live-confirmed top-tier list

Or browse them all in [awesome-linkedin-mcp](https://github.com/jpeslar1/awesome-linkedin-mcp) — the curated index of LinkedIn MCP workflows, servers, and adjacent tools.

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT. Use it, fork it, ship it.

## Who I am

John Peslar — solo founder, build outbound automations for B2B clients. If you want to talk shop: [johnpeslar.com](https://johnpeslar.com).
