# v2 meta-revision — what changed from v1 and why

> v1 (`v1/00-unified-brief.md`) was source-grounded and opinionated: **three apps, three services, eighteen components**. The user surfaced 11 cross-cutting themes that v1 either underweighted or didn't model at all. v2 is the integration pass.
>
> v2 does **not** rewrite v1's per-app specs (`v1/products/*.md`) or its token system (`v1/design-system/*.md`) — those remain the source-grounded baseline. v2 supersedes only the **unified brief**: read `v2/06-revised-unified-brief.md` first, then the five supporting specs `v2/01..v2/05`, then the v1 product files when you need to know what the existing code actually does.

---

## 1. The eleven themes, mapped to v2 homes

| Theme | Short name | Home in v2 | Status |
|---|---|---|---|
| A | Separate vs linked architecture | `v2/00` §3 + `v2/06` §2 | **revised** — see §3 below |
| B | Signal collection → user-level model | `v2/01` §1–§3 | new spec |
| C | Level-mismatch UX pattern | `v2/01` §4 | new pattern |
| D | Final synthesis pass | `v2/06` is the synthesis | done by existence |
| E | Issue-flagging UI | `v2/02` §3 | promoted from v1's single-app `<FlagIssue>` |
| F | Analytics funnel | `v2/02` §4 | new — meta-design constraint |
| G | Settings + dashboard + map + diary | `v2/03` (all of it) | new surfaces |
| H | Test-id annotations | `v2/05` (all of it) | new contract |
| I | Video export + YouTube + notification center | `v2/02` §1 + §2 | new subsystems |
| J | Avatar role redesign | `v2/02` §5 | upgrades the v1 "Pip" mascot question |
| K | Learn-anything sandbox | `v2/04` (all of it) | new mode on `practice.reeloo.ai` |

Eleven themes, eleven homes. No theme is dropped. The one I am least confident in is theme A — the architectural choice — so I treat it head-on in §3.

---

## 2. What v1 got right, kept verbatim

- **Three postures (ear-first listen, eyes-first read, output-first drill)** are physically different and deserve different layouts. v1 §1, defended in v1 §1 "Defence of the recommendation" bullets 1–2. v2 keeps this.
- **Three apps as separate Next.js deploys** (`podcast.reeloo.ai`, `book.reeloo.ai`, `practice.reeloo.ai`) + an internal `studio.reeloo.ai`. v1 §2. v2 keeps the deploy boundary, BUT (§3 below) tightens the *user-visible* linkage so it reads as one product.
- **Three shared services** (`dictionary.reeloo.ai`, `audio.reeloo.ai`, depot `/api/llm`). v1 §1. v2 ADDS three more: `signals.reeloo.ai`, `notifications.reeloo.ai`, `export.reeloo.ai` — see §3 and `v2/02`.
- **18-component inventory** at v1 §5. v2 extends to 24 components — see `v2/05` §2.
- **Token system** in `v1/design-system/proposed-tokens.md`. v2 adds new tokens for avatar states, notification states, and level-mismatch dialogue surfaces but does not retire any v1 tokens.

---

## 3. Architecture position: separate **deploys**, linked **product** (theme A)

The user's challenge to v1: *"should we find the links between them?"* My answer: **yes, but linkage is a product-surface concern, not a deploy-topology concern**. v1's 3-domain recommendation survives at the deploy layer; v2 adds enough cross-app surface to make the product *feel* like one app to the user.

### 3.1 Why the 3-domain recommendation survives

The four arguments from v1 §1 ("Defence of the recommendation") still hold, and the 11 themes do not invalidate any of them:

1. **Posture argument (v1 §1 bullet 1).** Themes B/C/G/J are user-state primitives that travel *between* surfaces — they don't change the fact that reading and listening are different postures.
2. **Gravity-well argument (v1 §1 bullet 2).** `practice.reeloo.ai` is still the retention surface. Themes B (signal model) and G (learning map / diary) actually *strengthen* this: every signal flows back to practice; the dashboard lives at `practice.reeloo.ai/me`.
3. **Readalong-as-studio (v1 §1 bullet 3).** Theme K (learn-anything sandbox) introduces a new authoring loop that lives in studio (generator) + practice (player), validating the studio/runtime split.
4. **Shared services beat shared apps (v1 §1 bullet 4).** Themes B (signals), F (analytics), I (export) all want to be shared services, not shared apps. v2 adds three new services rather than collapsing the apps.

### 3.2 What v2 changes to make the product feel linked

Concretely, the user lands on different domains for different postures, but the surfaces below tie the experience together:

