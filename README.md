# Sector Rotation Early Warning System

Tests whether unusual divergence between defensive and high-beta sectors can give early warning of S&P 500 drawdowns.

---

## The Main Idea

Professional investors often reposition ahead of market stress. Two observable patterns tend to appear before a broad market sell-off:

Money moves from high-beta sectors (Technology, Consumer Discretionary) into lower-risk sectors (Staples, Healthcare, Utilities) before the broad market peaks. Defensives start outperforming on individual days while SPX is still elevated.

On days when this rotation is occurring, `defensive_return > high_beta_return`. This notebook measures that directional gap, applies a threshold, and evaluates whether the resulting signal genuinely precedes drawdowns.

---

## Sectors Used

### Defensive, low-beta basket

| ETF | Sector           |
| --- | ---------------- |
| XLP | Consumer Staples |
| XLV | Health Care      |
| XLU | Utilities        |

Beta ≈ 0.5–0.7.

### High-beta basket

| ETF | Sector                 |
| --- | ---------------------- |
| XLK | Technology             |
| XLY | Consumer Discretionary |

Beta ≈ 1.2–1.4.

### Benchmark

| Ticker | What it is                               |
| ------ | ---------------------------------------- |
| SPY    | S&P 500 ETF — used directly as benchmark |

---

## How the Signal is Built

### Step 1 — Daily returns

For each trading day, compute the simple daily percentage return of every ETF.

### Step 2 — Basket averages

Average the daily returns across all ETFs in each basket:

```
defensive_return(t) = mean(XLP return, XLV return, XLU return)
high_beta_return(t) = mean(XLK return, XLY return)
```

### Step 3 — Rotation signal

Compute the directional return gap between the two baskets, clipped to zero on risk-ON days:

```
signal(t) = max(defensive_return(t) − high_beta_return(t), 0)
```

The signal is in raw return units. The notebook implements three variants of this signal for comparison:

#### Alt-A — 1-day return spread

```
signal(t) = max(mean_daily_return(Defensive) − mean_daily_return(HighBeta), 0)
```

Measures whether defensives outperformed high-beta today.

##### Alt-B — 20-day return spread

```
signal(t) = max(mean_20day_return(Defensive) − mean_20day_return(HighBeta), 0)
            expressed in percentage points
```

Uses `pct_change(20)` - the cumulative return over the past 20 trading days - for each basket, then takes the directional spread. Less noise from single-day sector moves. 

#### Alt-C — Log-ratio Z-score

```
Ratio(t)    = def_basket(t) / hb_basket(t)          [indexed to 100 at start]
D(t)        = log(Ratio(t) / Ratio(t−20))           [20-day log return of ratio]
Z(t)        = (D(t) − rolling_mean(D, 252)) / rolling_std(D, 252)

Warning:  Z > 1   (one sigma above the rolling average)
Extreme:  Z > 2
```

Each ETF is first indexed to 100 at the start date so no single ETF's price level dominates the basket. The ratio captures relative leadership independent of overall market direction - so for eg if both baskets fall 5% but defensives only fall 2%, the ratio rises. The 20-day log return of the ratio is then standardized against its own 252-day rolling history, so the threshold adapts to the current volatility regime.

#### Comparison

| Signal                | Recall | Precision | F1    |
| --------------------- | ------ | --------- | ----- |
| 1-day spread (Alt-A)  | 58%    | 55%       | 0.561 |
| 20-day spread (Alt-B) | 44%    | 63%       | 0.519 |
| Log-ratio Z (Alt-C)   | 77%    | 50%       | 0.607 |

---

## Drawdown Labels

For every trading day _t_, the notebook looks forward `FWD_WINDOW = 20` trading days and checks if SPX fall more than 5% at any point in that window.

```
fwd_max_drawdown(t) = min(SPX[t+1 ... t+20]) / SPX[t] − 1

is_pre_drawdown(t) = True   if fwd_max_drawdown(t) < −5%
                     False  otherwise
```

A day is labelled "pre-drawdown" if a significant crash is coming within the next month.

---

## Warning Zone

The warning zone combines two conditions:

```
warn_zone(t) = (signal(t) > warn_threshold)  AND  (SPX(t) > SPX 50-day MA)
```

It is true when two things are simultaneously true:

1. The rotation signal is elevated (defensives outperforming above the 95th-percentile threshold)
2. SPY is still above its 50-day moving average (market hasn't rolled over yet).

---

## Evaluation Methodology

The evaluation is event-level with a symmetric 40-day window for both precision and recall:

```
Recall:    of N crash events, the number which had signal > warning_threshold
           in the 40 days before that crash event started.

Precision: of M signal episodes, the number whihc had a crash event start
           in the 40 days after that episode started.
```
