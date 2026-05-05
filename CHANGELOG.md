# Cable Scalp — Changelog

## v2.15.0 — 2026-05-05 — Remove force-close; SL/TP on OANDA handles all trade exits

### What changed

All force-close logic removed. Every trade already has a hard SL and TP set on OANDA
at entry time. The bot-side force-close was a belt-and-suspenders mechanism that added
complexity without adding safety, and was the source of every bug chased from v2.5
through v2.14.

### Files changed

**bot.py**
- Removed force_close_stale_trades() function entirely
- Removed the call site and _sess_end_h variable in the main cycle
- Removed the dead zone management fallthrough (open trades no longer need
  bot-side management during 04:00-07:59 SGT — OANDA SL/TP handles it)
- Removed the dead zone exit guard before _guard_phase return
- Removed max_trade_duration_hours and force_close_at_session_end settings defaults
- Dead zone / outside-session handling restored to clean single early-return logic

**oanda_trader.py**
- Removed close_trade() method
- Removed get_today_closed_transactions() method and all pagination logic
- Cleaned get_recent_closed_trades() docstring

**reconcile_state.py**
- startup_oanda_reconcile() simplified — uses get_recent_closed_trades() with
  a clean date filter; no transactions endpoint, no pagination, no fallbacks

**settings.json / settings.json.example**
- Removed max_trade_duration_hours key
- Removed force_close_at_session_end key

### Trade management going forward

- SL and TP set on OANDA at entry — OANDA closes the trade automatically
- backfill_pnl() detects closure each cycle and updates local history
- reconcile_runtime_state() catches SL/TP closures mid-cycle
- startup_oanda_reconcile() catches closures that happened during a restart gap
- Manual close via OANDA interface anytime if needed
