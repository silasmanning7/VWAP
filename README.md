# TWAP Scheduling & Cost Estimation Model

Interactive TWAP scheduling and execution cost estimation tool using intraday equity data.  
This model **does not execute trades** — it only generates an execution schedule, estimates expected trading costs, and compares expected TWAP performance versus **VWAP**.

---

## Purpose

Simulate a **TWAP-style execution plan** for a user-defined equity order and estimate the **expected trading cost per slice and in total**, subject to participation and liquidity constraints. Benchmark expected TWAP execution performance against VWAP.

---

## User Inputs (Interactive Prompt)

| Input | Description | Example |
|---|---|---|
| `ticker` | Equity ticker | `AAPL` |
| `Q` | Parent order size (shares) | `50,000` |
| `direction` | Trade direction | `Long` or `Short` |
| `start_time` | Start time of execution | `10:00` |
| `bin_minutes` | Time bucket size | `5`, `10`, or `15` |
| `p_max` | Max participation rate | `0.02 – 0.10` |

---

## Data Source

- **yfinance** intraday OHLCV data  
- Intraday bars are resampled to the selected time bucket size (`bin_minutes`)  
- VWAP is computed using an **OHLCV-based approximation**, not true trade-level VWAP

---

## Core Rules & Constraints

- **Max participation per bucket:** `p_max` (2%–10%)  
- **Scheduling & cost estimation only** (no execution)  
- **No persistence** — all results are displayed immediately  
- Execution runs from `start_time` until order completion or market close  

---

## Volume per Slice (`V_i`) — 20-Day Rolling Intraday Estimate

### Definition

`V_i` is the **expected market trading volume** for time bucket `i`, estimated using a **20-day rolling window of historical intraday volume** for the same time-of-day bucket.

This mirrors the methodology used to estimate volatility.

---

### How `V_i` Is Calculated

1. Pull intraday OHLCV data for the last **≥20 trading days**
2. For each trading day, resample volume into `bin_minutes` buckets
3. Label buckets by **time-of-day** (e.g. `10:00–10:05`)
4. For each bucket `i`, collect historical volumes:

\[
\{V_{d-1,i}, V_{d-2,i}, \ldots, V_{d-20,i}\}
\]

5. Compute expected bucket volume:

\[
V_i = \text{mean}\left(V_{d-k,i}\right), \quad k = 1 \ldots 20
\]

If fewer than 20 observations exist:
- Use available data if ≥10 observations
- Otherwise return a warning:  
  `Insufficient intraday history to estimate bucket volume`

---

### Usage of `V_i`

The same `V_i` is used consistently for:
- Participation caps
- POV reporting
- Trading cost model

---

## Liquidity Feasibility Check (Pre-Schedule)

Before scheduling, check whether the parent order can be completed under the participation constraint using expected intraday liquidity.

### Maximum Executable Shares

\[
q^{max}_i = p_{\text{max}} \times V_i
\]

\[
Q_{max} = \sum_i q^{max}_i
\]

---

### Behavior

- If \( Q \le Q_{max} \): proceed with full schedule  
- If \( Q > Q_{max} \):  
  - Generate a **partial schedule only** (through market close)  
  - Display a **feedback box** below user input:

**Feedback Box**
- Status: `Cannot complete order within participation constraint`
- Requested shares: `Q`
- Max executable shares today: `Q_max`
- Unfilled shares: `Q - Q_max`

Cost and VWAP comparisons are calculated **only on executed shares**.

---

## Scheduling Logic

### Step 1: Estimate Time to Completion

\[
T = \left(\frac{Q}{p_{\text{max}} \times ADV}\right) \times 390
\]

Where:
- `T` = minutes to complete trade  
- `Q` = parent order size  
- `ADV` = average daily volume  
- `p_max` = max participation rate  
- `390` = minutes per trading day  

---

### Step 2: Number of Slices

\[
N = \left\lfloor \frac{T}{\text{bin\_minutes}} \right\rfloor
\]

---

### Step 3: Child Order Size

\[
q = \frac{Q}{N}
\]

---

### Step 4: Per-Bucket Execution

For each bucket `i`:

\[
q_{\text{executed},i} =
\min\left(q,\ p_{\text{max}} \times V_i,\ \text{remaining shares}\right)
\]

