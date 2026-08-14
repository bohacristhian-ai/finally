# API Contract

Exact request and response shapes for every endpoint in [`PLAN.md`](PLAN.md) §8. **This document is normative** — where prose in `PLAN.md` and a shape here disagree, this document wins.

Frontend and backend agents build against this file independently. Do not change a shape here without updating both sides.

## Conventions

- All bodies are JSON; all requests that carry a body send `Content-Type: application/json`
- All timestamps are ISO-8601 UTC, `Z` suffix, millisecond precision: `2026-08-11T14:00:00.000Z`
- All monetary values are numbers rounded to 2 decimal places
- Quantities are numbers and may be fractional
- Tickers are always uppercase in responses, and are uppercased/trimmed on input

### Errors

Every error response, on every endpoint:

```json
{ "error": "Insufficient cash" }
```

| Status | Used for |
|---|---|
| 400 | Validation failure (bad quantity, insufficient funds/shares, unknown side, no price available) |
| 404 | Ticker not on the watchlist (DELETE), unknown ticker (history) |
| 500 | Unexpected server fault only — never used for expected conditions such as a missing LLM key |

---

## `GET /api/stream/prices`

Server-Sent Events. One batched event per tick, containing **only tickers whose price changed** since the previous tick (`PLAN.md` §6).

```
event: prices
data: {"ts":"2026-08-11T14:00:00.000Z","prices":[{"ticker":"AAPL","price":190.12,"dir":"up","session_ref":189.40},{"ticker":"TSLA","price":242.88,"dir":"down","session_ref":245.10}]}

:heartbeat

event: prices
data: {"ts":"2026-08-11T14:00:00.500Z","prices":[{"ticker":"NVDA","price":121.44,"dir":"up","session_ref":120.00}]}
```

**Event object**

| Field | Type | Notes |
|---|---|---|
| `ts` | string | Timestamp of the tick |
| `prices` | array | One entry per changed ticker; never empty (a tick with no changes emits nothing) |

**Price entry**

| Field | Type | Notes |
|---|---|---|
| `ticker` | string | Uppercase |
| `price` | number | Current price |
| `dir` | string | `"up"` or `"down"`. Drives the flash color. No `"flat"` value exists — unchanged tickers are not sent |
| `session_ref` | number | Session reference price, for computing session change % client-side |

The previous price is **not** transmitted. The client rendered it and therefore already has it; `dir` carries the only derived fact the client needs, and sending three representations of the same thing creates three ways to disagree.

**Heartbeat**: a bare SSE comment line (`:heartbeat`) every ~15 seconds keeps the connection open through proxies during quiet periods. `EventSource` ignores comments; no client handling is required.

---

## `GET /api/history/{ticker}`

Recent price history from the server-side ring buffer, so the main chart is populated on selection.

**Response 200**

```json
{
  "ticker": "AAPL",
  "points": [
    { "t": "2026-08-11T13:59:58.000Z", "price": 189.98 },
    { "t": "2026-08-11T13:59:58.500Z", "price": 190.04 }
  ]
}
```

Oldest first. The server holds 60 minutes of history (`PLAN.md` §6) and **downsamples to at most 500 points on read**, so the response size is bounded whether the source is the 500ms simulator or a 15s Massive poll. Returns 404 with the standard error shape if the ticker is not tracked.

---

## `GET /api/portfolio`

```json
{
  "cash_balance": 8091.20,
  "total_value": 10142.75,
  "unrealized_pnl": 142.75,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 10,
      "avg_cost": 190.12,
      "current_price": 194.40,
      "market_value": 1944.00,
      "unrealized_pnl": 42.80,
      "pnl_pct": 2.25,
      "weight_pct": 19.17
    }
  ]
}
```

| Field | Notes |
|---|---|
| `total_value` | `cash_balance` + sum of `market_value` |
| `unrealized_pnl` | Sum of position `unrealized_pnl`. Realized P&L is not tracked (`PLAN.md` §8) |
| `pnl_pct` | Against `avg_cost` |
| `weight_pct` | Position `market_value` as a percentage of `total_value`. Drives treemap rectangle size |

`positions` is `[]` when there are no holdings — never `null`.

---

## `POST /api/portfolio/trade`

**Request**

```json
{ "ticker": "AAPL", "quantity": 10, "side": "buy" }
```

**Response 200** — the executed trade *and* the resulting portfolio, so the client re-renders from a single response rather than firing a follow-up `GET /api/portfolio`.

```json
{
  "trade": {
    "id": "9f8e7d6c-...",
    "ticker": "AAPL",
    "side": "buy",
    "quantity": 10,
    "price": 190.12,
    "executed_at": "2026-08-11T14:00:00.000Z"
  },
  "portfolio": { "...": "identical shape to GET /api/portfolio" }
}
```

`trade.price` is the actual fill price resolved at execution time, which may differ from the price the user saw on screen.

**Unwatched tickers are accepted.** The ticker is admitted to the tracked set *before* its price is resolved (`PLAN.md` §6), so a buy on a symbol that is on neither the watchlist nor in any position succeeds normally. In simulator mode the price is generated synchronously and the trade completes in the same request.

