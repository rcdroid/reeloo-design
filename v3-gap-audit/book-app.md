# v3 gap audit — Book-app

> Source: `/home/claw/work/book-app/` (~7,500 LOC)
> v1 spec: `/home/claw/work/reeloo-design/v1/products/book-app.md` (228 lines)
> Impl: `/home/claw/work/reeloo-v2/components/routes/book.tsx` (346 lines)
> Headline: v1 spec is **good on data model + flows, light on Deck reader internals**. The impl ships a static prose block — zero audio sync, zero font controls, one inline annotation card, no Drop-Zone, no Search, no Page-vs-Deck mode toggle.

---

## 1. Feature inventory (real app)

### Surface: library (`/`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 1 | Curated book grid: reads `data/books/*/meta.json` server-side via `loadCuratedBooks()`; ships LOTR + Atomic Habits | `app/page.tsx:7`, `lib/books-registry.ts:14` | atomic |
| 2 | `<DropZone>` drag-drop EPUB upload; client posts to `/api/import-epub`; result lands in IndexedDB | `components/DropZone.tsx` (80 lines), `lib/indexeddb.ts` | multi-step flow |
| 3 | `<BookCard>` linking to `/read/<slug>` with cover + meta | `components/BookCard.tsx` (21 lines) | atomic |
| 4 | Locally-uploaded books accessible (`uploaded-<sha>` slug) via IndexedDB read | `lib/indexeddb.ts`, `app/page.tsx` | composed |

### Surface: TOC (`/read/<slug>`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 5 | `<ChapterTOC>` — chapter list with title + chapter id | `components/ChapterTOC.tsx` (20 lines) | atomic |

### Surface: chapter reader (`/read/<slug>/ch/<n>?mode=&target=&anno=`)

The heart of the app. Three URL-driven toggles + two reader modes.

| # | Feature | Source | Tier |
|---|---|---|---|
| 6 | URL-driven `?mode=page|deck` toggle (default `page`); window-location polling every 200ms (Next.js popstate workaround) | `app/read/[slug]/ch/[n]/page.tsx:15`, `ChapterReader.tsx:42` | composed |
| 7 | URL-driven `?target=zh-TW|en|de` translation-language toggle | `LanguageToggle.tsx` (45 lines) | atomic |
| 8 | URL-driven `?anno=on|off` annotation visibility toggle | `AnnotationToggle.tsx` (38 lines) | atomic |
| 9 | **Page reader** — flow-paginated paragraphs, IBM Plex Serif body 19px, `--reading-line-height: 1.75`, `--reading-max-width: 720px` | `PageReader.tsx` (111 lines), `globals.css` | composed |
| 10 | Interlinear translation under each paragraph (muted, font-sans 16px) | `PageReader.tsx:96-99` | atomic |
| 11 | Annotation rendering with `<AnnotatedSpan>` greedy span replacement; supports overlap detection | `PageReader.tsx:13-52`, `AnnotationPopover.tsx` (99 lines) | composed |
| 12 | **6-category annotation taxonomy**: `idiom / proper_noun / place / archaism / grammar / vocab` — each with distinct Solarized hue; `archaism` is **dotted** (the only dotted underline in the family) | `lib/annotation-schema.ts:4`, `globals.css:152-160` | atomic |
| 13 | Hover/tap popover `<AnnotationPopover>` shows gloss + type label | `AnnotationPopover.tsx` | composed |
| 14 | **Deck reader** — paged audio-paced reader (770 lines) | `DeckReader.tsx` | composed |
| 15 | Per-paragraph audio fetch from GCS: `https://storage.googleapis.com/reeloo-book-audio/<slug>/<ch>/<paraId>.mp3` | `lib/audio-url.ts:18-25` | atomic |
| 16 | Word-boundary timing JSON sidecar `<paraId>.timing.json`; `boundariesInRange()` drives word-level highlight | `lib/word-timing.ts`, `DeckReader.tsx:79-80` | composed |
| 17 | Active-word `<span class="deck-word-active">` with high-contrast background designed to read over AI cover images | `DeckReader.tsx:158-180` | atomic |
| 18 | Adjustable **sentence font size** (default 38px desktop / 20px mobile, range 14-72) persisted to `book-reeloo:sentencePx` localStorage | `lib/reader-prefs.ts`, `DeckReader.tsx:244` | atomic |
| 19 | Adjustable **translation font size** (default 22px desktop / 15px mobile, range 12-40) persisted to `book-reeloo:translationPx` | `lib/reader-prefs.ts`, `DeckReader.tsx:245` | atomic |
| 20 | **Mobile viewport detection** at `isMobile` state, reflows floating UI (player bar position, Aa controls auto-hide, rail handle width) | `DeckReader.tsx:251` | composed |
| 21 | `<DeckPlayerBar>` — play/pause + scrub + speed (1.0× default) + paragraph cycle toggle | `components/deck/DeckPlayerBar.tsx` (276 lines) | composed |
| 22 | **Per-paragraph cycle** toggle — auto-replays the current paragraph after end | `DeckReader.tsx:235` (`cycle`) | atomic |
| 23 | **Annotation rail** with 2 tabs (`notes`, ?) — `<DeckAnnotationRail>` (314 lines) | `components/deck/DeckAnnotationRail.tsx`, `DeckReader.tsx:230-231` | composed |
| 24 | **Rail-open layout shift** — main reader area re-centers in remaining viewport when rail opens (avoids visual jump) | `DeckReader.tsx:213-219` (RAIL_WIDTH_PX = 380) | composed |
| 25 | **`<InlineNumberedSpan>`** — annotations get a small superscript number that maps to the rail entry | `components/deck/InlineNumberedSpan.tsx` (110 lines), `DeckReader.tsx:194-204` | atomic |
| 26 | Per-sentence timing windows — clip windows so audio plays through but visual deck advances per sentence (`fetchTimingWithRetry`) | `DeckReader.tsx:385-455` | composed |
| 27 | Keyboard shortcuts: Space/play, arrows (per `useEffect` at `:352-365`) | `DeckReader.tsx:352-365` | atomic |
| 28 | Paragraph timing retry with exponential backoff (network-tolerant) | `DeckReader.tsx:407-455` | atomic |
| 29 | `<ChapterNav>` prev/next chapter buttons | `components/ChapterNav.tsx` (43 lines) | atomic |
| 30 | `<DeckBackground>` — book-cover-derived background art for the deck mode (146 lines, image blur/scale logic) | `components/deck/DeckBackground.tsx` | composed |
| 31 | `<DeckHeader>` — chapter title + close + nav (125 lines) | `components/deck/DeckHeader.tsx` | composed |
| 32 | `<DeckReaderControls>` — Aa+/Aa- font controls, auto-hide timer (157 lines) | `components/deck/DeckReaderControls.tsx` | composed |

