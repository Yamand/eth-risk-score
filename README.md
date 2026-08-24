# ETH Risk Score

Daily composite 0–1 risk score for Ethereum (ETHUSDT) from Binance public OHLCV data, with automated Telegram alerts and GitHub Actions CI/CD.

**Identical methodology to the BTC Risk Score** — see [btc-risk-score/README](https://github.com/Yamand/btc-risk-score) for full details on the scoring framework, component weighting, and DCA zone logic.

## Quick start

### Local run
```bash
python eth_risk_score.py            # full history fetch
python eth_risk_score.py --update   # incremental daily update
```

Output: `data/eth_risk_history.json` (scored), `data/eth_prices_raw.json` (price cache)

### GitHub Actions setup

1. **Clone and push to GitHub**  
   The repo includes a GitHub Actions workflow (`.github/workflows/daily-update.yml`) that runs daily at **00:20 UTC** — 5 minutes after the BTC score to avoid Telegram API collisions.

2. **Set secrets in GitHub**  
   Add these as Actions Secrets:
   - `TELEGRAM_BOT_TOKEN` — from @BotFather
   - `TELEGRAM_CHAT_ID` — your user ID (see BTC repo README for how to get it)

3. **Verify the workflow runs**  
   The Actions tab will show the daily job. Alerts fire at 00:20 UTC every day.

## Key differences from BTC

- **Symbol:** ETHUSDT (not BTCUSDT)
- **Schedule:** 00:20 UTC (vs. 00:15 UTC for BTC)
- **Binance listing:** 2017-07-14 (vs. 2017-08-17 for BTC)
- **Genesis date:** 2015-07-30 (vs. 2009-01-03 for BTC) — used for the log-regression "days_since_genesis" anchor
- **Telegram emoji:** Ξ (xi, U+039E) instead of ₿ (bitcoin, U+20BF)
- **DCA zones:** Identical thresholds and multipliers; only the holdings-gate reminder text mentions "ETH"

## Components (35% + 25% + 20% + 20%)

1. **Log-regression band position** (35%)  
   Price vs. log-log growth curve over full history. High = overvalued, low = undervalued.

2. **200-day MA multiple** (25%)  
   Price / 200d MA. Self-calibrating percentile rank.

3. **RSI-14** (20%)  
   14-day relative strength index (daily candles). High = overbought, low = oversold.

4. **Volatility-adjusted momentum** (20%)  
   30-day return / 30-day realized volatility, EMA-smoothed pre-rank to dampen window-edge noise.

All components are normalized to 0–1 via expanding historical percentile rank (min 60 days warmup).

## DCA zones

| Score | Zone | Action | Multiplier | Weekly size |
|-------|------|--------|------------|-------------|
| 0.00–0.10 | Extreme Buy | Max accumulate | 3.0× | $30 |
| 0.10–0.20 | Strong Buy | Accumulate | 1.5× | $15 |
| 0.20–0.25 | Buy | Normal DCA | 1.0× | $10 |
| 0.25–0.35 | Reduced Buy | Slow down | 0.5× | $5 |
| 0.35–0.60 | Hold | Stop accumulation | 0× | — |
| 0.60–0.70 | Sell Tier 1 | Exit 5% | — | — |
| 0.70–0.80 | Sell Tier 2 | Exit 10% | — | — |
| 0.80–1.00 | Sell Tier 3 | Exit 20% or full | — | — |

**Sell tiers only activate if holdings ≥ $500 per asset** — this is a portfolio-level gate applied externally.

## Data structure

```
data/
├── eth_risk_history.json      # scored output, one row per day since warmup
└── eth_prices_raw.json         # full price cache for indicator recomputation
```

Each row in `eth_risk_history.json`:
```json
{
  "date": "2026-08-24",
  "close": 2345.67,
  "composite_score": 0.5234,
  "zone": "Buy",
  "tier": "buy",
  "multiplier": 1.0,
  "size_usd": 10.0,
  "action": "Normal DCA",
  "components": {
    "log_regression": 0.4512,
    "ma200_multiple": 0.5678,
    "rsi14": 0.3456,
    "vol_adj_momentum": 0.5890
  }
}
```

## Caveat: Binance data window

ETHUSDT on Binance starts 2017-07-14. The log-regression component fits log(price) ~ f(log(days)) over this window only. Full Ethereum history goes back to 2015-07-30, so the regression is "warm" but not as anchored as if Binance data covered the full span. The percentile ranks adapt as history accumulates, so this becomes less of an issue over time.

## Manual testing

```bash
# Fetch and score 100 most recent daily candles
python eth_risk_score.py --update

# Check the score
jq '.[- 1]' data/eth_risk_history.json
```

## License

Same as btc-risk-score.
