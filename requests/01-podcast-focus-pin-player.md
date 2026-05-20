# Design request: Podcast focus-pin player — full interaction inventory

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/podcast.md` §5–§6 (the existing partial spec)
- `v3-gap-audit/podcast.md` §1 (features 8–29), §4 (impl gaps)
- Source code: `/home/claw/work/podcast-app/components/KarottenPlayerV3.tsx` (2,442 lines)

## What we asked for in v2 (and what we got)

The v2 brief named `<FocusPinColumn>`, `<WordKaraoke>`, `<AnnotationRail>`, `<MediaPlayerBar>` as shared components in §7.1 and gestured at the 42% pin / Plex Serif / cobalt-dark theme via the v1 spec it linked to. Claude design delivered `reeloo.css` with a `.rp-focus` block + an `.rp-annotation` card inline, no rail, no audio element, no per-word click, no transform math, no Tweaks panel, no pre-teach modal, no keyboard shortcuts, no MediaSession, no Recent row, no ProgressModal, no PodcastPlayerV1 fallback, no KarottenVideoPlayer variant, no wizard.

The impl at `reeloo-v2/components/routes/podcast.tsx` is 347 lines vs the source's 2,442 — it has the SHAPE of a podcast player but ships none of the behaviours.

## What's actually in the source app

The `KarottenPlayerV3` is one of the most-iterated surfaces in the codebase (per the file's own header comment, "Option 3" of the player-design sequence). Specifically:

1. **Focus-pin transform** — active segment translated to 42% viewport height via `translateY(focusY - segCenter)` using `offsetTop` (not bounding-rect — that was a previous bug; see `:11`). Recomputed on `[activeIdx, totalMs, railOpen, isMobile, tweaks.theme]`. 460ms ease-out cubic-bezier. Source: `KarottenPlayerV3.tsx:579-606, 827`.
2. **Typography** — IBM Plex Serif 34px/700 for the active line; neighbouring segments fade to 18% opacity, ±1 to 42%. Source: header comment `:25-32`.
3. **Word karaoke** — each segment's words render as `<span data-kind=...>`, indexed via `annotatedWords()` (`:150-178`) to mark which words carry annotations. `findActive(segments, ms)` (`:128-143`) is a binary search; `findAnnotationWordIdx(seg, phrase)` (`:179`) resolves a rail-click to a specific word. Click → `seekToWord(i, wi)` (`:657-695`) sets `audio.currentTime`.
4. **Annotation rail** — 360px right column with collapse toggle (`<button class="kt3-rail-toggle">` at `:807-810`). Underlines colored by kind: `vocab #268bd2 / grammar #859900 / cultural #cb4b16 / idiom #6c71c4` (`:1474-1477`). Rail entries listed with kind label, gloss, POS, gender, CEFR; click → seek.
5. **Pre-teach vocab modal** — gated by `≥10 annotations across segments AND first-time-for-this-slug` (`VOCAB_PRETEACH_COUNT = 10` at `:95`). Persists to `localStorage["reeloo_podcast_vocab_seen_v1:<slug>"]`. Modal renders the first 10 vocab items as a grid. Component: `PreTeachVocabModal` at `:1176-1335`.
6. **Tweaks panel** — gear icon opens `<TweaksPanel>` (`:1336-1455`). Four controls: translation visibility (zh/en/both/none — but Hochdeutsch is a no-op stub per `:22`), theme (cream/night — drives `.kt3-app.theme-night` class), auto-open rail toggle, and a deep-link via `?tweaks=1` URL param (`:260`).
7. **Speed ladder** — `SPEEDS = [0.6, 0.75, 1] as const` (`:92`); persisted to `reeloo_podcast_speed_v1` (`:93, 267-305`). ↑/↓ arrows ladder the speed with a 600ms `<div data-testid="kt-seek-overlay">` accessibility-live announcement (`:477, 540-555, 742-746`).
8. **Keyboard shortcuts** — R = replay current sentence (`:471-472, 461-469`), Space = play/pause (`:509-510`), ←/→ = seek by 15s, ↑/↓ = speed ladder, all with `event.preventDefault()` and explicit ignore-when-focused-in-input rules.
9. **MediaSession** — `seekbackward / seekforward / seekto` action handlers for OS-level lockscreen controls (`:356-365`).
10. **Cobalt night theme** — `data-theme="solarized-light"` default; tweaks toggle swaps to a `.kt3-app.theme-night` class with `#0f1419 → #1a2332` gradient (`:1474`, `app/page.tsx:185-245`).
11. **PodcastPlayerV1 legacy fallback** — `?layout=v1` opt-in. Distinct shape (theatre-style). `PlayerSwitcher.tsx` (379 LOC) gates which player mounts and writes to `localStorage["reeloo_recent_podcasts"]` on mount.
12. **KarottenVideoPlayer (955 LOC)** — video-backed variant when the demo carries a video URL.

