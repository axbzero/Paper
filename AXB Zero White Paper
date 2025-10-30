Account Services and Trading Engine: Architecture and White Paper
Executive summary
This paper describes the design, capabilities, and operation of our institutional-grade Account Services and low‑latency Trading Engine. It covers the end‑to‑end lifecycle from authentication and market data, to pre‑trade risk, execution, ledgering, and real‑time portfolio metrics. We highlight innovations in account‑centric risk control, metrics‑ledger coupling, behavioral analytics, and robust reliability controls that protect users and the platform at scale.
System overview
Our platform is composed of independently deployable Go services communicating over HTTP/WebSocket and backed by durable storage. The core surfaces are:
Account Services: account state, credentials, balances, positions, groups, rules, leverage/commission/swap, order‑size limits, ledger, metrics.
Trading Engine: order routing, pre‑trade risk checks, position management, P&L/metrics updates, event fan‑out.
Market Data: real‑time ticks and historical time series delivered via WebSocket and APIs.
Users Service: identity, session, ratelimiting, email and status management.
Web Client: a TypeScript/React front‑end with advanced trading UI (e.g., Trading Panel, Bulk Trading, One‑Click trade flow).
High‑level data flow:
1) User authenticates and obtains JWT.
2) Client subscribes to market data and loads account/limits.
3) Order submission → real‑time risk checks → engine acceptance/reject.
4) Execution updates positions, ledger, and metrics in near real‑time.
5) Client UI reflects state via streaming updates and REST refresh.
Design principles
Safety first: enforce pre‑trade controls close to the point of order entry with explicit, explainable rejections.
Account‑centric: all activity is ledgered and traceable to accounts, groups, and rule sets.
Deterministic and observable: write‑ahead intent, idempotent operations, structured logs, and metrics for every critical step.
Low‑latency by construction: Go services, concurrency primitives, streaming I/O, and bounded work queues.
Configuration over code: leverage, commission, swap, symbols, and order limits are data‑driven and hot‑reloadable.
Defense in depth: authN/Z, ratelimiting, input validation, crypto utilities, and error recovery middleware.
Core components
Account Services
Accounts and Credentials: secure credential lifecycle, group membership, tenancy/scoping, JWT issuance, and session validation.
Balances and Ledger: double‑entry ledger with immutable audit trail; deposits/withdrawals; realized P&L; fees and swaps.
Positions: per‑symbol netting, average price, unrealized P&L, exposure, and margin consumption.
Limits and Rules:
Leverage tiers and cross/isolated margin parameters.
Commission schedules and swap (funding) rules.
Order size limits (min/max, notional caps) and symbol trading windows.
Group‑level rule sets to enforce firm policies.
Metrics: real‑time derived metrics (equity, margin usage, risk ratios, drawdown, velocity), exposed to UI and engine.
Supporting modules:
HTTP middleware for auth, security headers, structured logging, rate limiting (Redis‑backed), metrics, panic recovery.
Async worker for background tasks that do not belong on the request path (e.g., slow ledger exports).
Trading Engine
Pre‑trade risk:
Authentication and account status checks.
Symbol availability and session status checks.
Order parameter validation (side, type, size, price bands).
Exposure and leverage checks (initial/maintenance margin).
Per‑account/group order rate and notional caps.
Execution lifecycle:
Accept/reject with deterministic reason codes.
Position updates (netting, average price, realized/unrealized P&L).
Fee/commission accrual and swap application.
Metrics update and event fan‑out for UI refresh and monitoring.
Throughput and latency:
Concurrency control to prevent state races.
Idempotent processing to tolerate network retries and duplicate submissions.
Backpressure and bounded queues to protect downstream stores.
Market Data
Real‑time WebSocket stream for ticks, quotes, and status events.
Historical time series APIs for analytics, backfill, and charting.
Session/micro‑status broadcasting (e.g., trading halts) to interlock with pre‑trade checks.
Users Service
Identity management, email and status workflows.
Session handling, JWT validation, ratelimiting, and audit.
Web Client
Advanced trading UI with:
Trading Panel and Balance components for live risk context.
One‑Click Trade flow with confirmation modal and safety interlocks.
Bulk Trading tool for portfolio‑level operations under policy guardrails.
High‑performance charting and drawing tools for decision support.
Data model highlights
Accounts: identity, balances, margin mode, group membership, status flags.
Credentials: hashed secrets, rotation metadata, scopes/roles.
Positions: symbol, quantity, average price, realized/unrealized P&L, last update.
Ledger entries: immutable movements categorized by reason (trade, fee, swap, transfer).
Limits and Rules: leverage tiers, commission ladders, swap schedules, order size and notional caps, symbol permissions.
Groups: policy containers to apply rule sets and operational controls to multiple accounts.
Metrics: snapshot and time‑series derived values for risk and performance.
Authentication, authorization, and security
JWT‑based auth with signed tokens, rotated and bound to session context.
Role‑ and scope‑based authorization at the service boundary.
Rate limiting across IP, user, and endpoint dimensions (Redis token‑bucket/shared policies).
Security middleware: CORS, CSRF defenses, strict headers, input validation, and panic recovery with safe error surfaces.
Crypto utilities for hashing, signing, and secure random operations.
Least‑privilege data access patterns and audit logging for sensitive mutations.
Pre‑trade risk and controls
Parameter validation: order side/type, min/max size, price sanity bands.
Exposure control: max position size by symbol and notional; account equity‑based gates.
Margin control: initial and maintenance margin checks consistent with leverage tier.
Daily loss and drawdown limits, realized/unrealized P&L guardrails.
Velocity controls: orders/time and notional/time windows per account/group.
Symbol controls: trading session, halts, permissioning, and restricted lists.
Interlocks with UI:
One‑Click trades require confirmation modal and optional 2‑step verification.
Bulk Trading batches are pre‑validated; failing legs are isolated and reported.
Ledger and metrics pipeline
Every accepted trade generates ledger entries: principal, fee/commission, swap/funding as applicable.
Positions update triggers recalculation of realized/unrealized P&L.
Metrics‑Ledger coupling:
Equity, margin used/free, risk ratios, and velocity derived alongside ledger updates to minimize staleness.
Snapshots emitted for UI and monitoring.
Idempotency keys ensure duplicate submissions do not duplicate ledger effects.
Observability and reliability
Structured logs with correlation IDs across request, engine step, and ledger writes.
Metrics for latency, error rates, rejection reasons, resource saturation, and ratelimit activity.
Health checks and readiness probes for safe rollout and traffic shifting.
Bounded worker queues and backpressure to protect critical paths.
Graceful degradation: partial service impairment fails closed on trading, but keeps read APIs responsive.
Disaster recovery: database migrations with forward/backward compatibility; periodic backups and restore drills.
Client experience and UX safeguards
Real‑time account and position updates synchronized with market ticks.
Contextual risk indicators (margin usage, max size hints, price bands) in the Trading Panel.
Explicit rejection reasons surfaced in the UI for rapid operator response.
One‑Click mode gated by confirmations, account settings, and per‑symbol controls.
Bulk Trading provides previews, per‑leg validation, and transaction‑style reporting.
Behavioral analytics and coaching (optional module)
Derived signals: overtrading, revenge‑trading, emotional volatility, time‑of‑day performance, and drawdown recovery.
Portfolio analytics: concentration risk, correlation clusters, and symbol‑level performance drivers.
Coaching prompts: nudges and configurable guardrails informed by analytics without overriding hard risk limits.
Innovation highlights
Metrics‑Ledger fusion: synchronized derivations ensure the UI and engine operate on consistent, near real‑time risk numbers.
Rule‑as‑data risk layer: leverage, fees, swaps, symbol permissions, and size/notional caps are all data‑driven and hot‑reloadable.
Safety interlocks in the client: One‑Click and Bulk Trading are guarded by server‑side rules and client confirmations.
Deterministic rejections with reason codes: improves transparency, monitoring, and operator response.
Redis‑backed shared ratelimits: consistent enforcement across Users and Accounts boundaries for layer‑7 protection.
Idempotent processing end‑to‑end: safe retries without financial duplication.
API surface (representative)
Auth: session creation, token refresh, logout; token introspection for services.
Accounts: fetch account, balances, metrics; update settings and risk mode where permitted.
Positions: list/open/closed, per‑symbol exposure and P&L.
Limits/Rules: read effective limits, symbol permissions, leverage/commission/swap schedules.
Orders/Trades: submit, cancel, status; rejection reason catalog.
Ledger: list entries, export statements, balance reconciliation.
Market Data: subscribe to symbols (WebSocket), historical bars and ticks.
APIs honor JWT scopes, ratelimits, and return typed error payloads with reason codes for all rejections.
Performance characteristics
Go‑based services with careful memory management and concurrency control.
Low‑latency HTTP handlers and WebSocket streams for market and state updates.
Engine critical sections protected to ensure position and ledger integrity.
Bounded work pools and backpressure to avoid cascading failures.
Optimistic read paths with localized locking on mutating paths.
Deployment and operations
Independent service binaries for Accounts, Engine, Market Data, and Users.
Configurable via environment and config files; migration tooling for schema evolution.
Readiness and liveness probes; rolling deployments with zero‑downtime goals.
Centralized logging and metrics dashboards; alerting on risk‑critical signals and anomalies.
Compliance and auditability
Immutable ledger entries with full causal context (order → trade → fees/swaps).
Comprehensive audit logs for credential changes, rule updates, and sensitive operations.
Data retention policies and export capabilities to assist reporting.
Clear separation of duties between config/risk operators and trading users.
Roadmap
Per‑symbol dynamic margining responsive to volatility.
Advanced partial‑fill fee allocation and netting strategies.
Deeper AI‑driven coaching with opt‑in privacy controls.
Enhanced cross‑service tracing and SLO‑based autoscaling.
Glossary
Ledger: immutable record of financial movements attributed to accounts.
Swap/Funding: periodic financing charge on positions.
Maintenance Margin: minimum equity to keep a position open.
Pre‑trade Risk: checks performed before accepting an order.
Idempotency: property where repeating an operation yields the same effect once applied.
Conclusion
We built a safety‑first, account‑centric trading platform that couples a robust ledger, real‑time risk metrics, and deterministic pre‑trade controls with a low‑latency execution engine and a modern client UX. The result is a transparent, reliable system that scales—from single‑account operators to institutional deployments—while preserving capital safety and regulatory‑grade auditability.
Delivered a complete white paper covering architecture, data model, controls, security, risk, observability, and innovations.
Emphasized our unique strengths: metrics‑ledger coupling, rule‑as‑data controls, client safety interlocks, and behavioral analytics.