### Surface: search (`/search?q=&slug=`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 33 | Client-side **FlexSearch** full-text over the deserialized per-book index `data/books/<slug>/index.flexsearch.json` | `components/SearchBox.tsx` (112 lines), `lib/search-index.ts:18` | composed |
| 34 | Result links back to chapter + paragraph anchor | `SearchBox.tsx` | atomic |
| 35 | Default slug `lord-of-the-rings` when none provided | `app/search/page.tsx:5` | atomic |

### Surface: runtime annotation API

| # | Feature | Source | Tier |
|---|---|---|---|
| 36 | `POST /api/annotate` — Vertex Gemini 3 Pro Preview annotation; auto-fallback to Gemini 2.5 Pro; system prompt from `lib/annotation-prompt.ts`; 30s `maxDuration` | `lib/llm-runtime.ts:36, 76, 112-119`, `route.ts:6` | multi-step flow |
| 37 | Code-fence stripping + zod validation via `ParagraphAnnotationSchema` | `lib/llm-runtime.ts:46` | atomic |
| 38 | `POST /api/import-epub` — sha256 hash, write to `os.tmpdir()`, parse via `epub2.parseEpub`, return `{slug, meta, chapters}` JSON; 60s `maxDuration` | `app/api/import-epub/route.ts:9`, `lib/epub.ts` | multi-step flow |

### Surface: theming + a11y

| # | Feature | Source | Tier |
|---|---|---|---|
| 39 | Light/dark via `@media (prefers-color-scheme: dark)` — browser-driven, no toggle | `globals.css` | atomic |
| 40 | Tailwind v4 `@theme inline` token registration | `globals.css:40` | atomic |
| 41 | Two-font system: IBM Plex Sans (UI) + IBM Plex Serif (reading column) | `app/layout.tsx:2-16`, `PageReader.tsx:83` | atomic |

---

## 2. What v1 spec captured

v1 §1-§11 captured **all of the routes, the 6-category note taxonomy, the IndexedDB upload story, the GCS audio + timing.json contract, the Vertex AI annotation flow, the two reader modes, and the reading-prefs localStorage keys.** Specifically:

- §3 (lines 37-59): full route map + per-route validation
- §4 (lines 61-100): `AnnotationFile` zod shape + on-disk layout + the `book-reeloo:sentencePx/translationPx` localStorage keys
- §5 (lines 102-144): all six flows — Page mode, Deck mode, EPUB drop, runtime annotation, full-text search, toggle behaviours
- §6 (lines 146-169): the **dotted** archaism style, the `--reading-line-height: 1.75`, the 720px column, the 38/20 + 22/15 font defaults
- §7 (lines 171-189): components inventory with all 16 files

v1 was thorough on this app.

---

## 3. What v1 spec MISSED