| Linkage mechanism | What it does | Where it lives | New in v2? |
|---|---|---|---|
| **Shared signal bus** | every interaction emits an event → user-level model | `signals.reeloo.ai` (new) | yes |
| **Single account header** | one auth, one identity, one "Pip" mascot across all three domains | NextAuth on `account.reeloo.ai` (new auth subdomain) | yes |
| **Cross-app `<AppSwitcher>`** | header component shows the three apps as tabs with active highlight | extends v1's 18 → 24 components | yes |
| **Unified `me` route** | `*/me` resolves to `practice.reeloo.ai/me/<tab>` regardless of which app the user clicked from | client-side redirect | yes |
| **Notification center** | toast pill in every app header surfacing long-running export jobs (theme I) | `notifications.reeloo.ai` event stream | yes |
| **Issue-flagging chip** | the v1 learn-stations `<FlagIssue>` component (v1 products/learn-stations §5 Flow F) is mounted globally in all three apps | shared package | yes |
| **Level-mismatch dialogue** | when the model from theme B says "level >> content" while user is in any app, surface the same dialogue (theme C) | shared component, `v2/01` §4 | yes |
| **Avatar primitive** | Pip is one character across all three apps with two modes (proactive / passive) | shared `<Avatar>` component, `v2/02` §5 | yes |

The user navigates between domains, but the **header, the avatar, the signal model, the notification center, and the dashboard** are constant. This is the "linked" version of "separate".

### 3.3 The position in one sentence

> **One identity, one signal model, one design system, one mascot, one notification center — across three domains shaped to their postures.**

That is the v2 product line.

---

## 4. What v2 adds beyond v1 (one-line summaries)

- **`signals.reeloo.ai`** — append-only event sink + level-model materializer. Reads from every app; writes back the inferred level vector (`v2/01`).
- **`notifications.reeloo.ai`** — long-running task center (exports, deck generations, account-level alerts). Per-app toast + global inbox at `*/me/notifications` (`v2/02` §1).
- **`export.reeloo.ai`** — generalised MP4 export pipeline, lifted out of readalong's Cloud Run Job stack (cited from `v1/products/readalong.md` §5 Flow F). Now usable from podcast, book, flashcard, and readalong-style decks (`v2/02` §2).
- **YouTube OAuth dance** — second OAuth grant flow layered over NextAuth, scoped to `youtube.upload` (`v2/02` §1.4).
- **Global `<FlagIssue>`** — the learn-stations chip (`v1/products/learn-stations.md` §5 Flow F) goes shared, with an "attach screenshot" path (`v2/02` §3).
- **Analytics funnel definition** — 4 named funnels (onboarding, daily-loop, vocab-save, export-publish) (`v2/02` §4).
- **Avatar primitive** — Pip with proactive (gesture / whiteboard / facial expression) and passive (clickable info button) modes (`v2/02` §5). Resolves v1 open-question 7 ("Does Pip travel?") with a clear *yes, but as a primitive*.
- **Personal surfaces** — `*/me/{history,progress,map,diary,settings,notifications}` on `practice.reeloo.ai/me/*` (`v2/03`).
- **Learn-anything sandbox** — `practice.reeloo.ai/learn-anything/<subject>` with Pyodide or Sandpack embeds (`v2/04`).
- **Test-id contract** — `data-testid` naming convention + per-component checklist (`v2/05`).

---

## 5. What v2 does NOT change

- v1 token system (`v1/design-system/proposed-tokens.md`) is the canonical token set. v2 adds tokens, never removes.
- v1 per-app specs (`v1/products/*.md`) are the source-of-truth for *what each app does today*. v2 cites them; the per-app files do not move.
- v1 open questions 1–10 (`v1/00-unified-brief.md` §9) remain open. v2 resolves only #7 (avatar travels — yes).
- v1 in-scope / out-of-scope list (v1 §7) remains. v2 adds new in-scope items (avatar states, notification center, learn-anything sandbox screens, level-mismatch dialogue).

---

## 6. Reading order for Claude design

1. **`v2/06-revised-unified-brief.md`** — the canonical brief. Read this first.
2. `v2/01-signal-and-level-model.md` — what the user model knows and how it's used.
3. `v2/02-cross-cutting-systems.md` — the five new shared subsystems.
4. `v2/03-personal-surfaces.md` — `*/me` route family.
5. `v2/04-learn-anything-sandbox.md` — the new generative mode.
6. `v2/05-tdd-design-contract.md` — `data-testid` convention.
7. `v1/products/*.md` — per-app deep dives when you need source-grounded facts.
8. `v1/design-system/*.md` — token sources.
9. `v1/00-unified-brief.md` — historical baseline only; v2/06 supersedes.
