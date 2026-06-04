# Market Data Design Review

Date: 2026-06-04
Scope: Review of planning docs and current market data implementation

## Findings

### 1. High - Free-tier Massive support is documented but not implemented

- The design acknowledges that current code only parses `lastTrade.price`, which can be absent on Free tier.
- Current implementation still reads only `last_trade` fields and skips snapshots when unavailable.
- Project docs describe Free tier as acceptable and document fallback expectations.

References:
- [planning/MARKET_DATA_DESIGN.md#L689](planning/MARKET_DATA_DESIGN.md#L689)
- [backend/app/market/massive_client.py#L101](backend/app/market/massive_client.py#L101)
- [planning/PLAN.md#L163](planning/PLAN.md#L163)
- [planning/MASSIVE_API.md#L115](planning/MASSIVE_API.md#L115)
- [planning/MASSIVE_API.md#L275](planning/MASSIVE_API.md#L275)

Impact:
- Users with valid Free-tier keys may receive no cached prices in real-data mode.

### 2. Medium - Positive-definite guarantee is overstated

- The design states that off-diagonal values in (0,1) with unit diagonal guarantees Cholesky success.
- That condition is not sufficient in general; future correlation changes can still create a non-positive-definite matrix.

Reference:
- [planning/MARKET_DATA_DESIGN.md#L327](planning/MARKET_DATA_DESIGN.md#L327)

Impact:
- Potential runtime failures if correlation assumptions evolve without validation.

### 3. Low - Router-global claim conflicts with implementation

- The design says the stream router factory avoids globals.
- The implementation uses a module-level router object.

References:
- [planning/MARKET_DATA_DESIGN.md#L735](planning/MARKET_DATA_DESIGN.md#L735)
- [backend/app/market/stream.py#L17](backend/app/market/stream.py#L17)

Impact:
- Potential duplicate route registration confusion if the factory is called multiple times in tests or multi-app patterns.

## Validation

- Market data test suite passes: 73 passed.
- Command used: cd backend && uv run --extra dev pytest tests/market -q

## Open Question

- Should Massive Free-tier be fully supported in this milestone, or is paid-tier-only behavior acceptable for now?
