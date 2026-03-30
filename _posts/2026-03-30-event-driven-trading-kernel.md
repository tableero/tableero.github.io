---
layout: post
title: "Inside the Event-Driven Trading Kernel"
date: 2026-03-30
---

This is a deep dive into the event flow that powers the trading bot. The previous post covered the system at a high level. This one opens the kernel and traces what happens from the moment a price bar arrives to the moment an order is filled.

## The Event Bus

The core dispatcher is a single-threaded, FIFO queue with breadth-first ordering. Handlers subscribe to event types and optionally filter by symbol or timeframe.

When a handler processes an event, it may publish new events. Those go to the back of the same queue. The bus drains the entire queue before returning control to the kernel. This means all handlers for event N execute before any handler for event N+1.

There is no priority mechanism. Dispatch order is determined by subscription order, which is set at startup and stays fixed.

If a handler throws an exception, the kernel stops immediately. There is no retry or recovery logic. This is intentional — determinism requires absolute failure visibility.

## The Event Envelope

Every event is wrapped in an envelope that carries tracing metadata:

- **event_id** — Unique identifier for this event
- **event_type** — Discriminator: `bar_closed`, `signal_generated`, `order_sized`, etc.
- **payload** — Frozen dataclass with event-specific data
- **correlation_id** — Shared by all events in the same bar cycle
- **causation_id** — Points to the parent event that caused this one
- **metadata** — Strategy ID, version, idempotency key

The causation chain links every event to its origin: bar → signal → order → risk decision → fill. The correlation ID groups them into a single cycle for audit and replay.

## Event Types

Eight event types flow through the system:

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `bar_closed` | New price bar arrives | symbol, timeframe, OHLCV, timestamp |
| `signal_generated` | Strategy evaluates indicators | symbol, side (long/short/flat), signal_price, strength, stop_loss, take_profit |
| `order_sized` | Position sizer converts signal to quantity | symbol, side (buy/sell), quantity, order_type, order_ref |
| `risk_approved` | All risk rules pass | symbol, side, quantity, expected_fill_price, exposure before/after |
| `risk_rejected` | Any risk rule fails | symbol, side, quantity, reason |
| `order_filled` | Execution adapter confirms fill | symbol, side, quantity, fill_price, commission, slippage, realized P&L |
| `order_rejected` | Execution adapter rejects (e.g., duplicate) | symbol, side, quantity, reason |
| `order_requested` | Order submitted to broker (live mode) | symbol, side, quantity, expected_fill_price |

All payloads are frozen dataclasses. Once created, they cannot be modified. This makes them safe to pass between handlers and safe to serialize for audit.

## The Full Cascade

Here is the complete event flow for a single bar cycle:

```
BAR_CLOSED
  │
  ├─→ PortfolioHandler.handle_bar()
  │     Mark-to-market: recalculate unrealized P&L and equity
  │     Write updated account state to cache
  │
  └─→ SignalHandler.handle()
        Read latest bars from cache
        Call strategy.evaluate(bars, position_side)
        If signal is actionable → publish SIGNAL_GENERATED
          │
          └─→ SizingHandler.handle()
                Read account equity from cache
                Call sizer.size(request) → compute contract count
                If contracts > 0 → publish ORDER_SIZED
                  │
                  └─→ RiskEvaluator.handle()
                        Evaluate order against all rules (sequential, short-circuit)
                        │
                        ├─ Any rule fails → publish RISK_REJECTED (done)
                        │
                        └─ All rules pass → apply shadow reservation
                                             publish RISK_APPROVED
                          │
                          └─→ ExecutionHandler.handle()
                                Check order_ref for duplicates
                                Call adapter.execute()
                                publish ORDER_FILLED or ORDER_REJECTED
                                  │
                                  └─→ PortfolioHandler.handle_fill()
                                        Update position, cash, realized P&L
                                        Write to cache
                                          │
                                          └─→ FillNotificationHandler.handle()
                                                Call strategy.on_fill()
                                                Strategy updates internal state
```

Every step in this chain is driven by the event bus. No handler calls another handler directly. The bus enforces ordering and provides the audit trail.

## The Single Writer Principle

Only one handler writes to the shared cache: the PortfolioHandler. Every other handler reads from a read-only interface. This eliminates race conditions without locks — the kernel is single-threaded, and only one component mutates state.

The cache holds:

- **Account state** — Equity, cash, margin used, unrealized and realized P&L
- **Position snapshots** — Per symbol and strategy: side, quantity, average entry price, unrealized P&L
- **Bar history** — Last N bars per symbol (ring buffer)
- **Scale plans** — Locked entry and exit plans for scale-in/out sequences

## Shadow Reservations

