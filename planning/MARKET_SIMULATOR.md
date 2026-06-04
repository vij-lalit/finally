# Market Simulator — Approach & Code Structure

The simulator generates realistic-looking stock price movements without any external API dependency. It uses Geometric Brownian Motion (GBM) with correlated moves across tickers, matching the statistical properties of real equity markets.

---

## Mathematical Foundation

### Geometric Brownian Motion

Each ticker's price evolves by:

```
S(t + dt) = S(t) * exp((μ - σ²/2) * dt + σ * √dt * Z)
```

Where:
- `S(t)` — current price
- `μ` (mu) — annualized drift (expected return, e.g. 0.05 = 5%/year)
- `σ` (sigma) — annualized volatility (e.g. 0.25 = 25%/year)
- `dt` — time step as a fraction of a trading year
- `Z` — standard normal random variable

### Why GBM?

GBM is the standard model underlying Black-Scholes. It produces:
- Prices that can never go negative (multiplicative, not additive)
- Log-normal price distributions (matches real market behavior)
- Mean-reverting drift with random noise
- Calibratable per-ticker volatility

### Time Step

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # ≈ 5,896,800 seconds
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ≈ 8.48e-8
```

At 500ms intervals, `dt` is tiny. Each tick produces a sub-cent move; meaningful price changes accumulate naturally over minutes — which is what you see on a real chart.

### Correlated Moves (Cholesky Decomposition)

Real stocks in the same sector move together. To simulate this:

1. Define a correlation matrix `Σ` for all `n` tickers
2. Compute Cholesky factor `L` such that `Σ = L * L^T`
3. Generate `n` independent standard normal draws `Z_indep`
4. Apply: `Z_correlated = L @ Z_indep`
5. Feed `Z_correlated[i]` as the `Z` term for ticker `i`

This produces multivariate normal draws with the target correlations. The matrix is rebuilt whenever tickers are added or removed (O(n²), but n < 50 in practice).

---

## Correlation Structure

Tickers are grouped by sector with empirically-inspired correlations:

```python
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # e.g., AAPL and MSFT move together
INTRA_FINANCE_CORR = 0.5   # e.g., JPM and V move together
CROSS_GROUP_CORR   = 0.3   # tech vs. finance
TSLA_CORR          = 0.3   # TSLA does its own thing (even vs. other tech)
```

Tickers not in any group (user-added dynamically) default to `CROSS_GROUP_CORR = 0.3` against everything.

---

## Random Shock Events

To make the simulation visually engaging:

```python
event_probability = 0.001  # 0.1% chance per tick per ticker

if random.random() < event_probability:
    shock = random.uniform(0.02, 0.05)   # 2–5% move
    sign  = random.choice([-1, 1])
    price *= (1 + shock * sign)
```

With 10 tickers at 2 ticks/sec, expect roughly one shock event every 50 seconds — frequent enough to keep the UI dynamic without being absurd.

---

## Per-Ticker Parameters

Starting prices and GBM parameters for the default watchlist:

```python
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

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # unknown/dynamically-added tickers
```

Sigma values are annualized. At 500ms ticks: `effective_move_per_tick ≈ sigma * sqrt(dt) ≈ sigma * 0.00029`. For TSLA (σ=0.50): ~0.015% per tick, ~1.5% over a 10,000-tick run (~1.4 hours). Realistic.

---

## Code Structure

### `GBMSimulator` — the math engine

Located in `backend/app/market/simulator.py`.

```
GBMSimulator
├── __init__(tickers, dt, event_probability)
│     └── calls _add_ticker_internal() for each + _rebuild_cholesky()
│
├── step() → dict[str, float]
│     ├── generate Z_independent  (np.random.standard_normal(n))
│     ├── apply Cholesky: Z_correlated = L @ Z_independent
│     ├── for each ticker:
│     │     ├── apply GBM formula
│     │     ├── maybe apply random shock (0.1% chance)
│     │     └── round to 2 decimal places
│     └── return {ticker: price}
│
├── add_ticker(ticker) → None
│     └── _add_ticker_internal() + _rebuild_cholesky()
│
├── remove_ticker(ticker) → None
│     └── remove from _tickers, _prices, _params + _rebuild_cholesky()
│
├── get_price(ticker) → float | None
│
└── get_tickers() → list[str]
```

`step()` is called every 500ms from `SimulatorDataSource._run_loop()`. It must stay fast — no I/O, no lock contention.

**Internal state:**

```python
self._tickers: list[str]              # ordered list (matches Cholesky column order)
self._prices: dict[str, float]        # current price for each ticker
self._params: dict[str, dict]         # {"sigma": float, "mu": float} per ticker
self._cholesky: np.ndarray | None     # L such that Σ = L @ L.T; None if n <= 1
```

### `SimulatorDataSource` — asyncio adapter

Wraps `GBMSimulator` in the `MarketDataSource` interface. Runs a background asyncio task.

```
SimulatorDataSource (MarketDataSource)
├── start(tickers)
│     ├── creates GBMSimulator
│     ├── seeds PriceCache with initial prices (so SSE has data immediately)
│     └── creates asyncio Task: _run_loop()
│
├── stop()
│     └── cancels the asyncio Task
│
├── add_ticker(ticker)
│     ├── sim.add_ticker(ticker)
│     └── seeds PriceCache with initial price
│
├── remove_ticker(ticker)
│     ├── sim.remove_ticker(ticker)
│     └── cache.remove(ticker)
│
└── _run_loop()   ← background task
      loop:
        prices = sim.step()
        for ticker, price in prices.items():
            cache.update(ticker, price)
        await asyncio.sleep(0.5)
```

The `asyncio.sleep(0.5)` call is non-blocking — other coroutines (request handlers, SSE streams) run freely between ticks.

---

## Cholesky Rebuild

Triggered by `add_ticker()` and `remove_ticker()`. The correlation matrix is always `n × n` and positive definite (all diagonal entries = 1, off-diagonal entries in (0, 1)):

```python
corr = np.eye(n)
for i in range(n):
    for j in range(i + 1, n):
        rho = _pairwise_correlation(tickers[i], tickers[j])
        corr[i, j] = rho
        corr[j, i] = rho

cholesky = np.linalg.cholesky(corr)
```

`np.linalg.cholesky` requires the matrix to be positive definite. The chosen correlation values (all between 0 and 1, with 1 on the diagonal) guarantee this. If n ≤ 1 there is no need for correlation, so `_cholesky` is set to `None` and Z is used directly.

---

## Testing the Simulator

Key invariants validated in `backend/tests/market/test_simulator.py`:

| Test | What it checks |
|------|---------------|
| `test_prices_always_positive` | GBM stays positive over 1000 steps |
| `test_prices_change_over_time` | Prices actually move (simulator is running) |
| `test_step_returns_all_tickers` | `step()` returns an entry for every ticker |
| `test_correlation_structure` | Correlated tickers have positively correlated returns over 10k steps |
| `test_gbm_math` | Manual GBM calculation matches `step()` output (with mocked random) |
| `test_add_remove_ticker` | Dynamic add/remove rebuilds Cholesky, prices correct |
| `test_cholesky_none_single_ticker` | n=1 case handled (no Cholesky needed) |

Run with:
```bash
cd backend && uv run pytest tests/market/test_simulator.py -v
```
