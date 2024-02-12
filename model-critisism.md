---
title: 'Model checking'
teaching: 10
exercises: 2
---




:::::::::::::::::::::::::::::::::::::: questions 

- What is model checking?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Prior/Posterior predictive check

- Model comparison with 
  - AIC, BIC, WAIC
  
- Bayesian cross-validation

::::::::::::::::::::::::::::::::::::::::::::::::

This episode focuses on model checking, a crucial step in Bayesian data analysis when dealing with competing models that require systematic comparison. We'll explore three different approaches for this purpose.

Firstly, we'll delve into posterior predictive checks, a method that involves comparing a fitted model's predictions with observed data.

Next, we'll examine information criteria, a tool that measures the balance between model complexity and goodness-of-fit.

We'll end the episode with an exploration of Bayesian cross-validation.

Throughout the episode, we'll use the same simulated dataset for examples. 

## Data

For data, we're using $N=88$ univariate numerical data. Looking at a histogram, it's evident that the data is approximately symmetrically distributed around 0. However, there is some dispersion in the values, suggesting that the tails might be longer than those of the normal distribution. Next, we'll compare the suitability of the normal and Cauchy distributions on this data. 



```{.output}
 [1]  -2.270   1.941   0.502  -0.378  -0.226  -0.786  -0.209  -0.637   0.814
[10]   0.566  -1.901  -2.047  -0.689  -3.509   0.133  -4.353   1.067   0.722
[19]   0.861   0.523   0.681   2.982   0.429  -0.539  -0.512  -1.090  -8.044
[28]  -0.387  -0.007 -11.126   1.036   1.734  -0.203   1.036   0.582  -2.922
[37]  -0.543  -6.120  -0.649   4.547  -0.867   1.942   7.148  -0.044  -0.681
[46]  -3.461  -0.142   0.678   0.644  -0.039   0.354   1.783   0.369   0.175
[55]   0.980  -0.097  -4.408   0.442   0.158   0.255   0.084   0.775   2.786
[64]   0.008  -0.664  43.481   1.943   0.334  -0.118   3.901   1.736  -0.665
[73]   2.695   0.002  -1.904  -2.194  -4.015   0.329   1.140  -3.816 -14.788
[82]   0.047   6.205   1.119  -0.003   3.618   1.666 -10.845
```

<img src="fig/model-critisism-rendered-unnamed-chunk-1-1.png" style="display: block; margin: auto;" />


## Posterior predictive check

The idea of posterior predictive checking is to use the posterior predictive distribution to simulate a replicate data set and compare it to the observed data. The reasoning behind this approach is, as formulated in BDA3 p.143, "If the model fits, then replicated data generated under the model should look similar to observed data."

Any qualitative discrepancies between the simulated and observed data can imply shortcomings in the model that do not match the properties of the data or the domain. Comparison between simulated and actual data can be done in different ways. Visual check is one option but a more rigorous approach is to compute the posterior predictive p-value (ppp), which measures how well the the model can reproduce the observed data. 

The posterior predictive check, utilizing ppp, can be formulated as follows: 


1. Generate replicate data:
  Use the posterior predictive distribution to simulate new datasets $X^{rep}$ with characteristics matching the observed data. In our example, this amounts to generating a large number of replications with sample size $N=88$. 
2. Choose test quantity $T(X)$:
  Choose an aspect of the data that you wish to check. We'll use the maximum value of the data as the test quantity and compute it for the observed data and for each replication: $T(X^{rep})$. 
3. Compute ppp:
  The posterior predictive p-value defined as the probability $Pr(T(X^{rep}) \geq T(X) | X)$, that is the probability that the predictions produce test quantities at least as extreme as those found in the data. Using samples, it is computed as the proportion of replicate data sets with $T$ not smaller than that of $T(X)$. 
  

A small ppp-value would indicate that the model doesn't capture the properties of the data.


Next, we'll perform a posterior predictive check on the example data and compare the results for the normal and Cauchy models. 

### Normal model


We'll use a basic Stan program for the normal model and produce the replicate data in the generated quantities block. Notice that `X_rep` is a vector with length equal to the sample size $N$. The values of `X_rep` are generated in a loop. Notice that a single posterior value of $(\mu, \sigma)$ is used for each evaluation of the generated quantities block; the values do not change in the iterations of the for loop. 



