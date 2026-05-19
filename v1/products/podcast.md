# Product Spec — Podcast (`podcast.reeloo.ai`)

> Source path: `/home/claw/work/podcast-app/`
> GitHub: `rcdroid/podcast-app` (private)
> Live: <https://podcast.reeloo.ai> · status 200 at 2026-05-19 22:32 UTC
> Dev port: 3002 (`package.json:5`)
> Screenshots: `v1/screens/podcast-catalog.png`, `v1/screens/podcast-player-karotten.png`, `v1/screens/podcast-wizard.png`

---

## 1. Identity

- **Name:** Reeloo Podcast.
- **Tagline (synthesised):** *Listen along to language podcasts — word-level karaoke + bilingual captions, curated across DE / de-CH / EN / JA / zh-TW.*
- **The one user it serves today:** the operator (an immigrant Mandarin speaker living in Switzerland, learning German B1/B2 + Japanese N5 — see memory `user_profile.md`). Concretely: someone who already listens to *Easy German*, *Slow German*, *NHK Newsline*, *Schnabelweid*, and wants the audio annotated.
- **The user goal it advances:** passive-time language exposure with on-tap comprehension. Audio runs unattended; click a word and the player surfaces a vocab gloss without you tabbing away.

## 2. Tech stack

Reading `/home/claw/work/podcast-app/package.json`:

| Concern | Choice | Source |
|---|---|---|
| Framework | Next.js 16.2.2, App Router | `package.json:12` |
| React | 19.2.4 | `package.json:13` |
| Styling | Tailwind v4 (`@tailwindcss/postcss`), but **most player styles are an inline `const CSS = ...` string** (`KarottenPlayerV3.tsx:1456`). Zero `tailwind.config.*` file. | `package.json:22` |
| Fonts | `next/font/google`: IBM Plex Sans / Serif / Mono + Noto Sans TC, exposed as `--kt3-font-*` CSS variables. | `app/layout.tsx:2,8–32,45` |
| Auth | **None.** No `next-auth`, no `@auth/core`. Pure read-only public surface. | `package.json` (absent) |
| Storage | **None.** No DB; lessons are JSON files in `data/demos/<slug>.json`. localStorage is the only client-side state. | `app/page.tsx:11–13`, `app/podcast/[slug]/page.tsx:11–13`, `KarottenPlayerV3.tsx:93–95` |
| Test | Vitest + Playwright | `package.json:24–26` |
| Deployment | Vercel (`.vercel/project.json` present) | `.vercel/project.json` |

**How it differs from siblings:** the only app of the five with NO database, NO auth, NO multi-route surface. It's a thin "demo viewer" — three pages (`/`, `/podcast/[slug]`, `/wizard`) and three API routes (`/api/wizard`, `/api/tts`, `/api/audio/[hash]`), all of which are proxies to depot.

## 3. Route map

```
app/
├── layout.tsx                       # global font setup; 22 lines, no chrome
├── page.tsx                         # GET /  — language-filtered catalog grid
├── globals.css                      # 13-line reset; player owns its own styles
├── wizard/page.tsx                  # GET /wizard — on-demand generation form
├── podcast/[slug]/page.tsx          # GET /podcast/<slug> — public player
└── api/
    ├── wizard/route.ts              # POST /api/wizard      → depot /api/wizard/generate
    ├── tts/route.ts                 # POST /api/tts         → depot /api/tts
    └── audio/[hash]/route.ts        # GET  /api/audio/<hash>.mp3 — depot audio proxy with Range
```

Per-route one-liners:

- `GET /` (`app/page.tsx:103`) — server component, reads `data/podcast-recommendations.json` + scans `data/demos/<slug>.json` for cache hits, picks default language via `Accept-Language` header (`app/page.tsx:82–97`). Renders a hero, language tabs, and the curated grid.
- `GET /podcast/<slug>` (`app/podcast/[slug]/page.tsx:94`) — slug-safe regex gate `/^[a-z0-9][a-z0-9-]{0,79}$/` (`:21`), reads the demo JSON, 404s if `form` is not `podcast | podcast-listening` (`:59–61`). Default layout is `v3`; `?layout=v1` opt-in.
- `GET /wizard` (`app/wizard/page.tsx`) — 78-line client form: target ∈ {de,en,ja}, topic chip, native language → POSTs `/api/wizard` and redirects to `/podcast/<slug>` on completion.
- `POST /api/wizard` (`app/api/wizard/route.ts:7`) — validates `target ∈ {de,en,ja}` (`:14`), forwards to `${DEPOT_BASE_URL}/api/wizard/generate` with `X-App-Token` header (`lib/depot.ts:14–25`), 120s timeout.
- `POST /api/tts` (`app/api/tts/route.ts:25`) — proxies `{text, language}` to `${DEPOT_BASE_URL}/api/tts`; 503 if no `DEPOT_APP_TOKEN` env (`:41`); 30s timeout.
- `GET /api/audio/<hash>.mp3` (`app/api/audio/[hash]/route.ts:23`) — proxies to `${DEPOT_URL}/api/podcast-audio/<hash>.mp3` with HTTP Range support; gates hash on `/^[a-f0-9]{32}\.mp3$/i` (`:26`).

