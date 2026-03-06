# Screenpipe Dashboard — Product Document

## What This Is

A local web dashboard that sits on top of [screenpipe](https://github.com/mediar-ai/screenpipe), an open-source tool that continuously captures and OCRs the user's screen and audio. The dashboard gives Ishaan a live window into everything screenpipe is capturing, the ability to search that history, run arbitrary SQL against the underlying database, and ask an AI questions about what he's been doing — all running entirely on-device with no cloud dependency.

---

## The Startup Idea Behind It

Ishaan is building a startup around a **personal context layer for AI agents**.

The core problem: AI agents today are blank-slate. Every conversation starts from zero. They know nothing about who the user is, what they've been doing, or what they care about. Existing memory solutions (Mem0, Letta) require manual input — the user has to tell the AI things for it to remember them.

The better approach: **passive behavioral capture** that auto-feeds context to agents with zero setup. If an agent can see that you've been reading about venture-backed SaaS pricing for the last three hours, it doesn't need you to explain your context. It already knows.

Screenpipe is the research vehicle and prototype for this architecture. The dashboard is the interface for exploring what that captured data looks like, how useful it is, and how to make it queryable by agents.

The long-term vision: pipe this context directly into AI agents (not just a chat UI), creating a personal context layer that any agent can pull from automatically.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    User's Mac                           │
│                                                         │
│  ┌──────────────┐    REST API      ┌──────────────────┐ │
│  │  screenpipe  │ ◄──────────────► │  Dashboard HTML  │ │
│  │  (binary)    │  localhost:3030  │  (browser)       │ │
│  │              │                  │                  │ │
│  │  SQLite DB   │                  │  Vanilla JS      │ │
│  │  ~/.screenpipe│                 │  No build step   │ │
│  └──────────────┘                  └────────┬─────────┘ │
│                                             │           │
│  ┌──────────────┐    OpenAI API             │           │
│  │  LM Studio   │ ◄─────────────────────────┘           │
│  │  (app)       │  localhost:1234                       │
│  │              │                                       │
│  │  Any LLM     │                                       │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

Everything runs locally. No data leaves the machine. No API keys required.

---

## File Structure

```
~/Desktop/SCREENPIPE/
  screenpipe-dashboard.html   # The entire dashboard — one self-contained HTML file
  launch.command              # Double-click launcher (Python script)
  PRODUCT.md                  # This file
```

Supporting binaries / apps (not in this repo):
```
~/bin/screenpipe              # screenpipe binary
LM Studio.app                 # GUI app for running local LLMs
~/.screenpipe/data/           # Raw video/audio captured by screenpipe (~1-2 GB/day)
~/.screenpipe/db.sqlite       # SQLite database with all OCR + audio transcription records
~/.screenpipe/launcher.log    # Log file written by launch.command
```

---

## Tech Stack

### screenpipe
- Runs as a background process; exposes a REST API at `http://localhost:3030`
- Continuously captures the screen via OCR and microphone via Whisper transcription
- Stores everything in SQLite

**Key API endpoints used:**
| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Status, last frame timestamp, device info |
| `/search?q=QUERY&limit=N` | GET | Full-text search across all captures |
| `/search?limit=N&offset=0` | GET | Recent captures (no query = all) |
| `/raw_sql` | POST | Direct SQLite query (`{ query: "SELECT ..." }`) |

**API response shape (OCR frame):**
```json
{
  "type": "OCR",
  "content": {
    "frame_id": 12345,
    "timestamp": "2026-03-05T17:13:00Z",
    "app_name": "Google Chrome",
    "window_name": "Hacker News - Google Chrome",
    "browser_url": "https://news.ycombinator.com",
    "text": "...full OCR text of the screen..."
  }
}
```

**API response shape (Audio):**
```json
{
  "type": "Audio",
  "content": {
    "timestamp": "2026-03-05T17:13:00Z",
    "device_name": "MacBook Pro Microphone (input)",
    "transcription": "...Whisper transcription..."
  }
}
```

### LM Studio
- Desktop app for running local LLMs with an OpenAI-compatible server
- Runs at `http://127.0.0.1:1234`
- **Model detection:** On load, the dashboard calls `GET /v1/models`, filters out embedding models (anything with "embed" in the ID), and uses the first LLM it finds. The loaded model ID is passed in every `POST /v1/chat/completions` request. No model is hardcoded — switching models in LM Studio is automatically picked up.
- Context length: Models loaded in LM Studio should have "Context Length" set to 8192+ in their settings to avoid 400 errors from oversized prompts.

**Important:** The dashboard merges the system prompt into the first user message rather than using a `{ role: "system" }` entry. This ensures compatibility across all model architectures (some, like older Mistral variants, don't support the system role).

### Dashboard Frontend
- Single HTML file: `screenpipe-dashboard.html`
- No framework, no build system, no npm, no dependencies to install
- Vanilla JavaScript + CSS
- Fonts: JetBrains Mono (monospace) + Syne (display) from Google Fonts
- Opens directly in any browser via `file://` protocol

---

## UI Design System

### Visual Language
- **Aesthetic:** Dark terminal / hacker dashboard. Feels like a monitoring tool, not a consumer app.
- **Background:** `#0a0a0a` (near-black)
- **Surfaces:** `#111111`
- **Borders:** `#1e1e1e` (default) / `#2a2a2a` (bright)
- **Text:** `#e8e8e8` (primary) / `#555` (dim) / `#333` (dimmer)
- **Accent green:** `#00ff87` — used for active states, live indicators, success, primary actions
- **Red:** `#ff3b3b` — errors, offline, danger actions
- **Amber:** `#ffb800` — warnings, storage cleanup
- **Blue:** `#4d9fff` — OCR type badge, URLs

### Scanline Overlay
A CSS pseudo-element on `body` renders subtle horizontal scanlines across the entire UI using a repeating gradient. Adds to the terminal/CRT aesthetic without being distracting.

### Animations
- **`pulse`**: breathing glow on the logo dot and status indicators
- **`slideIn`**: cards and messages animate up from 8px below on render
- **`spin`**: loading spinner for in-progress operations
- **`typingBounce`**: three-dot bounce animation in the AI typing indicator

### Layout
Two-column grid: 280px fixed sidebar + fluid main content area. Full viewport height minus the sticky header.

---

## Dashboard Features

### Header (sticky, always visible)
- **Logo:** "screenpipe" in Syne font with a pulsing green dot
- **Status pill:** Shows `● RECORDING` (green) or `● offline` (red) based on `/health` poll every 5 seconds. When offline, an error banner appears in the main content area.
- **Last capture time:** Shows "last capture Xs ago" using `last_frame_timestamp` from the health response

---

### Sidebar

#### Capture Stats (4 stat cards)
Updated on every feed load:
- **Frames** — count of OCR-type records in the current page
- **Audio chunks** — count of Audio-type records in the current page
- **Total records** — `pagination.total` from the API (all-time total)
- **Uptime** — wall-clock time since the dashboard was opened (counts up in real time via `setInterval`)

#### Controls
Five buttons, top to bottom:
1. **Refresh feed** — re-fetches the live feed, shows a green toast
2. **Auto-refresh: OFF/ON** — toggles a 5-second interval that reloads the feed. Button turns green when active.
3. **Export JSON** — fetches the 100 most recent records and triggers a browser download of `screenpipe-export-{timestamp}.json`
4. **Today's Summary** — see "AI Features" below. Green-accented button.
5. **Stop screenpipe** — shows a `confirm()` dialog explaining how to actually stop the process (since the dashboard can't signal the binary directly — it runs as a detached process)

#### Devices
Dynamically populated from `device_status_details` in the health response. Parses the comma-separated device string, shows each device with a green `● active` or red `○ inactive` badge.

#### Storage
Dynamically queried from the SQLite database via `/raw_sql`:
- **Frames** — total row count from `frames` table
- **Audio chunks** — total row count from `audio_transcriptions` table
- **Est. DB size** — estimated at ~30KB per frame. Shown in MB or GB. Progress bar turns amber at 5GB, red at 10GB.
- **Last 7 days** — collapsible `<details>` element showing per-day frame counts
- **Cleanup controls** — dropdown (Keep last 7 / 14 / 30 days) + "Clean" button. On confirm, runs `DELETE FROM frames WHERE timestamp < datetime('now', '-N days')` and the equivalent on `audio_transcriptions`. Refreshes stats after. Shows a note that raw video/audio files in `~/.screenpipe/data/` require manual deletion (the dashboard can only clean DB records).
- Storage stats auto-refresh every 60 seconds.

---

### Search Bar (persistent, above tabs)
- Full-width text input with a `$` prefix (terminal aesthetic)
- `Enter` key or "SEARCH →" button triggers `doSearch()`
- Result limit selector: 10 / 20 / 50 / 100
- Switching to search results tab automatically when a search is triggered

---

### Tab 1: Live Feed
- Fetches `GET /search?limit=20&offset=0` (20 most recent captures of any type)
- Renders each result as a card (see Card component below)
- Empty state: "Capturing..." with subtext when screenpipe is running but has no data yet
- Error state: "Connection failed" with instructions

### Tab 2: Search Results
- Triggered by the search bar
- Shows result count: `N results for "query"` in green
- Highlights matched search terms in card text using `<span class="highlight">`
- Empty state: "No results for X"

### Tab 3: Raw SQL
- Direct `POST /raw_sql` interface
- Text input + "RUN →" button, `Enter` to submit
- Results rendered as an HTML `<table>` with all columns auto-detected from the first row
- Row count shown below table
- Two quick-access preset queries:
  - **Top apps by screen time:** `SELECT app_name, window_name, COUNT(*) as count FROM frames GROUP BY app_name ORDER BY count DESC LIMIT 20`
  - **Recent frames:** `SELECT * FROM frames ORDER BY timestamp DESC LIMIT 10`
- Clicking a preset auto-fills and switches to the SQL tab

### Tab 4: Ask AI (✦)
See "AI Features" section below.

---

### Card Component
Used in Live Feed and Search Results tabs. Each card shows one OCR frame or audio chunk.

**Structure:**
- **Header:** Type badge (OCR in blue / Audio in amber), app name + window name (truncated to 40 chars), timestamp (time + date)
- **Body:** The OCR text or audio transcription. Clamped to 120px height with a gradient fade-out at the bottom.
- **Footer:** Browser URL (blue, truncated) if present. "expand ↓ / collapse ↑" toggle button.

**Expand/collapse:** Clicking the button removes the max-height clamp and gradient overlay, revealing the full text. Each card has a unique random ID so multiple can be independently expanded.

**Animation:** Cards slide in from 8px below on render (`slideIn` keyframe, 0.2s ease-out).

---

## AI Features

### Architecture: Smart Context System

Every AI question goes through a two-phase process before hitting the LLM:

**Phase 1 — Keyword extraction:**
The question is lowercased, stripped of punctuation, and split into words. A hardcoded stop-word list filters out common words (`what`, `have`, `i`, `been`, `the`, etc.). Up to 5 meaningful keywords are extracted.

**Phase 2 — Context assembly:**
- `GET /search?limit=20` — 20 most recent captures (always included)
- `GET /search?q={keyword}&limit=10` — per-keyword targeted search, run in parallel for all keywords
- All results merged, deduplicated by `frame_id` or `timestamp`

**Phase 3 — Relevance scoring:**
Each candidate capture is scored:
```
score = (keyword_match_count × 3) + recency_score
```
- `keyword_match_count`: regex match count across `text + transcription + app_name + window_name`, capped at 5 per keyword to prevent spam domination
- `recency_score`: `max(0, 1 - age / 24h)` — 1.0 for captures in the last hour, decaying to 0 at 24h+
- Items are sorted by score descending, top 20 taken

**Phase 4 — Prompt construction:**
The top 20 captures are formatted as:
```
[HH:MM:SS] [AppName / WindowName]
<first 150 chars of text>
```
This block is appended to the system prompt as "Screen capture context (N captures, relevance-ranked for: keyword1, keyword2...)".

The system prompt + context is merged into the first user message (not a `system` role entry) for universal model compatibility.

### Chat Interface
- Message bubbles: user messages right-aligned (green tint), AI messages left-aligned (dark surface)
- Avatars: "you" badge (green) / "AI" badge (dim)
- Timestamps on each message
- Typing indicator: three dots bouncing (CSS animation) while waiting for the LLM
- `Enter` to send, `Shift+Enter` for newline
- Auto-growing textarea (min 44px, max 120px)
- Send button disabled while a response is in flight (prevents double-sends)
- Chat history: last 8 messages passed as context for multi-turn conversation

**Markdown rendering (basic):**
- `**bold**` → `<strong>`
- `` `code` `` → `<code>` with green monospace styling
- Newlines → `<p>` tags

**Suggested questions** (shown before first message, hidden after):
- "What have I been working on today?"
- "What apps did I use most?"
- "Summarize my recent browsing"
- "What was I reading about?"
- "Any todos or action items on screen?"

**Model badge** (top right of AI tab): shows live LM Studio status and the actual model ID. Green when connected, red when offline.

### Context Bar
Shows `✦ smart search · all history` — always-on indicator that context is being pulled automatically. The model badge on the right shows the detected model name.

### Today's Summary (sidebar button)
One-click structured daily digest:
1. Switches to the AI tab
2. Queries today's frames via SQL: `SELECT app_name, window_name, text, timestamp FROM frames WHERE date(timestamp) = '{today}' ORDER BY timestamp DESC LIMIT 60`
3. Queries app usage breakdown: `SELECT app_name, COUNT(*) as cnt FROM frames WHERE date(timestamp) = '{today}' GROUP BY app_name ORDER BY cnt DESC LIMIT 10`
4. Assembles a structured prompt asking the model to output:
   - **Main Activities** — what was the user primarily working on?
   - **Apps & Tools Used** — brief breakdown
   - **Topics & Content** — what were they reading, browsing, or researching?
   - **Action Items Spotted** — todos, tasks, or follow-ups visible on screen
5. Sends at `temperature: 0.5` (more deterministic than the chat's 0.7) with `max_tokens: 1000`
6. Gracefully handles "no data today" with a helpful message

### Error Handling
- **400 from LM Studio** — shown with a specific message: "The prompt may exceed your model's context window. In LM Studio → Local Server, increase 'Context Length' to 8192 or higher."
- **Network error** — shown with steps to check LM Studio is running
- **No captures today** — shown as an AI message instead of crashing

---

## Launch Script (`launch.command`)

Python 3 script. Double-click in Finder (or run in Terminal) to start everything.

**What it does:**
1. Checks if screenpipe binary exists at `~/bin/screenpipe`
2. Checks if screenpipe is already running (`GET http://localhost:3030/health`)
3. If not running: spawns it with `subprocess.Popen` in a new session (detached from the terminal), logging to `~/.screenpipe/launcher.log`
4. Polls every 1.5 seconds for up to 10 retries (15 seconds total) waiting for screenpipe to start
5. Checks LM Studio on port 1234 — warns if not running but doesn't block
6. Checks that `screenpipe-dashboard.html` exists in the same directory
7. Opens the dashboard in the default browser via `webbrowser.open(file://...)`
8. Enters a keep-alive loop printing live status (`[screenpipe: up] [LM Studio: not running]`) every 10 seconds
9. Ctrl+C exits the launcher cleanly while screenpipe keeps running in the background

**To stop screenpipe:** run `pkill screenpipe` in Terminal.

---

## Data Model (SQLite via screenpipe)

Two primary tables accessed by the dashboard:

**`frames`**
| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `timestamp` | TEXT | ISO 8601 datetime |
| `app_name` | TEXT | Frontmost application |
| `window_name` | TEXT | Window title |
| `browser_url` | TEXT | URL if browser was active |
| `text` | TEXT | Full OCR text of the frame |

**`audio_transcriptions`**
| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `timestamp` | TEXT | ISO 8601 datetime |
| `device_name` | TEXT | Microphone name |
| `transcription` | TEXT | Whisper transcription |

---

## Known Limitations

### Storage
- The dashboard can only delete SQLite DB records via `/raw_sql`. The raw video/audio files screenpipe saves to `~/.screenpipe/data/` must be manually deleted via Finder or `rm -rf ~/.screenpipe/data/*`. Data accumulates at roughly 1–2 GB/day.

### AI Context Window
- LM Studio's context length setting must be 8192 or higher for the AI features to work reliably. The default is often 2048–4096, which causes 400 errors. The dashboard detects this and surfaces the fix in the error message.

### AI Chat Memory
- Chat history is session-only (in-memory). Refreshing the page resets the conversation. Only the last 8 messages are passed to the LLM as context.

### Model Compatibility
- The system prompt is always merged into the first user message rather than passed as a `system` role. This works universally but means later messages in a conversation don't carry the system context (it's only injected on the first turn).

### Search Quality
- Screenpipe's `/search` endpoint is full-text search against OCR text. It has no semantic understanding — it finds exact string matches. The relevance scoring in the dashboard improves result ordering but doesn't make the underlying search smarter.

### No Real-Time Feed
- The "LIVE" badge is cosmetic — the feed doesn't stream. Auto-refresh polls every 5 seconds; the feed itself only shows the 20 most recent captures at the time of each poll.

### Stop Button
- The "Stop screenpipe" button cannot actually kill the screenpipe process from the browser. It shows a `confirm()` dialog with instructions to press Ctrl+C in the terminal. This is a browser security constraint (no access to OS processes).

---

## Potential Next Steps

These are not built yet:

1. **Pipe context to AI agents** — the actual product goal. Instead of a chat UI, expose the screenpipe context via an API that any AI agent can pull from. The dashboard's `getScreenpipeContext()` function is essentially the prototype for this.

2. **Semantic search** — embed captures into a vector store (e.g., Chroma running locally) so search finds conceptually related content, not just string matches.

3. **Auto-cleanup of raw files** — the launch.command script could run a cron-style cleanup of `~/.screenpipe/data/` files older than N days, mirroring what the dashboard does for DB records.

4. **Persistent conversation memory** — save chat history to `localStorage` so conversations survive page refreshes.

5. **Timeline view** — a chronological visualization of what apps were open when, like a gantt chart of the day. The raw SQL tab already enables ad-hoc queries for this.

6. **Context API endpoint** — a tiny local HTTP server (could be added to launch.command) that exposes `GET /context?q=query` returning relevance-ranked screenpipe captures. Any agent or tool could call this.

7. **Browser extension** — capture the full URL + page title + selected text as additional context signals beyond what screenpipe OCRs.
