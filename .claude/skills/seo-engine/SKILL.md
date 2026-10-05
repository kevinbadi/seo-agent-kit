---
name: seo-engine
description: Self-improving SEO blog engine. Scouts topics (winner spin-offs from PostHog signups, Search Console gaps, news, releases, seeds), writes and fact-gates posts against facts.json, auto-publishes to a WordPress blog through the Creator OS blog API, and re-weights topic clusters nightly by signups per post. Use when the user asks about blog posts, SEO, the SEO agent, search traffic, spin-offs of a winning post, setting up or scheduling the scout / publish / measure loop, rejected topics, or wants a specific blog topic written now. For writing or editing one article by hand use wordpress-blog.
license: MIT
compatibility: Requires Node 20+, Postgres, a Creator OS API key connected to WordPress, PostHog personal API key, and an Anthropic or OpenAI-compatible model key; Google Search Console service account and IndexNow key optional.
metadata:
  author: KevBuildsApps
  version: 1.0.0
---

# SEO engine

A Claude Code harness that runs in a loop: **write blog posts that rank, read PostHog to see which
ones bring signups, write more like the winners.** It is built on the PostHog snapshot tables in this
kit and the `wordpress-blog` skill for publishing.

```
 FEEDS                 PIPELINE (publish.mjs, 1 post per run)          LEARN (measure.mjs, nightly)
 spin-offs ─┐
 GSC gaps  ─┤  scout   brief -> write -> fact gate -> publish -> IndexNow   PostHog entry visitors,
 news RSS  ─┼──────>   (2 rewrites max, else rejected with a reason)        signups, AI referrals
 releases  ─┤  queue                                                      + Search Console clicks
 seeds     ─┘    ^                                                                  |
                 └──────── cluster weights (signups per post, 0.5x to 3x) <─────────┘
```

## Critical: the fact gate (do not loosen it)

publish rejects or fixes: em/en dashes (auto), under 700 words, under 4 H2s, `facts.banned`
strings, recurring prices not in `allowed_prices`, unknown CLI commands, invented flags
(auto-removed), undocumented API paths (if `api_prefixes` set), unknown internal links
(auto-unlinked), dead external links (auto-unlinked), missing signup CTA and YouTube link (auto-added).
Then an LLM audit lists unsupported product claims, competitor fact claims and invented stats.
Up to 2 rewrites, else the topic is rejected with the reason in `seo_topics.reason`.

Also critical: every topic must be a distinct search intent, competitor posts state only your
facts, and `facts.json` holds only real, checkable product facts. See Guard rails below.

## Setup

1. **Blog:** connect your WordPress site to Creator OS (see the `wordpress-blog` skill) and put
   that workspace's key in `.env` as `CREATOROS_API_KEY`.
2. **Config:** `cp seo.config.example.json seo.config.json` (repo root) and fill in your site,
   brand, signup path, the PostHog signup event, competitors, niches, YouTube channel, RSS feeds.
3. **Truth:** in this folder, `cp facts.example.json facts.json` and `cp seeds.example.json seeds.json`.
   Fill `facts.json` with real, checkable product facts. This is what stops the writer from inventing
   features. Optional: `cli.json` if your product has a CLI.
4. **Model:** `ANTHROPIC_API_KEY` (Claude), or `LLM_BASE_URL` + `LLM_API_KEY` + `LLM_MODEL` for any
   OpenAI-compatible API (OpenAI, OpenRouter, DeepSeek, Ollama). Set `POSTHOG_AI_KEY` to log every
   generation to PostHog LLM analytics.
5. **Search Console (optional but worth it):** create a Google Cloud service account, enable the
   Search Console API, add the service-account email as an **Owner** of your property, and set
   `GSC_SA_JSON_B64` (base64 of the JSON key) or `GSC_SA_JSON_PATH`.
6. **IndexNow (optional):** pick any 32-char hex key, serve it at `https://yoursite/<key>.txt`, set
   `INDEXNOW_KEY`. Bing (which feeds ChatGPT search and Copilot) then hears about each post instantly.

## Run it

```bash
npm run seo:scout -- --dry-run      # see what it would queue
npm run seo:publish -- --dry-run    # write + gate one post into a temp file, publish nothing
npm run seo:scout && npm run seo:publish && npm run seo:measure
```

Schedule (ET): scout 7:30, publish hourly 8..17 (one post per run, capped by `daily_max`),
measure 22:00. Any cron works: Railway cron, GitHub Actions, crontab.
Write a specific topic now: insert a `seo_topics` row (source 'manual', high score) and run
`npm run seo:publish -- --topic <id>`. Land posts as drafts instead of live: `SEO_PUBLISH_AS_DRAFT=1`.

