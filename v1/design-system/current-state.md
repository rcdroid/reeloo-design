# Design System — Current State (audited from source)

> All claims here have file:line refs. The five apps DO share a "Solarized-derived language-learning" palette in spirit, but each one re-implemented it independently. Five token systems, three font stacks, four theming strategies.

---

## 1. Color palette — ACTUAL

| App | Palette source | Vibe | Hex/HSL sample |
|---|---|---|---|
| **podcast-app** | Hard-coded `const CSS` in `components/KarottenPlayerV3.tsx:1461–1484` | Solarized light + cobalt dark catalog | bg `#fdf6e3`, primary `#cb4b16`, dark catalog `#0f1419 → #1a2332` |
| **learn-stations** | Tailwind v4 `@theme` + 4 themes in `app/globals.css:3–280` | Solarized + Duolingo green + dark + a11y | `--surface #fdf6e3 → #002b36 → #000000`, `--primary #cb4b16 / #58cc02 / #ffaa00` |
| **book-app** | Tailwind v4 `@theme inline` + native `prefers-color-scheme: dark` | Solarized notes on white/black | bg `#ffffff` / `#0a0a0a`, anno colors `#268bd2/#859900/#6c71c4/#cb4b16/#b58900` |
| **readalong** | Tailwind v3 HSL + 7 named themes in `src/app/globals.css:58–200+` | Multi-vibe shadcn pastels | warm cream `40 33% 96%` default; ocean / forest / lavender / midnight / emerald / amethyst / rose variants |
| **notes /practice/flashcards** | Inline CSS-in-JS with `var(--x, #fallback)` fallbacks only | None — undefined `--accent #4f46e5` (indigo), `--card-bg #fff`, `--border #ddd` | indigo `#4f46e5` |

**Conflicts:**

1. **Primary color disagrees.** podcast/book/learn-stations agree on Solarized orange `#cb4b16`; learn-stations PIP-Duo theme overrides to `#58cc02`; readalong default is blue `hsl(220 89% 55%)`. Notes flashcards has indigo `#4f46e5`. → If we keep Solarized orange as primary, 3/5 apps stay put; readalong + notes/practice need re-skin.

2. **Annotation underline colors** agree on **three** values:
   - `--anno-vocab #268bd2` (Solarized blue) — podcast-app `KarottenPlayerV3.tsx:1474`, book-app `app/globals.css:8`
   - `--anno-grammar #859900` (Solarized green) — `KarottenPlayerV3.tsx:1475`, `book-app globals.css:9`
   - `--anno-idiom #6c71c4` (Solarized violet) — `KarottenPlayerV3.tsx:1477`, `book-app globals.css:10`

   But book-app adds two not in podcast: `--anno-place #cb4b16` (orange) and `--anno-archaism #b58900` (yellow, **dotted** underline). Podcast-app calls its orange-anno `--anno-cultural` instead. → Merge: `vocab/grammar/idiom` are stable. Decide one of {`cultural`, `place`} for the orange slot. `archaism` is book-app-only and useful — keep.

3. **Surface color.** podcast/book agree on `#fdf6e3` (Solarized base3) in light mode. learn-stations agrees. Readalong uses warm cream `hsl(40 33% 96%) ≈ #f8f4ee` — close but not exact. Notes flashcards uses `#ffffff` fallback.

4. **Dark mode surface.** podcast `#0f1419 → #1a2332` linear-gradient. learn-stations solarized-dark `#002b36 → #073642`. Readalong default-dark `hsl(222 47% 11%) ≈ #11192a` to midnight `#0a0c12`. Book-app `#0a0a0a`. → No consensus. Recommend Solarized base03 `#002b36` family for consistency with the other tokens, OR podcast's cobalt `#0f1419` if we want a more "audio-app" cinematic feel.

5. **State colors** — only learn-stations has formalised graded states (`--correct-bg/fg/ring`, `--wrong-bg/fg/ring`, `--warning-bg/fg`) with WCAG-AA-verified ratios (`globals.css:21–35`). The other four apps don't ship any.

### POS coloring (learn-stations only)

`learn-stations app/globals.css:45–49`:
- `--pos-place #cb4b16` (Solarized orange)
- `--pos-subject #268bd2` (Solarized blue)
- `--pos-object #d33682` (Solarized magenta)
- `--pos-verb #6b7a00` (darkened Solarized green)
- `--pos-particle #93a1a1` (Solarized base1)

All four content colors AA-readable on `#fdf6e3`. Unique to learn-stations, but the **shape** (POS-tinted sentence rendering) is also used in readalong's `WordHighlightedText.tsx` with different colors and in book-app's grammar annotations without color.

---

## 2. Typography — ACTUAL

