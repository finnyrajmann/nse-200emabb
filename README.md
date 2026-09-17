# NSE 200EMABB

Automated NSE swing-trading paper system, deployed as a DigitalOcean Function.

## Design summary

- **Watchlist**: pre-vetted, uptrend-only, maintained by a separate periodic
  (monthly) screener — EMA200 3-checkpoint slope check
  (EMA200_today > EMA200_50d_back > EMA200_100d_back) + price above EMA200.
  This system trusts the watchlist and does not re-verify EMA200 itself.
- **Entry**: 9 EMA currently above 30 EMA (no fresh-cross requirement).
- **Target exit**: price touches/exceeds Bollinger Band upper (20, 2 std).
- **Stop exit**: 9 EMA currently below 30 EMA, OR price <= 10% below the
  highest daily high since entry (trailing stop). If both trigger the same
  day, logged as a distinct `STOP_BOTH` exit reason.
- **Hit/miss split**: exits are written to one of two trade logs instead of
  a single combined log — PnL% > 3.0 -> hit, PnL% <= 3.0 -> miss (this also
  covers flat and losing trades).
- **Position sizing**: Rs.10,000 per trade, same as BB Trader.

## Files

```
data/
  positions_200emabb.csv         # open paper positions
  trade_log_hit_200emabb.csv     # closed trades, PnL% > 3.0
  trade_log_miss_200emabb.csv    # closed trades, PnL% <= 3.0
  watchlist_200emabb.csv         # maintained by the separate screener
  entry_snapshot_200emabb.csv    # indicator values at moment of entry —
                                  # independent of positions.csv, never
                                  # trimmed on exit (permanent record)
functions/
  project.yml                  # LOCAL ONLY — not checked in, add before DO deploy
  packages/nse_200emabb/daily_run/__main__.py
```

## Environment variables (set in `functions/project.yml`, not committed)

- `GITHUB_PAT`, `GITHUB_REPO`
- `GMAIL_SENDER`, `GMAIL_APP_PASSWORD`, `GMAIL_RECIPIENT`

## Known Gaps / Improvements Backlog

- [ ] BB-upper target exit doesn't check EMA trend strength — can chop a
      sustained uptrend into repeated small trades (exit at target,
      re-enter next day since 9EMA still > 30EMA). Preferred fix: BB-upper
      becomes an alert-only signal, real exit gated on EMA weakening or
      another indicator — pending more chart review. Left as-is for now;
      watch the trade log for evidence this is actually happening in
      practice before prioritizing a fix.
- [ ] Whether to reintroduce a BB(50) condition into entry logic — dropped
      from entry for feeling like it fought the EMA200 uptrend requirement,
      but not a final decision. Revisit after more chart review.

## Status

Deployed live on DO (`trading` context account), cron `30 03 * * 1-5`.
Watchlist populated via the separate screener (133 stocks as of first run).
