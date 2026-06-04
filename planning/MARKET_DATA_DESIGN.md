# Market Data Backend — Design Document

> **Status:** Implementation complete and tested. This document is the authoritative
> reference for the market data subsystem at `backend/app/market/`.

---

## 1. Overview

The market data backend is a self-contained subsystem that:

- Generates or fetches live equity prices from one of two interchangeable sources
- Maintains a thread-safe in-memory price cache as the single source of truth
- Streams price updates to all connected browser clients via Server-Sent Events (SSE)
- Supports dynamic watchlist changes (add/remove tickers) at runtime

The design uses the **Strategy pattern** — all downstream code is agnostic to whether prices come from the GBM simulator or the Massive REST API.

---

## 2. Module Map

```
backend/app/market/
├── __init__.py          Public API exports
├── models.py            PriceUpdate — immutable price snapshot dataclass
├── interface.py         MarketDataSource — abstract base class
├── cache.py             PriceCache — thread-safe in-memory store
├── seed_prices.py       Starting prices and GBM parameters
├── simulator.py         GBMSimulator + SimulatorDataSource
├── massive_client.py    MassiveDataSource — REST polling via massive SDK
├── factory.py           create_market_data_source() — env-driven factory
└── stream.py            create_stream_router() — FastAPI SSE endpoint
```

---

## 3. Architecture Diagram

```
                          MASSIVE_API_KEY env var
                                  │
                         ┌────────▼────────┐
                         │     factory.py   │
                         │ create_market_   │
                         │ data_source()    │
                         └────────┬────────┘
                    unset/empty   │   set + non-empty
                 ┌───────────────┘└───────────────────┐
                 ▼                                     ▼
    ┌─────────────────────┐              ┌──────────────────────┐
    │  SimulatorDataSource│              │   MassiveDataSource  │
    │  (GBM, asyncio task)│              │  (REST poll, 15s)    │
    └──────────┬──────────┘              └──────────┬───────────┘
               │  cache.update()                    │  cache.update()
               └─────────────────┬──────────────────┘
                                  ▼
                        ┌─────────────────┐
                        │   PriceCache    │  ← single source of truth
                        │  (thread-safe)  │
                        └────────┬────────┘
               ┌─────────────────┼──────────────────┐
               ▼                 ▼                   ▼
    GET /api/stream/prices   GET /api/portfolio   POST /api/portfolio/trade
    (SSE EventSource)        (valuations)         (fill price)
```

---

## 4. Data Model

### `PriceUpdate` — `backend/app/market/models.py`

The single data type that flows through the entire system. Frozen dataclass — safe to share across threads without copying.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds (float)

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Key properties:**
- `change` and `change_percent` are computed — not stored. They're derived from `price` and `previous_price` on access.
- `direction` drives the frontend flash animation: green for `"up"`, red for `"down"`.
- `to_dict()` is the canonical serialization for SSE events and API responses.
- On first update for a ticker, `previous_price == price` so `direction == "flat"` (no spurious flash).

**SSE wire format** (one entry per ticker per event):

```json
{
  "AAPL": {
    "ticker": "AAPL",
    "price": 191.23,
    "previous_price": 191.10,
    "timestamp": 1704067200.123,
    "change": 0.13,
    "change_percent": 0.068,
    "direction": "up"
  },
  "MSFT": { ... }
}
```

---

## 5. The Unified Interface

### `MarketDataSource` — `backend/app/market/interface.py`

All data sources implement this ABC. No downstream code imports `SimulatorDataSource` or `MassiveDataSource` directly — it always uses this interface via the factory.

```python
class MarketDataSource(ABC):

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Call once at app startup."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker and evict it from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return currently tracked tickers. Synchronous — never blocks."""
```

**Design notes:**
- All mutation methods are `async` — the simulator uses asyncio, Massive uses `asyncio.to_thread`. The interface must accommodate both.
- `get_tickers()` is sync because it only reads in-memory state.
- `add_ticker` / `remove_ticker` are idempotent — safe to call for already-present/absent tickers.

