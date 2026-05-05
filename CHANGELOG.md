# Cable Scalp — Changelog

## v2.10.0 — 2026-05-05 — Fix startup reconcile (OANDA /transactions pagination bug)

### Bug fix — `startup_oanda_reconcile` always returned empty (`reconcile_state.py`)

**Root cause — OANDA /transactions endpoint returns a pagination envelope, not data:**

`get_today_closed_transactions()` called:
```
GET /v3/accounts/{id}/transactions?from=...&to=...&type=ORDER_FILL
```

OANDA's response for this endpoint is a **pagination envelope**:
```json
{"count": 2, "pages": ["https://api.oanda.com/.../transactions/idrange?from=915&to=926"], "lastTransactionID": "956"}
```

The actual transactions are behind the `pages` URLs — they are NOT in the root
response. The code called `r.json().get("transactions", [])` which always returned
`[]` because the `"transactions"` key does not exist in this response format.

As a result, `startup_oanda_reconcile` has logged "no closing transactions found"
on every startup since it was first written, regardless of how many trades closed
that day. This was the root cause behind:
- The loss cap not seeing today's losses correctly after restarts
- Trade P&L not being backfilled on startup (instead relying entirely on
  `reconcile_runtime_state` which runs on the first live cycle)

**Fix:**
Replaced `get_today_closed_transactions()` in `startup_oanda_reconcile` with
`get_recent_closed_trades(instrument, count=50)` — the same endpoint used by
`reconcile_runtime_state()` which is proven to work correctly. The returned trades
are filtered to `closeTime` matching today (SGT date) before being processed.

The broken `get_today_closed_transactions()` method is retained in `oanda_trader.py`
with an updated docstring documenting the pagination issue, in case it's needed
for other purposes in future.

---

## v2.9.0 — 2026-05-05 — Fix reconcile future to_utc + force-close actual P&L recovery

### Bug fix 1 — `get_today_closed_transactions` future `to` datetime (`oanda_trader.py`)

At any startup before 00:00 UTC (i.e., before midnight UTC = 08:00 SGT), the SGT
calendar day's end converts to a UTC time that is still in the future. For example,
at 17:58 SGT May 5 (= 09:58 UTC), the window `to` = `2026-05-05T16:00:00.000000Z`
which is 6 hours ahead. OANDA returns an empty transaction list when the `to`
parameter is in the future, causing `startup_oanda_reconcile` to always report
"no closing transactions found" for any startup before 08:00 SGT.

**Fix:** `to_utc` is now capped at `min(day_end_utc, now_utc + 5 minutes)` so the
query window never extends into the future while still covering the full trading day
for late-day startups.

### Bug fix 2 — Force-close fallback records actual P&L (`bot.py`)

When `close_trade()` returns `False` (trade already closed on broker), the fallback
calls `get_trade_pnl()` which uses `GET /v3/accounts/{id}/trades/{tradeID}`. This
endpoint **only returns OPEN trades** on OANDA — it returns HTTP 404 for closed trades,
causing `get_trade_pnl()` to always return `None` for already-closed positions. The
result: every stale trade that was closed by SL/TP gets recorded as `$0 P&L` instead
of the actual loss/gain, breaking the daily loss counter.

**Fix:** The fallback now first calls `get_recent_closed_trades(instrument, count=50)`
(uses `GET /v3/accounts/{id}/trades?state=CLOSED` which reliably returns closed trade
data) and matches by trade ID to get the real `realizedPL`. Only if that also fails
does it call `get_trade_pnl()`, and only as a last resort falls back to `$0`.

**Impact (v2.8, May 5 2026):** Trades #916 (SL −$31.93) and #923 (SL −$31.77) were
both recorded as `$0 P&L` on the first v2.8 cycle. The daily loss counter for May 5
did not reflect the actual −$64 loss. Deploy v2.9 to prevent this from recurring.

---

## v2.8.0 — 2026-05-05 — Fix dead zone management bypass

### Bug fix — Open trades not managed during dead zone 04:00–07:59 SGT (`bot.py`)

**Root cause:**
`_guard_phase()` has two separate exit points that both fire when `session is None`
(which includes the dead zone):

