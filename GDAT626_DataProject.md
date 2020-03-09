## EXPLORATORY DATA ANALYSIS

``` r
# Reading in the Data
read.csv("C:/Users/John.JVivs/Documents/COLLEGE/GRAD SCHOOL/GDAT626/PROJECT/MCD.csv") -> mcd

# Check for NAs
anyNA(mcd)
```

    ## [1] FALSE

``` r
# No NAs

# Historical data from 2014-2019 on McDonald's stock prices
str(mcd)
```

    ## 'data.frame':    1259 obs. of  7 variables:
    ##  $ Date     : Factor w/ 1259 levels "2014-12-08","2014-12-09",..: 1 2 3 4 5 6 7 8 9 10 ...
    ##  $ Open     : num  93.4 91.3 91 90.1 90.7 ...
    ##  $ High     : num  97.5 92 91.3 91.2 91.4 ...
    ##  $ Low      : num  92.2 91 89.5 90 90.4 ...
    ##  $ Close    : num  92.6 91.4 90 91 90.6 ...
    ##  $ Adj.Close: num  80.6 79.5 78.4 79.2 78.9 ...
    ##  $ Volume   : int  11792700 10066800 12020800 8972700 8691000 10121600 15106700 14010000 11277100 10284300 ...

``` r
# Convert into proper date format
mcd$Date <- as.Date(mcd$Date)

# Visualize the Data
ggplot(mcd, aes(Date, Close)) + 
  geom_line() + 
  ylab("Closing Stock Value") + 
  xlab("Time") + 
  ggtitle("Daily Closing Values of McDonald's Stock")
```

![](GDAT626_DataProject_files/figure-gfm/Exploratory%20Data%20Analysis-1.png)<!-- -->

``` r
# Maybe a positive, linear trend? Can't tell for sure if it's stationary.

# Stock prices are based on returns, and returns based on percentages. Important to normalize these values.

# Building a normalize() function
normalize <- function(x) {
  num <- x - min(x)
  denom <- max(x) - min(x)
  return (num/denom)
}

mcd_norm <- as.data.frame(lapply(mcd[5], normalize))
# Now we have normalized values (between 0 and 1) on the Close values (Going to focus on the closing values for this predictive model)

# Attaching date column to normalized data set
mcd$Date -> mcd_norm$Date

# Weekly moving averages
mcd_norm$cnt_ma = ma(mcd_norm$Close, order=7)

# Plotting the daily stock values versus the weekly moving average
ggplot() + 
  geom_line(data = mcd_norm, aes(x = Date, y = Close, colour = "Counts")) + 
  geom_line(data = mcd_norm, aes(x = Date, y = cnt_ma, colour = "Weekly Moving Average")) + 
  ylab("Closing Stock Value")
```

    ## Warning: Removed 6 rows containing missing values (geom_path).

![](GDAT626_DataProject_files/figure-gfm/Exploratory%20Data%20Analysis-2.png)<!-- -->

``` r
# Weekly moving average looks smoother. Be careful not to oversmooth; negatively impacts accuracy.
# Regardless, weekly still looks good to move on with.
```

## DECOMPOSING THE DATA

``` r
# Building a weekly moving average
count_ma = ts(na.omit(mcd_norm$cnt_ma), frequency = 252) # Frequency based on number of days the stock market is open
decomp = stl(count_ma, s.window = "periodic")
deseasonal_cnt <- seasadj(decomp) # Removes the seasonality; used later in ARIMA model
plot(decomp)
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-1.png)<!-- -->

``` r
# Check out the first graph; smoothed out since we're looking at the weekly moving average.
# Still not sold on it being stationary; let's test it.

# Let's look at the acf/pacf
acf(count_ma)
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-2.png)<!-- -->

``` r
pacf(count_ma)
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-3.png)<!-- -->

``` r
# Acf plot is looking rough... Let's apply diff to it, then try again.

# Differencing
diff(deseasonal_cnt, differences = 1) -> count_d1
plot(count_d1)
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-4.png)<!-- -->

``` r
# acf/pacf of the differenced values
acf(count_d1, lag.max = 20)
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-5.png)<!-- -->

``` r
pacf(count_d1, lag.max = 20)
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-6.png)<!-- -->

``` r
# Let's use the Dickey-Fuller test to screen for stationarity across the original, differenced, and weekly moving average time-series

# Original ts
adf.test(mcd_norm$Close)
```

    ## 
    ##  Augmented Dickey-Fuller Test
    ## 
    ## data:  mcd_norm$Close
    ## Dickey-Fuller = -2.4185, Lag order = 10, p-value = 0.4011
    ## alternative hypothesis: stationary