## What we need from Claude design

Deliverables (Figma + Code Connect map + per-component test-id):

1. **Active-line typography spec** with the exact 42% pin math, Plex Serif 34/700, neighbour opacity ramp (18% → 42% at ±1, max 100% at active), 460ms ease-out cubic-bezier `(0.22, 1, 0.36, 1)`. Show 5-segment vertical stack at three states: idle, mid-transition, post-transition. Mobile variant (smaller body) needed.
2. **Word karaoke spec** — `<span data-kind=…>` underline styles for 4 kinds (vocab / grammar / cultural / idiom); hover/active/visited states; click affordance (cursor: pointer + subtle bg on hover); how the "current word" cursor advances across the line.
3. **Annotation rail mockup** — 360px wide, open + closed states, rail-toggle button location (right edge), per-entry shape: kind chip (color-coded) + phrase + gloss (en + zh) + POS + gender + CEFR + "Save to SRS" button + "Hear it" button. Empty state when no annotations on current segment.
4. **Tweaks panel mockup** — slide-out from right (separate from rail) with the 4 controls. Gate behind a `kt-tweaks-toggle` gear icon in the header. Should be styled as DEV-LITE but discoverable for power users. Explicit "ⓘ This panel is experimental" copy.
5. **Pre-teach vocab modal** — first-open-only, dismissable, lists 10 vocab pre-grid with audio chip per item + "Continue to player" CTA. Confirmation copy: "We'll surface these as you listen." Localized into zh-TW / en / de.
6. **Speed control** — visual treatment of the 3-step ladder (0.6× / 0.75× / 1.0×) as a segmented pill in the player bar. Active state. Keyboard hint on hover.
7. **Seek overlay** — 600ms-lived center-screen accessibility-live announcement (`role="status" aria-live="polite"`); copy: "−15s", "+0.85×", etc.
8. **Player chrome** — header with breadcrumb + language family chip (EN/DE/中文/日本語 active state); right-side: tweaks gear + rail toggle + close. Bottom bar: play/pause + scrub + time + speed.
9. **PodcastPlayerV1 legacy fallback** — minimal theatre-style mockup. Question for design: keep or drop? If keep, define when the user sees it (opt-in URL flag only? auto-fallback for low-quality demos?). Recommended: drop it (the impl elides it; design ratifies).
10. **KarottenVideoPlayer variant** — same focus-pin behaviour but with a video background. Decide: is video aspect-ratio'd (16:9 letterbox in dark frame) or full-bleed-blurred? Cover with focus column overlaid?
11. **Catalog `<RecentRow>`** — horizontal scroll strip on `/`, localStorage-backed, with thumbnail + title + lang chip + last-played time. Empty state ("Your recently played podcasts will appear here").
12. **`<ProgressModal>`** — animated waiting modal shown when a tile is clicked (cache-hit + cache-miss states). Copy phases: "Loading…" → "Almost ready…" → fade.
13. **Wizard form** — `/wizard` standalone page (target ∈ {de,en,ja}, topic chips, native language). Compact, single-screen, with a "Surprise me" topic chip. Loading state during 120s generation. Error state on 502.

All mockups in vanilla + cobalt-dark + high-contrast.

## Out of scope

- The audio-data model (`PodcastSegment` + `PodcastAnnotation`) — already canonical; no design change needed.
- The depot wizard backend (`/api/wizard/generate`) — design only owns the form, not the model.
- The 12 existing demos — design renders whatever `LessonData` carries.
- Multi-user concerns (login, sharing) — podcast-app is read-only public.

## Success criteria

- Every numbered feature 1–13 has a Figma frame with the exact tokens, dimensions, animations, and test-ids documented.
- Every interactive element has a `<surface>:<component>:<element>` test-id (e.g. `podcast-player:tweaks-panel:translation-toggle:zh`).
- The implementation engineer can port the 2,442-LOC source into the v2 shell guided only by the design + the v1 spec — no further code-reading required.
- Accessibility: WCAG AA in all three themes; keyboard shortcuts documented + announced via `aria-keyshortcuts`; MediaSession metadata spec'd.
- Mobile: 360px-wide viewport variant for every frame (the focus-pin transform changes; the rail collapses to bottom-sheet on mobile).
