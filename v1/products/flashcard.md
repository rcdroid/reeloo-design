# Product Spec — Flashcard (the cross-app vocab review surface)

> Source paths (multi-rooted — there are three independent flashcard surfaces):
> 1. `/home/claw/work/notes/app/practice/flashcards/` — Flux-LoRA-illustrated random vocab
> 2. `/home/claw/work/learn-stations/app/cards/` — Anki-style multi-language deck with SRS
> 3. `/home/claw/work/readalong/nextjs/src/components/drill/`-adjacent deck list (deck-as-flashcard-deck via DeckBrowser)
>
> Live: <https://learn.reeloo.ai/cards> (the LRN deck), <https://depot.reeloo.ai/practice/flashcards> (the depot-internal illustrated card). Neither is currently a standalone domain.
> Screenshots: `flashcards.png` (notes/practice), `learn-stations-cards.png` (learn-stations deck)

---

## 1. Identity

- **Name:** none unified yet. Three implementations exist; the brief asks us to spec "Flashcard" as the consolidation target.
- **Tagline (synthesised):** *Illustrated vocab cards with SRS — random-reveal flow for discovery, deck-paced study for retention, one dictionary contract under the hood.*
- **The one user it serves today:** the operator reviewing JP / DE / EN vocab. In practice the operator hits `/cards` on learn-stations more than `/practice/flashcards` on depot because the former has SRS state and audio; the depot one is a recent LoRA-illustration prototype (`page.tsx:7–19`).
- **The user goal it advances:** *spaced recognition + recall* — the leaf surface of every vocab-introducing flow (lessons, podcasts, books, the dictionary). Tells the system what you've internalised.

## 2. Tech stack (per implementation)

### A. notes/practice/flashcards (`/home/claw/work/notes/app/practice/flashcards/page.tsx`)

| Concern | Choice | Source |
|---|---|---|
| Framework | Next.js 16.2.2 inside the depot monorepo | `notes/package.json:80` |
| State | `useState` only — no SRS, no telemetry | `flashcards/page.tsx:84–89` |
| Style | Inline CSS-in-JS via `style={{}}` — no Tailwind classes, no tokens. Falls back to `var(--accent, #4f46e5)` etc. | `flashcards/page.tsx:147–305` |
| Illustration | Flux.1 dev LoRA `flashcard-stroke-v1`, generated offline by `scripts/illustrations/generate_flashcards.py`, served by `app/api/dictionary-illustration/route.ts` | `flashcards/page.tsx:7–11`, `app/api/dictionary-illustration/route.ts:1–18` |
| DB | SQLite via `@libsql/client` (the depot main DB) | `notes/lib/db.ts`, `app/api/dictionary-illustration/route.ts:51` |
| Dictionary | Calls localhost:18112 dictionary service | `app/api/dictionary-illustration/random/route.ts:20–21` |

### B. learn-stations/app/cards (`/home/claw/work/learn-stations/app/cards/`)

| Concern | Choice | Source |
|---|---|---|
| Framework | Next.js 16.2.4 (learn-stations) | `learn-stations/package.json:14` |
| State | `useDeckState.ts` (per-card flip + lesson scope); SRS lib at `lib/srs.ts` exists but **not wired** | `app/cards/page.tsx:11–14`, `lib/srs.ts` (separate) |
| Server state | `GET/PUT /api/cards/state?userId=` — atomic file write to `data/cards-state/<userId>.json` | `app/api/cards/state/route.ts:1–25` |
| Auth | `X-Forwarded-User` (oauth2-proxy) or `anon-XXXX` localStorage id | `app/api/cards/state/route.ts:39–42` |
| Style | Tailwind v4 + the 4-theme token system (vanilla/pip-duo/solarized-dark/high-contrast) | inherited from learn-stations layout |
| Audio | Pre-built `/audio/<hash>.mp3` + `timing.json` from `scripts/build-audio.ts` | `lib/tts.ts:165, 300` |

### C. readalong DeckBrowser

A deck IS a flashcard deck in readalong's model (`src/components/DeckBrowser.tsx`, `DeckList.tsx`, `DeckHomePage.tsx`). The deck-player renders cards via `components/deck-player/layouts/{Card,Stack,Split,Bridge}Layout.tsx`. Backed by Firestore.

**How they differ:** A produces ONE random illustrated card per click (discovery mode); B produces a deck of cards bounded by a lesson scope with prev/next (study mode, flip-card); C produces a paginated multi-segment deck with audio playback (presentation mode).

## 3. Route map

