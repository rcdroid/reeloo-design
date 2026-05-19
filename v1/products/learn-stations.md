# Product Spec — Learn-stations (`learn.reeloo.ai`)

> Source path: `/home/claw/work/learn-stations/`
> GitHub: (private — local working tree, deployed to Vercel `learn-stations` project)
> Live: <https://learn.reeloo.ai> · status 200 at 2026-05-19 22:32 UTC
> Dev port: 3210 (`package.json:6`)
> Screenshots: `learn-stations-home.png`, `learn-stations-deutsch-course.png`, `learn-stations-minna-l1-station.png`, `learn-stations-cards.png`

---

## 1. Identity

- **Name:** Pip — 多語言練習 (`app/layout.tsx:43`). Brand mascot: Pip (a green-leafed avatar; `data/avatars.ts`).
- **Tagline (synthesised):** *Twelve-to-thirteen mini-games per lesson — listening, vocab, matching, dictation, pronunciation, grammar — for Japanese (Minna no Nihongo), Standard German A1, Swiss German Mundart, and URL-sourced courses.*
- **The one user it serves today:** the operator drilling Japanese N5 (Minna) and German A1/A2 daily. Mobile-first; primarily run as a PWA on phone.
- **The user goal it advances:** *active* skill drill — output-side language production (speaking + writing + dictation + listening discrimination) with no flashcard escape hatch. This is where vocab introduced elsewhere gets hammered into reflexive recall.

## 2. Tech stack

| Concern | Choice | Source |
|---|---|---|
| Framework | Next.js 16.2.4 (App Router) | `package.json:14` |
| React | 19.2.4 | `package.json:18` |
| Auth | `next-auth` 5.0.0-beta.31, Google provider | `auth.ts:1–22` |
| Schema validation | `zod` 4.4.2 | `package.json:21`, `data/stations.ts` |
| Speech-to-text | Azure Cognitive Services (`microsoft-cognitiveservices-speech-sdk` 1.49) **and** Google Cloud Speech | `package.json:15`, `app/api/stt-azure/route.ts`, `app/api/stt-gcp/route.ts` |
| Japanese morphology | `kuromoji` 0.1.2 (build copies dict via `scripts/copy-kuromoji-dict.sh`) | `package.json:13` |
| HTML parsing | `@mozilla/readability` for URL-courses | `package.json:11` |
| Emoji set | `openmoji` 17.0.0 | `package.json:17` |
| Auth lib | `google-auth-library` 10.6.2 (GCP SA for STT) | `package.json:12` |
| Shared package | `link:../shared-components/svg-ontology` | `package.json:10` |
| Test | Vitest 4.1.5 + Playwright + Testing Library | `package.json:34–48` |
| E2B sandboxes | `e2b` 2.20.0 — used by `scripts/eval-e2b-browser.ts` for auto-evals | `package.json:38` |

**How it differs from siblings:** the most fully-realised app of the five. Owns Google OAuth, has its own STT/TTS pipelines, has 140 lesson files (40 Minna JP + 30 Deutsch A1 + 50+ Mundart de-CH + URL-generated), and ships **18 distinct station types** as a discriminated zod union. Tailwind v4 with **four named themes** wired via `data-theme` on `<html>`.

## 3. Route map

```
app/
├── layout.tsx                       # font setup + theme-init script (pre-hydration)
├── page.tsx                         # GET / — course picker
├── globals.css                      # 4-theme token system (vanilla / pip-duo / solarized-dark / high-contrast)
├── [course]/page.tsx                # GET /<course> — lesson list per course
├── [course]/[lesson]/page.tsx       # GET /<course>/l<N>?s=<station> — station player (the heart of the app)
├── [course]/[lesson]/layout.tsx     # course-scoped layout shell
├── cards/page.tsx                   # GET /cards?lang=ja|de|gsw — Anki-style deck (FLASHCARD #1)
├── cards/CardsDeck.tsx              # client view for /cards
├── cards/decks.ts, useDeckState.ts  # SRS state + deck definitions
├── settings/page.tsx                # GET /settings — theme, tier, locale, romaji toggle
├── auth/signin/page.tsx             # GET /auth/signin — Google sign-in
├── demo/intro/page.tsx              # GET /demo/intro — Pip intro animation
├── test/                            # designer playgrounds — exp1/exp2/exp3/substitution/svg-gallery
└── api/
    ├── auth/[...nextauth]/route.ts   # NextAuth handlers
    ├── stt-azure/route.ts            # POST — Azure Speech-to-Text (active)
    ├── stt-gcp/route.ts              # POST — GCP Speech-to-Text (alt path)
    ├── cards/state/route.ts          # GET/PUT — SRS state per user (file-based)
    ├── telemetry/learning/route.ts   # POST — append-only JSONL telemetry
    └── flag-issue/route.ts           # POST — bug-report capture
```