## 4. Data model

There is no database. Data lives in **three** JSON files / file trees:

- `data/podcast-recommendations.json` — `{ languages: [{code,label,flag}], recommendations: Record<lang, RecRaw[]> }`. RecRaw = `{ title, host, description, url, slug?, emoji, gradient, minutes?, cefr?, disabled? }` (`app/page.tsx:39–50`).
- `data/demos/<slug>.json` — one file per cached podcast episode. Shape defined by `LessonData` in `lib/types.ts:65–80`:

```ts
interface LessonData {
  title: string;
  form?: string;                  // "podcast" | "podcast-listening"
  audioUrl?: string;              // (or legacy audio_url)
  audioDurationSec?: number;
  language?: string;              // "de" | "de-CH" | "en" | "ja" | "zh-TW"
  segments?: PodcastSegment[];
  sourceUrl?: string;             // (or legacy source_url)
}
interface PodcastSegment {
  text: string; hochdeutsch?: string; english?: string; chinese?: string;
  translation?: string; start_ms: number; end_ms: number;
  words: { text; start_ms; end_ms }[];
  annotations?: PodcastAnnotation[];
}
interface PodcastAnnotation {
  text: string;
  kind: "vocab" | "grammar" | "cultural" | "idiom";
  explanation_zh: string;
  pos?: string; gender?: "m"|"f"|"n"|"pl"; cefr?: "A1"-"C2";
}
```

Twelve demos shipped at the time of inspection (`data/demos/`, all >40 KB JSON):
`apprenticeships`, `ask-your-doctor-about`, `der-eurovision-song-contest-esc-sg-319`, `karotten`, `learn-japanese-at-the-bar`, `nhk-newsline-trump-xi`, `schnabelweid`, `slow-german-esc`, `us-history-100-objects`, `10-0513-10`, `2026-05-13-8-220`, `8-at-home-kitchen-refrigerator-microwave`.

- Client-only state: `localStorage`
  - `reeloo_podcast_speed_v1` — playback speed in {0.6, 0.75, 1.0} (`KarottenPlayerV3.tsx:93,92`)
  - `reeloo_podcast_vocab_seen_v1:<slug>` — pre-teach vocab modal "seen" flag (`KarottenPlayerV3.tsx:94`)
  - `reeloo_recent_podcasts` — Recent row (`components/RecentRow.tsx` + `PlayerSwitcher.tsx:28`)

**Audio + TTS storage:** lives upstream in depot. The standalone app never sees the raw `gs://` bucket.

## 5. Key user flows

### Flow A — Browse catalog → pick a tile → land on player

1. `GET /` → server reads recs + demos and filters out `disabled` and cache-miss entries (`app/page.tsx:116–123`).
2. User hits a language tab. `CatalogClient.tsx:101` `Link` rewrites the query string `/?lang=<code>`; server re-renders.
3. User clicks a `<button data-testid="podcast-card">` (`CatalogClient.tsx:130`). `onPick` (`:56`) opens a `ProgressModal` (cache-hit = fast). `onComplete` (`:64`) pushes to localStorage Recent row (`pushRecent`, `RecentRow.tsx`) and `router.push("/podcast/<slug>?welcome=1")`.
4. `app/podcast/[slug]/page.tsx:94` resolves the demo, renders `PlayerSwitcher` (`:112`) which mounts `PodcastPlayerV3` by default.
5. Player normalizes the legacy `slang` annotation kind into `cultural` (`lib/types.ts:20`) before render.

Exit state: audio paused at t=0, focus-pin column centered on segment 1.

### Flow B — Play segment, jump to a word

1. `<audio>` element mounts with `preload="metadata"` (`KarottenPlayerV3.tsx:717`).
2. User taps play. `timeupdate` polls drive `findActive(segments, ms)` (`KarottenPlayerV3.tsx:128–143`) — binary search → active segment index.
3. The focus column transforms via `translateY(${colTransform}px)` (`:827`) using `seg.offsetTop` — deliberately NOT `getBoundingClientRect` (`:11`) to stay immune to in-flight transitions.
4. Active segment's words are rendered as spans with `data-kind` matching the annotation `kind` (`:874`). Click → `seekToWord(i, wi)` (`:880`) seeks the `<audio>`.
5. R key (replay sentence) is wired at `:898`.

