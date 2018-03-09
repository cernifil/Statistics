---
title: "Multiple Group Comparisons"
output:
  html_document:
    code_folding: hide
    keep_md: true
---


## Setup


```r
library(gridExtra)
library(pairwiseCI)
library(car)
library(agricolae)


# create some data 
my_samples <- c("zyg","two","eight")

n_group <- 3

set.seed(1)
zyg <- rnorm(n_group, mean = 0.9, sd = 0.025)

set.seed(1)
two <- rnorm(n_group, mean = 0.6, sd = 0.1)

set.seed(1)
eight <- rnorm(n_group, mean = 0.7, sd = 0.075)


my_data <- data.frame(fractions = c(zyg, two, eight),
                      samples = factor(rep(my_samples, each = n_group), levels = my_samples))

n_all <- nrow(my_data)

my_data
```

```
##   fractions samples
## 1 0.8843387     zyg
## 2 0.9045911     zyg
## 3 0.8791093     zyg
## 4 0.5373546     two
## 5 0.6183643     two
## 6 0.5164371     two
## 7 0.6530160   eight
## 8 0.7137732   eight
## 9 0.6373279   eight
```




```r
xbar <- tapply(my_data$fractions, my_data$samples, mean, na.rm = TRUE)

par(mfrow=c(1,2))
barplot(xbar, ylim=c(0,1), ylab="Fraction",  main="Errorbars? Test?")
```

<img src="confint_files/figure-html/unnamed-chunk-2-1.png" style="display: block; margin: auto;" />



## Confidence Intervals

### Independent


```r
### shortcut ###
t.test(zyg)
```

```
## 
## 	One Sample t-test
## 
## data:  zyg
## t = 114.45, df = 2, p-value = 7.633e-05
## alternative hypothesis: true mean is not equal to 0
## 95 percent confidence interval:
##  0.8559129 0.9227798
## sample estimates:
## mean of x 
## 0.8893463
```

```r
ind_CI_lower <- tapply(my_data$fractions, my_data$samples, function(x){t.test(x)$conf.int[1]})
ind_CI_upper <- tapply(my_data$fractions, my_data$samples, function(x){t.test(x)$conf.int[2]})

### details ###

ind_CI_lower2 <- xbar - tapply(my_data$fractions, my_data$samples, 
                           function(x){qt(0.975, df = n_group-1)*sd(x)/sqrt(n_group)})

ind_CI_upper2 <- xbar + tapply(my_data$fractions, my_data$samples, 
                           function(x){qt(0.975, df = n_group-1)*sd(x)/sqrt(n_group)})

# check
c(
identical(round(as.numeric(ind_CI_lower),6), round(as.numeric(ind_CI_lower2),6)),
identical(round(as.numeric(ind_CI_upper),6), round(as.numeric(ind_CI_upper2),6))
)
```

```
## [1] TRUE TRUE
```

### Pooled SD


```r
### shortcut ###
lm_fit <- lm(fractions ~ 0 + samples, data = my_data)

summary(lm_fit)
```

```
## 
## Call:
## lm(formula = fractions ~ 0 + samples, data = my_data)
## 
## Residuals:
##      Min       1Q   Median       3Q      Max 
## -0.04095 -0.02003 -0.01024  0.01524  0.06098 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## sampleszyg    0.88935    0.02288   38.88 1.93e-08 ***
## samplestwo    0.55739    0.02288   24.37 3.14e-07 ***
## sampleseight  0.66804    0.02288   29.20 1.07e-07 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 0.03962 on 6 degrees of freedom
## Multiple R-squared:  0.998,	Adjusted R-squared:  0.997 
## F-statistic:   986 on 3 and 6 DF,  p-value: 1.813e-08
```

```r
pool_CI_lower <- confint(lm_fit)[,1]
pool_CI_upper <- confint(lm_fit)[,2]

### details ###
s <- tapply(my_data$fractions, my_data$samples, sd, na.rm = TRUE)
s
```

```
##        zyg        two      eight 
## 0.01345876 0.05383504 0.04037628
```

```r
n <- tapply(!is.na(my_data$fractions), my_data$samples, sum)
degf <- n - 1
degf
```

```
##   zyg   two eight 
##     2     2     2
```

```r
total.degf <- sum(degf)
total.degf
```

```
## [1] 6
```