Per-route one-liners:

- `GET /` (`app/page.tsx`) — server component, reads `COURSES` from `data/courses/index.ts:384`, renders 3-column grid of course cards (URL / Minna / Mundart / Deutsch).
- `GET /<course>` (`app/[course]/page.tsx`) — server component, renders lesson list with locked/unlocked state.
- `GET /<course>/l<N>?s=<station>` (`app/[course]/[lesson]/page.tsx:30`) — **the central screen**. Reads `lesson.stations`, filters by active learner tier (`:51`), renders one `<StationView>` at a time with `?s=N` (1-indexed, `:58`) syncing back via `router.replace` (`:75`). Auto-advance: 2s by default (`:21`), cancellable.
- `GET /cards?lang=ja|de|gsw&lesson=<key>` (`app/cards/page.tsx:25`) — Anki-style deck. Front = SVG illustration + surface word; click flip → meaning + romaji + pipLine.
- `GET /settings` — global theme + tier + locale + show-romaji toggle.
- `POST /api/stt-azure` (`route.ts:69`) — accepts `{audio: base64, mimeType, language?}`, transcodes webm/opus → ogg/opus via ffmpeg (`:40–63`), calls `https://${REGION}.stt.speech.microsoft.com/...` (`:106`).
- `POST /api/stt-gcp` (`route.ts:89`) — calls Google Cloud Speech v1 with SA at `~/.config/gcloud/reeloo-media-sa.json` (`:35`).
- `GET/PUT /api/cards/state` (`route.ts`) — file-backed SRS state at `data/cards-state/<userId>.json`. User ID resolves from `X-Forwarded-User` header (oauth2-proxy) or `?userId=anon-XXXX` (`:39`). Atomic write via tmp+rename.
- `POST /api/telemetry/learning` — appends to `data/learning-events.jsonl` (gitignored).

## 4. Data model

### Lesson catalogue

`data/courses/index.ts` exports four `LessonStub[]` arrays and binds them in `COURSES: Record<CourseSlug, CourseInfo>` (`:384–423`):

```ts
type CourseSlug = "url" | "minna" | "mundart" | "deutsch";

interface CourseInfo {
  slug: CourseSlug;
  brandWord: string;     // chip label
  brandSub: string;
  heroTitleZh: string;
  heroTitleEn: string;
  accent: string;        // hex — drives the chip color on /
  lessons: LessonStub[];
}
interface LessonStub {
  id: number; title: string; subtitle: string;
  unlocked: boolean; stations: Station[];
}
```

Course counts (from `data/courses/index.ts:226, 306, 358, 382`):
- **minna** — 40 Minna no Nihongo JP lessons (`MINNA_L1`…`MINNA_L40`)
- **mundart** — 50 Mundart (Swiss German) lessons
- **deutsch** — 30 Hochdeutsch A1 lessons
- **url** — open-ended URL-generated vocab lessons

### Station discriminated union (`data/stations.ts`)

A single `Station = z.discriminatedUnion("type", …)` enumerating **18 station kinds**:
- `intro`, `vocab-intro`, `mc`, `vocab-draw`, `grammar`, `reshuffle`, `substitution-grid`, `word-match`, `cloze`, `image-mc`, `listen-mc`, `complete`, `pattern-stack`, `substitution-pattern`, `trace`, `echo`, `sequential-drill`

