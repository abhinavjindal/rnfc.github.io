---
title: "Momentum, Markowitz, and Solving Rank-Deficient Covariance Matrices - The Constrained Critical Line Algorithm"
date: "June 5, 2015"
layout: post
---


This post will feature the differences in the implementation of my constrained critical line algorithm with that of Dr. Clarence Kwan's. The constrained critical line algorithm is a form of gradient descent that incorporates elements of momentum. My implementation includes a volatility-targeting binary search algorithm.

First off, rather than try and explain the algorithm piece by piece, I'll defer to Dr. Clarence Kwan's paper and excel spreadsheet, from where I obtained my original implementation. Since that paper and excel spreadsheet explains the functionality of the algorithm, I won't repeat that process here. Essentially, the constrained critical line algorithm incorporates its lambda constraints into the structure of the covariance matrix itself. This innovation actually allows the algorithm to invert previously rank-deficient matrices.

Now, while Markowitz mean-variance optimization may be a bit of old news for some, the ability to use a short lookback for momentum with monthly data has allowed me and my two coauthors (Dr. Wouter Keller, who came up with flexible and elastic asset allocation, and Adam Butler, of GestaltU) to perform a backtest on a century's worth of assets, with more than 30 assets in the backtest, despite using only a 12-month formation period. That paper can be found here.

Let's look at the code for the function.

