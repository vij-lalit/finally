# Market Data Interface — Unified Python API

This document describes the unified market data abstraction used throughout the FinAlly backend. All downstream code (portfolio valuation, SSE streaming, trade execution) reads from `PriceCache` and never touches a data source directly.

## Design

```
MarketDataSource (ABC)          ← contract / interface
├── SimulatorDataSource         ← GBM simulation (default, no API key needed)
└── MassiveDataSource           ← Massive REST polling (when MASSIVE_API_KEY set)
        │
        ▼
   PriceCache                   ← single source of truth for all price reads
        │
        ├──→ SSE /api/stream/prices
        ├──→ GET /api/portfolio  (current valuations)
        └──→ POST /api/portfolio/trade  (fill price)
```

The factory function `create_market_data_source()` selects the implementation at startup based on the `MASSIVE_API_KEY` environment variable. No other code in the project should branch on this variable.

---

## Module: `backend/app/market/`

| File | Responsibility |
|------|---------------|
| `models.py` | `PriceUpdate` — immutable dataclass for one price tick |
| `interface.py` | `MarketDataSource` — abstract base class |
| `cache.py` | `PriceCache` — thread-safe in-memory price store |
| `simulator.py` | `GBMSimulator` + `SimulatorDataSource` |
| `massive_client.py` | `MassiveDataSource` — REST polling via `massive` SDK |
| `seed_prices.py` | Starting prices and GBM parameters for default tickers |
| `factory.py` | `create_market_data_source()` — selects implementation |
| `stream.py` | SSE router factory (reads from `PriceCache`) |

---

## `PriceUpdate` — the data model

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds (float)

    # Computed properties (no storage):
    change: float             # price - previous_price
    change_percent: float     # (change / previous_price) * 100
    direction: str            # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...   # safe for JSON / SSE
```

Immutable and hashable. Safe to pass across threads. `to_dict()` is the canonical serialization used by SSE events and API responses.

---

## `MarketDataSource` — abstract interface

```python
class MarketDataSource(ABC):

    async def start(self, tickers: list[str]) -> None:
        """Begin producing updates. Call once at app startup."""

    async def stop(self) -> None:
        """Shut down background task. Call at app shutdown. Safe to call multiple times."""

    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. Takes effect on next update cycle."""

    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also evicts it from PriceCache."""

    def get_tickers(self) -> list[str]:
        """Return currently tracked tickers."""
```

All methods are `async` to accommodate both the asyncio-native simulator and the thread-dispatched Massive client uniformly. `get_tickers()` is synchronous because it never blocks.

---

## `PriceCache` — thread-safe price store

```python
class PriceCache:

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> None:
        """Write a new price. Previous price is the last known price (or same price on first update)."""

    def get(self, ticker: str) -> PriceUpdate | None:
        """Latest PriceUpdate for a ticker, or None if unknown."""

    def get_price(self, ticker: str) -> float | None:
        """Convenience: latest price float, or None."""

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all known tickers. Safe to iterate — returns a copy."""

    def remove(self, ticker: str) -> None:
        """Evict a ticker from the cache."""

    @property
    def version(self) -> int:
        """Monotonically increasing counter. Increments on every update.
        SSE endpoint polls this to detect changes without busy-waiting."""
```

Internally uses a `threading.Lock` for all mutations. The `version` counter is how the SSE endpoint knows when to push a new event — it compares `cache.version` against the last-sent version without holding the lock.

---

## `create_market_data_source()` — factory

```python
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)
# source is SimulatorDataSource if MASSIVE_API_KEY is unset
# source is MassiveDataSource if MASSIVE_API_KEY is set and non-empty
```

Selection logic (in `factory.py`):

```python
api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

if api_key:
    return MassiveDataSource(api_key=api_key, price_cache=cache)
else:
    return SimulatorDataSource(price_cache=cache)
```

---

## Application Lifecycle

Wire into FastAPI lifespan events:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source
from app.db import get_watchlist_tickers   # reads watchlist from SQLite

DEFAULT_TICKERS = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"]

price_cache = PriceCache()
market_source = create_market_data_source(price_cache)

@asynccontextmanager
async def lifespan(app: FastAPI):
    tickers = get_watchlist_tickers() or DEFAULT_TICKERS
    await market_source.start(tickers)
    yield
    await market_source.stop()

app = FastAPI(lifespan=lifespan)
```

---

## Reading Prices (downstream code)

```python
# Single ticker — portfolio valuation, trade fill price
update = price_cache.get("AAPL")       # PriceUpdate | None
price  = price_cache.get_price("AAPL") # float | None

# All tickers — watchlist API endpoint
all_prices = price_cache.get_all()     # dict[str, PriceUpdate]

# In a trade execution handler:
fill_price = price_cache.get_price(ticker)
if fill_price is None:
    raise HTTPException(400, f"No price available for {ticker}")
```

---

## Dynamic Watchlist Changes

When the user adds or removes a ticker via the API, propagate to both the database and the market source:

```python
# POST /api/watchlist
async def add_to_watchlist(ticker: str):
    db_add_ticker(ticker)                  # persist to SQLite
    await market_source.add_ticker(ticker) # start tracking price

# DELETE /api/watchlist/{ticker}
async def remove_from_watchlist(ticker: str):
    db_remove_ticker(ticker)                  # remove from SQLite
    await market_source.remove_ticker(ticker) # stops tracking + evicts cache
```

`add_ticker` and `remove_ticker` are no-ops if the ticker is already present/absent — safe to call multiple times.

---

## SSE Streaming

The SSE endpoint in `stream.py` uses `cache.version` for efficient change detection:

```python
async def event_generator():
    last_version = -1
    while True:
        current_version = cache.version
        if current_version != last_version:
            prices = cache.get_all()
            for update in prices.values():
                yield f"data: {json.dumps(update.to_dict())}\n\n"
            last_version = current_version
        await asyncio.sleep(0.1)   # 100ms poll — well within SSE latency budget
```

This approach means:
- No update → no SSE event (no spurious traffic)
- Any update to any ticker → push all current prices in one batch
- Simulator and Massive share identical SSE behavior — the endpoint is source-agnostic

---

## Environment Variables

| Variable | Effect |
|----------|--------|
| `MASSIVE_API_KEY` (unset or empty) | Uses `SimulatorDataSource` |
| `MASSIVE_API_KEY=<key>` | Uses `MassiveDataSource` with 15s poll interval |

The `MassiveDataSource` constructor accepts an optional `poll_interval` float (seconds) for tuning on paid tiers. The factory always passes the default (15s).
