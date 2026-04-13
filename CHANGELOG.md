# Cable Scalp — Changelog

---

## v1.0.0 — 2026-04-12

Initial release of **Cable Scalp v1.0** — dedicated GBP/USD (Cable) M5 scalping bot.

### Instrument
GBP/USD only. Single pair, clean data, focused execution.

### Strategy
EMA 9/21 crossover + Opening Range Breakout (ORB, time-decayed) + CPR daily pivot bias.
Score 0–6/6. Threshold: 4/6 for London and US cont, 5/6 for Tokyo.

### Active sessions

| Window | SGT | Threshold |
|---|---|---|
| Dead zone | 04:00–07:59 | No trading |
| Tokyo | 08:00–15:59 | ≥ 5/6 |
| London | 16:00–20:59 | ≥ 4/6 |
| US session | 21:00–23:59 | **Disabled** |
| US continuation | 00:00–03:59 | ≥ 4/6 |

**US session 21–23 disabled:** 0% WR in live testing (5 consecutive losses).
**US continuation 00–03 active:** 100% WR in live testing (5/5 TPs, +$190.66).

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
8. Daily loss cap (8 losses → pause until 08:00)
9. Session cap (per-window trade limit)
10. Concurrent cap (max 2 open across all pairs)
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
Score threshold: ≥ 4/6 (same as London and US continuation).
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
| US continuation | 00:00–03:59 | ≥ 4/6 |

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
