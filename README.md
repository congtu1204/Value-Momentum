# Vietnam Factor Portfolio Construction

This repository contains a Python/Google Colab workflow for constructing and backtesting long-only equity factor portfolios for the Vietnamese stock market. 
The project builds value-only, momentum-only, and combined value-momentum portfolios, then evaluates portfolio-level returns, turnover, transaction-cost drag, and market-cap-weighted stock allocations.

The workflow is designed for research and educational use. It is not investment advice.

## Project summary

The script builds three main portfolio models:

1. **Value-only model**
   - Rebalanced quarterly.
   - Uses sector-adjusted value signals.
   - Uses `1 / P/B` for financial sectors and `1 / P/E` for non-financial sectors.
   - Applies a publication lag to avoid using fundamental data before it would realistically be available.

2. **Momentum-only model**
   - Rebalanced weekly.
   - Uses the most recent weekly price momentum. Specifically, Uses Thursday-to-Thursday weekly price momentum: the return from last Thursday’s close to the current Thursday’s close.
   - The rebalancing action is assumed to be taken on Friday of the current week, after observing the Thursday close.
   - Sorts stocks within each sector into strong, middle, and weak momentum portfolios.

3. **Combined value + momentum model**
   - Rebalanced weekly.
   - Forward-fills quarterly value data to weekly dates.
   - Combines within-sector value and momentum z-scores into a 50/50 composite score.
   - Includes a diagnostic check for the correlation between value and momentum signals.

Across the models, stocks are sorted within sectors into three portfolios:

| Portfolio | Value model meaning | Momentum model meaning | Combined model meaning |
|---|---|---|---|
| `P1` | Cheap / high value signal | Strong momentum | High value and strong momentum |
| `P2` | Middle group | Middle group | Middle group |
| `P3` | Expensive / low value signal | Weak momentum | Low value and weak momentum |

The market-cap-weighted versions use sector-replicated market-cap weighting. Sector weights are estimated from the full eligible universe, 
then stock weights are assigned within each represented sector based on market capitalization. 
This structure is intended to preserve the sector weight of the broader universe while reducing unintended concentration risk from dominant large-cap stocks.

## Vietnamese market adaptations

The project includes several assumptions and adjustments for the Vietnamese equity market:

- Long-only portfolios only; no short selling.
- Sector-aware sorting to reduce sector concentration in factor portfolios.
- Banks and financial stocks are ranked by `1 / P/B` instead of `1 / P/E`.
- Non-financial stocks are ranked by `1 / P/E`.
- Negative, zero, missing, or extreme P/E values are excluded from the value ranking.
- Quarterly fundamental data is shifted by a publication lag before it is considered usable.
- Trading costs include buy brokerage fee, sell brokerage fee, and sell-side personal income tax.
- Market capitalization is forward-filled to the portfolio construction date to avoid look-ahead bias.

## Methodology

### Data loading

The code expects three types of input data:

1. **Daily price files**
   - CSV files.
   - Multiple files can be uploaded and concatenated.
   - Expected default columns:

| Field | Default column name |
|---|---|
| Date/time | `time` |
| Ticker | `ticker` |
| Close price | `close` |
| Volume | `volume` |

The default date format is:

```python
%m/%d/%Y %H:%M
```

2. **Fundamental valuation file**
   - Excel file.
   - Expected sheet: `Sheet1`.
   - Required columns:
     - `Ticker`
     - `Industrial sector (ICB) L1`
   - The script searches for quarterly P/E and P/B columns containing:
     - `P/E`
     - `P/B`
     - `Quarter`

3. **Market capitalization file**
   - CSV or Excel file.
   - Expected default columns:

| Field | Default column name |
|---|---|
| Ticker | `ticker` |
| Date | `Date` |
| Market capitalization | `Market cap` |

The default market-cap date format is:

```python
%m/%d/%Y
```

Edit the configuration section if your input data uses different column names or date formats.

### Value signal

The value signal is computed as follows:

```text
Financials / Banks: value_signal = 1 / P/B
Other sectors:      value_signal = 1 / P/E
```

For non-financial sectors, P/E values are excluded if they are missing, less than or equal to zero, or above the configured maximum P/E threshold.

A higher `value_signal` is interpreted as cheaper and therefore better for the value model.

### Momentum signal

The weekly momentum signal is computed from weekly resampled prices:

```text
momentum = price(t - start_lag) / price(t - end_lag) - 1
```

With the default configuration:

```python
MOM_WEEK_START = 0
MOM_WEEK_END   = 1
```

