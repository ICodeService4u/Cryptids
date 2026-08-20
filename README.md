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
this directory. Approve the server when prompted, then run `/mcp` and complete
the browser OAuth login with your Robinhood account.

To register it globally instead:

```sh
claude mcp add robinhood-trading --transport http https://agent.robinhood.com/mcp/trading
```

Other clients (Claude Desktop, ChatGPT Developer Mode, Cursor, Codex CLI, Grok)
take the same URL as a custom connector — see
[Robinhood's Agentic Trading overview](https://robinhood.com/us/en/support/articles/agentic-trading-overview/).

You also need a **Robinhood Agentic account** — a dedicated brokerage account
created and funded in the Robinhood app. Agents can *read* all of your
Robinhood accounts, but can only *trade* inside the Agentic account.

## What connected agents can do

- Read account numbers, balances, positions, transaction history, and
  watchlists across your Robinhood accounts
- Get quotes, search instruments, and manage watchlists
- Place, preview, and track equity orders in the dedicated Agentic account
  (options, crypto, event contracts, and futures are rolling out per
  Robinhood's announcement)

## Guardrails and risk — read before trading

- **Real money.** Orders placed through this MCP server execute in a live
  brokerage account. Fund the Agentic account only with capital you are
  prepared to let an agent manage.
- **Confirmation is configurable, not guaranteed.** Agents can be instructed
  to place trades without per-order confirmation. Keep order preview /
  manual review on unless you have a tested strategy.
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
