# v2/04 — Learn-anything sandbox

> Resolves theme **K**: convert any subject → a Duolingo-style tutorial on the fly, with embedded sandbox UI for interactive subjects.

---

## 1. Route + framing

**Route:** `practice.reeloo.ai/learn-anything/<subject-slug>`

- `<subject-slug>` is a URL-safe kebab-case slug derived from the user's input ("PyTorch", "古文觀止", "ECG-reading").
- The slug is also the cache key. Two users requesting "pytorch" share the same generated unit set.
- The entry surface (`practice.reeloo.ai/learn-anything`) is a single search box: "What do you want to learn today?" — `<LearnAnythingInput>` (`v2/05` §2).

This sits *next to* the four existing course families on practice (`/minna`, `/mundart`, `/deutsch`, `/url`) — same Pip mascot, same station player, same library integration, just a different content generation path.

Defence against putting it on its own subdomain: every signal it produces (`drill.attempt`, `card.rated`, `dict.save`) needs to flow into the same level model. Co-locating with the rest of practice avoids cross-origin signal SDK headaches.

---

## 2. Generation pipeline

```
User opens /learn-anything
        │
        ▼ enters "PyTorch", taps "Make a tutorial"
        │
        ▼
practice.reeloo.ai/api/learn-anything/generate
        │ slug = "pytorch", lang = "en", userId = "u_…"
        │
        ▼
1. Cache lookup
        │  key = `learn-anything:${slug}:${lang}:${cefrBucket}`
        │  hit  → return existing materialId
        │  miss → fall through
        ▼
2. Outline call (depot LLM gateway)
        POST depot.reeloo.ai/api/llm
        {model: "gemini-3.1-pro-preview",   // quality, one-shot
         prompt: outlinePrompt(slug, lang, level),
         outputSchema: TutorialOutline}
        │
        ▼ returns 8–14 lesson cards + which ones want sandboxes
3. Content call (parallel, 4 workers)
        For each card: gemini-3.1-flash-lite-preview for body + 3 quiz items
        │
        ▼
4. Sandbox provisioning (sandbox cards only)
        Each sandbox card chooses an embed strategy (§4 below) and is shipped with
        seed code + an expected-output marker for the verifier.
        │
        ▼
5. Persist
        Write to depot notes (category: "tutorial-material") with full JSON
        Materializer fires `tutorial.ready` notification (v2/02 §1)
        │
        ▼ user gets a toast: "PyTorch tutorial is ready"
6. Open
        /learn-anything/pytorch
        renders <StationRunner> over the generated card sequence
```

### 2.1 Outline shape

```ts
type TutorialOutline = {
  slug: string;
  title: string;                  // "PyTorch: a one-evening tour"
  targetCEFR: "A1" | ...;         // only meaningful for language subjects; "N/A" otherwise
  estimatedMinutes: number;
  cards: Array<{
    kind: "concept" | "quiz" | "sandbox" | "checkpoint";
    title: string;
    summary: string;              // 2 lines
    body?: string;                // markdown, for concept/checkpoint
    quiz?: {prompt: string; choices: string[]; correctIndex: number; infoExplain: string};
    sandbox?: {
      runtime: "pyodide" | "sandpack-react" | "sandpack-vanilla" | "iframe-only";
      seedFiles: Record<string, string>;
      expectedOutput?: string;     // for the verifier to mark "completed"
      runtimeNotes?: string;
    };
  }>;
};
```

### 2.2 Prompt sketch (outline, gemini-3.1-pro-preview)

> You generate a Duolingo-style one-evening tour of the topic. 8–14 cards. Mix concept cards with quiz cards. If the topic is hands-on (programming, data analysis, music notation, electronics simulation), interleave at least one sandbox card per 3 concepts. Sandbox cards must specify a runtime from {pyodide, sandpack-react, sandpack-vanilla, iframe-only}. For non-hands-on topics (history, literature, music theory) skip sandbox cards. Output JSON matching the provided schema. Language: ${lang}. CEFR bucket from user level: ${cefrBucket}.

The prompt is in `studio.reeloo.ai/admin/prompts/learn-anything-outline.md` so the operator can iterate without a deploy.

### 2.3 Caching policy

- Same `slug + lang + cefrBucket` reuses the cached material for 30 days, then auto-stales.
- The operator can pin a tutorial as "evergreen" via `studio.reeloo.ai/admin/tutorials/<id>?pin=1`, which exempts it from auto-stale.
- A user can request a regeneration via `/learn-anything/<slug>?regenerate=me` — produces a private copy bound to userId only.

---

## 3. Interaction with the level model (`v2/01`)

- The generator reads `signals.reeloo.ai/api/level/<u>?lang=<lang>` to set `cefrBucket`. For non-language subjects (PyTorch), it reads the user's `skills.reading` to pick a difficulty band roughly mapping to "novice / familiar / fluent".
- The tutorial cards inherit `weakTopics` — if the user has `weakTopics: [{topic:"dative-prepositions"}]` and slug=`german-prepositions`, the outline emphasises dative.
- Every interaction inside `/learn-anything/<slug>` emits the same `drill.*` / `card.*` events as a normal practice lesson. The level model doesn't distinguish — a substitution drill is a substitution drill.
- Completing a tutorial fires `deck.complete` with `payload.kind="learn-anything"`, which the diary surface (`v2/03` §5) surfaces as a milestone.

---

## 4. Sandbox embed strategy

The user mentioned Pyodide and Sandpack. v2 supports both, plus two simpler fallbacks. The runtime is chosen at outline time and is part of the cached material.

