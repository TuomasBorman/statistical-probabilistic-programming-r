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

- Learn how to construct  hierarchical models and fit them with Stan

::::::::::::::::::::::::::::::::::::::::::::::::



## Hierachical models

Bayesian hierarchical models are a class of models suited for scenarios where the study population consists of separate but related groups. For instance, analyzing student performance in different schools, income levels within different regions, or animal behavior within different populations are scenarios are scenarios where such models would be appropriate.  

A non-hierarchical model becomes hierarchical once the parameters of a prior are set as unknown and a prior is given to them. These hyperparameters and hyperpriors can be thought to exist on another level of hierarchy, hence the name. 

As an example, consider the beta-binomial model presented in Episode 2. It was used to estimate the prevalence of left-handedness based on a sample of 50 students If we had some additional information, such as the study majors, we could include this information in the model, like so

$$X_g \sim \text{Bin}(N_g, \theta_g) \\
\theta_g \sim \text{Beta}(\alpha, \beta) \\
\alpha, \beta \sim \Gamma(2, 0.1).$$

Here the subscript $g$ indices the groups for the majors. The group-specific prevalences for left-handedness $\theta_g$ are all given the Beta prior with hyperparameters $\alpha, \beta$ that are random variables. The final line denotes the hyperprior $\Gamma(2, 0.1)$ that controls the prior beliefs about the hyperparameters. 

Now, the students are modeled as identical within the groups, but no longer on the whole. On the other hand, there is some similarity between the groups, since they have a common prior, that is learned. 

These three different modeling approaches are also called unpooled, partially pooled (hierarchical), and completely pooled model. 

One key advantage of Bayesian hierarchical models is their ability to borrow strength across groups. By pooling information from multiple groups, these models can provide more stable estimates, especially when individual groups have limited data. 

Another difference to non-hierarchical models is that the prior, or the **population distribution** of the parameters is learned in the process. The population distribution can give insights about the parameter variation in larger context, that is, for groups we have no data on. For instance, if we had gathered data on the handedness of students majoring in natural sciences, the population distribution could give some insight about the students in humanities and social sciences. 


# Example: human height 

Let's analyze human adult height in different countries. We'll use the normal model with unknown mean $\mu$ and standard deviation $\sigma$ as the generative model, and give these parameters the hierarchical treatment. 

We'll simulate some data based on measured averages and standard deviations for boys and girls in different countries. The simulations are based on data in: Height: Height and body-mass index trajectories of school-aged children and adolescents from 1985 to 2019 in 200 countries and territories: a pooled analysis of 2181 population-based studies with 65 million participants. Lancet 2020, 396:1511-1524

We'll analyze these simulated data points and see how well we can recover the "true" parameters for each country. 

First, read the data and check its structure. 


```r
height <- read.csv("data/height_data.csv")

str(height)
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

Let's subset this data to simplify the analysis and focus on the height of adult women measured in 2019.


```r
height_women <- height %>% 
  filter(
    # 2019 measurements
    Year == 2019,        
    # use only a single age group
    Age.group == 19, 
    # Consider girls only
    Sex == "Girls"
    ) %>% 
  # Select variables of interest.
  select(Country, Sex, Mean.height, Mean.height.standard.error)
```

Let's select 10 countries randomly


```r
# Select countries
N_countries <- 10
Countries <- sample(unique(height_women$Country),
                    size = N_countries,
                    replace = FALSE) %>% sort

height_women10 <- height_women %>% filter(Country %in% Countries)

height_women10
```

```{.output}
    Country   Sex Mean.height Mean.height.standard.error
1   Bermuda Girls    166.1101                  4.3531671
2  Bulgaria Girls    164.5764                  3.8323766
3  Cambodia Girls    154.7495                  0.7493882
4  DR Congo Girls    156.3007                  0.8423344
5   Lao PDR Girls    153.0991                  0.9402629
6    Mexico Girls    157.9019                  0.5283058
7   Namibia Girls    160.2561                  0.8504509
8      Oman Girls    158.4350                  0.8546351
9   Romania Girls    164.7308                  0.7011577
10 Slovenia Girls    167.1976                  0.2571058
```


## Simulate data

Now, we can treat the values in the table above as ground truth and simulate some data based on them. Let's generate $N=25$ samples for each country from the normal model with $\mu = \text{Mean.height}$ and $\sigma = \text{Mean.height.standard.error}$.


```r
# Sample size per group 
N <- 25

# For each country, generate some random girl's heights
height_sim <- lapply(1:nrow(height_women10), function(i) {
  
  my_df <- height_women10[i, ]
  
  data.frame(Country = my_df$Country, 
             # Random normal values based on measured mu and sd
             Height = rnorm(N, my_df$Mean.height, my_df$Mean.height.standard.error))

}) %>% 
  do.call(rbind, .)


# Plot
height_sim %>% 
  ggplot() +
  geom_point(aes(x = Country, y = Height)) + 
  coord_flip() + 
  labs(title = "Simulated data")
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

Each point in the figure represents an individual. The data is simulated based on the measured mean and standard error in the respective countries. 

## Modeling

