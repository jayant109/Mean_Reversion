# Mean Reversion Trading System
### Advanced Z-Score Backtesting Framework with Regime Filtering & Risk Management

---

## Overview

This project implements a **production-grade mean reversion backtesting system** built from scratch in Python. It goes well beyond a textbook z-score strategy by incorporating multi-factor signal confirmation, dynamic risk management, walk-forward optimisation, and Monte Carlo analysis — techniques used by quantitative hedge funds.

The core hypothesis: **asset prices that deviate significantly from their statistical mean tend to revert**. This project rigorously tests whether that hypothesis can be profitably exploited after accounting for transaction costs, volatility regimes, and parameter uncertainty.

---

## Strategy Architecture

```
+---------------------------------------------------------+
|                    SIGNAL PIPELINE                      |
|                                                         |
|  Raw OHLCV  ->  Indicators  ->  Filters  ->  Signal    |
|                                                         |
|  Indicators:          Filters:                          |
|  - Z-score            - ADX regime filter               |
|  - Bollinger Bands    - RSI confirmation                |
|  - RSI                - ATR stop-loss                   |
|  - ATR                - Volatility position sizing      |
|  - ADX                                                  |
+---------------------------------------------------------+
```

### Entry Conditions (Long)

All four conditions must be satisfied simultaneously:

| Condition | Filter | Rationale |
|-----------|--------|-----------|
| Z-score crosses below -2 | `zscore < -Z_ENTRY` | Price statistically cheap |
| Price below lower Bollinger Band | `Close < BB_lower` | Confirms price extremity |
| RSI below oversold threshold | `RSI < 35` | Momentum reversal likely |
| ADX below trend threshold | `ADX < 25` | No strong prevailing trend |

Short entries use the mirror conditions. The ADX filter is the most important: mean reversion fails in strongly trending markets, so trades are blocked when `ADX > ADX_TREND_THRESH`.

### Exit Conditions

A position is closed when **any** of the following triggers:

1. Z-score reverts back through +/-0.5 (mean reversion achieved)
2. ATR-based stop-loss is hit (hard risk limit)
3. RSI reaches neutral zone 45-55 (momentum exhausted)

---

## Mathematical Foundations

### Z-Score
```
z_t = (P_t - mu_t) / sigma_t
```
where mu_t and sigma_t are the rolling mean and standard deviation over a configurable lookback window.

### Bollinger Bands
```
Upper_t = mu_t + k * sigma_t
Lower_t = mu_t - k * sigma_t
```

### Average True Range
```
TR_t  = max(H_t - L_t, |H_t - C_{t-1}|, |L_t - C_{t-1}|)
ATR_t = (1/n) * sum(TR_{t-i} for i in 0..n-1)
```

### Volatility-Scaled Position Sizing
```
Position Size = (Risk% x Equity) / (ATR_MULT x ATR_t)
```
This ensures a fixed-dollar risk per trade regardless of market volatility. In low-vol regimes the position is larger; in high-vol regimes it shrinks automatically.

### Performance Metrics
```
Sharpe  = sqrt(252) * E[r_t] / sigma(r_t)
Sortino = sqrt(252) * E[r_t] / sigma_downside(r_t)
CAGR    = (V_T / V_0)^(252/T) - 1
Calmar  = CAGR / |Max Drawdown|
```

---

## Features

### Core Strategy
- Rolling z-score with configurable window and threshold
- Bollinger Band confirmation (price must breach the band)
- RSI momentum filter (oversold/overbought confirmation)
- ADX regime filter (skips trades during strong trends)

### Risk Management
- ATR-based dynamic stop-losses (stops widen in volatile markets)
- Volatility-scaled position sizing (fixed % risk per trade)
- Maximum position cap (concentration risk control)
- Transaction costs applied on both entry and exit

### Backtesting Engine
- Event-driven row-by-row simulation (no lookahead bias)
- Full trade accounting: entry, stop-out, and signal exit
- Supports both long and short positions
- Equity curve and drawdown series

### Performance Analytics
- 15+ metrics: CAGR, Sharpe, Sortino, Calmar, Max Drawdown, VaR, CVaR
- Per-trade log with direction, entry/exit prices, stop price, PnL
- Profit factor and expectancy per trade
- Benchmark comparison (strategy vs buy-and-hold)

### Walk-Forward Optimisation
- Rolling in-sample / out-of-sample parameter search
- Grid search over (window, z_entry) combinations
- Out-of-sample Sharpe ratio as the robustness measure
- Prevents overfitting to historical parameters

### Monte Carlo Analysis
- Block bootstrap (preserves return autocorrelation)
- 500 alternative equity paths
- P5 / P50 / P95 confidence bands
- Value at Risk (VaR) and Conditional VaR (CVaR)
- Probability of profit and probability of ruin

### Visualisation
- 6-panel dark-theme dashboard
- Price + Bollinger Bands with trade markers (long/short/exit/stop)
- Z-score with shaded entry zones
- RSI + ADX dual-axis panel
- Equity curve vs buy-and-hold with outperformance shading
- Drawdown timeline
- Monte Carlo paths with confidence envelope

---

## Project Structure

```
mean-reversion-bot/
├── mean_reversion_bot.ipynb   # Main notebook — all code in one file
├── README.md
└── requirements.txt
```

The notebook is fully self-contained — every function is defined inline with detailed docstrings. Each section builds on the previous, making it easy to follow the full pipeline end to end.

---

## Quickstart

```bash
# 1. Clone
git clone https://github.com/your-username/mean-reversion-bot.git
cd mean-reversion-bot

# 2. Install
pip install -r requirements.txt

# 3. Open notebook
jupyter notebook mean_reversion_bot.ipynb
```

