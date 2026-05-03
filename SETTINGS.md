# Cable Scalp v2.2 — Settings Reference

All settings live in `settings.json`. The bot syncs this file to the Railway
volume on every startup. Edit on the volume and redeploy to apply changes.

---

## Core

| Key | Default | Description |
|---|---|---|
| `bot_name` | `Cable Scalp v2.2` | Shown in all Telegram alerts and logs. |
| `enabled` | `true` | Master on/off switch. `false` = bot skips all trade cycles but stays running. |
| `demo_mode` | `true` | `true` = OANDA practice account. `false` = live account. |

---

## Signal Engine

| Key | Default | Description |
|---|---|---|
| `signal_threshold` | `4` | Minimum score to place a trade (London / US Cont.). |
| `session_thresholds` | `{"London": 4, "US": 4, "US_Cont": 4, "Tokyo": 5}` | Per-session override. Tokyo is stricter (5/6) — GBP/USD is quieter in Asian hours. |
| `ema_fast_period` | `9` | Fast EMA period (M5). |
| `ema_slow_period` | `21` | Slow EMA period (M5). |
| `orb_fresh_minutes` | `60` | ORB break within this many minutes = +2 pts (fresh). |
| `orb_aging_minutes` | `120` | ORB break beyond fresh but within this = +1 pt (aging). Stale = +0. |
| `orb_formation_minutes` | `15` | Minimum time after session open before ORB is considered formed. |
| `exhaustion_atr_mult` | `3.0` | Price > this × ATR from EMA midpoint = −1 penalty. Skipped during ORB breakout. |
| `m5_candle_count` | `40` | Number of M5 candles fetched per cycle. |
| `atr_period` | `14` | ATR calculation period. |
| `signal_logging_enabled` | `false` | Enable/disable AI signal data collection. `false` = silent. Flip to `true` to start logging all signal evaluations to `/data/signal_log.csv`. |
| `max_trade_duration_hours` | `4` | Force-close any open trade after this many hours. Prevents M5 scalp setups becoming overnight swing trades. v2.0: 4 hours. |
| `force_close_at_session_end` | `true` | Force-close a trade when its originating session ends. London trade closes at 21:00 SGT. Tokyo at 16:00 SGT. US Cont. at 04:00 SGT. |
| `max_trades_us_cont` | `4` | Trade cap for US Continuation session (00:00–03:59 SGT). Separate from US session cap in v2.0. |
| `signal_log_min_score` | `3` | Minimum score to capture in signal log. Score 0-2 is pure noise. Score 3+ captures meaningful setups that approached but didn't reach threshold. |
| `telegram_min_score_alert` | `4` | WATCHING cards below this score are silently suppressed. Set to 4 in v1.8 — score 3 alerts removed (never trade, just noise). London/US Cont. threshold is 4, Tokyo is 5. |

---

## Sessions

| Key | Default | Description |
|---|---|---|
| `tokyo_session_start_hour` | `8` | Tokyo open (SGT). |
| `tokyo_session_end_hour` | `15` | Tokyo close (SGT). |
| `london_session_start_hour` | `16` | London open (SGT). |
| `london_session_end_hour` | `20` | London close (SGT). |
| `us_session_start_hour` | `99` | US session open. `99` = disabled. **Disabled in v1.6** (10% WR live data). |
| `us_session_end_hour` | `99` | US session close. `99` = disabled. **Disabled in v1.6**. |
| `us_session_early_end_hour` | `3` | US Continuation close hour (SGT). `3` = window is 00:00–03:59. Set to `-1` to disable. |
| `dead_zone_start_hour` | `4` | Dead zone start (SGT) — no new entries. |
| `dead_zone_end_hour` | `7` | Dead zone end (SGT). |

---

## Position Sizing

| Key | Default | Description |
|---|---|---|
| `position_full_usd` | `120` | Risk in USD for score 5–6 trades. Updated v2.1: $60 → $120. | Score 5–6 → ~20,000 units → **$2.00/pip** | Dollar risk for score 5–6 (full position). |
| `position_partial_usd` | `90` | Risk in USD for score 4 trades. Updated v2.1: $45 → $90. | Score 4 → ~15,000 units → **$1.50/pip** | Dollar risk for score 4 (partial position). |
| `min_rr_ratio` | `1.3` | Minimum RR ratio required to place a trade. Updated in v2.0 from 1.6 to 1.3 to match the new 25p TP / 18p SL configuration (actual RR = 1.39×). | Minimum RR — trade blocked if computed RR falls below this. |

### `pair_sl_tp` — GBP/USD fixed pip values

