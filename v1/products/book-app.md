# Product Spec — Book-app (`book.reeloo.ai`)

> Source path: `/home/claw/work/book-app/`
> GitHub: `rcdroid/book-app` (private)
> Live: <https://book.reeloo.ai> · status 200 at 2026-05-19 22:32 UTC
> Dev port: 3002 (`package.json:6`)
> Screenshots: `book-home.png`, `book-reader-chapter.png`

---

## 1. Identity

- **Name:** book.reeloo.ai (literal `app/layout.tsx:19`). No marketing name yet.
- **Tagline (synthesised):** *Read with translation + linguistic annotation between the lines — EPUBs you drop in, illustrated with idiom / proper-noun / place / archaism / grammar / vocab glosses, and read aloud paragraph by paragraph in two reader modes.*
- **The one user it serves today:** the operator reading English literature (LOTR, Atomic Habits) with zh-TW translation between the lines, and the operator's family reading along.
- **The user goal it advances:** *long-form reading retention* with friction-free comprehension — the gloss appears in-line so you never lose the line; audio in "deck" mode is paragraph-paced TTS that lets the eyes follow narration.

## 2. Tech stack

| Concern | Choice | Source |
|---|---|---|
| Framework | Next.js ^16.0.0 (App Router) | `package.json:13` |
| React | ^19.0.0 | `package.json:14` |
| EPUB parser | `epub2` 3.0.2 | `package.json:9` |
| Search | `flexsearch` 0.7.43 (in-memory; `forward` tokenize) | `package.json:10`, `lib/search-index.ts:8` |
| LLM | `@google-cloud/vertexai` 1.7.0 → Vertex Gemini 3 Pro Preview (fallback 2.5 Pro) | `package.json:8`, `lib/llm-runtime.ts:36` |
| Sentence segmentation | `sbd` 1.0.19 | `package.json:11` |
| TTS | `microsoft-cognitiveservices-speech-sdk` 1.50.0 — Azure Neural | `package.json:12` |
| Schema | `zod` 3.23 | `package.json:14`, `lib/annotation-schema.ts:1` |
| Auth | **None.** | (absent) |
| Database | **None.** Static JSON in `data/books/<slug>/` + GCS for audio. IndexedDB on client for uploads. | `lib/books-registry.ts:12`, `lib/indexeddb.ts`, `lib/audio-url.ts:25` |
| Style | Tailwind v4. No `tailwind.config.*`. | `package.json:23` |
| Fonts | `next/font/google` IBM Plex Sans + Plex Serif → `--font-plex-sans`, `--font-plex-serif` | `app/layout.tsx:2–16` |

**How it differs from siblings:** the only fully **static-rendered** app of the five (`export const dynamic = "force-static"` everywhere — `app/page.tsx:5`, `app/read/[slug]/page.tsx:6`, `app/read/[slug]/ch/[n]/page.tsx:9`, `app/search/page.tsx:3`). EPUB upload happens client-side: the route POSTs to `/api/import-epub`, the resulting JSON is stored in **IndexedDB** (`lib/indexeddb.ts`, called by `DropZone.tsx:30`), and the chapter route reads from disk for curated books only. Two reader modes (page / deck) toggled by `?mode=`.

## 3. Route map

```
app/
├── layout.tsx                       # font setup; sets html lang="en"
├── page.tsx                         # GET / — DropZone + curated BookCards
├── globals.css                      # token block; reading rhythm vars; deck-in keyframe
├── search/page.tsx                  # GET /search?q=&slug= — full-text via flexsearch
├── read/[slug]/page.tsx             # GET /read/<slug> — TOC for one book
├── read/[slug]/ch/[n]/page.tsx      # GET /read/<slug>/ch/<n> — chapter reader (Page or Deck mode)
└── api/
    ├── import-epub/route.ts         # POST — EPUB → JSON (tmpfile + epub2.parseEpub)
    └── annotate/route.ts            # POST — runtime LLM annotation for one paragraph
```

Per-route one-liners:

- `GET /` (`app/page.tsx:7`) — loads `data/books/*/meta.json` via `loadCuratedBooks()` (`lib/books-registry.ts:14`), renders `<DropZone>` + `<BookCard>` grid.
- `GET /read/<slug>` (`app/read/[slug]/page.tsx:7`) — loads `meta.json`, renders `<ChapterTOC>`.
- `GET /read/<slug>/ch/<n>?mode=&target=&anno=` (`app/read/[slug]/ch/[n]/page.tsx:15`) — the reader. Loads `content.json` + `annotations.<targetLang>.json` (defaults `zh-TW`, `mode=page`, `anno=on`). Mounts `<ChapterReader>` which switches between `<PageReader>` and `<DeckReader>` based on `?mode`.
- `GET /search?q=&slug=` (`app/search/page.tsx:5`) — client-side flexsearch (defaults to `lord-of-the-rings`).
- `POST /api/import-epub` (`route.ts:9`) — accepts a `File`, hashes (sha256 12-char), writes to `os.tmpdir()` (`:22`), parses via `lib/epub.ts::parseEpub`, returns `{slug:"uploaded-<sha>", meta, chapters}`. 60s `maxDuration`.
- `POST /api/annotate` (`route.ts:6`) — accepts `{paragraph, source_lang, target_lang}`, calls `annotateParagraphRuntime` (`lib/llm-runtime.ts`), 30s `maxDuration`.

## 4. Data model

No database. The "shape of a book" lives on disk under `data/books/<slug>/`:

```
data/books/<slug>/
├── meta.json                              # { title, author, source_lang, chapters: [{id, title}] }
├── index.flexsearch.json                  # serialized FlexSearch index for full-text
└── <chapterId>/                           # e.g. ch-1, ch-2 …
    ├── content.json                        # { title, paragraphs: [{id, text, sentences: [{id, text}]}] }
    └── annotations.<targetLang>.json       # AnnotationFile (see schema)
```

Annotation schema (`lib/annotation-schema.ts:24`):
```ts
AnnotationFile = {
  schema_version: 1,
  target_lang: string,         // "zh-TW" etc.
  generated_at: string,
  model: string,
  paragraphs: Record<paragraphId, {
    translation: string,
    sentence_translations?: Record<sentenceId, string>,  // populated by align-translations
    notes: { span: string, type: NoteType, gloss: string }[],
  }>
}
NoteType = "idiom" | "proper_noun" | "place" | "archaism" | "grammar" | "vocab"
```

Curated books shipped at inspection: `lord-of-the-rings`, `atomic-habits-tiny-changes-remarkable-results` (`data/books/`).

**Audio + word timings** are GCS-hosted, *not* in the repo (`lib/audio-url.ts:1–26`):
- `${NEXT_PUBLIC_BOOK_AUDIO_BASE}/<slug>/<chapterId>/<paragraphId>.mp3`
- `${NEXT_PUBLIC_BOOK_AUDIO_BASE}/<slug>/<chapterId>/<paragraphId>.timing.json`
- `${NEXT_PUBLIC_BOOK_AUDIO_BASE}/<slug>/<chapterId>/cover.jpg`
- Default bucket `https://storage.googleapis.com/reeloo-book-audio` (legacy; migration runbook noted in source).

User uploads live in **IndexedDB** (`lib/indexeddb.ts` — 56 lines). They never hit the server beyond the one-shot `parseEpub` call.

**Reader prefs** (`lib/reader-prefs.ts`): localStorage keys `book-reeloo:sentencePx` (default 38, min 14, max 72) and `book-reeloo:translationPx` (default 22, min 12, max 40). Mobile defaults 20 / 15.

## 5. Key user flows

### Flow A — Open a curated book, read in Page mode

1. `GET /` → `loadCuratedBooks()` (`books-registry.ts:14`) scans `data/books/*/meta.json`.
2. Click a `<BookCard>` → `/read/<slug>` shows `<ChapterTOC>`.
3. Click chapter → `/read/<slug>/ch/<n>` (`force-static`, `:9`).
4. Server loads `meta.json` + `content.json` + `annotations.zh-TW.json` and renders `<ChapterReader>` (`page.tsx:52`).
5. `<ChapterReader>` polls `window.location` every 200ms (`:42`) to sync `?mode` since Next's router doesn't fire popstate. Page mode default.
6. `<PageReader>` (`PageReader.tsx:61`) renders each paragraph: `renderWithNotes(p.text, notes)` greedily substitutes each note's `span` into an `<AnnotatedSpan>` (`:13–52`). Translation appears as a muted line below.

### Flow B — Switch to Deck mode (audio-paced reading)

