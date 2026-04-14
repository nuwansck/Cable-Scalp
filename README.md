# Cable Scalp v1.3 — GBP/USD M5 Scalping Bot

> **Deployed on Railway · OANDA API · Telegram Alerts**

Cable Scalp v1.3 is a dedicated automated M5 scalping bot for **GBP/USD (Cable)** on OANDA.
Single pair, clean data, focused execution.
Strategy: EMA 9/21 crossover + Opening Range Breakout (ORB) + CPR pivot bias, scored 1–6/6.

---

## Table of Contents

1. [Strategy Overview](#strategy-overview)
2. [Signal Scoring](#signal-scoring)
3. [Trading Sessions](#trading-sessions)
4. [Risk Management](#risk-management)
5. [Settings Reference](#settings-reference)
6. [Railway Deployment](#railway-deployment)
7. [Environment Variables](#environment-variables)
8. [File Structure](#file-structure)
9. [Telegram Alerts](#telegram-alerts)

---

## Strategy Overview

Cable Scalp v1.3 runs on **M5 (5-minute) candles** with a 5-minute cycle.
Every cycle the signal engine evaluates three components and scores them 0–6:

| Component | Points | Condition |
|---|---|---|
| EMA crossover | +3 (fresh cross) / +1 (aligned) | EMA9 vs EMA21 on M5 |
| ORB breakout | +2 (fresh <60min) / +1 (aging 60–120min) | Price beyond session open range |
| CPR bias | +1 | Price above/below daily pivot |

**Score ≥ 4 → trade eligible** (London / US cont). Score ≥ 5 required for Tokyo.
Score 5–6 → full position ($30). Score 4 → partial position ($20).

---

## Signal Scoring

```
Max score: 6/6
Threshold: 4/6 (London, US cont)  |  5/6 (Tokyo)

Score 1–2:  WATCHING — alert suppressed (noise)
Score 3:    WATCHING — alert sent (one below threshold)
Score 4:    TRADE — partial $20
Score 5–6:  TRADE — full $30
```

---

## Trading Sessions

All times SGT (UTC+8):

```
✈️  04:00–07:59  Dead zone       No new entries (pre-Tokyo gap)
🗼 08:00–15:59  Tokyo           score ≥ 5/6  cap 10
🇬🇧 16:00–20:59  London          score ≥ 4/6  cap 10
🗽 21:00–23:59  US session      score ≥ 4/6  cap 10
🗽 00:00–03:59  US continuation score ≥ 4/6  cap 10
```

Day reset: 08:00 SGT. Global cap: 2 open trades simultaneously.
Market closed: Saturday and Sunday.

---

## Risk Management

| Setting | Value | Notes |
|---|---|---|
| `position_full_usd` | $48 | 2.4% of $2,000 account (max margin) |
| `position_partial_usd` | $30 | 1.5% of $2,000 account |
| `max_total_open_trades` | 2 | Hard cap across all pairs |
| `max_losing_trades_day` | 8 | Bot pauses until 08:00 SGT |
| `min_trade_units` | 1,000 | Reject margin-adjusted micro-orders |

SL/TP (GBP/USD):

| Pair | SL | TP | RR | Break-even WR |
|---|---|---|---|---|
| GBP/USD | 18p | 30p | 1.67× | 37.5% |

---

## Settings Reference

See `SETTINGS.md` for the full key reference.

Key settings in `settings.json`:
```json
{
  "bot_name":              "Cable Scalp v1.3",
  "position_full_usd":     48,
  "position_partial_usd":  30,
  "max_total_open_trades": 2,
  "max_concurrent_trades": 1,
  "min_trade_units":       1000,
  "signal_threshold":      4,
  "telegram_min_score_alert": 3,
  "daily_report_hour_sgt": 4,
  "be_trigger_pips":       20,
  "pair_sl_tp": {
    "GBP_USD": {"sl_pips": 18, "tp_pips": 30, "pip_value_usd": 10.0, "be_trigger_pips": 20}
  }
}
```

---

## Railway Deployment

1. Push the `Cable Scalp v1.3` folder to a GitHub repository
2. Connect to Railway → New Project → Deploy from GitHub
3. Set environment variables (see below)
4. Add a persistent volume mounted at `/data`
5. Railway will build and deploy automatically

---

## Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `OANDA_API_KEY` | ✅ | Practice or live API key |
| `OANDA_ACCOUNT_ID` | ✅ | e.g. `101-003-XXXXXXX-001` |
| `OANDA_DEMO` | ✅ | `true` for practice, `false` for live |
| `TELEGRAM_BOT_TOKEN` | ✅ | From @BotFather |
| `TELEGRAM_CHAT_ID` | ✅ | Your chat/channel ID |

---

## File Structure

```
Cable Scalp v1.3/
├── scheduler.py          # APScheduler — main entry point
├── bot.py                # Trade cycle logic
├── signals.py            # EMA + ORB + CPR signal engine
├── oanda_trader.py       # OANDA REST API wrapper
├── telegram_templates.py # All Telegram message cards
├── telegram_alert.py     # Telegram sender
├── reporting.py          # Daily / weekly / monthly reports
├── config_loader.py      # Settings loading + defaults
├── database.py           # SQLite cycle + signal logging
├── reconcile_state.py    # Trade state reconciliation
├── news_filter.py        # Forex Factory calendar filter
├── calendar_fetcher.py   # Calendar data fetcher
├── settings.json         # All configuration (source of truth)
├── settings.json.example # Reference copy
├── version.py            # Version string
├── Procfile              # Railway start command
├── requirements.txt      # Python dependencies
└── railway.json          # Railway config
```

---

## Telegram Alerts

| Card | Trigger |
|---|---|
| 🚀 Startup | On deploy |
| 👁 Watching | Score ≥ 3 (score 1–2 suppressed) |
| ❌ Blocked | Signal blocked by guard |
| ✅ Ready | Score hits threshold |
| Trade Opened | Every fill |
| Trade Closed | Every TP/SL/BE close |
| Session Open | Each session window |
| 📊 Daily Summary | 04:00 SGT Mon–Fri |
| 📅 Weekly Report | Monday 08:15 SGT |
| 📆 Monthly Report | First Monday 08:00 SGT |
