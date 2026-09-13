# Teachly — Teaching Platform Prototype v1

A **single self-contained HTML file** (zero build step, zero CDNs, zero hosting) that starts as a chat and
*builds itself* into a teaching workspace the moment you send your first message.

Product flow (per the vision in `2026-09-12-teaching-platform-idea.md`):

1. **Start = chat.** Type what you want to be taught and send it.
2. The chat **drops to a bottom dock** while a **workspace** (mission + outline + navigable lesson pages with quizzes) is generated behind a visible progress panel.
3. The workspace becomes the main view; the chat stays docked at the bottom for follow-ups.
4. **Selection bubble:** highlight any text in a lesson → a bubble pops up to ask a question about the selection, with optional **image attachments** (vision), **links**, and **references to other workspace parts**. Answers can be **inserted into the workspace** as a note block.

All workspace + chat state **auto-saves to `localStorage`** and restores on reopen.

## Open it

- **Simplest:** double-click `index.html` (Chrome/Edge recommended — `localStorage` works on `file://`).
- **Or** a trivial static server: `python -m http.server 8000` in this folder → http://localhost:8000

## Demo mode (no key needed)

Demo mode is **ON by default** and simulates LLM responses locally, so the entire flow can be exercised offline:
workspace generation, quizzes, selection bubble with attachments, docked chat follow-ups, persistence.

- Toggle it from the **🧪 Demo mode chip** on the start screen or in ⚙️ Settings.
- While in demo mode the app makes **zero network calls**.

## Using your own LLM (OpenAI / OpenRouter / any OpenAI-compatible endpoint)

1. Click **⚙️ Settings**.
2. Pick a preset (**OpenAI** or **OpenRouter**), or paste your own **Base URL** (must end in `/v1`-style root, e.g. `https://api.openai.com/v1` or `https://openrouter.ai/api/v1`).
3. Enter a **model** (e.g. `gpt-4o-mini`, `openrouter/auto`). Vision (image attachments) needs a vision-capable model.
4. Paste your **API key** → **Save** → **Test connection** to verify before use.
5. Turn **Demo mode off**.

> ⚠️ **v1 key handling:** the key is stored in this browser's `localStorage` and sent only to the endpoint you configure. That's acceptable for a local prototype but not for a shared product — the intended v1→v2 path is a backend swap (below) that holds the key server-side.

## What works in v1

- ✅ Chat-first start state → workspace generation (real LLM **or** demo) with staged progress UI
- ✅ Chat docks to bottom; collapsible; follow-ups carry mission + outline + current-lesson context
- ✅ Workspace = mission, outline sidebar, numbered lesson pages, prev/next nav, interactive quizzes (correct/incorrect feedback + reveal)
- ✅ Selection bubble: highlight any lesson text → Q&A with **images** (data-URLs, max 2), **links** (sent as references), and **referenced workspace parts** (choose a lesson)
- ✅ “Insert into workspace” — Q&A becomes a note block in that lesson, deletable, persisted
- ✅ localStorage persistence: settings, workspace (incl. notes), dock chat; restored on reopen
- ✅ “New topic” resets to the chat-first start state
- ✅ Demo mode: full offline simulation, zero network calls
- ✅ All LLM I/O isolated behind the `LLMService` seam (details below)

## Known limitations (v1)

- **No streaming** — responses arrive whole. (The `chat()` seam returns a `Promise<string>`, so streaming can be added inside the service later without touching UI.)
- **Links are not fetched** — pasted links are sent to the model as references; the page never fetches their content (browser CORS + scope). Stubbed per the spec.
- **Images inline via data URLs** — up to 2 per question; large images bloat `localStorage` persistence only if inserted as notes (the note stores text, not the image — images are request-only).
- **Markdown renderer is minimal** (headings, bold/italic, code, fenced code, lists, blockquotes, links, hr) — no tables/HTML passthrough by design (XSS-safe, everything is escaped).
- **Key lives client-side** in `localStorage` — fine for a local prototype; move to a backend before sharing.
- **Real API from `file://`** — Chrome/Edge allow it; some browsers or providers (CORS) may block it — use the static-server option or a provider with open CORS.
- Workspace is **generated in one call** (single curriculum JSON); a staged generator (mission → resources → lessons) is a possible v2.
- Narrow screens hide the outline sidebar (desktop-first prototype).

## Architecture & the `LLMService` seam

```
 UI (markdown renderer, bubble, dock, workspace) 
   │  — knows nothing about fetch, endpoints, headers, keys
   ▼
 LLMService (index.html, one module)
   ├─ chat(messages, {json?, temperature?}) → Promise<string>   ← THE SEAM
   ├─ configure({baseUrl, model, apiKey, demo})
   ├─ ping() / isDemo() / source() / getConfig()
   └─ two interchangeable implementations behind it:
        • 'demo'   — local simulation (no network)
        • 'openai' — OpenAI / OpenRouter / any OpenAI-compatible
                     POST {baseUrl}/chat/completions
                     (headers, endpoint, auth all live HERE — nowhere else)
```

The rest of the app only ever calls `LLMService.chat(...)` with plain message arrays:

```
[{role:'system'|'user'|'assistant', content: string | [{type:'text',text} | {type:'image_url',image_url:{url}}]}]
```

**The future backend swap** (Option C from the idea note): implement the same three methods on a server —

- `POST /api/llm/chat` — server holds the key, forwards to the model, returns the text (optionally streams);
- `POST /api/llm/ping` — connectivity test;
- key never touches the page.

Then the in-page `LLMService` internals become a thin `fetch('/api/llm/chat', …)` — **zero UI changes**. The same swap point can also take over persistence (workspace/chat) when multi-device sync is needed.

## Files

| File | What |
|---|---|
| `index.html` | The entire app (~76 KB: CSS + HTML + JS inline) |
| `README.md` | This file |

## Tested

- JS extracted from the HTML passes `node --check` (syntax OK, 0 errors).
- Demo mode exercised end-to-end in real headless Chromium via CDP — **22/22 checks pass**: start state → topic send → workspace generated (4 lessons, mission, outline, 2 interactive quizzes) → chat docked → selection-bubble Q&A with image + link + cross-lesson reference (payload verified to route through the `LLMService` seam with `text` + `image_url` parts) → answer rendered → note inserted into lesson → docked-chat follow-up answered by the demo simulator → **reload → workspace, note, and chat all restored from `localStorage`**.
- Layout probe: sidebar at x=0 (264px), dock bottom-aligned over the main area, selection bubble fully inside the viewport, and the lesson scroll area reserves room so content clears the expanded dock.
- No real API calls made during verification (no key on this machine); demo mode makes zero network calls.