```r
pooled.sd <- sqrt(sum(s^2*degf)/total.degf)
pooled.sd
```

```
## [1] 0.03962151
```

```r
pool_CI_lower2 <- xbar -  qt(0.975, df = total.degf)*pooled.sd*sqrt(1/n_group)
pool_CI_upper2 <- xbar +  qt(0.975, df = total.degf)*pooled.sd*sqrt(1/n_group)

# check
c(
identical(round(as.numeric(pool_CI_lower),6), round(as.numeric(pool_CI_lower2),6)),
identical(round(as.numeric(pool_CI_upper),6), round(as.numeric(pool_CI_upper2),6))
)
```

```
## [1] TRUE TRUE
```



```r
# plot
par(mfrow=c(1,2))
bplot <- barplot(xbar, ylim=c(0,1), ylab="Fraction",  main="Independent")
arrows(x0=bplot,y0=ind_CI_upper,y1=ind_CI_lower,angle=90,code=3,length=0.1)

bplot <- barplot(xbar, ylim=c(0,1), ylab="Fraction",  main="Pooled")
arrows(x0=bplot,y0=pool_CI_upper,y1=pool_CI_lower,angle=90,code=3,length=0.1)
```

<img src="confint_files/figure-html/unnamed-chunk-5-1.png" style="display: block; margin: auto;" />






## CI on Differences

### pairwiseCI


```r
# shortcut #
pair_CI <- pairwiseCI(fractions ~ samples, data = my_data, var.equal=TRUE)

pair_CI_diffs   <-  pair_CI[[1]][[1]]$estimate
pair_CI_lower <- pair_CI[[1]][[1]]$lower
pair_CI_upper <- pair_CI[[1]][[1]]$upper


pair_CI
```

```
##   
## 95 %-confidence intervals 
##  Method:  Difference of means assuming Normal distribution and equal variances 
##   
##   
##           estimate   lower   upper
## two-zyg    -0.3320 -0.4209 -0.2430
## eight-zyg  -0.2213 -0.2895 -0.1531
## eight-two   0.1107  0.0028  0.2185
##   
## 
```

```r
### details ###
pair_CI_lower2 <- c(t.test(two,zyg, var.equal = T)$conf.int[1], 
                    t.test(eight,zyg, var.equal = T)$conf.int[1], 
                    t.test(eight,two, var.equal = T)$conf.int[1])
pair_CI_upper2 <- c(t.test(two,zyg, var.equal = T)$conf.int[2], 
                    t.test(eight,zyg, var.equal = T)$conf.int[2], 
                    t.test(eight,two, var.equal = T)$conf.int[2])

# or
my_CI <- function(x,y){
    diff <- mean(x)-mean(y)
    s1 <- sd(x)
    s2 <- sd(y)
    se <- sqrt( (1/n_group + 1/n_group) * ((n_group-1)*s1^2 + (n_group-1)*s2^2)/(n_group+n_group-2) ) 
    df <- n_group+n_group-2
    diff + (qt(0.975, df = df)*se)*c(-1,1) 
}

pair_CI_lower3 <- c(my_CI(two,zyg)[1],my_CI(eight,zyg)[1],my_CI(eight,two)[1])
pair_CI_upper3 <- c(my_CI(two,zyg)[2],my_CI(eight,zyg)[2],my_CI(eight,two)[2])


# check
c(
identical(round(as.numeric(pair_CI_lower),6), round(as.numeric(pair_CI_lower2),6)),
identical(round(as.numeric(pair_CI_upper),6), round(as.numeric(pair_CI_upper2),6)),
identical(round(as.numeric(pair_CI_lower),6), round(as.numeric(pair_CI_lower3),6)),
identical(round(as.numeric(pair_CI_upper),6), round(as.numeric(pair_CI_upper3),6))
)
```

```
## [1] TRUE TRUE TRUE TRUE
```

### Anova + Tukey


```r
### Anova ###
aov_fit <- aov(fractions ~ samples, data = my_data)

summary(aov_fit)
```