```stan
data {
  int<lower=0> N;
  vector[N] X;
}
parameters {
  real<lower=0> sigma;
  real mu;
}
model {
  X ~ normal(mu, sigma);
  
  mu ~ normal(0, 1);
  sigma ~ gamma(2, 1);
}

generated quantities {
  vector[N] X_rep;

  for(i in 1:N) {
    X_rep[i] = normal_rng(mu, sigma);
  }
}

```



Fit model and extract replicates. 


```r
# Fit
normal_fit <- sampling(normal_model,
                       list(N = N, X = df$X), 
                       refresh = 0)

# Extract 
X_rep <- rstan::extract(normal_fit, "X_rep")[[1]] %>% 
  data.frame() %>%
  mutate(sample = 1:nrow(.))
```



Below is a comparison of 12 samples of $X^{rep}$ against the data (the panel titles correspond to MCMC sample numbers). The discrepancy between the data and replicates indicates an issue with the model choice. It seems like the normal model underestimates the data tails.

<img src="fig/model-critisism-rendered-unnamed-chunk-4-1.png" style="display: block; margin: auto;" />



Let's quantify this discrepancy by computing the ppp using the maximum of the data as a test statistic. The maximum of the original data is max($X$) = 43.481. The following histogram shows this value (vertical line) against the maximum computed for each replicate data set $X^{rep}$. 



```r
# Compute X_rep max
rep_maxs <- X_rep %>%
  select(-sample) %>%
  apply(MARGIN = 1, FUN = max) %>%
  data.frame(max = ., sample = 1:length(.))

ggplot() +
  geom_histogram(data = rep_maxs,
                 aes(x = max),
                 bins = 50, fill = posterior_color) +
  geom_vline(xintercept = max(df$X)) +
  labs(title = "Max value of the replicate data sets")
```

<img src="fig/model-critisism-rendered-unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

The proportion of replications $X_{rep}$ that produce at least as extreme values as the data is called the posterior predictive p-value ($ppp$). The $ppp$ quantifies the evidence for the suitability of the model for the data with higher $ppp$ implying a lesser conflict. In this case, the value is $ppp =$ 1 which means that the maximum was as at least as large as in the data in 0% replications. This indicates strong evidence that the normal model is a poor choice for the data. 


### Cauchy model

Let's do a similar analysis utilizing the Cauchy model, and compute the $ppp$ for this model. 

The code used is essentially copy-pasted from above, with the distinction of the Stan program.


```stan
data {
  int<lower=0> N;
  vector[N] X;
}
parameters {
  // Scale
  real<lower=0> sigma;
  // location
  real mu;
}
model {
  // location = mu and scale = sigma
  X ~ cauchy(mu, sigma);
  
  mu ~ normal(0, 1);
  sigma ~ gamma(2, 1);
}
generated quantities {
  vector[N] X_rep;
  for(i in 1:N) {
    X_rep[i] = cauchy_rng(mu, sigma);
  }
}

```


With the cauchy model there is little discrepancy between the data and replicate data:

<img src="fig/model-critisism-rendered-unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

The maximum observed value is close to the average of the distribution of maximum value for replicate sets. Moreoever, the $ppp$ is large, indicating no issues with the suitability of the model on the data. 


```r
## Compute ppp
rep_maxs <- X_rep %>%
  select(-sample) %>%
  apply(MARGIN = 1, FUN = max) %>%
  data.frame(max = ., sample = 1:length(.))

ggplot() +
  geom_histogram(data = rep_maxs,
                 aes(x = max),
                 bins = 10000, fill = posterior_color) +
  geom_vline(xintercept = max(df$X)) +
  labs(title = paste0("ppp = ", mean(rep_maxs$max > max(df$X)))) +
  # set plot limits to aid with visualizations
  coord_cartesian(xlim = c(0, 1000)) 
```

<img src="fig/model-critisism-rendered-unnamed-chunk-8-1.png" style="display: block; margin: auto;" />



## Information criteria


Information criteria are statistics used in model selection and comparison within both framework of Bayesian and classical frequentist statistics. The aim of these criteria is to estimate out-of-sample predictive accuracy, and to provide a principled approach to assess the relative performance of competing models.

