---
name: ai-search-files
description: Make a website's blog readable by search engines and AI answer engines (ChatGPT, Claude, Perplexity, Gemini, Copilot) with auto-generated llms.txt and llms-full.txt, a sitemap with every post, robots rules for AI crawlers, BlogPosting + FAQPage JSON-LD, IndexNow pings, and AI-referral tracking in PostHog. Use when the user asks about GEO, AEO, llms.txt, sitemaps, robots.txt for AI bots, schema / JSON-LD, IndexNow, getting cited by ChatGPT, or why AI tools don't mention their site. Not for writing the posts themselves (use wordpress-blog or seo-engine).
license: MIT
compatibility: Templates target Next.js App Router (TypeScript) and read posts from a WordPress REST API; the logic ports to any framework. IndexNow key, Google Search Console and PostHog are optional.
metadata:
  author: KevBuildsApps
  version: 1.0.0
---

# AI search files (GEO / AEO)

Writing the blog post is half the job. Agents and search engines also need the post in formats
they read. This skill wires those formats to the live WordPress post list, so every new post (from
the `seo-engine` loop, the `wordpress-blog` skill or a human) shows up everywhere with no extra step.

| File | What it does | Template |
|---|---|---|
| `/sitemap.xml` | every post with its real `lastModified`, for Google and Bing | `templates/sitemap.ts` |
| `/robots.txt` | lets GPTBot, ClaudeBot, PerplexityBot etc. in, points to the sitemap | `templates/robots.ts` |
| `/llms.txt` | plain-text map of the site + every post with its excerpt | `templates/llms.ts` + `llms-route.ts` |
| `/llms-full.txt` | the same plus the full text of every article | same, `buildLlmsTxt(true)` |
| JSON-LD on each post | BlogPosting + BreadcrumbList + **FAQPage** from the post's FAQ section | `templates/article-jsonld.ts` |
| IndexNow | tells Bing (which feeds ChatGPT search and Copilot) about new URLs instantly | seo-engine `lib.mjs` `indexNow()` |

`templates/wp-blog.ts` reads posts from the WordPress REST API (`/wp-json/wp/v2/posts`). All the
files above use it.

## Install (Next.js App Router)

1. Copy `templates/wp-blog.ts`, `llms.ts`, `article-jsonld.ts` into `lib/`. Copy `sitemap.ts` and
   `robots.ts` into `app/`. Copy `llms-route.ts` to `app/llms.txt/route.ts` and again to
   `app/llms-full.txt/route.ts` (with `buildLlmsTxt(true)`).
2. Set `SITE_URL` and `BLOG_ORIGIN` (and `BLOG_WP_PATH` if WordPress is not at `/blog`). Fill in
   `NAME`, `SUMMARY` and `PAGES` in `llms.ts`, and the org name in `article-jsonld.ts`.
3. In the blog article page, render `articleJsonLd(post)` in a `<script type="application/ld+json">`.
4. **IndexNow:** pick a 32-char hex key, put a file `public/<key>.txt` containing just the key,
   set `INDEXNOW_KEY`. The seo-engine pings after each publish and in the nightly catch-up.
5. Submit `https://yoursite/sitemap.xml` in Google Search Console and Bing Webmaster Tools once.
   The seo-engine measure job resubmits it nightly.
6. Check: `curl -s https://yoursite/llms.txt | head`, Google's Rich Results Test on a post with a FAQ,
   `curl -s https://yoursite/sitemap.xml | grep -c '<loc>'` against your post count.

Other frameworks: the same logic works anywhere. Generate the text on request (or at build plus a
revalidate) from the WordPress REST API, never from a hand-maintained list.

## Getting cited, not just crawled

- Every post needs an `<h2>Frequently asked questions</h2>` with `<h3>` questions and short, direct
  answers. That is what FAQPage schema and answer engines lift. The seo-engine writer adds one.
- Put the answer in the first paragraph. Clear, extractable sentences beat clever intros.
- Keep facts consistent across the site, llms.txt and your docs. Contradictions get you dropped.

## Measuring AI traffic (PostHog)

AI referrals arrive as a referring domain (`chatgpt.com`, `perplexity.ai`, `claude.ai`,
`gemini.google.com`, `copilot.microsoft.com`) or `utm_source=chatgpt.com`. This kit's
`lib/posthog-sources.ts` already groups them into ChatGPT / Claude / Perplexity / Gemini / Copilot
families for the Traffic sources card. The seo-engine `measure.mjs` stores AI visitors per post
(`seo_metrics_daily.ai_visitors`) and uses them in the cluster weights.

## Examples

Example 1: Add llms.txt and a sitemap
User says: "Add llms.txt and a sitemap to my Next.js site so ChatGPT can read my blog"
Actions:
1. Copy the templates as in Install steps 1 and 2, set `SITE_URL` and `BLOG_ORIGIN`.
2. Run the checks in Install step 6.
Result: `/llms.txt`, `/llms-full.txt`, `/sitemap.xml` and `/robots.txt` generated live from WordPress posts.

Example 2: FAQ rich results
User says: "Why doesn't Google show my FAQ in search results?"
Actions:
1. Confirm the post has an `<h2>Frequently asked questions</h2>` with `<h3>` questions.
2. Render `articleJsonLd(post)` on the article page and test it in Google's Rich Results Test.
Result: BlogPosting + FAQPage JSON-LD on every post.

Example 3: Is AI sending traffic
User says: "Are ChatGPT or Perplexity sending me visitors?"
Actions:
1. Check the ChatGPT / Claude / Perplexity / Gemini / Copilot families in the Traffic sources card.
2. If the seo-engine runs, query `seo_metrics_daily.ai_visitors` per post.
Result: AI referral visitors per day and per post.

## Troubleshooting

Symptom: sitemap or llms.txt is empty after a deploy
Cause: A build-time render snapshotted an **empty** post list because WordPress was only reachable on a private network during the build.
Solution: Fall back to the public origin, or render on request with `revalidate`.

Symptom: the app cannot read posts, or the files list nothing
Cause: `/blog/wp-json` is blocked at the proxy.
Solution: Don't block `/blog/wp-json` if your app reads posts through it.

Symptom: titles show `&#8217;` and similar in JSON-LD or llms.txt
Cause: WordPress returns HTML entities.
Solution: Decode WordPress entities before putting titles into JSON-LD or llms.txt.

Symptom: sitemap `<loc>` count does not match your post count
Cause: Stale build-time render or pagination not followed.
Solution: Render on request from the WordPress REST API, never from a hand-maintained list.
