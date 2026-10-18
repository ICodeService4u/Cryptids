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
limit sell at +5% over its average cost. There are **no stop-losses**: a
position that drops is held (and may be averaged down) until the take-profit
fills. Every order is a **maker** limit order (0.5% fee at the starting
tier, versus 0.95% taker).

## Target

- Asset class: **crypto** (24/7). Do not trade equities or anything else.
- Universe: **BTC, ETH, ADA, SOL, XRP** (pairs `<SYMBOL>-USD`) until this line
  is edited. Skip a coin for the tick if `get_currency_pairs` shows it halted,
  not `tradable`, or `market_orders_only`.

## Hard caps (per CLAUDE.md rule 2)

- Buy size: **$400.00** per buy order (`dollar_amount: "400.00"`).
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
- Quote right before you price. Call `get_crypto_quotes` (with
  `rhs_account_number`) immediately before each preview and price from
  that quote, not from the tick's opening gather. The preview labels an
  order maker or taker against the live book at the moment of the preview,
  and a quote more than a few seconds old reads taker in a moving market
  (see the 2026-10-12 status note).
- Buys: limit price = current **bid** rounded **down** to the pair's
  `min_order_price_increment`, then **one increment lower**. Never at or
  above the ask.
- Sells: limit price is the take-profit price, and never below the current
  **ask** (if price has already run past the target, use the ask rounded
  **up** to the increment, then one increment higher).
- Run `preview_crypto_order` first; place only if its `fee_rate` says
  **maker**, and place at exactly the previewed price. If it says taker,
  re-quote, re-price by the rules above and preview once more; if that
  preview also says taker, skip the order. If anything else differs from
  what was intended, skip the order.

## Average cost

Never from memory of past ticks. Use the first source that works; if
neither does, the position has **no usable cost basis**.

**1. Position cost basis.** From the position itself
(`get_crypto_positions`), summing the `direct_*` fields over **every** entry
in the coin's `cost_bases`:

```
average cost = Σ direct_cost_basis / Σ direct_quantity
```

The `direct_*` fields already cover every held unit with a captured cost,
today's buys included. The `intraday_*` fields are the part of that bought
today, not a separate lot: a coin bought today shows the same quantity and
cost under both, so adding them double-counts it (see the 2026-10-12
cost-basis status note). Never add `intraday_*` to the sum. The cost basis
excludes the buy fee. Usable only if Σ `direct_quantity` is above 0 and
within one `min_order_quantity_increment` of the position's `quantity`.

**2. Derived from the loop's sell** (fallback when source 1 isn't usable —
the connector has been seen returning all-zero `cost_bases`). Every loop
sell is priced at average cost × 1.05, so it records the average cost of
the units it covers. Requires exactly one open loop sell on the coin (the
**anchor**):

```
anchor qty  = anchor quantity − anchor cumulative_quantity
anchor cost = anchor qty × anchor price / 1.05
new fills   = loop buys on the coin (side buy, initiator_type agentic,
              cumulative_quantity > 0) with updated_at ≥ anchor created_at
average cost = (anchor cost + Σ new-fill average_price × cumulative_quantity)
             / (anchor qty  + Σ new-fill cumulative_quantity)
```

A loop sell created before 2026-10-13 was priced at average cost × 1.10
(the take-profit was +10% until then): for such an anchor divide by 1.10
instead of 1.05.