---

### Step 5: Participation Rate (POV%)

\[
POV_i = \frac{q_{\text{executed},i}}{V_i}
\]

This realized **POV\% is reused directly in the trading cost model**.

---

## VWAP (Single Benchmark Over Execution Window)

### Representative Price per Bar

For each intraday bar `j`:

\[
\text{price}_j = \frac{H_j + L_j + C_j}{3}
\]

---

### VWAP Over Execution Window

Compute VWAP using **all intraday bars from `start_time` through the final executed slice end time**:

\[
VWAP_{\text{window}} =
\frac{\sum_j (\text{price}_j \times \text{volume}_j)}
{\sum_j \text{volume}_j}
\]

This is the **only VWAP** used for benchmarking.

---

## Expected Trading Cost Model

### Cost Per Slice (bps)

\[
C_i = \text{halfspread} + G \times S \times POV_i
\]

Where:

| Variable | Description |
|---|---|
| `halfspread` | Half of quoted bid-ask spread (bps) |
| `G` | Impact coefficient |
| `S` | Volatility over the time slice |
| `POV_i` | Realized participation rate from schedule |

---

### Volatility Over Slice

1. Compute log returns over matching intraday intervals for the **last 20 trading days**:

\[
L_x = \ln\left(\frac{P_x}{P_{x-1}}\right)
\]

2. Compute standard deviation:

\[
S = \text{stdev}(L_x)
\]

---

## Impact Coefficient \( G \) (by ADV)

| Liquidity Bucket | ADV | G |
|---|---|---|
| MEGA | > 50M | 0.10 |
| LARGE | 10–50M | 0.25 |
| MID | 2–10M | 0.50 |
| SMALL | 0.5–2M | 0.90 |
| MICRO | ≤ 0.5M | 1.50 |

---

## Half-Spread (bps) (by ADV)

| Liquidity Bucket | Halfspread |
|---|---|
| MEGA | 0.5 |
| LARGE | 1.0 |
| MID | 2.0 |
| SMALL | 5.0 |
| MICRO | 7.5 |

---

## Converting Costs to Price Terms (VWAP Comparison)

Costs are computed in **basis points** and converted using the single VWAP benchmark.

### Total Expected Cost (Dollars)

\[
\text{TotalCost}_{\$} =
\sum_i \left(
q_{\text{executed},i}
\times
\frac{C_i}{10{,}000}
\times
VWAP_{\text{window}}
\right)
\]

---

### Expected All-In Execution Price

Let:

\[
\Delta P =
\frac{\text{TotalCost}_{\$}}{\sum_i q_{\text{executed},i}}
\]

- **Long (Buy):**
\[
P_{\text{TWAP,allin}} = VWAP_{\text{window}} + \Delta P
\]

- **Short (Sell):**
\[
P_{\text{TWAP,allin}} = VWAP_{\text{window}} - \Delta P
\]

---

## Outputs (Displayed Below User Input)

1. **Feedback Box** (if insufficient liquidity)
   - Requested shares `Q`
   - Max executable shares `Q_max`
   - Unfilled shares `Q - Q_max`

2. **Execution Schedule Table**
   - Time bucket
   - Slice size
   - Cumulative shares
   - Expected bucket volume `V_i`
   - **POV%**
   - Expected slice cost (`C_i`, bps)

3. **Schedule Chart**
   - Bars: shares executed per slice  
   - Optional cumulative overlay  

4. **Expected Cost Summary**
   - Total expected cost (bps and $)
   - Expected cost per share

5. **Benchmark vs VWAP**
   - \( VWAP_{\text{window}} \)
   - \( P_{\text{TWAP,allin}} \)
   - Relative performance (bps and $), direction-aware:
     - **Long:** lower than VWAP is better  
     - **Short:** higher than VWAP is better  

---

## Assumptions & Limitations

- Intraday volume is estimated from historical data (expected liquidity)
- VWAP is OHLCV-approximated
- No order book, queue priority, or adverse selection modeling
- Expected costs, not realized outcomes

---

## Intended Use

- Execution research and education  
- Strategy prototyping  
- Market impact intuition  

**Not intended for live trading or production execution.**
