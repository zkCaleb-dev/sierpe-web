---
name: release-cut
description: Cut the site for a new sierpe release — version params, news post, affected docs pages. Use when a sierpe vX.Y.Z has been tagged and the site must catch up.
---

# Site release cut

The site follows every sierpe release. One branch, one commit, one PR.

## Inputs

- The version being cut (e.g. `1.7.0`) — ask if not given.
- `../sierpe/CHANGELOG.md`: the section for that version is the factual base.
- `../sierpe/docs/`: source of truth when a feature changes install, config,
  API surface, or metrics.

## Steps

1. **Branch**: `git fetch origin`, then `feat/site-X.Y.Z-cut` from
   `origin/main`.
2. **`hugo.toml` params**: set `version` (`vX.Y.Z`), `released` (tag date,
   `YYYY-MM-DD`), `image` (`ghcr.io/zkcaleb-dev/sierpe:vX.Y.Z`).
3. **News post** `content/news/sierpe-X-Y-Z-released.md`:
   - Front matter: `title` (version + an angle, not just the number), `date`
     (ISO with time, UTC), `summary` (one or two sentences).
   - `<!--more-->` after the opening paragraph.
   - Prose that tells the story of the release — why the change exists, what
     it fixes for a real deployment — not a changelog paste. Read the two most
     recent posts in `content/news/` first and match their voice.
   - Small releases can share a post (see the 1.4.1/1.4.2 post).
4. **Docs pages**: walk the changelog section and update every page in
   `content/docs/` the release touches (new flags → `install.md`/`quickstart.md`,
   new endpoints → `api.md`, new metrics → `observability.md`, etc.). If
   nothing is touched, say so explicitly when handing back.
5. **Verify**: `hugo build` and `hugo build -D` both clean; `hugo server` and
   eyeball the post and the home page version chip.
6. **Hand back**: closing package via `finish-work`. Commit style precedent:
   `feat: the X.Y.Z cut on the site`.

## Traps

- `params.version` appears rendered on the site — forgetting step 2 ships a
  post announcing a version the header contradicts.
- The news `date` orders the feed; a wrong date buries the post.
- `/llms-full.txt` is generated from the same docs sources — do not edit any
  `llms` output by hand.