| App | Sans | Serif | Mono | CJK | Source |
|---|---|---|---|---|---|
| **podcast-app** | IBM Plex Sans | IBM Plex Serif (active-line, 34px/700) | IBM Plex Mono (numerics, breadcrumbs) | Noto Sans TC (preload:false) | `app/layout.tsx:8–32` |
| **book-app** | IBM Plex Sans (body) | IBM Plex Serif (reading column) | — | — | `app/layout.tsx:2–16` |
| **learn-stations** | Geist Sans (body) + Noto Sans TC | Lora (paid Tiempos sub, headings) + Noto Serif JP | Geist Mono | Noto Sans TC + Noto Serif JP | `app/layout.tsx:2–39` |
| **readalong** | System UI stack | — | source-code-pro/Menlo (code) | `@fontsource/noto-sans-tc` 5.2.9 | `globals.css:9–12`; `package.json:34` |
| **notes /practice/flashcards** | `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif` (inline) | — | — | — | `page.tsx:127` |

**Conflicts:**
- Two distinct sans families ship today: **IBM Plex Sans** (podcast + book) and **Geist Sans** (learn-stations). Readalong uses system UI. **Plex Sans wins** if we want a consistent "language-learning" identity — already the default in two of the most polished surfaces.
- Serif: **IBM Plex Serif** (podcast + book) vs **Lora/Tiempos** (learn-stations). Plex Serif wins — same family as the sans, free, already loaded.
- CJK: **Noto Sans TC** is universal — keep.
- JP serif: only learn-stations needs Noto Serif JP (for furigana over kanji). Keep.

**Sizes called out in source:**

- Podcast-app active segment: **34px / 700 Plex Serif** at 42% viewport height (`KarottenPlayerV3.tsx:27`). Previous segments 18% opacity; near (±1) 42%.
- Book-app page mode body: **19px Plex Serif** (`PageReader.tsx:95`); translation 16px sans.
- Book-app deck mode default: sentence **38px**, translation **22px**; mobile **20 / 15** (`lib/reader-prefs.ts:14–25`).
- Learn-stations home h1: 4xl/5xl (Tailwind sm:5xl); lesson cards 18px font-semibold.
- Readalong: no documented type scale; relies on Tailwind defaults + per-component sizing.

---

## 3. Spacing scale

All Tailwind-based apps inherit Tailwind's default scale (0.25rem step). Concrete observations:

- **Page padding:** `p-8` (32px) is the universal outer chrome — podcast-app catalog (`app/page.tsx:193` `padding: 32px 20px 64px`), book-app `/` (`app/page.tsx:10` `p-8`), book-app chapter (`p-8`), readalong drill page nests inside its own shell.
- **Card border-radius:** `12px – 16px` mostly; podcast catalog uses 14px (`CatalogClient.tsx:250` `border-radius: 14px`), notes flashcards uses `16` (`page.tsx:180`), learn-stations `rounded-2xl`.
- **Button min-height:** `44px` (catalog tabs, `CatalogClient.tsx:199`) and `48px` (notes flashcards Next button, `page.tsx:301`) — touch-target compliant.
- **Reading column max-width:** `--reading-max-width: 720px` (book-app, `globals.css:17`); `max-width: 1000px` (podcast-app catalog, `page.tsx:193`); `max-w-3xl` (book-app `/`); `max-w-5xl` (book-app reader frame).
- **Section gap:** 32–56px between major sections (podcast catalog `margin-top: 56px` on footer).

No app has a documented spacing scale named separately from Tailwind's defaults.

---

## 4. Shared primitives

**There is no shared package today.** The closest thing is:

1. **`@claw/svg-ontology`** — `link:../shared-components/svg-ontology` in learn-stations (`package.json:10`). Drives the SVG illustration set used by `/cards` and the various stations. **Not used by any other of the five apps.**

2. **The dictionary contract.** `dictionary.reeloo.ai/api/dictionary?word=X&lang=de|en|ja` is the canonical shared service. Consumed by readalong (`src/components/DictionaryPanel.tsx:212` etc.), notes (`app/api/dictionary-illustration/random/route.ts:21`). Book-app, podcast-app, learn-stations do NOT call it (they bake glosses in at content time). Memory `reference_shared_dictionary_api.md` documents this.

3. **The depot `/api/llm` gateway.** Called by book-app (`lib/llm-build.ts:14`, `lib/emotion-tagger.ts:50`) for content generation; the others use it via build-time scripts in `notes/scripts/` not at runtime.

4. **The audio file convention.** Both learn-stations and book-app use a `/audio/<hash>.mp3` + `/audio/<hash>.timing.json` sidecar pattern (`learn-stations/lib/tts.ts:300`; `book-app/lib/audio-url.ts:38`). Hashes are computed independently — they are NOT a shared store today.

5. **Copy-pasted components** (real duplication):
   - `KarottenPlayerV3.tsx` lives in BOTH `podcast-app/components/` AND `notes/app/podcast/_components/` (per `find` output). 2,442 identical lines.
   - `PodcastPlayerV3 / V1 / PlayerSwitcher / CatalogClient / RecentRow / ProgressModal` — same story.
   - `types/drill.ts` is a deliberate trim of `readalong/src/types/drillContentTypes.ts` (`learn-stations/types/drill.ts:1–8`).
   - Annotation underline colors hand-copied between podcast-app, book-app — same values, no shared declaration.

