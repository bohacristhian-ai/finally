# FinAlly — AI Trading Workstation

## Project Specification

> **Companion documents.** This plan is the readable specification. Two documents hold the exact machine-readable contracts and are normative where they overlap with prose here:
> - **[`API_CONTRACT.md`](API_CONTRACT.md)** — exact request/response bodies, SSE payload, error shape, `actions` JSON.
> - **[`MARKET_DATA.md`](MARKET_DATA.md)** — simulator parameters, ticker seed table, Massive API request/response contract.
>
> A German translation exists at `PLAN.de.md`. **The English version is authoritative**; the translation reflects commit `6b568a9` and has not been updated with the revisions below.

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area, pre-populated with recent history from the server
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog, **no undo**
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

### On Next.js Specifically

This app is a single page with no routing, no SSR, and no server components — every Next.js feature beyond the build pipeline is switched off, and Vite would be a simpler tool for the job. **Next.js is retained deliberately** because it is part of the course curriculum and is the framework students are most likely to meet in practice.

**Agents must not "simplify" this away.** Do not migrate the frontend to Vite, do not enable SSR, do not add routing. If a task seems to call for one of these, raise it rather than acting on it.

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document — the readable specification
│   ├── API_CONTRACT.md       # Exact request/response, SSE payload, error shape
│   ├── MARKET_DATA.md        # Simulator parameters, seed prices, Massive contract
│   ├── REVIEW.md             # Review notes that produced this revision
│   └── *.de.md               # German translations (not authoritative)
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored)
├── .env.example              # Committed template; start scripts copy it to .env
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via a bind mount (see §11).
- **`planning/`** contains project-wide documentation. All agents reference files here as the shared contract. `PLAN.md` is the entry point; `API_CONTRACT.md` and `MARKET_DATA.md` are normative for the details they cover.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# OpenRouter API key — required for LLM chat only; the rest of the app runs without it
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend receives environment variables via `docker --env-file .env`. It does not read `.env` from disk itself.

### Missing or Broken API Key

`OPENROUTER_API_KEY` gates **chat only**. Prices, trading, portfolio, and watchlist all work without it. The app must therefore degrade gracefully rather than fail to start:

- The application boots normally with no key present
- `POST /api/chat` returns HTTP 200 with a friendly assistant message explaining that chat is unavailable, and an empty `actions` array — **not** a 500
- The same applies to upstream failures (timeout, rate limit, malformed response): the error surfaces as an assistant message, never as an unhandled exception
- The start scripts print a warning at launch when the key is absent

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

Exact parameters, the seed price table, and the Massive request/response contract live in **[`MARKET_DATA.md`](MARKET_DATA.md)**.

### Tracked Ticker Set

The background task tracks the union of three sources:

1. **The watchlist**
2. **Tickers with open positions** — a position in a ticker the user has removed from the watchlist still needs a live price, or its P&L silently freezes
3. **Tickers recently referenced by a trade request** — see *Admission on Trade* below

Removing a held ticker from the watchlist is permitted — it disappears from the watchlist but continues to be priced and continues to appear in the positions table and heatmap.

### Admission on Trade

A ticker that is on neither the watchlist nor in any position has no cached price. Requiring a price before accepting a trade, while only pricing tickers that are already watched or held, would make buying an unwatched ticker impossible: no price means rejection, and rejection means the position that would have caused it to be priced never exists.

So **a trade request admits its ticker to the tracked set before the price is resolved**, not after:

| Source | Behavior on admission |
|---|---|
| Simulator | A seed price is generated **synchronously** (deterministic per symbol, `MARKET_DATA.md`) and the ring buffer is backfilled. The trade proceeds within the same request |
| Massive | No price is known until the next poll. The trade is rejected with a **retry** message (§8), but the ticker is now tracked, so the next attempt — within one poll interval — succeeds |

Admission is idempotent and **sticky for 15 minutes**. A periodic sweep evicts tickers that are neither watchlisted nor held and have had no trade attempt within that window. Without the grace period a rejected Massive trade would evict its own ticker and the retry would fail forever; without the sweep, typos would accumulate in the tracked set indefinitely.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- **Correlation uses a single market factor**, not a sector taxonomy: `return = beta × market_factor + idiosyncratic`, with `beta` defaulting to 1.0. This produces convincing correlated movement and handles user-added tickers with no maintained sector map
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Unknown tickers (not in the seed table) get a generated seed price and default drift/volatility — see `MARKET_DATA.md`
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all tracked tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

