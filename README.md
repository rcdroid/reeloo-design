## reeloo-design

Design briefs + system specs for the Reeloo product family. Pointed at by Claude design (claude.ai/code) which reads this repo directly.

### Layout

- `00-v0.5-seed-brief.md` — fast initial pass (5.5K words). Outdated once the v1 lands.
- `v1/` — source-code-reverse-engineered product specs + unified design brief. **This is the canonical source.**
  - `v1/00-unified-brief.md` — top-level brief for Claude design
  - `v1/products/<app>.md` — per-app product spec (one per source app)
  - `v1/design-system/` — tokens, components, themes
  - `v1/screens/` — screenshot references
- `requests/` — open design requests (each is a self-contained markdown file Claude design can read in isolation)
- `outputs/` — Claude design's responses (mockups, tokens.json, etc.) — committed back here

### Why public

Claude design needs GitHub read access. Repo is intentionally code-free — no secrets, no source. Only product/design discussion.

### Source repos (private)

- `rcdroid/depot` — the monorepo. Hosts learn-stations + readalong + flashcards + podcast catalog.
- `rcdroid/podcast-app` — standalone podcast player (separate Next.js app)
- `rcdroid/book-app` — EPUB reader at book.reeloo.ai
- learn-stations — local-only experimental workspace
