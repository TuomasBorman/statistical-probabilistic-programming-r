---
title: 'Hierarchical Models'
teaching: 10
exercises: 2
---



:::::::::::::::::::::::::::::::::::::: questions 

- What are Bayesian hierarchical models?
- What are they good for?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Understand the idea of hierarchical models
- Learn how to build and with hierarchical models with Stan

::::::::::::::::::::::::::::::::::::::::::::::::



## Hierachical models

Bayesian hierarchical models are a class of models suited for modeling scenarios where the study population consists of separate but related groups. Hierarchical structure refers to this organization of data into multiple levels or groups, where each level can have its own set of parameters. These parameters are connected trough a common prior that is also learned when fitting the model. Some or all of the hyperparameters of the priors are unknown model parameters and they are given hyperpriors. 

One key advantage of Bayesian hierarchical models is their ability to borrow strength across groups. By pooling information from multiple groups, these models can provide more stable estimates, especially when individual groups have limited data. This pooling of information is particularly beneficial when there are sparse observations or when data from different groups exhibit similar patterns.


Examples of scenarios where hierarchical model could be a natural choice: 



## Example: Hierarchical binomial model

Let's take a look at a hierarchical binomial model. Let $X = \{X_1, X_2, \ldots, X_N\}$ be a set of observations representing the number of successes in $n$ Bernoulli trials in $N$ different scenarios. We assume that these scenarios are not identical, so there are $N$ unknown probability parameters, $p_1, p_2, \ldots, p_N$. This model can be specified as follows.

\begin{align}
X_i &\sim Binom(n, p_i) \\
p_i &\sim Beta(\alpha, \beta) \\
\alpha, \beta &\sim Gamma(2, 1).
\end{align}

The difference to the binomial model as used in the previous episodes is that the parameters $p_i$ have a prior with unknown hyperparameters $\alpha$ and $\beta.$ These hyperparameters are given a $Gamma$ prior and learned in the inference. 

:::::::::::::::::::::::::::::::::::::: challenge

Hierarchical models are also called partially pooled models in contrast to unpooled and completely pooled models. The former mean that the model parameters are assumed to be completely independent, while in the latter type the (parallel) parameters are equal. Write the unpooled and completely pooled variants of the hierarchical binomial model. 

:::::::::::::::::::: solution

Unpooled: 

\begin{align}
X_i &\sim Binom(n, p_i) \\
p_i &\sim Beta(2, 2) \\
\end{align}

Completely pooled: 

\begin{align}
X_i &\sim Binom(n, p) \\
p &\sim Beta(2, 2) \\
\end{align}

:::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::



# Data

Let's analyze human adult height in different countries. We'll use the normal model with unknown mean $\mu$ and standard deviation $\sigma$ as the generative model, and give these parameters a hierarchical treatment. 

We'll simulate some data based on measured averages and standard deviations for boys and girls in different countries. 

Data source: Height: Height and body-mass index trajectories of school-aged children and adolescents from 1985 to 2019 in 200 countries and territories: a pooled analysis of 2181 population-based studies with 65 million participants. Lancet 2020, 396:1511-1524

Data structure: 


```r
df_full <- read.csv("data/height_data.csv")

str(df_full)
```

```{.output}
'data.frame':	210000 obs. of  8 variables:
 $ Country                                   : chr  "Afghanistan" "Afghanistan" "Afghanistan" "Afghanistan" ...
 $ Sex                                       : chr  "Boys" "Boys" "Boys" "Boys" ...
 $ Year                                      : int  1985 1985 1985 1985 1985 1985 1985 1985 1985 1985 ...
 $ Age.group                                 : int  5 6 7 8 9 10 11 12 13 14 ...
 $ Mean.height                               : num  103 109 115 120 125 ...
 $ Mean.height.lower.95..uncertainty.interval: num  92.9 99.9 106.3 112.2 117.9 ...
 $ Mean.height.upper.95..uncertainty.interval: num  114 118 123 128 132 ...
 $ Mean.height.standard.error                : num  5.3 4.72 4.27 3.92 3.66 ...
```

Let's subset this data to simplify the analysis:


```r
df_full_sub <- df_full %>% 
  filter(
    # 2019 measurements
    Year == 2019,        
    # use only a single age group
    Age.group == 19, 
    # Consider girls only
    Sex == "Girls"
    ) %>% 
  # Select variables of interest. Use mu and sigma for mean and sd
  select(Country, Sex, mu = Mean.height, sigma = Mean.height.standard.error)
```

Let's select 10 countries randomly


```r
# Select countries
N_countries <- 10
Countries <- sample(unique(df_full_sub$Country),
                    size = N_countries,
                    replace = FALSE) %>% sort

df <- df_full_sub %>% filter(Country %in% Countries)

df
```