| Surface | Route | What renders |
|---|---|---|
| notes | `GET /practice/flashcards` | `app/practice/flashcards/page.tsx` — random card |
| notes | `GET /api/dictionary-illustration?word=&lang=` | PNG bytes from `~/cache/illustrations/dictionary/` (`route.ts:1–17`) |
| notes | `GET /api/dictionary-illustration/random?lang=` | `{word, lang, illustration_url, entry}` (`random/route.ts:1–12`) |
| learn-stations | `GET /cards?lang=ja\|de\|gsw&lesson=` | `app/cards/page.tsx:25` |
| learn-stations | `GET/PUT /api/cards/state?userId=` | SRS state file |
| readalong | `GET /` (the LibraryView) | Deck grid with cover, level, language pair |
| readalong | `GET /deck/<id>` | `app/deck/[id]/page.tsx` — full deck player |
| readalong | `GET /api/decks` | Firestore deck listing |

## 4. Data model

### A. Illustrated card (notes)

```ts
interface RandomCard {
  word: string;
  lang: "ja" | "de" | "en";
  illustration_url: string;     // /api/dictionary-illustration?word=&lang=
  entry: DictEntry | null;      // from dictionary service
}
interface DictEntry {
  word: string;
  readings?: string[];
  partOfSpeech?: string;
  level?: string;               // CEFR / JLPT
  senses?: { definitions?: {text, lang?}[]; examples?: {text, translation, reading}[] }[];
}
```

Storage (notes):
- `dictionary_illustrations` table in libsql (`notes/lib/db.ts`) — `{word, lang, png_path}` rows.
- PNG bytes under `~/cache/illustrations/dictionary/<lang>/<word>.png`.
- Dictionary entries fetched live from `http://localhost:18112/api/dictionary` (`random/route.ts:21`).

### B. learn-stations deck (`app/cards/decks.ts`)

Each deck is `Record<lang, {label, entries: VocabEntry[], lessons: [{key, label, entries}]}>`. The deck-state file stores `Record<cardId, CardState>` where `CardState` carries SM-2 fields (interval, easeFactor, recentResults) — same shape as `types/drill.ts::SlotMastery` for consistency.

### C. readalong deck (Firestore `decks/<id>`)

`type DeckFile` is at `readalong/nextjs/src/types/deckFile.ts` (not opened here for token cost) — multi-segment slides with `page.layout` (`stack | card | split | bridge`).

## 5. Key user flows

### A. notes random discovery loop

1. Mount → `useEffect` triggers `fetchNext()` (`flashcards/page.tsx:115`).
2. `GET /api/dictionary-illustration/random?lang=<lang>`.
3. Server picks a random row via `ORDER BY RANDOM()` (`random/route.ts:35–42`), then `fetch(${DICT_BASE}/?word=&lang=)` (`:20–21`) to enrich with definitions + examples.
4. Render card: illustration on top, big word + readings + POS, hidden meaning behind "Show meaning" button (`page.tsx:262`).
5. "Next card" → repeat. No state captured.
6. 503 on empty: surfaces the runbook hint "Run scripts/illustrations/generate_flashcards.py first" (`page.tsx:97–102`).

### B. learn-stations deck study (`/cards?lang=ja`)

1. `app/cards/page.tsx:25` reads `lang` from query, picks deck from `DECKS[lang]` (`app/cards/decks.ts`).
2. `<CardsDeck>` renders front: SVG illustration + JP/DE/gsw surface form.
3. Click flip → back: meaning + romaji (`useShowRomaji` toggle, `app/cards/page.tsx:46` ish) + example sentence.
4. Next/Prev navigation; auto-play TTS via `speak()` (`lib/tts.ts`) — first tries `/audio/<sha256>.mp3` static, falls back to `POST /api/tts` (`lib/tts.ts:165`).
5. On flip-with-answer (planned): `PUT /api/cards/state?userId=` writes the SM-2 update. (`useDeckState.ts` — confirmed in code, SRS not active until "increment 3" per `app/cards/page.tsx:12`.)

### C. readalong deck-as-deck (the deck-player flow)

Outside the strict "flashcard" scope but it's the third surface that uses card UI. See `readalong.md`.

## 6. Design tokens used

### notes `/practice/flashcards`
- **Pure inline CSS-in-JS, no token discipline.** Falls back to `var(--accent, #4f46e5)`, `var(--border, #ddd)`, `var(--card-bg, #fff)`, `var(--muted, #888)`, `var(--fg, #111)` (`flashcards/page.tsx:147–305`). The vars are NOT defined anywhere globally in notes — it's all default fallback. This is the lowest-token-discipline surface in the entire five-app set.