## The learning loop

- measure attributes every PostHog session to the blog post it **entered** on (entry visitors,
  signup-page visits, signups, AI-engine visitors) and adds GSC clicks/impressions/position.
- cluster weight = Bayesian-smoothed signups per post vs the blog average (0.5x to 3x), visitors
  and AI referrals as tiebreaks. Queued topics are re-scored `score = base x weight` (never compounding).
- scout spins the top 3 winners (1+ signup, 15+ entry visitors or 3+ AI visitors in 14 days) into
  6 distinct-intent ideas each: comparison, model, niche, use-case, how-to. Capped per day.

## Guard rails (why they exist)

- Google demotes scaled, interchangeable content. Every topic must be a distinct search intent
  (token + semantic dedupe in scout, 0.8 title similarity in publish). Each post uses only the facts
  for its topic with a worked example, never a fact dump.
- Competitor posts state only your facts; competitors are described neutrally.
- Every post gets an FAQ section (h3 questions). The `ai-search-files` skill turns it into FAQPage
  JSON-LD so Google and AI answer engines can lift the answers.
- Cheap models invent CLI flags and features. That is why the gate is deterministic first and the
  LLM audit second. Watch `seo_topics.reason` for rejections; if many share a cause, fix `facts.json`.
- A model being retired shows up as every run failing. `POSTHOG_AI_KEY` makes that visible the same day.

## Troubleshooting

- `select job, ok, detail, created_at from seo_runs order by id desc limit 20;`
- `select status, count(*) from seo_topics group by 1;` (queue empty: run scout)
- `select * from seo_clusters order by weight desc;`
- `npm run seo:measure -- --days 14` to backfill metrics.
- Old posts missing a YouTube link: `node --env-file=.env .claude/skills/seo-engine/scripts/add-youtube-links.mjs --apply`.

Error: `CREATOROS_API_KEY missing (Creator OS > Settings > API keys)`
Cause: No Creator OS key in `.env`.
Solution: Use the key of the workspace that connected the WordPress blog (see `wordpress-blog`).

Error: `anthropic 4xx/5xx: ...` or `llm 4xx/5xx: ...` on every run
Cause: Bad model key, or the model was retired.
Solution: Check `ANTHROPIC_API_KEY` or `LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL`; pick a current model. `POSTHOG_AI_KEY` makes this visible the same day.

Error: `no JSON in reply` / `unterminated JSON in reply`
Cause: The model returned prose or truncated output instead of JSON.
Solution: Rerun; if it repeats, switch to a stronger model.

Symptom: `gsc feed failed` / `gsc metrics failed` / `sitemap submit failed`
Cause: Search Console service account missing or not an Owner of the property.
Solution: Set `GSC_SA_JSON_B64` or `GSC_SA_JSON_PATH` and add the service-account email as an Owner. Scout and measure keep running without it.

Symptom: many topics with status rejected
Cause: The fact gate keeps failing on the same missing fact.
Solution: Read `seo_topics.reason`; if many share a cause, fix `facts.json`. Do not loosen the gate.

Error: `set youtube_channel in seo.config.json` from `add-youtube-links.mjs`
Cause: No YouTube channel configured.
Solution: Add `youtube_channel` to `seo.config.json`.

## Examples

Example 1: First run
User says: "Set up the SEO agent on my WordPress blog"
Actions:
1. Follow Setup steps 1 to 6 (Creator OS key, `seo.config.json`, `facts.json`, `seeds.json`, model key, optional GSC and IndexNow).
2. Run `npm run seo:scout -- --dry-run` and `npm run seo:publish -- --dry-run`, review the temp post.
3. Schedule scout 7:30, publish hourly 8..17, measure 22:00 (ET).
Result: One gated post per publish run, landing live (or as drafts with `SEO_PUBLISH_AS_DRAFT=1`).

Example 2: Write a specific topic now
User says: "Write a post comparing my product to Buffer today"
Actions:
1. Insert a `seo_topics` row (source 'manual', high score).
2. Run `npm run seo:publish -- --topic <id>`.
Result: Published post, or a rejection with the reason in `seo_topics.reason`.

Example 3: Spin off a winner
User says: "My comparison post got signups, write more like it"
Actions:
1. Run `npm run seo:measure`, then `npm run seo:scout -- --dry-run` to see the spin-offs.
2. Run scout for real; publish runs pick them up.
Result: Up to 6 distinct-intent siblings per winner, capped per day.