---

## 6. Price Cache

### `PriceCache` — `backend/app/market/cache.py`

The cache is the only place downstream code reads prices. It uses a `threading.Lock` because the simulator runs in an asyncio task but portfolio/trade handlers may access it from different thread contexts (FastAPI's thread pool for sync routes).

```python
class PriceCache:

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price   # flat on first update

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        with self._lock:
            return dict(self._prices)    # shallow copy — safe to iterate outside lock

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        return self._version             # no lock needed — int reads are atomic in CPython
```

**Version counter:** The `version` integer increments on every `update()` call. The SSE generator compares `cache.version` against `last_version` to know whether new data exists without holding the lock or busy-waiting. This is a lightweight polling signal, not a pub/sub mechanism.

**Reading prices in downstream routes:**

```python
# Trade execution — get fill price
fill_price = price_cache.get_price(ticker)
if fill_price is None:
    raise HTTPException(400, f"No price available for {ticker}")

# Portfolio valuation — iterate all positions
all_prices = price_cache.get_all()       # dict[str, PriceUpdate]
for pos in positions:
    update = all_prices.get(pos.ticker)
    if update:
        current_value = pos.quantity * update.price

# Watchlist endpoint — return prices alongside tickers
all_prices = price_cache.get_all()
return [
    {"ticker": t, "price": all_prices[t].price if t in all_prices else None}
    for t in watchlist_tickers
]
```

---

## 7. Market Simulator

### Mathematical Foundation

Each ticker's price evolves by **Geometric Brownian Motion (GBM)**:

```
S(t+dt) = S(t) * exp( (μ - σ²/2) * dt  +  σ * √dt * Z )
```

| Symbol | Meaning | Typical value |
|--------|---------|---------------|
| `S(t)` | Current price | e.g. $190 (AAPL) |
| `μ` | Annualized drift (expected return) | 0.05 (5%/year) |
| `σ` | Annualized volatility | 0.22 (AAPL), 0.50 (TSLA) |
| `dt` | Time step as fraction of a trading year | ~8.48e-8 (500ms) |
| `Z` | Standard normal random variable | drawn per tick |

**Why GBM?**
- Prices can never go negative (multiplicative update)
- Produces log-normal distributions (matches real market statistics)
- Calibratable per-ticker with real historical volatility

**Time step calculation:**

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800 seconds
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ≈ 8.48e-8
```

At 500ms ticks, `effective_move_per_tick ≈ σ * √dt`. For TSLA (σ=0.50):
`0.50 * √(8.48e-8) ≈ 0.00046` → ~0.046% per tick. Meaningul drift builds over minutes.

### Correlated Moves (Cholesky Decomposition)

Tech stocks move together. To simulate this:

1. Build an `n × n` correlation matrix `Σ` based on sector groups
2. Compute Cholesky factor `L` where `Σ = L @ L^T`
3. Each tick: draw `n` independent normals `Z_indep`, compute `Z_corr = L @ Z_indep`
4. Feed `Z_corr[i]` as the `Z` term for ticker `i`

```python
# In GBMSimulator._rebuild_cholesky()
corr = np.eye(n)
for i in range(n):
    for j in range(i + 1, n):
        rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
        corr[i, j] = rho
        corr[j, i] = rho

self._cholesky = np.linalg.cholesky(corr)  # requires positive definite matrix

# In step()
z_independent = np.random.standard_normal(n)
z_correlated = self._cholesky @ z_independent  # apply correlation
```

**Correlation structure:**

```python
# backend/app/market/seed_prices.py
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # AAPL and MSFT tend to move together
INTRA_FINANCE_CORR = 0.5   # JPM and V tend to move together
CROSS_GROUP_CORR   = 0.3   # tech vs. finance
TSLA_CORR          = 0.3   # TSLA does its own thing, even vs. other tech
```

Tickers not in any group (dynamically added by the user) get `CROSS_GROUP_CORR = 0.3` against everything. The matrix is always positive definite (all values in (0, 1), diagonal = 1), so `np.linalg.cholesky` will never fail.

When `n ≤ 1` there is nothing to correlate, so `_cholesky = None` and `Z_indep` is used directly (no matrix multiply).

### Random Shock Events

```python
# In GBMSimulator.step()
if random.random() < self._event_prob:   # 0.1% per tick per ticker
    shock_magnitude = random.uniform(0.02, 0.05)   # 2–5% move
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

With 10 tickers at 2 ticks/sec: expected event frequency ≈ `10 * 2 * 0.001 = 0.02/sec` → ~one event every 50 seconds. This keeps the UI dynamic without being absurd.

### Per-Ticker Seed Parameters

```python
# backend/app/market/seed_prices.py

SEED_PRICES = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00,
    "AMZN": 185.00, "TSLA": 250.00,  "NVDA": 800.00,
    "META": 500.00, "JPM":  195.00,  "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # high vol, modest drift
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # high vol, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM":  {"sigma": 0.18, "mu": 0.04},  # low vol (bank)
    "V":    {"sigma": 0.17, "mu": 0.04},  # low vol (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # for dynamically-added tickers
```

Tickers not in `SEED_PRICES` start at a random price in `[50.0, 300.0]`.

### `GBMSimulator` — Class Structure

```python
class GBMSimulator:

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ≈ 8.48e-8

    # Internal state
    _tickers: list[str]              # ordered list (matches Cholesky column order)
    _prices:  dict[str, float]       # current simulated price per ticker
    _params:  dict[str, dict]        # {"sigma": float, "mu": float} per ticker
    _cholesky: np.ndarray | None     # L factor; None if n <= 1

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Called every 500ms."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift     = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()          # must rebuild after adding

    def remove_ticker(self, ticker: str) -> None:
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()          # must rebuild after removing

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)
```

### `SimulatorDataSource` — asyncio Adapter

```python
class SimulatorDataSource(MarketDataSource):
    """Wraps GBMSimulator in the MarketDataSource interface."""

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)

        # Seed cache immediately — SSE has data before the first tick fires
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        """Core loop — runs forever until cancelled."""
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)   # 0.5s — non-blocking
```

**Lifecycle flow:**

```
app startup
    └─→ SimulatorDataSource.start(["AAPL", ...])
            ├─→ GBMSimulator.__init__()       # set up prices + Cholesky
            ├─→ seed PriceCache               # immediate data available
            └─→ asyncio.create_task(_run_loop)
                    └─→ [every 500ms]
                            ├─→ sim.step()    # update _prices dict
                            └─→ cache.update() for each ticker

user adds "PYPL"
    └─→ SimulatorDataSource.add_ticker("PYPL")
            ├─→ sim.add_ticker("PYPL")        # adds to _tickers, rebuilds Cholesky
            └─→ cache.update("PYPL", seed_price)

app shutdown
    └─→ SimulatorDataSource.stop()
            └─→ task.cancel()
```

---

## 8. Massive API Integration

### About Massive

Massive (formerly Polygon.io, rebranded October 2025) provides REST and WebSocket market data APIs. This project uses the multi-ticker snapshot endpoint via REST polling — simpler than WebSocket, compatible with all tiers.

**Base URL:** `https://api.massive.com`  
**Auth:** `Authorization: Bearer YOUR_API_KEY`  
**SDK:** `pip install -U massive`

### Rate Limits

| Plan | Requests/min | Recommended poll interval |
|------|-------------|--------------------------|
| Free | 5 | 15 s (default) |
| Starter | Unlimited | 5–10 s |
| Developer+ | Unlimited | 2–5 s |

### Primary Endpoint: Multi-Ticker Snapshot

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,NVDA
Authorization: Bearer YOUR_API_KEY
```

**Response shape:**

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
      "day":  { "o": 193.00, "h": 195.50, "l": 192.25, "c": 194.50, "v": 55000000, "vw": 193.80 },
      "min":  { "t": 1704067140000, "o": 194.40, "h": 194.60, "l": 194.30, "c": 194.50, "v": 120000 },
      "prevDay": { "c": 193.49, "v": 50000000 },
      "lastTrade": { "p": 194.50, "s": 100, "t": 1704067200000 },
      "lastQuote": { "P": 194.52, "p": 194.50 }
    }
  ]
}
```

**Price field priority (always fall back down the chain):**

| Priority | Field | Notes |
|----------|-------|-------|
| 1st | `lastTrade.p` | Most recent trade price — plan-gated, absent on Free |
| 2nd | `day.c` | Today's session current/closing price — available on all plans |
| 3rd | `prevDay.c` | Previous day close — last resort fallback |

**Timestamp handling:**

```python
# lastTrade.t is Unix milliseconds
ts_seconds = snap.last_trade.timestamp / 1000.0

# snapshot-level "updated" field is also milliseconds
ts_seconds = snap.updated / 1000.0
```

### Python SDK Usage

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from massive.exceptions import AuthError, BadResponse

client = RESTClient(api_key="YOUR_API_KEY")

# Multi-ticker snapshot (what this project uses)
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "MSFT", "NVDA", "TSLA"],
)