### Flow C — Open the annotation rail

1. User clicks `<button class="kt3-rail-toggle">` (`KarottenPlayerV3.tsx:807`). State `railOpen` toggles.
2. Each segment's `annotations[]` map to rail entries; underline colors come from CSS vars `--anno-vocab #268bd2 / --anno-grammar #859900 / --anno-cultural #cb4b16 / --anno-idiom #6c71c4` (`KarottenPlayerV3.tsx:1474–1477`).
3. Clicking a rail entry calls `findAnnotationWordIdx(seg, phrase)` (`:179`) and scrolls/seeks.

### Flow D — Pre-teach vocab modal

1. On player mount, read `reeloo_podcast_vocab_seen_v1:<slug>` (`KarottenPlayerV3.tsx:94`).
2. If unseen AND there are ≥10 annotations across all segments (`VOCAB_PRETEACH_COUNT = 10`, `:95`), open a modal listing the first 10 vocab items.
3. On dismiss → write the slug flag.

### Flow E — Hear a word ("hear it")

1. User clicks a word's TTS chip in the rail or active line.
2. Fetch `POST /api/tts` with `{ text, language }`.
3. The route forwards to depot `${DEPOT_BASE_URL}/api/tts` with `X-App-Token` (`app/api/tts/route.ts:51–58`). 30s timeout.
4. Response `{ audio?: string, audio_url?: string }` is played by an `Audio` element.

### Flow F — Wizard (on-demand generation)

1. `/wizard` form → POST `/api/wizard` with `{ target, chips, native }`.
2. Server validates inputs (`route.ts:14–22`), proxies to depot's wizard endpoint (120s timeout, `lib/depot.ts:25`).
3. On success: redirect to `/podcast/<slug>`. On 502: surface upstream error string (`route.ts:27`).

## 6. Design tokens used

Pulled from `components/KarottenPlayerV3.tsx:1461–1484` (the canonical block) + `app/globals.css`:

| Token | Value | Role |
|---|---|---|
| `--kt3-bg` | `#fdf6e3` | Solarized base3 — page background (light) |
| `--kt3-bg-card` | `#eee8d5` | base2 — card |
| `--kt3-bg-hover` | `#e8e3d0` | hover state |
| `--kt3-text` | `#586e75` | base01 — body text |
| `--kt3-text-heading` | `#073642` | base02 — headings |
| `--kt3-text-muted` | `#93a1a1` | base1 — muted |
| `--kt3-border` | `#e0dccc` | warm border |
| `--kt3-orange` | `#cb4b16` | Solarized orange (CTA / primary) |
| `--kt3-blue` | `#268bd2` | Solarized blue |
| `--kt3-green` | `#859900` | Solarized green |
| `--kt3-red` | `#dc322f` | Solarized red |
| `--kt3-cobalt` | `#0f1419` | Night theme background |
| `--kt3-cobalt-2` | `#1a2332` | Night theme card |
| `--anno-vocab` | `#268bd2` | Annotation underline (blue) |
| `--anno-grammar` | `#859900` | (green) |
| `--anno-cultural` | `#cb4b16` | (orange) |
| `--anno-idiom` | `#6c71c4` | Solarized violet |
| `--kt3-dur-fast` | `120ms` | micro-interaction |
| `--kt3-dur-med` | `240ms` | column slide |
| `--kt3-ease-out` | `cubic-bezier(0.22, 1, 0.36, 1)` | standard easing |

Fonts (`app/layout.tsx:8–32`): **IBM Plex Sans** (body), **IBM Plex Serif** (active-line focus typography — 34px / 700, see `KarottenPlayerV3.tsx:25`), **IBM Plex Mono** (numerics + breadcrumbs), **Noto Sans TC** (Traditional Chinese; `preload: false`).

Pixel targets called out in `KarottenPlayerV3.tsx:25–32`:
- Active segment: Plex Serif 34px / 700, pinned at **42% viewport height**
- Previous segments fade to 18% opacity; near-±1 to 42%
- 460ms ease-out column transform
- 360px right rail; collapses to 0
- Cobalt night theme `#0f1419 → #1a2332` linear-gradient

The catalog page (`/`) hard-codes its own dark palette inline in `app/page.tsx:185–245`:
- bg: `linear-gradient(180deg, #0f1419 0%, #1a2332 100%)`
- text: `#e8eef5`, lede: `#a8b8c8`, meta: `#6e8aa8`
- card border `rgba(255,255,255,0.08)`, active language pill `rgba(109,180,255,0.32)`

