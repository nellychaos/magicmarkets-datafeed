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

### Python — connect, stream, and reconnect with state reset

```python
import asyncio, json, websockets, requests

USERNAME = "YOUR_USER"
PASSWORD = "YOUR_PASS"

# In-memory state
events = {}   # (sport, event_id) -> metadata dict
prices = {}   # (sport, event_id, bet_type) -> price float
last_ts = 0.0

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

def get_token():
    resp = requests.post("https://data.magicmarkets.com/v1/login",
        json={"username": USERNAME, "password": PASSWORD})
    resp.raise_for_status()
    return resp.json()["token"]

async def stream_forever():
    """Reconnect loop — reset state on every (re)connect because the
    server sends a fresh full snapshot each time."""
    token = get_token()
    while True:
        events.clear()      # fresh snapshot coming
        prices.clear()
        try:
            uri = f"wss://data.magicmarkets.com/v1/stream?token={token}"
            async with websockets.connect(uri) as ws:
                async for raw in ws:
                    global last_ts
                    frame = json.loads(raw)
                    last_ts = frame["ts"]
                    for msg in frame["data"]:
                        process(msg)
        except (websockets.exceptions.ConnectionClosed, OSError) as e:
            print(f"Disconnected: {e}; reconnecting in 1s")
            await asyncio.sleep(1)
        except websockets.exceptions.InvalidStatus:
            # 401: token expired (24h TTL); re-login
            token = get_token()

asyncio.run(stream_forever())
```

**Why `events.clear()` before every reconnect:** the server sends a full
snapshot of the currently-active set on connect. Events that were deleted
upstream during your disconnect won't be in the snapshot. If you keep
stale entries, they'll never be removed. Clearing and re-ingesting is
the simpler invariant than trying to reconcile deltas.

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

### Persisting a snapshot to disk

For bulk analytics or to share state across processes, you'll often want
to snapshot the in-memory state to a file and re-load it later. Two
gotchas:

1. **Tuple keys aren't JSON-serialisable.** The natural `(sport, event_id)`
   tuple key works great in memory but `json.dump(events)` raises
   `TypeError: keys must be str, int, float, bool or None`. Convert to
   pipe-delimited strings on write and split on load.
2. **Snapshots are point-in-time** — the messages have no sequence number
   of their own. Record the server's last `frame["ts"]` you processed so
   downstream consumers can age-check rather than trusting file mtime.

```python
import json

def write_snapshot(events: dict, prices: dict, last_ts: float, path: str):
    payload = {
        "last_ts": last_ts,
        "events": {f"{sp}|{eid}": meta
                   for (sp, eid), meta in events.items()},
        "prices": {f"{sp}|{eid}|{bt}": {"price": p}
                   for (sp, eid, bt), p in prices.items()},
    }
    with open(path, "w") as f:
        json.dump(payload, f)

def load_snapshot(path: str) -> tuple[dict, dict, float]:
    with open(path) as f:
        payload = json.load(f)
    events = {tuple(k.split("|", 1)): v for k, v in payload["events"].items()}
    prices = {tuple(k.split("|", 2)): v["price"] for k, v in payload["prices"].items()}
    return events, prices, payload.get("last_ts", 0.0)
```

**Snapshot age policy.** Active markets update sub-second. A 5-minute
snapshot is usually fine for pre-match screening; for in-play analytics,
reconnect and re-ingest rather than trusting a stale snapshot. Check
`time.time() - last_ts` against a freshness budget before using the
loaded state.

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

Bet-type strings use a canonical comma-separated grammar shared with the
Trading API. Every string starts with `for,...` (pick the outcome to
happen) or `against,...` (pick it NOT to happen). For the full
sport-keyed list of moneyline / handicap / totals patterns, see the
Bet-type patterns section in the `magicmarkets-trading` skill — the
same grammar drives betslip creation there.

**Common patterns you'll see in the stream:**

| Pattern | Meaning |
|---------|---------|
| `for,h` / `for,a` / `for,d` | Soccer (`fb`) moneyline: home / away / draw |
| `for,ml,h` / `for,ml,a` | Basketball / MMA moneyline |
| `for,tp,all,ml,h` / `for,tp,all,ml,a` | Ice hockey / NFL / MLB moneyline incl. OT |
| `for,tp,reg,wdw,h/d/a` | Regulation-time win-draw-win (NHL, etc.) |
| `for,dnb,h` / `for,dnb,a` | Draw-no-bet (MMA, boxing) |
| `for,tset,all,vset1,p1/p2` | Tennis total sets, player 1 vs player 2 |
| `for,ah,h,-6` | Asian handicap home -1.5 |
| `for,ah,a,6` | Asian handicap away +1.5 |
| `for,ou,o,42` | Over/under, over line |
| `for,ahover,N` / `for,ahunder,N` | Combined AH-over / AH-under (fb corners etc.) |

**Quarter-point line encoding.** Asian handicap and over/under lines are
integers in **quarter-point units** — divide by 4 to get the displayed
line. So `-6` displays as `-1.5`, `-8` as `-2.0`, `20` as `+5.0`. The
same encoding applies on the Trading API side; a pattern you see in the
Datafeed can be used verbatim in a `POST /v1/betslips/` call.

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

## Known limitations

- **Server-to-client only.** No client-side subscribe/filter protocol —
  you receive the full stream for your account and filter locally. Any
  message you send back closes the connection with an error.
- **Source set can change silently.** The `prices_bookies` array in your
  login response can grow, shrink, or rotate between sessions. Don't
  cache it across process restarts.
- **24-hour token TTL, no renewal.** There is no refresh endpoint; at or
  before the 24-hour mark you must re-`POST /v1/login`. A long-running
  stream will disconnect silently after ~24 hours. Pre-emptively re-login
  at 23 hours if you're running a persistent process.
- **Point-in-time snapshots.** The server sends a fresh full snapshot on
  connect but no sequence number. On reconnect you must clear local
  state and re-ingest (see the reconnect example above).
- **In-play fields are football-specific.** `score` and `ir_time` are
  populated for in-play football events only; other sports leave them
  null even when `ir: true`. Check the sport before reading them.

---

## Live data queries

If the developer has credentials (check for a `.env` file or ask), you can make
live API calls to demonstrate the login flow or test connectivity. Always show
the curl/code you're running so the developer can reproduce it.
