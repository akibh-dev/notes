---
# ─────────────────────────────────────────────────────────────────────────
# POST TEMPLATE — copy to  content/<year>/<slug>/index.md  (images go in
# the same folder). The blog (blog.akibh.dev, repo akibh-dev/akiblog)
# validates this frontmatter at build time: a bad field fails the deploy
# with the file + field name, so nothing broken goes live. Schema:
# akiblog docs/ui-guide/content.config.ts. Also valid for Jekyll (layout).
# This folder starts with "_", so neither Jekyll nor the blog publishes it.
# ─────────────────────────────────────────────────────────────────────────
layout: post                      # Jekyll only — the blog ignores it
title: "Edge Caching Without Tears"                       # required, ≤ 90 chars
description: "How I cache Astro pages on Cloudflare Pages: headers, purge strategy, and the two mistakes that cost me a weekend."  # required, ≤ 160 (aim 50+)
date: 2026-10-20                  # required, YYYY-MM-DD
updated: 2026-10-22               # optional → "Updated" + dateModified
slug: edge-caching-without-tears  # optional, defaults to the folder name; never change after publishing
type: guide                       # guide | note | essay | reference   (default: note)
tags: ["cloudflare", "caching", "astro"]                  # required, 1–6, lowercase
keywords: ["cloudflare pages cache", "cache-control"]     # optional, defaults to tags
cover: ./cover.jpg                # optional, ≥ 1200×630, next to this file
coverAlt: "Cache layers drawn as stacked terminal windows" # required when cover is set
series: "Edge Notes"              # optional
featured: false                   # true = pin: first on the blog home + served to the portfolio
ai: none                          # none | assisted | generated   (default: none)
draft: false                      # true → excluded from production build
toc: true                         # default true
comments: true                    # default true (giscus)
# canonical: https://example.com/original   # only if first published elsewhere
---

The opening paragraph is what readers see first. **Don't add a `# H1`.**
The `title` above is the H1, and the renderer strips a body H1 if one slips in.

## Why caching gets weird at the edge

Relative images work on GitHub, Jekyll **and** the blog, and the blog
optimizes them at build time:

![Cache hit ratio after the fix](./hit-ratio.png)

> [!NOTE]
> GitHub-style alerts render as callouts on GitHub and on the blog.

> [!TIP]
> Also available: `[!IMPORTANT]`, `[!WARNING]`, `[!CAUTION]`.

### Code with a file tab

```ts title="src/middleware.ts"
export const onRequest = async (context, next) => {
  const res = await next();
  res.headers.set("Cache-Control", "public, max-age=0, must-revalidate");
  return res;
};
```

### Tables

| Asset | Cache-Control | Why |
| :--- | :--- | :--- |
| `/_astro/*` | `max-age=31536000, immutable` | content-hashed names |
| HTML | `max-age=0, must-revalidate` | deploys show up at once |

## Takeaways

- Short, scannable points.
- Press <kbd>Ctrl</kbd> + <kbd>F5</kbd> to bypass the browser cache.

A footnote for the curious[^1].

[^1]: Footnotes render at the end of the article with back-links.
