# Trading strategy — Cryptids loop

Master switch for the scheduled trading ticks. The loop trades only while
`ARMED: true`; every tick re-reads this file, so changing it takes effect on
the next tick.

```
ARMED: true
```

## Idea

Buy fear, sell greed, hold through drawdowns. Buy only on sharp daily drops,
and only when the new buy lowers the average cost. Every position rests a
limit sell at +10% over its average cost. There are **no stop-losses**: a
position that drops is held (and may be averaged down) until the take-profit
fills. Every order is a **maker** limit order (0.5% fee at the starting
tier, versus 0.95% taker).

## Target

- Asset class: **crypto** (24/7). Do not trade equities or anything else.
- Universe: **BTC, ETH, ADA, SOL, XRP** (pairs `<SYMBOL>-USD`) until this line
  is edited. Skip a coin for the tick if `get_currency_pairs` shows it halted,
  not `tradable`, or `market_orders_only`.

## Hard caps (per CLAUDE.md rule 2)

- Buy size: **$200.00** per buy order (`dollar_amount: "200.00"`).
- New buy orders per tick: at most **1 per coin** (so up to 5 per tick).
- Open buys per coin: at most **1**. Never place a buy for a coin that
  already has an open loop buy order (`initiator_type: agentic`).
- Cash: never place a buy that `crypto_buying_power` can't cover.
- Buy halt: if the account is down **10% or more** today, place **no new
  buys**. Take-profit sell maintenance (below) continues. "Today" is
  estimated as `total_value` from `get_portfolio` versus
  `cash + Σ(quantity × open_price)` over held coins, using `open_price`
  (previous midnight close) from `get_crypto_quotes`. This is the only loss
  rule — nothing is ever sold at a loss by this loop.

## Maker-only execution

- Order type is always `limit`. Never `market`, `stop_loss` or `stop_limit`.
- Buys: limit price = current **bid** rounded **down** to the pair's
  `min_order_price_increment`. Never at or above the ask.
- Sells: limit price is the take-profit price, and never below the current
  **ask** (if price has already run past the target, use the ask, rounded
  **up** to the increment).
- Run `preview_crypto_order` first; place only if its `fee_rate` says
  **maker**. If it says taker, or anything else differs from what was
  intended, skip the order.

## Average cost

Read it from the position itself (`get_crypto_positions`), never from order
history or memory of past ticks. Summing over **every** entry in the coin's
`cost_bases`:

```
average cost = (Σ direct_cost_basis + Σ intraday_cost_basis)
             / (Σ direct_quantity   + Σ intraday_quantity)
```

Today's buys are reported under `intraday_*` and roll into `direct_*` after
the day closes, so a coin bought across several days has part of its cost in
each — always add both. The cost basis excludes the buy fee. If the summed
quantity is 0, or differs from the position's `quantity` by more than one
`min_order_quantity_increment` (units with no captured cost), the position has
no usable cost basis.

## Each tick

1. Gather: `get_portfolio`, `get_crypto_positions`, open orders
   (`get_crypto_orders`, `state_group: open`), `get_crypto_quotes` for the
   universe (pass `rhs_account_number`), and `get_currency_pairs`
   constraints for the universe.
2. **Stale buys.** Cancel any open loop buy order older than **60 minutes**.
   Unfilled buys are re-decided fresh; a partial fill keeps its filled part.
3. **Take-profit maintenance** (runs even under the buy halt). For each held
   universe coin, the target is one open limit sell, entered by `quantity`,
   for the full held quantity (including coins already reserved by the
   loop's current sell), rounded down to `min_order_quantity_increment`, at
   `average cost × 1.10` (see **Average cost**) rounded up to the price
   increment.
   If there's no such sell, or its quantity or price doesn't match (e.g. a
   buy filled since it was placed), cancel the mismatched sell and place the
   correct one. Leave a matching sell alone. If a position has no usable
   cost basis, place no sell for it and report it as an anomaly.
4. **Fear buy** (skip entirely under the buy halt). A coin qualifies when:
   - `mark_price` is at least **4% below** `open_price` (previous close), and
   - if already held: `mark_price` is also at least **5% below** its
     average cost (so each buy lowers the average), and
   - it has no open loop buy order.
   Buy **every** qualifying coin, one $200 maker limit buy each (see
   execution rules), working from the largest drop versus `open_price` to
   the smallest. Before each buy, re-check `crypto_buying_power` (open buys
   placed earlier this tick reserve cash); if it can't cover the buy, stop
   buying for this tick. If a preview or placement errors, stop buying for
   this tick (CLAUDE.md rule 7).
5. Otherwise do nothing. Doing nothing is the expected outcome of most
   ticks.

Orders this loop manages are only those with `initiator_type: agentic` on
universe coins. Never cancel or replace anything else.

## Status notes

- 2026-09-23: Crypto tools are live on the connector (`get_crypto_*`,
  `preview_crypto_order`, `place_crypto_order`, `cancel_crypto_order`). The
  Agentic account is funded ($100) with a linked crypto account. Preview
  verified: limit buy below the market → 0.5% maker; market buy → 0.95%
  taker. All five universe pairs tradable, not halted, limit orders allowed.
- There is no crypto historicals tool; `open_price` (previous midnight
  close, US Eastern) is the only reference price besides average cost.
