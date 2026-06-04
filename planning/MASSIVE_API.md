# Massive API (formerly Polygon.io) — Reference

Polygon.io rebranded to **Massive** on October 30, 2025. All existing API keys and endpoints remain fully compatible. The new base URL is `api.massive.com`; the legacy `api.polygon.io` continues to work during the transition.

## Authentication

API key passed as a **Bearer token in the Authorization header** on every request:

```
Authorization: Bearer YOUR_API_KEY
```

Example with curl:
```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT"
```

> **Note:** The old Polygon.io-era `?apiKey=KEY` query parameter is a legacy pattern and no longer the correct approach after the Massive rebrand. The Python `RESTClient(api_key=...)` handles the Bearer header transparently — no change needed in SDK usage.

Get a key at [massive.com/dashboard/api-keys](https://massive.com/dashboard/api-keys).

## Rate Limits

| Plan | Requests/minute | Recommended poll interval |
|------|-----------------|--------------------------|
| Free | 5 | 15 s |
| Starter | Unlimited | 5–10 s |
| Developer+ | Unlimited | 2–5 s |

---

## Key Endpoints

### 1. Multi-Ticker Snapshot (primary endpoint for this project)

Fetches the latest trade, quote, and day bar for one or more tickers in a single request. This is the most efficient endpoint for a watchlist of 5–20 tickers.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,NVDA
Authorization: Bearer YOUR_API_KEY
```

**Query parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `tickers` | No | Comma-separated ticker list. Omit to get all 10,000+ tickers (very large). |
| `include_otc` | No | Include OTC securities. Default: `false`. |

**Response:**

```json
{
  "status": "OK",
  "count": 3,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChangePerc": 0.52,
      "todaysChange": 1.01,
      "updated": 1704067200000,
      "day": {
        "o": 193.00,
        "h": 195.50,
        "l": 192.25,
        "c": 194.50,
        "v": 55000000,
        "vw": 193.80
      },
      "min": {
        "av": 55000000,
        "t": 1704067140000,
        "o": 194.40,
        "h": 194.60,
        "l": 194.30,
        "c": 194.50,
        "v": 120000,
        "vw": 194.45
      },
      "prevDay": {
        "o": 192.00,
        "h": 193.80,
        "l": 191.50,
        "c": 193.49,
        "v": 50000000,
        "vw": 192.70
      },
      "lastTrade": {
        "p": 194.50,
        "s": 100,
        "x": 4,
        "t": 1704067200000000000
      },
      "lastQuote": {
        "P": 194.52,
        "S": 5,
        "p": 194.50,
        "s": 3,
        "t": 1704067200000000000
      },
      "fmv": 194.51
    }
  ]
}
```

**Field notes:**
- `lastTrade.p` — last trade price (what we use for intraday price)
- `lastTrade.t` — nanosecond Unix timestamp; divide by `1_000_000` for milliseconds or `1_000_000_000` for seconds
- `day.c` — today's session closing/current price — **use as fallback if `lastTrade` is absent**
- `prevDay.c` — previous day's close — use as fallback if both `lastTrade` and `day` are absent
- `todaysChangePerc` — percent change from previous close
- Free tier: data is 15-minute delayed; paid tiers: real-time
- **`lastTrade` and `lastQuote` are plan-gated** — only returned on plans that include trades/quotes. On Free tier these fields may be absent; always fall back to `day.c` then `prevDay.c`.

---

### 2. Single Ticker Snapshot

Same data as above but for one ticker. Use the multi-ticker endpoint instead when watching multiple tickers — it saves rate limit quota.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/AAPL
Authorization: Bearer YOUR_API_KEY
```

Response wraps the same ticker object under a `"ticker"` key (singular).

---

### 3. Daily OHLC (End-of-Day)

```
GET /v1/open-close/AAPL/2024-12-31?adjusted=true
Authorization: Bearer YOUR_API_KEY
```

**Response:**

```json
{
  "status": "OK",
  "from": "2024-12-31",
  "symbol": "AAPL",
  "open": 192.50,
  "high": 195.00,
  "low": 192.25,
  "close": 194.50,
  "volume": 55000000,
  "afterHours": 194.20,
  "preMarket": 193.80
}
```

Use this for historical close prices when building charts or calculating portfolio P&L vs. previous close.

---

### 4. Aggregates (OHLCV Bars)

```
GET /v2/aggs/ticker/AAPL/range/1/minute/2024-01-01/2024-01-31
Authorization: Bearer YOUR_API_KEY
```

Returns arrays of OHLCV bars at any timespan (second, minute, hour, day, week, month, quarter, year). Useful for chart history. Responses are paginated via `next_url`.

---

## Python SDK

The official Massive Python client (formerly the Polygon client):

```bash
pip install -U massive
```

### Multi-Ticker Snapshot (what the project uses)

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="YOUR_API_KEY")

# Fetch snapshots for a list of tickers
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "MSFT", "NVDA", "TSLA"],
)

for snap in snapshots:
    # lastTrade is plan-gated — absent on Free tier. Fall back to day.close.
    if snap.last_trade and snap.last_trade.price:
        price = snap.last_trade.price
        ts_seconds = snap.last_trade.timestamp / 1000.0
    elif snap.day and snap.day.close:
        price = snap.day.close
        ts_seconds = None
    else:
        continue  # no price data available for this ticker
    print(f"{snap.ticker}: ${price:.2f}")
```

### Single Ticker Trade

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"AAPL last trade: ${trade.results.p:.2f}")
```

### Aggregate Bars (with pagination)

```python
bars = []
for bar in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="minute",
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,
):
    bars.append(bar)
```

### Error Handling

```python
from massive.exceptions import AuthError, BadResponse

try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except AuthError:
    # 401 — invalid or expired API key
    logger.error("Massive: invalid API key")
except BadResponse as e:
    # 429 rate limit, 5xx server error, etc.
    logger.error("Massive API error: %s", e)
```

The `MassiveDataSource` in this project catches all exceptions and logs them; the poll loop retries on the next interval.

---

## Timestamp Conventions

Massive uses three different timestamp resolutions depending on endpoint:

| Field | Unit | Convert to seconds |
|-------|------|--------------------|
| `lastTrade.t` (REST snapshot) | Unix milliseconds | `/ 1000` |
| Trade tick `participant_timestamp` | Unix nanoseconds | `/ 1_000_000_000` |
| `updated` (snapshot top-level) | Unix milliseconds | `/ 1000` |

The project always stores seconds (float) in `PriceUpdate.timestamp`.

---

## Data Availability by Plan

| Feature | Free | Starter | Developer | Advanced/Business |
|---------|------|---------|-----------|-------------------|
| Snapshot (real-time) | 15 min delay | 15 min delay | 15 min delay | Real-time |
| `lastTrade` / `lastQuote` in snapshot | No | Depends on plan | Yes | Yes |
| Historical OHLCV | 2 years | 2 years | 2 years | Since 2003 |
| Tick data | No | No | Yes | Yes |
| Fair Market Value (`fmv`) | No | No | No | Business only |
| Rate limit | 5 req/min | Unlimited | Unlimited | Unlimited |

For this project (demo / course), the free tier with 15-minute delayed data is acceptable. Set `poll_interval=15.0` (the default in `MassiveDataSource`).