```json
"pair_sl_tp": {
  "GBP_USD": {"sl_pips": 18, "tp_pips": 30, "pip_value_usd": 10.0, "be_trigger_pips": 20}
}
```

| Key | Description |
|---|---|
| `sl_pips` | Stop loss in pips. GBP/USD default: 18p. |
| `tp_pips` | Take profit in pips. GBP/USD default: 30p (1.67× RR). |
| `pip_value_usd` | Dollar value of 1 pip per 100k units. GBP/USD = $10.00 (static). |
| `be_trigger_pips` | Break-even trigger. SL moves to entry when price travels this many pips. |

GBP/USD SL/TP summary:

| Pair | SL | TP | RR | Break-even WR |
|---|---|---|---|---|
| GBP/USD | 18p | 30p | 1.67× | 37.5% |

---

## Risk Guards

| Key | Default | Description |
|---|---|---|
| `max_total_open_trades` | `1` | Global hard cap — max open trades across all pairs simultaneously. |
| `max_concurrent_trades` | `1` | Max open trades per pair. |
| `max_losing_trades_day` | `4` | Daily loss cap — bot pauses until 08:00 SGT. |
| `max_trades_day` | `12` | Max trades per trading day. |
| `max_trades_london` | `4` | Max trades per London window. |
| `max_trades_us` | `4` | Max trades per US window. |
| `max_trades_tokyo` | `4` | Max trades per Tokyo window. |
| `max_losing_trades_session` | `2` | Max losses per session window before pausing. |
| `min_trade_units` | `1000` | Reject margin-adjusted orders smaller than this. |
| `loss_streak_cooldown_min` | `30` | Minutes to pause after consecutive losses. |
| `sl_reentry_gap_min` | `5` | Minimum minutes before re-entering after an SL close. |
| `friday_cutoff_hour_sgt` | `23` | No new entries after this hour on Fridays. |

---

## Margin Guard

| Key | Default | Description |
|---|---|---|
| `margin_safety_factor` | `0.6` | Max usable margin = free_margin × this factor. |
| `margin_retry_safety_factor` | `0.4` | Retry factor if first margin calc fails. |
| `margin_rate_override` | `0.0` | Override OANDA's margin rate. `0.0` = use broker rate. |
| `auto_scale_on_margin_reject` | `true` | Auto-retry with reduced units if OANDA rejects for margin. |

---

## Spread Guard

| Key | Default | Description |
|---|---|---|
| `spread_limits` | `{"London": 4, "US": 5, "Tokyo": 4}` | Max spread in pips per session. GBP/USD typical spread: 1–2p London, 2–4p US. |
| `max_spread_pips` | `5` | Global fallback spread limit if session not found. |

---

## News Filter

| Key | Default | Description |
|---|---|---|
| `news_filter_enabled` | `true` | Enable Forex Factory news blocking. |
| `news_block_before_min` | `30` | Block trading this many minutes before high-impact events. |
| `news_block_after_min` | `30` | Block trading this many minutes after high-impact events. |
| `news_lookahead_min` | `120` | Log upcoming events within this window. |
| `news_medium_penalty_score` | `-1` | Score penalty applied for medium-impact events. |

---

## Breakeven

| Key | Default | Description |
|---|---|---|
| `breakeven_enabled` | `false` | Enable break-even SL movement. Enable once `max_pips_reached` data confirms winners travel past 20p. |
| `be_trigger_pips` | `20` | Global fallback trigger (per-pair override in `pair_sl_tp`). |

---

## H1 Filter

| Key | Default | Description |
|---|---|---|
| `h1_filter_enabled` | `true` | Fetch H1 EMA21 and classify trades as aligned/counter-trend. |
| `h1_filter_mode` | `strict` | `soft` = label only. `strict` = block counter-trend entries. **Set to strict in v1.6** — live data showed 73% counter-trend trades losing. |
| `h1_ema_period` | `21` | H1 EMA period. |

---

## Reports

| Key | Default | Description |
|---|---|---|
| `daily_report_hour_sgt` | `4` | Daily report fires at this hour (04:00 SGT = dead zone start). |
| `daily_report_minute_sgt` | `0` | Daily report minute. |
| `weekly_report_hour_sgt` | `8` | Weekly report fires Monday at this hour. |
| `weekly_report_minute_sgt` | `15` | Weekly report minute. |
| `monthly_report_hour_sgt` | `8` | Monthly report fires first Monday at this hour. |
| `trading_day_start_hour_sgt` | `8` | Hour that defines the start of a new trading day. |
