# Teachly — Teaching Platform Prototype v1.2

A **single self-contained HTML file** (zero build step, zero CDNs, zero hosting) that starts as a chat and
*builds itself* into a teaching workspace the moment you send your first message.

Product flow (per the vision in `2026-09-12-teaching-platform-idea.md`):

1. **Start = chat.** Type what you want to be taught and send it.
2. A **workspace** (mission + outline + navigable lesson pages with quizzes) is generated behind a visible progress panel.
3. **Selection bubble:** highlight any text in a lesson → a bubble pops up to ask a question about the selection,
   with optional **image attachments** (button or clipboard paste) and **links**. Answers can be **inserted into the
   workspace** as a note block. The bubble is the **only question surface** — there is no docked chat.
4. **Selection Q&A history** lives in a **collapsible section in the left sidebar** — full history, never pruned.
   The bubble itself always shows only the latest exchange.
5. **Adaptive workspace:** the more selection questions you ask, the more the app learns about you (top-right meter).
   At 100% it proposes concrete workspace changes you can Accept, Discuss, or Reject — applying changes **live**
   through a translucent stepper overlay. The **Discuss thread sits directly below the proposal card**.

All workspace + selection-history + adaptive state **auto-saves to `localStorage`** and restores on reopen.

## Open it

- **Simplest:** double-click `index.html` (Chrome/Edge recommended — `localStorage` works on `file://`).
- **Or** a trivial static server: `python -m http.server 8000` in this folder → http://localhost:8000

## Demo mode (no key needed)

Demo mode is **ON by default** and simulates LLM responses locally, so the entire flow can be exercised offline:
workspace generation, quizzes, selection bubble with attachments, the **full adaptive workspace cycle
(meter → proposal → accept/discuss/reject → live stepper)**, and persistence.

- Toggle it from the **Demo mode chip** on the start screen or in **Settings**.
- While in demo mode the app makes **zero network calls**.

## Using your own LLM (OpenAI / OpenRouter / any OpenAI-compatible endpoint)

1. Click **Settings**.
2. Pick a **provider preset** from the dropdown (**OpenAI** or **OpenRouter**) — the Base URL is set for you and the
   **model dropdown** is populated with that provider's models. Choose **Custom** to type your own Base URL + model
   (any OpenAI-compatible endpoint; model list has a "Custom model…" fallback).
3. Paste your **API key** → **Save** → **Test connection** to verify before use.
4. Turn **Demo mode off**.

> ⚠️ **v1 key handling:** the key is stored in this browser's `localStorage` and sent only to the endpoint you configure. That's acceptable for a local prototype but not for a shared product — the intended v1→v2 path is a backend swap (below) that holds the key server-side.

## What's new in v1.2 (Card 0 — dock chat removed)

1. **Teaching Assistant dock removed entirely** — removal, not hiding. No dock UI (input/messages/expand/collapse),
   no dock CSS, no follow-up chat code paths. Post-lesson questions flow through the **selection bubble only**.
2. **Selection Q&A history re-homed** — the full history (which used to live in the dock) now lives in a
   **collapsible "Selection Q&A history" section in the left sidebar** (open by default; toggle with the caret).
   Nothing is pruned; the bubble stays latest-exchange-only.
3. **Discuss thread below the proposal card** — the adaptive proposal's **Discuss** button opens a focused thread
   that sits **directly below the change-proposal card** in the same panel. Asking for a revision reworks the
   proposal and re-presents it with an **"Adjusted after discussion" badge**. This is the only conversational
   surface besides the bubble. The thread transcript is per-session; the adaptive decision state persists.
4. **localStorage cleanup** — the dock-chat key `tp_chat_v1` is **deleted on load** (orphaned by the removal).
   Selection Q&A history (`tp_selqa_v1`), workspace, settings and adaptive state keep their existing keys.
5. Everything else is untouched: the `LLMService` seam, adaptive meter/proposal/stepper, notes + note view,
   settings (provider presets, model dropdown, custom model), workspace generation, persistence.
6. **Standing UI rule enforced:** no emojis in UI chrome — text labels or inline SVG icons only.

### v1.1 recap (previous release, still in place)

- **5 UX fixes:** no emojis; clipboard image paste (Ctrl+V, shares the max-2 path with the Image button);
  refined bubble layout (question → attachments → actions); viewable inserted notes; paste respects max-2.
- **Option C bubble history** (now in its sidebar home), adaptive workspace (demo-simulated), provider presets.