Fetch the new fills with `get_crypto_orders` (`symbol`, `side: buy`,
`created_at_gte` a day before the anchor's `created_at`, then filter).
Usable only if `anchor qty + Σ new-fill quantity` is within one
`min_order_quantity_increment` of the position's `quantity`; otherwise the
units can't be accounted for. Report which source you used in the summary
whenever it is source 2.

Rounding the sell price up makes the derived cost read at most one price
increment ÷ 1.05 high, which is acceptable. The derivation is only as good
as the anchor, so under source 2 never price a sell at the ask (see step 3).

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
   `average cost × 1.05` (see **Average cost**) rounded up to the price
   increment.
   If there's no such sell, or its quantity or price doesn't match (e.g. a
   buy filled since it was placed), cancel the mismatched sell and place the
   correct one. Leave a matching sell alone; a price within one price
   increment of the target counts as matching (dividing by 1.05 and
   multiplying back can round up a step). Work out the new sell's
   price and quantity **before** cancelling the old one — under source 2
   the old sell is the anchor. If the replacement can't be placed after the
   cancel, that is a write error (CLAUDE.md rule 7).
   Under source 2, if the target price is below the current ask, leave the
   existing sell as it is this tick instead of placing a sell at the ask.
   If a position has no usable cost basis, place no sell for it, cancel
   nothing on it, and report it as an anomaly.
4. **Fear buy** (skip entirely under the buy halt). A coin qualifies when:
   - `mark_price` is at least **4% below** `open_price` (previous close), and
   - if already held: `mark_price` is also at least **5% below** its
     average cost (so each buy lowers the average), and
   - if already held, the position has a usable cost basis (see **Average
     cost**) — a held coin without one never qualifies, and
   - it has no open loop buy order.
   Buy **every** qualifying coin, one $400 maker limit buy each (see
   execution rules), working from the largest drop versus `open_price` to
   the smallest. Before each buy, re-check `crypto_buying_power` (open buys
   placed earlier this tick reserve cash); if it can't cover the buy, stop
   buying for this tick. A taker label on a preview is not an error: handle
   it with the re-quote-and-retry rule under **Maker-only execution**, and
   skip the coin if the retry also reads taker. If a preview or placement
   errors, stop buying for this tick (CLAUDE.md rule 7).
5. Otherwise do nothing. Doing nothing is the expected outcome of most
   ticks.

Orders this loop manages are only those with `initiator_type: agentic` on
universe coins. Never cancel or replace anything else.

## Owner-authorized manual orders

The owner sometimes trades the Agentic account by hand through the same
connector, so those orders also show `initiator_type: agentic`. Orders
listed here are the owner's, not the loop's and not an anomaly: don't
flag them. They are all closed; there is nothing to manage.

- 2026-09-24 22:37–22:42 ET (2026-09-25 02:37–02:42 UTC): owner exited
  ADA and XRP. Cancelled the loop's take-profit sells, re-placed sells at a
  lower price and cancelled those too, then sold both at market:
  - ADA limit sell `6ab5de97-b731-42a9-a3d6-8edca145566b` (canceled)
  - XRP limit sell `6ab5de9e-6772-46ce-890f-a7472dec1d81` (canceled)
  - XRP market sell `6ab5df88-b978-4f50-ba66-f24199473987`, 132.317 filled
  - ADA market sell `6ab5df8b-7265-425e-9fcf-a0d328657e41`, 834.6 filled

  Afterwards the account held no crypto and no open orders, only cash.

- 2026-10-11 13:35–13:36 ET (17:35–17:36 UTC): owner exited ETH, XRP, SOL
  and ADA. Cancelled the loop's four take-profit sells, then sold all four
  at market:
  - ADA limit sell `6ac8805d-de25-4466-a865-166f0f066aa8` (canceled)
  - SOL limit sell `6ac7bef0-d87a-44ee-835a-5cf62112321a` (canceled)
  - XRP limit sell `6ac7beed-b0a4-42b3-a35a-d66c13e65aa1` (canceled)
  - ETH limit sell `6ac7beea-c6f8-4088-a738-623701c4d888` (canceled)
  - ETH market sell `6acbc91b-67a8-45b9-98e3-67b64103c255`, 0.081054 filled
  - XRP market sell `6acbc91c-ee4f-458c-b3a4-2408c4b4385e`, 147.301 filled
  - SOL market sell `6acbc91d-5c2b-4479-962d-149ecb2f0b1f`, 1.82875 filled
  - ADA market sell `6acbc91e-29be-4bf0-96f3-d497888d9cdf`, 3372.51 filled

  Afterwards the account held no crypto and no open orders, only cash.

## Status notes

- 2026-09-23: Crypto tools are live on the connector (`get_crypto_*`,
  `preview_crypto_order`, `place_crypto_order`, `cancel_crypto_order`). The
  Agentic account is funded with a linked crypto account. Preview
  verified: limit buy below the market → 0.5% maker; market buy → 0.95%
  taker. All five universe pairs tradable, not halted, limit orders allowed.
- There is no crypto historicals tool; `open_price` (previous midnight
  close, US Eastern) is the only reference price besides average cost.
- 2026-09-24: `get_crypto_positions` returns all-zero `cost_bases` for the
  XRP and ADA positions bought 2026-09-23, although the Robinhood app shows
  their average costs (XRP ≈ 1.504, ADA ≈ 0.23844). Average cost falls back
  to source 2 (derived from the loop's sell) for such positions.
- 2026-10-12: The 17:46 ET tick skipped its ADA and SOL buys because both
  previews read 0.95% taker. The fee tier and routing had not changed:
  every loop limit order through 2026-10-09 was 0.5% maker, and previews
  priced below the live bid still read 0.5% maker. The preview labels an
  order against the live book at the moment of the preview, and a buy at
  or above the live bid reads taker even when it is below the ask. The
  tick had priced ADA at the bid from a quote 26 seconds old and SOL from
  one 39 seconds old while the market was falling about 1.7% per quarter
  hour, so both prices were at or above the live bid by preview time.
  Quotes also show crossed books at times (bid above ask on BTC, SOL and
  ETH that afternoon), so the quoted bid is not a guaranteed resting
  price. Hence the quote-right-before-preview, one-increment-under and
  retry-once rules under **Maker-only execution**. The owner then had two
  $300 maker buys placed by hand under those rules (ADA at 0.2381447, SOL
  at 106.98, both `initiator_type: agentic`); the loop manages them like
  its own.
- 2026-10-12 (cost basis): The 18:16 ET tick left SOL without a take-profit
  sell. The position held 2.79031 SOL from the owner's $300 buy (filled
  18:00 ET at 106.98; cost 298.51 before the 1.49 fee), and its single
  `cost_bases` entry read 2.79031 / 298.51 under `direct_*` **and**
  2.79031 / 298.51 under `intraday_*`. Adding both, as **Average cost**
  then said to, gave 5.58062 units for 2.79031 held, so source 1 failed
  its quantity check; the owner's 2026-10-11 exit had cancelled every loop
  sell, so source 2 had no anchor either. ADA's buy filled at 18:17 ET and
  shows the same shape (1253.47 / 298.51 under both). The connector's own
  field notes say an entry's per-unit cost is `direct_cost_basis /
  direct_quantity` and that `intraday_*` only isolates today's still-held
  portion of it, so `intraday_*` is a subset of `direct_*`, not a second
  lot; the 2026-09-23 "roll into direct after the day closes" wording was
  an assumption. Source 1 now sums `direct_*` only, which gives SOL
  ≈ 106.98 and ADA ≈ 0.23815 with the full quantity accounted for.
- 2026-10-13 (take-profit): Lowered the take-profit from +10% to +5% over
  average cost to turn positions over faster. Fear buys enter at least 4%
  under the previous close, so +5% is roughly a return to the previous
  close (0.96 × 1.05 ≈ 1.008), while +10% needed about 5.6% above it. Of
  the loop's seven closed positions through 2026-10-11, three had reached
  +5% (the ADA take-profit fill on 10-04, and ADA at +8.2% and XRP at
  +5.3% when the owner exited on 10-11) but only one reached +10% (that
  ADA fill, 5.8 days after its buy). The owner's two manual exits (09-24,
  10-11) closed positions between +1.8% and +8.2% at the 0.95% taker rate;
  a resting +5% maker sell would have closed ADA and XRP on 10-11 at 0.5%.
  Net per $400 round trip at 0.5% maker each way is $15.82 (3.96%) at +5%
  versus $35.62 (8.91%) at +10%, so +5% needs about 2.25× as many fills to
  match realized P&L per cycle; it also frees each buy's cash sooner under
  the no-stop-loss rule. The owner had the open sells re-placed at × 1.05
  by hand on 2026-10-12 20:32–20:34 ET through the connector (so they show
  `initiator_type: agentic`); the loop manages them like its own:
  - ADA sell `6acd7c8e-2f0f-4cdf-b0ab-1b8349c449c0`, 1253.47 at 0.2500543
  - SOL sell `6acd7c8f-f4f9-4226-8516-8b2d76603cee`, 2.79031 at 112.33
  - ETH sell `6acd7ca2-5fda-401b-87c9-9dc81de117e1`, 0.123203 at 2544.06
  - XRP sell `6acd7ca3-70fd-4727-81b0-2f71a2136025`, 224.441 at 1.397

  The 20:31 ET tick, still on the +10% rule, ran at the same time: it
  placed an ETH sell at 2665.21 and re-placed XRP at 1.464 (the 20:18 tick
  had put XRP at 1.47, seven increments above its 1.463 target). Both were
  cancelled along with the earlier ADA 0.2619616 and SOL 117.68 sells.
- 2026-10-13 (buy size): Raised the buy size from $300 to $400 after the
  owner deposited another $1,000 (pending on 2026-10-12; cash was
  $2,275.06 before it).