There is no tailwind.config.* file.

## 7. Components inventory

`components/`:

- `KarottenPlayerV3.tsx` (**2,442 lines** — the heart of the app). The "vertical focus-pin" player. Owns audio, segment column, rail, pre-teach modal, tweaks panel.
- `PodcastPlayerV3.tsx` (77 lines) — thin re-export wrapper around `KarottenPlayerV3` with podcast-friendly defaults (breadcrumb, langChip, footer). Comment at `:8–15` says "NOT a fork: the underlying KarottenPlayerV3 is the source of truth."
- `KarottenVideoPlayer.tsx` (955 lines) — alternate video variant; used in `data/demos/karotten.json` flows.
- `PodcastPlayerV1.tsx` (31 lines) — legacy theatre-style player (`?layout=v1`).
- `PlayerSwitcher.tsx` (379 lines) — wraps V1/V3 + persists layout + writes Recent.
- `CatalogClient.tsx` (315 lines) — client-side language tabs + grid + progress modal.
- `RecentRow.tsx` (142 lines) — localStorage-backed "recently opened" strip on `/`.
- `ProgressModal.tsx` (302 lines) — animated waiting modal shown when a tile is clicked.

**Duplicates with siblings:**
- `KarottenPlayerV3.tsx`, `PodcastPlayerV3.tsx`, `PodcastPlayerV1.tsx`, `CatalogClient.tsx`, `RecentRow.tsx`, `ProgressModal.tsx`, `PlayerSwitcher.tsx` all also live at `/home/claw/work/notes/app/podcast/_components/` (per `find` output earlier). The podcast-app port is the published one.
- `KarottenPlayerV3` lineage: depot `app/demos/karotten-v3/` is the design-canonical version (see header comment `KarottenPlayerV3.tsx:6–13`).

## 8. External integrations

All upstream API calls go to depot:

| Call | File:line | Purpose |
|---|---|---|
| `POST ${DEPOT_BASE_URL}/api/wizard/generate` | `lib/depot.ts:18` | Wizard on-demand generation. `X-App-Token` header. |
| `POST ${DEPOT_BASE_URL}/api/tts` | `app/api/tts/route.ts:51` | Per-word "hear it" TTS. Pass-through. |
| `GET ${DEPOT_URL}/api/podcast-audio/<hash>.mp3` | `app/api/audio/[hash]/route.ts` | Audio file proxy with Range support. |

Auth: `X-App-Token` header read from `DEPOT_APP_TOKEN` env. Never exposed to the browser.

There is **no** call to `dictionary.reeloo.ai` from this app — annotation glosses are baked into the demo JSON at build time (depot owns the enrichment pipeline).

## 9. Screenshots

- `v1/screens/podcast-catalog.png` — live <https://podcast.reeloo.ai/> (cobalt dark, language tabs, curated grid)
- `v1/screens/podcast-player-karotten.png` — local `http://localhost:3002/podcast/karotten` (focus-pin column + active Plex Serif line)
- `v1/screens/podcast-wizard.png` — local `/wizard` (Tailwind-default form, intentionally minimal)

## 10. What this app does that others can't reasonably absorb

The **focus-pin player** (`KarottenPlayerV3`) is a specialised visual mode for listening-along, not reading. The current segment is pinned at 42% viewport height in IBM Plex Serif 34px while neighbours fade to 18% opacity — it's a teleprompter for audio comprehension, deliberately distinct from book-app's page reader (where text dominates) and readalong's deck-card layouts (where segments are paged). The 460ms ease-out column transform calibrated to a 0.6-1.0× speech rate is a real piece of craft that took ~7 player iterations to land (per the file header).

## 11. What this app does that overlaps with others

- The catalog grid + ProgressModal duplicates work in `readalong/.../LibraryView` and `notes/app/podcast/`. **Merge target: a shared `<MediaLibrary>` component.**
- Wizard is a depot facade — the actual generation lives in depot's `/api/wizard/generate`. **Move the form to the unified app's main shell**; podcast-app becomes a CDN-friendly read-only demo viewer (which is what it already is in practice).
- The TTS proxy is duplicated in notes (`app/api/tts`) and readalong (`app/api/synthesize`). **Single endpoint at `dictionary.reeloo.ai` or `audio.reeloo.ai`** would let all five apps drop their proxy boilerplate.
- The annotation-kind union `vocab | grammar | cultural | idiom` overlaps with **book-app's** `idiom | proper_noun | place | archaism | grammar | vocab` (`book-app/lib/annotation-schema.ts:3`). **Merge target: a single `AnnotationKind` enum** in a shared package, with book-app's superset.