A single background task (simulator or Massive poller) writes to an in-memory price cache. Per ticker the cache holds:

| Field | Purpose |
|---|---|
| `price` | Latest price |
| `prev` | Previous price — used only to compute direction server-side |
| `session_ref` | **Reference price for session change %** — the simulator's seed price, or in Massive mode the first price observed after startup |
| `updated_at` | ISO-8601 UTC timestamp |
| `history` | Ring buffer covering the last **60 minutes**, for the main chart |

**Session change %** is computed against `session_ref`, not against `prev`. Change versus the previous 500ms tick is a fraction of a percent and visually meaningless. The UI labels this "session change" rather than "daily change", which would be inaccurate.

**The ring buffer** backs `GET /api/history/{ticker}` so the main chart is populated the moment a ticker is selected, rather than filling in over the following minutes. It is seeded at startup with a synthetic backfill so the chart is never empty.

It is sized as a **duration, not a point count**. A flat "last 500 ticks" would mean ~4 minutes of history in simulator mode (500ms cadence) but ~2 hours under Massive's 15s free-tier poll — the same chart showing wildly different spans depending on an environment variable. Fixing the window at 60 minutes yields 7,200 points in simulator mode and 240 under Massive; `GET /api/history/{ticker}` downsamples to at most 500 points on read, so the response size is bounded regardless of source.

**On restart** the cache is rebuilt from scratch: simulated prices return to their seed values while `avg_cost` persists in SQLite, producing a visible jump in P&L and a discontinuity in the snapshot chart. **This is accepted, not a bug** — do not add price persistence to "fix" it.

### One Loop, Not Two

A **single** background loop ticks the price model and then broadcasts the result. There are not separate update and broadcast timers — two independent ~500ms loops would drift against each other and could emit duplicate or skipped ticks.

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- **One batched event per tick** containing all changed tickers — not one event per ticker
- **Only tickers whose price actually changed are included.** In Massive mode this cuts traffic roughly 30× (a 15s poll against a 500ms broadcast cadence would otherwise resend identical prices) and makes the client's flash logic trivially correct: every ticker in the payload changed, so every ticker in the payload flashes
- A comment heartbeat is emitted every ~15 seconds to hold the connection open through proxies, even when no prices changed
- Client handles reconnection automatically (EventSource has built-in retry)

Exact payload shape: **[`API_CONTRACT.md`](API_CONTRACT.md)**.

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Connection Settings

A background task writes portfolio snapshots every 30 seconds while request handlers write concurrently. With SQLite's defaults this produces intermittent `database is locked` errors that are painful to diagnose. Therefore, at connection setup:

- Enable **WAL mode** (`PRAGMA journal_mode=WAL`)
- Set a **busy timeout** (`PRAGMA busy_timeout=5000`)

### Timestamp Format

All timestamps are **ISO-8601 UTC with a `Z` suffix and millisecond precision** (`2026-08-11T14:00:00.000Z`). Ordering `portfolio_snapshots` and `chat_messages` relies on lexicographic sort of these TEXT columns, which only works if every writer uses an identical format.

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded **immediately at startup**, every 30 seconds thereafter, and immediately after each trade execution. The startup snapshot guarantees the chart always has at least one point instead of being empty for the first 30 seconds.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — outcomes of trades and watchlist changes, including failures; null for user messages — shape in `API_CONTRACT.md`)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

All request and response bodies are specified exactly in **[`API_CONTRACT.md`](API_CONTRACT.md)**. All errors return `{"error": "human-readable message"}` with an appropriate 4xx status.

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |
| GET | `/api/history/{ticker}` | Recent price history from the ring buffer (for the main chart) |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}`. Returns the executed trade **and** the updated portfolio |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time. Accepts `?limit=` (default 500), oldest-first |

> **Three different limits appear in this spec and all three are intentional**: the LLM sees the last **20** chat messages (context budget, §9), `GET /api/chat` returns **50** (panel scrollback), and `GET /api/portfolio/history` returns **500** (chart resolution). Do not unify them.

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/chat` | Recent conversation history, so the panel survives a page refresh |
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed action outcomes) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check. Returns `{"status": "ok"}` with HTTP 200. Used by the Docker `HEALTHCHECK` |