```
##             Df  Sum Sq Mean Sq F value   Pr(>F)    
## samples      2 0.17142 0.08571    54.6 0.000141 ***
## Residuals    6 0.00942 0.00157                     
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

```r
### side track on anove F-stat ##
resids <- c(zyg-mean(zyg), two-mean(two), eight-mean(eight))
SSE <- sum(resids^2)
MSE <-  (SSE/(length(resids)-length(my_samples)))
MSE
```

```
## [1] 0.001569864
```

```r
sqrt(MSE) == pooled.sd
```

```
## [1] TRUE
```

```r
SST <- sum((c(zyg,two,eight)-mean(c(zyg,two,eight)))^2)
SSR <- SST - SSE
MSR <- (SSR/(length(my_samples)-1))
MSR
```

```
## [1] 0.08570963
```

```r
Fstat <- MSR / MSE
Fstat
```

```
## [1] 54.59683
```

```r
pval <- pf(q = Fstat, df1 = length(my_samples)-1, df2 = n_all-length(my_samples), lower.tail = FALSE)
pval
```

```
## [1] 0.0001413084
```

```r
# check
identical(round(as.numeric(pval),6), round(as.numeric(summary(aov_fit)[[1]][1,5]),6))
```

```
## [1] TRUE
```

```r
### Tukey test ##

posthoc <- TukeyHSD(x=aov_fit, conf.level=0.95)

posthoc
```

```
##   Tukey multiple comparisons of means
##     95% family-wise confidence level
## 
## Fit: aov(formula = fractions ~ samples, data = my_data)
## 
## $samples
##                 diff         lwr        upr     p adj
## two-zyg   -0.3319610 -0.43122221 -0.2326997 0.0001232
## eight-zyg -0.2213073 -0.32056855 -0.1220461 0.0011690
## eight-two  0.1106537  0.01139243  0.2099149 0.0326382
```

```r
tuk_diffs   <-  posthoc$samples[,1]
tuk_CI_lower <- posthoc$samples[,2]
tuk_CI_upper <- posthoc$samples[,3]


# details #
q_tukey <- qtukey(p = 0.95, nmeans = length(my_samples), df = total.degf)

tuk_CI_lower2 <- tuk_diffs - (q_tukey * sqrt( (MSE/2) * (1/(n_group) + 1/(n_group))))
tuk_CI_upper2 <- tuk_diffs + (q_tukey * sqrt( (MSE/2) * (1/(n_group) + 1/(n_group))))


# check
c(
identical(round(as.numeric(tuk_CI_lower),6), round(as.numeric(tuk_CI_lower2),6)),
identical(round(as.numeric(tuk_CI_upper),6), round(as.numeric(tuk_CI_upper2),6))
)
```

```
## [1] TRUE TRUE
```



```r
# plot
par(mfrow=c(1,2))
bplot <- barplot(pair_CI_diffs, ylim=c(-0.5,0.5), ylab="Fraction",  main="pairwiseCI", las=2)
arrows(x0=bplot,y0=pair_CI_upper,y1=pair_CI_lower,angle=90,code=3,length=0.1)

bplot <- barplot(tuk_diffs, ylim=c(-0.5,0.5), ylab="Fraction",  main="AOV + Tukey", las=2)
arrows(x0=bplot,y0=tuk_CI_upper,y1=tuk_CI_lower,angle=90,code=3,length=0.1)
```

<img src="confint_files/figure-html/unnamed-chunk-8-1.png" style="display: block; margin: auto;" />



### Pair-wise t-test on Pooled SD


```r
# no adjustment for comparison with lm
pairdiff <- pairwise.t.test(x = my_data$fractions, g = my_data$samples, p.adjust.method = "none")

# outputs p-value
pairdiff
```

```
## 
## 	Pairwise comparisons using t tests with pooled SD 
## 
## data:  my_data$fractions and my_data$samples 
## 
##       zyg     two    
## two   5e-05   -      
## eight 0.00048 0.01414
## 
## P value adjustment method: none
```


```r
# from the function (pooled sd same as above)
x = my_data$fractions
g = my_data$samples

xbar <- tapply(x, g, mean, na.rm = TRUE)
s <- tapply(x, g, sd, na.rm = TRUE)
n <- tapply(!is.na(x), g, sum)
degf <- n - 1
total.degf <- sum(degf)
pooled.sd <- sqrt(sum(s^2 * degf)/total.degf)
dif <- xbar[2] - xbar[1]

my_CI <- function(x,y){
    dif <- mean(x) - mean(y)
    se.dif <- pooled.sd * sqrt(1/length(x) + 1/length(y))
    t.val <- dif/se.dif
    pval <- 2 * pt(-abs(t.val), total.degf)
    CI <- dif + qt(0.975, total.degf)*se.dif*c(-1,1)
    return(CI)
}