this is the most recent one-week return. Weekly prices are resampled using `W-THU`, so the weekly construction date is anchored to Thursday.

### Within-sector portfolio assignment

For each sector and each rebalance date:

- Stocks are ranked within the sector by the relevant signal.
- Top third goes to `P1`.
- Bottom third goes to `P3`.
- Remaining stocks go to `P2`.
- Sectors with fewer than `MIN_SECTOR_SIZE` valid stocks are assigned entirely to `P2`.

This design prevents large sectors from dominating the factor sort and avoids forcing small sectors into unstable tertile splits.

### Market-cap weighting

The market-cap-weighted models use a sector-replicated weighting method:

1. Calculate each sector's market-cap weight in the full eligible universe.
2. Keep only sectors represented in the target portfolio.
3. Redistribute unavailable sector weights across represented sectors.
4. Within each represented sector, weight stocks by market capitalization.

The final stock weight is:

```text
stock_weight = target_sector_weight × stock_market_cap / portfolio_sector_market_cap
```

Weights are normalized so each `date × portfolio` sums to approximately 100%.

### Turnover and transaction costs

The market-cap-weighted backtests calculate turnover using portfolio weights, not ticker counts.

At each rebalance, the code compares the new target weights against the previous portfolio's drifted pre-rebalance weights:

```text
turnover = (entering_weight + exiting_weight + holding_weight_adjustment) / 2
```

The first rebalance is assigned 100% turnover because the portfolio is assumed to be built from cash.

Transaction costs are calculated from active traded weights:

- Buy fee applies to entering stocks and weight increases in existing holdings.
- Sell fee and sell tax apply to exiting stocks and weight decreases in existing holdings.
- `fee_drag = buy_cost + sell_cost + rebal_cost`.

### Gross and net returns

For each portfolio and holding period:

```text
gross_ret = sum(stock_weight × stock_return)
net_ret   = gross_ret - fee_drag
```

`gross_ret` is the portfolio return before estimated trading costs. `net_ret` is the portfolio return after brokerage fees and sell tax.

Annualized returns are calculated using simple arithmetic annualization:

```text
Quarterly model: annualized_return = average_quarterly_return × 4
Weekly model:    annualized_return = average_weekly_return × 52
```

These are not compounded annual growth rates.

## Output files

The script downloads several CSV outputs from Colab.

### Value-only outputs

| File | Description |
|---|---|
| `value_model_3portfolios.csv` | Quarterly value portfolio assignments before market-cap weighting |
| `value_model_p1.csv` | P1 value-only assignments |
| `value_model_p2.csv` | P2 value-only assignments |
| `value_model_p3.csv` | P3 value-only assignments |
| `value_model_quarterly_market_cap_weighted_results.csv` | Quarterly portfolio-level market-cap-weighted results |
| `value_model_quarterly_market_cap_weighted_stock_assignments.csv` | Stock-level quarterly value assignments with market-cap weights |
| `value_model_quarterly_market_cap_weighted_p1.csv` | P1 value market-cap-weighted stock-level output |
| `value_model_quarterly_market_cap_weighted_p2.csv` | P2 value market-cap-weighted stock-level output |
| `value_model_quarterly_market_cap_weighted_p3.csv` | P3 value market-cap-weighted stock-level output |

### Momentum-only outputs

| File | Description |
|---|---|
| `momentum_model_weekly_market_cap_weighted_results.csv` | Weekly portfolio-level market-cap-weighted momentum results |
| `momentum_model_weekly_market_cap_weighted_stock_assignments.csv` | Stock-level weekly momentum assignments with market-cap weights |
| `momentum_model_weekly_market_cap_weighted_p1.csv` | P1 momentum market-cap-weighted stock-level output |
| `momentum_model_weekly_market_cap_weighted_p2.csv` | P2 momentum market-cap-weighted stock-level output |
| `momentum_model_weekly_market_cap_weighted_p3.csv` | P3 momentum market-cap-weighted stock-level output |

### Combined model outputs

The combined model creates portfolio-level and stock-level DataFrames:

| DataFrame | Description |
|---|---|
| `combo_results` | Weekly performance summary for the combined strategy |
| `combo_stock_assign` | Stock-level combined value-momentum assignments |

Depending on the final executed cells, these outputs can be exported to CSV using `to_csv()`.

## Key output columns

### Portfolio-level result files

