# BankNifty Short Strangle Backtest (09:20 Entry)

## 1. Overview

This project implements a **short strangle** options trading strategy on the BankNifty index using 1‑minute historical data. The strategy is backtested over the full year 2023 with the following core rules:

- **Entry**: Every trading day at **09:20:59** (close of the 1‑minute candle), sell one Call (CE) and one Put (PE) option.  
- **Strike selection**: From all available strikes, choose the CE and the PE whose premium (09:20 close price) is **closest to ₹50**.  
- **Exit**:  
  - Regular exit at **15:20:59** (close of that minute).  
  - **Stop loss**: If at any intraday minute the option’s `High` price reaches **1.5 × entry price** (50% loss on premium), exit immediately at that minute’s close.  
- **Position sizing**: Fixed **1 lot** per leg, lot size = 15. No compounding.  
- **Data**: Options and spot (underlying) 1‑minute OHLC data for BankNifty, covering the whole year 2023.  

The strategy is a pure **premium seller** – it profits from time decay (theta) when the market remains range‑bound, but faces unlimited theoretical risk. The stop loss (50% of premium) caps the loss on each leg.

---

## 2. Strategy Rationale (What it tries to exploit)

A short strangle is a **neutral‑to‑slightly‑directional** volatility strategy. The trader expects:

- The underlying (BankNifty) to stay within a range defined by the two strike prices.  
- Options’ time value to decay faster than the index moves.  
- The probability of a large move that would trigger the stop loss is low enough to make the strategy profitable after many trades.  

By selecting strikes with premiums near ₹50, the strategy aims to collect a meaningful credit while still being **out‑of‑the‑money** (OTM). The 50% stop loss ensures that a single large adverse move does not erase several days’ profits.

---

## 3. Data & Setup

### 3.1 Files required

| File | Description |
|------|-------------|
| `Options_data_2023.csv` | 1‑minute options data for the whole year. Must contain columns: `Date`, `Ticker`, `Time`, `Open`, `High`, `Low`, `Close`. |
| `BANKNIFTY_SPOT.csv` | 1‑minute spot (underlying) data. Must contain `ts` (timestamp) and `c` (close). |

### 3.2 Environment

- Python 3.8+ with libraries: `pandas`, `numpy`, `matplotlib`, `openpyxl`, `scipy`.  
- The notebook was run in **Google Colab** with a T4 GPU (the data processing is CPU‑bound; GPU not needed).

### 3.3 Data Preprocessing

The notebook loads the CSV files, combines `Date` and `Time` into a datetime index, extracts strike and option type from the `Ticker` column using a regex, and merges spot prices to each trade’s entry/exit times. The code is heavily vectorised to handle ~10 million rows efficiently.

---

## 4. Backtest Logic (Step‑by‑Step)

1. **Data loading & cleaning**  
   - Read options and spot data, parse timestamps.  
   - Extract strike price and option type (`CE` / `PE`).  
   - Calculate the nearest Wednesday (expiry) for each row (for later analysis).

2. **Strike selection (09:20:59)**  
   - Filter rows at `Time == "09:20:59"`.  
   - For each day and option type, compute `abs(Close – 50)`.  
   - Keep the strike with the smallest difference. This gives **one CE** and **one PE** per trading day.

3. **Exit simulation**  
   - For each leg, retrieve all intraday data from entry minute (exclusive) to 15:20:59.  
   - Check if the `High` price ever reaches or exceeds the stop level (`entry_price × 1.5`).  
   - If yes → exit at that minute’s `Close` (stop‑loss hit).  
   - Otherwise → exit at 15:20:59 `Close`.  

4. **PnL & NAV calculation**  
   - PnL per leg = `(entry_price – exit_price) × lot_size (15)`.  
   - Cumulative PnL summed over all legs in chronological order.  
   - NAV starts at **100** and changes proportionally with cumulative PnL (relative to an initial capital of ₹1,000,000).  

5. **Performance metrics**  
   - Win / loss counts (by leg).  
   - Average percentage return on expiry vs non‑expiry days.  
   - CAGR, maximum drawdown.  
   - Monthly returns and equity curve plot.  
   - Distribution analysis: skewness and kurtosis of daily returns.

---

## 5. Results & Key Findings

### 5.1 Overall Performance

| Metric | Value |
|--------|-------|
| **Total trades (legs)** | 494 (247 CE + 247 PE) |
| **Winners (CE)** | 124 (50.2%) |
| **Losers (CE)** | 123 (49.8%) |
| **Winners (PE)** | 120 (48.6%) |
| **Losers (PE)** | 127 (51.4%) |
| **Combined win rate** | 49.4% |
| **Average % P&L per trade leg** | ~0.09% (relative to ₹10 L capital) |
| **CAGR** | ~0.81% |
| **Maximum drawdown** | –0.40% |

