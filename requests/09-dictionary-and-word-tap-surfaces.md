# Design request: Dictionary popups + word-tap UX across every text surface

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v2/06` §3.3 (names `dictionary.reeloo.ai/api/dictionary` as a service)
- `v3-gap-audit/cross-cutting.md` §1 (the full theme)
- Source code: 5 readalong dictionary components (`DictionaryPanel.tsx`, `DictionaryLookup.tsx`, `DictionaryCard.tsx`, `AnnotationPanel.tsx`, `AnnotatedText.tsx`), `book-app/components/AnnotationPopover.tsx`, `podcast-app/components/KarottenPlayerV3.tsx:807-810` (rail toggle), `learn-stations/app/cards/` dict-link

## What we asked for in v2 (and what we got)

The v2 brief named `<DictionaryCard>` (§7.1) and `<AnnotatedSpan>` as the canonical components, with `dictionary.reeloo.ai/api/dictionary?word=&lang=` as the service. Claude design produced no dictionary-card mockup, no popover spec, no OOV fallback, no save-to-SRS button affordance, no per-surface variant.

Result: every app rolls its own. podcast has a rail-entry style, book has a tap popover, readalong has 5 variants, learn-stations has a dict-link icon. The unification the v2 brief promised did not happen in the design.

## What's actually in the source apps

| Surface | Trigger | Form-factor | Source |
|---|---|---|---|
| podcast player word-span | tap word | annotation rail entry (no live lookup; baked-in) | `KarottenPlayerV3.tsx:179, 807-810` |
| podcast rail entry | tap entry | seek + scroll | `KarottenPlayerV3.tsx:807` |
| book PageReader span | tap/hover | `<AnnotationPopover>` (99 LOC) | `book-app/components/AnnotationPopover.tsx` |
| book DeckReader active word | tap | rail card with numbered cross-ref | `book-app/components/deck/DeckAnnotationRail.tsx` |
| learn-stations vocab-intro | mounted | full inline card | `components/stations/vocab-intro.tsx` |
| learn-stations cards back | `cards-dict-link` icon | navigate to dictionary surface | `app/cards/CardsDeck.tsx` |
| readalong slide word | tap | `<DictionaryCard>` overlay | `readalong/src/components/DictionaryCard.tsx` |
| readalong drill slot | tap word | `<DictionaryLookup>` modal | `readalong/src/components/DictionaryLookup.tsx` |
| readalong scenario dialogue | tap | `<DictionaryCard>` overlay | (same) |
| readalong word-rain | match | `<MatchedWordFlash>` + brief dict flash | `WordRainGame.tsx:73` |
| notes/practice/flashcards | mounted | inline panel below illustration | `app/practice/flashcards/page.tsx` |

The canonical dictionary service is `dictionary.reeloo.ai/api/dictionary?word=&lang=` (per memory `reference_shared_dictionary_api.md`). 10K+ entries per language (DE/EN/JA), file-based JSON, schema with TTS audio + CEFR + inflections + JP pitch.

Dictionary entry shape (per `notes/app/api/dictionary-illustration/random/route.ts`):
```ts
interface DictEntry {
  word: string;
  readings?: string[];          // pronunciation; furigana for JP
  partOfSpeech?: string;
  level?: string;                // CEFR or JLPT
  senses?: {
    definitions?: {text, lang?}[];
    examples?: {text, translation, reading}[];
  }[];
}
```

Plus `inflections`, `pitchAccent` (JA), `genderArticles` (DE), `audio_url` per the canonical schema.

## What we need from Claude design

### 1. `<DictionaryCard>` master component

One card with all states + variants. Layout:

```
┌─────────────────────────────────────┐
│ word                ▶ play  ⊕ save  │  ← header
│ reading · POS · CEFR                 │
├─────────────────────────────────────┤
│ Definition 1                         │  ← senses
│ • example sentence ·  translation    │
│ Definition 2                         │
├─────────────────────────────────────┤
│ Inflections (collapsed by default)   │  ← progressive disclosure
│ Pitch / Gender (lang-specific)       │
├─────────────────────────────────────┤
│ "Saved from Karotten · 2d ago"       │  ← provenance (when present)
└─────────────────────────────────────┘
```

- All four sections (header, senses, lang-specific, provenance) are independently optional.
- The header `⊕ save` is the canonical save-to-SRS button (cross-cutting §4).
- The `▶ play` button hits the canonical TTS pipeline.

### 2. Form-factor variants

The same `<DictionaryCard>` data in five visual modes:

- **Popover** — anchored above a tapped word (book PageReader, readalong slide). 320px wide. Auto-positions above/below based on viewport.
- **Rail entry** — narrower (~340px), no provenance row, fits the 360–380px rail of book/podcast.
- **Inline panel** — full-width within a column, no shadow (learn-stations vocab-intro).
- **Modal overlay** — centered, 480px wide, with backdrop (readalong drill slot).
- **Flash** — minimal: word + reading + one definition; auto-dismiss in 2s (word-rain match).
- **Standalone page** — `/dictionary/<word>` deep-link: full card + related-words section + back-button. Already exists in readalong (`/dictionary/[word]/page.tsx`).

### 3. Loading state

Dictionary lookup is a network call (5–200ms). Design:
- Skeleton state: ghost rows in the senses area, word + reading already filled from the click context.
- Spinner: small, in the header.
- 1000ms+ timeout: "Still loading… use a different word?" affordance.

### 4. OOV fallback

When the dictionary doesn't have the word:
- "We don't know this word yet"
- "Try a simpler form?" (e.g. for German verbs, show the infinitive lookup)
- `<FlagIssue>` button — "Tell us about this word" (auto-fills the surface + the word as the issue body)
- Gemini fallback — option: try a runtime LLM lookup. Design specifies whether this is automatic, on-demand ("Try LLM lookup"), or behind a setting.

### 5. POS / kind tagging in the header

The POS chip in the header uses the same color taxonomy as annotation underlines (per `globals.css:45-49`): place / subject / object / verb / particle. For non-JP languages: `noun (m/f/n)` / `verb` / `adj` / `adv` etc. Each gets a hue.

### 6. JP-specific affordances

- Furigana over kanji in the word + examples
- Pitch-accent pattern (drawn or notated, e.g. `[2]` for atamadaka)
- Romaji toggle (global setting via the `cards-autotts-toggle` partner, `useShowRomaji`)

### 7. DE-specific affordances

- Gender + article (`der/die/das`)
- Inflections collapsible (Nominativ / Akkusativ / Dativ / Genitiv)
- For verbs: principal parts (e.g. `gehen, ging, gegangen`)

### 8. Save-to-SRS — the cross-cutting affordance

The `⊕ save` button writes to the user's SRS queue with the provenance attached (which surface + which word + which sentence context). Visual: toggles to a filled ✓ + bounce + toast "Saved — review in 5min."

### 9. Dictionary-card chrome consistency

Every variant uses the same:
- Word typography (24px serif, weight 600)
- Reading typography (14px sans, italic, muted)
- POS chip color tokens
- Save button position (always top-right)
- Audio play button position (always next to word)

### 10. Mobile

- Popover variant becomes a bottom sheet (tap-on-word opens sheet, swipe-down dismisses)
- Modal variant becomes a full-screen takeover with a back arrow
- Rail variant: rail collapses to a bottom sheet on small viewports (overrides the 380px rail)

### 11. Accessibility

- Popover: focus-trap, ESC dismisses, return focus to the source word
- Audio: `aria-label="Play pronunciation of <word>"`
- POS chip: `aria-label="Part of speech: noun"` since the color alone shouldn't carry meaning

### 12. Per-surface integration spec

For each of the 11 source-app surfaces (table above), the design must declare:
- Which form-factor variant fires
- What the trigger affordance looks like (the word's hover state, the rail entry's tap target)
- Where the save-to-SRS write lands (always the shared queue)

## Out of scope

- The dictionary service backend (`dictionary.reeloo.ai`) — canonical, no change.
- The dictionary data (how words get added) — content team.
- The SRS storage model — see cross-cutting §4.
- The LoRA illustration pipeline (covered in request 05).

## Success criteria

- One `<DictionaryCard>` data component with five visual variants.
- Loading + OOV + error states all designed.
- POS-color taxonomy unified with annotation-underline taxonomy (request 02's six kinds inform this).
- JP and DE language-specific affordances designed.
- Save-to-SRS works the same way from every surface.
- Mobile variants for popover, modal, rail-as-sheet.
- Per-surface integration table documented in the design system.
- WCAG AA in all three themes.
