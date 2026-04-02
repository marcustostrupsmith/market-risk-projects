# Market Risk Projects

## Acknowledgement
These projects were built with Claude (Anthropic) writing the code and structuring the analysis. I ran the code, interpreted the results, debugged issues, and develop a working understanding of the models and what the outputs meant in practice. All results are based on real or realistic financial data.

## Project 1, FX VaR Risk Engine (`risk-analysis.ipynb`)

### What it does
Measures the daily market risk of a multi-currency FX portfolio from the perspective of a Norwegian bank, using two standard VaR methodologies. The portfolio consists of long positions in EUR/NOK (10M), USD/NOK (5M), GBP/NOK (3M), and EUR/USD (2M), totalling 20M NOK notional.

FX data is fetched live from Yahoo Finance via `yfinance`, covering 5 years of daily closing rates.

### Methods covered
- **Historical Simulation VaR**, takes the 1st percentile of a rolling 250-day P&L window; no distributional assumptions
- **Parametric VaR**, assumes normally distributed returns and estimates VaR as `-(mu - z * sigma)` where z = 2.326 for 99% confidence
- **Backtesting**, counts VaR exceptions and classifies the model using the Basel Committee traffic-light framework (Green / Yellow / Red)
- **Stress scenarios**, applies three shocks: NOK depreciation of 10%, USD strengthening of 8%, and a replay of actual market returns during the COVID crash (Feb–Mar 2020)

### Key findings

**VaR estimates (99% confidence, 1-day horizon, 250-day window):**
| Method | VaR |
|---|---|
| Historical Simulation | 172,498 NOK |
| Parametric | 184,396 NOK |

The two methods produce similar estimates, suggesting returns are approximately normally distributed under normal market conditions.

**Backtesting:**
- 21 exceptions observed over the full sample period
- Basel classification: **RED zone** (10+ exceptions)
- Exceptions cluster around 2022–2024, driven by the Ukraine war, aggressive global rate hikes, and NOK volatility, a period where historical simulation systematically underestimated tail risk due to its reliance on a trailing window that did not yet contain the new regime

**Stress scenarios:**
| Scenario | P&L |
|---|---|
| NOK −10% depreciation | +1,715,583 NOK |
| USD +8% strength | +597,907 NOK |
| COVID crash, total (32 days) | +2,397,106 NOK |
| COVID crash, worst single day | −1,808,234 NOK |

The portfolio is long foreign currencies against NOK, so NOK depreciation scenarios produce gains. However, the COVID worst single day of −1.8M NOK is approximately 10x the daily VaR estimate, illustrating the fundamental limitation of VaR as a standalone risk measure and the necessity of stress testing alongside it.

---

## Project 2, IRRBB / ALM Analyser (`irrbb_alm.ipynb`)

### What it does
Models a simplified bank balance sheet and measures how changes in interest rates affect both short-term income (Net Interest Income) and long-term economic value (Economic Value of Equity). This is the core of what a bank's Asset-Liability Management function does in practice.

Balance sheet data is stored in a **SQLite database** (`bank.db`) and queried using SQL throughout the notebook, reflecting a realistic workflow where risk analysts query institutional data rather than hardcode it.

The balance sheet covers 1.15B NOK in assets (floating and fixed rate mortgages, corporate loans, government bonds, cash) funded by deposits, term deposits, senior bonds, and equity.

### Methods covered
- **Repricing gap analysis**, assigns balance sheet items to time buckets and identifies where assets and liabilities reprice at different speeds
- **NII sensitivity**, applies the five standard Basel IRRBB parallel rate shocks (−200bps to +300bps) and calculates the change in annual net interest income
- **EVE sensitivity**, uses a duration-based present value approximation to calculate the change in economic value of equity under each shock, with regulatory breach flagging at the Basel 15% of Tier 1 Capital threshold
- **Yield curve scenarios**, models non-parallel shifts (steepener and flattener) to capture more realistic rate environments

### Key findings

**Repricing gap:**
| Bucket | Gap |
|---|---|
| 0–3 months | −350M NOK (liability sensitive) |
| 3–12 months | +450M NOK (asset sensitive) |
| 1–3 years | −200M NOK (liability sensitive) |
| 3–5 years | +100M NOK (asset sensitive) |
| 5+ years | +100M NOK (asset sensitive) |

The large positive gap in the 3–12 month bucket (driven by 500M NOK of floating rate mortgages) makes this bank broadly asset sensitive over the medium term, rising rates improve income once short-term deposit repricing passes.

**NII sensitivity (base NII: 31.9M NOK):**
| Shock | ΔNII |
|---|---|
| −200bps | −2.0M NOK |
| −100bps | −1.0M NOK |
| +100bps | +1.0M NOK |
| +200bps | +2.0M NOK |
| +300bps | +3.0M NOK |

NII sensitivity is linear and modest, ±1M NOK per 100bps, reflecting a net short-term repricing gap of 100M NOK. The bank is well hedged from an income perspective.

**EVE sensitivity (Tier 1 Capital: 100M NOK, threshold: 15M NOK):**
| Shock | ΔEVE | Status |
|---|---|---|
| −200bps | +9.7M NOK | OK |
| −100bps | +4.9M NOK | OK |
| +100bps | −4.9M NOK | OK |
| +200bps | −9.7M NOK | OK |
| +300bps | −14.6M NOK | OK |

All scenarios within regulatory limits. EVE sensitivity is significantly larger than NII sensitivity in absolute terms, ±4.9M per 100bps vs ±1M, because EVE captures the full duration of all balance sheet items, including long-duration fixed rate assets that do not reprice within the NII horizon.

**Yield curve scenarios:**
| Scenario | ΔNII | ΔEVE |
|---|---|---|
| Steepener (short −50bps, long +100bps) | −0.75M NOK | −2.62M NOK |
| Flattener (short +100bps, long −50bps) | +1.50M NOK | +0.18M NOK |

The balance sheet is more exposed to steepening than flattening, a steepener hurts both NII and EVE simultaneously, while a flattener improves NII with virtually no EVE impact. This asymmetry arises because short-term asset repricing dominates NII while long-duration fixed assets drive EVE, and these two effects move in opposite directions under non-parallel rate moves.

**The core IRRBB insight:**
NII and EVE tell different stories about the same balance sheet. NII sensitivity of ±1M per 100bps suggests the bank is well hedged short-term. EVE sensitivity of ±4.9M per 100bps, nearly 5% of Tier 1 Capital per 100bps move, reveals meaningful long-term interest rate exposure that NII alone would not capture. This is precisely why regulators require banks to monitor both measures.


## Tools and libraries
- Python 3.9
- `pandas`, `numpy`, `matplotlib`, data manipulation and charting
- `yfinance`, live FX data
- `scipy`, statistical functions
- `sqlite3`, balance sheet database (built into Python)
