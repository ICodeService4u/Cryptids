# Trading strategy — Cryptids loop

Master switch for the scheduled trading ticks. The loop trades only while
`ARMED: true`; every tick re-reads this file, so changing it takes effect on
the next tick.

```
ARMED: false
```

## Target

- Asset class: **crypto** (24/7). Do not trade equities or anything else.
- Universe: BTC, ETH only until this line is edited.

## Hard caps (per CLAUDE.md rule 2)

- Max notional per order: $25
- Max orders per tick: 1
- Max total position: 50% of agentic-account value per asset
- Daily loss halt: stop placing orders for the rest of the day once the
  agentic account is down 5% from the day's starting value

## Decision rule (edit me — placeholder, intentionally conservative)

Each tick:

1. Fetch current quotes and recent historicals for the universe.
2. If flat and price is more than 3% below its 24h high → buy up to the
   per-order cap.
3. If holding and price is more than 2% above average cost, or more than 3%
   below average cost → sell the position (take-profit / stop-loss).
4. Otherwise do nothing. Doing nothing is the expected outcome of most ticks.

## Status notes

- 2026-08-20: Robinhood's MCP server exposes no crypto trading tools yet
  (equities and options only, verified against the live tool list) and the
  agentic account is unfunded. Until both change, every tick should stop at
  preflight. Leave ARMED false until you have funded the account, confirmed
  crypto tools exist, and reviewed the decision rule above.