```{.output}
                Country   Sex       mu     sigma
1               Albania Girls 162.2286 0.7286584
2               Bahamas Girls 163.4605 1.5458758
3               Bolivia Girls 155.5752 0.9389964
4              DR Congo Girls 156.3007 0.8423344
5      French Polynesia Girls 166.5187 1.0898402
6                  Iran Girls 161.1837 0.4042259
7               Liberia Girls 156.5414 0.8957401
8                 Malta Girls 162.9505 3.7687254
9                Tuvalu Girls 163.5701 1.0383303
10 United Arab Emirates Girls 160.5303 0.8292111
```


## Simulate data

Now, we can treat the values in the table above as ground truth and simulate some data based on them. Below, we'll analyze these simulated data points and see how well we can recover the "true" parameters for each country. 


```r
# Sample size per group 
N <- 25

# For each country, generate some random girl's heights
Height <- lapply(1:nrow(df), function(i) {
  
  my_df <- df[i, ]
  
  data.frame(Country = my_df$Country, 
             Sex = my_df$Sex, 
             # Random normal values based on measured mu and sd
             Height = rnorm(N, my_df$mu, my_df$sigma))

}) %>% 
  do.call(rbind, .)


# Plot
Height %>% 
  ggplot() +
  geom_point(aes(x = Country, y = Height, color = Sex)) + 
  coord_flip() + 
  labs(title = "Simulated data")
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

# Modeling

Let's build a normal model that uses partial pooling for the country means and standard deviations. The model can be written as follows. Let $i$ be an index that specifies the country, and $j$ index the data points. 

$$X_{ij} \sim \text{N}(\mu_i, \sigma_i) \\
\mu_i \sim \text{N}(\mu_\mu, \sigma_\mu) \\
\sigma_i \sim \Gamma(\alpha_\sigma, \beta_\sigma) \\
\mu_\mu \sim \text{N}(0, 100)\\
\sigma_\mu \sim \Gamma(2, 0.1) \\
\alpha_\sigma, \beta_\sigma  \sim \Gamma(2, 0.01)$$

```stan
data {
  int<lower=0> G; // number of groups
  int<lower=0> N[G]; // sample size within each group
  vector[sum(N)] X; // concatenated observations
}

transformed data {
  // get first and last index for each group in X
  
  int start_i[G];
  int end_i[G];
  
  for(i in 1:G) {
    
    if(i == 1) {
      start_i[1] = 1;
    } else {
      start_i[i] = start_i[i-1] + N[i-1];
    }
    
    end_i[i] = start_i[i] + N[i]-1;
  }
}

parameters {
  
  // parameters
  vector[G] mu;
  vector<lower=0>[G] sigma;
  
  // hyperparameters
  real mu_mu;
  real<lower=0> sigma_mu;
  real<lower=0> alpha_sigma;
  real<lower=0> beta_sigma;
}



model {
  
  // Likelihood for each group
  for(i in 1:G) {
    X[start_i[i]:end_i[i]] ~ normal(mu[i], sigma[i]);
  }
  
  // Priors
  mu ~ normal(mu_mu, sigma_mu);
  sigma ~ gamma(alpha_sigma, beta_sigma);
  
  // Hyperpriors
  mu_mu ~ normal(0, 100);
  sigma_mu ~ inv_gamma(2, 0.1);
  alpha_sigma ~ gamma(2, 0.01);
  beta_sigma ~ gamma(2, 0.01);
}


generated quantities {
  
  real mu_tilda;
  real<lower=0> sigma_tilda;
  real X_tilda; 
  
  // Population distributions
  mu_tilda = normal_rng(mu_mu, sigma_mu);
  sigma_tilda = gamma_rng(alpha_sigma, beta_sigma);
  
  // PPD
  X_tilda = normal_rng(mu_tilda, sigma_tilda);
  
} 

```


Let's call Stan. 

To avoid convergence issues we'll use 10000 iterations and set `adapt_delta = 0.99`. Moreover, we'll parallelize the 4 chains. 


```r
stan_data <- list(G = length(unique(Height$Country)), 
                  N = rep(N, length(Countries)), 
                  X = Height$Height)

# Parallelize 4 chains
options(mc.cores = 4)

normal_hier_fit <- rstan::sampling(normal_hier_model,
                              stan_data, 
                              iter = 10000,
                              chains = 4,
                                    # Use to get rid of divergent transitions:
                                    control = list(adapt_delta = 0.99), 
                            # Print messages?
                            # refresh = 0
                            )
```


# Results

## Country-specific estimates

Let's first compare the marginal posteriors for the country-specific estimates: 


```r
par_summary <- rstan::summary(normal_hier_fit, c("mu", "sigma"))$summary %>% 
  data.frame() %>%
  rownames_to_column(var = "par") %>%
  separate(par, into = c("par", "country"), sep = "\\[") %>%
  mutate(country = gsub("\\]", "", country)) %>%
  mutate(country = Countries[as.integer(country)])
