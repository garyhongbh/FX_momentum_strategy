# Systematic FX Strategy
In this project, the team has created a simplified quantitative strategy by constructing a L/S porfolio in USD/CNH forward contracts. The rationale is to exploit the interest rate differentials driven by economic cycles between U.S. and China, and the portfolio composition is adjusted based on momentum signals derived from macroeconomic data and Treasury yield spread. For ease of tracking portfolio returns and to set an anchor for the initial entry point, the portfolio pnl is aggregated into an index, which has an arbitrary starting value of 1,000.


## Investment rationale
Given the driving force behind interest rate differentials is the economic cyles between two countries, the strategy provides a quantitative approach to infer the current momentum of economic strength based on CPI, PMI and 10Y Treasury bond yield spread. The goal is offer a balanced exposure to dollar/yuan carry, and to generate stable returns.


## Portfolio specification
| Items                   | Description                                                  |
| ---------               | ---------------------                                        |
| Underlying              | 18M and 6M USD/CNH forward contracts.                        |
| Rebalance               | Monthly on the 3rd date of each calendar month.              |
| Rebalancing indicators  | Signals from CPI YoY, PMI, 10Y Treasury and 10Y CGB yields.  |
| Portfolio starting day  | January 2nd, 2018.                                           |
| Target volatility       | Annualized 5%.                                               |
| Notional                | Adjusted according to portfolio volatility                   |


## Methodology
To figure out if the portfolio should long or short forward contracts, it is important to first determine the relative economic strength between 2 countries. Economic strengths reflected by rising CPI or PMI indicates the power of one currency relative to others. In this strategy, economic strengths are determined through momentum signals based on CPI YoY, PMI and 10Y goverment bonds yield differentials. This analysis will output 3 signals used for portfolio rebalancing, namely "USD Strength", "RMB Strength" and "Neutral". On the first date of each calendar month, the signals are generated according to a layered approach. If the signal is "Neutral" after the 1st evaluation (i.e. Layer 1), then another method (i.e. Layer 2) will be used to evaluate if the signal remains to be "Neutral". The details are in the below description.

### Step 1: Signal generation

#### Layer 1: 
The first step is to compute the CPI difference between China and U.S. (i.e. China CPI - U.S. CPI) for every calendar months, the result are displayed in the 'CPI_diff' column in df (i.e. variable name in Python script). Next, the programme computes the exponential moving average (EMA) of CPI difference, which is applied using a 12-period lookback window and a smoothing factor of 2. The results are shown in 'EMA' column in df and the formula is given by

$`\text{EMA}_{\text{i}} = \frac{2}{N + 1} * \text{CPI diff}_{\text{i}} + (1 - \frac{2}{N + 1}) * \text{CPI diff}_{\text{i-1}}  `$

Where:
- N: no. of periods in lookback window
- i: current period

Comparing 'CPI_diff' with 'EMA' column in df. If the CPI difference is greater than EMA consecutively for 6 months, then it indicates "RMB Strength". If the CPI difference is smaller than EMA consecutively for 6 months, then it indicates "USD Strength". If the signal is "Neutral", Layer 2 would be implemented for further analysis.

#### Layer 2:
If the CPI difference between China and U.S. remain positive for 6 months, then it signals "RMB Strengths". If the CPI difference remain negatative for 6 months. Then it signals "USD Strengths". If the signal remains to be "Neutral", Layer 3 would be adopted.

#### Layer 3:
In this layer, the PMI difference between China and U.S. (i.e. China PMI - U.S. PMI) is computed for every calendar months. Following the same methodology in Layer 2, if the difference remains positive for 6 months, then it indicates "RMB Strengths", conversely it means "USD Strengths".

#### Layer 4:
Should the signal remains "Neutral", the analysis would move to the final layer. Here, yield differentials are computed every between 10Y CGBs and 10Y Treasury bonds (i.e. 10Y CGB yield - 10Y Treasury yield) for every calendar months. Since bond yields are of a higher frequency data compared to CPI YoY and PMI, a shorter lookback period is used to compute its effect on relative currency power. If the yield differential is increasing consecutively for the past 3 months, then it entails "RMB Strengths". If the yield differential is declining consecutively for the past 3 months, it entails "USD Strengths". If all the Layers combined failed to dissect relative currency power, the signal would be deemed as "Neutral".


### Step 2: Portfolio construction

