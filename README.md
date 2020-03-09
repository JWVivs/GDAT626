# GDAT626

## Section I. Introduction
Although it is difficult, being able to predict future stock prices is an extremely valuable tool that can help guide individuals towards profitable stock purchases, and to gauge what their current stocks’ values will be. With this in mind, I decided to retrieve historical data on McDonald’s stock prices from 12/7/2014 to 12/7/2019 in an attempt to create a model capable of predicting future stock prices. Assuming I can get anything meaningful from this analysis (e.g. model that shows accurate or at the least promising prediction capabilities), then it would serve as a very useful model going forward that may provide further insight on how a predictive model should be built in order to accurately assess stock prices. The dataset is multivariate; however, I will be focusing on being able to assess the closing stock price, making this a univariate time-series analysis. The reason I am focusing on the closing stock price is because it is generally regarded as the most important price when analyzing how a stock performed during a trading day. 

## Section II. Exploratory Data Analysis
The dataset consisted of the date, the open, high, low, close, and adjusted prices, along with volume. As mentioned earlier, I wanted to assess the closing stock prices, which means that I will have to filter my dataset to feature only the closing price and date. 

![](https://github.com/JWVivs/GDAT626/blob/master/GDAT626_DataProject_files/figure-gfm/Exploratory%20Data%20Analysis-1.png)

<b>Figure 1.</b> Daily Closing Prices of McDonald’s Stock

After converting the date to the proper format, I needed to normalize the stock values. The reason for this is so that the stock prices from five years ago are on a common scale with the stock prices from the current year (2019). By having these values fall between 0 and 1, they become comparable. The normalized closing stock prices and the date were added to a new data frame (mcd_norm). Figure 1 features the daily closing prices before normalization for the sake of data visualization and seeing if there is any notable trend. This plot hints towards potential stationarity, but it is not certain, so it was something to keep in mind as I progressed.

Following normalization, I wanted to make a new column in my normalized dataset that has a weekly moving average for the closing stock prices.

![](https://github.com/JWVivs/GDAT626/blob/master/GDAT626_DataProject_files/figure-gfm/Exploratory%20Data%20Analysis-2.png)

<b>Figure 2.</b> Daily Closing Stock Prices Versus Weekly Moving Average

Figure 2 features the newly created, normalized weekly moving average compared to the normalized daily closing stock prices. It is evident from the plot that the weekly moving average is smoother than the daily closing stock prices. This is beneficial in that it allows me to easier spot a trend in the data; however, I had to be careful not to over smooth the data as that will sacrifice my eventual model’s accuracy. The exploratory data analysis conducted allowed me to better determine what model should be created in order to predict stock prices by observing the stationarity. Stationarity indicates that the mean and variance is constant over time, which is vital to an ARIMA model. In this case, it seemed that an ARIMA of the moving average would be a possible option to predict future stock prices, so I invested my time in to building a proper ARIMA model.

## Section III. Decomposing the Data
Now that I have a general direction of what kind of model I want to build, I need to decompose the data and verify that the direction I am headed is justified. I built a weekly moving average featuring a frequency of 252, as the stock market is typically open 252 days each year. I decomposed that weekly moving average and removed the seasonality from the data. 

![](https://github.com/JWVivs/GDAT626/blob/master/GDAT626_DataProject_files/figure-gfm/DECOMPOSING%20THE%20DATA-1.png)

<b>Figure 3.</b> Decomposition of the Weekly Moving Average
