# Teachly — Teaching Platform Prototype v1.1

A **single self-contained HTML file** (zero build step, zero CDNs, zero hosting) that starts as a chat and
*builds itself* into a teaching workspace the moment you send your first message.

Product flow (per the vision in `2026-09-12-teaching-platform-idea.md`):

1. **Start = chat.** Type what you want to be taught and send it.
2. The chat **drops to a bottom dock** while a **workspace** (mission + outline + navigable lesson pages with quizzes) is generated behind a visible progress panel.
3. The workspace becomes the main view; the chat stays docked at the bottom for follow-ups.
4. **Selection bubble:** highlight any text in a lesson → a bubble pops up to ask a question about the selection, with optional **image attachments** (button or clipboard paste), **links**, and **references to other workspace parts**. Answers can be **inserted into the workspace** as a note block.
5. **Adaptive workspace:** the more selection questions you ask, the more the app learns about you (top-right meter). At 100% it proposes concrete workspace changes you can Accept, Discuss, or Reject — applying changes **live** through a translucent stepper overlay.

All workspace + chat + selection-history + adaptive state **auto-saves to `localStorage`** and restores on reopen.

## Open it

- **Simplest:** double-click `index.html` (Chrome/Edge recommended — `localStorage` works on `file://`).
- **Or** a trivial static server: `python -m http.server 8000` in this folder → http://localhost:8000

## Demo mode (no key needed)

Demo mode is **ON by default** and simulates LLM responses locally, so the entire flow can be exercised offline:
workspace generation, quizzes, selection bubble with attachments, docked chat follow-ups, the **full adaptive
workspace cycle (meter → proposal → accept/discuss/reject → live stepper)**, and persistence.

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

## What's new in v1.1

### 5 UX fixes
1. **No emojis anywhere** — every button, label, chip, stepper and meter uses plain text or clean inline SVG icons (standing rule).
2. **Clipboard image paste** — paste (Ctrl+V) an image straight into the selection bubble; shares the same max‑2 path as the Image button.
3. **Refined bubble layout** — question input on top; attachments row (Image / Link / Reference) below; actions row (Ask / Insert into workspace / New question) under that.
4. **Viewable inserted notes** — click any note in the sidebar (or in a lesson) to open a note view with the source highlight, your question, the full answer, and a "Locate in lesson" button.
5. **Clipboard paste respects the max-2-images rule** (same shared path as the Image button).

### Option C — Selection Q&A history
The bubble shows **only the latest exchange** (question + answer + actions) and never grows downward.
The **full history** accumulates in the docked chat as an expandable **"Selection Q&A"** section — each entry keeps
the highlighted text, question, answer and timestamp. Older entries **never disappear**; "New question" clears only the bubble.

### Adaptive workspace (new feature — demo simulation, `LLMService` seam intact)
- **Understanding meter** (top-right): a circular "Learning more about you" indicator with a percentage. It fills
  only from **selection-bubble questions** (~25% each → proposal at ~4 questions); dock follow-ups do **not** count.
  The step weight is a single constant so the weighting can change later without touching UI code.
- **Threshold → change proposal:** at 100% a card appears: *"This is what you are asking for. Apply these changes
  to the workspace?"* with concrete edits derived from your real question history (e.g. "add a section for what you
  keep asking about", "add a recall check", "append an adaptive note to the mission").
  - **Accept** → runs the stepper overlay.
  - **Discuss** → opens a focused chat thread in the dock; asking for a revision reworks the proposal and re-presents it (canned demo responses; real = LLM call behind the same seam).
  - **Reject** → dismisses the proposal, the meter resets and starts accumulating again.
- **Translucent stepper overlay:** while changes apply you keep seeing the workspace; a subtle stepper in the top-right
  lists each change with status **queued → pending → changing → completed** as the page transforms live. Input is
  locked during the run except the **Stop** control, which halts remaining queued changes.
- All of it is demo-simulated with **no API key**; real generation/revise logic is a plain LLM call behind
  `LLMService.proposeChanges()` / `LLMService.reviseProposal()`.

## What works in v1.1

