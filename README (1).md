# Stryde

**An end-to-end algorithmic trading system for crypto scalping on the 15-minute timeframe.**

Stryde combines a custom multi-signal TradingView indicator, a real-time prediction market dashboard, and a webhook-based execution layer into a cohesive, automated trading pipeline. Built entirely from scratch as a personal project.

---

## What This Project Demonstrates

- **System design** — architecting multiple independent components that communicate through a shared data contract (JSON webhooks)
- **Financial domain knowledge** — applied understanding of technical analysis, order-flow theory, risk management, and market microstructure
- **API integration** — RSA-PSS authenticated requests to the Kalshi prediction market API; live BTC price feed from Coinbase
- **Real-time data** — Python HTTP server with polling architecture serving a live browser dashboard at 15-second intervals
- **Pine Script (TradingView)** — custom indicator language used to build, score, and alert on multi-condition trading signals
- **Automation** — JSON-formatted webhook alerts structured for direct ingestion by TradersPost for brokerage execution on Webull

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        STRYDE SYSTEM                        │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   CSA-15     │    │   LSD-15     │    │   Kalshi     │  │
│  │  TradingView │    │  TradingView │    │  Dashboard   │  │
│  │  Pine Script │    │  Pine Script │    │  Python/HTTP │  │
│  │              │    │              │    │              │  │
│  │ 8-layer      │    │ Liquidity    │    │ Live BTC     │  │
│  │ momentum     │    │ sweep        │    │ price +      │  │
│  │ confirmation │    │ detection    │    │ Kalshi odds  │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                  │                   │           │
│         └──────────┬────────┘                  │           │
│                    ▼                            ▼           │
│             JSON Webhook Alert          Probability         │
│             (TradersPost)               Confirmation        │
│                    │                                        │
│                    ▼                                        │
│             Webull Execution                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Components

### CSA-15 · CryptoScalper Alerts
`/CSA_15/CSA_15.pine`

A Pine Script v5 indicator that scores every 15-minute bar across 8 independent technical conditions. A BUY or SELL signal fires only when 6 or more conditions align simultaneously — designed to eliminate low-conviction setups and reduce false entries.

**Conditions scored:**

| # | Layer | Logic |
|---|-------|-------|
| 1 | EMA Cross | Fast EMA (9) vs Slow EMA (21) |
| 2 | Trend Filter | Price position relative to EMA 100 |
| 3 | RSI Range | RSI within directional momentum zone |
| 4 | Stochastic RSI | K/D cross with overbought/oversold filter |
| 5 | Volume Surge | Volume exceeds 1.2× its 20-bar average |
| 6 | Squeeze Momentum | Lazybear momentum direction and acceleration |
| 7 | Heikin Ashi | Calculated candle body direction |
| 8 | Squeeze Release | Bollinger Bands expanded beyond Keltner Channels |

**Key technical decisions:**
- All signals use `barstate.isconfirmed` — no repainting on historical bars
- ATR-based TP and SL levels rendered directly on chart (default 2.5× / 1.5×)
- Alert payloads are JSON-structured for webhook automation
- Live HUD table renders current condition scores on every bar

---

### LSD-15 · Liquidity Sweep Detector
`/LSD_15/LSD_15.pine`

A second Pine Script indicator built on order-flow theory. Detects liquidity grabs — moments when price briefly wicks beyond a recent swing high or low to trigger retail stop-loss orders, then reverses sharply. These reversals mark institutional entry points.

Detection requires all of the following on a confirmed bar:
1. Wick extends beyond the swing level by a configurable threshold
2. Candle closes back inside the prior range
3. Candle body is directionally opposed to the sweep
4. Reversal strength meets minimum threshold
5. Volume confirmation (optional, default on)
6. EMA trend alignment (optional, default on)
7. RSI not exhausted in sweep direction (optional, default on)

A **Confluence ★** tier fires when a standard sweep is followed by three consecutive bars confirming the reversal — indicating sustained directional order flow.

**Used alongside CSA-15:** LSD-15 handles entry timing; CSA-15 handles directional bias. The highest-conviction setup requires both to align simultaneously.

---

### Kalshi Dashboard
`/dashboard/stryde_server.py`

A Python HTTP server that aggregates live market data from two external APIs and serves it as a local browser dashboard, updating every 15 seconds.

**Data sources:**
- **Coinbase API** — live BTC/USD spot price
- **Kalshi Elections API** — real-time YES/NO odds on the `KXBTC15M` market (will BTC be higher in the next 15 minutes?)

**Technical implementation:**
- RSA-PSS request signing using the `cryptography` library for Kalshi API authentication
- Single-file Python HTTP server (`http.server.BaseHTTPRequestHandler`) with JSON endpoints at `/btc` and `/ka`
- Frontend polling via `setInterval` with inline HTML/CSS/JS served from the Python process
- No external dependencies beyond the Python standard library and `cryptography`

**Signal interpretation:** Kalshi odds above 60% YES in the same direction as a CSA-15 + LSD-15 signal represents the highest-conviction entry the system can produce.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Signal detection | Pine Script v5 (TradingView) |
| Dashboard backend | Python 3.9 · `http.server` · `cryptography` |
| Dashboard frontend | Vanilla HTML/CSS/JS |
| External APIs | Coinbase REST · Kalshi Elections REST (RSA-PSS auth) |
| Execution bridge | TradersPost (webhook → broker) |
| Brokerage | Webull |

---

## Project Background

This system was designed and built to solve a specific personal problem: making higher-conviction scalping decisions on crypto in the 15-minute timeframe without relying on any single indicator or data source. Each component addresses a distinct layer of uncertainty — trend confirmation (CSA-15), entry timing (LSD-15), and macro probability (Kalshi) — and the three layers are designed to be used together.

The project involved working through real API authentication challenges (RSA-PSS key signing, endpoint migration), designing a no-dependency Python server architecture, and writing production-quality Pine Script with full input documentation, no-repaint guarantees, and webhook-ready alert payloads.

---

## Repository Structure

```
stryde/
├── README.md                  ← you are here
├── CSA_15/
│   ├── CSA_15.pine            ← CryptoScalper Alerts indicator
│   └── README.md              ← full documentation
├── LSD_15/
│   ├── LSD_15.pine            ← Liquidity Sweep Detector indicator
│   └── README.md              ← full documentation
└── dashboard/
    └── stryde_server.py       ← Kalshi + Coinbase live dashboard
```

---

## License

Free to use, modify, and build on.