``` r
# This confirms my assumption that it wasn't stationary earlier. High p-value of 0.4011 cannot reject the null hypothesis of non-stationarity.

# Differenced ts
diff(mcd_norm$Close) -> diff_mcd
adf.test(diff_mcd)
```

    ## Warning in adf.test(diff_mcd): p-value smaller than printed p-value

    ## 
    ##  Augmented Dickey-Fuller Test
    ## 
    ## data:  diff_mcd
    ## Dickey-Fuller = -11.216, Lag order = 10, p-value = 0.01
    ## alternative hypothesis: stationary

``` r
# The differenced ts yields a p-value of 0.01, which allows us to reject the null hypothesis of non-stationarity.
# Therefore, the differenced ts is stationary. May be valuable to try an auto.arima.
plot(ts(diff_mcd))
```

![](GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-7.png)<!-- -->

``` r
# Weekly moving average
adf.test(count_ma)
```

    ## 
    ##  Augmented Dickey-Fuller Test
    ## 
    ## data:  count_ma
    ## Dickey-Fuller = -2.5844, Lag order = 10, p-value = 0.3309
    ## alternative hypothesis: stationary

``` r
# p-value of 0.3309; cannot reject the null; therefore, non-stationary (as expected from ugly looking acf plot)

# Differenced weekly moving average
adf.test(count_d1)
```

    ## Warning in adf.test(count_d1): p-value smaller than printed p-value

    ## 
    ##  Augmented Dickey-Fuller Test
    ## 
    ## data:  count_d1
    ## Dickey-Fuller = -8.4557, Lag order = 10, p-value = 0.01
    ## alternative hypothesis: stationary

``` r
# Differenced ts of the weekly moving average yields a p-value of 0.01, which can reject the null hypothesis of non-stationarity.
# Therefore, the differenced ts of the weekly moving average is stationary.
```

## BUILDING AN ARIMA MODEL

``` r
# Let's see what auto.arima suggests for the model
auto.arima(deseasonal_cnt, seasonal = FALSE) # Note: Using the non-seasonal data (don't want seasonality)
```

    ## Series: deseasonal_cnt 
    ## ARIMA(4,1,5) with drift 
    ## 
    ## Coefficients:
    ##          ar1      ar2     ar3      ar4      ma1     ma2      ma3     ma4
    ##       0.8154  -0.7739  0.4864  -0.2357  -0.3297  0.9602  -0.0823  0.4909
    ## s.e.  0.0537   0.0628  0.0588   0.0413   0.0478  0.0384   0.0596  0.0304
    ##          ma5  drift
    ##       0.4269  6e-04
    ## s.e.  0.0338  3e-04
    ## 
    ## sigma^2 estimated as 6.578e-06:  log likelihood=5695.67
    ## AIC=-11369.34   AICc=-11369.13   BIC=-11312.88

``` r
# Suggests (4,1,5)

# Taking a look at the auto arima
fit <- auto.arima(deseasonal_cnt, seasonal = FALSE)
tsdisplay(residuals(fit), lag.max = 45, main = '(1,1,1) Model Residuals')
```

![](GDAT626_DataProject_files/figure-gfm/BUILDING%20AN%20ARIMA%20MODEL-1.png)<!-- -->

``` r
# Not sold on entirely trusting the auto arima

# ARIMA of deseasonal_cnt 4,1,5 (Suggested by auto.arima at one point)
fit3 <- arima(ts(deseasonal_cnt[522:772]), order=c(4,1,5))
forecast_fit3 <- forecast(fit3, h=30)
tsdisplay(residuals(fit3), lag.max = 45, main='Non-Seasonal Model Residuals (4,1,5)')
```

![](GDAT626_DataProject_files/figure-gfm/BUILDING%20AN%20ARIMA%20MODEL-2.png)<!-- -->

``` r
# Manually fitting with arima to find model with acceptable lags from acf/pacf plots
## 2nd order, 1 diff, lag at 8.
fit2 <- arima(deseasonal_cnt, order=c(2,1,8))
tsdisplay(residuals(fit2), lag.max = 45, main = 'Seasonal Model Residuals')
```

![](GDAT626_DataProject_files/figure-gfm/BUILDING%20AN%20ARIMA%20MODEL-3.png)<!-- -->

``` r
# This generates acf/pacf plots that look much better. All lags fall within the blue lines. Let's roll with this model.

# Subset
# 30 stock days that we want to predict
pred <- ts(deseasonal_cnt[773:802])
# Subset from 1/3/2017 to 12/29/2017. Using ARIMA that we manually made earlier.
fit_subset <- arima(ts(deseasonal_cnt[522:772]), order=c(2,1,8))
# Forecasting the subset to predict the next 30 stock days
forecast_subset <- forecast(fit_subset, h=30)
plot(forecast_subset)
lines(ts(deseasonal_cnt[522:802])) # Plotting it against the actual values that were withheld (the next 30 stock days).
```

![](GDAT626_DataProject_files/figure-gfm/BUILDING%20AN%20ARIMA%20MODEL-4.png)<!-- -->

