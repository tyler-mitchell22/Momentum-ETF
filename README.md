# Momentum ETF Strategy

## Overview
This repository is an implementation of a momentum-based ETF strategy that selects stocks from the NYSE based on risk-adjusted momentum. The strategy shows comparable results with the S&P 500 (SPY) from the 2013-2020, and shows significant out performance post-pandemic.

## Strategy Methodology
The strategy follows these key steps:
1. Filters NYSE stocks with market cap > $1B, price > $5, and IPO year ≤ 2012
2. Calculates risk-adjusted momentum as (12-month return - 1-month return) / 12-month volatility
3. Rebalances monthly into the top 10 stocks with highest risk-adjusted momentum
4. Equally weights the selected stocks in the portfolio

## Performance Highlights
The strategy delivers exceptional performance metrics:

### 3-Year Performance
- **CAGR**: 16.22%
- **Total Return**: 56.98%
- **Beta**: -0.25 (negative correlation to market)
- **Alpha**: 13.26%
- **Maximum Drawdown**: -19.76%
- **Sharpe Ratio**: 0.22
- **Sortino Ratio**: 1.52
- **Calmar Ratio**: 0.01
- **Volatility**: 20.77%

### Since Inception (2013-2023)
- **Total Return**: 234.36%
- **CAGR**: 49.53%
- **Beta**: 1.18
- **Alpha**: 0.33%
- **Maximum Drawdown**: -35.09%
- **Sharpe Ratio**: 0.21
- **Sortino Ratio**: 0.91
- **Calmar Ratio**: 0.01
- **Volatility**: 19.97%

## Key Insights
- The strategy's recent negative 3-year beta of -0.25 is extremely rare and valuable, indicating counter-market movement
- Strong positive alpha, 13.26% over 3 years, shows significant risk-adjusted outperformance
- The strategy appears to have evolved substantially over time, with the inception beta of 1.18 showing initial market correlation
- Despite similar volatility profiles over both time periods, the strategy has achieved significantly higher returns in the recent 3-year window

## File Structure
- `momentum_etf.R`: Main script implementing the momentum strategy
- `momentum_etf.md`: Markdown with a table of contents
- `Momentum ETF vs SPY Performance.png`: Visualization of results from the entire period

## Requirements
The strategy implementation requires the following R packages:
- tidyquant
- BatchGetSymbols
- xts
- tidyverse
- janitor
- lubridate
- TTR
- dygraphs

## Usage
To run the strategy:
1. Install required packages using `install.packages()` or `p_load()` function from the pacman library
2. Run the R script code in your IDE of choice to see the full analysis with visualizations
3. Feel free to use the framework and adjust any parameters or the stock universe and compare the results!

## Disclaimer
This strategy is provided for educational and research purposes only. Past performance is not indicative of future results.
