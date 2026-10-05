---
name: posthog-analytics
description: Set up, backfill, debug or extend the PostHog analytics kit (daily visitor/funnel/traffic-source snapshots from PostHog HogQL into Postgres, plus the Web chart, Web funnel and Traffic sources dashboard cards). Use when the user asks about website visitors, signups, conversion, traffic sources, AI referrals, adding a funnel event or traffic family, missing days, or PostHog numbers looking wrong (zero tonight, capped at 100 rows, 401/403). Not for blog post performance or topic weighting (use seo-engine).
license: MIT
compatibility: Requires Node 20+, Postgres, and a PostHog personal API key (phx_) with Query Read access. Dashboard cards need React 18+, Tailwind and Recharts.
metadata:
  author: KevBuildsApps
  version: 1.0.0
---

# PostHog analytics kit

## Critical: rules that matter
- **Days are America/New_York.**
  - In HogQL, bucket with `toDate(toTimeZone(timestamp, 'America/New_York'))`.
  - For "today", compare against `toDate(toTimeZone(now(), 'America/New_York'))`, never `today()`, which is UTC. `today()` makes the evening count read 0.
  - In JS, use `Intl.DateTimeFormat("en-CA", { timeZone: "America/New_York" })`, never `toISOString()`.
- **HogQL defaults to LIMIT 100.** Always set an explicit LIMIT on grouped queries. The sources query uses 20000.
- Visitors = `count(DISTINCT person_id)` on `$pageview`. Sessions = distinct `properties.$session_id`.
- Attribute sources by **session entry** (`session.$entry_referring_domain`, `session.$entry_utm_source`), not the per-pageview referrer. The per-pageview referrer is mostly your own domain after the first click.
- The API key must be a **personal** key (`phx_`) with Query: Read. A project key (`phc_`) returns 401/403.
- In the Web chart, signups and subscribers share the **visitors** y-axis with the **same bar width**. Height shows the conversion. Never give them their own axis, or 1 sale renders as a full-height bar.

## How it works
- `scripts/posthog-snapshot.mjs` runs two HogQL queries against `POST {host}/api/projects/{id}/query/`:
  1. Visitors, pageviews, sessions and funnel events grouped by **ET day**. The results go to `web_analytics_snapshots`, keyed on (day, channel).
  2. The session entry source by ET day, which goes to `web_referrer_snapshots`. The source is `utm:<source>` when a UTM exists, else the referring domain, else `$direct`.
- It upserts, so re-running is always safe. `--days N` backfills (max 400).
- It always writes a heartbeat row for today, so a monitor can tell the job ran even on a zero-traffic day.
- `lib/posthog-client.ts` `getWebVisitorsDaily()` reads the table and first runs a live HogQL top-up for today. The top-up has a 90 s cooldown and a 3 s time box and fails open.
- `lib/posthog-sources.ts` groups raw sources into families with the ordered regex `RULES`. The first match wins. Unknown domains show as themselves with a favicon.

## Common tasks
- **Add a funnel event:** add `countIf(event = 'x') AS x` to both HogQL queries (the snapshot and the live top-up). Add the column with `alter table ... add column if not exists`, add it to the upserts, then map it in `getWebVisitorsDaily`.
- **Add a traffic family:** add a rule to `RULES` above the generic `search` / `direct` rules.
- **Numbers look low or zero tonight:** check the ET vs UTC "today" comparison. Then run `npm run snapshot` and read the per-project log line.
- **Missing days:** run `npm run backfill`.
- **Tracking a new site:** set `POSTHOG_PROJECT_ID` (single-site mode), or add a row to the `channels` table (multi-site mode).

## Setup

```bash
cp .env.example .env     # fill DATABASE_URL, POSTHOG_API_KEY, POSTHOG_PROJECT_ID (and POSTHOG_HOST for EU)
npm install
npm run backfill         # last 90 days
npm run snapshot         # then schedule at 9 AM + 9 PM ET with any cron
```

## Examples

Example 1: First setup
User says: "Hook my site's PostHog data into the dashboard"
Actions:
1. Confirm PostHog is installed on the site and funnel events are captured.
2. Fill `.env`, run `npm run backfill`, then schedule `npm run snapshot` twice a day.
Result: `web_analytics_snapshots` and `web_referrer_snapshots` filled for 90 days, cards render.

Example 2: Track a new funnel step
User says: "Add trial_started to my funnel"
Actions:
1. Follow "Add a funnel event" above (both HogQL queries, column, upserts, `getWebVisitorsDaily`).
2. Run `npm run backfill` to fill history.
Result: The new step shows in the Web funnel.

Example 3: Numbers look wrong
User says: "PostHog says 0 visitors today but I know people visited"
Actions:
1. Check the "today" comparison uses `toTimeZone(now(), 'America/New_York')`, not `today()`.
2. Run `npm run snapshot` and read the per-project log line.
Result: Evening counts match PostHog.

## Troubleshooting

Error: `POSTHOG_API_KEY / DATABASE_URL required`
Cause: Missing env.
Solution: Fill `.env` from `.env.example` and run through the npm scripts (they pass `--env-file=.env`).

Error: `PostHog <project> -> 401` or `403`
Cause: A project key (`phc_`) was used, or the personal key lacks Query: Read.
Solution: Create a personal key (`phx_`) with Query: Read.

Symptom: tonight's numbers read 0 or low
Cause: "today" compared with UTC `today()`.
Solution: Compare against `toDate(toTimeZone(now(), 'America/New_York'))`.

Symptom: traffic sources stop at 100 rows
Cause: HogQL defaults to LIMIT 100.
Solution: Set an explicit LIMIT on grouped queries (sources uses 20000).

Symptom: most traffic shows as your own domain
Cause: Per-pageview referrer used instead of session entry.
Solution: Attribute by `session.$entry_referring_domain` / `session.$entry_utm_source`, and set `SITE_DOMAIN` so self-referrals group as Internal.

Symptom: missing days in charts
Cause: Snapshot job did not run those days.
Solution: `npm run backfill` (or `--days N`, max 400).

Error: SSL connection error against a local Postgres
Cause: The script connects with SSL by default.
Solution: Set `PGSSL=0` for a local Postgres without SSL.
