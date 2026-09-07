# CLAUDE.md — rules of this repository

sierpe-web is the website for [Sierpe](https://github.com/zkCaleb-dev/sierpe),
a self-hosted Stellar indexer. Built with Hugo: **no theme, hand-written
layouts, no JavaScript**. Deployed on Vercel (framework preset: Hugo).

The indexer repo lives next door at `../sierpe` (same `Sierpe/Code/` folder).
Its `CHANGELOG.md` and `docs/` are the source of truth for anything this site
claims about the product — never invent behaviour the indexer repo does not
document.

## Structure

```
content/docs/       product documentation (one page per topic)
content/news/       release posts + use-case posts
content/why-sierpe.md
layouts/            hand-written; baseof, home, page, section + llms outputs
hugo.toml           site config; params.version/released/image track releases
```

## Hard rules

1. **The site moves in lockstep with sierpe releases.** Every indexer release
   gets a site cut on a `feat/site-X.Y.Z-cut` branch: version params, a news
   post, and any docs pages the release touches. Procedure:
   `.claude/skills/release-cut`.
2. **`/llms.txt` and `/llms-full.txt` render from the SAME source files as the
   human pages** (llmstxt.org, wired as Hugo output formats in `hugo.toml` +
   `layouts/home.llms*.txt`). Never fork content into an AI-only copy; edit the
   one source and both audiences stay in sync.
3. **News posts are prose that tells a story, not changelog pastes.** Read two
   or three earlier posts in `content/news/` for the voice before writing one.
4. **No JavaScript, no theme.** Additions are HTML + CSS in `layouts/` and
   `assets/`. If a feature seems to need JS, raise it first.
5. `params.version`, `params.released` and `params.image` in `hugo.toml` must
   always match the latest sierpe release — they render on the site.

## Verification

`hugo build` must pass clean **with and without drafts** (`hugo build` and
`hugo build -D`) before any commit is proposed. Eyeball `hugo server` output
for layout changes.

## Git conventions

Same as the sierpe repo:

- Conventional commits, imperative mood; no quotes, apostrophes or backticks
  in commit messages.
- **No AI co-authorship trailers of any kind.** Hard project rule.
- Branch from `origin/main` (fetch first); land through a GitHub PR + merge by
  the maintainer.
- Closing package (commit / PR / merge texts) comes from the `caleb-workflow`
  plugin (`finish-work`, `workflow-rules`).

## Language

Content, code, and commit messages in **English** (public OSS project).
Conversation with the maintainer may be in Spanish.