| Column | Meaning |
|---|---|
| `date` | Portfolio construction date |
| `period_end` | End date of the holding period |
| `portfolio` | Portfolio label: `P1`, `P2`, or `P3` |
| `n_stocks` | Number of stocks in the portfolio |
| `n_entering` | Number of stocks newly entering the portfolio |
| `n_exiting` | Number of stocks exiting the portfolio |
| `turnover` | Weight-based traded fraction of the portfolio |
| `buy_cost` | Estimated buy-side transaction cost |
| `sell_cost` | Estimated sell-side transaction cost and tax |
| `rebal_cost` | Cost from adjusting weights of continuing holdings |
| `fee_drag` | Total cost deducted from gross return |
| `gross_ret` | Holding-period return before trading costs |
| `net_ret` | Holding-period return after trading costs |
| `total_mcap` | Total market capitalization of stocks in the portfolio |

### Stock-level assignment files

| Column | Meaning |
|---|---|
| `ticker` | Stock ticker |
| `sector` | ICB Level 1 sector |
| `measure` | Value measure used, usually `1/P/E` or `1/P/B` |
| `portfolio` | Assigned portfolio: `P1`, `P2`, or `P3` |
| `value_signal` | Value signal used for ranking |
| `mom_signal` | Momentum signal used for ranking |
| `value_zscore` | Within-sector value z-score in the combined model |
| `mom_zscore` | Within-sector momentum z-score in the combined model |
| `combo_score` | 50/50 value-momentum composite score |
| `within_sector_rank` | Rank within the sector |
| `within_sector_size` | Number of eligible stocks in the sector |
| `mcap` | Market capitalization used for weighting |
| `mcap_date` | Actual market-cap observation date, where available |
| `sector_weight_target` | Target sector weight assigned to the portfolio |
| `weight_within_sector` | Stock weight inside its portfolio sector |
| `weight` | Final portfolio weight |
| `status` | `NEW` or `HOLD` status at the rebalance date |

## How to run

The script is currently written for Google Colab because it uses:

```python
from google.colab import files
files.upload()
files.download()
```

Recommended workflow:

1. Open the script or notebook in Google Colab.
2. Edit the configuration section to match your data files.
3. Upload all daily price CSV files when prompted.
4. Upload the valuation Excel file when prompted.
5. Upload the market capitalization file when prompted.
6. Run the cells sequentially.
7. Download the generated CSV output files.

For local execution, replace the Colab upload/download calls with local file paths, for example:

```python
pd.read_csv("data/prices.csv")
pd.read_excel("data/valuation.xlsx")
output_df.to_csv("outputs/result.csv", index=False)
```

## Dependencies

Minimum Python dependencies:

```text
pandas
numpy
openpyxl
```

When running in Google Colab, `google.colab` is available by default. For local execution, remove or replace Colab-specific upload and download functions.

## Suggested repository structure

```text
.
├── README.md
├── semifinal.py
├── data/
│   ├── raw/              # not committed
│   └── processed/        # optional, not committed
├── outputs/              # generated CSV files, usually not committed
└── requirements.txt
```

Suggested `requirements.txt`:

```text
pandas
numpy
openpyxl
```

Suggested `.gitignore`:

```text
__pycache__/
.ipynb_checkpoints/
*.csv
*.xlsx
data/
outputs/
```

Do not commit proprietary raw financial data unless you have permission to redistribute it.

## Important assumptions and limitations

- The backtest depends entirely on the quality and coverage of the input data.
- The script does not automatically adjust for survivorship bias. If the input universe excludes delisted stocks, results may be overstated.
- The transaction-cost model is simplified and does not include bid-ask spread, market impact, lot-size constraints, foreign ownership limits, or liquidity screens.
- Market-cap data is forward-filled to rebalance dates. This avoids look-ahead bias but can still be stale if market-cap observations are sparse.
- Arithmetic annualization is used for summary reporting. For long-horizon performance analysis, CAGR and cumulative return curves should also be examined.
- The model is long-only. It does not construct a long-short factor portfolio.

## Research interpretation

A factor is successful if the ranking separates future returns in the expected direction. For the value model, `P1` should ideally outperform `P3`. For the momentum model, `P1` should ideally outperform `P3`. If `P3` outperforms `P1`, the factor may still have positive absolute returns, but it does not generate a positive factor premium under the tested specification.

The difference between `P1` and `P3` is often more informative than the standalone return of `P1`:

```text
factor_spread = P1_return - P3_return
```

A positive spread indicates that the signal ranks stocks in the intended direction. A negative spread indicates that the opposite side of the ranking performed better.

## Status

This project is a research backtesting workflow. It is not yet a production trading system. Before using it for investment decisions, add more validation, benchmark comparison, cumulative performance charts, liquidity controls, drawdown analysis, and out-of-sample tests.