my_pval <- function(x,y){
    dif <- mean(x) - mean(y)
    se.dif <- pooled.sd * sqrt(1/length(x) + 1/length(y))
    t.val <- dif/se.dif
    pval <- 2 * pt(-abs(t.val), total.degf)
    return(pval)
}

pooldiff_CI_lower <- c(my_CI(two,zyg)[1],my_CI(eight,zyg)[1],my_CI(eight,two)[1])
pooldiff_CI_upper <- c(my_CI(two,zyg)[2],my_CI(eight,zyg)[2],my_CI(eight,two)[2])
pooldiff_pval <- c(my_pval(two,zyg)[1],my_pval(eight,zyg)[1],my_pval(eight,two)[1])
pooldiffs <- c(mean(two)-mean(zyg), mean(eight)-mean(zyg), mean(eight)-mean(two))
names(pooldiffs) <- names(pair_CI_diffs)
```


### Linear Regression


```r
# fit intercept
lm_fit2 <- lm(fractions ~ samples, data = my_data)
summary(lm_fit2)
```

```
## 
## Call:
## lm(formula = fractions ~ samples, data = my_data)
## 
## Residuals:
##      Min       1Q   Median       3Q      Max 
## -0.04095 -0.02003 -0.01024  0.01524  0.06098 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept)   0.88935    0.02288  38.878 1.93e-08 ***
## samplestwo   -0.33196    0.03235 -10.261 5.00e-05 ***
## sampleseight -0.22131    0.03235  -6.841  0.00048 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 0.03962 on 6 degrees of freedom
## Multiple R-squared:  0.9479,	Adjusted R-squared:  0.9306 
## F-statistic:  54.6 on 2 and 6 DF,  p-value: 0.0001413
```

```r
# relevel for 3rd comparison
my_data$samples <- relevel(my_data$samples, ref = "two")

lm_fit3 <- lm(fractions ~ samples, data = my_data)
summary(lm_fit3)
```

```
## 
## Call:
## lm(formula = fractions ~ samples, data = my_data)
## 
## Residuals:
##      Min       1Q   Median       3Q      Max 
## -0.04095 -0.02003 -0.01024  0.01524  0.06098 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept)   0.55739    0.02288   24.37 3.14e-07 ***
## sampleszyg    0.33196    0.03235   10.26 5.00e-05 ***
## sampleseight  0.11065    0.03235    3.42   0.0141 *  
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 0.03962 on 6 degrees of freedom
## Multiple R-squared:  0.9479,	Adjusted R-squared:  0.9306 
## F-statistic:  54.6 on 2 and 6 DF,  p-value: 0.0001413
```

```r
# get values 
lm_fit_pval <- c(summary(lm_fit2)$coef[2:3,4], summary(lm_fit3)$coef[3,4]) 
lm_fit_diffs <- c(summary(lm_fit2)$coef[2:3,1], summary(lm_fit3)$coef[3,1]) 
names(lm_fit_diffs) <- names(pair_CI_diffs)
lm_fit_CI_lower <- c(confint(lm_fit2)[2:3,1], confint(lm_fit3)[3,1])
lm_fit_CI_upper <- c(confint(lm_fit2)[2:3,2], confint(lm_fit3)[3,2])


# check
c(
identical(round(as.numeric(lm_fit_CI_lower),6), round(as.numeric(pooldiff_CI_lower),6)),
identical(round(as.numeric(lm_fit_CI_upper),6), round(as.numeric(pooldiff_CI_upper),6)),
identical(round(as.numeric(lm_fit_pval),6), round(as.numeric(pooldiff_pval),6))
)
```

```
## [1] TRUE TRUE TRUE
```


```r
# plot
par(mfrow=c(1,4), cex=1.2)

bplot <- barplot(pair_CI_diffs, ylim=c(-0.5,0.5), ylab="Fraction",  main="pairwiseCI", las=2)
arrows(x0=bplot,y0=pair_CI_upper,y1=pair_CI_lower,angle=90,code=3,length=0.1)

bplot <- barplot(tuk_diffs, ylim=c(-0.5,0.5), ylab="Fraction",  main="AOV + Tukey", las=2)
arrows(x0=bplot,y0=tuk_CI_upper,y1=tuk_CI_lower,angle=90,code=3,length=0.1)