### Trading Rules & Validation

Manual trades and LLM-initiated trades **share one code path**. Every rule below applies identically to both.

**Accounting**
- Cost basis is **weighted average**. On a buy: `avg_cost = (old_qty × old_avg + qty × price) / (old_qty + qty)`
- On a sell, `avg_cost` is **unchanged**; only `quantity` and `cash_balance` move
- **Realized P&L is not tracked.** There is no column for it and no tax lots. Only unrealized P&L (against `avg_cost`) is displayed
- A position row is **deleted** when quantity reaches 0, rather than kept as a zero row

**Money precision**

Cash and prices are `REAL`, so float drift is real. Round cash and trade cost to **2 decimal places on every write**, and compare with a small epsilon (`cost <= cash + 0.005`). Without this, a legitimate "sell everything, then buy with all cash" sequence gets rejected because cash sits at `-0.0000001`.

**Execution order.** Price resolution happens *between* the cheap checks and the balance checks, because admitting the ticker is what makes a price available in the first place (§6):

1. Validate `side` and `quantity` — no ticker knowledge needed
2. Validate the symbol format, then **admit the ticker to the tracked set** (§6, idempotent)
3. Resolve the price: from the cache, or generated on demand in simulator mode
4. Validate cash (buys) or held quantity (sells) against that price
5. Execute, snapshot the portfolio, return the fill

**Rejection cases** — all return 400 with `{"error": ...}`:

| Condition | Message |
|---|---|
| `side` not `"buy"` or `"sell"` | Side must be buy or sell |
| `quantity <= 0` | Quantity must be greater than zero |
| Symbol fails format validation | Invalid ticker symbol |
| No price yet for a newly admitted ticker (Massive mode only) | No price available for {ticker} yet — try again in a moment |
| Buy with `cost > cash + epsilon` | Insufficient cash |
| Sell with `quantity > position quantity` | Insufficient shares |

The fourth case **cannot occur in simulator mode**, where step 3 always produces a price. It exists only for the Massive path, where the first attempt on an unknown ticker races the poll interval. It is a retry, not a refusal, and the message says so.

**Permitted**
- Fractional quantities, from both the trade bar and the LLM
- Buying a ticker that is **not** on the watchlist — step 2 admits it, so it is priced from that moment on. It does not join the watchlist; it becomes visible through the resulting position

**Fill price** is the price resolved at step 3, which may differ slightly from the price the user saw. The response returns the actual fill price so the UI shows what really happened.

**Ticker validation** (`POST /api/watchlist` and LLM-initiated additions) — see `MARKET_DATA.md` for the symbol rules. Tickers are uppercased and trimmed; adding a ticker already on the watchlist returns 200 idempotently rather than 409; the watchlist is capped at 50 tickers.

---

## 9. LLM Integration

When writing code to make calls to LLMs, use the **`cerebras` skill** (at `.claude/skills/cerebras/`) to reach the `openrouter/openai/gpt-oss-120b` model via LiteLLM → OpenRouter with Cerebras as the inference provider. Structured Outputs are used to interpret the results.

> Invoke the skill as `cerebras` — that is the name the runtime exposes, and it differs from the `name:` field inside the skill's own frontmatter.

There is an `OPENROUTER_API_KEY` in the `.env` file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the **last 20 messages** from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the `cerebras` skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response, **recording the outcome of each**
7. Stores the message and the action outcomes in `chat_messages`
8. Returns the message plus the action outcomes to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (§8)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:

- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

### Reporting Failed Actions

**The assistant's `message` is generated before any trade executes.** If a trade then fails validation, the message may confidently describe a trade that never happened. There is no second LLM call to correct this, and one must not be added — it would double latency and cost for a case the UI can handle deterministically.

Instead, **outcomes are rendered by the frontend from the `actions` array**, not narrated by the model:

- Each executed action carries a `status` of `"ok"` or `"error"` plus, on failure, the validation message
- The chat panel renders these as inline chips beneath the assistant's message — green for success, red for failure, e.g. `✗ BUY 10 AAPL — Insufficient cash`
- A failed action is therefore always visible to the user even when the model's prose contradicts it

This is cheaper, faster, and impossible to hallucinate. Shape in `API_CONTRACT.md`.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:

- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables fast, free, reproducible E2E tests; development without an API key; and CI/CD pipelines.

**The mock contract is a hard dependency of the E2E suite** (§12 asserts that a trade execution appears inline), so it is specified rather than left to the implementer:

| Input message | Mock response |
|---|---|
| Matches `/(buy\|sell)\s+(\d+(?:\.\d+)?)\s+([A-Za-z]{1,5})/i` | A canned confirmation message **plus** that trade in `trades`, which then executes through the normal validation path |
| Matches `/(add\|remove)\s+([A-Za-z]{1,5})/i` | A canned message plus the corresponding `watchlist_changes` entry |
| Anything else | A fixed portfolio-summary message, no actions |

Because mocked trades run through real validation, an E2E test can assert both the success and the failure chip by choosing quantities that fit or exceed the $10,000 balance.

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), **session change %** (computed against `session_ref`, see §6), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, price over time, **pre-populated from `GET /api/history/{ticker}`** on selection and extended live from the SSE stream
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history (loaded from `GET /api/chat` on mount), loading indicator while waiting for LLM response. Action outcomes shown inline as success/failure chips (§9)
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Selected Ticker

A **single selected-ticker state** drives both the main chart and the trade bar's ticker field. Clicking a ticker in the watchlist populates both, so buying what you are looking at takes one field of typing rather than two.

### Empty States

A fresh install has no positions, which is exactly what a first-time user sees. Both must render deliberately:

- **Positions table**: a centered hint — no positions yet, use the trade bar or ask the assistant
- **Portfolio heatmap**: same treatment, not a blank rectangle or a zero-area treemap
- **P&L chart**: renders the startup snapshot as a flat line at $10,000 rather than an empty axis

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- **Charting: Recharts, for every chart in the app** — main chart, sparklines, P&L line chart, and the portfolio treemap. Recharts ships a `Treemap` component; Lightweight Charts does not, and would force a second library into the bundle for the heatmap alone. One library, one set of idioms
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it. Every ticker in an SSE payload has changed by definition (§6), so no client-side comparison is needed
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - HEALTHCHECK: curl -f http://localhost:8000/api/health
      --interval=30s --timeout=3s --start-period=10s --retries=3
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Development Workflow

Agents will not rebuild a Docker image for every frontend change. Development therefore runs the two halves separately:

```bash
# Terminal 1 — backend
cd backend && uv run uvicorn app.main:app --reload --port 8000

# Terminal 2 — frontend
cd frontend && npm run dev        # serves on :3000
```

To keep this same-origin and avoid introducing CORS configuration that production does not need, `next.config.js` proxies the API in development only:

```js
async rewrites() {
  return process.env.NODE_ENV === 'development'
    ? [{ source: '/api/:path*', destination: 'http://localhost:8000/api/:path*' }]
    : [];
}
```

Frontend code calls `/api/*` in both modes, with **no environment-dependent base URL anywhere in the application code**. SSE works through the proxy unchanged.

### Docker Volume

The SQLite database persists via a bind mount to the repository's `db/` directory (§4):

```bash
docker run -v "$(pwd)/db:/app/db" -p 127.0.0.1:8000:8000 --env-file .env finally
```

A bind mount is used rather than a named volume so that students can see, inspect, and delete `db/finally.db` directly — worth more in a teaching context than the portability a named volume would buy.