Each kind has its own zod schema; lessons import their canonical type from `data/stations.ts`. Tiers (`casual | modest | serious`) are an optional filter — stations missing `tiers` default to all (`app/[course]/[lesson]/page.tsx:51`).

### Drill cards (`types/drill.ts`)

Ported from readalong (`types/drill.ts:1–8`). Trimmed: 3-phase only (substitution / transformation / extension), no response phase, in-session-only SRS (SM-2 variant: `interval`, `easeFactor`, `recentResults` tracked but not persisted across sessions).

```ts
type DrillPhaseType = "substitution" | "transformation" | "extension";
type ReactionBand = "reflex" | "fast" | "normal" | "slow" | "timeout";
type MasteryLevel =
  | "unseen" | "introduced" | "learning" | "familiar" | "mastered" | "reflexive";
```

### Cards (SRS deck)

`app/cards/decks.ts` builds three decks (`ja`, `de`, `gsw`) from the cross-language vocab corpus. Per-user state is `Record<cardId, CardState>` stored at `data/cards-state/<userId>.json` (`api/cards/state/route.ts:32`). SRS code at `lib/srs.ts` (+ `srs.test.ts`).

### Telemetry

`data/learning-events.jsonl` (gitignored). One JSON object per event: `{ts, userId, course, lesson, station, eventType, accuracy?, reaction_ms?, transcript?, stt_confidence?}` (`lib/learning-telemetry.ts:69`).

## 5. Key user flows

### Flow A — Pick course → pick lesson → drill station

1. `GET /` (`app/page.tsx`). Click a card. Server-side `auth()` (`:7`) probes session (cookie); fails open.
2. `GET /<course>` lists lessons.
3. `GET /<course>/l<N>` lands on `?s=1`. State machine starts in `Stage.no-grade` (`page.tsx:91`).
4. `StationView` (`components/Station.tsx:80`) dispatches on `data.type` to the right station component (`mc.tsx`, `cloze.tsx`, `listen-mc.tsx`, etc.).

### Flow B — Sequential-drill (the high-craft station)

Locked design rules from `components/stations/sequential-drill.tsx:8–28`:
- One card = one slot. Cards sequential, never overlap.
- State machine: `card_load → tts → pause → listening_armed → scoring → card_correct | card_retry → next_card`. ≤1 sound + ≤1 visual change + ≤1 action at any moment.
- TTS and STT **never overlap**.
- Wrong = silence + transcript shown. Never red flash, never wrong-buzz haptic.

Smart-stop VAD (`:43–52`):
- 15s hard ceiling
- 8s if no voice
- 4s silence after voice
- 1.2s after match ≥ 0.7
- Immediate stop on ≥ 0.99

Scoring (`:55–57`): 60% slot match, 40% full-frame match. Pass at 0.7.

Pipeline: card mounts → `speak()` from `lib/tts.ts` (which first tries pre-built `/audio/<hash>.mp3` then falls back to `POST /api/tts`, `lib/tts.ts:165`) → 300ms `pre_mic_pause` → mic on → VAD smart-stop → `POST /api/stt-azure` → `scoreTranscript()` (`lib/score-transcript.ts`) → chord SFX on pass → next card. Telemetry event written via `recordEvent` (`lib/learning-telemetry.ts`).

### Flow C — Anki-style card study (`/cards`)

1. `app/cards/page.tsx:25` reads `?lang` and `?lesson`, loads the matching deck from `app/cards/decks.ts`.
2. `CardsDeck.tsx` renders SVG illustration on front, flip → meaning + romaji + example sentence.
3. State sync via `useDeckState.ts` — local-first, then `PUT /api/cards/state` if signed in.

### Flow D — Tier change re-shuffles station list

`app/[course]/[lesson]/page.tsx:50` filters stations by active learner tier. When the tier changes mid-lesson, the effect at `:84` clamps station index `i` to the new length rather than resetting to 0 — preserves deep-linked `?s=N`.

### Flow E — Sign-in (optional)