Run all cells top to bottom. The only cell you need to edit is **Section 2 — Configuration**.

---

## Configuration Reference

```python
# Asset
SYMBOL     = 'AAPL'        # Any yfinance ticker
START_DATE = '2019-01-01'
END_DATE   = '2024-01-01'

# Core strategy
WINDOW        = 20     # Rolling lookback window (trading days)
Z_ENTRY       = 2.0    # Z-score threshold for entry
BB_STD        = 2.0    # Bollinger Band width (std deviations)

# Confirmation filters
RSI_PERIOD       = 14
RSI_OVERSOLD     = 35   # RSI must be below this to confirm long entry
RSI_OVERBOUGHT   = 65   # RSI must be above this to confirm short entry
ADX_PERIOD       = 14
ADX_TREND_THRESH = 25   # Skip trades when ADX > this (strong trend present)

# Risk management
ATR_PERIOD     = 14
ATR_STOP_MULT  = 2.0   # Stop = entry +/- (ATR_STOP_MULT x ATR)
RISK_PER_TRADE = 0.01  # Risk 1% of equity per trade
MAX_POSITION   = 0.20  # Max 20% of equity in any single trade
FEE            = 0.001 # 0.1% transaction cost per trade

# Walk-forward
WFO_TRAIN_MONTHS = 18
WFO_TEST_MONTHS  =  6

# Monte Carlo
MC_SIMULATIONS = 500
MC_BLOCK_SIZE  =  10
```

### Parameter Sensitivity Guide

| Parameter | Tighten | Loosen |
|-----------|---------|--------|
| `WINDOW` lower | More signals, more noise | Fewer, cleaner signals |
| `Z_ENTRY` lower | More trades, lower quality | Fewer, higher-quality trades |
| `RSI_OVERSOLD` higher | More permissive, more entries | Stricter confirmation |
| `ADX_TREND_THRESH` lower | More trades filtered out | Fewer blocked trades |
| `ATR_STOP_MULT` lower | Tighter stops, more stop-outs | Wider stops, more room |

---

## Notebook Sections

| Section | Description |
|---------|-------------|
| **1. Setup** | pip install + imports + colour palette |
| **2. Configuration** | All parameters in one cell |
| **3. Data & Indicators** | OHLCV fetch, Z-score, BB, RSI, ATR, ADX, realised vol |
| **4. Signal Engine** | Multi-factor crossover logic with regime filter |
| **5. Risk Management** | ATR stop computation + volatility position sizing |
| **6. Backtester** | Event-driven engine with stop-loss and full accounting |
| **7. Performance Analytics** | 15+ metrics + formatted trade log |
| **8. Benchmark Comparison** | Strategy vs buy-and-hold, alpha calculation |
| **9. Walk-Forward Optimisation** | Rolling OOS parameter validation |
| **10. Monte Carlo** | Block bootstrap, VaR/CVaR, probability of ruin |
| **11. Dashboard** | 6-panel dark-theme interactive chart |

---

## Suggested Tickers

| Ticker | Asset | Notes |
|--------|-------|-------|
| `AAPL` | Apple | Liquid, moderate vol |
| `TSLA` | Tesla | High vol — tighter position sizes |
| `SPY` | S&P 500 ETF | Lower vol, fewer extremes |
| `GLD` | Gold ETF | Often mean-reverting |
| `BTC-USD` | Bitcoin | Extreme vol — good stress test |
| `MSFT` | Microsoft | Stable, services-driven |

---

## Dependencies

```
yfinance>=0.2.36
pandas>=2.0.0
numpy>=1.26.0
matplotlib>=3.8.0
scipy>=1.11.0
```

---

## Design Decisions

**Why crossover signals instead of level signals?**
A level-based signal (zscore < -2) stays active for many bars, causing multiple conflicting entries. Crossover detection fires once at the exact bar the threshold is breached, giving the backtester a clean, unambiguous entry point with no position-state conflicts.

**Why block bootstrap instead of parametric Monte Carlo?**
A parametric GBM Monte Carlo assumes returns are i.i.d. and normally distributed — neither is true for real equity returns. Block bootstrapping resamples consecutive chunks of actual returns, preserving autocorrelation, fat tails, and volatility clustering without distributional assumptions.

**Why event-driven instead of vectorised backtesting?**
Vectorised backtests are faster but require careful handling of position state and stop-losses. An event-driven loop is explicit and readable — every decision is visible in the code and trivially extensible.

**Why ATR stops instead of fixed-% stops?**
ATR adapts to market conditions. In low-volatility environments, a 2% stop might be hit by noise. In high-volatility environments, the same stop might not give the trade enough room. ATR-based stops scale with actual observed market movement.

**Why ADX as a regime filter?**
Mean reversion strategies have a well-known failure mode: they lose badly in trending markets because they keep buying into falling prices (or selling into rising prices). ADX quantifies trend strength without regard to direction, making it an effective, direction-agnostic gatekeeper.

---

## Possible Extensions

- **Pairs trading**: apply the z-score to the spread between two correlated assets (e.g. AAPL vs MSFT)
- **Kalman filter**: replace rolling mean with an adaptive state-space estimate that responds faster to regime changes
- **FinBERT sentiment overlay**: filter entries using NLP sentiment scores on financial news headlines
- **Portfolio-level risk**: extend to multiple assets with correlation-based allocation and portfolio-level drawdown limits
- **Live trading**: connect to a broker API (Alpaca, Interactive Brokers) for paper or live trading

---

## License

MIT
