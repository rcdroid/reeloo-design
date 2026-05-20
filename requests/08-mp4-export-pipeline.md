# Design request: Video export — the `<VideoExportSheet>` UX across all 5 surfaces

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v2/02` §1–§2 (notifications + export center, named but unsketched)
- `v3-gap-audit/cross-cutting.md` §6 (full theme analysis)
- `v3-gap-audit/readalong.md` §1.A (features 37–43)
- `v3-gap-audit/podcast.md` §1 (feature 30 = KarottenVideoPlayer)
- Source code: `/home/claw/work/readalong/nextjs/src/components/VideoExportTab.tsx` (1,576 LOC), `/home/claw/work/notes/app/api/tutorial/[id]/mp4/route.ts`, `/api/social-video/`, `/api/videos/`

## What we asked for in v2 (and what we got)

The v2 brief proposes `export.reeloo.ai` as a shared service (§3.3) and `<VideoExportSheet>` as a surface that opens from each app's overflow menu (§4.1, §4.2). Claude design produced no `<VideoExportSheet>` mockup, no progress UI, no YouTube grant flow, no template picker, no aspect-ratio picker, no cancel UX.

The impl wires `makeVideo()` from `useShell()` and surfaces "Export" CTAs on podcast / book / practice — but `makeVideo` is currently a no-op `console.log`.

## What's actually in the source apps

### readalong `<VideoExportTab>` (the canonical implementation — 1,576 LOC)

`/home/claw/work/readalong/nextjs/src/components/VideoExportTab.tsx`:

**Stage labels with human-readable copy** (`:62-75`):
- `local-export-started` → "Starting export process…"
- `rendering-frames` → "Capturing video frames…"
- `rendering-complete` → "Frames captured, preparing encoder…"
- `uploading-youtube` → "Uploading to YouTube…"
- `local-export-failed` → "Export failed"

**Two activeMode values** (`:34`): `video` (download MP4) | `youtube` (publish to YouTube).

**Per-tab settings overrides** (`:438-446`):
- `expDarkMode` (toggle export theme regardless of app theme)
- `expShowRomaji`
- `expShowWordHighlight`
- `expShowSegmentHighlight`
- `expShowAnnotations`

**YouTube fields** (`:146-160`): title, video ID, URL, privacy status, upload result.

**Caching** (`:44-114`):
- Module-level `exportDataCache` (Map)
- Module-level `youtubeStatusCache`
- SessionStorage persistence per deckId (survives reload)

**Job state** (`:457`): `job` + `loading` state machine; cache promotion to module-level on read; `cancelExportJob` available.

**Timeout** (`:50`): `WARNING_TIMEOUT_MS` for stuck `uploading-youtube` stage.

### notes-app pipeline

- `POST /api/tutorial/[id]/mp4` — generate MP4 from a tutorial
- `/api/tutorial/[id]/animate` — animation rendering step
- `/api/social-video` — social-share-formatted video
- `/api/videos/tag_ingest` — video-tagging ingest

### podcast-app `KarottenVideoPlayer` (955 LOC)

Not an export pipeline but a video-backed player variant. Worth integrating with export's "preview" stage — the preview shows what the exported video will look like.

## What we need from Claude design

1. **`<VideoExportSheet>` master mockup** — a slide-out sheet (NOT a modal — sheets don't block, modals do, and export is async). Top section: preview thumbnail; middle: template + aspect-ratio + destination; bottom: confirm + cancel. Mobile variant slides up from bottom.

2. **Per-surface entry point** — design the entry CTA for each surface:
   - **Podcast player**: "Make a video" in overflow menu (already `data-testid="podcast-player:make-video-button"`)
   - **Book chapter**: "Export chapter" in chapter header (already in impl, `book-reader:make-video-button`)
   - **Practice station** (lesson-end): "Export this drill" in completion card
   - **Practice cards** (deck-end): "Export this deck" in deck-summary
   - **Readalong deck-player**: "Export" in deck overflow (preserve readalong's existing tab)

3. **Template picker** — per surface, suggest a default template:
   - Podcast → `karaoke-vertical` 9:16
   - Book → `book-reading` 16:9
   - Practice drill → `drill-vertical` 9:16
   - Practice cards → `flashcard-square` 1:1 (for social)
   - Readalong deck → user-configurable

4. **Aspect-ratio picker** — `9:16 (Reels/Shorts/TikTok)` / `16:9 (YouTube)` / `1:1 (Instagram square)` / `4:5 (Instagram portrait)`. Visual chips with preview thumbnails.

5. **Destination picker** — `Download (MP4)` / `Publish to YouTube` / `Send to Reels` (future). Each destination unlocks different settings.

6. **YouTube OAuth grant flow** — when "Publish to YouTube" is picked and the user hasn't linked, the sheet shows a "Link YouTube" CTA → opens consent screen (Google OAuth) → returns to the sheet with `state=youtube_linked`. Per v2 brief §4.2 J2.

7. **Privacy-status picker (YouTube-only)** — `Public` / `Unlisted` / `Private`. Default `Unlisted`. With a small "?" tooltip explaining each.

8. **Per-template settings (theme override)** — toggle row: Dark mode (off/on), Show romaji (JA only), Show word-highlight, Show segment highlight, Show annotations. Each is a row with explanation copy.

9. **Preview pane** — small thumbnail with a "Render preview" button (renders a 3-second sample). On render, the thumbnail animates.

10. **Submit confirmation** — "Render & download" or "Render & publish to YouTube." Once tapped, the sheet doesn't close — it transitions to progress.

11. **Progress UI (the 6 stages)** — per-stage label + spinner + progress bar OR percentage. Each stage label uses the readalong copy verbatim. Cancellable at every stage.

12. **Stuck-export warning state** — when `uploading-youtube` exceeds `WARNING_TIMEOUT_MS`, show "This is taking longer than usual. Want to check status or cancel?" + "Check status" + "Cancel" buttons.

13. **Success state** — "Your video is ready!" + thumbnail + "Open download" / "View on YouTube" CTA. Persists across page reloads via sessionStorage.

14. **Failure state** — "Export failed" + reason + "Try again" CTA. With a "Report bug" link to `<FlagIssue>`.

15. **Notification integration** — When the sheet closes mid-render, the export continues in the background and a `<NotificationToast>` appears periodically: "Karotten video — 60% rendered" → "ready". On done, the inbox at `/me/notifications` carries a row.

16. **Inbox row UX** — `/me/notifications` shows export rows: thumbnail + title + status + download/view CTA. Hover/tap → re-opens the sheet's success state.

17. **Cancel-mid-render UX** — "Cancel export" button always available during stages 1–4. Stage 5 (`uploading-youtube`) may be uncancellable depending on backend; design surfaces that.

18. **Quota indicator (future-proofing)** — v2 brief open question §14 (Q16): YouTube export may become quota-limited. Design provides a quota-chip slot next to "Publish to YouTube" so when monetization lands, the chip can populate without redesign.

19. **Cross-surface consistency** — every entry point opens the SAME sheet with surface-appropriate defaults. The user mental model is: "Make a video" works the same way everywhere.

## Out of scope

- The Cloud Run job runner backend.
- The Postgres export-job state store.
- The GCS bucket `gs://readalong-exports/`.
- The actual MP4 encoder + frame-capture pipeline.
- YouTube API surface (upload, privacy, metadata).
- Quota / billing surface (just leave a chip slot).

## Success criteria

- One `<VideoExportSheet>` component composes all 5 surfaces' entry points.
- The 6 progress stages have defined visual states with the readalong copy.
- YouTube OAuth grant flow has a defined in-app transition (no plain redirect-and-pray).
- Cancel + warning + failure + success states are all designed.
- The inbox at `/me/notifications` carries export rows with a defined row shape.
- The sheet works on mobile (slides up, full-width on small screens).
- All four template defaults are documented per surface.
- WCAG AA in all three themes.