The Widely Applicable Information Criterion (WAIC) is an of information criteria that was developed within the Bayesian paradigm. WAIC is computed using the log pointwise predictive density, lppd, and a penalization term, $p_{WAIC}$: 

$$WAIC = -2(\text{lppd} - p_{WAIC}).$$

The log pointwise predictive density is computed as $\sum_{i=1}^N \log(\frac{1}{S} \sum_{s=1}^S p(X_i | \theta^s), $, where $X_i, \,i=1,\ldots,N$ are data points and $S$ the number of posterior samples. Since the predictive density is computed on the data used to fit the model, the estimate may be over-confident. The penalization term $p_{WAIC} = \sum_{i=1}^N \text{Var}(\log p(y_i | \theta^s))$ correct for this bias

Lower WAIC values imply better fit. 

Let's then compare the normal and Cauchy models with the WAIC. First we'll need to fit both models on the data. 


```r
stan_data <- list(N = N, X = df$X)

# Fit
normal_fit <- sampling(normal_model, stan_data,
                       refresh = 0)
cauchy_fit <- sampling(cauchy_model, stan_data, 
                       refresh = 0)


# Extract samples
normal_samples <- rstan::extract(normal_fit, c("mu", "sigma")) %>% data.frame
cauchy_samples <- rstan::extract(cauchy_fit, c("mu", "sigma")) %>% data.frame
```


Then we can write a function that compute the WAIC. 


```r
WAIC <- function(samples, data, model){
  
  # Loop over data points
  pp_dens <- lapply(1:length(data), function(i) {
    
    my_x <- data[i]
    
    # Loop over posterior samples  
    point_pp_dens <- lapply(1:nrow(samples), function(S) {
      
      my_mu <- samples[S, "mu"]
      my_sigma <- samples[S, "sigma"]
      
      if(model == "normal") {
        # Model: y ~ normal(mu, sigma)
        dnorm(x = my_x,
              mean = my_mu,
              sd = my_sigma)
      } else if(model == "cauchy") {
        # Model: y ~ cauchy(mu, sigma)
        dcauchy(x = my_x,
                location = my_mu,
                scale = my_sigma)
      }
      
    }) %>%
      unlist()
    
    return(point_pp_dens)
    
  }) %>%
    do.call(rbind, .)
  
  
  # See BDA3 p.169
  lppd <- apply(X = pp_dens,
                MARGIN = 1, 
                FUN = function(x) log(mean(x))) %>% 
    sum
  
  # See BDA3 p.173
  bias <- apply(X = pp_dens,
                MARGIN = 1, 
                FUN = function(x) var(log(x))) %>% 
    sum
  
  # WAIC
  waic = -2*(lppd - bias)
  
  return(waic)
}
```

Applying this function to the posterior samples, we'll recover a lower value for the Cauchy model, implying a better fit on the data. 


```r
WAIC(normal_samples, df$X, model = "normal")
```

```{.output}
[1] 582.2829
```

```r
WAIC(cauchy_samples, df$X, model = "cauchy")
```

```{.output}
[1] 413.9462
```


## Bayesian cross-validation

Idea in Bayesian cross-validation is.... 


Helper function that computes.. 


```r
# Get log predictive density for a point x,
# given data X and posterior samples
# See BDA3 p.175
get_lpd <- function(x, X, samples, model) {
  
  # Loop over posterior samples  
  pp_dens <- lapply(1:nrow(samples), function(S) {
    
    if(model == "normal") {
      # Normal(x | mu, sigma^2)
      dnorm(x = x,
            mean = samples[S, "mu"],
            sd = samples[S, "sigma"])
    } else if (model == "cauchy") {
      # Cauchy(x | location = mu, scale = sigma^2)
      dcauchy(x = x,
              location = samples[S, "mu"],
              scale = samples[S, "sigma"])
    }
    
    
  }) %>%
    unlist()
  
  lpd <- log(mean(pp_dens))
  
  return(lpd)
}
```


Now we can perform CV


```r
# Loop over data partitions
normal_loo_lpds <- lapply(1:N, function(i) {
  
  # Training set
  my_X <- X[-i]
  
  # Test set
  my_x <- X[i]
  
  # Fit model
  my_normal_fit <- sampling(normal_model,
                            list(N = length(my_X),
                                 X = my_X),
                            refresh = 0 # omits output
                            ) 
  
  # Get data
  my_samples <- rstan::extract(my_normal_fit, c("mu", "sigma")) %>% 
    do.call(cbind, .) %>% 
    set_colnames(c("mu", "sigma"))
  
  # Get lpd
  my_lpd <- get_lpd(my_x, my_X, my_samples, "normal")
  
  data.frame(i, lpd = my_lpd, model = "normal_loo")
  
}) %>%
  do.call(rbind, .)

# Predictive density for data points using full data in training
normal_full_lpd <- lapply(1, function(dummy) {
  
  # Fit model
  my_normal_fit <- sampling(normal_model,
                            list(N = length(X),
                                 X = X), 
                            refresh = 0)
  
  # Get data
  my_samples <- rstan::extract(my_normal_fit, c("mu", "sigma")) %>% 
    do.call(cbind, .) %>% 
    set_colnames(c("mu", "sigma"))
  
  # Compute lpds
  lpds <- lapply(1:N, function(i) {
    
    my_lpd <- get_lpd(X[i], X, my_samples, "normal")
    
    data.frame(i, lpd = my_lpd, model = "normal")
  }) %>% do.call(rbind, .)
  
  return(lpds)
}) %>%
  do.call(rbind, .)


# Same for Cauchy:
cauchy_loo_lpds <- lapply(1:N, function(i) {
  
  # Subset data
  my_X <- X[-i]
  my_x <- X[i]
  
  # Fit model
  my_normal_fit <- sampling(cauchy_model,
                            list(N = length(my_X),
                                 X = my_X), 
                            refresh = 0)
  
  # Get data
  my_samples <- rstan::extract(my_normal_fit, c("mu", "sigma")) %>% 
    do.call(cbind, .) %>% 
    set_colnames(c("mu", "sigma"))
  
  # Get lpd
  my_lpd <- get_lpd(my_x, my_X, my_samples, "cauchy")
  
  data.frame(i, lpd = my_lpd, model = "cauchy_loo")
  
}) %>%
  do.call(rbind, .)

cauchy_full_lpd <- lapply(1, function(dummy) {
  
  # Fit model
  my_cachy_fit <- sampling(cauchy_model,
                           list(N = length(X),
                                X = X), 
                           refresh = 0)
  
  # Get data
  my_samples <- rstan::extract(my_cachy_fit, c("mu", "sigma")) %>% 
    do.call(cbind, .) %>% 
    set_colnames(c("mu", "sigma"))
  
  # Compute lpds
  lpds <- lapply(1:N, function(i) {
    
    my_lpd <- get_lpd(X[i], X, my_samples, "cauchy")
    
    data.frame(i, lpd = my_lpd, model = "cauchy")
  }) %>% do.call(rbind, .)
  
  return(lpds)
}) %>%
  do.call(rbind, .)
```



```r
# Combine
lpds <- rbind(normal_loo_lpds, 
              normal_full_lpd, 
              cauchy_loo_lpds,
              cauchy_full_lpd)

lpd_summary <- lpds %>% 
  group_by(model) %>% 
  summarize(lppd = sum(lpd))


# Effective number of parameters
p_loo_cv_normal <- lpd_summary[lpd_summary$model == "normal", "lppd"] - lpd_summary[lpd_summary$model == "normal_loo", "lppd"]
p_loo_cv_cauchy <- lpd_summary[lpd_summary$model == "cauchy", "lppd"] - lpd_summary[lpd_summary$model == "cauchy_loo", "lppd"]


paste0("Effective number of parameters, normal = ", p_loo_cv_normal)
```

```{.output}
[1] "Effective number of parameters, normal = 33.7046896782354"
```

```r
paste0("Effective number of parameters, cauchy = ", p_loo_cv_cauchy)
```

```{.output}
[1] "Effective number of parameters, cauchy = 1.903135942788"
```




::::::::::::::::::::::::::::::::::::: keypoints 

- point 1

::::::::::::::::::::::::::::::::::::::::::::::::



## Reading

- Statistical Rethinking: Ch. 7
- BDA3: p.143: 6.3 Posterior predictive checking
