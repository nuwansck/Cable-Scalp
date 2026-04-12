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