#### Composition:
The portfolio composition is determiend by the signals from Step 1. When the signal shows "RMB Strengths", the portfolio will have a short position in long-dated USD/CNH forward (i.e. 18M) along with a long position in short-dated USD/CNH forward (i.e. 6M). When the signal shows "USD Strengths", the portfolio will have a long position in long-dated forward and a short position in short-dated forward.

Over the month between each rebalancing, the long-dated forward will roll down from 18 months to 17 months, and the short-dated forward will roll down from 6 months to 5 months. On rebalancing date, if the signals remain the same the forward contracts will be rolled into a new pair of 18 months vs. 6 months forwards, otherwise, apart from rolling into a new pair of forwards the L/S direction will invert as well.

#### Daily return:
The return from L/S positions in forward contracts are reflected in forward points (forward points = forward rate - spot rate).

Example:
Suppose spot USD/CNH is 7.2 and the 1M USD/CNH forward rate is 7.3. For a long position in 1M USD/CNH forward (i.e. you deliver USD and receive CNH at contract expiration), if everything else remains the same, the long position is expected to gain 10 pips (a.k.a. basis points).

In the portfolio, there is NO delivery on rebalancing date, and the daily return are computed following the below methods:
1. The portfolio forward points are computed by taking the difference of forward points for the pair of 
    forwards. The formula is subject to the trade direction (i.e. RMB or USD strengths).

2. Daily return is calculated as

    $` \text{Daily ret}_{\text{i}} = \frac{\text{Port forward points}_{\text{i}} - \text{Port forward points}_{\text{i-1}}/ 10000}{\text{USD/CNH}_{\text{i-1}}} `$

#### Interpolation between each rebalancing date:
As the pair of forwards roll down from the current period to the next period, the tenor of the forward contract will be reduced and the forward curve would also change due to constant trading activity. To accurately compute the portfolio forward points for the entire period, an interpolation is needed to compute the forward points in between each rebalancing date. 

The project adopts a cubic spline method to interpolate forward points between each month. In brief, by using 5M, 6M, 12M and 18M USD/CNH forward points, a piecewise polynomial is constructed for every calcualtion period to connect the data points. Since the polynomial is twice differentiable, it is possible to use the slope and the concavity (i.e. curvature) of the function to infer the interpolated value in between each tenor (i.e. between 18M and 12M vs. between 6M and 5M).

After obtaining the interpolated forward points, the daily return for every period can then be calculated using the formula mentioned above.

#### Target vol (volatility):
An exponentially weighted model is used to compute the portfolio vol. The model adopts a 21-period lookback window and a smoothing factor of 2. 

$` \text{Daily vol}_{\text{i}} = \sqrt{(1 - \frac{2}{N + 1}) * \text{Daily vol}^2_{\text{i-1}} + \frac{2}{N + 1} * \text{Simple vol}^2_{\text{i}}}`$

Where:
- N: no. of periods in lookback window
- Simple vol: calculated using previous 21-days of daily return with sample standard deviation formula

The initial period Daily vol is set to be equal to Simple vol in the same period. For standardization, daily vol computed via the exponentially weighted model is being annualized by multiplying $` \sqrt{252} `$.


### Step 3: Index value computation

#### Notional:
The portfolio vol is adjusted through leverage, which is the ratio between target vol (i.e. 5%) and annualized vol. Note the annualized vol input to compute leverage is lagged by 2-periods to reflect T+2 settlement for FX transactions. The maximum allowed leverage is 15. During monthly rebalancing, if annualized vol is below target vol, leverage is applied to the portfolio, which determines the portfolio notional.

$` \text{Port notional}_{\text{i}} = \text{Index value}_{\text{i-2}} * \text{Leverage ratio}_{\text{i}}`$

Where:
- $`\text{Leverage ratio}_{\text{i}} = \frac{0.05}{\text{Annualized vol}_{\text{i-2}}}`$

#### Portfolio profit and loss (pnl):
The daily portfolio pnl is determined based on notional and the daily return calculated using portfolio forward points as well as spot USD/CNH.

$` \text{PnL}_{\text{i}} = \frac{\text{Port forward points}_{\text{i}} - \text{Port forward points}_{\text{i-1}}/ 10000}{\text{USD/CNH}_{\text{i-1}}} * \text{Port notional}_{\text{i}}`$

#### Index value:
Finally, the index value on each day is calcalued as below.

$`\text{Index value}_{\text{i}} = \text{Index value}_{\text{i-1}} + \text{PnL}_{\text{i}} `$


