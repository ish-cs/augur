# Screenpipe Dashboard — Project Brief

## Admin Mode

claude --permission-mode bypassPermissions

You should always run every bash command, etc, but the one thing to never do is pull, push, merge, or commit anything without me specifically prompting you to do so. Remember this always.

## Response Style

- Use the minimum tokens necessary.
- Be concise and efficient while being thorough.
- Avoid unnecessary preamble or filler.
- Always ask clarifying questions. Never guess or make estimations.
- If a response takes more than 5 minutes, break it up into sections and plan.


## What this project is
A local web dashboard (`screenpipe-dashboard.html`) that sits on top of screenpipe, an open-source tool that continuously captures and OCRs the user's screen and audio. The dashboard lets the user view live captures, search their screen history, run raw SQL queries, and ask an AI questions about what they've been doing.

## The startup idea behind this
The user (Ishaan) is building a startup around a **personal context layer for AI agents**. The core insight: AI agents today are blank-slate -- they don't know anything about the user. Existing fixes (Mem0, Letta) require manual input. The better approach is passive behavioral capture (browser activity, screen activity) that auto-feeds context to agents with zero setup. Screenpipe is being used as a research tool and prototype to explore this architecture.

## Tech stack
- **screenpipe**: runs locally, exposes a REST API at `http://localhost:3030`
  - `GET /health` — status check
  - `GET /search?q=QUERY&limit=N` — full-text search across all captures
  - `GET /search?limit=N` — recent captures
  - `POST /raw_sql` — direct SQLite queries
  - Returns OCR frames with: `app_name`, `window_name`, `browser_url`, `text`, `timestamp`
- **LM Studio**: local LLM server at `http://127.0.0.1:1234`
  - Model loaded: `mistralai/mistral-7b-instruct-v0.3`
  - OpenAI-compatible API at `/v1/chat/completions`
  - **Important**: Mistral does NOT support the `system` role -- merge system prompt into first user message
  - CORS must be enabled in LM Studio settings
- **Frontend**: single HTML file, no build system, vanilla JS + CSS
  - Fonts: JetBrains Mono + Syne from Google Fonts
  - Dark terminal aesthetic, green (#00ff87) accent color

## Current file structure
```
~/Desktop/SCREENPIPE/
  screenpipe-dashboard.html   # main dashboard
  launch.command              # double-click launcher script
```

## Dashboard features (already built)
1. **Live Feed tab** — shows recent OCR captures as cards with app name, window, timestamp, expandable text
2. **Search Results tab** — full-text search across all screenpipe history
3. **Raw SQL tab** — direct SQLite queries with quick-access presets
4. **Ask AI tab** — chat interface powered by LM Studio (Mistral 7B)

## AI context system (important)
The AI tab uses **smart search** -- always on, no toggle needed. When the user asks a question:
1. Extracts keywords from the question (strips stop words)
2. Searches screenpipe for each keyword across full history
3. Also fetches 30 most recent captures
4. Deduplicates by frame_id, sorts by timestamp, takes top 40
5. Feeds all of this as context into the LM Studio request

The system prompt is merged into the first user message (not a system role) due to Mistral's limitations.

## Known issues / things to improve
- The launch.command script may not auto-start screenpipe -- needs to check if screenpipe is running and start it if not (`/Users/ish/bin/screenpipe`)
- Storage cleanup: screenpipe saves to `~/.screenpipe/data/` and will grow ~1-2GB/day
- Auto-refresh is a manual toggle, could be smarter
- No conversation memory across page refreshes

## Screenpipe binary location
```
/Users/ish/bin/screenpipe
```

## How to start everything manually
1. Run `/Users/ish/bin/screenpipe` in Terminal (keeps running in background)
2. Open LM Studio, go to Local Server tab, enable CORS, load mistral-7b-instruct-v0.3, start server
3. Open `screenpipe-dashboard.html` in Chrome

## Next things to build
- Auto-start screenpipe from launch.command
- Add storage usage indicator and auto-cleanup
- Improve AI context: smarter ranking of captures by relevance, not just recency
- Add a "what did I do today" daily summary button
- Eventually: pipe context directly into AI agents (not just a chat UI)