- ✅ Chat-first start state → workspace generation (real LLM **or** demo) with staged progress UI
- ✅ Chat docks to bottom; collapsible; follow-ups carry mission + outline + current-lesson context
- ✅ Workspace = mission, outline sidebar, numbered lesson pages, prev/next nav, interactive quizzes (correct/incorrect feedback + reveal)
- ✅ Selection bubble: highlight any lesson text → Q&A with **images** (button **or clipboard paste**, data-URLs, max 2), **links** (sent as references), and **referenced workspace parts** (choose a lesson)
- ✅ Refined bubble layout (question → attachments → actions)
- ✅ “Insert into workspace” — Q&A becomes a note block in that lesson, deletable, persisted, **and viewable** (sidebar click or note-card click → full content + locate)
- ✅ **Option C:** bubble shows only the latest exchange; full Selection Q&A history lives in the dock (expandable, never pruned, timestamped)
- ✅ **Adaptive workspace:** meter → proposal card (Accept/Discuss/Reject) → translucent stepper overlay with queued→pending→changing→completed statuses and a Stop control; changes apply live to the workspace and persist
- ✅ localStorage persistence: settings (incl. provider preset), workspace (incl. notes), dock chat, Selection Q&A history, **meter + proposal state** (refresh mid-flow doesn't lose position)
- ✅ “New topic” resets to the chat-first start state (clears workspace/notes/chat/history/adaptive)
- ✅ Demo mode: full offline simulation, zero network calls
- ✅ All LLM I/O isolated behind the `LLMService` seam (details below)

## Known limitations (v1.1)

- **No streaming** — responses arrive whole. (The `chat()` seam returns a `Promise<string>`, so streaming can be added inside the service later without touching UI.)
- **Links are not fetched** — pasted links are sent to the model as references; the page never fetches their content (browser CORS + scope). Stubbed per the spec.
- **Images inline via data URLs** — up to 2 per question; large images bloat `localStorage` persistence only if inserted as notes (the note stores text, not the image — images are request-only).
- **Markdown renderer is minimal** (headings, bold/italic, code, fenced code, lists, blockquotes, links, hr) — no tables/HTML passthrough by design (XSS-safe, everything is escaped).
- **Adaptive proposal is demo-simulated** — in Demo mode `proposeChanges()`/`reviseProposal()` derive changes from your real question history locally; with a live model they become ordinary LLM calls (JSON in/out). The demo proposal always appends sections / extends the mission — no in-place rewriting yet.
- **Mid-apply reload** — the stepper run itself isn't resumable: if you refresh mid-apply, already-applied changes stay in the workspace and the meter resumes cleanly at 0 (partially-applied state is preserved, not rolled back).
- **Key lives client-side** in `localStorage` — fine for a local prototype; move to a backend before sharing.
- **Real API from `file://`** — Chrome/Edge allow it; some browsers or providers (CORS) may block it — use the static-server option or a provider with open CORS.
- Workspace is **generated in one call** (single curriculum JSON); a staged generator (mission → resources → lessons) is a possible v2.
- Narrow screens hide the outline sidebar (desktop-first prototype).

## Architecture & the `LLMService` seam

```
 UI (markdown renderer, bubble, dock, workspace, adaptive meter/card/stepper)
   │  — knows nothing about fetch, endpoints, headers, keys
   ▼
 LLMService (index.html, one module)
   ├─ chat(messages, {json?, temperature?}) → Promise<string>   ← THE SEAM
   ├─ proposeChanges({workspace, selQA}) → Promise<{changes[], rationale}>   ← adaptive
   ├─ reviseProposal(proposal, messages) → Promise<{...proposal}>            ← adaptive discussion
   ├─ configure({baseUrl, model, apiKey, demo, provider})
   ├─ ping() / isDemo() / source() / getConfig()
   └─ two interchangeable implementations behind it:
        • 'demo'   — local simulation (no network; adaptive proposals derived from real history)
        • 'openai' — OpenAI / OpenRouter / any OpenAI-compatible
                     POST {baseUrl}/chat/completions
                     (headers, endpoint, auth all live HERE — nowhere else)
```

The rest of the app only ever calls `LLMService.chat(...)` / `proposeChanges(...)` / `reviseProposal(...)` with plain
message arrays / context objects:

```
[{role:'system'|'user'|'assistant', content: string | [{type:'text',text} | {type:'image_url',image_url:{url}}]}]
```

**The future backend swap** (Option C from the idea note): implement the same methods on a server —

- `POST /api/llm/chat` — server holds the key, forwards to the model, returns the text (optionally streams);
- `POST /api/llm/propose` / `POST /api/llm/revise` — adaptive proposal generation;
- `POST /api/llm/ping` — connectivity test;
- key never touches the page.

Then the in-page `LLMService` internals become thin `fetch('/api/llm/…', …)` calls — **zero UI changes**. The same
swap point can also take over persistence (workspace/chat/history/adaptive) when multi-device sync is needed.

## Files

| File | What |
|---|---|
| `index.html` | The entire app (~117 KB: CSS + HTML + JS inline) |
| `README.md` | This file |

## Tested (v1.1)

- JS extracted from the HTML passes `node --check` (syntax OK, 0 errors).
- Demo mode exercised end-to-end in real headless Edge via CDP — **94/94 checks pass**:
  start state → settings **provider presets** (OpenAI/OpenRouter/Custom + per-provider model dropdown + custom-model fallback, persisted) →
  workspace generated (4 lessons, mission, outline) → selection bubble opens on real DOM selection →
  **clipboard paste path** (synthesized paste event with image items; **max-2 enforced**, thumbs removable) →
  4 selection Q&As → **bubble shows only the latest exchange** while the dock **Selection Q&A** section accumulates
  2+ timestamped entries (new-question clears only the bubble) → **note inserted and opened via sidebar** (source
  highlight + question + answer + locate) → **meter 0%→100%** after 4 bubble questions → **proposal card** at 100%
  → **Reject resets meter** → second cycle → **Discuss opens dock thread**, revision request reworks the proposal
  (Revised badge, re-presented) → **Accept runs the translucent stepper** with queued→pending→changing→completed
  transitions observed live, input locked during the run, lesson + mission **actually changed** on the page →
  **reload restores** workspace (incl. applied changes), 8-entry Q&A history, chat, meter state → **Stop control**
  halts remaining queued changes → **New topic** resets and clears storage → rendered UI surfaces scanned
  emoji-free.
- No real API calls made during verification (no key on this machine); demo mode makes zero network calls.