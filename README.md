# Cryptids

Trading-agent workspace wired to **Robinhood Agentic Trading** via the Model
Context Protocol (MCP).

This repo's [`.mcp.json`](.mcp.json) registers Robinhood's official Trading MCP
server, so any MCP-capable agent opened in this project (Claude Code, Cursor,
Codex CLI, …) can connect to Robinhood after a one-time OAuth login.

## The server

| | |
|---|---|
| Endpoint | `https://agent.robinhood.com/mcp/trading` |
| Transport | Streamable HTTP |
| Auth | OAuth 2.0 authorization code + PKCE (S256), dynamic client registration, refresh tokens |
| Authorization | `https://robinhood.com/oauth` (you log in on Robinhood — the agent never sees your password) |

## Setup

The project config in `.mcp.json` is picked up automatically by Claude Code in
this directory. Approve the server when prompted, then run `/mcp` to start the
OAuth login. To register it globally instead:

```sh
claude mcp add robinhood-trading --transport http https://agent.robinhood.com/mcp/trading
```

What the connection flow looks like:

1. Your client auto-registers with Robinhood (dynamic client registration) and
   opens `robinhood.com/oauth` in a browser. **Desktop only** — the OAuth flow
   doesn't work from a phone.
2. Log in on Robinhood's page; the agent never handles your password.
3. Approve the connection in the **Robinhood mobile app** when prompted.
4. The browser redirects to a `localhost` URL. A "can't connect" browser error
   at that step is normal — if your client asks for it, paste that redirect
   URL back in; the authorization code inside it is what matters.
5. Refresh tokens keep the session alive; you can instantly disconnect the
   agent from the Robinhood app at any time.

Other clients (Claude Desktop, ChatGPT Developer Mode, Cursor, Codex CLI, Grok)
take the same URL as a custom connector — see
[Robinhood's Agentic Trading overview](https://robinhood.com/us/en/support/articles/agentic-trading-overview/).

You also need a **Robinhood Agentic account** — a dedicated brokerage account
created and funded in the Robinhood app. Agents can *read* all of your
Robinhood accounts, but can only *trade* inside the Agentic account.

Note for Claude Code: the app must stay open on your computer for the agent to
place or manage orders — there's no server-side automation once you close it.

## What connected agents can do

- Read account numbers, balances, positions, transaction history, and
  watchlists across your Robinhood accounts
- Get quotes, search instruments, and manage watchlists
- Preview, place, cancel, and track equity orders in the dedicated Agentic
  account

At launch the beta trades **equities only**. Robinhood's product pages
advertise options and crypto, and the announcement lists options, crypto,
event contracts, and futures as "coming soon" — check the current state in
the app before assuming an asset class is tradable.

### Live tool list (observed 2026-08-20 via the connector)

| Category | Tools |
|---|---|
| Accounts & portfolio | `get_accounts`, `get_portfolio`, `get_pnl_trade_history`, `get_realized_pnl`, `get_equity_tax_lots`, `get_limited_margin_upgrade_info` |
| Equity data | `get_equity_quotes`, `get_equity_historicals`, `get_equity_fundamentals`, `get_equity_price_book`, `get_equity_technical_indicators`, `get_equity_tradability`, `get_financials`, `search` |
| Equity trading | `review_equity_order`, `place_equity_order`, `cancel_equity_order`, `get_equity_orders`, `get_equity_positions` |
| Options | `get_option_chains`, `get_option_quotes`, `get_option_instruments`, `get_option_historicals`, `get_option_positions`, `get_option_orders`, `review_option_order`, `place_option_order`, `cancel_option_order`, `exercise_option`, `cancel_option_exercise`, `get_option_level_upgrade_info` |
| Indexes & earnings | `get_indexes`, `get_index_quotes`, `get_index_historicals`, `get_earnings_calendar`, `get_earnings_results` |
| Watchlists | `get_watchlists`, `get_watchlist_items`, `create_watchlist`, `update_watchlist`, `add_to_watchlist`, `remove_from_watchlist`, `follow_watchlist`, `unfollow_watchlist`, `get_popular_watchlists`, `get_option_watchlist`, `add_option_to_watchlist`, `remove_option_from_watchlist` |
| Scanners | `get_scans`, `create_scan`, `run_scan`, `update_scan_config`, `update_scan_filters`, `get_scanner_filter_specs` |

**No crypto tools are exposed yet** despite Robinhood's marketing pages —
crypto trading through this MCP is not possible until they appear.

## Unattended operation (cloud, 24/7)

This repo is set up to run as an autonomous trading loop in Claude Code cloud
sessions, with the Robinhood server authorized as a claude.ai connector
(named **Cryptids**) so no local machine needs to be on.

- Four scheduled Routines ("Cryptids trading tick" at :01/:16/:31/:46) give a
  15-minute cadence — the Routine scheduler's minimum interval is hourly, so
  the cadence is built from four staggered hourly triggers. Each tick is a
  fresh session that clones this repo.
- [`CLAUDE.md`](CLAUDE.md) defines the hard operating rules for tick
  sessions; [`STRATEGY.md`](STRATEGY.md) defines the strategy, caps, and the
  `ARMED` master switch (currently **false**).
- `.claude/settings.json` pre-approves the Robinhood connector tools so
  unattended sessions never stall on a permission prompt, deny-lists option
  placement/exercise, and deny-lists WebFetch/WebSearch so sessions holding
  trade authority never ingest untrusted web content.

To go live: fund the Agentic account, confirm crypto tools exist, enable the
four Routines, and set `ARMED: true` in STRATEGY.md.

## Guardrails and risk — read before trading

- **Real money.** Orders placed through this MCP server execute in a live
  brokerage account. Fund the Agentic account only with capital you are
  prepared to let an agent manage.
- **Confirmation is configurable, not guaranteed.** Agents can be instructed
  to place trades without per-order confirmation. Keep order preview /
  manual review on unless you have a tested strategy — a good standing rule
  is to require the agent to run `review_equity_order` and show you the
  result before any `place_equity_order` call.
- **Prompt injection is a live threat.** An agent with broad read access and
  unconfirmed-trade capability can be manipulated by malicious content it
  reads (web pages, documents, tool output) into placing trades. Treat
  anything the agent ingests while connected as part of your attack surface.
- **Read scope is wide.** The server exposes read access to *all* your
  Robinhood accounts, not just the Agentic one. Only connect agents you trust,
  and remember data shared with an AI provider leaves Robinhood's environment.
- **You own the outcome.** Per Robinhood's terms, losses from agent-generated
  decisions are the user's responsibility. Agents can misread instructions or
  act on stale data.
- **Disconnect is instant.** You can revoke agent access at any time from the
  Robinhood app.
- Limits: at most 10 self-directed individual accounts per person (the Agentic
  account counts); crypto trading is unavailable in some states (including
  New York); agents can trade crypto but not transfer, stake, or lend it.

## References

- [Agentic Trading overview (support)](https://robinhood.com/us/en/support/articles/agentic-trading-overview/)
- [Agentic Trading product page](https://robinhood.com/us/en/agentic-trading/)
- [Robinhood is Now Open to Agents (announcement)](https://robinhood.com/us/en/newsroom/robinhood-is-now-open-to-agents/)