```r
CCLA <- function(covMat, retForecast, maxIter = 1000, 
                 verbose = FALSE, scale = 252, 
                 weightLimit = .7, volThresh = .1) {
  if(length(retForecast) > length(unique(retForecast))) {
    sequentialNoise <- seq(1:length(retForecast)) * 1e-12
    retForecast <- retForecast + sequentialNoise
  }
  # Adds sequential noise if there are duplicates
  
  #initialize original out/in/up status
  if(length(weightLimit) == 1) {
    weightLimit <- rep(weightLimit, ncol(covMat))
  }
  # If only 1 weight limit specified, then it is assumed it is weight limit for all assets
  
  rankForecast <- length(retForecast) - rank(retForecast) + 1
  # Rank forecasts by descending
  remainingWeight <- 1 #have 100% of weight to allocate
  upStatus <- inStatus <- rep(0, ncol(covMat))
  # up and in status set at 0 for all asset classes
  
  i <- 1
  while(remainingWeight > 0) {
    securityLimit <- weightLimit[rankForecast == i]
    # Takes weightlimit for the highest rank forecast
    if(securityLimit < remainingWeight) {
      upStatus[rankForecast == i] <- 1 
      #if we can't invest all remaining weight into the security
      remainingWeight <- remainingWeight - securityLimit
      # Calculate remaining weight
    } else {
      inStatus[rankForecast == i] <- 1
      remainingWeight <- 0
      # Invested all remaining weight
    }
    i <- i + 1
    # Move on to seecond asset class
  }
  
  #initial matrices (W, H, K, identity, negative identity)
  covMat <- as.matrix(covMat)
  # Put covariance in matrix form
  retForecast <- as.numeric(retForecast)
  # put as Numeric
  init_W <- cbind(2*covMat, rep(-1, ncol(covMat)))
  init_W <- rbind(init_W, c(rep(1, ncol(covMat)), 0))
  # Add -1 as an extra column, add a row of 1s for each column, last column has a 0
  H_vec <- c(rep(0, ncol(covMat)), 1)
  K_vec <- c(retForecast, 0)
  negIdentity <- -1*diag(ncol(init_W))
  # -1 diagonal matrix 
  identity <- diag(ncol(init_W))
  # 1 diagional matrix
  matrixDim <- nrow(init_W)
  # Number of rows
  weightLimMat <- matrix(rep(weightLimit, matrixDim), ncol=ncol(covMat), byrow=TRUE)
  # Create a matrix with the weight limits
  
  #out status is simply what isn't in or up
  outStatus <- 1 - inStatus - upStatus
  
  #initialize expected volatility/count/turning points data structure
  expVol <- Inf
  lambda <- 100
  count <- 0
  turningPoints <- list()
  while(lambda > 0 & count < maxIter) { # Run 1000 times while lamba is bigger than -
    
    #old lambda and old expected volatility for use with numerical algorithms
    oldLambda <- lambda
    oldVol <- expVol
    # Save old values
    count <- count + 1
    # Increate count
    
    #compute W, A, B
    inMat <- matrix(rep(c(inStatus, 1), matrixDim), nrow = matrixDim, byrow = TRUE)
    upMat <- matrix(rep(c(upStatus, 0), matrixDim), nrow = matrixDim, byrow = TRUE)
    outMat <- matrix(rep(c(outStatus, 0), matrixDim), nrow = matrixDim, byrow = TRUE)
    # in, up, and out matrices
    
    W <- inMat * init_W + upMat * identity + outMat * negIdentity
    
    inv_W <- solve(W)
    modified_H <- H_vec - rowSums(weightLimMat* upMat[,-matrixDim] * init_W[,-matrixDim])
    A_vec <- inv_W %*% modified_H
    B_vec <- inv_W %*% K_vec
    
    #remove the last elements from A and B vectors
    truncA <- A_vec[-length(A_vec)]
    truncB <- B_vec[-length(B_vec)]
    
    #compute in Ratio (aka Ratio(1) in Kwan.xls)
    inRatio <- rep(0, ncol(covMat))
    inRatio[truncB > 0] <- -truncA[truncB > 0]/truncB[truncB > 0]
    
    #compute up Ratio (aka Ratio(2) in Kwan.xls)
    upRatio <- rep(0, ncol(covMat))
    upRatioIndices <- which(inStatus==TRUE & truncB < 0)
    if(length(upRatioIndices) > 0) {
      upRatio[upRatioIndices] <- (weightLimit[upRatioIndices] - truncA[upRatioIndices]) / truncB[upRatioIndices]
    }
    
    #find lambda -- max of up and in ratios
    maxInRatio <- max(inRatio)
    maxUpRatio <- max(upRatio)
    lambda <- max(maxInRatio, maxUpRatio)
    
    #compute new weights
    wts <- inStatus*(truncA + truncB * lambda) + upStatus * weightLimit + outStatus * 0
    
    #compute expected return and new expected volatility
    expRet <- t(retForecast) %*% wts
    expVol <- sqrt(wts %*% covMat %*% wts) * sqrt(scale)
    
    #create turning point data row and append it to turning points
    turningPoint <- cbind(count, expRet, lambda, expVol, t(wts))
    colnames(turningPoint) <- c("CP", "Exp. Ret.", "Lambda", "Exp. Vol.", colnames(covMat))
    turningPoints[[count]] <- turningPoint
    
    #binary search for volatility threshold -- if the first iteration is lower than the threshold,
    #then immediately return, otherwise perform the binary search until convergence of lambda
    if(oldVol == Inf & expVol < volThresh) {
      turningPoints <- do.call(rbind, turningPoints)
      threshWts <- tail(turningPoints, 1)
      return(list(turningPoints, threshWts))
    } else if(oldVol > volThresh & expVol < volThresh) {
      upLambda <- oldLambda
      dnLambda <- lambda
      meanLambda <- (upLambda + dnLambda)/2
      while(upLambda - dnLambda > .00001) {
        
        #compute mean lambda and recompute weights, expected return, and expected vol
        meanLambda <- (upLambda + dnLambda)/2
        wts <- inStatus*(truncA + truncB * meanLambda) + upStatus * weightLimit + outStatus * 0
        expRet <- t(retForecast) %*% wts
        expVol <- sqrt(wts %*% covMat %*% wts) * sqrt(scale)
        
        #if new expected vol is less than threshold, mean becomes lower bound
        #otherwise, it becomes the upper bound, and loop repeats
        if(expVol < volThresh) {
          dnLambda <- meanLambda
        } else {
          upLambda <- meanLambda
        }
      }
      
      #once the binary search completes, return those weights, and the corner points
      #computed until the binary search. The corner points aren't used anywhere, but they're there.
      threshWts <- cbind(count, expRet, meanLambda, expVol, t(wts))
      colnames(turningPoint) <- colnames(threshWts) <- c("CP", "Exp. Ret.", "Lambda", "Exp. Vol.", colnames(covMat))
      turningPoints[[count]] <- turningPoint
      turningPoints <- do.call(rbind, turningPoints)
      return(list(turningPoints, threshWts))
    }
    
    #this is only run for the corner points during which binary search doesn't take place
    #change status of security that has new lambda
    if(maxInRatio > maxUpRatio) {
      inStatus[inRatio == maxInRatio] <- 1 - inStatus[inRatio == maxInRatio]
      upStatus[inRatio == maxInRatio] <- 0
    } else {
      upStatus[upRatio == maxUpRatio] <- 1 - upStatus[upRatio == maxUpRatio]
      inStatus[upRatio == maxUpRatio] <- 0
    }
    outStatus <- 1 - inStatus - upStatus
  }
  
  #we only get here if the volatility threshold isn't reached
  #can actually happen if set sufficiently low
  turningPoints <- do.call(rbind, turningPoints)
  
  threshWts <- tail(turningPoints, 1)
  
  return(list(turningPoints, threshWts))
}
```