### 5.2 Expiry vs Non‑Expiry Average Return (per leg)

| Day Type | CE avg % | PE avg % | Combined avg % |
|----------|----------|----------|----------------|
| **Expiry (Wednesday)** | 0.87% | 0.02% | 0.45% |
| **Non‑expiry** | 0.02% | 0.15% | 0.09% |

> *Note:* Percentages are relative to the initial capital of ₹1 000 000, **not** to the option premium itself.

### 5.3 Monthly Returns

Most months show small positive or slightly negative returns; the best month was May 2023 (+0.31%), the worst was January 2023 (–0.16%). The equity curve is almost flat with very low volatility.

### 5.4 Distribution of Returns

- **Skewness**: +0.37 (slightly positive, i.e., a few larger gains on the right tail).  
- **Kurtosis**: –1.36 (platykurtic – thin tails, fewer extreme outliers than a normal distribution).

**Interpretation**  
The strategy does **not** suffer from the severe negative skew that plagues many short‑volatility strategies (where small profits are wiped out by infrequent huge losses). The 50% stop loss appears effective at trimming the left tail. The low kurtosis indicates that returns are relatively stable and predictable.

### 5.5 What does this tell us about the strategy?

- **Safety**: Very high. Maximum drawdown is only –0.40%, and the return distribution is well‑behaved.  
- **Profitability**: Very low. A CAGR of ~0.8% is negligible, especially considering transaction costs, slippage, and margin requirements.  

**The strategy, in its current form, is statistically sound but not commercially viable.** The small edge is easily wiped out by real‑world frictions.

---

## 6. Potential Improvements

1. **Optimise strike selection**  
   - Target a higher premium (e.g., ₹100 or ₹150) to increase credit and allow a wider stop‑loss distance (or keep the 50% stop, which then becomes larger in absolute terms).  
   - Alternatively, use an **ATM / slightly OTM** strangle to capture higher theta.

2. **Dynamic stop loss**  
   - Use a volatility‑based stop (e.g., 2× ATR of the underlying) instead of a fixed 50% of premium.

3. **Filtering**  
   - Avoid trading on days before major economic announcements or expiry week.  
   - Enter only when implied volatility is above a certain percentile.

4. **Position sizing**  
   - Increase leverage cautiously – the low drawdown suggests room for scaling.

5. **Incorporate costs**  
   - Brokerage, exchange charges, and slippage will turn the marginal positive return negative unless the edge is improved.

---

## 7. How to Run the Code

The entire backtest is contained in a single Jupyter notebook (`STRANDLE_BANKNIFTY_BACKTESTING.ipynb`).  

### 7.1 Preparation

- Place `Options_data_2023.csv` and `BANKNIFTY_SPOT.csv` in your Google Drive under `MyDrive/STRANGLE_BANKNIFTY/`.  
- Open the notebook in Google Colab (or any Jupyter environment with the required libraries).

### 7.2 Execution

Run all cells sequentially. The notebook will:

1. Mount Google Drive.  
2. Load and preprocess the data (this step takes ~40 seconds for 10 million rows).  
3. Select strikes at 09:20.  
4. Simulate exits with stop loss.  
5. Compute all performance metrics and plots.  
6. Export an Excel report (`short_strangle_backtest.xlsx`) to the same Drive folder.  

### 7.3 Outputs

The Excel file contains five sheets:

- **Guide** – Explanation of the strategy and backtest.  
- **Tradesheet** – Each leg (CE/PE) with entry/exit times, prices, PnL, and spot price at entry.  
- **Statistics** – CAGR, max drawdown, win/loss summary, average returns by day type.  
- **Equity_Curve** – NAV and drawdown over time.  
- **Monthly_PnL** – Month‑by‑month returns.

Plots (equity curve and return distribution) are displayed inline in the notebook.

---

## 8. Conclusion

The **09:20 short strangle with ₹50 premium target** is a **very safe** strategy that produces a nearly flat equity curve. Its distribution is positively skewed and has thin tails – exactly what one wants from a short‑volatility strategy. However, the absolute return is too low to be profitable after transaction costs. The code and methodology are solid, and the framework can be easily adapted to test other strike‑selection rules, stop‑loss levels, or entry times.

**Final verdict** (based on the primary data):  
✅ Statistically robust, low risk.  
❌ Economically unattractive in its current form – needs optimisation.
