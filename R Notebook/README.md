Numerai Financial Analysis
================
Last updated: 2024-02-23

## Preliminary Work: Install/Load Packages

To try and ensure that this R Notebook will run successfully, we’ll use
the [renv
package](https://cran.r-project.org/web/packages/renv/index.html) to
create a project-specific library of packages. This will allow us to
install the packages that we need for this project without affecting any
other projects that we may be working on. Additionally, the project
library will track the specific versions of the dependency packages so
that any updates to those packages will not break this project.

The code chunk below will first install the renv package if it is not
already installed. Then we will load the package. Next, we’ll use the
`restore()` function to install any packages listed in the renv.lock
file. Once these packages are installed, we can load them into the R
session using the `library()` commands. Below the code chunk, we’ll list
out the packages that will be used in the project demo. And if you run
into any trouble using renv, then you can use the second code chunk
below and that should be an even more reliable approach to install the
required packages.

``` r
# Install renv package if not already installed
if(!"renv" %in% installed.packages()[,"Package"]) install.packages("renv")
# Load renv package
library(renv)
# Use restore() to install any packages listed in the renv.lock file
renv::restore(clean=TRUE, lockfile="../renv.lock")
# Load in the packages
library(fredr)
library(quantmod)
library(readr)
library(xts)
library(lubridate)
library(ggplot2)
library(dplyr)
```

- The [fredr package](https://cran.r-project.org/package=fredr) is an R
  package that wraps the FRED API for easy importing of FRED data into
  R.
- The [quantmod package](https://cran.r-project.org/package=quantmod)
  contains tools for importing and analyzing financial data.
- The [readr package](https://cran.r-project.org/package=readr) is a
  package for reading in data files. We’ll use it to read in the csv
  file from the Numerai Fund website.
- The [xts package](https://cran.r-project.org/package=xts) is short for
  ‘eXtensible Time Series’, which contains tools for working with time
  series data.
- The [lubridate package](https://cran.r-project.org/package=lubridate)
  simplifies calculation involving dates.
- The [ggplot2 package](https://cran.r-project.org/package=ggplot2) for
  graphics and visuals.
- The [dplyr package](https://cran.r-project.org/package=dplyr) contains
  tools for data manipulation.

Since the rmarkdown functionality is built into RStudio, this one is
automatically loaded when we open the RStudio. So no need to use the
`library()` function for this one. Another observation to make about the
code chunk above is that it is labeled as ‘setup’, which is a special
name, which the R Notebook will recognize and automatically run prior to
running any other code chunk. This is useful for loading in packages and
setting up other global options that will be used throughout the
notebook.

Then if you wish to try and update the versions of the various R
packages in the lock file, you can use the `renv::update()` function to
update the packages in the project library. However, it is possible that
these updates could break the code in this notebook. If so, you may need
to adapt the code to work with the updated packages.

My recommendation is to first run through the code using the versions of
the packages in the lock file. Then if you want to try and update the
packages, you can do so and then run through the code again to see if it
still works. If not, you can always revert back to the lock file
versions using the `renv::restore()` function.

If you update the packages and get everything working successfully, then
you can update the lock file using the `renv::snapshot()` function. This
will update the lock file with the versions of the packages that are
currently installed in the project library. Then you can commit the
updated lock file to the repository so that others can use the updated
versions of the packages.

### Alternative Package Installation Code

If you run into any trouble using renv in the code chunk above, then you
can use the code chunk below to install the required packages for this
analysis. This method will first check if you have already installed the
packages. If any are missing, it will then install them. Then it will
load the packages into the R session. A potential flaw in this approach
compared to using renv is that it will simply install the latest
versions of the packages, which could potentially break some of the code
in this notebook if any of the updates aren’t backwards compatible.

As long as you have downloaded the entire project repository, the renv
chunk above will likely be managing the packages. Thus, the `eval=FALSE`
option is used to prevent this chunk from running unless manually
executed. So if you only downloaded this one Rmd file, this code chunk
should take care of installing the packages for you.

``` r
# Create list of packages needed for this exercise
list.of.packages = c("fredr",
                     "quantmod",
                     "readr",
                     "xts",
                     "lubridate",
                     "ggplot2",
                     "dplyr",
                     "rmarkdown")
# Check if any have not yet been installed
new.packages = list.of.packages[!(list.of.packages %in% installed.packages()[,"Package"])]
# If any need to be installed, install them
if(length(new.packages)) install.packages(new.packages)
# Load in the packages
library(fredr)
library(quantmod)
library(readr)
library(xts)
library(lubridate)
library(ggplot2)
library(dplyr)
```

## Import and Clean Data

We will be collecting data from three different sources here. First,
we’ll import the monthly returns for the Numerai One and Numerai Supreme
hedge funds from the [Numerai Fund website](https://numerai.fund/).
Second, we’ll import the monthly returns for the S&P 500 index from the
[FRED website](https://fred.stlouisfed.org/). Lastly, we’ll import the
monthly returns for the Numeraire (NMR) token and Ether (ETH) from
[Yahoo Finance](https://finance.yahoo.com/) via the quantmod package.
The code below will import the data from each of these sources and clean
it up for analysis.

### Numerai Hedge Fund Performance Data

Due to their classification as hedge funds, there is far less
publicly-available information compared to mutual funds or ETFs.
However, since Numerai self-publishes data for monthly fund returns on
their webpage, we can make use of this to examine the performance and
test their claim of investing with a market-neutral strategy. To find
the url for the code chunk below, go to the main [fund
website](https://numerai.fund/) and right-click on “Download
Performance” and select “Copy link address.” If you left-click on the
link, most browsers will simply attempt to download the current csv
file. However, we can incorporate the static url into our script here
and access the csv with `read_csv()`. This should retrieve all the most
recent data each time this code is run.

``` r
url = "https://api-financial.numer.ai/get_lp_performance_csv"
fundrets = read_csv(url, show_col_types=FALSE)
```

Now, let’s take the monthly returns from the data and convert them to an
xts object of annualized percentages. Take note of the date conversion
in the `order.by` parameter. Since these dates are the last day of each
month, we add one date and subtract one month to convert the date to the
first day of each respective month. This is in anticipation of merging
with the FRED data that will be imported below, which uses dates at the
beginning of each month.

``` r
nmr = data.frame(nmr_one=fundrets$numerai_one_monthly_return_net_of_fees*12*100,
                     nmr_sup=fundrets$supreme_monthly_return_net_of_fees*12*100)
nmr = xts(nmr,
          order.by=fundrets$month_end_date + days(1) - months(1))
```

For the Numerai Supreme Fund, the data shows a 0% return prior to the
launch of the fund in August 2022. To avoid the models from actually
interpreting a 0% return, let’s replace those 0’s with missing values
`NA`. The xts package makes this easier by allowing us to index by time.
The `"/2022-07-01"` in the square brackets specifies the rows
corresponding to dates up to and including July 2022.

``` r
nmr$nmr_sup["/2022-07-01"] = NA
```

Then to simplify the download of the FRED data, let’s save the date of
the earliest observation to `startdate`. This will help us only download
the data that we will actually need for the analysis. Then,
`startdate_1` will go back one additional month for the FRED download
since we will lose an observation due to differencing. Similarly,
`startdate_2` will go back two months from `startdate`, which is what
downloads the correct amount of data for the crypto assets.

``` r
startdate = min(fundrets$month_end_date)
startdate_1 = startdate %m-% months(1)
startdate_2 = startdate %m-% months(2)
```

### FRED Data Import

To access the FRED API, you must first create an account and [request an
API key](https://fred.stlouisfed.org/docs/api/api_key.html). If you wish
to run the code and replicate the results, you’ll need to make an
account, generate your own API key, and run this command un-commented
with your key in place of the placeholder text.

``` r
#fredr_set_key("<YOUR-FRED-API-KEY>")
```

Using the `fredr()` function, we will import the 10-year Treasury note
yields. This is a typical proxy for the risk-free return when applying
CAPM and calculating stock betas. The Sys.Date function is simply using
the computer’s current time to get the most up-to-date data.

``` r
RFraw = fredr(
  series_id = "DGS10",
  observation_start = startdate,
  observation_end = as.Date(Sys.Date()),
  frequency = "m"
)
INFraw = fredr(
  series_id = "CPIAUCSL",
  observation_start = startdate_1,
  observation_end = as.Date(Sys.Date()),
  frequency = "m"
)
SPraw = fredr(
  series_id = "SP500",
  observation_start = startdate_1,
  observation_end = as.Date(Sys.Date()),
  frequency = "m"
)
```

Next, calculate the inflation rates from the CPI and merge those
annualized growth rates to the merged data frame.

``` r
INF = xts(INFraw,order.by=INFraw$date)
colnames(INF)[colnames(INF)=="value"] = "CPI"
INF = subset(INF,select=-c(date,series_id,realtime_start,realtime_end))
INF$INFmonthly = log(as.numeric(INF$CPI)) - log(as.numeric(lag(INF$CPI)))
INF$inf = INF$INFmonthly*12*100
```

Now convert the S&P 500 index levels into annualized market returns and
merge to `ALL`.

``` r
SP500 = xts(SPraw,order.by=SPraw$date)
colnames(SP500)[colnames(SP500)=="value"] = "SP500"
SP500 = subset(SP500,select=-c(date,series_id,realtime_start,realtime_end))
SP500$SPmonthly = log(as.numeric(SP500$SP500)) - log(as.numeric(lag(SP500$SP500)))
SP500$sp500 = SP500$SPmonthly*12*100
```

### NMR and ETH Price Data

In addition to the Numerai Fund data, let’s also import the price data
for the Numeraire (NMR) token and Ether (ETH), which is the native token
of the Ethereum blockchain that NMR transacts on.

``` r
tickers = c("ETH-USD",
            "NMR-USD")
getSymbols(tickers,
           src="yahoo",
           from=startdate_2,
           to=Sys.Date(),
           periodicity="monthly")
```

First, we’ll give the crypto data names that do not have a special
character (`-`), then we’ll compute the continuously compounded annual
returns for the monthly data series.

``` r
ETH = `ETH-USD`
NMR = `NMR-USD`
# Compute returns
ETH$Return = c(NA, diff(log(as.numeric(ETH$`ETH-USD.Adjusted`))))
NMR$Return = c(NA, diff(log(as.numeric(NMR$`NMR-USD.Adjusted`))))
# Annualize returns
ETH$ETH = ETH$Return*12*100
NMR$NMR = NMR$Return*12*100
```

### Merging and Cleaning

Now, let’s merge all these variables to a single data frame named `ALL`.

``` r
ALL = xts(RFraw,order.by=RFraw$date)
colnames(ALL)[colnames(ALL)=="value"] = "rf"
ALL = subset(ALL,select=-c(date,series_id,realtime_start,realtime_end))
ALL = merge(ALL,
            INF$inf,
            SP500$sp500,
            nmr$nmr_one,
            nmr$nmr_sup,
            NMR$NMR,
            ETH$ETH)
```

Then the last cleaning step is to remove any missing values. The
`complete.cases()` function returns the indices (rows) of observations
with no missing values.

``` r
ntrim = sum(!complete.cases(tail(ALL)))
FINAL = ALL[2:(nrow(ALL)-ntrim),]
```

## Data Summaries and Plots

When summarizing and visualizing the asset returns, we’ll begin with the
nominal returns. Then we’ll examine the real (inflation-adjusted) return
series. Lastly, we’ll convert each of those return series into risk
premiums, resulting in a nominal risk premium and a real risk premium.
Then we can compare those two sets of results to see the impact of
inflation on the observed relationships.

### Nominal Returns

Let’s first take a look at what each time series looks like:

``` r
ggplot(FINAL,aes(x=Index,y=rf))+
  geom_col()+
  ggtitle("Risk-Free Asset Returns")
```

![](README_files/figure-gfm/plots-1.png)<!-- -->

``` r
ggplot(FINAL,aes(x=Index,y=inf))+
  geom_col()+
  ggtitle("Annualized Inflation Rates")
```

![](README_files/figure-gfm/plots-2.png)<!-- -->

``` r
ggplot(FINAL,aes(x=Index,y=sp500))+
  geom_col()+
  ggtitle("S&P 500 Returns")
```

![](README_files/figure-gfm/plots-3.png)<!-- -->

``` r
ggplot(FINAL,aes(x=Index,y=nmr_one))+
  geom_col()+
  ggtitle("Numerai One Returns")
```

![](README_files/figure-gfm/plots-4.png)<!-- -->

``` r
ggplot(FINAL,aes(x=Index,y=nmr_sup))+
  geom_col()+
  ggtitle("Numerai Supreme Returns")
```

    ## Warning: Removed 35 rows containing missing values or values outside the scale range
    ## (`geom_col()`).

![](README_files/figure-gfm/plots-5.png)<!-- -->

``` r
ggplot(FINAL,aes(x=Index,y=NMR))+
  geom_col()+
  ggtitle("Numeraire (NMR) Returns")
```

![](README_files/figure-gfm/plots-6.png)<!-- -->

``` r
ggplot(FINAL,aes(x=Index,y=ETH))+
  geom_col()+
  ggtitle("Ether (ETH) Returns")
```

![](README_files/figure-gfm/plots-7.png)<!-- -->

Next, let’s calculate the average annual return (mean) and volatility
(standard deviation).

``` r
Er_nom = colMeans(FINAL,na.rm=TRUE)
Er_nom |> round(2)
```

    ##      rf     inf   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    2.30    4.31   11.45    5.00   -3.84   32.98   58.48

``` r
sigma_nom = apply(FINAL,2,sd,na.rm=TRUE)
sigma_nom |> round(2)
```

    ##      rf     inf   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    1.28    4.11   53.75   34.84   55.90  354.39  290.79

Then let’s compare the correlations across all these asset returns:

``` r
Rho = cor(FINAL, use="pairwise.complete.obs")
Rho |> round(2)
```

    ##            rf   inf sp500 nmr_one nmr_sup   NMR   ETH
    ## rf       1.00  0.03 -0.09   -0.12    0.09 -0.10 -0.21
    ## inf      0.03  1.00  0.00    0.15    0.17 -0.13 -0.21
    ## sp500   -0.09  0.00  1.00    0.04    0.00 -0.04  0.47
    ## nmr_one -0.12  0.15  0.04    1.00    0.88 -0.15 -0.16
    ## nmr_sup  0.09  0.17  0.00    0.88    1.00  0.13 -0.20
    ## NMR     -0.10 -0.13 -0.04   -0.15    0.13  1.00  0.22
    ## ETH     -0.21 -0.21  0.47   -0.16   -0.20  0.22  1.00

### Real Returns

To calculate real returns from the nominal returns, we must subtract the
inflation rate, and then divide the difference by the quantity of
(1+inflation). Note that the inflation percentage must be divided by 100
to convert back to a decimal.

``` r
REAL = xts(order.by=index(FINAL))
REAL$rf = (FINAL$rf-FINAL$inf)/(1+(FINAL$inf/100))
REAL$sp500 = (FINAL$sp500-FINAL$inf)/(1+(FINAL$inf/100))
REAL$nmr_one = (FINAL$nmr_one-FINAL$inf)/(1+(FINAL$inf/100))
REAL$nmr_sup = (FINAL$nmr_sup-FINAL$inf)/(1+(FINAL$inf/100))
REAL$NMR = (FINAL$NMR-FINAL$inf)/(1+(FINAL$inf/100))
REAL$ETH = (FINAL$ETH-FINAL$inf)/(1+(FINAL$inf/100))
```

Now let’s compare the means, standard deviations, and correlations for
these inflation-adjusted return series:

``` r
Er_real = colMeans(REAL,na.rm=TRUE)
Er_real |> round(2)
```

    ##      rf   sp500 nmr_one nmr_sup     NMR     ETH 
    ##   -1.77    6.98    0.61   -6.99   29.62   54.41

``` r
sigma_real = apply(REAL,2,sd,na.rm=TRUE)
sigma_real |> round(2)
```

    ##      rf   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    4.06   53.79   33.37   54.48  339.20  280.87

``` r
Rho_real = cor(REAL, use="pairwise.complete.obs")
Rho_real |> round(2)
```

    ##            rf sp500 nmr_one nmr_sup   NMR   ETH
    ## rf       1.00  0.02   -0.08   -0.11  0.14  0.15
    ## sp500    0.02  1.00    0.04   -0.01 -0.06  0.49
    ## nmr_one -0.08  0.04    1.00    0.88 -0.14 -0.14
    ## nmr_sup -0.11 -0.01    0.88    1.00  0.12 -0.22
    ## NMR      0.14 -0.06   -0.14    0.12  1.00  0.22
    ## ETH      0.15  0.49   -0.14   -0.22  0.22  1.00

### Risk Premiums

Then to normalize risk by the risk-free rate, we can difference each
return series by the risk-free rate to compute risk premiums (or excess
returns). This can be done with the nominal returns or the real returns.
So let’s do each and compare how inflation-adjusting the returns impacts
the results.

``` r
XSnom = xts(order.by=index(FINAL))
XSnom$sp500 = FINAL$sp500-FINAL$rf
XSnom$nmr_one = FINAL$nmr_one-FINAL$rf
XSnom$nmr_sup = FINAL$nmr_sup-FINAL$rf
XSnom$NMR = FINAL$NMR-FINAL$rf
XSnom$ETH = FINAL$ETH-FINAL$rf
```

``` r
XSreal = xts(order.by=index(REAL))
XSreal$sp500 = REAL$sp500-REAL$rf
XSreal$nmr_one = REAL$nmr_one-REAL$rf
XSreal$nmr_sup = REAL$nmr_sup-REAL$rf
XSreal$NMR = REAL$NMR-REAL$rf
XSreal$ETH = REAL$ETH-REAL$rf
```

For each set of risk premiums, let’s compute the average annual returns
and volatilities, as well as the correlation matrices to see how much
the fund returns co-move with the market returns of the S&P 500.

``` r
xsEr_nom = colMeans(XSnom, na.rm=TRUE)
xsEr_nom |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    9.15    2.69   -7.70   30.68   56.18

``` r
xssigma_nom = apply(XSnom, 2, sd, na.rm=TRUE)
xssigma_nom |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH 
    ##   53.88   35.01   55.86  354.52  291.06

``` r
xsRho_nom = cor(XSnom, use="pairwise.complete.obs")
xsRho_nom |> round(2)
```

    ##         sp500 nmr_one nmr_sup   NMR   ETH
    ## sp500    1.00    0.05    0.00 -0.04  0.47
    ## nmr_one  0.05    1.00    0.88 -0.15 -0.15
    ## nmr_sup  0.00    0.88    1.00  0.12 -0.20
    ## NMR     -0.04   -0.15    0.12  1.00  0.22
    ## ETH      0.47   -0.15   -0.20  0.22  1.00

``` r
xsEr_real = colMeans(XSreal, na.rm=TRUE)
xsEr_real |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    8.75    2.38   -7.62   31.39   56.18

``` r
xssigma_real = apply(XSreal, 2, sd, na.rm=TRUE)
xssigma_real |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH 
    ##   53.87   33.92   54.72  338.67  280.28

``` r
xsRho_real = cor(XSreal, use="pairwise.complete.obs")
xsRho_real |> round(2)
```

    ##         sp500 nmr_one nmr_sup   NMR   ETH
    ## sp500    1.00    0.05   -0.01 -0.07  0.48
    ## nmr_one  0.05    1.00    0.88 -0.15 -0.15
    ## nmr_sup -0.01    0.88    1.00  0.12 -0.21
    ## NMR     -0.07   -0.15    0.12  1.00  0.22
    ## ETH      0.48   -0.15   -0.21  0.22  1.00

## Sharpe Ratios and CAPM Betas

From the risk premiums, we can easily compute the Sharpe ratio of the
S&P 500 and the Numerai One hedge fund to compare risk-adjusted returns.

``` r
sharpes_nom = xsEr_nom/xssigma_nom
sharpes_nom |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    0.17    0.08   -0.14    0.09    0.19

``` r
sharpes_real = xsEr_real/xssigma_real
sharpes_real |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH 
    ##    0.16    0.07   -0.14    0.09    0.20

### Numerai One

To estimate the fund’s alpha and beta from CAPM, we’ll run a linear
regression model of each set of fund risk premiums on the market risk
premium. The p-value (`Pr(>|t|)`) for the beta (`sp500`) estimate is for
a statistical test where the null hypothesis is that the fund has a beta
of zero. This property of a zero beta corresponds with a strategy of
market neutrality. In other words, if the market goes up or down by any
amount, the expected return of the fund should be unchanged. The alpha
(intercept) estimate then indicates the beta-neutral performance of the
fund. As of January 2024, the beta estimate for Numerai One is 0.03 in
both nominal and real units. With t-stats under 0.5 for those estimates,
this lack of statistical significance provides good evidence in support
of their claim to invest with a market-neutral strategy. The alpha
estimates are a little over 2% in both nominal and real units suggesting
a modest return. In both cases, the $R^2$ of is fairly small (\<1%),
which also follows well given the claim of market-neutrality.

``` r
NMR_one_fit_nom = lm(nmr_one~sp500,data=XSnom)
summary(NMR_one_fit_nom)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ sp500, data = XSnom)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -125.738  -15.059    1.328   19.836   75.048 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  2.41985    4.92155   0.492    0.625
    ## sp500        0.02991    0.09089   0.329    0.743
    ## 
    ## Residual standard error: 35.31 on 51 degrees of freedom
    ## Multiple R-squared:  0.002119,   Adjusted R-squared:  -0.01745 
    ## F-statistic: 0.1083 on 1 and 51 DF,  p-value: 0.7435

``` r
ggplot(XSnom,aes(x=sp500,y=nmr_one))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/nmronecapm-1.png)<!-- -->

``` r
NMR_one_fit_real = lm(nmr_one~sp500,data=XSreal)
summary(NMR_one_fit_real)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ sp500, data = XSreal)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -123.825  -15.074    1.334   18.648   72.871 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  2.10310    4.76135   0.442    0.661
    ## sp500        0.03215    0.08806   0.365    0.717
    ## 
    ## Residual standard error: 34.21 on 51 degrees of freedom
    ## Multiple R-squared:  0.002606,   Adjusted R-squared:  -0.01695 
    ## F-statistic: 0.1333 on 1 and 51 DF,  p-value: 0.7166

``` r
ggplot(XSreal,aes(x=sp500,y=nmr_one))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/nmronecapm-2.png)<!-- -->

### Numerai Supreme

As of January 2024, there are still just a few observations for the
Numerai Supreme fund. We’ll build out this discussion as we get more
data. But for now, the beta estimates and $R^2$ values are similarly
small and insignificant. However, the alpha is quite negative suggesting
that the fund has struggled to deliver positive returns from its
market-neutral strategy.

``` r
NMR_sup_fit_nom = lm(nmr_sup~sp500,data=XSnom)
summary(NMR_sup_fit_nom)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ sp500, data = XSnom)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -143.002  -19.528    2.238   23.529  102.166 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) -7.682466  13.908304  -0.552    0.588
    ## sp500       -0.001668   0.309314  -0.005    0.996
    ## 
    ## Residual standard error: 57.58 on 16 degrees of freedom
    ##   (35 observations deleted due to missingness)
    ## Multiple R-squared:  1.817e-06,  Adjusted R-squared:  -0.0625 
    ## F-statistic: 2.908e-05 on 1 and 16 DF,  p-value: 0.9958

``` r
ggplot(XSnom,aes(x=sp500,y=nmr_sup))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

    ## Warning: Removed 35 rows containing non-finite outside the scale range
    ## (`stat_smooth()`).

    ## Warning: Removed 35 rows containing missing values or values outside the scale range
    ## (`geom_point()`).

![](README_files/figure-gfm/nmrsupcapm-1.png)<!-- -->

``` r
NMR_sup_fit_real = lm(nmr_sup~sp500,data=XSreal)
summary(NMR_sup_fit_real)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ sp500, data = XSreal)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -141.159  -19.136    2.294   22.404   99.449 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) -7.54441   13.63527  -0.553    0.588
    ## sp500       -0.00742    0.31275  -0.024    0.981
    ## 
    ## Residual standard error: 56.4 on 16 degrees of freedom
    ##   (35 observations deleted due to missingness)
    ## Multiple R-squared:  3.517e-05,  Adjusted R-squared:  -0.06246 
    ## F-statistic: 0.0005628 on 1 and 16 DF,  p-value: 0.9814

``` r
ggplot(XSreal,aes(x=sp500,y=nmr_sup))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

    ## Warning: Removed 35 rows containing non-finite outside the scale range
    ## (`stat_smooth()`).
    ## Removed 35 rows containing missing values or values outside the scale range
    ## (`geom_point()`).

![](README_files/figure-gfm/nmrsupcapm-2.png)<!-- -->

### Numeraire (NMR) token

For the NMR token, we get some interesting results with the negative
beta estimates, and the alpha estimates are quite large as well.
However, the $R^2$ values are very small. So the bigger take away is
that there isn’t much of a relationship here.

``` r
NMR_fit_nom = lm(NMR~sp500,data=XSnom)
summary(NMR_fit_nom)
```

    ## 
    ## Call:
    ## lm(formula = NMR ~ sp500, data = XSnom)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -799.68 -228.14  -26.36  202.84 1181.08 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  32.9227    49.8541   0.660    0.512
    ## sp500        -0.2457     0.9207  -0.267    0.791
    ## 
    ## Residual standard error: 357.7 on 51 degrees of freedom
    ## Multiple R-squared:  0.001395,   Adjusted R-squared:  -0.01819 
    ## F-statistic: 0.07124 on 1 and 51 DF,  p-value: 0.7906

``` r
ggplot(XSnom,aes(x=sp500,y=NMR))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/nmrcapm-1.png)<!-- -->

``` r
NMR_fit_real = lm(NMR~sp500,data=XSreal)
summary(NMR_fit_real)
```

    ## 
    ## Call:
    ## lm(formula = NMR ~ sp500, data = XSreal)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -747.19 -230.52  -32.19  194.27 1140.28 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  35.3681    47.4765   0.745    0.460
    ## sp500        -0.4546     0.8780  -0.518    0.607
    ## 
    ## Residual standard error: 341.1 on 51 degrees of freedom
    ## Multiple R-squared:  0.005229,   Adjusted R-squared:  -0.01428 
    ## F-statistic: 0.2681 on 1 and 51 DF,  p-value: 0.6069

``` r
ggplot(XSreal,aes(x=sp500,y=NMR))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/nmrcapm-2.png)<!-- -->

### Ether (ETH)

Ether produces a similarly large alpha estimate. However, the beta
estimate flips to a large, positive, and significant value. This
suggests that ETH has a large degree of systematic risk with a
reasonably strong relationship to the stock market. The \*\*\* in the
p-value column indicates that this beta estimate is significantly
different than 0 at the 1% level. However, a beta equal to 1 is also
meaningful to determine whether there is more systematic risk than the
market portfolio. Thus, you can subtract 1 from the beta estimate and
divide by the Std. Error to get a t-stat for the null hypothesis that
the beta is equal to 1.

``` r
ETH_fit_nom = lm(ETH~sp500,data=XSnom)
summary(ETH_fit_nom)
```

    ## 
    ## Call:
    ## lm(formula = ETH ~ sp500, data = XSnom)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -637.01 -181.11  -13.69  209.23  581.68 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)  32.7746    36.0741   0.909 0.367868    
    ## sp500         2.5585     0.6662   3.840 0.000341 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 258.8 on 51 degrees of freedom
    ## Multiple R-squared:  0.2243, Adjusted R-squared:  0.2091 
    ## F-statistic: 14.75 on 1 and 51 DF,  p-value: 0.000341

``` r
ggplot(XSnom,aes(x=sp500,y=ETH))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/ethcapm-1.png)<!-- -->

``` r
ETH_fit_real = lm(ETH~sp500,data=XSreal)
summary(ETH_fit_real)
```

    ## 
    ## Call:
    ## lm(formula = ETH ~ sp500, data = XSreal)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -561.67 -171.80  -14.92  204.11  567.40 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)  34.1425    34.4706   0.990  0.32661    
    ## sp500         2.5187     0.6375   3.951  0.00024 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 247.6 on 51 degrees of freedom
    ## Multiple R-squared:  0.2343, Adjusted R-squared:  0.2193 
    ## F-statistic: 15.61 on 1 and 51 DF,  p-value: 0.0002401

``` r
ggplot(XSreal,aes(x=sp500,y=ETH))+
  geom_point()+
  geom_smooth(method="lm")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](README_files/figure-gfm/ethcapm-2.png)<!-- -->

## Multi-Factor Models

Let’s try to incorporate some additional factors into the model to see
if we can improve the explanatory power.

### ETH-Factor Models

First, let’s try to incorporate the ETH risk premium as an additional
factor beyond the market risk premium.

``` r
nmr_one_fit2_nom = lm(nmr_one~sp500+ETH,data=XSnom)
summary(nmr_one_fit2_nom)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ sp500 + ETH, data = XSnom)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -127.010  -12.729   -0.758   19.031   70.235 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  3.30050    4.91244   0.672    0.505
    ## sp500        0.09866    0.10219   0.965    0.339
    ## ETH         -0.02687    0.01892  -1.420    0.162
    ## 
    ## Residual standard error: 34.97 on 50 degrees of freedom
    ## Multiple R-squared:  0.04083,    Adjusted R-squared:  0.00246 
    ## F-statistic: 1.064 on 2 and 50 DF,  p-value: 0.3527

``` r
nmr_one_fit2_real = lm(nmr_one~sp500+ETH,data=XSreal)
summary(nmr_one_fit2_real)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ sp500 + ETH, data = XSreal)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -125.158  -11.838   -0.715   18.237   64.947 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  3.04412    4.75713    0.64    0.525
    ## sp500        0.10157    0.09959    1.02    0.313
    ## ETH         -0.02756    0.01914   -1.44    0.156
    ## 
    ## Residual standard error: 33.85 on 50 degrees of freedom
    ## Multiple R-squared:  0.04232,    Adjusted R-squared:  0.00401 
    ## F-statistic: 1.105 on 2 and 50 DF,  p-value: 0.3393

``` r
nmr_sup_fit2_nom = lm(nmr_sup~sp500+ETH,data=XSnom)
summary(nmr_sup_fit2_nom)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ sp500 + ETH, data = XSnom)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -144.743  -25.328    0.431   38.145   82.430 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) -6.350227  14.163927  -0.448    0.660
    ## sp500       -0.005051   0.312853  -0.016    0.987
    ## ETH         -0.078656   0.098095  -0.802    0.435
    ## 
    ## Residual standard error: 58.23 on 15 degrees of freedom
    ##   (35 observations deleted due to missingness)
    ## Multiple R-squared:  0.0411, Adjusted R-squared:  -0.08675 
    ## F-statistic: 0.3215 on 2 and 15 DF,  p-value: 0.7299

``` r
nmr_sup_fit2_real = lm(nmr_sup~sp500+ETH,data=XSreal)
summary(nmr_sup_fit2_real)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ sp500 + ETH, data = XSreal)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -142.94  -24.72    0.91   36.97   79.26 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) -6.18100   13.86296  -0.446    0.662
    ## sp500       -0.01320    0.31584  -0.042    0.967
    ## ETH         -0.08352    0.10006  -0.835    0.417
    ## 
    ## Residual standard error: 56.95 on 15 degrees of freedom
    ##   (35 observations deleted due to missingness)
    ## Multiple R-squared:  0.04442,    Adjusted R-squared:  -0.08299 
    ## F-statistic: 0.3486 on 2 and 15 DF,  p-value: 0.7112

``` r
NMR_fit2_nom = lm(NMR~sp500+ETH,data=XSnom)
summary(NMR_fit2_nom)
```

    ## 
    ## Call:
    ## lm(formula = NMR ~ sp500 + ETH, data = XSnom)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -732.05 -184.81  -21.47  141.94 1153.69 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)  
    ## (Intercept)  20.6835    48.8678   0.423   0.6739  
    ## sp500        -1.2012     1.0165  -1.182   0.2429  
    ## ETH           0.3734     0.1882   1.985   0.0527 .
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 347.8 on 50 degrees of freedom
    ## Multiple R-squared:  0.07431,    Adjusted R-squared:  0.03728 
    ## F-statistic: 2.007 on 2 and 50 DF,  p-value: 0.1451

``` r
NMR_fit2_real = lm(NMR~sp500+ETH,data=XSreal)
summary(NMR_fit2_real)
```

    ## 
    ## Call:
    ## lm(formula = NMR ~ sp500 + ETH, data = XSreal)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -678.70 -180.57  -31.93  153.64 1112.37 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)  
    ## (Intercept)  21.7568    46.3357   0.470   0.6407  
    ## sp500        -1.4587     0.9701  -1.504   0.1389  
    ## ETH           0.3987     0.1864   2.138   0.0374 *
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 329.7 on 50 degrees of freedom
    ## Multiple R-squared:  0.08857,    Adjusted R-squared:  0.05211 
    ## F-statistic: 2.429 on 2 and 50 DF,  p-value: 0.09841

### NMR-Factor Models

Then another factor we can try to incorporate into the model is the
various Numerai assets. In other words, let’s see if the NMR token
returns help explain the fund returns.

``` r
num_one_fit3_nom = lm(nmr_one~sp500+ETH+NMR,data=XSnom)
summary(num_one_fit3_nom)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ sp500 + ETH + NMR, data = XSnom)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -128.837  -11.210    0.063   18.770   72.594 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  3.506274   4.946788   0.709    0.482
    ## sp500        0.086705   0.104141   0.833    0.409
    ## ETH         -0.023155   0.019749  -1.172    0.247
    ## NMR         -0.009949   0.014290  -0.696    0.490
    ## 
    ## Residual standard error: 35.15 on 49 degrees of freedom
    ## Multiple R-squared:  0.05022,    Adjusted R-squared:  -0.007929 
    ## F-statistic: 0.8637 on 3 and 49 DF,  p-value: 0.4663

``` r
num_one_fit3_real = lm(nmr_one~sp500+ETH+NMR,data=XSreal)
summary(num_one_fit3_real)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ sp500 + ETH + NMR, data = XSreal)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -127.013  -10.744    0.158   16.924   67.214 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  3.26633    4.79212   0.682    0.499
    ## sp500        0.08667    0.10234   0.847    0.401
    ## ETH         -0.02349    0.02010  -1.169    0.248
    ## NMR         -0.01021    0.01459  -0.700    0.487
    ## 
    ## Residual standard error: 34.03 on 49 degrees of freedom
    ## Multiple R-squared:  0.0518, Adjusted R-squared:  -0.006258 
    ## F-statistic: 0.8922 on 3 and 49 DF,  p-value: 0.4519

``` r
num_sup_fit3_nom = lm(nmr_sup~sp500+ETH+NMR,data=XSnom)
summary(num_sup_fit3_nom)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ sp500 + ETH + NMR, data = XSnom)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -126.18  -35.88   19.16   28.17   92.02 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) -3.03486   13.88230  -0.219    0.830
    ## sp500       -0.05235    0.30415  -0.172    0.866
    ## ETH         -0.20191    0.12792  -1.578    0.137
    ## NMR          0.13211    0.09205   1.435    0.173
    ## 
    ## Residual standard error: 56.28 on 14 degrees of freedom
    ##   (35 observations deleted due to missingness)
    ## Multiple R-squared:  0.1641, Adjusted R-squared:  -0.01505 
    ## F-statistic: 0.916 on 3 and 14 DF,  p-value: 0.4584

``` r
num_sup_fit3_real = lm(nmr_sup~sp500+ETH+NMR,data=XSreal)
summary(num_sup_fit3_real)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ sp500 + ETH + NMR, data = XSreal)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -124.07  -35.07   19.41   28.03   88.79 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) -2.89371   13.55139  -0.214    0.834
    ## sp500       -0.05994    0.30614  -0.196    0.848
    ## ETH         -0.20967    0.12937  -1.621    0.127
    ## NMR          0.13559    0.09266   1.463    0.165
    ## 
    ## Residual standard error: 54.9 on 14 degrees of freedom
    ##   (35 observations deleted due to missingness)
    ## Multiple R-squared:  0.1712, Adjusted R-squared:  -0.006421 
    ## F-statistic: 0.9638 on 3 and 14 DF,  p-value: 0.4371

### Fama/French Research Factors

A popular asset pricing model in the finance literature has been the
factor models from [Fama and French
(1993)](https://doi.org/10.1016/0304-405X(93)90023-5), which introduced
a three-factor model that includes SMB (Small Minus Big) as a ‘size’
factor and HML (High Minus Low) as a ‘value’/‘growth’ factor. See [the
3-Factors
webpage](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/Data_Library/f-f_factors.html)
for more detail. Additionally the more recent [Fama and French
(2015)](https://doi.org/10.1016/j.jfineco.2014.10.010) includes two
additional factors: RMW (Robust Minus Weak) as a ‘profitability’ factor
and CMA (Conservative Minus Aggressive) factor. The [5-Factors
webpage](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/Data_Library/f-f_5_factors_2x3.html)
has more detail. The codes below import the factor data from the website
and conduct those regressions on the Numerai returns.

``` r
ff3url = "https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_Factors_CSV.zip"
# Create subdirectory for file downloads
subdirectory = "Factor Data"
dir.create(subdirectory, showWarnings=FALSE)
# Define the file paths
zip_filepath = file.path(subdirectory, "FF3-factors.zip")
csv_filepath = file.path(subdirectory, "FF3-factors.csv")
# Download the zip file
download.file(ff3url, destfile=zip_filepath)
# Extract the CSV file from the zip file
unzip(zip_filepath, exdir=subdirectory)
file.rename("Factor Data/F-F_Research_Data_Factors.CSV", csv_filepath)
```

    ## [1] TRUE

``` r
FF3 = read_csv(csv_filepath,
               col_types = cols(...1 = col_date(format = "%Y%m")), 
               skip = 2)
```

    ## New names:
    ## • `` -> `...1`

    ## Warning: One or more parsing issues, call `problems()` on your data frame for details,
    ## e.g.:
    ##   dat <- vroom(...)
    ##   problems(dat)

``` r
FF3 = FF3 |> rename(Date=...1)
# Trim annual observations from bottom of date frame (dates import as missing)
FF3 = FF3[complete.cases(FF3),]
# Reformat to xts object
FF3xts = xts(FF3[,-1], order.by=FF3$Date)
# Remove data prior to first BTC observation
FF3xts = FF3xts[paste(startdate,"/",sep="")]
```

``` r
# Compile data frame of annualized crypto returns (un-annualize)
assets_ff3 = merge(XSreal,
                   FF3xts*12)
#assets_ff3 = assets_ff3[complete.cases(assets_ff3),]
# Compute average annual returns, volatility, and correlations
Er_ff3 = colMeans(assets_ff3,na.rm=TRUE)
Er_ff3 |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH  Mkt.RF     SMB     HML      RF 
    ##    8.75    2.38   -7.62   31.39   56.18   12.75    0.81    0.90    1.68

``` r
sd_ff3 = apply(assets_ff3,2,sd,na.rm=TRUE)
sd_ff3 |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH  Mkt.RF     SMB     HML      RF 
    ##   53.87   33.92   54.72  338.67  280.28   68.82   36.59   58.88    2.01

``` r
Sharpe_ff3 = Er_ff3/sd_ff3
Sharpe_ff3 |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH  Mkt.RF     SMB     HML      RF 
    ##    0.16    0.07   -0.14    0.09    0.20    0.19    0.02    0.02    0.84

``` r
cor(assets_ff3, use="pairwise.complete.obs") |> round(2)
```

    ##         sp500 nmr_one nmr_sup   NMR   ETH Mkt.RF   SMB   HML    RF
    ## sp500    1.00    0.05   -0.01 -0.07  0.48   0.58  0.37  0.17 -0.04
    ## nmr_one  0.05    1.00    0.88 -0.15 -0.15  -0.10 -0.18  0.19 -0.26
    ## nmr_sup -0.01    0.88    1.00  0.12 -0.21  -0.22 -0.26  0.37 -0.23
    ## NMR     -0.07   -0.15    0.12  1.00  0.22   0.08  0.12 -0.25  0.02
    ## ETH      0.48   -0.15   -0.21  0.22  1.00   0.64  0.31  0.00 -0.12
    ## Mkt.RF   0.58   -0.10   -0.22  0.08  0.64   1.00  0.32  0.04 -0.05
    ## SMB      0.37   -0.18   -0.26  0.12  0.31   0.32  1.00  0.05 -0.09
    ## HML      0.17    0.19    0.37 -0.25  0.00   0.04  0.05  1.00 -0.11
    ## RF      -0.04   -0.26   -0.23  0.02 -0.12  -0.05 -0.09 -0.11  1.00

``` r
# FF3 regressions
FF3reg_nmr_one = lm(nmr_one~Mkt.RF+SMB+HML, data=assets_ff3)
summary(FF3reg_nmr_one)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ Mkt.RF + SMB + HML, data = assets_ff3)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -110.364  -15.877   -2.962   20.979   66.729 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  2.82856    4.89303   0.578    0.566
    ## Mkt.RF      -0.02812    0.07441  -0.378    0.707
    ## SMB         -0.16055    0.14001  -1.147    0.257
    ## HML          0.11714    0.08256   1.419    0.163
    ## 
    ## Residual standard error: 34.32 on 47 degrees of freedom
    ##   (2 observations deleted due to missingness)
    ## Multiple R-squared:  0.07364,    Adjusted R-squared:  0.01451 
    ## F-statistic: 1.245 on 3 and 47 DF,  p-value: 0.3039

``` r
FF3reg_nmr_sup = lm(nmr_sup~Mkt.RF+SMB+HML, data=assets_ff3)
summary(FF3reg_nmr_sup)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ Mkt.RF + SMB + HML, data = assets_ff3)
    ## 
    ## Residuals:
    ##    Min     1Q Median     3Q    Max 
    ## -80.04 -31.06 -11.64  16.12  80.94 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)  
    ## (Intercept)  -8.9580    12.8814  -0.695   0.4990  
    ## Mkt.RF       -0.1607     0.2122  -0.757   0.4625  
    ## SMB          -0.4598     0.3661  -1.256   0.2313  
    ## HML           0.5416     0.2724   1.988   0.0683 .
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 52.1 on 13 degrees of freedom
    ##   (36 observations deleted due to missingness)
    ## Multiple R-squared:  0.3013, Adjusted R-squared:   0.14 
    ## F-statistic: 1.868 on 3 and 13 DF,  p-value: 0.1848

``` r
FF3reg_NMR = lm(NMR~Mkt.RF+SMB+HML, data=assets_ff3)
summary(FF3reg_NMR)
```

    ## 
    ## Call:
    ## lm(formula = NMR ~ Mkt.RF + SMB + HML, data = assets_ff3)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -744.30 -209.81   -9.11  200.10 1038.20 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)  
    ## (Intercept)  31.6724    48.5483   0.652   0.5173  
    ## Mkt.RF        0.2874     0.7383   0.389   0.6988  
    ## SMB           1.1061     1.3891   0.796   0.4299  
    ## HML          -1.5016     0.8192  -1.833   0.0731 .
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 340.5 on 47 degrees of freedom
    ##   (2 observations deleted due to missingness)
    ## Multiple R-squared:  0.08299,    Adjusted R-squared:  0.02446 
    ## F-statistic: 1.418 on 3 and 47 DF,  p-value: 0.2494

``` r
FF3reg_ETH = lm(ETH~Mkt.RF+SMB+HML, data=assets_ff3)
summary(FF3reg_ETH)
```

    ## 
    ## Call:
    ## lm(formula = ETH ~ Mkt.RF + SMB + HML, data = assets_ff3)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -434.25 -129.38    7.98  115.38  572.82 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)  25.0110    31.9051   0.784    0.437    
    ## Mkt.RF        2.4985     0.4852   5.150 5.06e-06 ***
    ## SMB           0.9568     0.9129   1.048    0.300    
    ## HML          -0.1360     0.5383  -0.253    0.802    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 223.8 on 47 degrees of freedom
    ##   (2 observations deleted due to missingness)
    ## Multiple R-squared:  0.4233, Adjusted R-squared:  0.3864 
    ## F-statistic:  11.5 on 3 and 47 DF,  p-value: 8.972e-06

``` r
ff5url = "https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_5_Factors_2x3_CSV.zip"
# Define the file paths
zip_filepath = file.path(subdirectory, "FF5-factors.zip")
csv_filepath = file.path(subdirectory, "FF5-factors.csv")
# Download the zip file
download.file(ff5url, destfile=zip_filepath)
# Extract the CSV file from the zip file
unzip(zip_filepath, exdir=subdirectory)
file.rename("Factor Data/F-F_Research_Data_5_Factors_2x3.CSV", csv_filepath)
```

    ## [1] TRUE

``` r
FF5 = read_csv(csv_filepath,
               col_types = cols(...1 = col_date(format = "%Y%m")), 
               skip = 2)
```

    ## New names:
    ## • `` -> `...1`

    ## Warning: One or more parsing issues, call `problems()` on your data frame for details,
    ## e.g.:
    ##   dat <- vroom(...)
    ##   problems(dat)

``` r
FF5 = FF5 |> rename(Date=...1)
# Trim annual observations from bottom of date frame (dates import as missing)
FF5 = FF5[complete.cases(FF5),]
# For some reason, the factors are importing as characters, so this will fix those
FF5$`Mkt-RF` = as.numeric(FF5$`Mkt-RF`)*12
FF5$SMB = as.numeric(FF5$SMB)*12
FF5$HML = as.numeric(FF5$HML)*12
FF5$RMW = as.numeric(FF5$RMW)*12
FF5$CMA = as.numeric(FF5$CMA)*12
# Reformat to xts object
FF5xts = xts(FF5[,-1], order.by=FF5$Date)
# Remove data prior to first BTC observation
FF5xts = FF5xts[paste(startdate,"/",sep="")]
```

``` r
# Compile data frame of annualized crypto returns (un-annualize)
assets_ff5 = merge(XSreal, FF5xts)
#assets_ff5 = assets_ff5[complete.cases(assets_ff5),]
# Compute average annual returns, volatility, and correlations
Er_ff5 = colMeans(assets_ff5,na.rm=TRUE)
Er_ff5 |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH  Mkt.RF     SMB     HML     RMW     CMA 
    ##    8.75    2.38   -7.62   31.39   56.18   12.75    0.66    0.90    7.51    2.40 
    ##      RF 
    ##    0.14

``` r
sd_ff5 = apply(assets_ff5,2,sd,na.rm=TRUE)
sd_ff5 |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH  Mkt.RF     SMB     HML     RMW     CMA 
    ##   53.87   33.92   54.72  338.67  280.28   68.82   39.61   58.88   33.33   37.86 
    ##      RF 
    ##    0.17

``` r
Sharpe_ff5 = Er_ff5/sd_ff5
Sharpe_ff5 |> round(2)
```

    ##   sp500 nmr_one nmr_sup     NMR     ETH  Mkt.RF     SMB     HML     RMW     CMA 
    ##    0.16    0.07   -0.14    0.09    0.20    0.19    0.02    0.02    0.23    0.06 
    ##      RF 
    ##    0.84

``` r
cor(assets_ff5, use="pairwise.complete.obs") |> round(2)
```

    ##         sp500 nmr_one nmr_sup   NMR   ETH Mkt.RF   SMB   HML   RMW   CMA    RF
    ## sp500    1.00    0.05   -0.01 -0.07  0.48   0.58  0.42  0.17  0.05 -0.13 -0.04
    ## nmr_one  0.05    1.00    0.88 -0.15 -0.15  -0.10 -0.08  0.19  0.13  0.26 -0.26
    ## nmr_sup -0.01    0.88    1.00  0.12 -0.21  -0.22 -0.11  0.37  0.40  0.62 -0.23
    ## NMR     -0.07   -0.15    0.12  1.00  0.22   0.08  0.05 -0.25  0.00 -0.21  0.02
    ## ETH      0.48   -0.15   -0.21  0.22  1.00   0.64  0.29  0.00 -0.05 -0.19 -0.12
    ## Mkt.RF   0.58   -0.10   -0.22  0.08  0.64   1.00  0.34  0.04  0.08 -0.18 -0.05
    ## SMB      0.42   -0.08   -0.11  0.05  0.29   0.34  1.00  0.35 -0.41  0.01 -0.10
    ## HML      0.17    0.19    0.37 -0.25  0.00   0.04  0.35  1.00  0.17  0.67 -0.11
    ## RMW      0.05    0.13    0.40  0.00 -0.05   0.08 -0.41  0.17  1.00  0.17 -0.08
    ## CMA     -0.13    0.26    0.62 -0.21 -0.19  -0.18  0.01  0.67  0.17  1.00 -0.19
    ## RF      -0.04   -0.26   -0.23  0.02 -0.12  -0.05 -0.10 -0.11 -0.08 -0.19  1.00

``` r
# FF3 regressions
FF5reg_nmr_one = lm(nmr_one~Mkt.RF+SMB+HML+RMW+CMA, data=assets_ff5)
summary(FF5reg_nmr_one)
```

    ## 
    ## Call:
    ## lm(formula = nmr_one ~ Mkt.RF + SMB + HML + RMW + CMA, data = assets_ff5)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -103.223  -18.878    0.737   23.282   64.774 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept)  1.91331    5.13823   0.372    0.711
    ## Mkt.RF      -0.02713    0.08180  -0.332    0.742
    ## SMB         -0.05846    0.17876  -0.327    0.745
    ## HML          0.04737    0.13454   0.352    0.726
    ## RMW          0.06209    0.18454   0.336    0.738
    ## CMA          0.16982    0.18808   0.903    0.371
    ## 
    ## Residual standard error: 34.91 on 45 degrees of freedom
    ##   (2 observations deleted due to missingness)
    ## Multiple R-squared:  0.0821, Adjusted R-squared:  -0.01989 
    ## F-statistic: 0.805 on 5 and 45 DF,  p-value: 0.5522

``` r
FF5reg_nmr_sup = lm(nmr_sup~Mkt.RF+SMB+HML+RMW+CMA, data=assets_ff5)
summary(FF5reg_nmr_sup)
```

    ## 
    ## Call:
    ## lm(formula = nmr_sup ~ Mkt.RF + SMB + HML + RMW + CMA, data = assets_ff5)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -68.814 -25.306  -1.263  23.001  67.982 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)  
    ## (Intercept)  -6.4193    11.8098  -0.544    0.598  
    ## Mkt.RF       -0.2559     0.2034  -1.258    0.235  
    ## SMB           0.3242     0.5122   0.633    0.540  
    ## HML          -0.3028     0.4933  -0.614    0.552  
    ## RMW           0.5276     0.4714   1.119    0.287  
    ## CMA           1.0103     0.5397   1.872    0.088 .
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 47.06 on 11 degrees of freedom
    ##   (36 observations deleted due to missingness)
    ## Multiple R-squared:  0.5175, Adjusted R-squared:  0.2982 
    ## F-statistic:  2.36 on 5 and 11 DF,  p-value: 0.1094

``` r
FF5reg_NMR = lm(NMR~Mkt.RF+SMB+HML+RMW+CMA, data=assets_ff5)
summary(FF5reg_NMR)
```

    ## 
    ## Call:
    ## lm(formula = NMR ~ Mkt.RF + SMB + HML + RMW + CMA, data = assets_ff5)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -738.65 -224.76   22.99  186.23  982.53 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)
    ## (Intercept) 22.24370   50.64941   0.439    0.663
    ## Mkt.RF       0.01200    0.80629   0.015    0.988
    ## SMB          2.14569    1.76208   1.218    0.230
    ## HML         -2.14107    1.32619  -1.614    0.113
    ## RMW          1.71646    1.81906   0.944    0.350
    ## CMA          0.04456    1.85400   0.024    0.981
    ## 
    ## Residual standard error: 344.2 on 45 degrees of freedom
    ##   (2 observations deleted due to missingness)
    ## Multiple R-squared:  0.1032, Adjusted R-squared:  0.0035 
    ## F-statistic: 1.035 on 5 and 45 DF,  p-value: 0.4087

``` r
FF5reg_ETH = lm(ETH~Mkt.RF+SMB+HML+RMW+CMA, data=assets_ff5)
summary(FF5reg_ETH)
```

    ## 
    ## Call:
    ## lm(formula = ETH ~ Mkt.RF + SMB + HML + RMW + CMA, data = assets_ff5)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -419.79 -132.03   18.03  132.66  633.60 
    ## 
    ## Coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)  31.2476    33.5716   0.931    0.357    
    ## Mkt.RF        2.5461     0.5344   4.764 2.01e-05 ***
    ## SMB           0.2602     1.1679   0.223    0.825    
    ## HML           0.2174     0.8790   0.247    0.806    
    ## RMW          -0.6285     1.2057  -0.521    0.605    
    ## CMA          -0.7664     1.2289  -0.624    0.536    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 228.1 on 45 degrees of freedom
    ##   (2 observations deleted due to missingness)
    ## Multiple R-squared:  0.4262, Adjusted R-squared:  0.3625 
    ## F-statistic: 6.685 on 5 and 45 DF,  p-value: 9.839e-05
