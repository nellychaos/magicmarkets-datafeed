# Datafeed API — Full Reference

Base URL: `https://data.magicmarkets.com`

## Table of Contents

1. [Authentication](#authentication) (lines ~10-60)
2. [Configuration](#configuration) (lines ~65-95)
3. [WebSocket stream](#websocket-stream) (lines ~100-150)
4. [Message types](#message-types) (lines ~155-265)
5. [Best price calculation](#best-price-calculation) (lines ~270-300)
6. [Reconnection](#reconnection) (lines ~305-320)
7. [Sport codes](#sport-codes) (lines ~325-365)
8. [Bookmakers](#bookmakers) (lines ~370-400)
9. [Bet types](#bet-types) (lines ~405-430)

---

## Authentication

### POST `/v1/login`

Authenticate with credentials to receive a 24-hour session token.

**Request:**

```json
{
  "username": "<username>",
  "password": "<password>"
}
```

**Response (200):**

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

The `token` is a 32-character hex string, valid for **24 hours**.

The `prices_bookies` array determines which sources are included in your
best-price calculation. Your response will contain opaque source identifiers
specific to your account subscription. Treat them as identifiers — the set
can change over time and varies by account.

**Error:** `401 Unauthorized` if credentials are invalid.

### POST `/v1/logout`

Invalidate the current session token.

```
Authorization: Token <token>
```

---

## Configuration

### GET `/v1/config`

Returns current account settings (same shape as login response).

```
Authorization: Token <token>
```

### POST `/v1/config`

Update account settings. Currently supports password changes only.

**Password policy:** at least 8 characters, including at least one uppercase
letter, one lowercase letter, and one number.

```json
{
  "password": "<new_password>"
}
```

---

## WebSocket stream

### Connecting

```
wss://data.magicmarkets.com/v1/stream?token=<token>
```

On connection, the server:
1. Authenticates the token
2. Sends a snapshot of **all current events** (batch of `upsert events` messages)
3. Sends a snapshot of **all current best prices** (batch of `upsert sptmkt` messages)
4. Streams real-time updates as prices and event states change

The stream is **server-to-client only**. Any client-sent message returns:

```json
["response", {"status": "error", "code": "invalid_input"}]
```

### Frame format

Each WebSocket frame is a JSON object:

```json
{
  "ts": 1765982942.805292,
  "data": [<message1>, <message2>, ...]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `ts` | float | Unix timestamp (seconds, microsecond precision) |
| `data` | array | One or more messages batched together (up to ~4 KB per frame) |

---

## Message types

Two data tables: **events** (sport event metadata) and **sptmkt** (best prices).
Each has an `upsert` and `delete` action.

### Event upsert

Sent when a new event appears or its metadata/in-play status changes.

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
| `competition_id` | integer | Competition/league numeric ID |
| `competition_name` | string | Competition name |
| `competition_country` | string | Country code |
| `home` | string | Home team/player |
| `away` | string | Away team/player |
| `start_ts` | string or null | ISO 8601 start time |
| `ir` | boolean | `true` if currently in-play |
| `score` | `[int, int]` or null | `[home, away]` for in-play football; `null` otherwise |
| `ir_time` | `[string, int]` or null | `[period, minutes]` for in-play football; `null` otherwise |

**`ir_time` period values:**
- `"1h"` — first half
- `"2h"` — second half
- `"ht"` — half time

### Event delete

Sent when an event is removed (finished or cancelled).

```json
["delete", "events", ["fb", "2025-12-20,54321,12345"]]
```

### Price upsert

Sent when the best available price changes for a market selection.

```json
["upsert", "sptmkt", ["basket", "2025-12-17,73821,40727", "for,ah,h,-6"], {"price": 1.49}]
```

**Key:** `[sport_code, event_id, bet_type]`

| Field | Type | Description |
|-------|------|-------------|
| `price` | float | Best price in decimal odds, rounded to 3 decimal places |

### Price delete

Sent when no bookmaker offers a price for this market anymore.

```json
["delete", "sptmkt", ["basket", "2025-12-17,73821,40727", "for,ah,h,-6"]]
```

---

## Best price calculation

For each `(sport, event_id, bet_type)` combination, the server:

1. Collects all offers from sources in your `prices_bookies` set
2. For exchange-style sources, adjusts for commission:
   ```
   adjusted = price - (price - 1) * commission
   ```
3. Rounds down to 3 decimal places
4. Returns the **maximum** adjusted price

Commission rates are typically in the 1–2% range for exchange-style sources
and zero for others. The precise schedule is account-specific; contact
MagicMarkets for the values applicable to your subscription.

**Example:** a source offering 2.50 at 2% commission → adjusted =
`2.50 - (2.50 - 1) * 0.02` = **2.47**.

---

## Reconnection

- Each WebSocket connection is independent — no persistent subscriptions
- If disconnected, re-authenticate with `/v1/login` (or reuse the token if still
  valid within its 24h window) and open a new connection
- The server sends a fresh full snapshot on each new connection
- There is no message replay or sequence numbering

---

## Sport codes

| Code | Sport |
|------|-------|
| `fb` | Football (soccer) |
| `fb_ht` | Football - 1st half |
| `fb_et` | Football - extra time |
| `fb_corn` | Football - corners |
| `fb_corn_ht` | Football - corners 1st half |
| `fb_book` | Football - bookings |
| `basket` | Basketball |
| `basket_ht` | Basketball - 1st half |
| `tennis` | Tennis |
| `af` | American football |
| `ru` | Rugby union |
| `rl` | Rugby league |
| `ih` | Ice hockey |
| `baseball` | Baseball |
| `hand` | Handball |
| `darts` | Darts |
| `mma` | MMA |
| `boxing` | Boxing |
| `snooker` | Snooker |
| `arf` | Australian rules football |
| `volley` | Volleyball |

---

## Sources

Each subscription includes a set of opaque source identifiers delivered in the
`prices_bookies` field of the login response. Sources fall into two broad
categories:

- **Exchange-style** — commission adjustment applied before ranking
- **Sharp books / fixed-odds books** — no commission adjustment

Treat the source codes as opaque identifiers. Do not hard-code them into your
application logic since the set can change over time and varies by account.
If you need the current list of sources, use the `prices_bookies` returned by
`/v1/login` or `/v1/config`. For commission values and source inventory
applicable to your account, contact MagicMarkets.

---

## Bet types

Canonical comma-separated identifiers:

| Pattern | Meaning |
|---------|---------|
| `for,ah,h,-6` | Asian handicap, home -6 |
| `for,ah,a,6` | Asian handicap, away +6 |
| `for,ou,o,42` | Over/under, over 42 |
| `for,ou,u,42` | Over/under, under 42 |
| `for,1x2,1` | Match result, home win |
| `for,1x2,x` | Match result, draw |
| `for,1x2,2` | Match result, away win |

The exact set of bet types varies by sport and market availability. The
`bet_type` string is used as a key in both the Datafeed API (as the third
element of the `sptmkt` key) and the Trading API (in betslip and order data).
