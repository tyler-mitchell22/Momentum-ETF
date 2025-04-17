Momentum ETF Analysis
================
Tyler Mitchell
2025-04-17

- [Package Download](#package-download)
- [Market Parameters](#market-parameters)
- [Data/Stock Collection](#datastock-collection)
- [Data Aggregation](#data-aggregation)
- [Initial Investment](#initial-investment)
- [Monthly Rebalance of the ETF](#monthly-rebalance-of-the-etf)
- [Convert ETF NAV to a timeseries](#convert-etf-nav-to-a-timeseries)
- [Collect S&P 500 Data](#collect-sp-500-data)
- [S&P Initial Investment](#sp-initial-investment)
- [Rebalance to Monthly Returns](#rebalance-to-monthly-returns)
- [Convert to timeseries](#convert-to-timeseries)
- [Plot Performance Comparison](#plot-performance-comparison)
- [Key Performance Metrics](#key-performance-metrics)
  - [Compound Annual Growth](#compound-annual-growth)
  - [Total Return Rate](#total-return-rate)
  - [Maximum Drawdown](#maximum-drawdown)
  - [Sharpe Ratio Assuming 0 Risk Free
    Rate](#sharpe-ratio-assuming-0-risk-free-rate)
  - [Volatility](#volatility)
  - [Sortino Ratio Assuming 0 Risk Free
    Rate](#sortino-ratio-assuming-0-risk-free-rate)
  - [Calmar Ratio](#calmar-ratio)
  - [Beta and Alpha](#beta-and-alpha)
  - [Alpha](#alpha)

# Package Download

``` r
library(pacman)
p_load(tidyquant, BatchGetSymbols, 
       xts, tidyverse, lubridate, TTR, janitor, dygraphs)
options(scipen = 999)

nyse <- tq_exchange("NYSE") |> clean_names()
```

    ## Getting data...

``` r
colnames(nyse)
```

    ## [1] "symbol"          "company"         "last_sale_price" "market_cap"     
    ## [5] "country"         "ipo_year"        "industry"

# Market Parameters

``` r
nyse <- nyse |> filter(market_cap > 1000000000, last_sale_price > 5, 
                       ipo_year <= 2012, 
                       country == "United States", 
                       !grepl("\\^|\\.|-", symbol))

tickers <- nyse$symbol
```

# Data/Stock Collection

``` r
nyse_monthly_data <- BatchGetSymbols(
  tickers = tickers, 
  freq.data = "monthly", thresh.bad.data = 0.5,
  first.date = as.Date("2013-01-01"),
  last.date = as.Date("2023-12-31"), 
  cache.folder = file.path(tempdir(), "BGS_Cache"))

price_data <- nyse_monthly_data$df.tickers |> clean_names()

summary(price_data$price_adjusted)
```

# Data Aggregation

``` r
price_data <- price_data |> 
  group_by(ticker) |> 
  arrange(ref_date) |> 
  mutate(monthly_return = price_adjusted / lag(price_adjusted) - 1,
    return_12m = price_adjusted / lag(price_adjusted, 12) - 1,
    return_1m = price_adjusted / lag(price_adjusted, 1) - 1, 
    momentum = return_12m - return_1m,
    volatility_12m = rollapply(monthly_return, width = 12, FUN = sd, align = "right", fill = NA),
    risk_adjusted_momentum = momentum / volatility_12m)

rebalance_dates <- price_data |> filter(!is.na(momentum)) |> 
  pull(ref_date) |> 
  unique() |> 
  sort()
```

# Initial Investment

``` r
etf_nav <- c(100)
nav_dates <- c()
selected_tickers <- list()
```

# Monthly Rebalance of the ETF

``` r
for (i in 1:(length(rebalance_dates) - 1)) {
  current_date <- rebalance_dates[i]
  next_date <- rebalance_dates[i + 1]
  current_data <- filter(price_data, ref_date == current_date, !is.na(risk_adjusted_momentum))
  current_data <- slice_max(current_data, order_by = risk_adjusted_momentum, n = 10)
  top_tickers <- current_data$ticker
  next_returns <- filter(price_data, ref_date == next_date, ticker %in% top_tickers)
  avg_return <- mean(next_returns$monthly_return, na.rm = TRUE)
  
  if (!is.na(avg_return)) {
    previous_nav <- last(etf_nav)
    new_nav <- previous_nav * (1 + avg_return)
    etf_nav <- append(etf_nav, new_nav)
    nav_dates <- append(nav_dates, next_date)
    selected_tickers <- append(selected_tickers, list(top_tickers))
  }
}
```

# Convert ETF NAV to a timeseries

``` r
etf_xts <- xts(etf_nav[-1], order.by = nav_dates)
```

# Collect S&P 500 Data

``` r
spy <- BatchGetSymbols(
  tickers = "SPY",
  freq.data = "monthly",
  first.date = as.Date("2013-01-01"), thresh.bad.data = 0.5,
  last.date = as.Date("2023-12-31"), 
  cache.folder = file.path(tempdir(), "BGS_Cache"))
```

    ## 
    ## Running BatchGetSymbols for:
    ##    tickers =SPY
    ##    Downloading data for benchmark ticker
    ## ^GSPC | yahoo (1|1) | Found cache file
    ## SPY | yahoo (1|1) | Not Cached | Saving cache - Got 100% of valid prices | Looking good!

``` r
spy_price_data <- spy$df.tickers |> clean_names()

spy_price_data <- spy_price_data |> arrange(ref_date) |> 
  mutate(monthly_returns = price_adjusted / lag(price_adjusted) - 1) |> 
  filter(!is.na(monthly_returns), ref_date %in% index(etf_xts))
```

# S&P Initial Investment

``` r
spy_nav <- c(100)
```

# Rebalance to Monthly Returns

``` r
for (i in 1:nrow(spy_price_data)) {
  returns <- spy_price_data$monthly_returns[i]
  spy_previous_nav <- last(spy_nav)
  spy_new_nav <- spy_previous_nav * (1 + returns)
  spy_nav <- append(spy_nav, spy_new_nav)
}
```

# Convert to timeseries

``` r
spy_returns <- xts(spy_price_data$monthly_returns, order.by = spy_price_data$ref_date)

spy_xts <- 100 * cumprod(1 + spy_returns)
```

# Plot Performance Comparison

``` r
colnames(etf_xts) <- "ETF"
colnames(spy_xts) <- "SPY"
combined_nav <- cbind(etf_xts, spy_xts)

dygraph(combined_nav, main = "Momentum ETF vs SPY")  |> 
  dySeries("ETF", label = "Momentum ETF", color = "blue") |> 
  dySeries("SPY", label = "SPY", color = "red") |> 
  dyOptions(stackedGraph = FALSE, drawGrid = TRUE) |> 
  dyRangeSelector()
```

<div class="dygraphs html-widget html-fill-item" id="htmlwidget-7cf3264c9609147a9473" style="width:672px;height:480px;"></div>
<script type="application/json" data-for="htmlwidget-7cf3264c9609147a9473">{"x":{"attrs":{"title":"Momentum ETF vs SPY","labels":["month","Momentum ETF","SPY"],"legend":"auto","retainDateWindow":false,"axes":{"x":{"pixelsPerLabel":60,"drawAxis":true},"y":{"drawAxis":true}},"colors":["blue","red"],"series":{"Momentum ETF":{"axis":"y"},"SPY":{"axis":"y"}},"stackedGraph":false,"fillGraph":false,"fillAlpha":0.15,"stepPlot":false,"drawPoints":false,"pointSize":1,"drawGapEdgePoints":false,"connectSeparatedPoints":false,"strokeWidth":1,"strokeBorderColor":"white","colorValue":0.5,"colorSaturation":1,"includeZero":false,"drawAxesAtZero":false,"logscale":false,"axisTickSize":3,"axisLineColor":"black","axisLineWidth":0.3,"axisLabelColor":"black","axisLabelFontSize":14,"axisLabelWidth":60,"drawGrid":true,"gridLineWidth":0.3,"rightGap":5,"digitsAfterDecimal":2,"labelsKMB":false,"labelsKMG2":false,"labelsUTC":false,"maxNumberWidth":6,"animatedZooms":false,"mobileDisableYTouch":true,"disableZoom":false,"showRangeSelector":true,"rangeSelectorHeight":40,"rangeSelectorPlotFillColor":" #A7B1C4","rangeSelectorPlotStrokeColor":"#808FAB","interactionModel":"Dygraph.Interaction.defaultModel"},"scale":"monthly","annotations":[],"shadings":[],"events":[],"format":"date","data":[["2014-02-03T00:00:00.000Z","2014-03-03T00:00:00.000Z","2014-04-01T00:00:00.000Z","2014-05-01T00:00:00.000Z","2014-06-02T00:00:00.000Z","2014-07-01T00:00:00.000Z","2014-08-01T00:00:00.000Z","2014-09-02T00:00:00.000Z","2014-10-01T00:00:00.000Z","2014-11-03T00:00:00.000Z","2014-12-01T00:00:00.000Z","2015-01-02T00:00:00.000Z","2015-02-02T00:00:00.000Z","2015-03-02T00:00:00.000Z","2015-04-01T00:00:00.000Z","2015-05-01T00:00:00.000Z","2015-06-01T00:00:00.000Z","2015-07-01T00:00:00.000Z","2015-08-03T00:00:00.000Z","2015-09-01T00:00:00.000Z","2015-10-01T00:00:00.000Z","2015-11-02T00:00:00.000Z","2015-12-01T00:00:00.000Z","2016-01-04T00:00:00.000Z","2016-02-01T00:00:00.000Z","2016-03-01T00:00:00.000Z","2016-04-01T00:00:00.000Z","2016-05-02T00:00:00.000Z","2016-06-01T00:00:00.000Z","2016-07-01T00:00:00.000Z","2016-08-01T00:00:00.000Z","2016-09-01T00:00:00.000Z","2016-10-03T00:00:00.000Z","2016-11-01T00:00:00.000Z","2016-12-01T00:00:00.000Z","2017-01-03T00:00:00.000Z","2017-02-01T00:00:00.000Z","2017-03-01T00:00:00.000Z","2017-04-03T00:00:00.000Z","2017-05-01T00:00:00.000Z","2017-06-01T00:00:00.000Z","2017-07-03T00:00:00.000Z","2017-08-01T00:00:00.000Z","2017-09-01T00:00:00.000Z","2017-10-02T00:00:00.000Z","2017-11-01T00:00:00.000Z","2017-12-01T00:00:00.000Z","2018-01-02T00:00:00.000Z","2018-02-01T00:00:00.000Z","2018-03-01T00:00:00.000Z","2018-04-02T00:00:00.000Z","2018-05-01T00:00:00.000Z","2018-06-01T00:00:00.000Z","2018-07-02T00:00:00.000Z","2018-08-01T00:00:00.000Z","2018-09-04T00:00:00.000Z","2018-10-01T00:00:00.000Z","2018-11-01T00:00:00.000Z","2018-12-03T00:00:00.000Z","2019-01-02T00:00:00.000Z","2019-02-01T00:00:00.000Z","2019-03-01T00:00:00.000Z","2019-04-01T00:00:00.000Z","2019-05-01T00:00:00.000Z","2019-06-03T00:00:00.000Z","2019-07-01T00:00:00.000Z","2019-08-01T00:00:00.000Z","2019-09-03T00:00:00.000Z","2019-10-01T00:00:00.000Z","2019-11-01T00:00:00.000Z","2019-12-02T00:00:00.000Z","2020-01-02T00:00:00.000Z","2020-02-03T00:00:00.000Z","2020-03-02T00:00:00.000Z","2020-04-01T00:00:00.000Z","2020-05-01T00:00:00.000Z","2020-06-01T00:00:00.000Z","2020-07-01T00:00:00.000Z","2020-08-03T00:00:00.000Z","2020-09-01T00:00:00.000Z","2020-10-01T00:00:00.000Z","2020-11-02T00:00:00.000Z","2020-12-01T00:00:00.000Z","2021-01-04T00:00:00.000Z","2021-02-01T00:00:00.000Z","2021-03-01T00:00:00.000Z","2021-04-01T00:00:00.000Z","2021-05-03T00:00:00.000Z","2021-06-01T00:00:00.000Z","2021-07-01T00:00:00.000Z","2021-08-02T00:00:00.000Z","2021-09-01T00:00:00.000Z","2021-10-01T00:00:00.000Z","2021-11-01T00:00:00.000Z","2021-12-01T00:00:00.000Z","2022-01-03T00:00:00.000Z","2022-02-01T00:00:00.000Z","2022-03-01T00:00:00.000Z","2022-04-01T00:00:00.000Z","2022-05-02T00:00:00.000Z","2022-06-01T00:00:00.000Z","2022-07-01T00:00:00.000Z","2022-08-01T00:00:00.000Z","2022-09-01T00:00:00.000Z","2022-10-03T00:00:00.000Z","2022-11-01T00:00:00.000Z","2022-12-01T00:00:00.000Z","2023-01-03T00:00:00.000Z","2023-02-01T00:00:00.000Z","2023-03-01T00:00:00.000Z","2023-04-03T00:00:00.000Z","2023-05-01T00:00:00.000Z","2023-06-01T00:00:00.000Z","2023-07-03T00:00:00.000Z","2023-08-01T00:00:00.000Z","2023-09-01T00:00:00.000Z","2023-10-02T00:00:00.000Z","2023-11-01T00:00:00.000Z","2023-12-01T00:00:00.000Z"],[104.7219884077536,105.7649135067241,105.7805241111393,107.8989745221891,111.4597746593187,107.8067198365425,112.3896485651474,106.965299090755,112.8890432902067,113.7458158110705,114.6357985239605,112.8919298629731,118.2682780777118,119.496357222057,118.9907625895328,120.2677826095668,118.8104971729588,119.0467514994879,112.7769010426203,107.8566535307596,114.8164963231553,116.4139029386048,111.1251759350518,102.5071144514032,103.9907748509394,113.1609305227264,116.0350131850949,117.5483069657798,117.0251804913487,123.0267987298883,124.8452198362819,124.8770825064183,120.019332856104,129.7533993206534,133.3305156333867,136.315016166454,138.657958274,139.0775368562168,140.580276037627,139.5134256986945,142.8795894349196,144.5500879930245,142.9877062670879,149.9202324142867,152.7042639178062,158.1266314803733,159.2408433773425,162.2524727458154,155.2735041509677,156.1267650700845,157.7225445598524,164.8082419503668,166.2794986232637,171.5829790841312,176.386544793793,174.2363920241186,161.6182613739217,166.78309088204,150.4523107217966,166.8267055288554,172.3710354160159,171.8194448157114,178.5415713616245,167.9024819845471,179.3536042704093,182.2140803930451,177.9878903824238,185.5534373411526,188.8022314659479,194.0648021911131,198.9773084117866,193.9070434537822,175.1645389251583,129.1578463932435,154.6244694530183,164.7320690618445,167.6226255987951,173.6867257088418,183.4257732083168,176.7939280338489,178.1576026381591,213.0038059519174,224.802453374879,244.9434782281809,266.0886021070662,280.7643901377309,295.3953073024083,302.4790042022042,305.3419921472463,304.8039343607091,312.1943085201934,307.6417086310935,322.8607022534542,311.634745617988,327.8375134292635,311.6485358829277,314.2138308441912,321.1775443735301,302.2521728473528,305.2247264636748,275.50260717615,306.0234194021634,294.7897406664962,263.0626050333489,291.0619889194467,306.6877467639191,289.0649044254292,318.4879454919409,311.216560917274,301.2738099377551,300.816593709786,293.6931526523487,321.6882238229764,338.1889749437073,332.1320453203181,316.4400204870284,301.8258560533934,334.3643335371104,362.4774245175882],[104.5516024784433,105.4188950238926,106.1516731076562,108.6150962229215,110.8574590744571,109.3678143427335,113.6838367089289,112.1154429064547,114.7558455680169,117.9084142574281,117.6093345635769,114.1246751535685,120.5389980408107,118.6458411943425,119.8126095242701,121.3528913156504,118.8879984956895,121.573576335628,114.1636483865978,111.2506519860192,120.7135801964693,121.1548043082986,119.0609650185266,113.1333437011792,113.0398871600344,120.6436180835801,121.1191248815376,123.1795466304987,123.6076789047552,128.115784378269,128.26922780256,128.2766597135389,126.0527567699184,130.6963252379358,133.3457858667359,135.7319504628215,141.0650918659246,141.2414147356554,142.6433566637662,144.6564872765267,145.5786696441564,148.5709258639258,149.0044053616495,152.0067618924126,155.5886511279619,160.3443966309783,162.2890035652326,171.4355074817351,165.2019895423736,160.6737564109353,161.5041291405543,165.4301481670802,166.3815057364347,172.5454055363464,178.0530561310452,179.1115770619332,166.7342348633967,169.8270251384129,154.8740202489868,167.2741063882748,172.6964724168742,175.8223362592618,183.0051479273412,171.3346294530489,183.2572491881185,186.0279865129058,182.9132757487052,186.4723168091273,190.5942443445556,197.4934134298072,203.2316617336012,203.1496292800199,187.0671198499359,163.7077897684371,184.4960538950094,193.2863264095303,196.7141780848476,208.2991201367889,222.8377308426599,214.4938733118722,209.1457325821232,231.8959690899214,240.4873207220858,238.0366454709539,244.6554240858959,255.7625974539295,269.2951171198292,271.0633282202899,277.1425897072308,283.9083387917403,292.3574105012382,278.7319137723214,298.2887425623563,295.8920607177748,309.5764081751439,293.2489896561976,284.5931377924244,295.2911078488069,269.3736889921413,269.9817089517734,247.7189140779367,270.5307114079523,259.4925620434275,235.5034999818633,254.6441834508334,268.8002539983266,253.3097541277857,269.2397478195213,262.4703505318665,272.2021513141246,276.5505801476878,277.8271593922888,295.8303331205564,305.5138108462958,300.5486272124269,286.2922415018758,280.0772460110549,305.6605728471121,319.6155961309869]],"fixedtz":false,"tzone":"UTC"},"evals":["attrs.interactionModel"],"jsHooks":[]}</script>

# Key Performance Metrics

### Compound Annual Growth

``` r
etf_nav <- etf_nav[order(nav_dates)]

### 3-Year
start_date_3y <- nav_dates[length(nav_dates) - 36]
end_date_3y <- nav_dates[length(nav_dates)]
n_years_3y <- as.numeric(difftime(end_date_3y, start_date_3y, units = "days")) / 365

cagr_3y <- (last(etf_nav) / etf_nav[length(etf_nav) - 36]) ^ (1 / n_years_3y) - 1
cagr_3y
```

    ## [1] 0.1621911

``` r
### Since Inception
start_date_inc <- nav_dates[1]
end_date_inc <- nav_dates[length(nav_dates)]
n_years_inc <- as.numeric(difftime(end_date_inc, start_date_inc, units = "days")) / 365

cagr_inc <- (last(etf_nav) / first(etf_nav)) ^ (1 / n_years_inc) - 1
cagr_inc
```

    ## [1] 0.1306491

### Total Return Rate

``` r
### 3-Year
total_return_3y <- ((last(etf_nav) - etf_nav[length(etf_nav) - 36]) / etf_nav[length(etf_nav) - 36]) * 100
total_return_3y
```

    ## [1] 56.97576

``` r
### Since Inception
total_return_inc <- ((last(etf_nav) - first(etf_nav)) / first(etf_nav)) * 100
total_return_inc
```

    ## [1] 234.3643

### Maximum Drawdown

``` r
### 3-Year
dd_3y <- tail(etf_xts, 36)
max_dd_3y <- max(dd_3y)
drawdown_3y <- (dd_3y - cummax(dd_3y))/cummax(dd_3y)

max_drawdown_3y <- min(drawdown_3y) * 100
max_drawdown_3y
```

    ## [1] -19.75824

``` r
### Since Inception
run_max_nav_inc <- cummax(etf_xts)
drawdown_inc <- (etf_xts - run_max_nav_inc) / run_max_nav_inc 
max_dd_inc <- min(drawdown_inc) * 100
max_dd_inc
```

    ## [1] -35.08916

### Sharpe Ratio Assuming 0 Risk Free Rate

``` r
### 3-Year
monthly_returns_3y <- diff(tail(etf_nav, 36)) / head(tail(etf_nav, 36), - 1)

sharpe_ratio_3y <- mean(monthly_returns_3y) / sd(monthly_returns_3y)
sharpe_ratio_3y
```

    ## [1] 0.2190439

``` r
### Since Inception
monthly_returns_inc <- diff(etf_nav) / head(etf_nav, -1) 

sharpe_ratio_inc <- mean(monthly_returns_inc) / sd(monthly_returns_inc)
sharpe_ratio_inc
```

    ## [1] 0.2074703

### Volatility

``` r
### 3-Year
volatility_3y <- sd(monthly_returns_3y) * sqrt(12) * 100
volatility_3y
```

    ## [1] 20.77448

``` r
### Since Inception
volatility_inc <- sd(monthly_returns_inc) * sqrt(12) * 100
volatility_inc
```

    ## [1] 19.97024

### Sortino Ratio Assuming 0 Risk Free Rate

``` r
### 3-Year
negatives_3y <- monthly_returns_3y[monthly_returns_3y < 0]
sortino_3y <- (mean(monthly_returns_3y) * 12) / (sd(negatives_3y) * sqrt(12))
sortino_3y 
```

    ## [1] 1.520089

``` r
### Since Inception
negatives_inc <- monthly_returns_inc[monthly_returns_inc < 0]
sortino_inception <- (mean(monthly_returns_inc) * 12) / (sd(negatives_inc) * sqrt(12))
sortino_inception 
```

    ## [1] 0.9121652

### Calmar Ratio

``` r
### 3-Year
calmar_3y <- cagr_3y / abs(max_drawdown_3y)
calmar_3y
```

    ## [1] 0.008208782

``` r
### Since Inception
calmar_inc <- cagr_inc / abs(max_dd_inc)
calmar_inc
```

    ## [1] 0.003723346

### Beta and Alpha

``` r
spy_returns_3y <- as.numeric(tail(spy_returns, 36))
spy_returns_3y <- diff(tail(spy_nav, 36)) / head(tail(spy_nav, 36), -1)

spy_returns_inc <- as.numeric(spy_returns)
spy_returns_inc <- spy_returns_inc[1:length(monthly_returns_inc)]

### Risk Free Rate - Based on current rate of 3 month T-bills
rf_annual <- 0.0433
rf_monthly <- 0.0433 / 12

### Beta 3-Year
beta_3y <- cov(monthly_returns_3y, spy_returns_3y) / var(spy_returns_3y)
beta_3y
```

    ## [1] -0.2486597

``` r
### Beta Since Inception
beta_inc <- cov(monthly_returns_inc, spy_returns_inc) / var(spy_returns_inc)
beta_inc
```

    ## [1] 1.177147

### Alpha

``` r
### Alpha 3-Year
alpha_3y <- mean(monthly_returns_3y) - (rf_monthly + beta_3y * (mean(spy_returns_3y) - rf_monthly))
alpha_3y_annualized_pct <- alpha_3y * 12 * 100
alpha_3y_annualized_pct
```

    ## [1] 13.26432

``` r
### Alpha Since Inception
alpha_inc <- mean(monthly_returns_inc) - (rf_monthly + beta_inc * (mean(spy_returns_inc) - rf_monthly))
alpha_inc_annualized_pct <- alpha_inc * 12 * 100
alpha_inc_annualized_pct
```

    ## [1] 0.332266