### learn-stations `/cards`
- Inherits the full 4-theme token system from learn-stations globals (`learn.md` section 6). Solarized-cream default. Card chrome respects `--surface`, `--border`, `--primary`, the four state colors.

## 7. Components inventory

### notes
- `app/practice/flashcards/page.tsx` (309 lines) — single-file component, no extraction.

### learn-stations
- `app/cards/CardsDeck.tsx` — front/back card view
- `app/cards/decks.ts` — deck definitions (JA, DE, gsw)
- `app/cards/useDeckState.ts` — state hook
- `lib/srs.ts` (+ `srs.test.ts`) — SM-2 algorithm
- `lib/vocab-cards.ts` (+ test) — vocab data layer

### readalong
- `src/components/DeckBrowser.tsx`, `DeckList.tsx`, `DeckHomePage.tsx`, `DeckDetailPage.tsx`, `DeckEditorModal.tsx`, `DeckLoadingView.tsx`, `DeckPromptsModal.tsx`, `DeckSettingsTab.tsx`, `DeckGenerationProgress.tsx`, `DeckCreationWizard.tsx` — ten deck UI components in `src/components/`.
- `src/components/deck-player/layouts/{Card,Split,Stack,Bridge}Layout.tsx` — four layout templates per slide.

**Duplicates: severe.** Three implementations of "show a vocab card and ask the user if they got it" with no shared code path.

## 8. External integrations

| Call | File:line | Purpose |
|---|---|---|
| `${DICTIONARY_API_URL or localhost:18112}/api/dictionary?word=&lang=` | `notes random/route.ts:21` | Definition/example enrichment |
| Vertex/depot (via build-time script `scripts/illustrations/generate_flashcards.py`) | offline | LoRA-illustrated PNG generation; results written to libsql + `~/cache/` |
| `/api/tts` (learn-stations) | `learn-stations lib/tts.ts:165` | Card front audio |
| Google OAuth (learn-stations) | `learn-stations auth.ts:18` | Identifies the user for SRS sync |
| Firebase Auth (readalong) | `readalong src/utils/firebaseAuth.ts` | Identifies the user for deck ownership |

**The dictionary contract is the single most-shared service across these surfaces.** Per memory `reference_shared_dictionary_api.md`, `dictionary.reeloo.ai/api/dictionary?word=X&lang=de|en|ja` is the canonical source (CORS=*, normalization built-in). At inspection: live (HTTP 200 at 22:32 UTC for `?word=Hund&lang=de`).

## 9. Screenshots

- `v1/screens/flashcards.png` — `localhost:18100/practice/flashcards` (notes / illustrated random)
- `v1/screens/learn-stations-cards.png` — `localhost:3210/cards` (learn-stations deck)
- *(No readalong DeckHomePage screenshot — `readalong-browse.png` shows the closest equivalent.)*

## 10. What this app does that others can't reasonably absorb

A unified Flashcard surface needs all three behaviours:
1. **Discovery loop** (notes pattern) — "I don't know what I don't know"; surface one random illustrated card from a corpus.
2. **Deck-bounded study** (learn-stations pattern) — "I want to drill lesson 5 of Minna today"; deterministic order, flip card, SRS-graded.
3. **Deck-as-curriculum** (readalong pattern) — "Here's a 20-card deck someone built for B1 phone-call phrases"; pageable, audio-paced.

The illustration pipeline (Flux LoRA `flashcard-stroke-v1`, `notes scripts/illustrations/`) is unique. The SM-2 SRS algorithm exists in two places (`learn-stations/lib/srs.ts` and `readalong/.../adaptiveEngine.ts`).

## 11. What this app does that overlaps with others

Almost everything overlaps. **The only honest answer is: merge to one surface.** Recommended consolidation target:

- One route tree: `/library/cards`, `/library/cards/<deck>`, `/library/cards/random`.
- One data contract: `Card = { id, lang, surface, reading?, gloss, illustration?, audio?, example?, srs: SlotMastery }` (the `SlotMastery` shape already in `learn-stations/types/drill.ts:34–51`).
- One SRS implementation: `learn-stations/lib/srs.ts` (has tests).
- One dictionary contract: `dictionary.reeloo.ai/api/dictionary` (already exists).
- One illustration contract: `GET /api/illustration?word=&lang=` (currently in notes, easily portable).
- One audio contract: `/audio/<sha256>.mp3` + `.timing.json` (the learn-stations pattern wins on simplicity).

Migration map for the 3 existing surfaces is in the unified brief.
