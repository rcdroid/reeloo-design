# Proposed Unified Design Tokens — sourced

> Every value below has a **source app**. If a value is INVENTED, it's labelled and justified in one sentence.

---

## Color

### Primary palette (Solarized-derived)

| Token | Light | Dark | Source |
|---|---|---|---|
| `--color-bg` | `#fdf6e3` (Solarized base3) | `#0f1419` (cobalt) | Light from `learn-stations/app/globals.css:58` + `podcast-app/KarottenPlayerV3.tsx:1461`. Dark from `podcast-app/app/page.tsx:188` (linear-gradient base) — sharper cinematic feel than Solarized base03; recommend over `#002b36` |
| `--color-bg-container` | `#ffffff` | `#1a2332` | Light: `learn-stations:59`. Dark: `podcast-app/app/page.tsx:188` (gradient endpoint) |
| `--color-bg-elevated` | `#fffcf0` | `#0e4856` | `learn-stations:60` light; `learn-stations:212` dark (one step up from container) |
| `--color-text` | `#073642` | `#e8eef5` | Light: `learn-stations:63` (base02). Dark: `podcast-app/app/page.tsx:189` |
| `--color-text-muted` | `#586e75` | `#a8b8c8` | Light: `learn-stations:64`. Dark: `podcast-app/app/page.tsx:223` |
| `--color-text-subtle` | `#657b83` | `#6e8aa8` | Light: `learn-stations:65`. Dark: `podcast-app/app/page.tsx:201` |
| `--color-primary` | `#cb4b16` | `#cb4b16` | Solarized orange. Universal across podcast/book/learn-stations |
| `--color-on-primary` | `#ffffff` | `#ffffff` | `learn-stations:69`; AA verified |
| `--color-primary-container` | `#fdebd9` | `#4a200e` | `learn-stations:70` light; `:220` dark. The latter is the "the token that fixes the listen-mc dark-theme bug" per source |
| `--color-on-primary-container` | `#7a2810` | `#fdebd9` | `learn-stations:71`; inverted for dark |
| `--color-border` | `#8a8670` | `#7a8e93` | `learn-stations:74` light (warm grey hairline; 3.40:1 AA UI); `:223` dark |
| `--color-border-strong` | `#cb4b16` | `#cb4b16` | `learn-stations:75`; orange used for focused/selected |

### Annotation palette (semantic underline colors)

| Token | Hex | Underline style | Source |
|---|---|---|---|
| `--anno-vocab` | `#268bd2` (Solarized blue) | solid | podcast-app `KarottenPlayerV3.tsx:1474`, book-app `globals.css:8` (agree) |
| `--anno-grammar` | `#859900` (Solarized green) | solid | both apps (agree) |
| `--anno-idiom` | `#6c71c4` (Solarized violet) | solid | both apps (agree) |
| `--anno-cultural` | `#cb4b16` (Solarized orange) | solid | **Disputed slot.** podcast-app calls it `cultural` (`KarottenPlayerV3.tsx:1476`); book-app calls the same color `place` (`globals.css:11`). Recommend keeping both names mapped to the same token; treat `place` as a subtype of `cultural` |
| `--anno-archaism` | `#b58900` (Solarized yellow) | **dotted** | book-app only (`globals.css:12`). Useful — keep for book-app, no-op elsewhere |
| `--anno-proper-noun` | `#268bd2` (Solarized blue) | solid | book-app `globals.css:13`; alias to `--anno-vocab` |

### State colors (graded feedback)

All from `learn-stations/app/globals.css:21–35`, WCAG-AA verified by `lib/design-tokens.test.ts` (the only app that has these formalised):

| Token | Hex | Notes |
|---|---|---|
| `--correct-bg / fg / ring` | `#eaf3c2 / #2d3f00 / #4a5d00` | 9.9:1 / 6.8:1 |
| `--correct-solid-bg / fg` | `#4a5d00 / #ffffff` | 7.4:1 |
| `--wrong-bg / fg / ring` | `#fbdcd9 / #6e0a08 / #a01818` | 9.6:1 / 5.6:1 |
| `--wrong-solid-bg / fg` | `#a01818 / #ffffff` | 7.9:1 |
| `--warning-bg / fg / ring` | `#fdf3cf / #5c4400 / #b58900` | 9.7:1 |

### POS coloring (used by language-grammar surfaces)