Essentially, the algorithm can be divided into three parts:

The first part is the initialization, which does the following:

It creates three status vectors: in, up, and out. The up vector denotes which securities are at their weight constraint cap, the in status are securities that are not at their weight cap, and the out status are securities that receive no weighting on that iteration of the algorithm.

The rest of the algorithm essentially does the following:

It takes a gradient descent approach by changing the status of the security that minimizes lambda, which by extension minimizes the volatility at the local point. As long as lambda is greater than zero, the algorithm continues to iterate. Letting the algorithm run until convergence effectively provides the volatility-minimization portfolio on the efficient frontier.

However, one change that Dr. Keller and I made to it is the functionality of volatility targeting, allowing the algorithm to stop between iterations. As the SSRN paper shows, a higher volatility threshold, over the long run (the *VERY* long run) will deliver higher returns.

In any case, the algorithm takes into account several main arguments:

A return forecast, a covariance matrix, a volatility threshold, and weight limits, which can be either one number that will result in a uniform weight limit, or a per-security weight limit. Another argument is scale, which is 252 for days, 12 for months, and so on. Lastly, there is a volatility threshold component, which allows the user to modify how aggressive or conservative the strategy can be.

In any case, to demonstrate this function, let's run a backtest. The idea in this case will come from a recent article published by Frank Grossmann from SeekingAlpha, in which he obtained a 20% CAGR but with a 36% max drawdown.

So here's the backtest:


```
## Error in library(quantstrat): there is no package called 'quantstrat'
```



