---
name: wordpress-blog
description: Connect a WordPress site (self-hosted or WordPress.com) through Creator OS and list, write, update, publish or delete blog articles on your own domain from Claude (MCP tools, REST API or the creatoros CLI). Use when the user wants to connect their blog, write or edit one specific article, repurpose a winning social post into an SEO article, refresh an old post, bulk-fix every post, or publish an approved draft. For the automated daily loop use the seo-engine skill.
license: MIT
compatibility: Requires a Creator OS account and API key with a connected WordPress site (self-hosted with an Application Password, or WordPress.com), used through the Creator OS MCP connector, the REST API, or npx @creatoros/cli.
metadata:
  author: KevBuildsApps
  version: 1.0.0
---

# WordPress blog via Creator OS

Creator OS writes to your WordPress blog for you. The articles live on **your** domain (good for
SEO); your app does not have to run on WordPress. Three ways to drive it, same backend:

| Way | Use it when |
|---|---|
| MCP tools (`blog_list_sites`, `blog_list_articles`, `blog_get_article`, `blog_create_article`, `blog_update_article`, `blog_delete_article`, `blog_get_connect_link`) | Claude / Claude Code / ChatGPT with the Creator OS connector (`https://mcp.creatoros.ca/mcp`) |
| REST: `https://creatoros-production-5658.up.railway.app/v1/...` with `Authorization: Bearer $CREATOROS_API_KEY` | scripts, cron jobs, the seo-engine |
| CLI: `npx @creatoros/cli blog:...` | terminal |

Get the key at https://www.creatoros.ca (Settings > API keys). A key is pinned to one workspace,
so use the key of the workspace that connected the blog.

## Critical

- **Drafts by default** (`isPublished: false`) unless the user said to publish. Ask before publishing or deleting anything when a human is in the loop.
- **Only state product facts you can point to** (docs, pricing page, the seo-engine `facts.json`). No invented stats, customers or quotes. No em dashes. Describe competitors neutrally.
- **One distinct search intent per post.** Publishing 10+ near-identical posts a day gets a site demoted.
- Articles are **HTML** (`bodyHtml`), not Markdown.
- Use the API key of the workspace that connected the blog (a key is pinned to one workspace).

## 1. Connect the site (once)

- **Self-hosted:** WordPress admin > Users > Profile > Application Passwords > create one. Then
  `POST /v1/connect/wordpress` with `{ "site_url": "https://yoursite.com", "username": "...", "application_password": "abcd efgh ..." }`
  or `creatoros blog:connect-selfhosted --site https://yoursite.com --username you --password "abcd efgh ..."`.
- **WordPress.com:** `GET /v1/connect/wordpress` (or MCP `blog_get_connect_link`) returns an
  `auth_url`; send the user there.
- Check it worked: `GET /v1/blog` / `blog_list_sites`.

Tip: run WordPress on a subpath (`yoursite.com/blog`) behind your main app, or on a subdomain, and
render the posts in your own design through the WordPress REST API (`/blog/wp-json/wp/v2/posts`).
The `ai-search-files` skill builds llms.txt, sitemaps and FAQ schema from those same posts.

## 2. Endpoints

```
GET    /v1/blog/articles?limit=50&cursor=...   list (published + drafts): id, handle, title, status, tags
GET    /v1/blog/articles/{id}                  one article with bodyHtml
POST   /v1/blog/articles                       { title, bodyHtml, handle, excerpt, tags[], image{url,altText}, isPublished }
PATCH  /v1/blog/articles/{id}                  any of the above, e.g. { "isPublished": true }
DELETE /v1/blog/articles/{id}
```

- Articles are **HTML** (`bodyHtml`), not Markdown.
- `handle` is the slug. `excerpt` is what most themes use as the meta description (150-160 chars).
- Yoast / Rank Math fields are not set through the API.
- **Drafts by default** (`isPublished: false`) unless the user said to publish. Ask before
  publishing or deleting anything when a human is in the loop.

## 3. Workflows

**Write one article now**
1. `blog_list_articles` so the new post covers new ground and can link to old ones.
2. Pick ONE search intent. The keyword goes in the title (near the start, under 70 chars), the first
   paragraph, one `<h2>` and the handle.
3. Write 1,100 to 1,700 words of HTML: short intro, a worked example (prompt, command, workflow),
   4+ `<h2>` sections, an `<h2>Frequently asked questions</h2>` with 3-5 `<h3>` questions, and a
   Get started section linking the signup page. Link 3-6 existing posts.
4. Only state product facts you can point to (docs, pricing page, the seo-engine `facts.json`).
   No invented stats, customers or quotes. No em dashes. Describe competitors neutrally.
5. `blog_create_article` as a draft, give the user the link, publish on approval with
   `blog_update_article { isPublished: true }`.

**Turn a winning social post into an article**
"Look at my three best-performing posts this month and turn the strongest one into a 1,200-word
SEO article for my blog. Save it as a draft." Pull post analytics (Creator OS `get_post_analytics`
or the PostHog sources card), expand the post's one idea into the article above, link the post.

**Find what to write next (from PostHog)**
Use this kit's tables: which blog posts people **entered** on and then signed up from
(`seo_metrics_daily` once the seo-engine measure job runs, or the Traffic sources card). Write
distinct-intent siblings of the winners: a comparison, a niche version, a how-to, a model pairing.

**Refresh an old post**
`blog_get_article`, update facts, add an FAQ section if missing, keep the handle, `blog_update_article`.
Fresh `modified` dates get re-crawled; ping IndexNow after (see `ai-search-files`).

**Bulk fix (every post)**
Loop `GET /v1/blog/articles` and `PATCH` each. Dry-run first and print what would change.
Example: `.claude/skills/seo-engine/scripts/add-youtube-links.mjs` adds your YouTube link to every post.

## Examples

Example 1: Connect a self-hosted blog
User says: "Connect my WordPress blog at example.com"
Actions:
1. Have the user create an Application Password in WordPress admin.
2. `POST /v1/connect/wordpress` (or `creatoros blog:connect-selfhosted`) with site URL, username and password.
3. Verify with `blog_list_sites`.
Result: The site is listed and ready for articles.

Example 2: Write one article as a draft
User says: "Write a blog post about scheduling Instagram posts with AI"
Actions:
1. `blog_list_articles` to avoid overlap and find posts to link.
2. Write 1,100 to 1,700 words of HTML following "Write one article now".
3. `blog_create_article` with `isPublished: false`, share the link, publish on approval.
Result: Draft on the user's blog, published only after approval.

Example 3: Refresh an old post
User says: "Update my 2024 pricing post with current facts"
Actions:
1. `blog_get_article`, update facts, add an FAQ if missing, keep the handle.
2. `blog_update_article`, then ping IndexNow (see `ai-search-files`).
Result: Refreshed post with a new modified date.

## Troubleshooting

Symptom: `blog_list_sites` / `GET /v1/blog` shows no site
Cause: The WordPress site is not connected to this workspace, or the key belongs to another workspace.
Solution: Connect the site (section 1) with the key of the workspace that owns it.

Symptom: only the first page of articles comes back
Cause: The list endpoint pages with `nextCursor`.
Solution: Loop until `nextCursor` is empty.

Symptom: SEO title / meta fields not set
Cause: Yoast / Rank Math fields are not set through the API.
Solution: Put the meta description in `excerpt` (150-160 chars), which most themes use.

Symptom: ranking drops after a burst of posts
Cause: Near-identical posts or links to pages that don't exist.
Solution: One distinct intent per post; only link pages that exist (your sitemap + the article list).
