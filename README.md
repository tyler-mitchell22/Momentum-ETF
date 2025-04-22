# Momentum ETF Strategy

## Overview
This repository is an implementation of a momentum-based ETF strategy that selects stocks from the NYSE based on risk-adjusted momentum. The strategy shows comparable results with the S&P 500 (SPY) from 2013-2020, and shows significant out performance post-pandemic through 2023.

## Strategy Methodology
The strategy follows these key steps:
1. Filters NYSE stocks with market cap > $1B, price > $5, and IPO year ≤ 2012
2. Calculates risk-adjusted momentum as (12-month return - 1-month return) / 12-month volatility
3. Rebalances monthly into the top 10 stocks with highest risk-adjusted momentum
4. Equally weights the selected stocks in the portfolio

## Performance Highlights
The strategy delivers exceptional 3-year performance metrics:

### 3-Year Performance
- **CAGR**: 58.98%
- **Total Return**: 301.85%
- **Beta**: -0.08 (negative correlation to market)
- **Alpha**: 66.14%
- **Maximum Drawdown**: -14.92%
- **Sharpe Ratio**: 0.21
- **Sortino Ratio**: 4.99
- **Calmar Ratio**: 0.04
- **Volatility**: 97.69%

### Since Inception (2013-2023)
- **Total Return**: 674.74%
- **CAGR**: 23.15%
- **Beta**: 0.69
- **Alpha**: 19.29%
- **Maximum Drawdown**: -19.54%
- **Sharpe Ratio**: 0.15
- **Sortino Ratio**: 2.49
- **Calmar Ratio**: 0.01
- **Volatility**: 54.93%

## Key Insights
- The strategy's recent negative 3-year beta of -0.08 is extremely rare and valuable, indicating counter-market movement
- Strong positive alpha, 66.14% over 3 years, shows significant risk-adjusted outperformance
- The strategy appears to have evolved substantially over time, with the inception beta of 0.69 showing initial market correlation
- Despite similar volatility profiles over both time periods, the strategy has achieved significantly higher returns in the recent 3-year window

## File Structure
- `Model-2-risk-adjusted.R`: Main script implementing the momentum strategy
- `Momentum_ETF_Markdown.md`: Markdown with a table of contents
- `Momentum ETF vs SPY Performance.png`: Visualization of results from the entire period

## Requirements
The strategy implementation requires the following tools / packages:
## Tools:
- R/RStudio
- DB Browser for SQLite
## Packages:
- tidyquant
- BatchGetSymbols
- xts
- tidyverse
- janitor
- lubridate
- TTR
- dygraphs
- RSQLite
- DBI

## Usage
To run the strategy:
1. Install required packages using `install.packages()` or `p_load()` function from the pacman library
2. Run the R script code in your IDE of choice to see the full analysis with visualizations
3. The script will automatically create a SQLite database (`momentum_etf.sqlite`) to track:
   - Selected stocks at each rebalance date
   - Historical ETF and SPY performance
   - All performance metrics
4. After the script finishes, open the generated database file in DB Browser to:
   - Browse the ETF composition history
   - Run custom queries to analyze performance patterns
   - Track the most frequently selected stocks
5. Feel free to use the framework and adjust any parameters or the stock universe and compare the results!

## Disclaimer
This strategy is provided for educational and research purposes only. Past performance is not indicative of future results.