```r
symbols <- c("AFK", "ASHR", "ECH", "EGPT",
             "EIDO", "EIRL", "EIS", "ENZL",
             "EPHE", "EPI", "EPOL", "EPU",
             "EWA", "EWC", "EWD", "EWG",
             "EWH", "EWI", "EWJ", "EWK",
             "EWL", "EWM", "EWN", "EWO",
             "EWP", "EWQ", "EWS", "EWT",
             "EWU", "EWW", "EWY", "EWZ",
             "EZA", "FM", "FRN", "FXI",
             "GAF", "GULF", "GREK", "GXG",
             "IDX", "MCHI", "MES", "NORW",
             "QQQ", "RSX", "THD", "TUR",
             "VNM", "TLT"
)

getSymbols(symbols, from = "2003-01-01")
# Get symbols since 2003

prices <- list()
entryRets <- list()
for(i in 1:length(symbols)) {
  prices[[i]] <- Ad(get(symbols[i]))
}
prices <- do.call(cbind, prices)
colnames(prices) <- gsub("\\.[A-z]*", "", colnames(prices))
# Use adjusted prices

returns <- Return.calculate(prices)
returns <- returns[-1,]
# Calculate returns

sumIsNa <- function(col) {
  return(sum(is.na(col)))
}
# Sum of returns excluding NAs

appendZeroes <- function(selected, originalSetNames) {
  zeroes <- rep(0, length(originalSetNames) - length(selected))
  # Get 0s for the number of names not selected
  names(zeroes) <- originalSetNames[!originalSetNames %in% names(selected)]
  # Name the zeroes
  all <- c(selected, zeroes)
  all <- all[originalSetNames]
  # Return the original
  return(all)
}
# Set everything not selected to zeroes

computeStats <- function(rets) {
  stats <- rbind(table.AnnualizedReturns(rets), maxDrawdown(rets), CalmarRatio(rets))
  return(round(stats, 3))
}
# Compute statistics

CLAAbacktest <- function(returns, lookback = 3, volThresh = .1, assetCaps = .5, tltCap = 1,
                         returnWeights = FALSE, useTMF = FALSE) {
  if(useTMF) {
    returns$TLT <- returns$TLT * 3
  }
  # Option to use 3X leveraged TLT instead
  
  ep <- endpoints(returns, on = "months")
  # get monthly endpoints
  
  weights <- list()
  for(i in 2:(length(ep) - lookback)) {
    retSubset <- returns[(ep[i]+1):ep[i+lookback],]
    # For all returns in lookback period
    retNAs <- apply(retSubset, 2, sumIsNa)
    # Sum up the NAs in the columns
    validRets <- retSubset[, retNAs==0]
    # Only want those columns with no NAs
    retForecast <- Return.cumulative(validRets)
    # Get cumulative returns
    covRets <- cov(validRets)
    # Covariance of returns
    weightLims <- rep(assetCaps, ncol(covRets))
    # Set weight limits for all assets
    weightLims[colnames(covRets)=="TLT"] <- tltCap
    # Seperate cap for the bond
    weight <- CCLA(covMat = covRets, retForecast = retForecast, weightLimit = weightLims, volThresh = volThresh)
    weight <- weight[[2]][,5:ncol(weight[[2]])]
    # Select the weights
    weight <- appendZeroes(selected = weight, colnames(retSubset))
    weight <- xts(t(weight), order.by=last(index(validRets)))
    weights[[i]] <- weight
  }
  weights <- do.call(rbind, weights)
  # Cobmine weights
  stratRets <- Return.portfolio(R = returns, weights = weights)
  
  if(returnWeights) {
    return(list(weights, stratRets))
  }
  return(stratRets)
  # Return results
}
```

In essence, we take the returns over a specified monthly lookback period, specify a volatility threshold, specify asset caps, specify the bond asset cap, and whether or not we wish to use TLT or TMF (a 3x leveraged variant, which just multiplies all returns of TLT by 3, for simplicity). The output of the CCLA (Constrained Critical Line Algorithm) is a list that contains the corner points, and the volatility threshold corner point which contains the corner point number, expected return, expected volatility, and the lambda value. So, we want the fifth element onward of the second element of the list.

Here are some results:


```r
config1 <- CLAAbacktest(returns = returns)
config2 <- CLAAbacktest(returns = returns, useTMF = TRUE)
config3 <- CLAAbacktest(returns = returns, lookback = 4)
config4 <- CLAAbacktest(returns = returns, lookback = 2, useTMF = TRUE)

comparison <- na.omit(cbind(config1, config2, config3, config4))
colnames(comparison) <- c("Default", "TMF instead of TLT", "Lookback 4", "Lookback 2 and TMF")
```

Here's the equity curve:

```r
charts.PerformanceSummary(comparison)
```

<img src="/public/images/2015-06-05-Markowitz/unnamed-chunk-5-1.png" title="plot of chunk unnamed-chunk-5" alt="plot of chunk unnamed-chunk-5" style="display: block; margin: auto;" />


With the following statistics:


```r
kable(computeStats(comparison))
```