1. Anonymous users get an `anon-XXXX` localStorage id. All state file-backed under that id.
2. `/auth/signin` → Google OAuth (`auth.ts:18`). On success, `X-Forwarded-User` carries email through Caddy (production) → `/api/cards/state` migrates state under the email key.

### Flow F — Flag an issue

`<FlagIssue>` (`components/FlagIssue.tsx`, mounted in `app/layout.tsx:106`) is a floating chip on every page. Click → modal → POST `/api/flag-issue` → appended to `data/flag-issues.jsonl`. Used by the `inspect` skill (per `~/.claude/skills/inspect/SKILL.md`).

## 6. Design tokens used

`app/globals.css` is the **most token-disciplined** of the five apps. Four named themes, all WCAG AA-verified by `lib/design-tokens.test.ts`:

### Vanilla (default, light) — Solarized base3
- `--surface #fdf6e3` / `--surface-container #ffffff` / `--surface-elevated #fffcf0`
- `--on-surface #073642` / `--on-surface-muted #586e75` / `--on-surface-subtle #657b83`
- `--primary #cb4b16` / `--on-primary #ffffff` / `--primary-container #fdebd9` / `--on-primary-container #7a2810`
- `--border #8a8670` / `--border-strong #cb4b16`
- POS colors (`globals.css:45–49`):
  - `--pos-place #cb4b16` · `--pos-subject #268bd2` · `--pos-object #d33682` · `--pos-verb #6b7a00` · `--pos-particle #93a1a1`
- TTS highlight: `--tts-highlight-bg #daa520` (goldenrod) / `--tts-highlight-fg #073642`
- Graded states (`globals.css:21–35`): `correct` olive `#eaf3c2`/`#2d3f00`, `wrong` maroon `#fbdcd9`/`#6e0a08`, `warning` brown `#fdf3cf`/`#5c4400`.

### Pip Duo (`html[data-theme="pip-duo"]`, light)
- Surface `#ffffff`, primary `#58cc02` (Duolingo green), `--border-strong #2f7a00`. (`globals.css:149–163`)

### Solarized Dark (`html[data-theme="solarized-dark"]`)
- Surface `#002b36` (base03), container `#073642` (base02), elevated `#0e4856`. Primary `#cb4b16` preserved. (`globals.css:210–224`)

### High Contrast (`html[data-theme="high-contrast"]`)
- Surface `#000000`, primary `#ffaa00` (amber). (`globals.css:263–277`)

Theme persistence: `localStorage.getItem('learn-stations.theme')` read in a pre-hydration script (`app/layout.tsx:84–89`) to avoid FOUC.

Fonts (`app/layout.tsx:2–39`): `Geist Sans`, `Geist Mono`, `Noto Serif JP`, `Noto Sans TC`, `Lora` (paid Tiempos substitute). `body` family: `var(--font-noto-sans-tc), var(--font-geist-sans), 'Inter', system-ui` (`:98`).

Custom POS coloring is unique to this app — used in grammar example sentences to highlight place/subject/object/verb/particle (`components/JpPosColored.tsx`).

## 7. Components inventory

`components/` (top-level — 23 files):
- `Station.tsx` — dispatcher; reads `data.type`, renders the right station view.
- `RubyText.tsx` — furigana renderer for JP.
- `JaText.tsx` — JP text wrapper with `RomajiOnly` helper.
- `JpPosColored.tsx` — POS-tinted JP sentence renderer.
- `Mascot.tsx` — Pip avatar.
- `Scene.tsx` — illustration frame.
- `BottomCTA.tsx` — single morphing CTA (per `lib/stage.ts` — stations push state, stepper renders one button).
- `AutoAdvancePill.tsx` — 2s countdown chip.
- `EtymologyBubble.tsx` — popover with word origin.
- `DrillDensityBadge.tsx` — UI tag for drill density.
- `TierPicker.tsx` — casual/modest/serious selector.
- `PlayTTSButton.tsx`, `PlaySlowTTSButton.tsx`, `TTSDebugPanel.tsx`, `STTDebugPanel.tsx`, `SettingsButton.tsx`, `FlagIssue.tsx`.