From `learn-stations/app/globals.css:45–49`:

| Token | Hex | Role |
|---|---|---|
| `--pos-place` | `#cb4b16` | spatial anchor |
| `--pos-subject` | `#268bd2` | the thing |
| `--pos-object` | `#d33682` (Solarized magenta) | the affected |
| `--pos-verb` | `#6b7a00` | the action |
| `--pos-particle` | `#93a1a1` | glue |

### TTS karaoke highlight

| Token | Hex | Source |
|---|---|---|
| `--tts-highlight-bg` | `#daa520` (goldenrod) | `learn-stations:12` — bumped from `#b58900` to clear AA |
| `--tts-highlight-fg` | `#073642` | `learn-stations:13` |
| `--tts-highlight-ring` | `#cb4b16` | `learn-stations:14` |

### Dropped values (do NOT use)

- `#4f46e5` indigo from notes flashcards (`page.tsx:147`) — INVENTED fallback, not part of the system. Drop.
- readalong's 7 named themes (`ocean-light`, `forest-light`, `lavender-light`, `midnight-dark`, `emerald-dark`, `amethyst-dark`, `rose-dark`) — shadcn-default cosmetic variants. Recommend dropping all 7 in the unified app; expose only the canonical light + cobalt-dark + high-contrast.
- learn-stations's `pip-duo` Duolingo-green theme (`globals.css:149–163`) — kept as an opt-in for the kid-friendly mode if we ship one, otherwise drop.

---

## Typography

### Font stack

| Role | Family | Source |
|---|---|---|
| Sans | **IBM Plex Sans** | podcast-app `layout.tsx:8`, book-app `layout.tsx:4`. (Drop Geist Sans from learn-stations, system-UI from readalong.) |
| Serif | **IBM Plex Serif** | podcast-app `layout.tsx:14`, book-app `layout.tsx:10`. (Drop Lora from learn-stations.) |
| Mono | **IBM Plex Mono** | podcast-app `layout.tsx:21`. (Drop Geist Mono from learn-stations.) |
| CJK | **Noto Sans TC** | all four (universal). Weights 400 / 500 / 600 / 700 |
| JP serif | **Noto Serif JP** | learn-stations only — needed for furigana over kanji; keep as opt-in by language |

All loaded via `next/font/google` with `display: swap` and bound to CSS variables.

### Scale (from observed source)

| Token | Px | Role | Source |
|---|---|---|---|
| `--font-size-xs` | 11 | meta chips | podcast `.pl-card-meta` `CatalogClient.tsx:300` |
| `--font-size-sm` | 13 | secondary body | podcast `.pl-card-desc` `CatalogClient.tsx:306` |
| `--font-size-base` | 15 | body | podcast `.pl-card-title` `CatalogClient.tsx:293`, lede |
| `--font-size-md` | 19 | reading column | book-app `PageReader.tsx:95` |
| `--font-size-lg` | 22 | translation deck | book-app `reader-prefs.ts:14` |
| `--font-size-xl` | 30 | hero h1 | podcast `app/page.tsx:214` |
| `--font-size-2xl` | 34 | focus-pin active line | podcast `KarottenPlayerV3.tsx:27` |
| `--font-size-3xl` | 38 | reading column deck mode | book-app `reader-prefs.ts:13` |

Headings use Plex Serif (book-app reader, podcast active-line); body + UI use Plex Sans.

### Tracking + line-height

| Token | Value | Source |
|---|---|---|
| `--reading-line-height` | `1.75` | book-app `globals.css:16` |
| `--ui-tracking-uppercase` | `0.06em – 0.10em` | podcast pills + section headers (`CatalogClient.tsx:303` `0.06em`, `.pl-section-h` `0.10em`) |
| `--heading-tracking-tight` | `-0.01em` | learn-stations `globals.css:104` |

---

## Spacing

Inherit Tailwind's 4px-base scale. Specific tokens worth naming:

| Token | Value | Source |
|---|---|---|
| `--space-page-pad` | `32px` (mobile) → `48px` (sm+) | podcast `app/page.tsx:193, 228` |
| `--space-card-pad` | `14px – 24px` | podcast `.pl-card-body` 14–16px; flashcards 24px (`page.tsx:181`) |
| `--space-section-gap` | `32 – 56px` | podcast catalog hero→sections 36px; footer 56px |
| `--space-touch-target` | `44 – 48px` min-height | podcast `.pl-lang` 44px (`CatalogClient.tsx:199`); flashcards Next button 48px (`page.tsx:301`) |
| `--reading-max-width` | `720px` | book-app `globals.css:18` |
| `--catalog-max-width` | `1000px` | podcast `app/page.tsx:193` |

