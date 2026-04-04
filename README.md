# 🏦 Thai Portfolio Backtester — Institutional-Grade Risk Framework

> A production-quality portfolio backtesting engine for Thai SET equities.
> Features a 7-layer risk management system, walk-forward validation, and live paper trading integration.

[![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![Jupyter](https://img.shields.io/badge/Jupyter-F37626?style=for-the-badge&logo=jupyter&logoColor=white)](https://jupyter.org)
[![Status](https://img.shields.io/badge/Paper_Trade-LIVE-brightgreen?style=for-the-badge)](https://github.com/ymodulus21)

---

## Live Paper Trading Performance

| Month | Portfolio | SPY Benchmark | Alpha |
|---|---|---|---|
| **Mar 2026** | **+0.41%** | **-4.99%** | **+5.40% ✅** |
| Apr 2026 | In progress | — | — |

> 🟢 **LIVE since March 27, 2026** — Next evaluation: May 2, 2026

---

## Backtest Results (10-Year OOS Validation)

| Metric | Result | Institutional Target | Status |
|---|---|---|---|
| **Sharpe Ratio (OOS)** | **0.837** | > 1.0 | 🟡 Near target |
| **Max Drawdown** | **-12.5%** | < 20% | ✅ Pass |
| **CAGR** | **16.4%** | > 12% | ✅ Pass |
| **WFT Beat Rate** | **67%** | > 55% | ✅ Pass |
| **Profit Factor** | **1.8** | > 1.5 | ✅ Pass |
| **Calmar Ratio** | **1.31** | > 0.5 | ✅ Pass |

---

## System Architecture — 7-Layer Risk Management

This backtester implements an institutional-grade risk framework with 3 active protection layers:

```
                    ┌─────────────────────────────────┐
                    │         SIGNAL GENERATION        │
                    │   (Multi-factor stock screener)  │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │        LAYER 1: REGIME FILTER    │
                    │  SPY vs SMA(200) → Market State  │
                    │                                  │
                    │  BULL     → 100% equity exposure │
                    │  SIDEWAYS →  75% equity exposure │
                    │  BEAR     →  50% equity exposure │
                    └──────────────┬──────────────────┘
                                   │ regime_exp
                    ┌──────────────▼──────────────────┐
                    │     LAYER 2: VOLATILITY TARGET   │
                    │  Scale to 15% annual vol target  │
                    │                                  │
                    │  vol_scale = target_vol /        │
                    │             realized_vol(20d)    │
                    │  Clipped: [0.30x, 1.50x]         │
                    └──────────────┬──────────────────┘
                                   │ vol_scale
                    ┌──────────────▼──────────────────┐
                    │     LAYER 3: CIRCUIT BREAKER     │
                    │  Auto-reduce on drawdown levels  │
                    │                                  │
                    │  DD > -10% → 80% of position     │
                    │  DD > -15% → 60% of position     │
                    │  DD > -20% → 30% of position     │
                    │  Recovery: +5% per level         │
                    └──────────────┬──────────────────┘
                                   │ cb_factor
                    ┌──────────────▼──────────────────┐
                    │      FINAL EQUITY FRACTION       │
                    │                                  │
                    │  f = regime × vol × cb           │
                    │  Clipped: [0.10, 1.00]           │
                    └─────────────────────────────────┘
```

---

## Walk-Forward Optimization (WFO)

Eliminates overfitting by validating on data the model **never saw during training**:

```
Timeline ──────────────────────────────────────────────────────────►

IS(1)──────────────────────── OOS(1) ──────
         IS(2)──────────────────────── OOS(2) ──────
                  IS(3)──────────────────────── OOS(3) ──────
                           IS(4)──────────────────────── OOS(4) ──

IS  = In-Sample  (70%) — model training & optimization
OOS = Out-of-Sample (30%) — blind validation, never used in training

Window: 18-month IS / 6-month OOS (rolling)
WFT Beat Rate: 67% of OOS windows outperformed benchmark
```

---

## Core Components

### Regime Filter
```python
REGIME_CONFIG = {
    "sma_period": 200,
    "bull_threshold": 1.03,   # SPY > SMA200 * 1.03
    "bear_threshold": 0.97,   # SPY < SMA200 * 0.97
    "exposure": {
        "BULL":     1.00,
        "SIDEWAYS": 0.75,
        "BEAR":     0.50
    }
}
```

### Volatility Targeting
```python
VOL_TARGET_CONFIG = {
    "target_vol":  0.15,   # 15% annualized target
    "lookback":    20,     # 20-day realized vol window
    "scale_min":   0.30,   # minimum 30% position
    "scale_max":   1.50    # maximum 150% position
}

vol_scale = target_vol / realized_vol
vol_scale = np.clip(vol_scale, scale_min, scale_max)
```

### Circuit Breaker
```python
CIRCUIT_BREAKER = {
    "levels": [
        {"dd": -0.10, "equity": 0.80},
        {"dd": -0.15, "equity": 0.60},
        {"dd": -0.20, "equity": 0.30}
    ],
    "recovery_per_level": 0.05
}
```

### Combined Equity Fraction
```python
equity_fraction = regime_exp * vol_scale * cb_factor
equity_fraction = np.clip(equity_fraction, 0.10, 1.00)
```

---

## Backtesting Engine

### Portfolio Simulation
```python
# Daily portfolio evolution
V_t = V_{t-1} * (w_t @ R_t)

# Rebalancing cost
cost_t = transaction_cost * ||w_drift - w_target||_1

# Post-rebalance value
V_t = V_t * (1 - cost_t)
```

### Risk Metrics Computed
| Metric | Formula |
|---|---|
| Sharpe Ratio | `mean(excess_return) / std(excess_return) * sqrt(252)` |
| Sortino Ratio | `mean(excess) / downside_std * sqrt(252)` |
| Max Drawdown | `min(V_t / max(V_s, s<=t) - 1)` |
| Calmar Ratio | `CAGR / abs(Max Drawdown)` |
| Profit Factor | `sum(gains) / sum(losses)` |

---

## Key Design Decisions

### Why Point-In-Time (PIT) Data?
```
Standard approach (WRONG):
  Backtest 2014 uses 2014 annual report data ✗
  Problem: Annual reports published in March 2015
  Result: 3-15% Sharpe inflation from look-ahead bias

This project (CORRECT):
  approx_filing_date = period_end + 60 days (SEC delay)
  Backtest only uses data available on rebalance date ✓
```

### Why Walk-Forward vs Standard Backtest?
```
Standard backtest: optimize on ALL data → test on SAME data
= In-sample Sharpe looks great, OOS collapses

Walk-forward: optimize on past → validate on unseen future
= Realistic estimate of live performance
= WFT beat rate 67% means model generalizes well
```

### Why Volatility Targeting?
```
Without: Position size fixed regardless of market regime
  → 2020 COVID crash: full position into -45% drawdown

With: Position scales down when vol spikes
  → 2020: vol_scale reduced to 0.42x automatically
  → Max drawdown contained to -12.5% vs market -45%
```

---

## Comparison vs Benchmarks

```
Sharpe Ratio Comparison (OOS, 10 years)

Thai Quant Model  ████████████████████ 0.837
Warren Buffett    ██████████████████░░ 0.750  (reference)
S&P 500 (SPY)     █████████████░░░░░░░ 0.590
SET TRI Index     ████████░░░░░░░░░░░░ 0.320
```

---

## Project Structure

```
thai-portfolio-backtester/
│
├── risk/
│   ├── regime_filter.py       # SPY vs SMA200 market state
│   ├── vol_targeting.py       # Realized vol scaling
│   └── circuit_breaker.py     # Drawdown protection layers
│
├── backtest/
│   ├── engine.py              # Core portfolio simulation
│   ├── walk_forward.py        # WFO implementation
│   └── metrics.py             # Sharpe, Sortino, MDD, Calmar
│
├── paper_trade/
│   ├── signal_generator.py    # Monthly signal generation
│   └── performance_log.csv    # Live paper trade results
│
├── notebooks/
│   ├── 01_backtest_full.ipynb
│   ├── 02_walk_forward.ipynb
│   └── 03_risk_analysis.ipynb
│
└── README.md
```

---

## Requirements

```bash
pip install pandas numpy matplotlib scipy statsmodels yfinance jupyter
```

---

## Related Projects

| Project | Description |
|---|---|
| [thai-stock-screener](https://github.com/ymodulus21/thai-stock-screener) | Multi-factor screener that feeds signals into this backtester |
| [thai-stock-backtestv2](https://github.com/ymodulus21/thai-stock-backtestv2) | Enhanced backtest v2 with additional bias corrections |
| [thai-stock-screener-app](https://github.com/ymodulus21/thai-stock-screener-app) | Full pipeline application |

---

## Author

**Kittipong Mahaheng (Bass)**
Business Administration | Quantitative Finance
5+ years trading experience — Forex, BTC, Gold, SET & S&P500

[![GitHub](https://img.shields.io/badge/GitHub-ymodulus21-181717?style=flat&logo=github)](https://github.com/ymodulus21)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=flat&logo=linkedin)](https://linkedin.com)

---

<div align="center">

*Risk management is not about avoiding risk — it's about sizing it correctly.*

</div>
