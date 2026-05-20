# v3 gap audit — Podcast

> Source: `/home/claw/work/podcast-app/` (5,915 LOC)
> v1 spec: `/home/claw/work/reeloo-design/v1/products/podcast.md` (234 lines)
> Impl: `/home/claw/work/reeloo-v2/components/routes/podcast.tsx` (347 lines)
> Headline: v1 spec is ~80% complete. **The big gap is the impl, not the spec.** The impl is a 347-line shell that elides ~2,000 lines of player craft that v1 actually documented.

---

## 1. Feature inventory (real app)

### Surface: catalog (`/`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 1 | Language-tab filter with `Accept-Language` server default | `app/page.tsx:82-97` | composed |
| 2 | Curated tile grid (12 demos) reading `data/podcast-recommendations.json` + `data/demos/<slug>.json` | `app/page.tsx:103-123` | atomic |
| 3 | Hard-coded cobalt-dark page palette (`linear-gradient(180deg, #0f1419, #1a2332)` page-only, NOT the theme system) | `app/page.tsx:185-245` | atomic |
| 4 | `<ProgressModal>` on tile click — animated waiting modal with phased copy | `components/ProgressModal.tsx` (302 lines) | composed |
| 5 | `<RecentRow>` — localStorage-backed "recently opened" strip with cover thumb + slug deeplink | `components/RecentRow.tsx` (142 lines) | composed |
| 6 | `pushRecent(slug, payload)` writer with capped queue in `reeloo_recent_podcasts` localStorage | `RecentRow.tsx`, used `CatalogClient.tsx:64` | atomic |
| 7 | Disabled-tile rendering — recs with `disabled: true` or cache-miss are visually distinct | `app/page.tsx:116-123` | atomic |

### Surface: player (`/podcast/<slug>`)

The heart of the app. `KarottenPlayerV3` = **2,442 lines**. Every item below cites a line range or symbol.

| # | Feature | Source | Tier |
|---|---|---|---|
| 8 | Vertical focus-pin column pinned at **42% viewport height** with `translateY(offsetTop)` transform (NOT `getBoundingClientRect` — that was a bug) | `KarottenPlayerV3.tsx:579-606`, `:827` | composed |
| 9 | Neighbouring segments fade to 18% opacity; ±1 to 42% — typographic depth-of-field | header comment `:25-32` | atomic |
| 10 | Active sentence rendered in **IBM Plex Serif 34px/700** (a deliberate typographic choice for listening, not reading) | `KarottenPlayerV3.tsx:25`, font in `app/layout.tsx:8-32` | atomic |
| 11 | Word-level karaoke: each word is a `<span data-kind=…>` driven by `findActive(segments, ms)` binary search | `KarottenPlayerV3.tsx:128-143`, `:874` | composed |
| 12 | Click any word → `seekToWord(i, wi)` seeks the `<audio>` to that word's `start_ms` | `:657-695`, `:880` | atomic |
| 13 | **Annotation underline kinds**: `vocab`/`grammar`/`cultural`/`idiom` rendered with distinct Solarized hues `#268bd2 / #859900 / #cb4b16 / #6c71c4` | `KarottenPlayerV3.tsx:1474-1477` | atomic |
| 14 | **Annotation rail** (360px right column) with collapsible toggle; click rail entry → `findAnnotationWordIdx(seg, phrase)` seeks to that word | `KarottenPlayerV3.tsx:179`, `:807-810` | composed |
| 15 | Rail underline-color legend rendered inline with each entry | `:1474-1477` + rail render block | atomic |
| 16 | **Pre-teach vocab modal** — fires on first open if ≥10 annotations across segments; one-time `reeloo_podcast_vocab_seen_v1:<slug>` flag | `:94`, `:407-428`, `PreTeachVocabModal` at `:1176` | multi-step flow |
| 17 | **Tweaks panel** (dev-style, gated behind a gear icon): translation visibility (zh/en/both/none), Hochdeutsch toggle, theme (cream/night), auto-open rail | `TweaksPanel` at `:1336-1455`, header `:11-23` | composed |
| 18 | **Cobalt night theme** (`data-theme` swap, `:0f1419 → #1a2332` gradient) toggled inside the player | `:1474`, `:696`, `tweaks.theme` plumbing | atomic |
| 19 | Playback speed ladder `[0.6, 0.75, 1.0]` persisted to `reeloo_podcast_speed_v1` localStorage; ↑/↓ arrow ladder | `:92-93`, `:267-305`, `:540-555` | atomic |
| 20 | Keyboard shortcuts: R = replay current sentence; Space = play/pause; ←/→ = seek by N ms; ↑/↓ = speed ladder | `:471-569` | composed |
| 21 | Seek overlay (`role=status, aria-live=polite`): shows effective jump magnitude for ~600ms after last seek | `:477`, `:742-746` | atomic |
| 22 | MediaSession API: `seekbackward`, `seekforward`, `seekto` action handlers for lockscreen / OS media controls | `:356-365` | atomic |
| 23 | **`PodcastPlayerV1` legacy theatre-style player** (opt-in via `?layout=v1`) | `PodcastPlayerV1.tsx`, gated `PlayerSwitcher.tsx` | composed |
| 24 | `PlayerSwitcher` writes Recent on mount, persists layout choice | `PlayerSwitcher.tsx` (379 lines) | atomic |
| 25 | Header breadcrumb with live language family chip (EN/DE/中文/日本語), active-language `class="active"` | `:748-790` | atomic |
| 26 | "kt-locale-chip" inline locale display (NOT the global LanguageTabs — player-local) | `:768-790` | atomic |
| 27 | Audio `<audio preload="metadata">` mount with explicit `data-testid="kt-audio"`; fallback `kt-audio-unavailable` state when src missing | `:720-725` | atomic |
| 28 | Time formatter `fmt(ms)` for player chrome | `:121-126` | atomic |
| 29 | `?tweaks=1` URL param to auto-open tweaks panel (deep-link for QA) | `:260` | atomic |