```



```r
# Plot
par_summary %>%
  ggplot() + 
  geom_point(aes(x = country, y = mean),
             color = posterior_color) +
  geom_errorbar(aes(x = country, ymin = X2.5., ymax = X97.5.),
                color = posterior_color) + 
  geom_point(data = df %>% 
               gather(key = "par",
                      value = "value",
                      -c(Country, Sex)), 
             aes(x = Country, y = value)) + 
  facet_wrap(~ par, scales = "free", ncol = 1) +
  coord_flip() +
  labs(title = "Blue = posterior; black = true value")
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-9-1.png" style="display: block; margin: auto;" />

## Hyperparameters

Let's then plot the population distribution's parameters, that is, the hyperparameters. The sample-based values are included in the plots of $\mu_\mu$ and $\sigma_\mu$ (why not for the other two hyperparameters?). 


```r
## Population distributions:
population_samples_l <- rstan::extract(normal_hier_fit, c("mu_mu", "sigma_mu", "alpha_sigma", "beta_sigma")) %>% 
  do.call(cbind, .) %>% 
  set_colnames(c("mu_mu", "sigma_mu", "alpha_sigma", "beta_sigma")) %>% 
  data.frame() %>% 
  mutate(sample = 1:nrow(.)) %>% 
  gather(key = "hyperpar", value = "value", -sample)


ggplot() +
  geom_histogram(data = population_samples_l, 
                 aes(x = value),
                 fill = posterior_color,
                 bins = 100) + 
  geom_vline(data = df_full_sub %>% 
               filter(Sex == "Girls") %>% 
               summarise(mu_mu = mean(mu), sigma_mu = sd(mu)) %>% 
               gather(key = "hyperpar", value = "value"),
             aes(xintercept = value)
             )+
  facet_wrap(~hyperpar, scales = "free")
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-10-1.png" style="display: block; margin: auto;" />

## Population distribution 
Let's then plot the population distributions and compare to the sample $\mu$'s and $\sigma$'s


```r
population_l <- rstan::extract(normal_hier_fit, c("mu_tilda", "sigma_tilda")) %>% 
  do.call(cbind, .) %>% 
  data.frame() %>% 
  set_colnames( c("mu", "sigma")) %>% 
  mutate(sample = 1:nrow(.)) %>% 
  gather(key = "par", value = "value", -sample)


 
ggplot() + 
  geom_histogram(data = population_l,
                 aes(x = value, y = ..density..),
                 bins = 100, fill = posterior_color) +
    geom_histogram(data = df_full_sub %>% 
                     gather(key = "par", value = "value", -c(Country, Sex)) %>% 
                     filter(Sex == "Girls"), 
                   aes(x = value, y = ..density..), 
                   alpha = 0.75, bins = 30) +
  facet_wrap(~par, scales = "free") + 
  labs(title = "Blue = posterior; black = sample")
```

```{.warning}
Warning: The dot-dot notation (`..density..`) was deprecated in ggplot2 3.4.0.
ℹ Please use `after_stat(density)` instead.
This warning is displayed once every 8 hours.
Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
generated.
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-11-1.png" style="display: block; margin: auto;" />


## Posterior predictive distribution

Let's then plot the posterior predictive distribution. Let's overlay it with  simulated data based on all countries.


```r
# Sample size per group 
N <- 25

# For each country, generate some random girl's heights
Height_all <- lapply(1:nrow(df_full_sub), function(i) {
  
  my_df <- df_full_sub[i, ]
  
  data.frame(Country = my_df$Country, 
             Sex = my_df$Sex, 
             # Random normal values based on measured mu and sd
             Height = rnorm(N, my_df$mu, my_df$sigma))

}) %>% 
  do.call(rbind, .)
```




```r
PPD <- rstan::extract(normal_hier_fit, c("X_tilda")) %>% 
  data.frame() %>% 
  set_colnames( c("X_tilda")) %>% 
  mutate(sample = 1:nrow(.))


ggplot() + 
  geom_histogram(data = PPD, 
                 aes(x = X_tilda, y = ..density..),
                 bins = 100,
                 fill = posterior_color) +
  geom_histogram(data = Height_all, 
                 aes(x = Height, y = ..density..), 
                 alpha = 0.75, 
                 bins = 100)
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-13-1.png" style="display: block; margin: auto;" />








# Extensions

We analyzed girl's heights in a few countries and modeled them hierarchically. You could make the structure richer in many ways, for instance by adding hierarchy between sexes, continents, or developed/developing countries etc. 


::::::::::::::::::::::::::::::::::::: keypoints 

- point 1

::::::::::::::::::::::::::::::::::::::::::::::::

## Reading