---

## 5. Theme strategy today

| App | Strategy | Themes available | Trigger |
|---|---|---|---|
| podcast-app | Inline `<style>` in component; `data-theme="solarized-light"` attribute set statically | 2 (cream + cobalt night, switched via `data-theme` inside `KarottenPlayerV3`) | Component-local |
| book-app | `prefers-color-scheme: dark` | 2 (light + system-dark) | Browser-driven |
| learn-stations | `document.documentElement.dataset.theme` + `data-theme=...` CSS selectors | 4 (vanilla, pip-duo, solarized-dark, high-contrast) | localStorage `learn-stations.theme` read pre-hydration (`layout.tsx:84`) |
| readalong | `<ThemeProvider>` (next-themes) → `.dark` class + `data-theme=...` | 9 (default ×2 + 4 light + 4 dark named) | localStorage `theme` read pre-hydration (`<script>` in `layout.tsx`) |
| notes /flashcards | None — undefined CSS vars with literal fallbacks | 1 | n/a |

**Three different theme triggers:** OS preference (book), localStorage with pre-hydration script (learn-stations + readalong), inline data attribute (podcast-app). The two app-level systems agree on the *mechanism* (`data-theme` + `dataset.theme`) but disagree on the **token namespace** (HSL-triplets in readalong; hex literals in learn-stations).

---

## 6. Contracts that already exist

External-facing (the ones a unified UI can rely on):

| Contract | URL | Consumers | Verified |
|---|---|---|---|
| **Dictionary** | `https://dictionary.reeloo.ai/api/dictionary?word=X&lang=de\|en\|ja` (CORS=*) | readalong (5 files), notes flashcards | HTTP 200 at 22:32 UTC for `?word=Hund&lang=de` |
| **Depot LLM gateway** | `${DEPOT_URL}/api/llm` (X-App-Token; `internal.reeloo.ai` default) | book-app `lib/llm-build.ts:14`, `lib/emotion-tagger.ts:212` | requires token |
| **Depot wizard** | `${DEPOT_URL}/api/wizard/generate` | podcast-app `lib/depot.ts:18` | requires token |
| **Depot podcast audio** | `${DEPOT_URL}/api/podcast-audio/<hash>.mp3` | podcast-app `app/api/audio/[hash]/route.ts` | requires token |
| **Depot TTS** | `${DEPOT_URL}/api/tts` | podcast-app `app/api/tts/route.ts:51`; learn-stations `lib/tts.ts:165` | requires token |
| **GCS book audio** | `https://storage.googleapis.com/reeloo-book-audio/<slug>/<chapter>/<paragraph>.mp3` (+ `.timing.json`, `cover.jpg`) | book-app `lib/audio-url.ts:25` | bucket is public-read; legacy GCP project |
| **GCS readalong exports** | `gs://readalong-exports/<exportId>.mp4` | readalong | server-side only |
| **Anthropic OAuth usage** | `/api/oauth/usage` | quota-poll scripts (out of scope here) | — |

Internal endpoints (per-app):
- learn-stations `/api/cards/state` (file-backed user SRS state, atomic writes)
- learn-stations `/api/stt-azure`, `/api/stt-gcp`
- readalong's 119 `/api/*` routes — most are app-internal
- book-app `/api/import-epub`, `/api/annotate`
- notes `/api/dictionary-illustration[/random]`

**Auth contracts:**
- Google OAuth via NextAuth: learn-stations (`auth.ts:18`), readalong (migrated from Firebase; `src/config/firebase.ts:1–7` comment says so).
- Firebase Auth (legacy): readalong still imports `firebase` 12.7 for analytics.
- `X-App-Token` header (server-side only): podcast-app, book-app, notes — for depot calls.
- `X-Forwarded-User` (via oauth2-proxy in Caddy): learn-stations `/api/cards/state` (`route.ts:39`).

---

## 7. The headline diagnosis

**One audio convention exists in two implementations** (`/audio/<hash>.mp3 + .timing.json`) — easy to merge.

**One dictionary contract exists and works** — needs to be consumed by the two apps that currently don't (book-app, learn-stations).

**One annotation taxonomy is 70% shared** — three colors agreed, one slot disputed (cultural vs place), one extension to add (archaism).

**Five typography stacks reduce cleanly to two: IBM Plex (sans + serif + mono) + Noto Sans TC + Noto Serif JP for furigana.** Drop Geist, Lora, system-UI.

**Theme strategy needs alignment.** The most-used pattern (learn-stations + readalong both use `data-theme` + pre-hydration localStorage script) is the winner. Three themes are sufficient: cream (default light Solarized), cobalt-dark (audio surfaces), and high-contrast (a11y).

**The biggest hidden debt:** the duplicated `KarottenPlayerV3` (2,442 lines) shipped in both podcast-app and notes. Any UI change must currently be applied in both places.