bplot <- barplot(lm_fit_diffs, ylim=c(-0.5,0.5), ylab="Fraction", main="Pw t-test Pooled SD", las=2)
arrows(x0=bplot,y0=lm_fit_CI_upper,y1=lm_fit_CI_lower,angle=90,code=3,length=0.1)

bplot <- barplot(pooldiffs, ylim=c(-0.5,0.5), ylab="Fraction",  main="Linear Regression", las=2)
arrows(x0=bplot,y0=pooldiff_CI_upper,y1=pooldiff_CI_lower,angle=90,code=3,length=0.1)
```

<img src="confint_files/figure-html/unnamed-chunk-12-1.png" style="display: block; margin: auto;" />



```r
### further details ###

mm <- model.matrix(lm_fit2)
mm
```

```
##   (Intercept) samplestwo sampleseight
## 1           1          0            0
## 2           1          0            0
## 3           1          0            0
## 4           1          1            0
## 5           1          1            0
## 6           1          1            0
## 7           1          0            1
## 8           1          0            1
## 9           1          0            1
## attr(,"assign")
## [1] 0 1 1
## attr(,"contrasts")
## attr(,"contrasts")$samples
## [1] "contr.treatment"
```

```r
XX <- solve(t(mm) %*% mm)
XX
```

```
##              (Intercept) samplestwo sampleseight
## (Intercept)    0.3333333 -0.3333333   -0.3333333
## samplestwo    -0.3333333  0.6666667    0.3333333
## sampleseight  -0.3333333  0.3333333    0.6666667
```

```r
sqrt(diag(XX * MSE))
```

```
##  (Intercept)   samplestwo sampleseight 
##   0.02287549   0.03235083   0.03235083
```

```r
mm <- model.matrix(lm_fit3)
mm
```

```
##   (Intercept) sampleszyg sampleseight
## 1           1          1            0
## 2           1          1            0
## 3           1          1            0
## 4           1          0            0
## 5           1          0            0
## 6           1          0            0
## 7           1          0            1
## 8           1          0            1
## 9           1          0            1
## attr(,"assign")
## [1] 0 1 1
## attr(,"contrasts")
## attr(,"contrasts")$samples
## [1] "contr.treatment"
```

```r
XX <- solve(t(mm) %*% mm)
XX
```

```
##              (Intercept) sampleszyg sampleseight
## (Intercept)    0.3333333 -0.3333333   -0.3333333
## sampleszyg    -0.3333333  0.6666667    0.3333333
## sampleseight  -0.3333333  0.3333333    0.6666667
```

```r
sqrt(diag(XX * MSE))
```

```
##  (Intercept)   sampleszyg sampleseight 
##   0.02287549   0.03235083   0.03235083
```

```r
# without intercept 
mm <- model.matrix(lm_fit)
mm
```

```
##   sampleszyg samplestwo sampleseight
## 1          1          0            0
## 2          1          0            0
## 3          1          0            0
## 4          0          1            0
## 5          0          1            0
## 6          0          1            0
## 7          0          0            1
## 8          0          0            1
## 9          0          0            1
## attr(,"assign")
## [1] 1 1 1
## attr(,"contrasts")
## attr(,"contrasts")$samples
## [1] "contr.treatment"
```

```r
XX <- solve(t(mm) %*% mm)
XX
```

```
##              sampleszyg samplestwo sampleseight
## sampleszyg    0.3333333  0.0000000    0.0000000
## samplestwo    0.0000000  0.3333333    0.0000000
## sampleseight  0.0000000  0.0000000    0.3333333
```

```r
sqrt(diag(XX * MSE))
```

```
##   sampleszyg   samplestwo sampleseight 
##   0.02287549   0.02287549   0.02287549
```



Some links:

https://stats.stackexchange.com/questions/210515/confidence-intervals-for-group-means-r

https://stats.stackexchange.com/questions/126588/different-confidence-intervals-from-direct-calculation-and-rs-confint-function

https://rstudio-pubs-static.s3.amazonaws.com/181709_eec7a5bc24c04b8badeb297c4807109a.html

https://stats.stackexchange.com/questions/174861/calculating-regression-sum-of-square-in-r

https://en.wikipedia.org/wiki/Tukey%27s_range_test