Let's build a normal model that uses partial pooling for the country means and standard deviations. The model can be written as follows. We'll use $g$ to index the country, and $i$ for the data points.

\begin{align}
X_{gi} &\sim \text{N}(\mu_g, \sigma_g) \\
\mu_g &\sim \text{N}(\mu_\mu, \sigma_\mu) \\
\sigma_g &\sim \Gamma(\alpha_\sigma, \beta_\sigma) \\
\mu_\mu &\sim \text{N}(0, 100)\\
\sigma_\mu &\sim \Gamma(2, 0.1) \\
\alpha_\sigma, \beta_\sigma  &\sim \Gamma(2, 0.01)
\end{align}


Here is the Stan program for the hierarchical normal model. The data points are input as a concatenated vector as this would allow using data with uneven sample sizes. The country-specific start and end indices are computed in the transformed data block. The parameters block contains the declarations of vectors for the means and standard deviations, along with the hyperparameters. The hyperparameter subscripts denote the parameter they are assigned to. The generated quantities block generates samples from the population distributions of $\mu$ and $\sigma$ and a posterior predictive distribution. 



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
  
  for(g in 1:G) {
    
    if(g == 1) {
      start_i[1] = 1;
    } else {
      start_i[g] = start_i[g-1] + N[g-1];
    }
    
    end_i[g] = start_i[g] + N[g]-1;
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
  
  // Posterior predictive distribution
  X_tilda = normal_rng(mu_tilda, sigma_tilda);
  
} 

```


Now we can call Stan and fit the model. Hierarchical models can often encounter convergence issues and for this reason, we'll use 10000 iterations and set `adapt_delta = 0.99`. Moreover, we'll run the 2 chains in parallel by setting `cores = 2`. 


```r
stan_data <- list(G = length(unique(height_sim$Country)), 
                  N = rep(N, length(Countries)), 
                  X = height_sim$Height)

normal_hier_fit <- rstan::sampling(normal_hier_model,
                              stan_data, 
                              iter = 10000,
                              chains = 2,
                                    # Use to get rid of divergent transitions:
                                    control = list(adapt_delta = 0.99), 
                              cores = 2,
                            # Print messages?
                            refresh = 5000
                            )
```


## Results

### Country-specific estimates

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
  geom_point(data = height_women10 %>% 
               rename_with(~ c('mu', 'sigma'), 3:4) %>% 
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
population_samples_l <- rstan::extract(normal_hier_fit,
                                       c("mu_mu", "sigma_mu", "alpha_sigma", "beta_sigma")) %>% 
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
  geom_vline(data = height_women %>% 
               rename_with(~ c('mu', 'sigma'), 3:4) %>%
               filter(Sex == "Girls") %>% 
               summarise(mu_mu = mean(mu), sigma_mu = sd(mu)) %>% 
               gather(key = "hyperpar", value = "value"),
             aes(xintercept = value)
             )+
  facet_wrap(~hyperpar, scales = "free")
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-10-1.png" style="display: block; margin: auto;" />

## Population distributions 

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
    geom_histogram(data = height_women %>%
                     rename_with(~ c('mu', 'sigma'), 3:4) %>%
                     gather(key = "par", value = "value", -c(Country, Sex)) %>% 
                     filter(Sex == "Girls"), 
                   aes(x = value, y = after_stat(density)), 
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

Finally, let's plot the posterior predictive distribution. Let's overlay it with the simulated data based on all countries.


```r
# Sample size per group 
N <- 25

# For each country, generate some random girl's heights
Height_all <- lapply(1:nrow(height_women), function(i) {
  
  my_df <- height_women[i, ] %>% 
    rename_with(~ c('mu', 'sigma'), 3:4)
  
  data.frame(Country = my_df$Country, 
             Sex = my_df$Sex, 
             # Random normal values based on sample mu and sd
             Height = rnorm(N, my_df$mu, my_df$sigma))

}) %>% 
  do.call(rbind, .)
```




```r
# Extract the posterior predictive distribution
PPD <- rstan::extract(normal_hier_fit, c("X_tilda")) %>% 
  data.frame() %>% 
  set_colnames( c("X_tilda")) %>% 
  mutate(sample = 1:nrow(.))


ggplot() + 
  geom_histogram(data = PPD, 
                 aes(x = X_tilda, y = after_stat(density)),
                 bins = 100,
                 fill = posterior_color) +
  geom_histogram(data = Height_all, 
                 aes(x = Height, y = after_stat(density)), 
                 alpha = 0.75, 
                 bins = 100)
```

<img src="fig/hierarchical-models-rendered-unnamed-chunk-13-1.png" style="display: block; margin: auto;" />








## Extensions

We analyzed women's heights in a few countries and modeled them hierarchically. You could make the structure richer in many ways, for instance by adding hierarchy between sexes, continents, developed/developing countries etc. 


::::::::::::::::::::::::::::::::::::: keypoints 

- Hierarchical models are appropriate for scenarios where the study population naturally divides into subgroups. 
- Hierarchical model borrow statistical strength across the groups. 
- Population distributions hold information about the variation of the model parameters in the whole population. 

::::::::::::::::::::::::::::::::::::::::::::::::

## Reading