- **Deck reader two-tab rail** (`notes` + ?) — `RailTab` enum (`DeckReader.tsx:231`) implies a second tab exists. v1 mentions a "rail" but not the tabbed structure or what the second tab contains. (need spec)
- **`<InlineNumberedSpan>` cross-reference** — annotations get a superscript number tying them to numbered rail entries. v1 mentions rail-numbered glosses but doesn't specify the typography or accessibility (the number must be readable at body size). (`DeckReader.tsx:317-329`)
- **Per-paragraph cycle toggle** — auto-replay of the current paragraph. v1 doesn't surface this; it's a real feature with a UI affordance. (`DeckReader.tsx:235`)
- **DeckBackground** book-cover-derived art — the deck reader has a background image derived from the book cover with blur. Distinct from the prose surface; needs a design treatment for AI-generated covers vs commissioned art. (`DeckBackground.tsx`)
- **High-contrast active-word styling over AI cover images** — there's an explicit code comment about this concern: "High-contrast highlight that's visible over busy background images; on the active word the bright bg makes it…neighbouring glyphs never shift when the highlight steps" (`DeckReader.tsx:158-180`). Real visual problem; needs design.
- **Mobile viewport reflow** — Aa controls auto-hide, player bar reposition, rail handle width. v1 mentions mobile defaults (20/15) but not the reflow behaviour itself. (`DeckReader.tsx:251`)
- **Timing-fetch retry with exponential backoff** — graceful degradation when GCS sidecar is slow. UX state for "loading audio…" between paragraphs.
- **`<SearchBox>` UX detail** — 30-result limit, prefix vs full-token matching, "no results" empty state. v1 mentions search exists but doesn't define the empty state or the result-row design.
- **`DeckPlayerBar` mobile layout** — the bar is 276 lines, mostly responsive layout. Worth a mobile mock.
- **EPUB-upload error states** — what happens when the EPUB is malformed, too large, or `parseEpub` throws? Currently no UX defined.
- **Runtime-annotation latency UX** — 30s Vertex call. The spec doesn't say whether the user sees a spinner per-paragraph or whether annotations populate progressively. (`lib/llm-runtime.ts`)
- **The "uploaded-<sha>" boundary** — v1 §5 Flow C ends with "TODO: confirm route logic — this is the boundary between server and client reads." This is unresolved and design needs to know.

---

## 4. What the impl skipped (the BIG gap)

Against `reeloo-v2/components/routes/book.tsx` (346 lines):

| v1-spec'd feature | In impl? |
|---|---|
| `<DropZone>` EPUB upload | NO — just a stub `<button>Drop a book</button>` |
| Two reader modes (Page vs Deck) toggle | NO — one prose layout only |
| URL-driven `?mode=` / `?target=` / `?anno=` | NO |
| `<PageReader>` with interlinear translation | NO — chapter prose is a 3-paragraph hardcoded array |
| `<DeckReader>` paged audio | NO |
| Per-paragraph audio + word-boundary highlight | NO |
| Adjustable font sizes (Aa+/Aa-) | NO |
| 6-category annotation taxonomy with distinct colors | NO — one inline span class `rb-anno`, single style |
| Annotation popover (hover/tap) | NO — only static text rendered |
| Annotation rail with numbered cross-references | PARTIAL — a rail exists with 2 hardcoded entries; no numbered span linking |
| `<DeckPlayerBar>` (play/pause/scrub/speed/cycle) | NO |
| `<DeckBackground>` cover-derived art | NO |
| `<DeckHeader>` + auto-hiding controls | NO |
| Full-text search via FlexSearch | NO — no `/search` route |
| `POST /api/annotate` runtime annotation | NO |
| `POST /api/import-epub` | NO |
| IndexedDB-backed uploaded books | NO |
| Chapter navigation (prev/next) | YES — but no chapter list lookup, just `parseInt + 1` |
| `<ChapterTOC>` | PARTIAL — `BookTOC` exists with hardcoded chapters |
| Mobile viewport reflow | NO — no `isMobile` detection |
| Dark mode | PARTIAL — comes from theme tokens, not from per-app `prefers-color-scheme` |
| Keyboard shortcuts | NO |

**Score: ~3 of 32 features ported. The impl is essentially a static landing page wearing a book costume.**

---

## 5. What's actually pretty good

- The library-card grid + cover-gradient styling is a clean foundation; level chips integrate naturally.
- The `<BookTOC>` row layout (chapter number + title + fit chip + minutes) is good IA — preserves the v1 chapter-listing semantics.
- The "Export chapter" CTA is placed correctly in the chapter header and has a stable `data-testid="book-reader:make-video-button"`.
- The annotation rail's row structure (heading + type label + gloss + Add-to-SRS button) is the right shape; it just needs to be wired to real annotation data and a numbered-span hover state.
- Test-id `book-reader:reading-column` and `book-reader:annotation-rail` are reserved correctly.
