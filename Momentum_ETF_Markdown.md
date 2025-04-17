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
p_load(tidyquant, quantmod, BatchGetSymbols, 
       xts, tibbletime, tidyverse, lubridate, 
       PerformanceAnalytics, TTR, PortfolioAnalytics, janitor, dygraphs)
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
    ## SPY | yahoo (1|1) | Not Cached | Saving cache - Got 100% of valid prices | Got it!

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

<div class="dygraphs html-widget html-fill-item" id="htmlwidget-0ddb532b0216c5c6ac82" style="width:672px;height:480px;"></div>
<script type="application/json" data-for="htmlwidget-0ddb532b0216c5c6ac82">{"x":{"attrs":{"title":"Momentum ETF vs SPY","labels":["month","Momentum ETF","SPY"],"legend":"auto","retainDateWindow":false,"axes":{"x":{"pixelsPerLabel":60,"drawAxis":true},"y":{"drawAxis":true}},"colors":["blue","red"],"series":{"Momentum ETF":{"axis":"y"},"SPY":{"axis":"y"}},"stackedGraph":false,"fillGraph":false,"fillAlpha":0.15,"stepPlot":false,"drawPoints":false,"pointSize":1,"drawGapEdgePoints":false,"connectSeparatedPoints":false,"strokeWidth":1,"strokeBorderColor":"white","colorValue":0.5,"colorSaturation":1,"includeZero":false,"drawAxesAtZero":false,"logscale":false,"axisTickSize":3,"axisLineColor":"black","axisLineWidth":0.3,"axisLabelColor":"black","axisLabelFontSize":14,"axisLabelWidth":60,"drawGrid":true,"gridLineWidth":0.3,"rightGap":5,"digitsAfterDecimal":2,"labelsKMB":false,"labelsKMG2":false,"labelsUTC":false,"maxNumberWidth":6,"animatedZooms":false,"mobileDisableYTouch":true,"disableZoom":false,"showRangeSelector":true,"rangeSelectorHeight":40,"rangeSelectorPlotFillColor":" #A7B1C4","rangeSelectorPlotStrokeColor":"#808FAB","interactionModel":"Dygraph.Interaction.defaultModel"},"scale":"monthly","annotations":[],"shadings":[],"events":[],"format":"date","data":[["2014-02-03T00:00:00.000Z","2014-03-03T00:00:00.000Z","2014-04-01T00:00:00.000Z","2014-05-01T00:00:00.000Z","2014-06-02T00:00:00.000Z","2014-07-01T00:00:00.000Z","2014-08-01T00:00:00.000Z","2014-09-02T00:00:00.000Z","2014-10-01T00:00:00.000Z","2014-11-03T00:00:00.000Z","2014-12-01T00:00:00.000Z","2015-01-02T00:00:00.000Z","2015-02-02T00:00:00.000Z","2015-03-02T00:00:00.000Z","2015-04-01T00:00:00.000Z","2015-05-01T00:00:00.000Z","2015-06-01T00:00:00.000Z","2015-07-01T00:00:00.000Z","2015-08-03T00:00:00.000Z","2015-09-01T00:00:00.000Z","2015-10-01T00:00:00.000Z","2015-11-02T00:00:00.000Z","2015-12-01T00:00:00.000Z","2016-01-04T00:00:00.000Z","2016-02-01T00:00:00.000Z","2016-03-01T00:00:00.000Z","2016-04-01T00:00:00.000Z","2016-05-02T00:00:00.000Z","2016-06-01T00:00:00.000Z","2016-07-01T00:00:00.000Z","2016-08-01T00:00:00.000Z","2016-09-01T00:00:00.000Z","2016-10-03T00:00:00.000Z","2016-11-01T00:00:00.000Z","2016-12-01T00:00:00.000Z","2017-01-03T00:00:00.000Z","2017-02-01T00:00:00.000Z","2017-03-01T00:00:00.000Z","2017-04-03T00:00:00.000Z","2017-05-01T00:00:00.000Z","2017-06-01T00:00:00.000Z","2017-07-03T00:00:00.000Z","2017-08-01T00:00:00.000Z","2017-09-01T00:00:00.000Z","2017-10-02T00:00:00.000Z","2017-11-01T00:00:00.000Z","2017-12-01T00:00:00.000Z","2018-01-02T00:00:00.000Z","2018-02-01T00:00:00.000Z","2018-03-01T00:00:00.000Z","2018-04-02T00:00:00.000Z","2018-05-01T00:00:00.000Z","2018-06-01T00:00:00.000Z","2018-07-02T00:00:00.000Z","2018-08-01T00:00:00.000Z","2018-09-04T00:00:00.000Z","2018-10-01T00:00:00.000Z","2018-11-01T00:00:00.000Z","2018-12-03T00:00:00.000Z","2019-01-02T00:00:00.000Z","2019-02-01T00:00:00.000Z","2019-03-01T00:00:00.000Z","2019-04-01T00:00:00.000Z","2019-05-01T00:00:00.000Z","2019-06-03T00:00:00.000Z","2019-07-01T00:00:00.000Z","2019-08-01T00:00:00.000Z","2019-09-03T00:00:00.000Z","2019-10-01T00:00:00.000Z","2019-11-01T00:00:00.000Z","2019-12-02T00:00:00.000Z","2020-01-02T00:00:00.000Z","2020-02-03T00:00:00.000Z","2020-03-02T00:00:00.000Z","2020-04-01T00:00:00.000Z","2020-05-01T00:00:00.000Z","2020-06-01T00:00:00.000Z","2020-07-01T00:00:00.000Z","2020-08-03T00:00:00.000Z","2020-09-01T00:00:00.000Z","2020-10-01T00:00:00.000Z","2020-11-02T00:00:00.000Z","2020-12-01T00:00:00.000Z","2021-01-04T00:00:00.000Z","2021-02-01T00:00:00.000Z","2021-03-01T00:00:00.000Z","2021-04-01T00:00:00.000Z","2021-05-03T00:00:00.000Z","2021-06-01T00:00:00.000Z","2021-07-01T00:00:00.000Z","2021-08-02T00:00:00.000Z","2021-09-01T00:00:00.000Z","2021-10-01T00:00:00.000Z","2021-11-01T00:00:00.000Z","2021-12-01T00:00:00.000Z","2022-01-03T00:00:00.000Z","2022-02-01T00:00:00.000Z","2022-03-01T00:00:00.000Z","2022-04-01T00:00:00.000Z","2022-05-02T00:00:00.000Z","2022-06-01T00:00:00.000Z","2022-07-01T00:00:00.000Z","2022-08-01T00:00:00.000Z","2022-09-01T00:00:00.000Z","2022-10-03T00:00:00.000Z","2022-11-01T00:00:00.000Z","2022-12-01T00:00:00.000Z","2023-01-03T00:00:00.000Z","2023-02-01T00:00:00.000Z","2023-03-01T00:00:00.000Z","2023-04-03T00:00:00.000Z","2023-05-01T00:00:00.000Z","2023-06-01T00:00:00.000Z","2023-07-03T00:00:00.000Z","2023-08-01T00:00:00.000Z","2023-09-01T00:00:00.000Z","2023-10-02T00:00:00.000Z","2023-11-01T00:00:00.000Z","2023-12-01T00:00:00.000Z"],[104.7219877993697,105.7649149499202,105.7805254057516,107.8989737109847,111.4597788812529,107.8067208737739,112.3896498449458,106.965297128741,112.8890452814184,113.745815249279,114.6357995160706,112.8919328059987,118.2682780932629,119.4963612020301,118.9907657754968,120.2677841730757,118.8105015095349,119.0467544862241,112.7769028261978,107.8566539849097,114.8164953028869,116.4139061822359,111.1251754335916,102.50711573613,103.9907748023476,113.1609329058756,116.0350155440439,117.548307984567,117.0251819401072,123.0268018990228,124.8452252242503,124.8770852656534,120.0193352998442,129.7534016552194,133.3305194805115,136.3150192428858,138.6579649713786,139.0775436688771,140.5802798962285,139.5134294910106,142.8795941375176,144.5500937935533,142.987712128894,149.9202344577851,152.7042653822312,158.1266339782513,159.2408441955558,162.2524779058899,155.2735112923899,156.1267689964307,157.7225530576484,164.8082418144799,166.2795037092901,171.5829824971292,176.3865459591548,174.2363922995008,161.6182667803033,166.7830956927188,150.4523169511287,166.8267085841619,172.3710374506034,171.8194523874492,178.5415740125704,167.9024861317226,179.3536080427887,182.2140846996111,177.9878940422116,185.5534401351574,188.8022358962355,194.0648066248111,198.9773107682192,193.9070474502758,175.1645414809571,129.1578477917508,154.624471432985,164.7320780975999,167.6226286313596,173.6867296482865,183.4257760747726,176.7939312800173,178.1576057003042,213.00381036051,224.8024590661932,244.9434871705136,266.088610540368,280.764395320503,295.3953145746171,302.4790107599986,305.3420041606774,304.8039425837624,312.194311962099,307.6417205181739,322.8607171661951,311.6347558749846,327.8375234033547,311.6485445536019,314.2138423267825,321.1775567967247,302.2521785323149,305.2247365684123,275.5026152433671,306.0234292640077,294.7897511718134,263.0626109767783,291.0619987996764,306.6877538070912,289.0649150619748,318.4879537503541,311.2165674604953,301.2738208286983,300.8166027267083,293.6931616496109,321.6882320765549,338.1889863849519,332.1320543732498,316.4400290058995,301.8258655370378,334.3643452381216,362.4774365112097],[104.5516124274464,105.4188631887481,106.1517245836228,108.6150849014959,110.8574370958639,109.3677925194124,113.6838457060393,112.1154103732998,114.7558336065143,117.9083811204465,117.6092910343333,114.1246215641097,120.5389958999414,118.6458288273687,119.812576188806,121.3529307836914,118.8879861034746,121.5735740869201,114.1636260633861,111.2506403898783,120.7135571905296,121.1548229501141,119.0609526082827,113.1333423322336,113.0398441070842,120.6436055083693,121.1191018333262,123.1795025206476,123.6076660205874,128.1158127179476,128.2692352793716,128.2766567661394,126.052733207453,130.6963116148875,133.3457406972133,135.7319467383244,141.0650771620944,141.2413895900099,142.643373065736,144.6564721983497,145.5786440464198,148.5708999542933,149.0043794068336,152.006756471521,155.5886453337143,160.3443382238361,162.288976225705,171.4354270716451,165.2019827460857,160.6737605100814,161.504133153147,165.4301413470104,166.3815092406371,172.5453562808763,178.0529958780521,179.1115792392247,166.7342800245785,169.8270282834738,154.8740353761011,167.274120222874,172.6964335690942,175.8222970856593,183.00508715817,171.3346220175292,183.2572300864156,186.0279462755238,182.9132358359831,186.4723182191768,190.5942036312127,197.4934136910796,203.2316197030051,203.1496081048473,187.0671003511135,163.7078039747691,184.4960138173074,193.2863479561987,196.7141575804703,208.2990775779895,222.8377076153106,214.4938926479855,209.1457316288235,231.8959449183923,240.4873581956615,238.0366623531013,244.6553985843932,255.7625499478047,269.2950682031507,271.063320813049,277.142560819449,283.908288351862,292.3573591806764,278.7319264126234,298.2887114704187,295.8920090287808,309.5763759066436,293.2489382427025,284.5930872811666,295.29103537558,269.3736400672729,269.9816808104014,247.7188674102399,270.5307040562282,259.4925141485122,235.5034754343055,254.6441777550316,268.8001842863569,253.3097277242042,269.239761449233,262.4702814796902,272.2021437881791,276.5505304747406,277.8271512800241,295.8303231317424,305.5137790012577,300.5485958849316,286.2922116603858,280.0772168173809,305.6605409867764,319.6155628160582]],"fixedtz":false,"tzone":"UTC"},"evals":["attrs.interactionModel"],"jsHooks":[]}</script>

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

    ## [1] -0.2486612

``` r
### Beta Since Inception
beta_inc <- cov(monthly_returns_inc, spy_returns_inc) / var(spy_returns_inc)
beta_inc
```

    ## [1] 1.177148

### Alpha

``` r
### Alpha 3-Year
alpha_3y <- mean(monthly_returns_3y) - (rf_monthly + beta_3y * (mean(spy_returns_3y) - rf_monthly))
alpha_3y_annualized_pct <- alpha_3y * 12 * 100
alpha_3y_annualized_pct
```

    ## [1] 13.26433

``` r
### Alpha Since Inception
alpha_inc <- mean(monthly_returns_inc) - (rf_monthly + beta_inc * (mean(spy_returns_inc) - rf_monthly))
alpha_inc_annualized_pct <- alpha_inc * 12 * 100
alpha_inc_annualized_pct
```

    ## [1] 0.3322655