**Errors** — 400 with the standard shape, per the rejection table in `PLAN.md` §8. One of them is a retry rather than a refusal:

```json
{ "error": "No price available for PYPL yet — try again in a moment" }
```

This occurs **only in Massive mode**, on the first trade against a ticker that was not previously tracked, when the request arrives before the next poll. The ticker is tracked from that moment, so a retry within one poll interval succeeds. Clients should present it as a transient condition — a retry affordance, not a validation failure. It never occurs in simulator mode.

---

## `GET /api/portfolio/history`

Query: `?limit=` (optional, default 500, max 5000).

```json
{
  "points": [
    { "t": "2026-08-11T13:30:00.000Z", "total_value": 10000.00 },
    { "t": "2026-08-11T13:30:30.000Z", "total_value": 10012.40 }
  ]
}
```

Oldest first. When `limit` truncates, the **most recent** `limit` points are returned, still oldest-first.

---

## `GET /api/watchlist`

```json
{
  "tickers": [
    {
      "ticker": "AAPL",
      "price": 190.12,
      "session_ref": 189.40,
      "added_at": "2026-08-11T13:00:00.000Z"
    }
  ]
}
```

**Session change % is not returned.** The client computes it from `price` and `session_ref` — the same two fields the SSE stream delivers on every tick (`PLAN.md` §6). Returning a server-computed `change_pct` here as well would give the same number two independent derivations, and any rounding difference between them would show as a value that jitters on the first tick after page load.

`price` and `session_ref` are `null` if the ticker has no cached price yet; the client renders a placeholder rather than 0%.

---

## `POST /api/watchlist`

**Request**

```json
{ "ticker": "pypl" }
```

**Response 200** — the created or already-existing entry. Adding a duplicate is **idempotent**, returning 200 rather than 409.

```json
{ "ticker": "PYPL", "added_at": "2026-08-11T14:01:00.000Z" }
```

**Errors**: 400 if the symbol fails validation (trimmed/uppercased, must match `^[A-Z]{1,5}$`; see `PLAN.md` §8 and `MARKET_DATA_SUMMARY.md`) or the 50-ticker cap is reached.

---

## `DELETE /api/watchlist/{ticker}`

**Response 200**

```json
{ "ticker": "PYPL", "removed": true }
```

404 if the ticker is not on the watchlist. Removing a ticker with an open position is permitted — it remains priced and remains in the positions table (`PLAN.md` §6).

---

## `GET /api/chat`

Conversation history, so the panel survives a page refresh. Query: `?limit=` (optional, default 50).

```json
{
  "messages": [
    { "id": "...", "role": "user", "content": "buy 10 AAPL", "actions": null, "created_at": "2026-08-11T14:00:00.000Z" },
    { "id": "...", "role": "assistant", "content": "Bought 10 AAPL.", "actions": [], "created_at": "2026-08-11T14:00:01.000Z" }
  ]
}
```

Oldest first. `actions` is `null` for user messages and an array (possibly empty) for assistant messages.

---

## `POST /api/chat`

**Request**

```json
{ "message": "buy 10 AAPL and add PYPL to my watchlist" }
```

**Response 200**

```json
{
  "message": "Bought 10 AAPL at $190.12 and added PYPL to your watchlist.",
  "actions": [
    { "type": "trade", "ticker": "AAPL", "side": "buy", "quantity": 10, "price": 190.12, "status": "ok" },
    { "type": "watchlist", "ticker": "PYPL", "action": "add", "status": "ok" }
  ]
}
```

Always 200, including when the LLM is unavailable or a requested action fails (`PLAN.md` §5, §9).

### The `actions` array

This is the contract that lets the UI report outcomes the model cannot know, because `message` is generated *before* execution (`PLAN.md` §9). It is stored verbatim in `chat_messages.actions`.

**Trade action**

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"trade"` |
| `ticker` | string | Uppercase |
| `side` | string | `"buy"` or `"sell"` |
| `quantity` | number | As requested by the model |
| `price` | number \| null | Actual fill price; `null` when `status` is `"error"` |
| `status` | string | `"ok"` or `"error"` |
| `error` | string | Present only when `status` is `"error"`; the same message a manual trade would return |

**Watchlist action**

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"watchlist"` |
| `ticker` | string | Uppercase |
| `action` | string | `"add"` or `"remove"` |
| `status` | string | `"ok"` or `"error"` |
| `error` | string | Present only when `status` is `"error"` |

**Failure example** — note the prose contradicts reality, which is exactly the case the chips exist to cover:

```json
{
  "message": "Bought 500 AAPL for you.",
  "actions": [
    { "type": "trade", "ticker": "AAPL", "side": "buy", "quantity": 500,
      "price": null, "status": "error", "error": "Insufficient cash" }
  ]
}
```

The frontend renders `✗ BUY 500 AAPL — Insufficient cash` in red beneath the message.

**LLM unavailable**

```json
{
  "message": "Chat is unavailable — no OpenRouter API key is configured. Everything else in FinAlly works normally.",
  "actions": []
}
```

---

## `GET /api/health`

```json
{ "status": "ok" }
```

HTTP 200. Used by the Docker `HEALTHCHECK`.
