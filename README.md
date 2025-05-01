# Buffer Momentum ETF Strategy

## Overview
This repository contains a Python implementation of a momentum-based ETF strategy that uses a buffer mechanism to optimize portfolio rebalancing decisions. The strategy selects stocks from the S&P 500 based on risk-adjusted momentum and employs a buffering system to reduce unnecessary turnover and capture momentum efficiently.

## Strategy Methodology

### Key Components
- **Risk-Adjusted Momentum**: Combines 12-month returns, 1-month returns, and volatility to identify stocks with the strongest momentum characteristics
- **Buffer-Based Rebalancing**: Requires new candidates to exceed existing holdings' momentum by a significant margin (optimally 40%)
- **Quarterly Rebalancing**: Portfolio updates occur every 3 months to balance capturing new trends vs. minimizing transaction costs
- **Rank-Based Weighting**: Holdings are weighted according to their momentum rank, with stronger performers receiving higher allocations

### Key Parameters
- **Momentum Lookback**: 12 months for trend, 1 month for mean reversion
- **Holding Buffer**: 40% (optimal based on backtesting)
- **Rebalance Frequency**: Quarterly (every 3 months)
- **Portfolio Size**: 10 stocks
- **Transaction Cost**: 0.1% per trade (one-way)

## Performance
The strategy has been backtested from 2010 to 2025 and shows impressive performance:
- **~5600% cumulative return** (~56x initial investment)
- Effective at capturing momentum while minimizing unnecessary turnover
- Optimal parameter combination found through extensive testing

## Implementation

### Data Requirements
- Historical price data for S&P 500 constituents
- Monthly data frequency
- Minimum of 12 months of historical data to start

### Key Functions
- `download_batch()`: Fetches historical price data from Yahoo Finance (yfinance pacakge)
- Risk-adjusted momentum calculation pipeline
- Portfolio rebalancing logic with buffer-based replacement
- Performance tracking and visualization

### Dependencies
- pandas
- numpy
- matplotlib
- yfinance
- psycopg2
- sqlalchemy

## How to Use

1. Ensure all dependencies are installed:
2. Place your S&P 500 tickers file (CSV format) in the same directory
3. Run the script:
4. 4. Results will be stored in PostgreSQL and visualized

## Key Innovations

The most important innovation in this strategy is the optimized replacement logic that:
1. Considers all potential replacements simultaneously
2. Calculates improvement ratios for each candidate-holding pair
3. Prioritizes the strongest improvements first
4. Only replaces holdings when candidates show substantially better momentum (40% buffer)

This approach reduces unnecessary turnover while still capturing significant momentum shifts.

## Future Improvements
- Sector-based constraints to improve diversification
- Dynamic buffer based on market conditions
- Alternative weighting schemes
- Additional risk management rules

## Contact
Feel free to reach out with any questions or suggestions for improvements! Try to find even more optimal parameters and beat my returns!
