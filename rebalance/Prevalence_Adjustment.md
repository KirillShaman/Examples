Prevalence Adjustment
================

``` r
library(wrapr)
library(numbers)
```

``` r
bal_size <- 8
n_imb_rep <- 80
d <- data.frame(
  prediction = c(
    rep((bal_size - 1)/bal_size, bal_size),
    rep(c(1/2, 1/2, 1/2, 1/4, 1/4, 1/4, 1/4, 1/4, 1/4), n_imb_rep),
    rep(1/bal_size, bal_size)),
  truth = c(
    rep(TRUE, (bal_size - 1)), FALSE,
    rep(c(TRUE, TRUE, FALSE, TRUE, rep(FALSE, 5)), n_imb_rep),
    TRUE, rep(FALSE, (bal_size - 1))
  ))
d$orig_row_id <- seq_len(nrow(d))

knitr::kable(head(d))
```

| prediction | truth | orig\_row\_id |
| ---------: | :---- | ------------: |
|      0.875 | TRUE  |             1 |
|      0.875 | TRUE  |             2 |
|      0.875 | TRUE  |             3 |
|      0.875 | TRUE  |             4 |
|      0.875 | TRUE  |             5 |
|      0.875 | TRUE  |             6 |

``` r
colMeans(subset(subset(d, select= -orig_row_id)))
```

    ## prediction      truth 
    ##  0.3369565  0.3369565

``` r
d %.>%
  subset(., select = -orig_row_id) %.>%
  aggregate(
    . ~ prediction, 
    data = ., 
    FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth |
| ---------: | --------: |
|      0.125 | 0.1250000 |
|      0.250 | 0.1666667 |
|      0.500 | 0.6666667 |
|      0.875 | 0.8750000 |

``` r
prevalence <- mean(d$truth)

prevalence
```

    ## [1] 0.3369565

``` r
epsilon <- 1.0e-9
stopifnot(abs(prevalence  -  mean(d$prediction)) < epsilon)
prevalence  -  mean(d$prediction)
```

    ## [1] 0

Build a deterministic re-sampled data set with 50% prevalence.

``` r
# get how many times to replicate each row group
count_table <- aggregate(
  count ~ truth, 
  data = transform(d, count = 1), 
  FUN = length)

multiple <- Reduce(LCM, count_table$count)
multiple
```

    ## [1] 15128

``` r
count_table$n_reps <- multiple / count_table$count

knitr::kable(count_table)
```

| truth | count | n\_reps |
| :---- | ----: | ------: |
| FALSE |   488 |      31 |
| TRUE  |   248 |      61 |

``` r
# replicate each row group by the target number of times
rep_table <- count_table %.>%
 lapply(
  seq_len(nrow(.)),
  function(i) {
    data.frame(
      truth = .$truth[[i]],
      row_rep = seq_len(.$n_reps[[i]]))
  }) %.>%
  do.call(rbind, .)

d_2 <- merge(d, rep_table, by = 'truth') %.>%
  .[order(.$orig_row_id, .$row_rep), ]
rownames(d_2) <- NULL

knitr::kable(head(d_2))
```

| truth | prediction | orig\_row\_id | row\_rep |
| :---- | ---------: | ------------: | -------: |
| TRUE  |      0.875 |             1 |        1 |
| TRUE  |      0.875 |             1 |        2 |
| TRUE  |      0.875 |             1 |        3 |
| TRUE  |      0.875 |             1 |        4 |
| TRUE  |      0.875 |             1 |        5 |
| TRUE  |      0.875 |             1 |        6 |

``` r
d_2$orig_row_id <- NULL
d_2$row_rep <- NULL
```

``` r
prevalence_2 <- mean(d_2$truth)

prevalence_2
```

    ## [1] 0.5

``` r
stopifnot(abs(prevalence_2  - mean(d_2$prediction)) > 1e-2)
mean(d_2$prediction)
```

    ## [1] 0.3594494

<https://win-vector.com/2020/10/10/upcoming-series-probability-model-homotopy/>

``` r
sigmoid <- function(x) {
  1 / (1 + exp(-x))
}

logit <- function(x) {
  log( x / (1 - x) )
}

delta <- -logit(prevalence) + logit(prevalence_2)

