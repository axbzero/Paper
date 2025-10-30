## AXB Zero Whitepaper: Accounts Service and Trading Engine

### Executive summary
This whitepaper describes the architecture, data flow, and operational model of our Accounts Service and low‑latency Trading Engine. It explains how authentication, rules/limits, pre‑trade risk, execution, positions, ledger, and real‑time metrics collaborate to deliver a capital‑safe, transparent trading experience. Small code examples are included to illustrate usage.

---

## System overview
- **Accounts Service**: Manages accounts, credentials, status, groups, rules, leverage/commission/swap, order‑size limits, ledger, and metrics. It exposes a secure HTTP API with strict auth, rate limits, and audit features.
- **Trading Engine**: Accepts orders, enforces pre‑trade risk, executes market/limit/stop orders, maintains positions, computes P&L, and emits events and metrics.
- **Market Data**: Real‑time WebSocket quotes and historical/time‑series REST endpoints used for pricing and analytics.
- **Web Client**: React/TypeScript trading UI that integrates with both Accounts and Engine.

High‑level flow:
1) User obtains a JWT via Accounts login.
2) Client loads account, rules, leverage/commission/swap, and order‑size limits.
3) Order is submitted to the Engine; risk gates evaluate in real‑time.
4) If accepted, execution updates positions and generates ledger/metrics updates.
5) UI reflects changes via polling/streaming plus REST reads.

---

## Accounts Service

### Responsibilities
- Account lifecycle: create, activate/suspend, group assignment.
- Credentials and sessions: JWT issuance/validation; tooling to generate credentials.
- Rules and limits: symbol allow/deny policy; leverage, commission, swap; order‑size limits.
- Ledger and metrics: immutable financial movements, closed positions, metrics snapshots.
- Ops & safety: auth, security headers, rate limits (Redis), request body caps, structured logs, and panic recovery.

### Security and middleware
- JWT validation (RS256 or JWKS) with issuer/audience checks and `tenant_id` enforcement.
- CORS with allow‑list, strict headers (X‑Content‑Type‑Options, X‑Frame‑Options, Referrer‑Policy, CSP), and OPTIONS handling.
- Route‑scoped rate limits with Redis fixed‑window counters; separate limits per path/method.
- Request body size limits globally and per route.
- Panic recovery middleware to respond 500 without server crash.

### Data model (selected)
- Accounts: id, tenant, status, base equity, settings.
- Rules: allow/deny by symbol and asset class; group/tenant overrides.
- Leverage: `max_leverage`, `maintenance_margin_pct` with per‑symbol/class overrides.
- Commission: `per_lot_usd` per symbol/class; charged per side.
- Swap: per‑annum financing rates with day‑count basis and triple‑Wednesday option.
- Order size limits: min/max quantity, notional caps by tenant/symbol.
- Positions (closed): realized P&L, commission, SL/TP audit fields.
- Ledger entries: immutable, categorized by reason (trade, fee, swap, transfer).

### Representative API surface
- Auth: `POST /api/login`
- Accounts: `GET/POST /api/accounts`, `GET /api/accounts/{id}`, `PATCH /api/accounts/{id}/status`
- Rules: `PUT /api/rules/tenant`, `PUT /api/rules/accounts/{id}`, `GET /api/rules/accounts/effective/{id}`
- Leverage: `GET/PUT /api/leverage/config`, `GET /api/leverage/effective`, `GET /api/leverage/tenant`
- Commission: `GET/PUT /api/commission/config`, `GET /api/commission/effective`
- Swap: `GET/PUT /api/swap/config`, `GET /api/swap/effective`
- Limits: `GET/PUT /api/order-limits`, `GET /api/order-limits/{id}`
- Groups: `GET/POST /api/groups`, `PATCH /api/groups/{id}`
- Async ops: `GET /api/accounts/requests/{id}` (status for async create)
- Health/metrics: `GET /healthz`, `GET /readyz`, `GET /metrics`

Notes:
- All endpoints expect valid JWT with tenant scope; rate limits vary per route.
- Idempotency keys are supported for async create to prevent duplicate work.

---

## Trading Engine

### Responsibilities
- Order acceptance: market, limit, stop; IOC/FOK/GTC semantics.
- Pre‑trade risk: account status, symbol allow/deny, trading hours, exposure and margin checks, notional/size caps, precision checks.
- Position management: netting, average price, realized/unrealized P&L; SL/TP carry‑forward.
- Persistence and events: orders/trades repository, balance/position events, and Prometheus‑style metrics.

### Risk and controls (high‑level)
- Account blocks (e.g., margin breach) and status checks; engine caches Accounts bearer for backend calls.
- Symbol policy evaluation against effective rules (tenant/account/group).
- Trading hours enforcement per symbol spec; precision enforcement for price/quantity increments.
- Order‑size and notional caps evaluated with latest quote prices; limits fetched via Accounts.
- Margin checks using USD‑normalized notionals and latest quotes; maintenance/initial thresholds from leverage config.

