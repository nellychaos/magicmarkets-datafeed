---
name: magicmarkets-datafeed
description: >
  MagicMarkets Datafeed API assistant — real-time sports betting odds streamed
  over WebSocket at data.magicmarkets.com. Helps external developers integrate
  the B2B price feed into odds comparison platforms, trading dashboards, and data
  products. Use this skill whenever the user mentions MagicMarkets datafeed,
  odds streaming, WebSocket prices, live odds, price comparison, best price feed,
  or wants to display or consume real-time sports betting market data. Also
  trigger when the user asks about connecting to data.magicmarkets.com, sport
  codes, bookmaker price aggregation, or exchange commission adjustments.
allowed-tools: Bash(curl:*), Bash(python3:*), Bash(wscat:*), Bash(cat:*), Bash(which:*)
---

# MagicMarkets Datafeed API

You are helping an external developer integrate the MagicMarkets Datafeed API —
a B2B real-time price feed that streams the best available odds across multiple
bookmakers over WebSocket.

| Detail | Value |
|--------|-------|
| **Base URL** | `https://data.magicmarkets.com` |
| **WebSocket** | `wss://data.magicmarkets.com/v1/stream?token=<token>` |
| **Auth** | Session token via `POST /v1/login` (24-hour TTL) |
| **Protocol** | Server-to-client only; full snapshot on connect, then real-time deltas |

For full field-by-field details, read `references/datafeed-reference.md` in this
skill directory.

---

## How it works

1. **Authenticate** — `POST /v1/login` with username/password → 24-hour hex token
2. **Connect WebSocket** — `wss://data.magicmarkets.com/v1/stream?token=<token>`
3. **Receive snapshot** — server sends all current events, then all current prices
4. **Process updates** — real-time upserts and deletes as prices and events change

The stream is **server-to-client only**. Sending anything returns an error.

---

## Authentication

```bash
curl -X POST https://data.magicmarkets.com/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username": "YOUR_USER", "password": "YOUR_PASS"}'
```

**Response:**

```json
{
  "token": "2cd88d290124890ed2b6e1debbb43111",
  "customer": {
    "id": 167,
    "username": "my-user",
    "active": true,
    "ccy_code": "GBP",
    "config": {
      "prices_bookies": ["<source_1>", "<source_2>", "..."],
      "superuser": false
    }
  }
}
```

The `prices_bookies` array contains the opaque source codes that feed into
your best-price calculation. Your own login response will contain the actual
codes your account is subscribed to — treat them as opaque identifiers.
The token is a 32-character hex string valid for **24 hours**.

Other REST endpoints:
- `POST /v1/logout` — invalidate token (`Authorization: Token <token>`)
- `GET /v1/config` — view account settings
- `POST /v1/config` — change password (8+ chars, mixed case + number)

---

## WebSocket frame format

Each frame is JSON with a timestamp and batched messages:

```json
{
  "ts": 1765982942.805292,
  "data": [<message1>, <message2>, ...]
}
```

Messages are batched up to ~4 KB per frame. `ts` is Unix seconds with
microsecond precision.

---

## Message types

Two data tables — **events** and **sptmkt** — each with upsert and delete:

### Event upsert

New event or metadata/in-play change:

```json
["upsert", "events", ["fb", "2025-12-20,54321,12345"], {
  "competition_id": 100,
  "competition_name": "Premier League",
  "competition_country": "England",
  "home": "Liverpool",
  "away": "Manchester United",
  "start_ts": "2025-12-20T15:00:00Z",
  "ir": true,
  "score": [2, 1],
  "ir_time": ["2h", 65]
}]
```

**Key:** `[sport_code, event_id]`

| Field | Type | Description |
|-------|------|-------------|
| `competition_id` | integer | Competition/league ID |
| `competition_name` | string | Competition name |
| `competition_country` | string | Country code |
| `home` | string | Home team/player |
| `away` | string | Away team/player |
| `start_ts` | string or null | ISO 8601 start time |
| `ir` | boolean | `true` if in-play |
| `score` | `[int, int]` or null | `[home, away]` for in-play football; `null` otherwise |
| `ir_time` | `[string, int]` or null | `[period, minutes]` for in-play football; `null` otherwise |

**`ir_time` periods:** `"1h"` (first half), `"2h"` (second half), `"ht"` (half time)

### Event delete

Event finished or cancelled:

```json
["delete", "events", ["fb", "2025-12-20,54321,12345"]]
```

### Price upsert

Best price changed for a market selection:

```json
["upsert", "sptmkt", ["basket", "2025-12-17,73821,40727", "for,ah,h,-6"], {"price": 1.49}]
```

**Key:** `[sport_code, event_id, bet_type]`

The `price` is the best decimal odds across your configured bookmakers, rounded
to 3 decimal places.

### Price delete

No bookmaker offers this market anymore:

```json
["delete", "sptmkt", ["basket", "2025-12-17,73821,40727", "for,ah,h,-6"]]
```

---

## Best price calculation

For each `(sport, event_id, bet_type)`:

1. Collect offers from all sources in your `prices_bookies` set
2. For exchange-style sources, adjust for commission:
   `adjusted = price - (price - 1) * commission`