1. User clicks `<ModeToggle>` (`components/ModeToggle.tsx`). `?mode=deck` writes to URL.
2. `<ChapterReader>` mounts `<DeckReader>` (`components/DeckReader.tsx`).
3. DeckReader fetches `audioUrl(slug, chapterId, paragraphId)` + `timingUrl(...)` (`audio-url.ts:30,38`) for the current paragraph.
4. As `<audio>` plays, `boundariesInRange()` (`lib/word-timing.ts`) highlights words in sync.
5. User adjusts text size with Aa+ / Aa- (DeckReaderControls.tsx); `writeSentencePx` / `writeTranslationPx` persist (`reader-prefs.ts`).
6. Annotation rail (`DeckAnnotationRail.tsx`) shows numbered glosses per paragraph.

### Flow C — Drop an EPUB

1. User drops a `.epub` into `<DropZone>` (`components/DropZone.tsx:9`).
2. POST `/api/import-epub` (`:18`). Server hashes, writes `os.tmpdir()/upload-<sha>.epub` (`:22`), calls `parseEpub(tmp)` (`:31`). Returns `{slug: "uploaded-<sha>", meta, chapters}`.
3. Client `saveUploadedBook(data)` to IndexedDB (`lib/indexeddb.ts`).
4. `router.push("/read/<slug>")` — but the `read/[slug]` route only knows curated paths, so the client-side reader needs to read from IndexedDB. (TODO: confirm route logic — this is the boundary between server and client reads.)

### Flow D — Runtime annotation

For uploaded books or paragraphs without baked annotations:
1. Client calls `POST /api/annotate` with `{paragraph, source_lang, target_lang}`.
2. `annotateParagraphRuntime` (`lib/llm-runtime.ts`) calls Vertex AI Gemini 3 Pro Preview, system prompt from `lib/annotation-prompt.ts`. Strips code fences (`:46`) and parses via `ParagraphAnnotationSchema`.
3. Auto-fallback to `gemini-2.5-pro` if Gemini 3 is not enabled on the Vertex SA (`llm-runtime.ts:112–119`).

### Flow E — Full-text search

1. `/search?q=ring&slug=lord-of-the-rings` (`app/search/page.tsx`).
2. `<SearchBox>` (client) loads `data/books/<slug>/index.flexsearch.json`, deserializes, runs `idx.search(query, {limit:30})` (`search-index.ts:18`).
3. Results link back into the chapter reader with a paragraph anchor.

### Flow F — Toggle annotation visibility, translation target

`?anno=0` hides notes; `?target=en|de|zh-TW` switches translation file. `<PageReader>` re-derives both client-side because the server route is `force-static` (`PageReader.tsx:66–73`).

## 6. Design tokens used

`app/globals.css`:

| Token | Value | Role |
|---|---|---|
| `--background` | `#ffffff` (light) / `#0a0a0a` (dark) | page |
| `--foreground` | `#171717` / `#ededed` | body text |
| `--anno-vocab` | `#268bd2` (Solarized blue, solid underline) | vocab notes |
| `--anno-grammar` | `#859900` (Solarized green, solid) | grammar notes |
| `--anno-idiom` | `#6c71c4` (Solarized violet, solid) | idiom notes |
| `--anno-place` | `#cb4b16` (Solarized orange, solid) | place notes |
| `--anno-archaism` | `#b58900` (Solarized yellow, **dotted**) | archaism notes |
| `--anno-proper-noun` | `#268bd2` (Solarized blue, solid) | proper-noun notes |
| `--reading-line-height` | `1.75` | paragraph rhythm |
| `--reading-max-width` | `720px` | column width |

Dark mode is `@media (prefers-color-scheme: dark)` — no `data-theme` selector; the browser drives it.

Fonts (`app/layout.tsx:2–16`): IBM Plex Sans (UI) + IBM Plex Serif (reading column — `font-serif` class applied at `PageReader.tsx:83`). Wired into Tailwind v4's `@theme inline` (`globals.css:40`).

Reading column: `max-w-5xl mx-auto p-8` outer (`app/read/[slug]/ch/[n]/page.tsx:41`); inner reading column constrained by `--reading-max-width: 720px`.

Page-mode body text: 19px (`PageReader.tsx:95` `text-[19px]`). Deck-mode sentence default 38px (mobile 20px), translation 22px (mobile 15px).

## 7. Components inventory

`components/`:

- `BookCard.tsx` — card on `/`.
- `DropZone.tsx` — drag-drop EPUB upload.
- `ChapterTOC.tsx` — chapter list.
- `ChapterNav.tsx` — prev/next chapter buttons.
- `SearchBox.tsx` — client-side flexsearch UI.
- `ModeToggle.tsx`, `LanguageToggle.tsx`, `AnnotationToggle.tsx` — three URL-driven toggles in the chapter header.
- `ChapterReader.tsx` — router between Page and Deck.
- `PageReader.tsx` — interlinear translation + inline annotations.
- `DeckReader.tsx` — paged audio reader.
- `AnnotationPopover.tsx` — `<AnnotatedSpan>` hover/tap card.