### API surface (representative)
- Orders: `POST /api/orders` (market/limit/stop), `PATCH /api/orders/cancel?account_id=...&order_id=...`
- Positions: `GET /api/positions?account_id=...`, `POST /api/positions/close` (by id or symbol/side/qty), `POST /api/positions/bulk-close`
- SL/TP update: `PATCH /api/positions/sltp`
- Balance/status sync from Accounts (webhooks): `POST /api/engine/balance/refresh`, `POST /api/engine/balance/set`, `POST /api/engine/status`
- Metrics: `GET /metrics` (Prometheus text), `GET /api/metrics` (JSON snapshot)

### Performance and correctness
- Concurrency control around shared account state; idempotency keys for safe retries.
- Price staleness checks for IOC/FOK; backpressure on queues; bounded memory.
- Deterministic accept/reject with reason codes for operator visibility.

---

## Market Data
- WebSocket: `/ws` for real‑time quotes; supports subscribe/unsubscribe per symbol.
- Historical: `/api/historical/{symbol}`; price change and time‑series APIs for analytics.
- Engine integrates last quote into size/notional and margin checks; staleness windows enforced for IOC/FOK.

---

## End‑to‑end sequence (market order)
1) Client sends `POST /api/orders` with account, symbol, side, type=market, qty, optional SL/TP.
2) Engine validates JWT/account match, symbol policy, trading hours, size/precision caps, and margin/exposure.
3) On accept, order fills at best bid/ask; position is netted/created; realized/unrealized P&L updated.
4) Engine emits events, increments metrics; Accounts‑facing actions (e.g., closed positions, ledger) are authorized using cached bearer.

---

## Small examples

### 1) Place a market order (TypeScript, client)
```ts
import { placeMarketOrder } from '../src/lib/engineClient';

// Ensure localStorage keys are set: active_account_id, tenant_bearer
await placeMarketOrder('EUR/USD', 'buy', 1.0, {
  stopLoss: 1.0750,
  takeProfit: 1.0900,
  tif: 'IOC',
});
```

### 2) Place a limit order (curl)
```bash
curl -X POST "$ENGINE_BASE/api/orders" \
  -H "Authorization: Bearer $TENANT_BEARER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "account_id": "acc_123",
    "symbol": "BTC/USD",
    "side": "sell",
    "type": "limit",
    "quantity": 0.2,
    "price": 70500,
    "time_in_force": "GTC"
  }'
```

### 3) Fetch account data (curl)
```bash
curl -H "Authorization: Bearer $TENANT_BEARER" \
  "$ACCOUNTS_BASE/api/accounts/acc_123"
```

### 4) Configure leverage (JSON payload)
```json
{
  "asset_class": "crypto",
  "symbol": "BTC/USD",
  "max_leverage": 20,
  "maintenance_margin_pct": 0.5
}
```

### 5) Minimal Go HTTP client for engine order
```go
type orderReq struct {
    AccountID   string  `json:"account_id"`
    Symbol      string  `json:"symbol"`
    Side        string  `json:"side"`
    Type        string  `json:"type"`
    Quantity    float64 `json:"quantity"`
    TimeInForce string  `json:"time_in_force"`
}

req := orderReq{AccountID: "acc_123", Symbol: "EUR/USD", Side: "buy", Type: "market", Quantity: 1, TimeInForce: "IOC"}
b, _ := json.Marshal(req)
http.Post(ENGINE_BASE+"/api/orders", "application/json", bytes.NewReader(b))
```

---

## Innovation highlights
- Rule‑as‑data: leverage, commission, swap, symbol permissions, and order‑size limits are configured data, hot‑reloadable without code changes.
- Deterministic risk: precise, explainable rejection reasons at the point of order entry.
- Metrics‑ledger coupling: consistent risk metrics with position/ledger events for near real‑time UI.
- Client safety interlocks: One‑Click and Bulk Trading flows are guarded by server‑side risk and client confirmations.
- Redis‑backed shared rate limits across layers for predictable load shaping.

---

## Operations and SRE posture
- Health/readiness endpoints; structured logs; Prometheus counters; alerting on rejections, margin warnings, queue depth, and latency.
- Backpressure on async queues and request body caps to maintain service quality.
- Disaster‑readiness: schema migrations, backups, restore drills, and gradual rollouts.

---

## Glossary
- Maintenance Margin: minimum equity required to keep a position open.
- Swap/Funding: periodic financing charge on open positions.
- Idempotency: repeated submission is applied once.
- IOC/FOK/GTC: time‑in‑force semantics for order execution and persistence.

---

### Conclusion
The platform delivers a safety‑first, account‑centric trading system coupling strict pre‑trade controls with a capable execution engine and a modern client. It emphasizes transparency, deterministic risk decisions, and operational robustness suitable for both retail and institutional deployments.


