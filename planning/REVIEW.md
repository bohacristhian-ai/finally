# PLAN.md — Documentation Review

*Review of `planning/PLAN.md`. Nothing below is a change to the spec — these are open questions and recommendations for the plan's author to accept or reject. Each item states a recommendation so it can be resolved by decision rather than discussion.*

**Repository state at review time:** greenfield. `backend/`, `frontend/`, `test/`, and `db/` are all empty; `PLAN.md` is the only substantive content. This review therefore assesses the plan as a *contract for parallel agents to build against*, not as a description of existing code.

The plan is strong on vision, stack rationale, and directory boundaries. The gaps are almost all in **inter-agent contracts**: places where the Frontend and Backend agents would each make a reasonable but incompatible guess. Those are listed first because they block parallel work.

---

## 1. Contract Gaps That Block Parallel Agents

**Q1. The SSE event shape is not specified.** §6 says "each SSE event contains ticker, price, previous price, timestamp, and change direction" but not whether that is one event per ticker or one batched event for all tickers per tick, nor the SSE `event:` name, nor the exact JSON keys. Two agents will guess differently and the integration will fail.
*Recommendation:* pin one batched event per tick — fewer messages, one render pass on the client, and it makes "which tickers exist" self-describing:
```
event: prices
data: {"ts":"2026-08-11T14:00:00.000Z","prices":[{"ticker":"AAPL","price":190.12,"prev":190.05,"dir":"up"}]}
```

**Q2. There is no endpoint to read chat history.** `chat_messages` is persisted with an `actions` column, but §8 exposes only `POST /api/chat`. On page refresh the conversation vanishes from the UI while still sitting in the database.
*Recommendation:* add `GET /api/chat` returning recent messages, or drop the table and hold the conversation in frontend state only. The first is a two-line endpoint and matches the persistence already specified — prefer it.

**Q3. The `actions` JSON shape is undefined.** The frontend renders "trade executions and watchlist changes shown inline as confirmations" (§10), so the shape is a hard contract, and it must carry *outcomes* (including failures — see Q17), not just the LLM's requested actions.
*Recommendation:* specify it in the plan, e.g. `[{"type":"trade","ticker":"AAPL","side":"buy","quantity":10,"price":190.12,"status":"ok"}, {"type":"trade",...,"status":"error","error":"insufficient cash"}]`.

**Q4. The Docker volume section contradicts the directory structure section.** §4 says top-level `db/` "is the runtime volume mount point" with a `.gitkeep`; §11 shows `docker run -v finally-data:/app/db`, a *named* volume that does not touch the host `db/` directory at all. Only one can be true.
*Recommendation:* use a bind mount `-v "$(pwd)/db:/app/db"`. It matches §4, and for a course it is worth a lot that students can see, inspect, and delete `db/finally.db` directly.

**Q5. E2E tests as written are not repeatable.** Every listed scenario mutates persistent state (buys shares, adds tickers). A second run starts from a different cash balance and watchlist, so assertions like "$10k balance shown" fail.
*Recommendation:* have `docker-compose.test.yml` mount an anonymous/tmpfs volume at `/app/db` so each run gets a fresh lazily-seeded database. This is cleaner than adding a reset endpoint and exercises the lazy-init path on every test run.

**Q6. `POST /api/portfolio/trade` response shape is unspecified.** The UI needs the actual fill price (the cache may have ticked since the displayed price) and the resulting cash/position.
*Recommendation:* return the executed trade plus the updated portfolio, so the client re-renders from one response instead of firing a follow-up `GET /api/portfolio`.

---

## 2. Market Data

**Q7. Where does the *main chart* get its history?** §2 is explicit that sparklines accumulate client-side from SSE since page load, but §10's "larger chart for the currently selected ticker" has no stated data source and there is no history endpoint or price table. As written, clicking a ticker shows an empty chart that fills in over the following minutes — a weak first impression for the flagship visual.
*Recommendation:* keep a server-side rolling ring buffer (say the last 500 ticks per tracked ticker, in memory alongside the price cache) and expose `GET /api/history/{ticker}`. It costs very little and means the chart is populated the moment it is opened. Optionally pre-seed the buffer with a synthetic backfill at startup so the chart is never empty.

