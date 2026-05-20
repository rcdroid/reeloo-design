# Design request: Book reader — Page mode + Deck mode + EPUB upload + search

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/book-app.md` §5–§7 (the existing spec — solid on data model, light on Deck-reader internals)
- `v3-gap-audit/book-app.md` §1 (features 6–41), §4 (impl gaps)
- Source code: `/home/claw/work/book-app/components/DeckReader.tsx` (770 LOC), `PageReader.tsx` (111 LOC), `components/deck/*.tsx` (6 files, ~1,128 LOC)

## What we asked for in v2 (and what we got)

The v2 brief named `<ReadingColumn>`, `<DeckCard>`, `<AnnotatedSpan>`, `<AnnotationRail>` as shared components and assumed the v1 book-app spec was complete. Claude design delivered `routes/book.jsx` (276 LOC) with a static prose block, two hardcoded annotation cards, no audio, no font controls, no Page-vs-Deck toggle, no EPUB drop, no search, no annotation taxonomy (one inline `.rb-anno` span class for all kinds).

The impl `reeloo-v2/components/routes/book.tsx` is 346 LOC ported faithfully from the design-source. It looks like a book; it does almost nothing a book reader does.

## What's actually in the source app

Three reader surfaces, all live:

### Page mode (default)

- `<PageReader>` (`components/PageReader.tsx`, 111 LOC) renders paragraphs with IBM Plex Serif body 19px, `line-height: 1.75`, `max-width: 720px`. Each paragraph has an optional interlinear translation line below (font-sans 16px, muted color).
- Annotation spans rendered via `renderWithNotes(text, notes)` (`PageReader.tsx:13-52`) which performs greedy non-overlapping span replacement and hands each span to `<AnnotatedSpan>` (`AnnotationPopover.tsx`).
- **6-category annotation taxonomy** with distinct Solarized hues (one of which — archaism — uses a unique **dotted** underline). Source: `lib/annotation-schema.ts:4`, tokens in `globals.css:152-160`.
- Three URL-driven toggles in the chapter header: `?mode=page|deck`, `?target=zh-TW|en|de`, `?anno=on|off`. The toggle components (`ModeToggle`, `LanguageToggle`, `AnnotationToggle`) write to URL and the reader re-derives via window-location polling every 200ms.

### Deck mode

- `<DeckReader>` (`components/DeckReader.tsx`, 770 LOC) is a paged audio-paced reader.
- Per-paragraph audio fetched from GCS: `https://storage.googleapis.com/reeloo-book-audio/<slug>/<ch>/<paraId>.mp3` plus a `<paraId>.timing.json` word-boundary sidecar. Source: `lib/audio-url.ts:18-25, 30, 38`.
- `boundariesInRange()` (`lib/word-timing.ts`) drives word-by-word highlight; active word wrapped in `<span class="deck-word-active">` with a high-contrast background designed to read over AI cover images. Source: `DeckReader.tsx:158-180`.
- **Adjustable font sizes**: sentence default 38px desktop / 20px mobile (range 14–72); translation default 22px / 15px (range 12–40); persisted to localStorage `book-reeloo:sentencePx` + `book-reeloo:translationPx`. Aa+/Aa- controls in `<DeckReaderControls>` (157 LOC, auto-hide timer).
- **Player bar** (`<DeckPlayerBar>`, 276 LOC) — play/pause + scrub + speed + paragraph-cycle toggle (auto-replay current paragraph when end reached, state `cycle` at `DeckReader.tsx:235`).
- **Annotation rail** with 2 tabs (`notes` + ?): `<DeckAnnotationRail>` (314 LOC), `RAIL_WIDTH_PX = 380`. Annotations get **numbered superscripts** via `<InlineNumberedSpan>` (110 LOC) that map to numbered rail entries. The main reader re-centers in remaining viewport when rail opens.
- **Mobile viewport reflow** — `isMobile` state observes viewport; reflows player bar position, hides Aa controls behind tap-to-show, shrinks rail handle width.
- **`<DeckBackground>`** (146 LOC) — book-cover-derived blurred background art for the deck mode.
- **Per-sentence timing windows** so audio plays through but visual deck advances per sentence. Includes exponential-backoff retry for slow GCS sidecars (`:407-455`).

### Library + uploads + search

- `<DropZone>` (80 LOC) accepts EPUB drag-drop, POSTs to `/api/import-epub` (server hashes sha256 12-char, writes to `os.tmpdir()`, parses via `epub2`, returns JSON), result stored in IndexedDB; user redirected to `/read/uploaded-<sha>`.
- `/search?q=&slug=` — client-side **FlexSearch** full-text index over `data/books/<slug>/index.flexsearch.json`, 30-result limit, result links to chapter + paragraph anchor.
- Runtime annotation: `POST /api/annotate` calls Vertex AI Gemini 3 Pro (fallback 2.5 Pro) with the system prompt at `lib/annotation-prompt.ts`. 30s timeout.

## What we need from Claude design

1. **Page mode mockup** with the 720px max-width column, Plex Serif body, interlinear translation row, and **all six annotation kinds visible** (idiom, proper_noun, place, archaism, grammar, vocab) so the underline color taxonomy + the archaism dotted style are documented. Mobile variant.
2. **`<AnnotatedSpan>` popover spec** — tap or hover, opens a card with kind chip, phrase, gloss, "Save phrase" CTA, "Hear it" CTA, "Dictionary" link. Dismiss on outside-tap. Accessible (focus trap, ESC closes).
3. **Chapter header chrome** — book title + chapter number + three toggles (mode / target / anno) + "Export chapter as video" CTA. Mobile: toggles collapse into a "Aa•中•⋯" combo.
4. **Deck mode mockup** with the active-paragraph card centered, big text (38px Plex Serif), translation below, audio bar pinned bottom, **active-word highlight** clearly visible (high-contrast bg over the deck-background art). At least three frame states: paused / playing / mid-word-transition.
5. **`<DeckBackground>` treatment** — show how the book cover blurs (Gaussian blur radius, opacity ramp), how the active card stands above. AI cover vs commissioned art treatments.
6. **`<DeckPlayerBar>`** — play/pause + scrub + time + speed + paragraph-cycle toggle + close. Cycle toggle visual is critical — when on, the bar shows a "loop" pictogram.
7. **`<DeckReaderControls>` (Aa+/Aa-)** — floating mini-toolbar; auto-hide after 3s of inactivity, reappear on touch/move. Aa+ / Aa- buttons with the current px value visible.
8. **Annotation rail (380px) + InlineNumberedSpan** — superscript numbers in body text (small but legible at 19px base), rail entries with the matching number + kind chip + gloss + actions. Show the layout shift when rail opens (re-center, don't reflow).
9. **Rail two-tab structure** — `notes` (annotations) + `?` (second tab — design must decide: bookmarks? definitions? sentences? Spec'd in `RailTab` enum at `DeckReader.tsx:231` but content undefined).
10. **Library `/`** — DropZone (drag-target visible state, drag-over hover, post-upload loading state, error state) + `<BookCard>` grid (cover + title + author + lang chip + level chip + chapter count).
11. **Drop-zone error states** — malformed EPUB, file too large, `parseEpub` throws. Each needs copy + recovery affordance.
12. **`<ChapterTOC>`** — chapter list with chapter num + title + level-fit chip + minutes estimate + read-status (read/in-progress/unread).
13. **`/search` UI** — query input + per-result row (chapter + paragraph excerpt with query terms highlighted) + empty state + "no results" state. Default `lord-of-the-rings` slug behavior.
14. **Runtime annotation loading state** — when the user opens an uploaded book chapter, `POST /api/annotate` may take 30s per paragraph. Design needs: per-paragraph "annotating…" spinner, progressive reveal as annotations land, or batched (annotate whole chapter, show progress bar).
15. **Light/dark + WCAG AA** — book-app uses `prefers-color-scheme` not a toggle. Design must decide: keep browser-driven, OR add the cross-domain `.reeloo.ai` cookie theme switch (per v2 brief §6.2). Recommendation: browser-driven for book-app to match the "reading is environment-respectful" framing.

## Out of scope

- The EPUB parsing pipeline + GCS audio generation (`scripts/synth-audio.ts`) — backend, no design change.
- The annotation JSON schema — already locked at `lib/annotation-schema.ts`; design renders what the data says.
- The Vertex AI Gemini model choice — backend.
- The IndexedDB client storage — design only owns the upload UX, not the persistence model.

## Success criteria

- Page mode renders an actual page of LOTR with all 6 annotation kinds visible and visually distinct.
- Deck mode supports per-paragraph audio with word-highlight; mobile and desktop variants documented.
- EPUB drop produces a working upload UX with all four states (idle / drag-over / uploading / error).
- Search UI supports the existing `data-testid="search-result"` contract and the 30-result limit.
- Annotation rail tabbed structure has a defined second tab (no `TBD`).
- All deliverables include a Code Connect map entry to the source-component path (`components/PageReader.tsx`, `components/DeckReader.tsx`, etc.) so the impl team knows what to port.