delta
```

    ## [1] 0.6768867

Add our p-adjusted prediction and show the intereseting rows.

``` r
d_2$p_adjusted_prediction <- sigmoid(
  logit(d_2$prediction) + delta)

aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction |
| ---------: | --------: | ----------------------: |
|      0.125 | 0.2194245 |               0.2194245 |
|      0.250 | 0.2824074 |               0.3961039 |
|      0.500 | 0.7973856 |               0.6630435 |
|      0.875 | 0.9323144 |               0.9323144 |

``` r
stopifnot(abs(prevalence_2  - mean(d_2$p_adjusted_prediction)) > 1e-2)
mean(d_2$p_adjusted_prediction)
```

    ## [1] 0.5105872

``` r
f <- function(d) {
  mean(d_2$truth) - mean(sigmoid(logit(d_2$prediction) + d))
}

delta_2 <- uniroot(f, c(-2, 2), tol = .Machine$double.eps)

delta_2
```

    ## $root
    ## [1] 0.630759
    ## 
    ## $f.root
    ## [1] 0
    ## 
    ## $iter
    ## [1] 6
    ## 
    ## $init.it
    ## [1] NA
    ## 
    ## $estim.prec
    ## [1] 1.245352e-08

``` r
d_2$u_adjusted_prediction <- sigmoid(
  logit(d_2$prediction) + delta_2$root)

aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction | u\_adjusted\_prediction |
| ---------: | --------: | ----------------------: | ----------------------: |
|      0.125 | 0.2194245 |               0.2194245 |               0.2116261 |
|      0.250 | 0.2824074 |               0.3961039 |               0.3851245 |
|      0.500 | 0.7973856 |               0.6630435 |               0.6526615 |
|      0.875 | 0.9323144 |               0.9323144 |               0.9293449 |

``` r
stopifnot(abs(prevalence_2  -  mean(d_2$u_adjusted_prediction)) < epsilon)
mean(d_2$u_adjusted_prediction)
```

    ## [1] 0.5

Platt scaling <https://en.wikipedia.org/wiki/Platt_scaling>

``` r
platt_scaler <- glm(
  truth ~ logit(prediction), 
  data = d_2, 
  family = binomial())

d_2$platt_scaled_prediction <- predict(
  platt_scaler,
  newdata = d_2,
  type = 'response')

aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction | u\_adjusted\_prediction | platt\_scaled\_prediction |
| ---------: | --------: | ----------------------: | ----------------------: | ------------------------: |
|      0.125 | 0.2194245 |               0.2194245 |               0.2116261 |                 0.0689418 |
|      0.250 | 0.2824074 |               0.3961039 |               0.3851245 |                 0.2896237 |
|      0.500 | 0.7973856 |               0.6630435 |               0.6526615 |                 0.7882819 |
|      0.875 | 0.9323144 |               0.9323144 |               0.9293449 |                 0.9946869 |

``` r
stopifnot(abs(prevalence_2  -  mean(d_2$platt_scaled_prediction)) < epsilon)
mean(d_2$platt_scaled_prediction)
```

    ## [1] 0.5

``` r
platt_shifter <- glm(
  truth ~ 1, 
  offset = logit(prediction),
  data = d_2, 
  family = binomial())

d_2$platt_shifted_prediction <- predict(
  platt_shifter,
  newdata = d_2,
  type = 'response')

aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction | u\_adjusted\_prediction | platt\_scaled\_prediction | platt\_shifted\_prediction |
| ---------: | --------: | ----------------------: | ----------------------: | ------------------------: | -------------------------: |
|      0.125 | 0.2194245 |               0.2194245 |               0.2116261 |                 0.0689418 |                  0.2116261 |
|      0.250 | 0.2824074 |               0.3961039 |               0.3851245 |                 0.2896237 |                  0.3851245 |
|      0.500 | 0.7973856 |               0.6630435 |               0.6526615 |                 0.7882819 |                  0.6526615 |
|      0.875 | 0.9323144 |               0.9323144 |               0.9293449 |                 0.9946869 |                  0.9293449 |

``` r
stopifnot(abs(prevalence_2  -  mean(d_2$platt_shifted_prediction)) < epsilon)
mean(d_2$platt_shifted_prediction)
```

    ## [1] 0.5