# Design request: Flashcards — unify three implementations + LoRA illustration pipeline

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/flashcard.md` (183 lines — honestly documents the 3-implementation problem)
- `v3-gap-audit/learn-stations.md` §1 (features 54–63 — the `/cards` surface)
- `v3-gap-audit/readalong.md` §1.D (features 68–72)
- Source code: `/home/claw/work/learn-stations/app/cards/`, `/home/claw/work/notes/app/practice/flashcards/page.tsx` (308 LOC), `/home/claw/work/readalong/nextjs/src/components/DeckBrowser.tsx` + 9 sibling Deck* components

## What we asked for in v2 (and what we got)

The v2 brief named `<Flashcard>` as one shared component (`v2/06` §7.1) and proposed `/library/cards` as the unified destination. Claude design produced **nothing** dedicated to flashcards. The impl `reeloo-v2/components/routes/practice.tsx` has a "Not yet / Got it" pair on a card-shaped UI but it's part of the station player, not a deck-browsable flashcard surface. `reeloo-v2/components/routes/me.tsx` (938 LOC) does NOT include `/me/library/cards`.

This means: three flashcard implementations live in production, the v2 brief asked for unification, design did nothing, the impl has no flashcard route at all.

## What's actually in the source apps

**Implementation A — notes/practice/flashcards (random discovery)**

`/home/claw/work/notes/app/practice/flashcards/page.tsx` (308 LOC). A random LoRA-illustrated card surfaces; "Show meaning" reveals the back. Wired to:
- `GET /api/dictionary-illustration/random?lang=ja|de|en` — picks random word + serves PNG bytes from `~/cache/illustrations/dictionary/<lang>/<word>.png` (built offline by `scripts/illustrations/generate_flashcards.py` using Flux.1 dev LoRA `flashcard-stroke-v1`).
- `GET /api/dictionary?word=&lang=` — enriches with readings, POS, CEFR level, definitions, examples.
- Pure inline CSS-in-JS, no token discipline — falls back to `var(--accent, #4f46e5)` etc. The lowest-token-discipline surface in the codebase.

**Implementation B — learn-stations /cards (Anki-style deck with SRS)**

`/home/claw/work/learn-stations/app/cards/` — multi-file:
- `page.tsx` (~50 LOC) — `?lang=ja|de|gsw&lesson=<key>` query routing
- `CardsDeck.tsx` — front (SVG illustration + surface word + auto-TTS) / back (meaning + romaji + example sentence + pipLine)
- `decks.ts` — DECKS[lang] catalogue
- `useDeckState.ts` — local-first SRS state, syncs to `/api/cards/state`
- `lib/srs.ts` — SM-2 algorithm with tests
- SRS state path: `data/cards-state/<userId>.json`, atomic tmp+rename writes
- User ID from `X-Forwarded-User` (oauth2-proxy) or `?userId=anon-XXXX` (localStorage)
- Pre-built audio first (`/audio/<sha256>.mp3` + `.timing.json`), falls back to live TTS

Existing testids: `cards-front-surface`, `cards-flip`, `cards-flip-button`, `cards-back-pipline`, `cards-autotts-toggle`, `cards-dict-link`, `cards-example-play`, `cards-grade-feedback`, `cards-grade-panel`.

**Implementation C — readalong DeckBrowser family (deck as curriculum)**

10 components in `/home/claw/work/readalong/nextjs/src/components/`:
- `<DeckBrowser>` — main browse view
- `<DeckList>` / `<DeckHomePage>` / `<DeckDetailPage>`
- `<DeckEditorModal>` — author/edit a deck
- `<DeckLoadingView>` — loading state
- `<DeckPromptsModal>` — pre-deck prompts
- `<DeckSettingsTab>` — deck settings
- `<DeckGenerationProgress>` — SSE deck-from-URL progress
- `<DeckCreationWizard>` — multi-step wizard

Plus `<DeckBrowser>` defers to 4 layouts at `components/deck-player/layouts/`: `CardLayout`, `StackLayout`, `SplitLayout`, `BridgeLayout` (discriminated by `page.layout` enum).

## What we need from Claude design

A unified flashcard surface that **absorbs all three behaviours**:

1. **One route tree** — design `/library/cards` (random + deck-bounded study) and `/library/decks` (deck-as-curriculum). Both live at `practice.reeloo.ai/library/*` per v2 brief §3.

2. **`<Flashcard>` primitive** — front + back + flip animation. Front: illustration (LoRA PNG OR SVG-emoji OR text-only fallback), surface word, optional reading (furigana for JP), auto-TTS chip. Back: meaning, romaji (toggle), POS, CEFR/JLPT level, example sentence, dict-link icon, grade panel. Both faces in light/dark/high-contrast.

3. **Four-state grade panel** — `again / hard / good / easy` (the SM-2 grades, required by v2 brief §9 funnel F3). Visual: 4-button row, each with the grade label + estimated next-interval ("again = in 1 min", "easy = in 4 days"). Color-coded: again = warm warning, hard = neutral, good = accent, easy = success. Per `data-testid="cards-grade-panel"` + `cards-grade-feedback`.

4. **Random discovery mode** (notes pattern) — single card, "Show meaning" reveal, "Next card" CTA, language filter chip. Empty state when illustration cache is empty ("Run scripts/illustrations/generate_flashcards.py first" — but as a friendly "We're still illustrating words in this language — try another?").

5. **Deck-bounded study mode** (learn-stations pattern) — `?lang=` + `?lesson=<key>`. Deterministic order. Prev/Next nav. Grade panel writes to SRS. Auto-TTS on front display.

6. **Deck-as-curriculum mode** (readalong pattern) — `/library/decks/<id>` with the 4 layout templates (Card / Stack / Split / Bridge). Each layout is a slide variant inside a paginated deck-player.

7. **`<DeckBrowser>` mockup** — grid of deck cards with cover, title, language pair, level, card count, last-studied. Filter chips (language, level, deck type). Search.

8. **Deck-creation flow** — `<DeckCreationWizard>` multi-step UX: pick source (URL, YouTube, manual, generate-from-vocab) → preview → confirm → generation progress (SSE) → done. The MP4 export pipeline learnings apply here (stage-by-stage progress, cancellable).

9. **LoRA illustration UX** — when an illustration is being generated for the first time, what does the user see? Currently the backend pre-generates; design should plan for runtime generation (latency 5–10s).

10. **Dictionary link affordance** — `cards-dict-link` testid. Tap → opens dictionary popover OR navigates to `/dictionary/<word>`. Design picks the form-factor.

11. **Romaji toggle** — global setting (`useShowRomaji`) AND per-card override. Design: where does the global toggle live (settings? card chrome?)?

12. **Auto-TTS toggle** — global setting (`cards-autotts-toggle`). When OFF, the card surface still has a manual play chip.

13. **Mobile gesture model** — swipe-right to flip? swipe-up for "easy"? Tap = flip; tap-and-hold = "again"? Design picks; the convention must be consistent with the rest of the practice app.

14. **The pip-line back-of-card** — `cards-back-pipline` testid. What is the pipLine? (Looking at learn-stations source: it's the structured back-of-card with reading + meaning + example.) Design specs the layout.

15. **SRS queue indicator** — a "30 cards due today" chip somewhere in the header. Tap → deck/random launch. The `/me/diary` weekly summary references "17 cards rated" which implies an inbox.

16. **The "saved from podcast" / "saved from book" provenance** — when a word lands in the SRS via dict.save (v2 brief §4.1 J1), where does the provenance show? Maybe a tiny chip on the card back ("saved 2d ago from Karotten · Eine Geschichte"). Design picks.

## Out of scope

- The Flux.1 dev LoRA training pipeline (`scripts/illustrations/`) — backend.
- The dictionary service (`dictionary.reeloo.ai/api/dictionary`) — canonical, no change.
- The SM-2 algorithm in `lib/srs.ts` — backend; design only owns the UX of the grade panel.
- The Firestore vs file vs IndexedDB storage debate — backend.

## Success criteria

- One `<Flashcard>` primitive composes all three modes (random / deck-bounded / deck-as-curriculum).
- The four-state grade panel is the canonical interaction (no two-state "Got it / Try again" — that's what learn-stations has and the v2 brief explicitly rejects in §9).
- LoRA-illustration cache-miss has a defined UX.
- `/library/cards` (random + deck) and `/library/decks` (curriculum) are two top-level surfaces with clear separation.
- All existing testids (`cards-flip`, `cards-grade-feedback`, etc.) map to the new convention without losing semantic identity.
- The 4 readalong layout templates (Card / Stack / Split / Bridge) survive as `<Flashcard variant="card|stack|split|bridge">` props.