|                          | Default| TMF instead of TLT| Lookback 4| Lookback 2 and TMF|
|:-------------------------|-------:|------------------:|----------:|------------------:|
|Annualized Return         |   0.122|              0.131|      0.119|              0.127|
|Annualized Std Dev        |   0.126|              0.144|      0.124|              0.148|
|Annualized Sharpe (Rf=0%) |   0.965|              0.911|      0.957|              0.855|
|Worst Drawdown            |   0.219|              0.344|      0.186|              0.357|
|Calmar Ratio              |   0.557|              0.382|      0.640|              0.355|

The variants that use TMF instead of TLT suffer far worse drawdowns. Not much of a hedge, apparently.

Taking the 4 month lookback configuration (strongest Calmar), we'll play around with the volatility setting.

Here's the backtest:


```r
config5 <- CLAAbacktest(returns = returns, lookback = 4, volThresh = .15)
config6 <- CLAAbacktest(returns = returns, lookback = 4, volThresh = .2)

comparison2 <- na.omit(cbind(config3, config5, config6))
colnames(comparison2) <- c("Vol10", "Vol15", "Vol20")
```

With the results:

Here's the equity curve.

```r
charts.PerformanceSummary(comparison2)
```

<img src="/public/images/2015-06-05-Markowitz/unnamed-chunk-8-1.png" title="plot of chunk unnamed-chunk-8" alt="plot of chunk unnamed-chunk-8" style="display: block; margin: auto;" />

With the results:

```r
kable(computeStats(comparison2))
```



|                          | Vol10| Vol15| Vol20|
|:-------------------------|-----:|-----:|-----:|
|Annualized Return         | 0.119| 0.135| 0.153|
|Annualized Std Dev        | 0.124| 0.173| 0.206|
|Annualized Sharpe (Rf=0%) | 0.957| 0.778| 0.741|
|Worst Drawdown            | 0.186| 0.218| 0.275|
|Calmar Ratio              | 0.640| 0.619| 0.554|

In this case, more risk, more reward, lower risk/reward ratios as you push the volatility threshold. So for once, the volatility puzzle doesn't rear its head, and higher risk indeed does translate to higher returns (at the cost of everything else, though).

Lastly, let's try toggling the asset cap limits with the vol threshold back at 10.


```r
config7 <- CLAAbacktest(returns = returns, lookback = 4, assetCaps = .1)
config8 <- CLAAbacktest(returns = returns, lookback = 4, assetCaps = .25)
config9 <- CLAAbacktest(returns = returns, lookback = 4, assetCaps = 1/3)
config10 <- CLAAbacktest(returns = returns, lookback = 4, assetCaps = 1)

comparison3 <- na.omit(cbind(config7, config8, config9, config3, config10))
colnames(comparison3) <- c("Cap10", "Cap25", "Cap33", "Cap50", "Uncapped")
```

Here's the equity curve:

```r
charts.PerformanceSummary(comparison3)
```

<img src="/public/images/2015-06-05-Markowitz/unnamed-chunk-11-1.png" title="plot of chunk unnamed-chunk-11" alt="plot of chunk unnamed-chunk-11" style="display: block; margin: auto;" />

With the results:


```r
kable(computeStats(comparison3))
```



|                          | Cap10| Cap25| Cap33| Cap50| Uncapped|
|:-------------------------|-----:|-----:|-----:|-----:|--------:|
|Annualized Return         | 0.105| 0.106| 0.110| 0.119|    0.121|
|Annualized Std Dev        | 0.118| 0.123| 0.124| 0.124|    0.125|
|Annualized Sharpe (Rf=0%) | 0.892| 0.859| 0.895| 0.957|    0.973|
|Worst Drawdown            | 0.194| 0.199| 0.192| 0.186|    0.186|
|Calmar Ratio              | 0.544| 0.530| 0.574| 0.640|    0.652|

Essentially, in this case, there was very little actual change from simply tweaking weight limits. Here's an equity curve:

To conclude, while not exactly achieving the same aggregate returns or Sharpe ratio that the SeekingAlpha article did, it did highlight a probable cause of its major drawdown, and also demonstrated the levers of how to apply the constrained critical line algorithm, the mechanics of which are detailed in the papers linked to earlier.