---

## Radius

| Token | Value | Source |
|---|---|---|
| `--radius-sm` | `4px` | code chip in 404 view (`podcast-app/podcast/[slug]/page.tsx:149`) |
| `--radius-md` | `8px` | learn-stations cards `rounded-2xl` (16px) ≠ this — use `8px` for buttons + small inputs |
| `--radius-lg` | `12px – 14px` | book-app DropZone 8–12px; podcast card 14px `CatalogClient.tsx:250` |
| `--radius-xl` | `16px` | learn-stations course card `rounded-2xl` |
| `--radius-pill` | `999px` | podcast language pill `CatalogClient.tsx:193`, learn-stations course chip |

---

## Shadow

Two consistent values across light surfaces:

| Token | Value | Source |
|---|---|---|
| `--shadow-card` | `0 1px 2px rgba(0,0,0,0.04)` | notes flashcards `page.tsx:183` |
| `--shadow-elevated` | `0 1px 2px rgba(0,0,0,0.04), 0 4px 12px rgba(0,0,0,0.06)` | INVENTED — extension of `--shadow-card` for raised cards. Justification: no app has both layers today; unified product needs a "raised" variant for the deck-reader card-mode entrance |
| `--shadow-focus` | `0 0 0 2px var(--color-border-strong)` | INVENTED but the colors are sourced; needed for keyboard focus rings (currently each app does it differently) |

---

## Motion

From `podcast-app/components/KarottenPlayerV3.tsx:1482–1484`:

| Token | Value | Role |
|---|---|---|
| `--dur-fast` | `120ms` | micro-interactions (hover, focus) |
| `--dur-med` | `240ms` | panel slide, mode toggle |
| `--dur-slow` | `460ms` | column transform (focus-pin), deck-in keyframe |
| `--ease-out` | `cubic-bezier(0.22, 1, 0.36, 1)` | standard easing |

Book-app's `deck-in` animation (`globals.css:47–56`) uses 460ms — matches `--dur-slow`. Keep this aligned.

---

## Theme strategy

Adopt **learn-stations's pattern**:
1. Set theme via `document.documentElement.dataset.theme` (allowed values: `vanilla` (default light), `cobalt-dark`, `high-contrast`, `pip-duo` (opt-in)).
2. Pre-hydration `<script>` reads `localStorage.getItem('reeloo.theme')` and applies before React mounts — see `learn-stations/app/layout.tsx:84–89` for the canonical implementation.
3. Each theme defines the same role tokens; components reference roles only (no hex literals).

Drop:
- Readalong's `next-themes` `<ThemeProvider>` (lazy, post-hydration) — replace with the inline script.
- Book-app's `prefers-color-scheme` only — replace with `data-theme` + system fallback.
- Podcast-app's component-local `data-theme="solarized-light"` static attribute — promote to the html-level variable.

---

## Dark mode for the audio surfaces (special case)

Podcast-app's `#0f1419 → #1a2332` cobalt feels distinct and right for a "listen-along" surface. **Recommendation:** make `cobalt-dark` the dark theme. Solarized-dark (`#002b36`) is an *alternative* for users who want the unified light-and-dark Solarized identity, but not the default.

---

## Open token-system questions for Claude design

1. Should `--anno-cultural` and `--anno-place` collapse into one slot (Solarized orange), or stay as two separately-named slots that happen to share a hex?
2. Should the unified app keep four themes (vanilla / cobalt-dark / high-contrast / pip-duo) or three (drop pip-duo)?
3. Light-mode text on `--color-bg-elevated #fffcf0`: should we tighten to `#073642` always, or allow `--color-text-muted #586e75` for secondary content? (learn-stations does both.)
4. Should we ship a separate `--color-bg-podcast` variant for the podcast surface only (cobalt-cinematic) and force light surfaces elsewhere, OR ship one cobalt-dark applicable everywhere?
5. The `glow-pulse` animation from readalong (`globals.css:32–56`) — keep as a celebration token, or replace with a simpler `scale + opacity` motif?