``` r
# The predictive model fails. It assumes a positive, linear trend, whereas the actual model only increases briefly before plummeting.
```

## ADDITIONAL ANALYSIS

``` r
# Auto ARIMA of non-seasonal data
tsdisplay(residuals(fit), lag.max = 20, main='Non-Seasonal Model Residuals Auto ARIMA')
```

![](GDAT626_DataProject_files/figure-gfm/ADDITIONAL%20ANALYSIS-1.png)<!-- -->

``` r
# Auto ARIMA of deseasonal_cnt 4,1,5
tsdisplay(residuals(fit3), lag.max = 20, main='Non-Seasonal Model Residuals (4,1,5)')
```

![](GDAT626_DataProject_files/figure-gfm/ADDITIONAL%20ANALYSIS-2.png)<!-- -->

``` r
# ARIMA deseasonal_cnt 2,1,8 (model we decided would be best)
tsdisplay(residuals(fit2), lag.max = 20, main='Non-Seasonal Model Residuals (2,1,8)')
```

![](GDAT626_DataProject_files/figure-gfm/ADDITIONAL%20ANALYSIS-3.png)<!-- -->

``` r
# Compare each of the models
par(mfrow=c(3,1))

# Auto ARIMA of non-seasonal data
fit1 <- auto.arima(ts(deseasonal_cnt[522:772]))
forecast_fit1 <- forecast(fit1, h=30)
plot(forecast_fit1)

# ARIMA of deseasonal_cnt 4,1,5 (Suggested by auto.arima at one point)
plot(forecast_fit3)

# ARIMA deseasonal_cnt 2,1,8 (model we decided would be best)
plot(forecast_subset)
```

![](GDAT626_DataProject_files/figure-gfm/ADDITIONAL%20ANALYSIS-4.png)<!-- -->

``` r
# Adding seasonality back in to auto ARIMA
fit1_seasonal <- auto.arima(ts(deseasonal_cnt[522:772]), seasonal = TRUE)
forecast_fit1_seasonal <- forecast(fit1_seasonal, h=30)
plot(forecast_fit1_seasonal)
tsdisplay(residuals(fit1_seasonal), lag.max = 20, main = 'Seasonal Auto ARIMA Model')
```

![](GDAT626_DataProject_files/figure-gfm/ADDITIONAL%20ANALYSIS-5.png)<!-- -->![](GDAT626_DataProject_files/figure-gfm/ADDITIONAL%20ANALYSIS-6.png)<!-- -->

``` r
# Still bad performs poorly.

# Auto ARIMA (4,1,5)
summary(fit)
```

    ## Series: deseasonal_cnt 
    ## ARIMA(4,1,5) with drift 
    ## 
    ## Coefficients:
    ##          ar1      ar2     ar3      ar4      ma1     ma2      ma3     ma4
    ##       0.8154  -0.7739  0.4864  -0.2357  -0.3297  0.9602  -0.0823  0.4909
    ## s.e.  0.0537   0.0628  0.0588   0.0413   0.0478  0.0384   0.0596  0.0304
    ##          ma5  drift
    ##       0.4269  6e-04
    ## s.e.  0.0338  3e-04
    ## 
    ## sigma^2 estimated as 6.578e-06:  log likelihood=5695.67
    ## AIC=-11369.34   AICc=-11369.13   BIC=-11312.88
    ## 
    ## Training set error measures:
    ##                        ME        RMSE         MAE       MPE      MAPE
    ## Training set 1.677469e-06 0.002553464 0.001582026 0.1672472 0.9394156
    ##                     MASE        ACF1
    ## Training set 0.008491665 0.009199497

``` r
# MASE = 0.008

# ARIMA (2,1,8)
summary(fit2)
```

    ## 
    ## Call:
    ## arima(x = deseasonal_cnt, order = c(2, 1, 8))
    ## 
    ## Coefficients:
    ##          ar1      ar2      ma1     ma2      ma3     ma4      ma5      ma6
    ##       1.1247  -0.1351  -0.5274  0.0255  -0.0049  0.0001  -0.0040  -0.0399
    ## s.e.  0.1264   0.1256   0.1206  0.0377   0.0232  0.0236   0.0237   0.0242
    ##           ma7     ma8
    ##       -0.7216  0.3368
    ## s.e.   0.0261  0.0818
    ## 
    ## sigma^2 estimated as 5.845e-06:  log likelihood = 5763.9,  aic = -11505.8
    ## 
    ## Training set error measures:
    ##                        ME        RMSE         MAE       MPE      MAPE
    ## Training set 9.704232e-05 0.002416759 0.001432046 0.2161735 0.8496253
    ##                   MASE         ACF1
    ## Training set 0.4823299 -0.002334826

``` r
# MASE = 0.48

# MASE much higher in the 2,1,8 ARIMA, indicating a higher forecast accuracy.
```
