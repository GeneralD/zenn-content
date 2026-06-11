# zenn-content

Content repository for the user's tech blog on Zenn (zenn.dev), linked via
Zenn's GitHub integration — pushing to the default branch publishes articles.

- Stack: plain Markdown managed with Zenn CLI conventions (no package.json
  committed; `node_modules` is gitignored, so `npx zenn` is used ad hoc).
- Status: active — articles are added by the `write-blog` skill workflow.
- Layout:
  - `articles/` — one Markdown file per article, Zenn frontmatter
    (`title`, `emoji`, `type`, `topics`, `published`).
  - `images/<article-slug>/` — per-article images (hero.png etc.),
    referenced as `/images/<slug>/...` in article bodies.
  - `books/` — empty (no Zenn books yet).
- Articles are written in Japanese.
- Preview locally with `npx zenn preview` (requires zenn-cli; not pinned
  in-repo). Publishing = commit + push; no build step or CI.