### Surface: video variant (`KarottenVideoPlayer`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 30 | Video-backed alternate player (955 lines) used when a demo carries a video URL — same focus-pin behaviour, video instead of audio | `components/KarottenVideoPlayer.tsx` | composed |

### Surface: wizard (`/wizard`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 31 | 78-line on-demand generation form — target ∈ {de,en,ja}, topic chips, native language → POST `/api/wizard` → redirect to `/podcast/<slug>` on completion | `app/wizard/page.tsx`, `app/api/wizard/route.ts:7-22` | multi-step flow |
| 32 | 502-handling: shows upstream error string when depot wizard endpoint fails | `app/api/wizard/route.ts:27` | atomic |

### Surface: API + infra

| # | Feature | Source | Tier |
|---|---|---|---|
| 33 | `GET /api/audio/<hash>.mp3` — depot proxy with HTTP Range support; gates hash on `/^[a-f0-9]{32}\.mp3$/i` regex | `app/api/audio/[hash]/route.ts:23-26` | atomic |
| 34 | `POST /api/tts` — pass-through to depot for per-word "hear it"; 30s timeout; 503 if no `DEPOT_APP_TOKEN` env | `app/api/tts/route.ts:25-51` | atomic |
| 35 | Slug regex gate `/^[a-z0-9][a-z0-9-]{0,79}$/` on player route — 404 if bad | `app/podcast/[slug]/page.tsx:21` | atomic |
| 36 | `form` validation — 404 if demo JSON has `form` outside `{podcast, podcast-listening}` | `:59-61` | atomic |
| 37 | Legacy `slang` annotation kind normalized to `cultural` at render time | `lib/types.ts:20` | atomic |

---

## 2. What v1 spec captured (it's most of it)

v1 spec at `v1/products/podcast.md` correctly identified:
- Identity + tagline (`§1`, lines 12-16)
- Full tech stack (`§2`, lines 18-33) including the inline-CSS-in-JS player styling and absence of `tailwind.config.*`
- Full route map (`§3`, lines 37-57) — all 4 pages + 3 API routes
- Data model (`§4`, lines 61-98) — `LessonData`, `PodcastSegment`, `PodcastAnnotation`, the 12 demos
- All 5 user flows including the **focus-pin maths** (lines 100-138)
- Design tokens (`§6`, lines 145-186): the full Solarized palette, the 42% pin, the 360px rail, the cobalt gradient, the 460ms ease-out
- Components inventory (`§7`, lines 188-203) — all 8 components with line counts
- External integrations (`§8`, lines 205-217)
- "What it does that others can't absorb" (`§10`, lines 226-227) — the focus-pin teleprompter argument

v1 spec went deeper than the v2 brief gave it credit for.

---

## 3. What v1 spec MISSED

Even at 234 lines, a few features didn't make the spec:

- **PreTeachVocabModal**: v1 §5 Flow D mentions the modal exists, but the spec doesn't carry the **`VOCAB_PRETEACH_COUNT = 10` threshold** or the per-slug localStorage key naming convention. Both matter for design — the threshold determines whether the modal fires for a 3-vocab demo or only for vocab-dense ones, and the key namespace defines whether vocab "seen" survives a slug rename. (`KarottenPlayerV3.tsx:95`, `:418-428`)
- **Tweaks panel `?tweaks=1` deep-link** and the four sub-controls. v1 §7 mentions "Tweaks panel" but doesn't enumerate the four controls (zh/en/both/none translation visibility, Hochdeutsch toggle, theme, autoOpenRail) — design needs to know whether this surface stays "dev-only" or graduates to user-visible Settings. (`TweaksPanel:1336-1455`)
- **MediaSession API integration** — lockscreen play/pause/seek. v1 doesn't mention; design needs to specify the title/artwork payload. (`:356-365`)
- **Seek overlay micro-affordance** — the 600ms accessibility-live overlay that surfaces effective jump magnitude after keyboard seeks. Distinct from the scrub UI. (`:477`, `:742-746`)
- **`PodcastPlayerV1` legacy fallback** — v1 mentions it exists but doesn't define when the user should see it. Is it a quality-degraded mode? A nostalgia link? Design needs a position.
- **`KarottenVideoPlayer` (955 lines)** — v1 §7 mentions one line. This is a whole second player variant; design needs to know it's a parallel surface, not a layout switch.
- **Stage-clamp behaviour** for short segments — `findActive` binary search behaviour at the head + tail of the audio track. Edge cases worth specifying.
- **The "kt3-app" / "kt-root" classname taxonomy** — the player namespaces its own CSS to avoid collision with the global shell. v1 doesn't model how a shared shell (v2 §8) coexists with player-owned styles. The 1,000-line inline `const CSS = \`…\`` block (`:1456`) is a real architectural decision that v2 will fight.

---

## 4. What the impl skipped (the BIG gap)

Compared against `/home/claw/work/reeloo-v2/components/routes/podcast.tsx` (347 lines):

| v1-spec'd feature | In impl? |
|---|---|
| Focus-pin 42% viewport math | NO — uses static `.rp-focus` block, no transform |
| Plex Serif 34px/700 active line | NO — uses `var(--font-serif)` but not the 34/700 spec |
| Per-word `<span>` karaoke with binary-search active-word resolution | PARTIAL — words are spans but no `<audio>` element, no `findActive`, no `seekToWord` |
| Annotation rail (360px collapsible) | NO — uses pop-up "rp-annotation" inline card; no rail toggle |
| Pre-teach vocab modal | NO |
| Tweaks panel (4 controls) | NO |
| Cobalt-dark night theme inside the player | NO — page is hard-coded dark; no toggle |
| Speed ladder `[0.6, 0.75, 1.0]` + localStorage | NO — uses 3-tier `[1.0, 0.85, 0.7]` (different ladder!) + no persistence |
| Keyboard shortcuts (R, Space, arrows) | NO |
| MediaSession lockscreen integration | NO |
| Seek overlay accessibility affordance | NO |
| `<RecentRow>` with localStorage | NO — `PODCAST_RECENT` is hard-coded |
| `<ProgressModal>` on tile click | NO — `nav()` goes straight |
| `<PodcastPlayerV1>` legacy fallback | NO |
| `<KarottenVideoPlayer>` video variant | NO |
| `/wizard` on-demand generation | NO |
| Annotation underline color taxonomy (4 kinds × 4 hues) | NO — one `rp-annotation` card, kind not used |
| Audio `<audio>` element + timeupdate polling | NO — `playing` is a `useState` boolean only |
| Recent-podcasts deep-link via `?welcome=1` | NO |

**Score: the impl ports the visual shell but ships 0 of the 25+ behaviours that make this player the player.**

---

## 5. What's actually pretty good

- The Solarized annotation palette is preserved in the impl (orange/blue/green) even if only one kind is rendered.
- The "Make a video" affordance is present, stable test-id `podcast-player:make-video-button`, ready to wire to `export.reeloo.ai`.
- The save-to-SRS affordance is correctly wired to `useShell().saveWord(word)` and the saved-state styling flows through.
- The catalog tile grid layout has the right semantic structure (cover + body + meta + level chip) and is straightforwardly extendable.
- `data-testid="podcast-player:focus-pin-column"` exists — the contract is honored even if the contents are wrong.
