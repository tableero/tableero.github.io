---
layout: post
title: "Building an Event-Driven Trading Bot: Architecture Overview"
date: 2026-03-30
---

This post walks through the architecture of a trading bot designed for quantitative research and live execution. The system covers the full lifecycle: data ingestion, strategy evaluation, backtesting, optimization, and live order execution across multiple brokers.

## What It Does

The bot operates in two modes:

**Research mode** — Run backtests and parameter optimization against historical data. Define a strategy, pick symbols and date ranges, and get back 43+ performance metrics (Sharpe, Sortino, Calmar, Omega, VaR, CVaR, max drawdown, win rate, and more). A grid search optimizer iterates over parameter combinations and ranks results by any chosen metric.

**Live mode** — Connect to a broker, ingest real-time price bars, generate signals, size positions, check risk rules, and execute orders. The same code that runs backtests runs live — no translation layer, no divergence.

## Supported Markets and Brokers

The system is broker-agnostic. Domain models never hardcode broker defaults; every call requires explicit `broker` and `data_provider` arguments.

**Asset classes:** Equities, Futures, Forex, CFDs, Options (experimental).

**Broker adapters:** TradeStation, Interactive Brokers (IBKR), MetaTrader 5 (MT5), plus a Simulator for development without a live connection.

**Data providers:** Yahoo Finance, Alpha Vantage, Quantdle — each pluggable through a common interface.

## The Event-Driven Kernel

The core of the system is an event bus with breadth-first dispatch. Every action — a bar closing, a signal firing, an order being sized, risk being evaluated, a fill arriving — is an immutable event flowing through the bus.

The processing pipeline looks like this:

1. **BAR_CLOSED** — Portfolio marks positions to market
2. **SIGNAL_GENERATED** — Strategy evaluates indicators and emits a signal
3. **ORDER_SIZED** — Position sizer converts the signal into a concrete order quantity
4. **RISK_APPROVED / RISK_REJECTED** — Rule-based evaluator checks exposure, daily loss limits, margin
5. **ORDER_FILLED** — Execution adapter places the order; portfolio updates state

This is deterministic and single-threaded. A backtest produces an identical event sequence every run. The full event trail is persisted for audit and replay.

## Strategies

All strategies implement a common interface with configurable parameters, required bar counts, supported timeframes, and indicator extraction.

**Aberration** — Trend-following. Uses SMA + ATR breakout channels with configurable entry/exit thresholds.

**Golden Cross** — Moving average crossover. SMA(50) crosses SMA(200), configurable for long-only or bidirectional.

**TPS Connors** — Mean-reversion. SMA(200) trend filter + RSI(2) with four-level scale-in on pullbacks.

**Oracle strategies** — Perfect-foresight benchmarks that define theoretical upper bounds. Useful for calibrating expectations.

Adding a new strategy means implementing the interface and registering it — no changes to the kernel or execution layer.

## Position Sizing

The bot supports multiple sizing algorithms, composable together:

- **Turtle** — Volatility-normalized: `(Equity x Risk%) / (ATR x PointValue)`
- **Kelly Criterion** — Win-rate and payoff ratio based, capped by max units
- **Fixed** — Constant contract count
- **Scale-In / Scale-Out** — Gradual entry and exit to reduce market impact
- **Composite** — Combine sizers via min, max, or average for conservative or aggressive sizing

## Risk Management

A rule-based evaluator runs before every order:

- Maximum position count per symbol
- Maximum exposure per symbol (% of equity)
- Global exposure limit
- Daily loss limit
- Margin utilization threshold

If any rule fails, the order is rejected and a `risk_rejected` event is published. Every rejection is traceable.

## Financial Models

The system models real-world trading costs and mechanics:

**P&L:** Linear (equities, futures, CFDs) and Forex (with FX rate conversion).

**Margin:** Cash-only (equities), rate-based (CFDs/Forex), and futures-specific (fixed per-contract, long/short asymmetry, intraday discount).

**Carry:** Financing, tiered financing, dividend adjustments, cash interest — composable.

**Commissions:** Flat, per-contract, percentage, spread cost, tiered — composable.

## Backtesting Output

Every backtest produces:

- **Records CSV** — Bar-by-bar decision log with indicators, signals, fills, P&L
- **Trades CSV** — Closed trades with entry/exit prices, quantity, P&L
- **Indicators CSV** — Raw indicator values for verification
- **Full JSON** — Complete event log for replay and audit
- **HTML Report** — Equity curve, drawdown chart, quantstats performance report

The multi-strategy portfolio backtester supports shared equity across positions, so strategies compete for capital just like they would in production.

## Orchestration and Observability

Apache Airflow schedules data ingestion and strategy execution DAGs. Business logic lives in a service layer with zero Airflow imports — DAGs are thin wrappers.

Monitoring runs through OpenTelemetry feeding into Grafana dashboards. Metrics cover DAG runtime, backtest speed, bar latency, and container resource usage. Each broker adapter gets its own dashboard folder.

## Configuration

Everything is YAML-driven. A shared `app.yaml` defines defaults (sizer, commission model, data provider). Strategy-specific backtest configs specify equity, date ranges, symbols, and parameters. The optimizer config adds parameter ranges and a target metric.

Running a backtest or optimization is a single CLI command through a Docker wrapper script.

## Architecture Patterns

A few design decisions that shape the system:

- **Event-driven, not loop-based** — The event bus handles cascading effects naturally. Same code for backtest and live.
- **Single-writer cache** — Only the portfolio handler writes to state. No race conditions.
- **Protocol-based interfaces** — Strategies, sizers, and models use duck typing. Extend without modifying core.
- **Hexagonal architecture** — Business logic is fully decoupled from infrastructure (Airflow, brokers, databases).
- **Frozen dataclasses for events** — Immutable, safe to share, easy to serialize.
- **Registry pattern** — Strategies and instruments are discovered and loaded by name.

## What Comes Next

The roadmap includes an instrument catalog expansion, broker state recovery for crash resilience, reliability hardening, and a unified scaling framework for coordinated scale-in/out across multi-leg positions.

---

The goal is a system where research and production share the same execution path, every decision is auditable, and adding a new strategy or broker is a matter of implementing an interface — not rewiring the pipeline.