`components/stations/` (20 station kinds, one tsx per kind):
- `intro`, `vocab-intro`, `mc`, `cloze`, `listen-mc`, `image-mc`, `complete`, `vocab-draw`, `grammar`, `substitution-grid`, `substitution-pattern`, `word-match`, `reshuffle`, `echo`, `trace`, `tier-text`, `pattern-stack`, `sequential-drill`, `sequential-drill-station`, `particle-burst`.

`components/exp3/` — designer playground (`chrome.tsx`, `drill.tsx`, `stations.tsx`, `tokens.ts`).

**Duplicates with siblings:** Most of the station UI is unique. The `RubyText`/POS coloring overlap with notes/readalong but the implementations here are the canonical ones.

## 8. External integrations

| Call | File:line | Purpose |
|---|---|---|
| Azure Cognitive Services STT REST | `app/api/stt-azure/route.ts:106` | Sync STT for sequential-drill |
| Google Cloud Speech v1 REST | `app/api/stt-gcp/route.ts:142` | Alt STT path (billing pending per `:5`) |
| Google Cloud TTS via depot `/api/tts` | `lib/tts.ts:165` | Fresh TTS at runtime when no pre-built `/audio/<hash>.mp3` exists |
| Pre-built audio | `lib/tts.ts:300` `fetch('/audio/<hash>.mp3')` and `.timing.json` sidecar | First-line cache; emitted at build by `scripts/build-audio.ts` |
| Google OAuth | `auth.ts:18` | Sign-in |
| GCP Speech SA file | `~/.config/gcloud/reeloo-media-sa.json` | (`stt-gcp/route.ts:35`) project `claw-491508` |

No call to `dictionary.reeloo.ai` — vocab glosses are baked into lesson files. (This is a candidate to migrate to the shared dictionary contract.)

## 9. Screenshots

- `v1/screens/learn-stations-home.png` — live `learn.reeloo.ai/` (course picker; Pip mascot; Solarized cream)
- `v1/screens/learn-stations-deutsch-course.png` — `localhost:3210/deutsch` (lesson list)
- `v1/screens/learn-stations-minna-l1-station.png` — `localhost:3210/minna/l1?s=1` (first station)
- `v1/screens/learn-stations-cards.png` — `localhost:3210/cards` (Anki deck view, JA default)

## 10. What this app does that others can't reasonably absorb

The **sequential-drill state machine** + **VAD smart-stop** + **`scoreTranscript`** stack. It's six months of co-evolution between the player (`components/stations/sequential-drill.tsx`), the TTS pipeline (`lib/tts.ts` with `/audio/<hash>.mp3` cache + `timing.json` sidecars), the phonetic-distance scorer (`lib/phonetic-distance.ts` + `score-transcript.ts`), and the Azure STT proxy. The locked design (`:8–28`) is real — every rule has a "do NOT relax" because it was violated in an earlier iteration. Nothing else in the codebase comes close on speaking-drill latency and feedback discipline.

Beyond that: the **18-kind discriminated-union station model** is a more mature pedagogy taxonomy than anything in readalong's free-form deck schema.

## 11. What this app does that overlaps with others

- **`/cards` (Anki deck) overlaps with notes `/practice/flashcards` and readalong's deck library.** Three flashcard surfaces, three SRS stores, zero data sync between them. Merge target: a single `/library/cards/...` surface backed by one SRS contract.
- **TTS pipeline** (`lib/tts.ts`) duplicates readalong's `lib/drillVoiceClient.ts` and book-app's `scripts/synth-audio*.ts`. Three apps generate Google Cloud TTS independently. Merge target: a shared `audio.reeloo.ai` service producing `/audio/<sha256>.mp3` + `timing.json` sidecars.
- **Drill types** are a trim of readalong's drill type tree (`types/drill.ts:1–8` says so explicitly). Merge target: pull the trimmed schema upstream into readalong, ship the shared package.
- **Telemetry** (`data/learning-events.jsonl`) duplicates readalong's `firestore-analytics.ts`. Merge target: one event sink.