## What works in v1.2

- ✅ Chat-first start state → workspace generation (real LLM **or** demo) with staged progress UI
- ✅ Workspace = mission, outline sidebar, numbered lesson pages, prev/next nav, interactive quizzes (correct/incorrect feedback + reveal)
- ✅ Selection bubble: highlight any lesson text → Q&A with **images** (button **or clipboard paste**, data-URLs, max 2), **links** (sent as references), and answers insertable as notes
- ✅ "Insert into workspace" — Q&A becomes a note block in that lesson, deletable, persisted, **and viewable** (sidebar click or note-card click → full content + locate)
- ✅ **Option C:** bubble shows only the latest exchange; full Selection Q&A history lives in the **sidebar's collapsible section** (never pruned, timestamped, count badge)
- ✅ **Adaptive workspace:** meter → proposal card (Accept/Discuss/Reject) → **Discuss thread directly below the proposal card** → translucent stepper overlay with queued→pending→changing→completed statuses and a Stop control; changes apply live to the workspace and persist
- ✅ localStorage persistence: settings (incl. provider preset), workspace (incl. notes), Selection Q&A history, **meter + proposal state** (refresh mid-flow doesn't lose position)
- ✅ **No dock chat anywhere** — no dock element, no dock CSS, no follow-up chat flow
- ✅ "New topic" resets to the chat-first start state (clears workspace/notes/history/adaptive) and purges the orphaned dock-chat key
- ✅ Demo mode: full offline simulation, zero network calls
- ✅ All LLM I/O isolated behind the `LLMService` seam (details below)
- ✅ **Streaming bubble answers** via `LLMService.chatStream` — chunks render incrementally (caret + "streaming..." indicator); graceful non-streaming fallback when a provider blocks CORS

## localStorage keys & v1.2 migration

| Key | Contents | Status |
|---|---|---|
| `tp_settings_v1` | provider preset, base URL, model, demo flag, API key | unchanged |
| `tp_workspace_v1` | workspace (mission, outline, lessons, notes, topic) | unchanged |
| `tp_selqa_v1` | Selection Q&A history (full, never pruned) | unchanged — re-homed UI only |
| `tp_adaptive_v1` | meter %, adaptive state, proposal, counters | unchanged |
| `tp_chat_v1` | dock-chat transcript (v1.0–v1.1) | **deleted on load** — orphaned by the dock removal; the selection bubble + discuss thread replace the dock's chat surface |

On first load after upgrading, `tp_chat_v1` is removed automatically; every other key is read as before, so no
workspace, history, notes or adaptive progress is lost.

## Known limitations (v1.2)

- **Streaming is provider-dependent** — `chatStream` streams chunk-by-chunk in Demo mode and with the OpenRouter preset (open CORS); providers that block CORS (e.g. OpenAI chat completions) transparently fall back to non-streaming (full answer, one piece). See "Streaming status" under Architecture.
- **Links are not fetched** — pasted links are sent to the model as references; the page never fetches their content (browser CORS + scope). Stubbed per the spec.
- **Images inline via data URLs** — up to 2 per question; large images bloat `localStorage` persistence only if inserted as notes (the note stores text, not the image — images are request-only).
- **Markdown renderer is minimal** (headings, bold/italic, code, fenced code, lists, blockquotes, links, hr) — no tables/HTML passthrough by design (XSS-safe, everything is escaped).
- **Adaptive proposal supports the FULL op set** — `add` (new section), `rewrite` (replace an existing section by its exact `### heading`), `restructure` (reorder a lesson earlier in the running order), `remove` (delete a redundant section by heading), and `mission` (extend the mission text). In Demo mode the demo proposal deliberately yields a **mixed op mix** (add/rewrite/restructure/remove/mission) so a non-additive change is demonstrable end-to-end without a key. With a live model these become ordinary LLM calls (JSON in/out; see “Cost per adaptive cycle”).
- **Discuss thread is per-session** — the open thread's transcript does not survive a reload (the adaptive decision state does; a mid-discussion reload resumes at the proposal card).
- **Mid-apply reload** — the stepper run itself isn't resumable: if you refresh mid-apply, already-applied changes stay in the workspace and the meter resumes cleanly at 0. **Stop or a failed apply aborts** the run and restores the exact pre-overlay snapshot, so a partial application can never leave the workspace corrupted.
- **Key lives client-side** in `localStorage` — fine for a local prototype; move to a backend before sharing.
- **Real API from `file://`** — Chrome/Edge allow it; some browsers or providers (CORS) may block it — use the static-server option or a provider with open CORS.
- Workspace is **generated in one call** (single curriculum JSON); a staged generator (mission → resources → lessons) is a possible v2.
- Narrow screens hide the outline sidebar (desktop-first prototype).

## Architecture & the `LLMService` seam

```
 UI (markdown renderer, bubble, sidebar Q&A history, workspace, adaptive meter/card/stepper, discuss thread)
   │  — knows nothing about fetch, endpoints, headers, keys
   ▼
 LLMService (index.html, one module)
   ├─ chat(messages, {json?, temperature?}) → Promise<string>          ← THE SEAM
   ├─ chatStream(messages, opts, onChunk) → Promise<string>            ← streaming seam
   │     same result as chat(), but text is delivered incrementally via
   │     onChunk(chunk); transparently falls back to one-piece chat()
   │     when the provider blocks streaming (CORS / non-OK / no
   │     ReadableStream / empty stream — onChunk is never called then)
   ├─ proposeChanges({workspace, selQA}) → Promise<{changes[], rationale}>   ← adaptive
   ├─ reviseProposal(proposal, messages) → Promise<{...proposal}>            ← adaptive discussion
   ├─ applyChange(change, lesson) → Promise<{body, content, missionOverwrite}> ← adaptive per-op transform
   ├─ configure({baseUrl, model, apiKey, demo, provider})
   ├─ ping() / isDemo() / source() / getConfig()
   └─ two interchangeable implementations behind it:
        • 'demo'   — local simulation (no network; adaptive proposals derived
                     from real history; mock provider streams the answer in
                     ~24-char timed word chunks, zero real LLM calls)
        • 'openai' — OpenAI / OpenRouter / any OpenAI-compatible
                     POST {baseUrl}/chat/completions (stream:true for chatStream,
                     with SSE parsing + fallback to non-streaming)
                     (headers, endpoint, auth all live HERE — nowhere else)
```

The rest of the app only ever calls `LLMService.chat(...)` / `chatStream(...)` / `proposeChanges(...)` / `reviseProposal(...)` with plain
message arrays / context objects:

```
[{role:'system'|'user'|'assistant', content: string | [{type:'text',text} | {type:'image_url',image_url:{url}}]}]
```

### Streaming status (v1.2 Card 5)

**Streaming WORKS, verified end-to-end through the seam** — with one provider-dependent caveat:

- **Demo mode (mock provider):** `chatStream` simulates the answer and delivers it to the bubble in ~24-character
  timed word chunks; the bubble renders chunks incrementally with a blinking caret, a "streaming..." label and an
  accent border, then finalizes via the existing answer display. Zero real LLM calls.
- **OpenRouter preset (real mode):** browser-side streaming from a static origin **works** — verified via public
  preflight evidence (`Access-Control-Allow-Origin: *` on the preflight).
- **OpenAI preset (real mode):** `api.openai.com` has **tightened CORS for chat completions**, so with this preset
  the code transparently falls back to the non-streaming `chat()` path — the full answer still renders, just in one
  piece. The fallback triggers on any refusal: CORS, non-OK status, missing `ReadableStream`, or an empty stream.

**Workaround if your provider blocks CORS (static-server + same-origin reverse proxy):**

> serve `prototype/index.html` from any static server and front the LLM endpoint with a local same-origin proxy,
> e.g. `npx http-server` for the app plus a tiny reverse proxy mapping `/llm` -> `https://openrouter.ai/api/v1`
> (Node http/express, or Caddy: `:8080 { reverse_proxy /llm/* openrouter.ai:443 }`) and set **Custom provider**
> Base URL to `http://localhost:PORT/llm` — key stays browser-side, no app code change (the seam is the only fetch site).

### The backend swap (Option C from the idea note)

To replace the client-side mock/REST providers with a real backend that holds the API key:

1. **Implement the same interface server-side** — expose these endpoints (the interface `LLMService` already speaks):
   - `POST /api/llm/chat` — server holds the key, forwards to the model, returns the text (optionally streams SSE).
   - `POST /api/llm/chat/stream` — optional streaming variant (SSE chunks); or add streaming to `/chat`.
   - `POST /api/llm/propose` — adaptive proposal generation (maps `proposeChanges`).
   - `POST /api/llm/revise` — proposal revision after Discuss (maps `reviseProposal`).
   - `POST /api/llm/ping` — connectivity test.
2. **Files touched: exactly one** — `index.html`, inside the `LLMService` module (the only fetch site in the app).
   The internals of `chat`/`chatStream`/`proposeChanges`/`reviseProposal`/`applyChange`/`ping` become thin
   `fetch('/api/llm/…', …)` calls. **Zero UI changes** — everything above the seam (bubble, adaptive cycle,
   discuss thread, history) is untouched.
3. **Keys:** the API key never touches the page — remove it from Settings/localStorage on the client side; the
   server injects it. `configure()` keeps accepting the demo/provider fields; the key field becomes inert.
4. **Same swap point** can take over persistence (workspace/history/adaptive) when multi-device sync is needed.

## Adaptive op types & cost per cycle

**Operation types** (each change in a proposal carries one of these, shown as a chip on the proposal card and in the stepper):

| op | effect | real-mode calls |
|---|---|---|
| `add` | append a new `### section` to a lesson | 1 × `applyChange` (model writes the new body) |
| `rewrite` | replace an existing `### section` (matched by its exact heading) | 1 × `applyChange` (model returns revised content) |
| `restructure` | move a lesson one position earlier in the running order | 1 × `applyChange` (model returns the reordered outline/content) |
| `remove` | delete a redundant `### section` (matched by its exact heading) | 1 × `applyChange` (model returns content minus the section) |
| `mission` | append an adaptive note to the mission text | 1 × `applyChange` when model rewrites the mission |

**One full adaptive cycle** (meter 0% → 100%, four selection-bubble questions) costs the following in **real** mode:

- 1 × `proposeChanges` — the proposal-generation call (sends workspace + full Q&A history; receives the structured change list).
- 1 × `applyChange` **per change** being applied — the per-op content transform for the affected lesson/mission (skip if the proposal is Rejected).
- **only if discussed…** N × `chat` (discuss turns) + 1 × `reviseProposal` if the proposal is revised before applying.
- The meter itself costs nothing extra — it only counts bubble Q&A locally.

Demo mode makes **zero network calls**: `proposeChanges`/`reviseProposal`/`applyChange` are all simulated locally. So with a full proposal of ~4 changes the real-key steady state is **≈ 1 propose + 4 apply** calls per cycle,
plus discussion/revision only when the student engages the Discuss thread.

Each `applyChange` call re-sends the current lesson content (or mission) and expects the full revised markdown back — so its
token cost scales with lesson size, not the proposal length. Keep lessons short to keep per-cycle cost low.

## Files

| File | What |
|---|---|
| `index.html` | The entire app (~125 KB: CSS + HTML + JS inline) |
| `README.md` | This file |

## Tested (v1.2)

- JS extracted from the HTML passes `node --check` (syntax OK, 0 errors).
- Demo mode exercised end-to-end in real headless **Edge via CDP** (HTTP-served from a fresh profile), **all checks pass**:
  - start state → **workspace generated** (4 lessons, mission, outline) via the start chat;
  - selection bubble opens on a real DOM selection; **8 selection Q&As** asked;
  - bubble shows only the latest exchange while the **sidebar "Selection Q&A history" section accumulates**
    1 → 2 → 3 → 4 → … → 8 timestamped entries (count badge matches), across two cycles;
  - **meter 25% → 50% → 75% → 100%** across the bubble questions, twice;
  - **proposal card at 100%** (2 changes listed) → **Reject resets meter to 0%** and hides the card;
  - second cycle → **Discuss opens the thread directly below the proposal card**; "revise it please" reworks the
    proposal → thread shows 3 messages, **"Adjusted after discussion" badge appears**, proposal re-presented;
  - **Accept runs the translucent stepper** with queued → pending → changing → completed all observed live,
    lesson section + mission note **actually changed**, meter reset, card hidden;
  - **reload restores** workspace (incl. applied changes), 8-entry Q&A history in the sidebar, meter state;
  - **no dock element remains in the DOM** (0 ids / 0 classes matching "dock"; 0 chat message nodes);
  - `tp_chat_v1` **purged on load** (seeded stale key deleted; final key set is only `tp_selqa_v1`,
    `tp_workspace_v1`, `tp_adaptive_v1`, `tp_settings_v1`);
  - rendered UI surfaces scanned **emoji-free** (0 hits) and the HTML source contains 0 emoji lines.
- No real API calls made during verification (no key on this machine); demo mode makes zero network calls.