`components/deck/`:
- `DeckBackground.tsx`, `DeckHeader.tsx`, `DeckPlayerBar.tsx`, `DeckReaderControls.tsx`, `DeckAnnotationRail.tsx`, `InlineNumberedSpan.tsx`.

**Duplicates with siblings:** `AnnotationPopover.tsx` shape resembles podcast-app's annotation rail entries. `DeckPlayerBar.tsx` mirrors `podcast-app/components/KarottenPlayerV3.tsx`'s `.kt3-bar` block but is reimplemented. Merge target: shared `<MediaPlayerBar>` + shared `<AnnotationSpan>`.

## 8. External integrations

| Call | File:line | Purpose |
|---|---|---|
| Vertex AI `generateContent` | `lib/llm-runtime.ts:76` | Runtime annotation (uploads + missing zh-TW glosses) |
| Depot `${DEPOT_URL}/api/llm` | `lib/llm-build.ts:14`, `lib/emotion-tagger.ts:50,212` | Build-time annotation + emotion tagging (X-App-Token) |
| GCS `storage.googleapis.com/reeloo-book-audio` | `lib/audio-url.ts:18–25` | Static audio + word-timing + cover assets |
| `ffmpeg` / Coqui XTTS / F5-TTS scripts | `scripts/synth-audio.ts`, `scripts/synth-audio-f5.ts` | Build-time TTS (not runtime) |

**No call to `dictionary.reeloo.ai`** — annotation glosses are LLM-generated, not dictionary-sourced.

Auth model: server-side via `GOOGLE_APPLICATION_CREDENTIALS_JSON` env (Vertex SA, NEVER AI Studio per `lib/llm-runtime.ts:14`); depot gateway via `X-App-Token` from `~/.config/depot/app-token`.

## 9. Screenshots

- `v1/screens/book-home.png` — live `book.reeloo.ai/` (DropZone + book cards in light mode)
- `v1/screens/book-reader-chapter.png` — `book.reeloo.ai/read/atomic-habits-tiny-changes-remarkable-results/ch/1` (chapter reader)

Note: the `/read/atomic-habits.../` index returns 500 in production (verified by `page-shot.ts` at 22:32 UTC) — likely missing curated `meta.json` ingestion for that book. Chapter route works.

## 10. What this app does that others can't reasonably absorb

The **interlinear translation + paragraph-paced audio reader**. The combination of:
- on-disk JSON-only model (`force-static` everywhere) — no DB, deploys to Vercel as pure CDN
- IBM Plex Serif body at `--reading-line-height: 1.75` with `--reading-max-width: 720px` — actual long-form reading typography, not snippets
- two reader modes (`page` for eyes-only, `deck` for ears+eyes) toggled by URL
- 6-category note taxonomy (idiom / proper_noun / place / archaism / grammar / vocab) with distinct Solarized hues including a unique **dotted** archaism style
- per-paragraph audio + word-boundary timing JSON sidecars in GCS

…is the long-form-reading specialisation. Podcast-app's player is column-pinned for audio; book-app's PageReader is page-flowed for text. They are different physical postures.

## 11. What this app does that overlaps with others

- **Annotation underline colors** are nearly identical to podcast-app's (same vocab/grammar/idiom/place hex). Three values match exactly (`#268bd2`, `#859900`, `#6c71c4`). Book-app adds `archaism #b58900` and `proper_noun`. Merge target: one canonical Solarized-derived annotation palette.
- **DeckReader player bar** duplicates podcast-app's `.kt3-bar` (play/pause + scrub + speed). Merge target: shared `<MediaPlayerBar>`.
- **Vertex AI / depot LLM gateway plumbing** (`lib/llm-runtime.ts`, `lib/llm-build.ts`, `lib/emotion-tagger.ts`) duplicates exact patterns in notes (`scripts/dict-illustrate/`) and readalong (`src/lib/dictionaryLookup.ts`). Merge target: a shared `@reeloo/llm` package.
- **TTS pipeline** (Azure Neural here, Google Cloud TTS elsewhere) is the third independent implementation. Merge target: `audio.reeloo.ai` service.
- **Reader prefs** (`lib/reader-prefs.ts`) duplicates learn-stations's `lib/settings.ts` font-size logic. Merge target: shared `<ReaderPrefs>` hook.
