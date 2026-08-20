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

### Reported tool list

No official Robinhood page enumerates the MCP tools; the list below is
reported by third-party documentation of this endpoint and should be treated
as unverified until you connect and run `/mcp` to see the live list.

| Category | Tools |
|---|---|
| Read | `get_accounts`, `get_portfolio`, `get_equity_positions`, `get_equity_quotes`, `get_equity_orders`, `search` |
| Watchlists | `get_watchlists`, `add_to_watchlist`, `update_watchlist` |
| Trading | `review_equity_order`, `place_equity_order`, `cancel_equity_order` |

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