| Runtime | Best for | Implementation | Constraints |
|---|---|---|---|
| **pyodide** | Python, NumPy, simple PyTorch CPU snippets | iframe-hosted Pyodide REPL (we host the html + worker) | first-load ~5MB; cold start ~3s; CPU only; PyTorch via `pip install --index-url=https://… torch==2.0.0+cpu` |
| **sandpack-react** | JS / React UI tutorials | CodeSandbox's `@codesandbox/sandpack-react`, esm.sh deps | no Node native modules |
| **sandpack-vanilla** | JS / DOM tutorials | sandpack with the `static` template | same |
| **iframe-only** | tutorials that link to an external playground (Wolfram, Desmos, MuseScore) | plain `<iframe sandbox="allow-scripts">` | no signals from inside; only "I tried it" CTA writes a signal |

### 4.1 Component contract

```tsx
<SandboxCard
  runtime
  seedFiles
  expectedOutput            // optional; when set, a verifier compares stdout / DOM
  height                    // default 480px, max 720px
  onCompleted               // fires when expectedOutput matches OR user clicks "I got it"
  onSignal                  // optional; forwards interpreter events to signals SDK
/>
```

- The sandbox is fenced from the parent surface with `sandbox="allow-scripts allow-same-origin"` to limit DOM clobber.
- Sandboxes cannot reach `signals.reeloo.ai` directly; they postMessage to the parent, which forwards via the SDK.

### 4.2 Pyodide loading UX

- Sandbox cards with `runtime="pyodide"` show a `<ProgressModal>` (v1 §5) on first paint: "Bringing Python into your browser… (≈3s)".
- After load, the sandbox stays warm for the rest of the session.
- A small "Reset" button reloads the seed; "Run" executes; "Hint" reveals `runtimeNotes`.

### 4.3 Per-tutorial sandbox count

The outline prompt caps sandbox cards at 1 per 3 concept cards to avoid blowing up first-paint cost. Cached materials get re-validated against this cap before render.

---

## 5. Worked example — PyTorch one-evening tour

The user said: *"user wants to learn PyTorch → tutorial generated with embedded sandbox UI showing interactive results"*. Here are 5 cards + 1 sandbox embed as an illustrative slice (the real outline produces 8–14 cards).

| # | kind | title | content sketch |
|---|---|---|---|
| 1 | concept | "Tensors, not arrays" | "A tensor is a NumPy array that can live on a GPU and remembers how to differentiate itself. We're going to build one." |
| 2 | sandbox (pyodide) | "Make a tensor" | seed: `import torch; x = torch.tensor([1., 2., 3.]); print(x, x.dtype, x.shape)` — expectedOutput contains `torch.float32`. |
| 3 | concept | "Why `requires_grad`" | "If you want PyTorch to track gradients, mark the tensor with `requires_grad=True`. Otherwise it's just a number." |
| 4 | quiz | "Spot the difference" | Q: which tensor below will accumulate gradients? Choices show 3 declarations; correct one has `requires_grad=True`. `infoExplain` (used by `<Avatar mode="passive">`, `v2/02` §5.3) clarifies. |
| 5 | sandbox (pyodide) | "Compute a gradient" | seed: `x = torch.tensor(2., requires_grad=True); y = x**3; y.backward(); print(x.grad)` — expectedOutput `12`. |
| 6 | checkpoint | "What you've got" | Recap card listing the 3 concepts, gating "Mark complete" on previous 5 cards. |

Each card is rendered by `<StationRunner>` (v1 §5) with an extended station kind: `learn-anything-concept`, `learn-anything-sandbox`. The runner already handles auto-advance + stage CTAs, so the new station kinds plug in as additional discriminated-union members in `learn-stations/data/stations.ts` (`v1/products/learn-stations.md` §4 "Station discriminated union").

---

## 6. Failure modes + UX guardrails

| Failure | UX |
|---|---|
| Outline LLM call times out | Show the input form with "That took longer than expected. Try again, or pick from existing tutorials." + show top 5 recently generated tutorials. |
| One body card LLM fails | Outline still ships; bad card renders as "We're polishing this one — come back in a minute" with a regenerate-just-this-card CTA. |
| Pyodide download blocked | Sandbox card falls back to "iframe-only" mode with a link to the external playground; emits `sandbox.fallback` signal. |
| Generated content low-quality | `<FlagIssue>` (v2/02 §3) is mounted on every card; operator triages from `studio.reeloo.ai/admin/flags?source=learn-anything`. |
| User abandons mid-tutorial | The level model still gets partial credit; the diary (`v2/03` §5) records "started PyTorch tour, completed 3 of 8 cards" — no failure framing. |

---

## 7. Open questions for design

1. Should `/learn-anything/<slug>` show a public-ish "browse what others generated" surface, or stay private to each user? My vote: private MVP, public opt-in V2.
2. The Pyodide first-paint cost (~5MB / 3s cold) is high enough that we should consider showing a "this tutorial needs Python in your browser, ~3s on a fast connection" pre-flight notice before the first sandbox card. Worth a one-time toast vs always-show?
3. Sandpack vs Pyodide — should non-Python tutorials default to Sandpack, or should we lean on iframe-only for everything to keep first-paint small? My vote: Sandpack for JS/React tutorials (cheap, ~200KB), iframe-only for everything that doesn't fit those two runtimes.
4. Cross-language support — does `learn-anything` work in zh-TW + de + ja, or English-first MVP? My vote: English-first MVP; the level model already disambiguates per-language, so adding language is mostly a prompt template thing.
