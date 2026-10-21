# My Public Notes & Blog

## Writing a post

Posts live in `content/<year>/<slug>/index.md`, with their images in the same
folder. They're published on [blog.akibh.dev](https://blog.akibh.dev)
as `blog.akibh.dev/<slug>/` (and on this GitHub Pages site).

1. Copy [`_templates/sample.md`](./_templates/sample.md) to `content/<year>/<slug>/index.md`.
2. Fill in the frontmatter: `title`, `description` (≤ 160 chars), `date` and `tags`
   are required. `featured: true` pins the post on the blog home and on the portfolio.
3. Start the body at `##`. The `title` is the page's only `# H1`.
4. Push, then run `bun run deploy:blog` in akiblog to publish.