The port binds to **`127.0.0.1` only**. The app has no authentication, holds an API key, and executes trades; there is no reason to expose it to the local network.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Copies `.env.example` to `.env` if `.env` is missing, and prints which values need filling in — `--env-file` fails with an opaque Docker error otherwise, which is a poor first-run experience
- Warns if `OPENROUTER_API_KEY` is empty (the app still runs; chat is unavailable — §5)
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the bind mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the database (data persists in `db/`)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss, float rounding at the cash boundary)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow, failed actions recorded with `status: "error"`
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering, loading state, and success/failure action chips

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Isolation**: the app container mounts an **anonymous (or tmpfs) volume at `/app/db`**, never the repository's `db/` directory. Every run therefore starts from a fresh lazily-seeded database. Without this the suite is not repeatable — each scenario mutates persistent state, so a second run starts with a different cash balance and watchlist and the "$10k balance" assertion fails. It also exercises the lazy-init path on every run.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism (mock contract in §9).

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- AI chat failure path: request a trade exceeding available cash, assert the red failure chip renders
- SSE resilience: disconnect and verify reconnection

**Forcing an SSE disconnect**: Playwright cannot kill a live `EventSource` directly. Use route interception:

```js
await page.route('**/api/stream/prices', route => route.abort());
// assert the status dot turns yellow, then red
await page.unroute('**/api/stream/prices');
// assert EventSource retry reconnects and the dot returns to green
```

---

## 13. Build Order

The contracts in `API_CONTRACT.md` and `MARKET_DATA.md` are frozen, which is what lets the frontend and backend proceed in parallel without waiting on each other. Build against the contract, not against the other agent's code.

### Stage 1 — Foundation (blocks everything)

| # | Work | Owner |
|---|---|---|
| 1.1 | `backend/` uv project, FastAPI app, `/api/health`, static file serving | Backend |
| 1.2 | SQLite schema, lazy init, seed data, WAL + busy_timeout (§7) | Backend |
| 1.3 | `frontend/` Next.js project, static export config, dev rewrites proxy (§11) | Frontend |
| 1.4 | `Dockerfile`, `.env.example`, start/stop scripts (§11) | Backend |

Stage 1 is done when `docker run` serves an empty page at `:8000` and `/api/health` returns `{"status":"ok"}`.

### Stage 2 — Parallel Tracks

Once Stage 1 lands, these run concurrently. Each depends only on the frozen contracts.

**Track A — Market data & streaming**
- `MarketDataSource` interface, `SimulatorSource` with the one-factor model and seed table (`MARKET_DATA.md`)
- Price cache with `session_ref`, 60-minute ring buffer, synthetic backfill
- Single background loop: tick → cache → broadcast changed only (§6)
- `GET /api/stream/prices`, `GET /api/history/{ticker}`
- `MassiveSource` last — it is optional, unverified in places, and nothing depends on it

**Track B — Portfolio & trading**
- `GET /api/portfolio`, `POST /api/portfolio/trade` with the five-step execution order (§8)
- Watchlist CRUD with ticker admission (§6)
- Snapshot task: startup, every 30s, after each trade
- `GET /api/portfolio/history`

**Track C — Frontend shell**
- Layout, dark theme, Tailwind config, header with connection dot
- `EventSource` hook, price flash, watchlist grid with sparklines
- Recharts: main chart, P&L line, treemap
- Positions table, trade bar, empty states (§10)

**Track D — Unit tests**
Written alongside each track by its owner, not deferred to a separate pass.

### Stage 3 — Integration

| # | Work | Depends on |
|---|---|---|
| 3.1 | LLM chat: `cerebras` skill, structured output, action execution + outcomes (§9) | B |
| 3.2 | `LLM_MOCK` mode per the §9 contract | 3.1 |
| 3.3 | Chat panel with success/failure chips, `GET /api/chat` on mount | 3.1, C |
| 3.4 | Playwright E2E with fresh-DB isolation (§12) | all |

### Working Agreements

- **The contracts are frozen.** If implementation reveals a contract is wrong, change `API_CONTRACT.md` / `MARKET_DATA.md` first and say so — do not diverge silently and do not work around it locally
- **Do not migrate off Next.js, Recharts, or SSE** (§3, §10). These were decided deliberately; the reasoning is recorded where each is specified
- **`planning/REVIEW.md`** holds the open items with priorities. R6 (Massive response field names) is the only unverified area of the spec and is confined to Track A's last task
- **Record progress in `planning/PROGRESS.md`** — one line per completed item, so a fresh agent can see the state without reading the whole tree