When multiple orders flow through the same bar cycle, the risk evaluator faces a problem: the second order sees the same pre-execution snapshot as the first. If both orders pass risk checks against the same cash balance, the combined exposure could exceed limits.

The solution is shadow reservations. After approving an order, the risk evaluator creates a modified snapshot with the expected cash reduction and exposure increase. The next order in the same cycle evaluates against this shadow, not the real state. The shadow resets at the start of each bar.

This prevents time-of-check/time-of-use errors without waiting for fills to settle.

## The Bar Sequencer

Before a bar enters the event bus, the kernel checks it against the sequencer. The sequencer tracks the last timestamp seen per symbol-timeframe pair. If a bar arrives with a timestamp equal to or earlier than the last, it is rejected as stale.

This matters in live mode where network issues or data provider quirks can deliver duplicate or out-of-order bars. The sequencer ensures the kernel never processes the same bar twice.

## Backtest vs Live: Same Loop

The kernel has two entry points — `run_backtest()` and `run_live()` — but the internal loop is identical:

```
for bar in data_adapter.iter_bars():
    process_bar(bar)
```

In backtest mode, the adapter iterates through historical data in memory. In live mode, it blocks waiting for the next bar from the broker. The event bus, handlers, cache, and risk evaluator are the same code path.

The only difference at the end: backtest mode flushes collected events to the database in a single batch transaction. Events are collected in memory during the run (one append per event, no I/O) and written all at once. This preserves determinism — no database latency affects handler timing.

## Scale-In Flow

Scale-in distributes a position across multiple bars using predefined fractions. The sequence:

1. Strategy emits a signal with `scale_level=1` and `scale_fractions=[0.3, 0.4, 0.3]`
2. The sizer computes the total position size using the base algorithm (Turtle, Kelly, etc.)
3. The total is locked in the cache as a "plan"
4. Level 1 fills 30% of the plan
5. On subsequent bars, the strategy emits `scale_level=2`, then `scale_level=3`
6. Each level reads the locked plan and applies its fraction
7. The base sizer is never re-invoked after level 1

Locking the plan at level 1 ensures consistent sizing even if volatility (ATR) changes between levels. The total is decided once and distributed.

After a fill, the FillNotificationHandler calls `strategy.on_fill()` so the strategy can update its internal state — typically the last entry price, which subsequent scale-in conditions depend on.

## Scale-Out Flow

Scale-out mirrors scale-in for exits:

1. Strategy emits a flat signal with `exit_level=1` and `exit_fractions=[0.4, 0.35, 0.25]`
2. The sizer reads the original scale-in plan (or falls back to current position size)
3. An exit plan is locked in the cache
4. The scale-in plan is cleared (you cannot scale in while scaling out)
5. Level 1 closes 40% of the exit plan
6. Subsequent levels close their fractions
7. The final level closes all remaining shares (anti-dust: no fractional leftovers)

A special `exit_level=-1` triggers an emergency full close, clearing the exit plan immediately.

## Order Deduplication

The execution handler maintains a bounded set of submitted order references. Each order carries a deterministic reference string built from: strategy ID, symbol, timeframe, bar timestamp, side, percentage, and a run hash.

If the same order reference appears twice (possible in edge cases with reprocessing), the second submission is rejected with reason `duplicate_order_ref`. The set is bounded to 10,000 entries with FIFO eviction.

## Telemetry

When enabled, the kernel creates an OpenTelemetry span for each bar cycle, with child spans for each dispatched event. Metrics track:

- Events dispatched per type
- Signals generated per symbol and strategy
- Bar processing duration

These feed into Grafana dashboards for live monitoring. Each event carries its type and ID as span attributes, so traces can be correlated with the event audit trail.

## The Event Ledger

Separate from the event bus, an optional ledger records high-level portfolio events: fills, forced closes, margin calls, carry charges, contract rolls, and FX rate updates. The ledger is append-only and queryable by event type or portfolio entry.

This serves a different purpose than the event store. The store captures every kernel event for replay. The ledger captures portfolio-level lifecycle events for reporting and analysis.

## Design Invariants

A few rules the system enforces:

1. **Breadth-first dispatch** — All handlers for one event complete before the next event starts
2. **Single writer** — Only PortfolioHandler writes to cache
3. **Fail-fast** — Any handler exception stops the kernel immediately
4. **Deterministic references** — Order refs are computed from inputs, not random
5. **No TTL on state** — Positions and plans never expire in cache; only bars and prices can
6. **Shadow before fill** — Risk evaluation uses shadow snapshots, not stale pre-execution state
7. **Plan lock on level 1** — Scale-in total is decided once and never recomputed

These constraints exist so that the same bar sequence always produces the same event sequence, regardless of whether it runs in backtest or live mode.