1. **Line ~1053** — Dead zone early-exit: correctly checks for open trades. If open
   trades exist in local history, it logs "management mode only" and falls through.
   If no open trades, returns `None` immediately. ✅ Correct intent.

2. **Line ~1112** — Session-only check: when `session is None`, returns `None`
   **unconditionally** — even when line 1053 already decided to fall through for
   management. ❌ This is the bug.

The reconcile, `backfill_pnl`, and `force_close_stale_trades` calls are all below
line 1112, so they **never ran** during the dead zone — regardless of whether open
trades needed managing.

**Impact in production (v2.5–v2.7, May 5 2026):**
- Trade #916 (SELL, opened 02:27 SGT) should have been force-closed by the session-end
  logic at 04:00 SGT when the US Cont. session ended. The dead zone management bypass
  meant force_close_stale_trades never ran 04:00–07:59 SGT.
- When trade #916's SL was hit at 05:05 SGT (dead zone), `backfill_pnl` and
  `reconcile_runtime_state` also never ran — so the SL closure was not detected until
  08:00 SGT (first Tokyo cycle), 55 minutes later.
- The daily report at 07:50 SGT (inside dead zone) correctly showed "Open now: 1
  position(s)" because the dead zone management bypass prevented the SL detection from
  being persisted.

**Additional fix — `realized_pnl_usd` falsy check (`bot.py`):**
Two dead zone open-trade checks used `not t.get("realized_pnl_usd")` (falsy),
which treats `0.0` (break-even / not-found fallback) the same as `None` (truly open).
A trade marked closed at `$0` would appear as still open to both dead zone checks,
causing management mode to loop every 3 minutes indefinitely with no harm but wasted
API calls. Both occurrences changed to `t.get("realized_pnl_usd") is None` —
consistent with how `force_close_stale_trades` and `backfill_pnl` already check it.

**Fix:**
- In the `session is None` block (session-only check), added a `_dz_open` check: if
  dead zone AND local history has open trades, do NOT return `None` — fall through to
  the management code (OANDA login → reconcile → backfill → force-close).
- Added a dead zone exit guard immediately before `_guard_phase`'s final `return` dict.
  When in dead zone management mode, this guard returns `None` after management is
  complete, preventing `_signal_phase` and `_execution_phase` from running (no new
  trade entries during dead zone).

---

## v2.7.0 — 2026-05-05 — Fix reconcile datetime format + force-close infinite retry loop

### Bug fix 1 — `get_today_closed_transactions` nanosecond datetime format (`oanda_trader.py`)

The OANDA v20 transactions API expects RFC 3339 datetimes with **microsecond** precision
(`2026-05-04T16:00:00.000000Z`). The time-window parameters were formatted with
**nanosecond** precision (`2026-05-04T16:00:00.000000000Z` — 9 decimal places).

OANDA silently returned an empty transaction list rather than erroring, causing
`startup_oanda_reconcile()` to log "no closing transactions found" and leave the local
trade history stale — even when SL/TP closures had occurred while the bot was running.

**Fix:** Changed both `from_utc` and `to_utc` strftime format strings in
`get_today_closed_transactions()` from `.000000000Z` to `.000000Z`.

### Bug fix 2 — `force_close_stale_trades` infinite retry on already-closed trades (`bot.py`)

