# Connectors

Every MCP server this prompt uses, with setup links and the placeholders to replace in `prompt.md`.

## Zevari — LinkedIn MCP for Claude (REQUIRED, load-bearing)

**What it does:** Live LinkedIn profile lookup (`linkedin_get_profile`), name+company live search (`linkedin_search_profiles`), email-to-LinkedIn resolution (`inbox_enrich_sender`), company intelligence (`agents_company_intelligence`), ICP scoring (`agents_icp_score`), hook personalization (`agents_personalize`), and target save (`targets_save`).

**Why it's the core of this workflow:** Every "event lead enrichment" tool — Apollo CSV enrichment, Clay, Lead411, ZoomInfo bulk — matches on email or company string and returns whatever the vendor cached. That data is 30-90 days behind LinkedIn. Event attendee lists are *additionally* stale because registrations happen 2-6 weeks before the event. The whole point of the prompt is to resolve every attendee live on event day so the title in your outbound email is the title they actually have today — not the title the registration form caught.

The other huge unlock is name+company live resolution. Most attendee rows from Luma / Eventbrite / On24 / Hopin don't have a LinkedIn URL — they have a free email and a self-typed company. Apollo can't match these confidently for common names. Zevari's `linkedin_search_profiles` lets you actually pick the right match instead of guessing.

**Setup:** [zevari.ai](https://zevari.ai) → connect your LinkedIn account → copy the MCP server URL into your Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders to replace in `prompt.md`:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID (looks like `mcp__a1b2c3d4-...`)

**Endpoints used:**
- `linkedin_get_profile` — live current title + company (the load-bearing call)
- `linkedin_search_profiles` — name+company live search for rows without a LinkedIn URL
- `inbox_enrich_sender` — email → LinkedIn URL resolution
- `agents_company_intelligence` — live confirmed-company size, industry, news, signals
- `linkedin_get_company` — fallback if `agents_company_intelligence` returns thin data
- `profile_get` — load current ICP rules
- `agents_icp_score` — score each attendee against ICP
- `agents_personalize` + `library_context_get` (`craft-outreach`) — hook formatting
- `targets_save` + `targets_create_list` — save the audit list
- `targets_archive` — archive DQs

## Instantly — Event-specific cold email enrollment (REQUIRED)

**What it does:** Enroll TIER_1 and TIER_2 attendees in an event-specific outbound campaign with `{event_name, event_date, hook, companyRef, industryRef}` as custom variables.

**Why it's in this workflow:** Campaign sequence + signature + sending pool are already configured for the event series. The prompt just enrolls — no campaign editing, no send-side logic.

**Setup:** [Instantly MCP docs](https://developer.instantly.ai/mcp) — install the MCP, paste your API key. Build a campaign per event (or per recurring event series) with a 4-email sequence: D+0 event-referencing hook, D+3 value-add, D+7 soft CTA, D+14 breakup. Get the campaign UUID.

**Placeholders:**
- `[INSTANTLY_MCP_ID]` — your Instantly MCP server ID
- `[INSTANTLY_EVENT_CAMPAIGN_ID]` — the event-specific campaign UUID

**Endpoints used:**
- `list_leads` — dedupe check (avoid double-enrolling attendees who were in a prior batch)
- `add_leads_to_campaign_or_list_bulk` — enrollment

## Pipedrive — Contact creation + event tagging (REQUIRED)

**What it does:** Creates or merges contacts with the LIVE Zevari title + company (NOT the stale registration form title), and tags every attendee with an event-specific tag so the AE can filter "all attendees from this event" later.

**Why it's in this workflow:** Multi-event attribution. If your team runs 8 events a year, you need to know which event a deal came from. The tag is how.

**Setup:** Any Pipedrive MCP server. I use `mcp__pipedrive__` from the community list.

**Endpoints used:**
- `search_contacts` / `search_organizations` — dedupe before creating
- `add_person` / `update_person` — contact create/update
- `add_organization` — create org if it doesn't exist
- `add_note` — log the event attendance + live title for audit
- `add_tag` / `update_person_tags` — apply `event-[EVENT_SLUG]` tag

## Attendee source — Pick ONE (REQUIRED)

The prompt accepts three source types. Configure exactly one.

### Option A — CSV file (most common)

**Where it comes from:** Eventbrite export, Luma export, Hopin export, On24 export, BigMarker export, Zoom registration export, badge scanner export, manual list.

**Placeholder:**
- `[ATTENDEE_CSV_PATH]` — absolute path to the CSV on your machine

No MCP needed — the prompt reads the file directly.

### Option B — Luma API

**What it does:** Pulls the attendee list directly via Luma's `/event/get-guests` endpoint.

**Setup:** Get a Luma API key from your Luma calendar settings.

**Placeholder:**
- `[LUMA_EVENT_ID]` — the Luma event ID (looks like `evt_...`)

### Option C — Eventbrite API

**What it does:** Pulls attendees via Eventbrite's `/v3/events/{id}/attendees` endpoint.

**Setup:** Eventbrite Personal OAuth token from your account settings.

**Placeholder:**
- `[EVENTBRITE_EVENT_ID]` — the Eventbrite numeric event ID

## Claude in Chrome — Slack webhook

**What it does:** Posts to `hooks.slack.com` (the bash sandbox blocks this URL — Chrome is the reliable path).

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook
- `[EVENT_CHANNEL]` — Slack channel where TIER_1 pings + run summary post
- `[ALERT_CHANNEL]` — Slack channel where errors go
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) so the bot @-mentions you

## WebSearch

Built into Claude Code. Used for:
- Pulling the event agenda (keynote, sessions, sponsors) for richer hooks
- Looking up vendor session names if attendees registered for a specific track
- Surfacing recent press on the company an attendee works at (for TIER_1 AE briefings)

No placeholder — just available.

## Event placeholders (set once per event)

These aren't MCP connectors but you need to set them in `prompt.md`:

- `[EVENT_NAME]` — the human-readable event name (e.g. `AI in CPG 2026`)
- `[EVENT_DATE]` — `YYYY-MM-DD`
- `[EVENT_SLUG]` — lowercase, hyphen-separated, no spaces (e.g. `ai-in-cpg-2026`). Used in Zevari list names and Pipedrive tags.