**Q8. "Daily change %" is not computable from the specified cache.** §10 asks the watchlist to show daily change %, but the cache holds only latest price, previous price (the last ~500ms tick) and timestamp. Change versus the previous tick is fractions of a percent — visually meaningless.
*Recommendation:* define a per-ticker **session reference price** (the simulator's seed price, or the first price observed after startup in Massive mode), store it in the cache, and compute change % against it. Name it "session change" if "daily" is misleading.

**Q9. What happens when an unknown ticker is added?** In simulator mode there is no universe of symbols — `POST /api/watchlist {"ticker":"ZZZZ"}` has no seed price, no drift, no volatility. The LLM can add tickers too, through the same path.
*Recommendation:* state the rule explicitly. Simplest that stays impressive: a small table of ~30 known symbols with realistic seeds, and any unknown symbol gets a generated seed price (e.g. uniform $20–$400) with default drift/vol. Also specify normalization (uppercase, trim), a max watchlist size, and the duplicate-add response (409 vs idempotent 200).

**Q10. Which tickers does the background task actually track?** §6 says "all tickers known to the system … equivalent to the user's watchlist". But a position in a ticker the user has since removed from the watchlist still needs a live price, or its P&L silently freezes.
*Recommendation:* define the tracked set as `union(watchlist, tickers with open positions)`, and allow removing a held ticker from the watchlist (blocking it would be surprising).

**Q11. Massive's polling interval and the SSE cadence disagree.** Free tier polls every 15s while SSE pushes every 500ms, so the same price is re-sent ~30 times and the client must not flash on each. The Massive request/response contract (base URL, auth header, endpoint, JSON shape) is also entirely unspecified — the agent will have to guess or invent it.
*Recommendation:* (a) push only tickers whose price actually changed, plus an SSE comment heartbeat every ~15s to hold the connection open through proxies; (b) add a short `planning/MARKET_DATA.md` with the real Massive request/response contract before the market-data agent starts.

---

## 3. Portfolio & Trading

**Q12. Accounting rules are implied but never stated.** Specifically: is cost basis weighted-average (the `avg_cost` column suggests yes)? Is `avg_cost` left unchanged on a sell? Is realized P&L tracked anywhere (no column exists for it)? Is a position row deleted at quantity 0 or kept as a zero row? §2 says "position updates or disappears," implying deletion.
*Recommendation:* state plainly: weighted-average cost, `avg_cost` untouched on sells, no realized-P&L tracking and no tax lots, position row deleted when quantity reaches 0.

**Q13. Float money will drift.** Cash and prices are `REAL`. A sequence of fractional trades can leave cash at `-0.0000001` and a naive `cash >= cost` check will reject a legitimate "sell everything / buy with all cash" flow.
*Recommendation:* round cash and cost to 2dp on every write and compare with a small epsilon. One sentence in the plan prevents a class of confusing bugs.

**Q14. Trade validation rules are unlisted.** Quantity must be > 0; are fractional quantities accepted from the trade bar (the schema supports them)? What is returned if the ticker has no cached price yet? What if the ticker is not in the watchlist — is buying an unwatched ticker allowed?
*Recommendation:* enumerate the rejection cases and their status codes (400 with a message) once, so manual trades and LLM trades demonstrably share one code path — which §9 already promises.

**Q15. `GET /api/portfolio/history` has no parameters.** Snapshots accrue at 2,880 rows/day and the response grows without bound.
*Recommendation:* accept `?limit=` (default e.g. 500) and return newest-last. Retention/downsampling is unnecessary at course scale, but the limit is worth having.

**Q16. The P&L chart is empty for the first 30 seconds and flat until the first trade.** Worth acknowledging.
*Recommendation:* write a snapshot immediately on startup (in addition to every 30s and after each trade) so the chart always has at least one point, and specify the empty-state rendering.

---

## 4. Chat & LLM

**Q17. The auto-execution ordering contradicts the error-handling promise.** §9 step 4 generates `message`, step 6 then executes the trades, and step 8 returns the response — but the text was written *before* execution, so if a trade fails the assistant's message may cheerfully confirm a trade that never happened. "The error is included in the chat response so the LLM can inform the user" cannot happen without a second LLM call, which the flow does not have.
*Recommendation:* do not add a second call. Render outcomes deterministically in the UI from the `actions` array (Q3) — a red inline chip reading `✗ BUY 10 AAPL — insufficient cash` next to the assistant's message. Cheaper, faster, and impossible to hallucinate. The plan should say this explicitly.

**Q18. "Recent conversation history" is unquantified.** Unbounded growth eventually breaks the context window.
*Recommendation:* fix it at the last N messages (N ≈ 20) and say so.

**Q19. Mock mode behavior is undefined, but E2E depends on it.** §12 requires the mocked-chat test to assert "trade execution appears inline," which means the mock must be able to emit a trade action — yet §9 says only "deterministic mock responses."
*Recommendation:* specify the mock contract in the plan, e.g. a message matching `/(buy|sell) (\d+) ([A-Z]+)/i` returns a canned message plus that trade; anything else returns a fixed portfolio-summary message with no actions. This is a contract between the backend agent and the E2E agent and belongs in writing.

**Q20. What happens when `OPENROUTER_API_KEY` is missing or the LLM call fails?** §5 calls it "required," but everything except chat works without it.
*Recommendation:* degrade gracefully — the app boots, chat returns a friendly error message in the message bubble rather than a 500, and the start scripts warn at launch.

---

## 5. Frontend & Dev Workflow

**Q21. There is no development loop.** The plan describes only the production single-container setup. Agents iterating on the frontend will not rebuild a Docker image per change, so they need `next dev` on :3000 talking to `uvicorn` on :8000 — which reintroduces the cross-origin problem the architecture was designed to avoid.
*Recommendation:* document a dev mode using Next.js `rewrites` to proxy `/api/*` to `localhost:8000`. Same-origin in dev, no CORS config, no code differences between dev and prod. This is the single most valuable addition for build velocity.

**Q22. Two charting libraries are offered ("Lightweight Charts or Recharts") but they are not interchangeable** — and neither the heatmap/treemap nor the sparklines are assigned to one.
*Recommendation:* choose Recharts for everything. It ships a `Treemap` component (Lightweight Charts does not), covers the line charts and sparklines, and one library beats two. If canvas performance for the main chart matters more, choose Lightweight Charts *and* name a separate treemap approach.

**Q23. Does clicking a watchlist ticker also populate the trade bar?** Unspecified, and it is the difference between a fluid demo and a clunky one.
*Recommendation:* yes — one selected-ticker state drives the main chart and the trade bar's ticker field.

**Q24. Empty states are unspecified** for the positions table and heatmap on a fresh start (which is exactly what a first-time user sees).
*Recommendation:* one line each describing the zero-position rendering.

**Q25. How does the E2E "SSE resilience" test force a disconnect?** Playwright cannot trivially kill a live EventSource.
*Recommendation:* use `page.route('**/api/stream/prices', r => r.abort())` to break it, then unroute and assert the status dot returns to green — and name that technique in the plan so the test agent does not improvise.

---

## 6. Infrastructure & Ops

**Q26. SQLite concurrency is not addressed.** A 30-second snapshot task writes concurrently with request handlers; with default settings this produces intermittent `database is locked` errors that are painful to diagnose.
*Recommendation:* one line — enable WAL mode and set `busy_timeout` at connection setup.

**Q27. Timestamps are `TEXT` with no format pinned.** Ordering `portfolio_snapshots` and `chat_messages` relies on lexicographic sort, which only works if every writer uses the same UTC format.
*Recommendation:* mandate ISO-8601 UTC with a `Z` suffix and millisecond precision everywhere.

**Q28. `docker run -p 8000:8000` binds all interfaces,** exposing an unauthenticated app that holds an API key and executes trades to the local network.
*Recommendation:* bind `-p 127.0.0.1:8000:8000` in the start scripts. No downside for local use.

**Q29. `.env` is required by `--env-file` but gitignored,** so a fresh clone fails at first run with a Docker error rather than a helpful message.
*Recommendation:* have the start scripts copy `.env.example` → `.env` when absent and print what to fill in. Also §5 hedges between mounting `.env` and using `--env-file` — pick `--env-file` only.

**Q30. In-memory price cache means a container restart resets simulated prices to seed values,** producing a visible jump in P&L and a discontinuity in the snapshot chart while `avg_cost` persists.
*Recommendation:* accept it and say so in one line, or persist last prices on shutdown. Accepting is fine — but it should be a stated decision so an agent does not "fix" it unprompted.

---

## 7. Simplification Opportunities

**S1. Merge the two 500ms timers.** §6 describes the simulator updating the cache at ~500ms *and* SSE pushing at ~500ms — two independent loops that will drift and can emit duplicate or skipped ticks. Use one loop: tick the model, then broadcast the result. Fewer moving parts, no drift.

**S2. Replace the sector taxonomy with a single market factor.** "Correlated moves across tickers (e.g., tech stocks move together)" implies maintaining a sector map — which cannot cover arbitrary user-added tickers anyway. A one-factor model (`return = beta × market_factor + idiosyncratic`, beta defaulting to 1.0) produces visually convincing correlation with a fraction of the code and handles unknown symbols for free.

**S3. Push only changed prices.** Combined with S1, this removes the "did it actually change?" check from the client, makes the flash logic trivially correct, and cuts Massive-mode traffic ~30×.

**S4. Consider whether Next.js is earning its place.** The app is one page, no routing, no SSR, no server components, statically exported and served by FastAPI — every Next.js feature is switched off. Vite + React would build faster and have a simpler config. *However*, if the course intends to teach Next.js, that pedagogical reason outweighs the simplification and this should simply be recorded as a deliberate choice rather than left implicit.

**S5. One chart library, not two.** See Q22.

**S6. Drop `users_profile.created_at`.** It is written once and never read. Minor, but the schema is a contract several agents will implement against and unused columns invite speculative code.

**S7. Drop `previous price` from the SSE payload** if `dir` is already sent. The client has the previous price by definition — it rendered it. Sending three representations of the same fact (`prev`, `dir`, and the implicit prior value) creates three ways to disagree.

---

## 8. Smaller Nits

- §4's tree omits `.env.example`, though §5 and §11 both depend on it.
- §8 lists no error responses anywhere. A single "all errors return `{"error": "..."}` with 4xx" line covers the whole API.
- `GET /api/health` has no specified response body; if it feeds a Docker `HEALTHCHECK`, say so.
- §2 promises "no confirmation dialog" for trades and §9 repeats it for LLM trades — consistent, but worth also stating there is no undo.
- CLAUDE.md says "Agents interact through files in `planning/`", yet `planning/` contains only PLAN.md and no convention for what else belongs there. Given the gaps above, the natural companions are `API_CONTRACT.md` (exact request/response and SSE JSON), `MARKET_DATA.md` (Massive contract + simulator parameters), and a running `PROGRESS.md`. Naming them in PLAN.md turns "agents coordinate through files" into something agents can actually act on.

---

## 9. Suggested Resolution Order

1. **Q1, Q3, Q6** — freeze the JSON contracts; everything else is downstream of these.
2. **Q4, Q21** — fix the volume contradiction and define the dev loop, or every agent pays a rebuild tax.
3. **Q7, Q8** — decide the history and change-% story; both change the backend's data model.
4. **Q17, Q19** — settle chat action reporting and the mock contract before the LLM and E2E agents diverge.
5. Everything else can be resolved inline during the build.

---

# Part II — Review of Uncommitted Changes vs. HEAD (`6b568a9`)

*Sections 1–9 above review `PLAN.md` as a specification. This part reviews the actual working-tree diff.*

**Scope of the diff.** Five untracked files, no modified or deleted files, nothing staged:

| File | Lines | Origin |
|---|---|---|
| `planning/REVIEW.md` | 163 → this file | Part I above |
| `planning/PLAN.de.md` | 459 | German translation of `PLAN.md` |
| `planning/REVIEW.de.md` | 163 | German translation of Part I |
| `CLAUDE.de.md` | 7 | German translation of `CLAUDE.md` |
| `README.de.md` | 3 | German translation of `README.md` |

All five are documentation. No source code, config, or schema was added or changed, so there is no runtime behavior to review — the risk surface is *documentation fidelity* and *which document agents will actually obey*.

**Disclosure:** the four translations were produced in this same session, so this part is a self-review. Mechanical parity was verified with tooling (results below); judgments about German phrasing are not independently verified and should be spot-checked by a native reader.

## 10. Verification Performed

Structural parity between `PLAN.md` and `PLAN.de.md`, checked mechanically:

| Check | English | German | Result |
|---|---|---|---|
| `##` headings | 13 | 13 | pass |
| Heading sequence | — | — | pass (line-for-line, none added/dropped/reordered) |
| Fenced code blocks | 12 | 12 | pass |
| Table rows | 27 | 27 | pass |
| Top-level bullets | 144 | 144 | pass |
| Blank lines | 116 | 119 | **+3** — see D4 |

All 16 sampled technical identifiers appear with identical occurrence counts in both files: the six endpoint paths, `cash_balance`, `avg_cost`, `portfolio_snapshots`, `chat_messages`, `users_profile`, `OPENROUTER_API_KEY`, `MASSIVE_API_KEY`, `LLM_MOCK`, `EventSource`, `finally.db`. No identifier was translated into German — the code contract is intact.

## 11. Findings

**D1. The English and German plans are now a fork with no stated authority — and agents will follow the English one.** `CLAUDE.md` (unchanged, tracked) still contains `@planning/PLAN.md`. That `@`-include is what actually loads the spec into every agent's context. `CLAUDE.de.md` contains `@planning/PLAN.de.md`, but **Claude Code only auto-loads `CLAUDE.md`** — the German file is never read, so its include never resolves. The German plan is therefore documentation for humans only, and the moment `PLAN.md` is edited the two silently diverge with nothing to detect it.
*Recommendation:* decide explicitly, and record the decision in both files. Either (a) English stays authoritative and each German file gets a header — `> Übersetzung von PLAN.md, Stand 6b568a9. Maßgeblich ist die englische Fassung.` — or (b) German becomes authoritative, in which case `CLAUDE.md` itself must be repointed to `@planning/PLAN.de.md` and the English retired. Leaving it implicit is the one option that reliably goes wrong.

**D2. `cerebras-inference` is not the invocable skill name.** *(New — this was missed in Part I and is a defect in the original `PLAN.md`, faithfully carried into `PLAN.de.md`.)* §9 instructs agents twice to "use cerebras-inference skill" (`PLAN.md:284`, `PLAN.md:295`). The skill's `SKILL.md` frontmatter does say `name: cerebras-inference`, but the directory is `.claude/skills/cerebras/` and the runtime exposes it to agents as **`cerebras`**. An agent following the plan literally would call `Skill(skill="cerebras-inference")` and fail.
*Recommendation:* align the three. Simplest is to correct `PLAN.md` (and `PLAN.de.md`) to say `cerebras`; alternatively rename the directory to match the frontmatter. This blocks the LLM-integration agent at its first step, so it is worth fixing before the build starts.

**D3. The translation faithfully preserves every defect found in Part I.** This is correct behavior for a translation, but it has a consequence worth stating: `PLAN.de.md` contains the same `db/` bind-mount vs. named-volume contradiction (§4 vs. §11), the same unspecified SSE payload, the same pre-execution `message` ordering in §9. **Any fix must now be applied twice.** This is the concrete cost of D1 and the main argument for resolving D1 before, not after, the spec fixes land.

**D4. Three cosmetic blank lines were added** in `PLAN.de.md` §9 (before the bullet lists under *Automatische Ausführung*, *Leitlinien für den System-Prompt*, and *LLM-Mock-Modus*). Renders identically in Markdown; noted only to account for the 456 → 459 line delta so it is not mistaken for dropped or added content.

**D5. `REVIEW.de.md` renumbers `Q` → `F` (Frage) with a strict 1:1 mapping** (Q1↔F1 … Q30↔F30, S1↔V1 … S7↔V7). Internal cross-references were verified consistent (`see Q17`↔`siehe F17`, `(Q3)`↔`(F3)`, `See Q22`↔`Siehe F22`, and the resolution order `Q1, Q3, Q6`↔`F1, F3, F6`). The hazard is *cross-document*: a reviewer citing "Q17" in a German-language discussion, or "F17" in an English one, will not be found by search.
*Recommendation:* one line at the top of `REVIEW.de.md` stating `F<n> entspricht Q<n> der englischen Fassung`.

**D6. `.env` placeholder text was translated** — `your-openrouter-api-key-here` → `dein-openrouter-api-schluessel-hier` inside the §5 bash block. Harmless (it is a placeholder meant to be overwritten), but it sits in a copy-pasteable code block. Flagging as a deliberate choice, not an oversight; revert if code blocks should stay byte-identical across languages.

**D7. `.claude/**/*.md` was deliberately left untranslated** — `agents/change-reviewer.md`, `commands/doc-review.md`, `skills/cerebras/SKILL.md`. These are executable configuration, not prose: skill `description` frontmatter drives whether a skill triggers at all, and command files are prompts. Translating them would change behavior, not just language. Recorded here so the omission reads as intentional rather than incomplete.

**D8. Nothing is committed.** All five files are untracked; a `git clean` would discard roughly 790 lines of work.
*Recommendation:* commit before further changes. Note the current branch is `start`, not `main`.

## 12. Assessment

No blocking defects in the translations themselves — structural parity is verified and the code contract is intact. The two items that need a decision rather than an edit are **D1** (which document is authoritative, given that `CLAUDE.md` still routes every agent to the English plan) and **D2** (the wrong skill name, which is a pre-existing `PLAN.md` bug that will stop the LLM agent immediately). D2 is a one-word fix and should be made in both language versions once D1 has settled how such fixes propagate.

---

# Part III — Review of the Revised Specification

*Reviews the changes made in response to Parts I and II: the `PLAN.md` rewrite (456 → 626 lines, +223/−53) and the two new companion documents.*

**Scope of the diff.** One modified file, two new ones, on top of the untracked files already covered in Part II:

| File | Lines | Change |
|---|---|---|
| `planning/PLAN.md` | 456 → 626 | Revised against all 30 questions, 7 simplifications, and D2 |
| `planning/API_CONTRACT.md` | 313 | New — normative request/response, SSE payload, `actions` shape |
| `planning/MARKET_DATA.md` | 156 | New — simulator parameters, seed table, Massive contract |

**Disclosure:** the author of these documents is the author of this review. The verification below is mechanical and reproducible; the design judgments are not independently checked.

## 13. Coverage Verification

Confirmed by grep that superseded decisions leave no residue and replacements are present:

| Was | Now | Status |
|---|---|---|
| "daily change %" | "session change" vs `session_ref` | resolved (only surviving mention explains the rejection) |
| Lightweight Charts *or* Recharts | Recharts throughout | resolved (only surviving mention explains the rejection) |
| `-v finally-data:/app/db` | bind mount `$(pwd)/db` | resolved, no occurrences remain |
| `cerebras-inference` | `cerebras` | resolved, no occurrences remain |
| `prev` in SSE payload | `dir` only | resolved, no occurrences remain |

Present and singular: WAL mode, `busy_timeout`, `127.0.0.1` binding, anonymous/tmpfs test volume, `page.route` abort technique, `next.config.js` rewrites, weighted-average cost, `session_ref` (×3), last-20-message context, Recharts.

Section numbering 1–12 was deliberately held stable, so the §-references in Parts I and II still resolve.

## 14. Defects Introduced by the Revision

**R1 — Buying an unwatched ticker is unreachable. (Blocking; self-inflicted.)** §8 lists under *Permitted*: "Buying a ticker that is **not** on the watchlist — it is added to the tracked set automatically by virtue of holding a position (§6)". But §6 defines the tracked set as `union(watchlist, tickers with open positions)`, and §8's rejection table rejects any trade where there is *no cached price for the ticker*. An unwatched ticker with no position is in neither half of the union, so it is never priced, so the trade is always rejected — and the position that would have made it tracked is never created. The permitted case is a deadlock that cannot execute.
*Recommendation:* make the trade path add the ticker to the tracked set and fetch/generate a price **before** validating, rather than requiring a price to already exist. In simulator mode that is immediate (a seed price is generated on demand, `MARKET_DATA.md`); in Massive mode the trade should return 400 *"No price available for {ticker} yet, try again in a moment"* and the ticker enters the tracked set so the next attempt succeeds. Whichever is chosen, §6 and §8 must state the same rule.

> **✅ Resolved.** `PLAN.md` §6 gains an *Admission on Trade* subsection: the tracked set now has three sources (watchlist, open positions, tickers recently referenced by a trade), and a trade admits its ticker **before** the price is resolved. §8 gains an explicit five-step execution order placing admission at step 2 and price resolution at step 3, and the unconditional *"No cached price"* rejection is replaced by a Massive-only **retry** case that cannot occur in simulator mode. Admission is sticky for 15 minutes with a periodic sweep, so a rejected Massive trade does not evict its own ticker and typos do not accumulate. `MARKET_DATA.md` adds `try_immediate_price()` to the source interface — synchronous seed price in the simulator, always `None` under Massive (spending the 5 calls/min budget on the trade path would starve the polling loop). `API_CONTRACT.md` documents the retry as a transient condition rather than a validation failure. Wording of the retry message verified identical across `PLAN.md` and `API_CONTRACT.md`.

**R2 — The main chart's time span silently depends on the data source.** The ring buffer is specified as a flat "last 500 ticks" (`PLAN.md` §6, `API_CONTRACT.md`, `MARKET_DATA.md` all agree on 500). At the simulator's 500ms cadence that is **~4 minutes** of history; at Massive's free-tier 15s poll it is **~2 hours**. Two problems: the flagship "detailed chart" shows four minutes in the default configuration, which is thin for something modelled on a Bloomberg terminal, and the same UI means different things depending on an environment variable.
*Recommendation:* specify the buffer as a **duration** rather than a count — e.g. 60 minutes of history, giving 7,200 points in simulator mode and 240 under Massive — and let the point count follow from the cadence. If memory is a concern, downsample on read in `GET /api/history/{ticker}` rather than shortening the window.

**R3 — `HEALTHCHECK` is referenced but never specified.** §8 states `/api/health` is "Used by the Docker `HEALTHCHECK`", but the Dockerfile stage description in §11 lists no `HEALTHCHECK` instruction, so nothing would actually call it.
*Recommendation:* add the instruction to the §11 stage list, with its interval and retry count, or drop the claim from §8.

**R4 — `change_pct` is computed in two places.** `GET /api/watchlist` returns a server-computed `change_pct`, while the SSE stream ships `session_ref` so the client can compute the same figure on every tick. Both are needed (initial paint vs. live updates), but nothing says they must agree, and a rounding difference between the two would show as a value that jitters on the first tick after load.
*Recommendation:* state that `change_pct` is a convenience for initial render only, and that once the SSE stream is live the client is the sole authority — or drop it from the watchlist response and have the client compute from `price` and `session_ref` in both cases. The second is simpler and removes the possibility of disagreement.

**R5 — Three different history limits, all deliberate, none explained together.** LLM context uses the last **20** messages (§9); `GET /api/chat` defaults to **50** (`API_CONTRACT.md`); `GET /api/portfolio/history` defaults to **500**. These serve different purposes and are individually correct, but an agent encountering them may "fix" the inconsistency.
*Recommendation:* one sentence noting the three are intentionally distinct — model context budget, chat panel scrollback, and chart resolution.

## 15. Carried-Forward Risks

**R6 — The Massive API contract is an educated guess.** `MARKET_DATA.md` specifies the endpoint path, auth header, and response shape following Polygon.io's v2 snapshot API on the assumption that Massive proxies it. This was **not** verified against Massive's documentation. The document carries a visible warning above the section, and the risk is confined to `MassiveSource` — the default simulator path is unaffected — but the market-data agent will otherwise implement against invented shapes.
*Recommendation:* verify against live documentation before that agent starts; treat every field name in that section as provisional until then.

**R7 — `PLAN.de.md` is now ~170 lines out of date, and does not say so.** Per the decision to treat English as authoritative, only `PLAN.md` was revised. `PLAN.md` carries a header noting the translation is stale, but the German file itself carries no such marker, so a reader who opens it directly gets the pre-revision spec with no warning — including the volume contradiction, the wrong skill name, and the unreachable chat-error promise.
*Recommendation:* add a one-line header to `PLAN.de.md` (`> Veraltet: Stand 6b568a9. Maßgeblich ist PLAN.md.`), or re-translate. The warning belongs on the stale document, not only on the current one.

## 16. Verified Sound

- `.env.example` is **not** caught by `.gitignore` — the pattern at line 138 is a literal `.env`, which does not match `.env.example`. It will commit normally
- `users_profile.created_at` is fully removed (S6) with no dangling references
- Watchlist cap (50), unknown-symbol beta (1.0), and the `?limit=` default of 500 agree across all three documents
- The `actions` failure example in `API_CONTRACT.md` deliberately shows prose contradicting the outcome, which is the exact case the chips exist to cover — the contract demonstrates its own rationale

## 17. Assessment

The revision resolves all 30 questions, all 7 simplifications, and D2, and the three documents are mutually consistent on every quantitative value checked. **R1 was the one blocking item** — a genuine deadlock introduced while writing the new trading rules, which would have surfaced the first time anyone typed an unwatched ticker into the trade bar. **It is now fixed** across all three documents (see the resolution note under R1).

All remaining items have since been resolved:

| # | Item | Resolution |
|---|---|---|
| **R6** | Massive API contract unverified | **Partly verified.** Massive is Polygon.io rebranded (30 Oct 2025); endpoints, SDKs and keys carry over unchanged, which validates using Polygon's v2 snapshot shapes. Base URL corrected `api.massive.dev` → **`api.massive.com`** (the original was simply wrong), with `api.polygon.io` documented as the still-supported legacy base. Auth confirmed as Bearer header or `?apiKey=`. **Response field names remain unverified** — the warning is narrowed to that, and Massive is sequenced last in Track A (§13) so nothing depends on it |
| **R2** | Ring buffer as a count, not a duration | Respecified as **60 minutes** rather than 500 ticks, so the chart span no longer depends on the data source (~4 min vs ~2 h). `GET /api/history/{ticker}` downsamples to ≤500 points on read, bounding the response either way |
| **R7** | `PLAN.de.md` stale with no marker | Prominent ⚠️ header added **to the German file itself**, naming the specific defects it still contains and stating it must not be used as a build reference |
| **R3** | `HEALTHCHECK` referenced, never specified | `HEALTHCHECK` instruction added to the §11 Dockerfile stage with interval, timeout, start period and retries |
| **R4** | `change_pct` computed in two places | Dropped from the `GET /api/watchlist` response; the client now computes it from `price` and `session_ref` in both the initial paint and the live path, so there is one derivation |
| **R5** | Three history limits (20/50/500) | Callout added under the §8 endpoint tables stating all three are intentional and must not be unified |

**Also added: §13 Build Order** — three stages with four parallel Stage 2 tracks, plus working agreements (contracts are frozen; no migrating off Next.js/Recharts/SSE; progress recorded in `PROGRESS.md`). This is what the specification was missing to be actionable rather than merely correct.

The specification is ready to build against.