3. Round down to 3 decimal places
4. Return the **maximum** adjusted price

Commission rates are typically around 1–2% for exchange-style sources and zero
for the rest. Your account configuration determines which sources are
included and whether their commission is already applied to the delivered
prices. Contact MagicMarkets for the commission schedule applicable to your
account.

**Example:** a source offering 2.50 at 2% commission → adjusted =
`2.50 - (2.50 - 1) * 0.02` = **2.47**. If another source in your set offers
2.48 with no commission, the best price returned is 2.48.

---

## Code examples

### Python — connect and stream

```python
import asyncio, json, websockets, requests

USERNAME = "YOUR_USER"
PASSWORD = "YOUR_PASS"

# 1. Login
token = requests.post("https://data.magicmarkets.com/v1/login",
    json={"username": USERNAME, "password": PASSWORD}).json()["token"]

# 2. In-memory state
events = {}   # (sport, event_id) -> metadata dict
prices = {}   # (sport, event_id, bet_type) -> price float

def process(msg):
    action, table = msg[0], msg[1]
    key = tuple(msg[2])
    if table == "events":
        if action == "upsert":
            events[key] = msg[3]
        elif action == "delete":
            events.pop(key, None)
    elif table == "sptmkt":
        if action == "upsert":
            prices[key] = msg[3]["price"]
        elif action == "delete":
            prices.pop(key, None)

# 3. Stream
async def stream():
    uri = f"wss://data.magicmarkets.com/v1/stream?token={token}"
    async with websockets.connect(uri) as ws:
        async for raw in ws:
            frame = json.loads(raw)
            for msg in frame["data"]:
                process(msg)
            print(f"{len(events)} events, {len(prices)} prices")

asyncio.run(stream())
```

### Node.js — connect and stream

```javascript
import WebSocket from "ws";

const token = "YOUR_TOKEN";
const events = new Map();
const prices = new Map();

const ws = new WebSocket(
  `wss://data.magicmarkets.com/v1/stream?token=${token}`
);

ws.on("message", (raw) => {
  const { data } = JSON.parse(raw);
  for (const msg of data) {
    const [action, table, key, payload] = msg;
    const k = key.join("|");
    if (table === "events") {
      action === "upsert" ? events.set(k, payload) : events.delete(k);
    } else if (table === "sptmkt") {
      action === "upsert" ? prices.set(k, payload.price) : prices.delete(k);
    }
  }
  console.log(`${events.size} events, ${prices.size} prices`);
});
```

---

## Reconnection

- Connections are **stateless** — no subscriptions, no sequence numbers
- If disconnected, reuse the token (if within 24h) or re-login
- The server sends a **fresh full snapshot** on every new connection
- Clear local state before processing the new snapshot

---

## Reference tables

### Sport codes

| Code | Sport | Code | Sport |
|------|-------|------|-------|
| `fb` | Football (soccer) | `tennis` | Tennis |
| `fb_ht` | Football - 1st half | `af` | American football |
| `fb_et` | Football - extra time | `ru` | Rugby union |
| `fb_corn` | Football - corners | `rl` | Rugby league |
| `fb_corn_ht` | Football - corners 1st half | `ih` | Ice hockey |
| `fb_book` | Football - bookings | `baseball` | Baseball |
| `basket` | Basketball | `hand` | Handball |
| `basket_ht` | Basketball - 1st half | `darts` | Darts |
| `mma` | MMA | `boxing` | Boxing |
| `snooker` | Snooker | `arf` | Australian rules football |
| `volley` | Volleyball | | |

### Sources

The `prices_bookies` array in your login response contains the opaque source
codes your account is subscribed to. Treat them as identifiers — do not hard-
code them into your application logic, since the set can change over time and
varies by account. If you need the current list of sources active on your
account, use the `prices_bookies` returned by `/v1/login` or `/v1/config`.

Some sources are exchange-style (commission applied before ranking), others
are sharp books (no commission adjustment). Contact MagicMarkets for details
on the commission schedule and source inventory applicable to your account.

### Bet types

Canonical comma-separated identifiers:

| Pattern | Meaning |
|---------|---------|
| `for,1x2,1` | Match result, home win |
| `for,1x2,x` | Match result, draw |
| `for,1x2,2` | Match result, away win |
| `for,ah,h,-6` | Asian handicap, home -6 |
| `for,ah,a,6` | Asian handicap, away +6 |
| `for,ou,o,42` | Over/under, over 42 |
| `for,ou,u,42` | Over/under, under 42 |
| `for,h` | Head-to-head / moneyline |

The exact set varies by sport and market availability.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `401` on `/v1/login` | Invalid username or password |
| WebSocket closes immediately | Token invalid or expired (24h TTL) |
| No prices for a sport | Your `prices_bookies` config may not cover that sport |
| Prices seem low | Exchange commission is being deducted (correct behavior) |
| Missing events after reconnect | Expected — snapshot only includes currently active events |

---

## Live data queries

If the developer has credentials (check for a `.env` file or ask), you can make
live API calls to demonstrate the login flow or test connectivity. Always show
the curl/code you're running so the developer can reproduce it.