for snap in snapshots:
    # lastTrade absent on Free tier — fall back to day.close
    if snap.last_trade and snap.last_trade.price:
        price = snap.last_trade.price
        ts    = snap.last_trade.timestamp / 1000.0
    elif snap.day and snap.day.close:
        price = snap.day.close
        ts    = time.time()
    else:
        continue   # no usable price data
    print(f"{snap.ticker}: ${price:.2f}")
```

### `MassiveDataSource` Implementation

The Massive `RESTClient` is **synchronous** (blocking I/O). To avoid blocking the asyncio event loop, all API calls are dispatched with `asyncio.to_thread()`.

```python
class MassiveDataSource(MarketDataSource):

    def __init__(self, api_key: str, price_cache: PriceCache, poll_interval: float = 15.0):
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)

        # Immediate first poll — populate cache before serving requests
        await self._poll_once()

        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            # New ticker will appear in cache on next poll cycle

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # Run blocking SDK call in a thread pool
            snapshots = await asyncio.to_thread(self._fetch_snapshots)

            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    timestamp = snap.last_trade.timestamp / 1000.0
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                except (AttributeError, TypeError):
                    # lastTrade absent (Free tier) — log and skip
                    logger.warning("No lastTrade for %s", getattr(snap, "ticker", "???"))

        except AuthError:
            logger.error("Massive: invalid API key — check MASSIVE_API_KEY")
        except BadResponse as e:
            logger.error("Massive poll failed: %s", e)
        except Exception as e:
            logger.error("Massive poll unexpected error: %s", e)
        # Always swallow errors — retry on next interval

    def _fetch_snapshots(self) -> list:
        """Synchronous. Called via asyncio.to_thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

**Important:** The current implementation only parses `lastTrade.price`. On a Free tier API key (which returns `lastTrade=None`), all tickers will log warnings and no prices will be cached. For production Free-tier usage, the fallback chain (`day.close`, then `prevDay.close`) should be implemented:

```python
# Recommended fallback chain (enhancement for Free tier):
def _extract_price(snap) -> tuple[float, float] | None:
    """Returns (price, timestamp) or None."""
    try:
        if snap.last_trade and snap.last_trade.price:
            return snap.last_trade.price, snap.last_trade.timestamp / 1000.0
        if snap.day and snap.day.close:
            return snap.day.close, time.time()
        if snap.prev_day and snap.prev_day.close:
            return snap.prev_day.close, time.time()
    except AttributeError:
        pass
    return None
```

---

## 9. Factory

### `create_market_data_source()` — `backend/app/market/factory.py`

Reads `MASSIVE_API_KEY` from the environment and returns the correct implementation. This is the **only place in the entire codebase that branches on this variable.**

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        logger.info("Market data: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

The returned source is **unstarted**. The caller must `await source.start(tickers)`.

---

## 10. SSE Streaming

### `create_stream_router()` — `backend/app/market/stream.py`

A factory that creates a FastAPI router with the SSE endpoint wired to a specific `PriceCache`. The factory pattern avoids globals — the cache is injected.

```python
def create_stream_router(price_cache: PriceCache) -> APIRouter:
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",   # disable nginx buffering if behind proxy
            },
        )

    return router
```

### Event Generator

```python
async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:

    yield "retry: 1000\n\n"   # tell browser to reconnect after 1s if dropped

    last_version = -1

    while True:
        if await request.is_disconnected():
            break

        current_version = price_cache.version
        if current_version != last_version:
            last_version = current_version
            prices = price_cache.get_all()

            if prices:
                data = {ticker: update.to_dict() for ticker, update in prices.items()}
                yield f"data: {json.dumps(data)}\n\n"

        await asyncio.sleep(interval)
```

**Change detection via version counter:**
- No update → `current_version == last_version` → no SSE event → no network traffic
- Any update to any ticker → version increments → full snapshot pushed to all clients
- This batches all ticker updates from a single simulator tick into one SSE event

**SSE format recap:**
```
retry: 1000

data: {"AAPL": {"ticker":"AAPL","price":191.23,...}, "MSFT": {...}}

data: {"AAPL": {"ticker":"AAPL","price":191.25,...}, "MSFT": {...}}
```

Each `data:` line is followed by `\n\n` (double newline = event separator).

### Frontend connection (JavaScript):

```javascript
const es = new EventSource('/api/stream/prices');

es.onmessage = (event) => {
  const prices = JSON.parse(event.data);  // { "AAPL": PriceUpdate, ... }
  for (const [ticker, update] of Object.entries(prices)) {
    updateWatchlist(ticker, update);  // trigger flash animation
  }
};

es.onerror = () => {
  // EventSource auto-reconnects after `retry: 1000` ms
};
```

---

## 11. Application Lifecycle (FastAPI Integration)

Wire the market data subsystem into FastAPI's lifespan context manager:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router
from app.db import get_watchlist_tickers   # reads from SQLite watchlist table

DEFAULT_TICKERS = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"]

price_cache = PriceCache()
market_source = create_market_data_source(price_cache)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Load watchlist from DB (falls back to defaults on first run)
    tickers = get_watchlist_tickers() or DEFAULT_TICKERS
    await market_source.start(tickers)
    yield
    await market_source.stop()

app = FastAPI(lifespan=lifespan)
app.include_router(create_stream_router(price_cache))
```

### Watchlist Route Integration

When the user adds or removes a ticker via the API, **both** the database and the market source must be updated:

```python
# POST /api/watchlist
async def add_ticker_to_watchlist(ticker: str, db: Session = Depends(get_db)):
    ticker = ticker.upper().strip()
    db_add_watchlist_ticker(db, ticker)       # persist to SQLite
    await market_source.add_ticker(ticker)    # start generating prices immediately
    return {"ticker": ticker, "status": "added"}

# DELETE /api/watchlist/{ticker}
async def remove_ticker_from_watchlist(ticker: str, db: Session = Depends(get_db)):
    ticker = ticker.upper().strip()
    db_remove_watchlist_ticker(db, ticker)       # remove from SQLite
    await market_source.remove_ticker(ticker)    # stop tracking + evict from cache
    return {"ticker": ticker, "status": "removed"}
```

---

## 12. Public API (`__init__.py`)

Everything downstream code needs is exported from the package root:

```python
# backend/app/market/__init__.py
from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

Downstream imports only need:

```python
from app.market import PriceCache, create_market_data_source, create_stream_router
```

---

## 13. Environment Variables

| Variable | Value | Effect |
|----------|-------|--------|
| `MASSIVE_API_KEY` | unset or `""` | `SimulatorDataSource` selected |
| `MASSIVE_API_KEY` | non-empty string | `MassiveDataSource` selected, 15s poll |

No other code should read `MASSIVE_API_KEY` — only `factory.py`.

---

## 14. Testing

Test suite lives in `backend/tests/market/` — 73 tests, all passing, 84% overall coverage.

| File | What it tests |
|------|--------------|
| `test_models.py` | `PriceUpdate` immutability, `to_dict()`, computed properties, edge cases |
| `test_cache.py` | Thread safety, version counter, update/get/remove, first-update semantics |
| `test_simulator.py` | GBM math, prices always positive, Cholesky rebuild, correlation structure |
| `test_simulator_source.py` | `SimulatorDataSource` lifecycle, add/remove ticker, cache seeding |
| `test_factory.py` | Env-driven source selection, correct types returned |
| `test_massive.py` | `MassiveDataSource` polling, error handling (mocked), ticker management |

**Run all market data tests:**

```bash
cd backend
uv run pytest tests/market/ -v
```

**Run with coverage:**

```bash
uv run pytest tests/market/ --cov=app/market --cov-report=term-missing
```

**Key test patterns:**

```python
# test_cache.py — thread safety
def test_concurrent_updates_are_safe():
    cache = PriceCache()
    def writer():
        for _ in range(1000):
            cache.update("AAPL", random.uniform(100, 200))
    threads = [threading.Thread(target=writer) for _ in range(10)]
    for t in threads: t.start()
    for t in threads: t.join()
    assert cache.get("AAPL") is not None

# test_simulator.py — GBM stays positive
def test_prices_always_positive():
    sim = GBMSimulator(["AAPL"])
    for _ in range(1000):
        prices = sim.step()
        assert all(p > 0 for p in prices.values())

# test_simulator.py — correlation check
def test_correlation_structure():
    sim = GBMSimulator(["AAPL", "MSFT", "JPM"])
    aapl_returns, msft_returns, jpm_returns = [], [], []
    for _ in range(10000):
        p = sim.step()
        # collect log returns...
    corr_tech = np.corrcoef(aapl_returns, msft_returns)[0, 1]
    corr_cross = np.corrcoef(aapl_returns, jpm_returns)[0, 1]
    assert corr_tech > corr_cross   # tech stocks more correlated than cross-sector
```

---

## 15. Demo

A Rich terminal dashboard is available for manual testing:

```bash
cd backend
uv run market_data_demo.py
```

Displays all 10 tickers with live prices, sparklines, color-coded direction arrows, and a shock event log. Runs 60 seconds or until Ctrl+C.

---

## 16. Quick-Reference: Common Tasks

### Add this subsystem to a new FastAPI app

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

price_cache = PriceCache()
market_source = create_market_data_source(price_cache)  # reads MASSIVE_API_KEY

@asynccontextmanager
async def lifespan(app):
    await market_source.start(["AAPL", "MSFT", "TSLA"])
    yield
    await market_source.stop()

app = FastAPI(lifespan=lifespan)
app.include_router(create_stream_router(price_cache))
```

### Get a price in a route handler

```python
price = price_cache.get_price("AAPL")    # float | None
update = price_cache.get("AAPL")         # PriceUpdate | None
all_prices = price_cache.get_all()       # dict[str, PriceUpdate]
```

### Propagate watchlist change

```python
await market_source.add_ticker("PYPL")      # starts tracking + seeds cache
await market_source.remove_ticker("NFLX")   # stops tracking + evicts cache
```

### Switch to real data

Set `MASSIVE_API_KEY=your_key` in `.env`. No code changes needed — `create_market_data_source()` picks it up automatically.
