## reeloo-design

Design briefs + system specs for the Reeloo product family. Pointed at by Claude design (claude.ai/code) which reads this repo directly.

### Layout

- `00-v0.5-seed-brief.md` — fast initial pass (5.5K words). Historical only.
- `v1/` — source-code-reverse-engineered per-app specs + token system. Source-grounded; stays authoritative for "what each app does today" and "where each hex was sourced".
  - `v1/00-unified-brief.md` — historical baseline; superseded by `v2/06`.
  - `v1/products/<app>.md` — per-app product spec (one per source app). Authoritative.
  - `v1/design-system/` — tokens, components, themes. Authoritative.
  - `v1/screens/` — screenshot references.
- **`v2/` — the canonical brief. Read this first.** Integrates 11 cross-cutting product themes the v1 pass underweighted (signal model, level-mismatch UX, notification center, video export, issue flagging, analytics funnel, avatar primitive, personal surfaces, learn-anything sandbox, test-id contract). v2 supersedes v1 only at the unified-brief layer; per-app + token specs in v1 are unchanged.
  - **`v2/06-revised-unified-brief.md` — the canonical starting point for Claude design.**
  - `v2/00-meta-revision.md` — what changed from v1 and why.
  - `v2/01-signal-and-level-model.md` — signals taxonomy + per-user CEFR model + level-mismatch UX pattern.
  - `v2/02-cross-cutting-systems.md` — notification center, video export pipeline, flag-issue UI, analytics funnel, avatar primitive.
  - `v2/03-personal-surfaces.md` — `/me/{progress,history,map,diary,settings,privacy,notifications}`.
  - `v2/04-learn-anything-sandbox.md` — generative tutorial mode with Pyodide / Sandpack embeds.
  - `v2/05-tdd-design-contract.md` — `data-testid` naming convention and per-component contract.
- `requests/` — open design requests (each is a self-contained markdown file Claude design can read in isolation).
- `outputs/` — Claude design's responses (mockups, tokens.json, etc.) — committed back here.

### Reading order for Claude design

1. `v2/06-revised-unified-brief.md` (canonical)
2. `v2/01..v2/05` (supporting specs in numeric order)
3. `v1/products/*.md` (per-app source-of-truth when you need source-grounded facts)
4. `v1/design-system/proposed-tokens.md` (token sources)
5. `v1/00-unified-brief.md` (historical baseline)

### Why public

Claude design needs GitHub read access. Repo is intentionally code-free — no secrets, no source. Only product/design discussion.

### Source repos (private)

- `rcdroid/depot` — the monorepo. Hosts learn-stations + readalong + flashcards + podcast catalog.
- `rcdroid/podcast-app` — standalone podcast player (separate Next.js app)
- `rcdroid/book-app` — EPUB reader at book.reeloo.ai
- learn-stations — local-only experimental workspace