When `close_trade()` returned `False` (e.g. HTTP 404 `TRADE_DOESNT_EXIST` because a
trade had already been closed by SL/TP), the bot logged "will retry next cycle" and
did exactly that — every 3 minutes, indefinitely. This generated a flood of
`MARKET_ORDER_REJECT` entries on OANDA (#928–935 visible in the transaction CSV) and
persisted across restarts because the local history was never updated.

**Fix:** After a failed `close_trade()` call, the bot now calls `get_trade_pnl()` to
check the broker's view of the trade:

- **Trade is CLOSED on broker** (SL/TP/manual close detected): backfills the real P&L,
  marks the local record as `BROKER_CLOSED`, fires a Telegram info alert, and stops
  retrying.
- **Trade not found on broker at all**: marks it closed with $0 P&L and logs a warning,
  preventing the infinite retry loop regardless.

**Impact in production (v2.6, May 5 2026):** Trades #916 and #923 both hit SL while
the force-close mechanism was broken (v2.5 bug). After the v2.6 deploy, the bot
correctly called `close_trade()` but received 404s and kept retrying, producing 8
`MARKET_ORDER_REJECT` entries on OANDA between 17:23–17:29 SGT.

---

## v2.6.0 — 2026-05-05 — Fix force_close_stale_trades (missing close_trade method)

### Bug fix — `OandaTrader.close_trade()` missing (`oanda_trader.py`)

`force_close_stale_trades()` in `bot.py` calls `trader.close_trade(trade_id)` to
force-close individual stale trades by their OANDA trade ID. This method did not
exist in `OandaTrader` — only `close_position(instrument)` was implemented — causing
an `AttributeError` to be raised and caught silently every 3 minutes from the moment
any trade became eligible for force-closure.

**Impact in production (v2.5, May 4–5 2026):**
- Trade #916 (US Cont. SELL, opened 02:27 SGT May 5) was not closed at the US Cont.
  session end (04:00 SGT) as intended, and continued to run unmanaged.
- Trade #923 (Tokyo SELL 6/6, opened 08:30 SGT May 5) was subsequently opened on top
  of #916, resulting in two concurrent SELL positions despite the max-concurrent-trades
  limit of 1. The duplicate was not detected because the force-close error masked the
  open-trade state.
- The error fired 280+ times across the two affected log files, escalating to
  twice per cycle once both trades were active (one error per stale trade).

**Fix:** Added `OandaTrader.close_trade(trade_id: str) -> bool` in `oanda_trader.py`.
The method calls `PUT /v3/accounts/{account_id}/trades/{trade_id}/close` and returns
`True` on HTTP 200, `False` otherwise — matching the return-value contract that
`force_close_stale_trades()` already expected.

No changes to `bot.py` or any other file were required.

---

## v2.5.0 — 2026-05-04 — Schedule refinement, weekly CSV export, import fix

### Bug fix — `_clean_session` not imported in `bot.py`

Every trade cycle crashed with `NameError: name '_clean_session' is not defined`
the moment Tokyo session opened on Monday. The helper existed in
`telegram_templates.py` (introduced in v2.1) but was never added to the
`from telegram_templates import (...)` block in `bot.py`. Sunday masked the
bug because the Sunday guard exits before the affected line is reached.

**Fix:** `_clean_session` added to the import block in `bot.py`.

### Report schedule changes

| Report | Before | After |
|---|---|---|
| Daily summary | 04:00 SGT | **07:50 SGT** |
| Weekly report | 08:15 SGT | **08:00 SGT** |
| Weekly trade export | 08:20 SGT | **08:05 SGT** |
| Monthly report | 08:00 SGT | **08:10 SGT** |

Daily report moved to 07:50 SGT — after US Continuation closes at 04:00,
capturing the full overnight session before the dead zone begins.

### Weekly export changed from JSON → CSV

`send_weekly_export` now sends `cable_scalp_trades_to_YYYY-MM-DD.csv` instead
of the raw `trade_history.json`. CSV columns are identical to the monthly
export: `date_sgt`, `time_sgt`, `session`, `direction`, `score`, `result`,
`pl_usd`, `balance`, `h1_trend`, `h1_aligned`, `ema_pts`, `orb_pts`,
`cpr_pts`, `duration_min`, `spread_pips`, `units`, `position_usd`.

Both weekly and monthly exports now open directly in Excel with consistent
column layout. The JSON file is no longer sent via Telegram.

### Dead code removed

`msg_session_cap()` in `telegram_templates.py` was never imported or called
anywhere in the codebase. Removed.

### Stale references cleaned up

All timing references in `reporting.py`, `scheduler.py`, `README.md`,
`CONFLUENCE_READY.md`, and `SETTINGS.md` updated to reflect the new schedule.
`_V16_START` / `v16` / `v2.3` labels in `reporting.py` updated to version-neutral
names (`_DEPLOY_START`, `cable_scalp_trades_to_`).

### Files changed

`bot.py`, `reporting.py`, `scheduler.py`, `telegram_templates.py`,
`settings.json`, `version.py`, `README.md`, `SETTINGS.md`,
`CONFLUENCE_READY.md`, `CHANGELOG.md`

---

## v2.4.0 — GBP/USD news filter and reduced risk profile

- Version updated to **Cable Scalp v2.4**.
- News filter now treats both **GBP** and **USD** events as relevant for GBP/USD.
- Reduced demo/live risk sizing to the safer profile:
  - Score 4 → **$45** partial risk setting
  - Score 5–6 → **$60** full risk setting
- Kept US and US_Cont separately identifiable across logic, logs, reports, and Telegram.
- Kept Telegram session wording as **max** instead of **cap**.


---

## v2.1.0 — 2026-04-30

### Telegram template fixes

**Problem 1 — GBP_USD showing instead of GBP/USD**
`msg_trade_opened`, `msg_order_failed`, `msg_margin_adjustment` were displaying
the raw OANDA instrument code with underscore. Fixed with `_clean_pair()` helper.

**Problem 2 — "US session" showing instead of "US Cont."**
US Continuation trades (00:00–03:59 SGT) were labelled "US session" in trade
open/close messages — same label as the disabled US session. Fixed with
`_clean_session()` helper that maps "US_Cont" → "US Cont." and adds 🌙 icon.

**Problem 3 — US Cont. startup card icon**
Startup card was showing 🗽 for US Cont. Changed to 🌙 to match the new
US_Cont session label separation introduced in v2.0.

**New helpers added to telegram_templates.py:**
- `_clean_pair(s)` — converts GBP_USD → GBP/USD
- `_clean_session(s)` — normalises session display names

### Position sizing increase

Based on $3,000 live balance and 2/2 TP hit rate under v2.0:

| Score | Before | After | Risk % |
|---|---|---|---|
| Score 4 | $60 | **$90** | 3.0% of $3k |
| Score 5–6 | $90 | **$120** | 4.0% of $3k |

### Files changed
`telegram_templates.py`, `bot.py`, `settings.json`, `version.py`,
`telegram_alert.py`, `reporting.py`, `scheduler.py`, `signal_logger.py`,
`signals.py`, `README.md`, `SETTINGS.md`, `CONFLUENCE_READY.md`

---

## v2.0.0 — 2026-04-28

### TP reduced: 30 pips → 25 pips

**Decision basis:** Live data showed avg achieved win of $40–45 on a $60 target,
equivalent to 22–25 pip effective capture. Spread cost (1p) means 30p TP requires
31p raw movement — too far in consolidating conditions, causing overnight holds.

- New TP: **25 pips** (26p effective including spread)
- SL unchanged: 18 pips
- New RR: 1.39× | Break-even WR: 41.9%
- Expected benefit: trades resolve within London session, less overnight exposure

### Force close guards (stale trade prevention)

**Problem:** Apr 27 trade opened at 16:48 SGT, held 8 hours, SL hit at 01:03 SGT.
M5 signal context was stale within 2 hours. Became accidental swing trade.

Two new guards added to `bot.py` via `force_close_stale_trades()`:

**1. Max trade duration** (`max_trade_duration_hours: 4`, default)
Any open trade held > 4 hours is force-closed at market. Sends Telegram alert
with P&L and max pips reached.

**2. Session-end force close** (`force_close_at_session_end: true`, default)
Closes trade when its originating session ends:
- London → 21:00 SGT
- Tokyo → 16:00 SGT
- US Cont. → 04:00 SGT

### US / US Continuation session label separation

Previously both US session (21:00–23:59) and US Continuation (00:00–03:59) used
`macro_session: "US"` — impossible to separate in reports or signal log.

v2.0 gives US Continuation its own label:
- US session → `"US"` (disabled, us_session_start_hour: 99)
- US Continuation → `"US_Cont"` (enabled, threshold 4/6, max 10)

New setting: `max_trades_us_cont: 10`
New session threshold key: `session_thresholds.US_Cont: 4`
New SESSION_BANNER: `"US_Cont": "🌙 US Cont."`

### MFE tracking

`track_max_pips()` was already present in v1.9 but not documented.
v2.0 makes it visible — `max_pips_reached` now appears in:
- Trade history JSON
- Monthly CSV export
- Force-close Telegram alert

### min_rr_ratio fix (v2.0.1 hotfix)

After deploy, first signal log showed:
`Signal BLOCKED | score=3/6 blockers=R:R 1.39 < 1:1.6`

Root cause: `min_rr_ratio` was still 1.6 but new 25p/18p RR = 1.39×.
Every trade was being blocked before it could fire.

Fix: `min_rr_ratio` updated from `1.6` → `1.3` (0.09 buffer below actual 1.39×).

### Files changed
`settings.json`, `version.py`, `bot.py`, `telegram_templates.py`,
`telegram_alert.py`, `reporting.py`, `signal_logger.py`, `signals.py`,
`scheduler.py`, `README.md`, `SETTINGS.md`, `CONFLUENCE_READY.md`

---

## v1.9.0 — 2026-04-27

### Signal logging for AI/ML data collection

Added opt-in signal logging that captures every signal evaluation
(score ≥ `signal_log_min_score`, default 3) to `/data/signal_log.csv`.

**Disabled by default** — zero impact unless enabled:
```json
"signal_logging_enabled": true,
"signal_log_min_score":   3
```

**What gets captured per row:**
- Timestamp, session, direction, score
- Feature breakdown: ema_pts, orb_pts, cpr_pts
- Setup name, H1 trend, H1 aligned
- ORB age, CPR width, ATR, spread
- Action: FIRED / WATCHED / BLOCKED_H1 / BLOCKED_SPREAD / BLOCKED_RR / NOISE
- Block reason (when applicable)
- Outcome: TP / SL / BE (back-filled automatically when trade closes)
- Trade ID (links to trade_history.json)

**Monthly export:** Last day of month at 08:35 SGT — 5 min after trade CSV.
Only fires when `signal_logging_enabled = true`.

**Purpose:** Build cumulative AI training dataset. After 6 months (~1,800 rows)
a confidence scoring model can be trained to filter low-quality signals and
improve dynamic position sizing.

**Files changed:** `signal_logger.py` (new), `bot.py`, `reporting.py`,
`scheduler.py`, `settings.json`, `version.py`, `telegram_templates.py`,
`telegram_alert.py`, `README.md`, `SETTINGS.md`, `CONFLUENCE_READY.md`

---

## v1.8.0 — 2026-04-27

### Score 3 Telegram alerts suppressed — reduced message flooding

**Problem:** Tokyo session was generating score 3/6 WATCHING alerts every
5 minutes — same signal repeating with aging ORB (102, 107, 112, 117 min).
These alerts never resulted in trades (Tokyo threshold is 5/6, London/US
cont is 4/6) and flooded the Telegram chat with noise.

**Fix:** `telegram_min_score_alert` raised from `3` → `4`.

**Impact:**
- Score 3 WATCHING alerts → silently suppressed ✅
- Score 4 WATCHING alerts → still shown (London/US Cont. threshold) ✅
- Score 5+ WATCHING alerts → still shown (Tokyo threshold) ✅
- All trade open/close/blocked alerts → unchanged ✅

**Files changed:** `settings.json`, `version.py`, `bot.py`,
`telegram_templates.py`, `telegram_alert.py`, `README.md`,
`SETTINGS.md`, `CONFLUENCE_READY.md`, `reporting.py`

---

## v1.7.0 — 2026-04-27

### Cumulative monthly CSV export via Telegram

Added automated cumulative CSV export sent on the last day of every month
at 08:30 SGT. The CSV covers all trades from v1.6 start date (2026-04-26)
forward and grows each month — April only → April+May → April+May+June etc.

**Files changed:** `reporting.py`, `scheduler.py`, `bot.py`

**CSV columns:**
`date_sgt, time_sgt, day_of_week, session, direction, score, setup,
entry_price, sl_price, tp_price, result, pl_usd, balance,
h1_trend, h1_aligned, ema_pts, orb_pts, cpr_pts,
duration_min, spread_pips, units, position_usd`

**Schedule:** Last day of month · 08:30 SGT
(30 min after weekly trade history export at 08:20 SGT)

**Purpose:** Build cumulative dataset for future AI/ML analysis.
Each monthly CSV is a complete v1.6+ trade log — open directly in
Excel or Google Sheets from Telegram.

---

## v1.6.0 — 2026-04-26

### US session disabled — confirmed by live data

**Decision basis:** 42 live trades (Apr 1–26, 2026) across all three data sources
(OANDA transaction CSV + Railway container logs + Telegram message history).

US session (21:00–23:59 SGT) performance:
- Record: 1W / 9L — **10% win rate**
- Net P&L: **-$185**
- Break-even WR required: 37.5%
- Gap below break-even: **27.5 points**
- Sample: 10 trades — statistically conclusive

`us_session_start_hour` and `us_session_end_hour` set to `99` (sentinel disabled).
`session_thresholds.US` set to `99`. `max_trades_us` set to `0`.
US Continuation (00:00–03:59 SGT) remains **enabled** (78% WR, 7W/2L, +$229).

### H1 filter — upgraded from soft to strict

**Decision basis:** Log and Telegram analysis confirmed 73% of recent trades
were counter-trend (11/15 over Apr 20–24). Counter-trend trades caused the
Apr 22–24 losing streak (-$344 in 3 days):

| Date | Trade | H1 | Result |
|---|---|---|---|
| Apr 20 20:23 | SELL London | BULLISH ⚠️ | SL -$35 |
| Apr 21 20:58 | SELL London | BULLISH ⚠️ | SL -$36 |
| Apr 22 09:23 | SELL Tokyo | BULLISH ⚠️ | SL -$35 |
| Apr 22 18:08 | SELL London | BULLISH ⚠️ | SL -$34 |
| Apr 23 16:43 | BUY London | BEARISH ⚠️ | SL -$34 |
| Apr 23 19:58 | BUY London | BEARISH ⚠️ | SL -$36 |
| Apr 24 20:58 | SELL London | BULLISH ⚠️ | SL -$33 |

`h1_filter_mode` changed from `soft` → `strict`.
In strict mode counter-trend entries are **blocked** before execution — no alert,
no order. Estimated savings: ~$243 over the Apr 22–24 period alone.

### Active sessions after v1.6

| Window | SGT | Threshold |
|---|---|---|
| Dead zone | 04:00–07:59 | No trading |
| Tokyo | 08:00–15:59 | ≥ 5/6 |
| London | 16:00–20:59 | ≥ 4/6 |
| US session | 21:00–23:59 | **Disabled** |
| US Continuation | 00:00–03:59 | ≥ 4/6 |

### Files changed
`settings.json`, `version.py`, `README.md`, `SETTINGS.md`,
`CONFLUENCE_READY.md`, `CHANGELOG.md`

---

## v1.0.0 — 2026-04-12

Initial release of **Cable Scalp v1.0** — dedicated GBP/USD (Cable) M5 scalping bot.

### Instrument
GBP/USD only. Single pair, clean data, focused execution.

### Strategy
EMA 9/21 crossover + Opening Range Breakout (ORB, time-decayed) + CPR daily pivot bias.
Score 0–6/6. Threshold: 4/6 for London and US Cont, 5/6 for Tokyo.

### Active sessions

| Window | SGT | Threshold |
|---|---|---|
| Dead zone | 04:00–07:59 | No trading |
| Tokyo | 08:00–15:59 | ≥ 5/6 |
| London | 16:00–20:59 | ≥ 4/6 |
| US session | 21:00–23:59 | **Disabled** |
| US Continuation | 00:00–03:59 | ≥ 4/6 |

**US session 21–23 disabled:** 0% WR in live testing (5 consecutive losses).
**US Continuation 00–03 active:** 100% WR in live testing (5/5 TPs, +$190.66).

### Position sizing

| Score | Position | Risk |
|---|---|---|
| 4 | $30 partial | 1.5% of $2k |
| 5–6 | $48 full | 2.4% of $2k (margin ceiling at $2k account) |

### SL / TP

| Pair | SL | TP | RR | Break-even WR |
|---|---|---|---|---|
| GBP/USD | 18p | 30p | 1.67× | 37.5% |

### Risk guards (13 ordered checks)

1. Market closed (Sat/Sun/Mon pre-08:00)
2. Dead zone early exit (04:00–07:59, no open trades)
3. News hard block (±30min high-impact events)
4. News penalty (medium events → −1 score)
5. Loss cooldown (consecutive losses → 30min pause)
6. Friday cutoff (after 23:00 SGT)
7. Session check (outside active windows)
8. Daily loss max (8 losses → pause until 08:00)
9. Session max (per-window trade limit)
10. Concurrent max (max 2 open across all pairs)
11. Margin guard (auto-reduce units if margin insufficient)
12. Min units (reject if <1,000 units after margin guard)
13. Spread guard (skip if spread > session limit)

### H1 trend filter
Enabled in **soft mode** — labels each trade as H1 aligned or counter-trend.
Flip to `"h1_filter_mode": "strict"` to block counter-trend entries once data confirms.

### Breakeven
Disabled (`breakeven_enabled: false`). Enable by setting to `true` once
`max_pips_reached` data confirms winners consistently travel past 20p.

---

## v1.1.0 — 2026-04-12

### US session re-enabled
US session (21:00–23:59 SGT) re-enabled for data collection.
Previous disable was based on 5 trades — insufficient sample for a structural decision.
Score threshold: ≥ 4/6 (same as London and US Continuation).
Requires 30–50 trades per window before any session-level conclusions.

### Enhanced per-session Telegram reporting
- **Daily summary:** Session breakdown now shows `W/L split + win rate + net PnL`
  per session (was trade count + PnL + icon only).
- **Weekly report:** "By Session" rows now include explicit `W/L counts` alongside
  the win rate bar and PnL.
- **Monthly report:** Same W/L count addition to "By Session" rows.

### Active sessions after v1.1

| Window | SGT | Threshold |
|---|---|---|
| Dead zone | 04:00–07:59 | No trading |
| Tokyo | 08:00–15:59 | ≥ 5/6 |
| London | 16:00–20:59 | ≥ 4/6 |
| US session | 21:00–23:59 | ≥ 4/6 ← re-enabled |
| US Continuation | 00:00–03:59 | ≥ 4/6 |

---

## v1.2.0 — 2026-04-13

### Investigation findings (no code defects found)

Full review of container logs and OANDA transaction history confirmed:

**Dead zone concern — resolved as false alarm.**
Telegram card showed "06:09" timestamp — this was the user's phone displaying UTC
time, not SGT. OANDA CSV confirms trade entry at `2026-04-13 08:39:52 +08 SGT`,
correctly inside Tokyo session (08:00–15:59 SGT). Dead zone (04:00–07:59 SGT)
had already ended 39 minutes prior. No dead zone entry occurred.

**Trade performance (first live trade):**
- Entry: BUY 17,911 units @ 1.33991 SGT 08:39
- Close: TP hit @ 1.34292 SGT 15:51 (+30.1p, +$53.91)
- Duration: 7h 11m — slow Tokyo drift, won correctly
- Balance: $2,000.01 → $2,053.92

**Margin guard:** Correctly adjusted 26,667 → 17,911 units due to $2k account
margin ceiling with 0.60 safety factor. Self-corrects as balance grows.

**Calendar 429 rate limiting:** Handled correctly with 15-min backoff.
Zero trading impact.

### Changes

**Defense in depth — hard execution-phase dead zone block:**
Added explicit `is_dead_zone_time()` check at the top of `_execution_phase()`
as a final hard stop before any OANDA order call. If somehow the guard chain
is bypassed, this block fires with a WARNING log and suppresses the order.
Under normal operation this block never triggers — it exists purely as a
safety net.

**Dead zone fallthrough logging upgraded:**
When the bot falls through the dead zone check due to open trades (management
mode), log level raised from DEBUG → INFO so it's visible in Railway logs.
Message: "Dead zone — N open trade(s) present, management mode only. No new entries."

**Startup Telegram card — timezone clarity:**
Sessions header updated from "Sessions (SGT)" to "Sessions (SGT = UTC+8)" to
eliminate confusion when viewing Telegram from phones in non-SGT timezones.

---

## v1.3.0 — 2026-04-14

### Bug fix — Daily report day total inconsistent with session breakdown

**Problem:**
Session breakdown and day total showed different counts for the same day.
Example from Apr 13:
  Session: Tokyo 1W/0L + London 0W/2L + US 1W/0L = **2W/2L (4 trades)**
  Day total: **1W/2L (3 trades)** ← wrong

**Root cause:**
Two different time windows were used in the same report:
- Session breakdown used `pd_trades` = full day 00:00–24:00 SGT
- Day total used `today_trades` starting at 16:00 SGT (London open)
- Any Tokyo trade (08:00–15:59 SGT) appeared in session breakdown
  but not in day total — missed because it fell before 16:00.

**Fix:**
Day total now uses `pd_trades` — the same full-day window as the session
breakdown. `today_start` / `today_trades` variables removed.
Both sections always consistent.

**File changed:** `reporting.py`

---

## v1.4.0 — 2026-04-14

Parity release — brings Cable Scalp in line with Ninja Scalp v1.1 architecture.

### Fix 1 — Railway healthcheck 503 during warmup
Health handler returned `503` when scheduler not yet running. Matches the fix
applied to Ninja Scalp v1.0.2. Now always returns `200` while process is alive.
Status field: `"starting"` during warmup, `"ok"` once scheduler running.

### Fix 2 — Health server at entry point
`_start_health_server()` moved to `__main__` before `main()`. Matches Ninja
Scalp v1.0.1 fix. Health server binds immediately on process start.

### Fix 3 — Session sentinel guard (`_build_sessions`)
`_build_sessions()` now excludes US windows when `us_session_start_hour >= 99`.
Prevents "US session" session label appearing when US is disabled in future.
Matches Ninja Scalp v1.1 fix.

### Fix 4 — ORB sentinel guards (`signals.py`)
`_build_orb_sessions()` and `_get_active_session()` now guard against the `99`
disabled sentinel. Prevents `ValueError: hour must be in 0..23` if US session
is ever set to disabled. Matches Ninja Scalp v1.0.3 fix.

### Fix 5 — Legacy code removed
- `startup_checks.py`: removed `sl_pct` / `rr_ratio` pair-level checks
  (not applicable to fixed-pip `pair_sl_tp` architecture)
- `bot.py`: `"RF Scalp"` fallback string → `"Cable Scalp"`

### Fix 6 — Telegram US Continuation disabled label
Startup card now shows `🚫 US Cont.    disabled` when
`us_session_early_end_hour >= 99`, consistent with Ninja Scalp v1.1.

---

## v1.5.0 — 2026-04-16

### Position sizing — $2.00/$1.50 per pip target

| | v1.4 | v1.5 | $/pip |
|---|---|---|---|
| Full (score 5–6) | $48 → 26,667 units | **$60 → 20,000 units** | **$2.00/pip** |
| Partial (score 4) | $30 → 16,667 units | **$45 → 15,000 units** | **$1.50/pip** |

Sizing aligned with Fiber Scalp v1.5 — consistent pip value across the Cable/Fiber fleet.

### Fix — max_total_open_trades corrected to 1

Cable Scalp is single-pair (GBP/USD) with max_concurrent_trades: 1.
Global max was 2 — misleading since per-pair max of 1 always bound first.
Fixed: `max_total_open_trades: 2 → 1`. Startup card now shows "Global max: 1 open trade".

### H1 filter split in weekly and monthly reports

Aligned vs counter-trend WR now appears automatically every Monday in the
weekly report and monthly report. No manual JSON analysis needed.

```
H1 Filter [soft]
  Aligned    ██████████  72.0%  9W/3L  $+156.00
  Counter ⚠️  ████░░░░░░  33.3%  1W/2L  $-42.00
  → Counter-trend 38.7pts lower — consider strict mode
```

Recommendation logic:
- < 5 counter trades → need more data
- Diff ≥ 20pts → consider strict mode
- Diff ≥ 10pts → monitor closely
- Diff < 10pts → soft mode justified

**Files changed:** `settings.json`, `bot.py`, `config_loader.py`, `scheduler.py`,
`startup_checks.py`, `telegram_templates.py`, `reporting.py`, all docs.
