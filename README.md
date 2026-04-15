# MagicMarkets Datafeed Plugin for Claude Code

A Claude Code plugin that helps you integrate the **MagicMarkets Datafeed API** — a real-time WebSocket feed of best-available sports betting odds from aggregated bookmakers.

## What it does

When you ask Claude Code about streaming odds, connecting to the datafeed, or building an odds comparison site, this plugin provides:

- **Accurate API guidance** — correct endpoints, auth flows, WebSocket protocol, and message formats
- **Working code snippets** — Python and Node.js examples for connecting, parsing, and maintaining state
- **Live API calls** — if you have credentials configured, Claude can demonstrate the login and test connectivity
- **Reference lookups** — sport codes, bookmaker codes, bet type formats, exchange commissions

## Install

```bash
claude plugin install magicmarkets-datafeed
```

## Quick start

After installing, just ask Claude Code naturally:

- *"How do I connect to the MagicMarkets datafeed?"*
- *"Write a Python script that streams live football odds"*
- *"What sport codes does MagicMarkets support?"*
- *"Explain the WebSocket message format for price updates"*

## API overview

| Detail | Value |
|--------|-------|
| **Base URL** | `https://data.magicmarkets.com` |
| **WebSocket** | `wss://data.magicmarkets.com/v1/stream?token=<token>` |
| **Auth** | Session token via `POST /v1/login` (24h TTL) |
| **Protocol** | Server-to-client only; snapshot on connect, then real-time deltas |
| **Sports** | Football, basketball, tennis, ice hockey, baseball, and 16 more |
| **Bookmakers** | Betfair, Betdaq, Matchbook, Pinnacle, SBOBet, and 4 more |

## Configuration

If you want Claude to make live API calls on your behalf, add your credentials to a `.env` file in your project:

```
DATAFEEDS_USERNAME=your-username
DATAFEEDS_PASSWORD=your-password
```

## Documentation

- [Full API docs](https://data.magicmarkets.com)
- Plugin reference files are in `skills/magicmarkets-datafeed/references/`
