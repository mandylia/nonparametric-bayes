# Comparison of Nonparametric Bayesian Gaussian Process estimates to standard the Parametric Bayesian approach

Plotting and knitr options, (can generally be ignored)






```r
require(modeest)
posterior.mode <- function(x) {
  mlv(x, method="shorth")$M
}
```





### Model and parameters

Uses the model derived in ` citet("10.1080/10236190412331335373")`, of a Ricker-like growth curve with an allee effect, defined in the pdgControl package,



```r
f <- RickerAllee
p <- c(2, 8, 5)
K <- 10  # approx, a li'l' less
allee <- 5 # approx, a li'l' less
```


Various parameters defining noise dynamics, grid, and policy costs.  


```r
sigma_g <- 0.05
sigma_m <- 0.0
z_g <- function() rlnorm(1, 0, sigma_g)
z_m <- function() 1+(2*runif(1, 0,  1)-1) * sigma_m
x_grid <- seq(0, 1.5 * K, length=50)
h_grid <- x_grid
profit <- function(x,h) pmin(x, h)
delta <- 0.01
OptTime <- 50  # stationarity with unstable models is tricky thing
reward <- 0
xT <- 0
Xo <-  allee+.5# observations start from
x0 <- K # simulation under policy starts from
Tobs <- 40
```


### Sample Data


```r
  set.seed(1234)
  #harvest <- sort(rep(seq(0, .5, length=7), 5))
  x <- numeric(Tobs)
  x[1] <- Xo
  nz <- 1
  for(t in 1:(Tobs-1))
    x[t+1] = z_g() * f(x[t], h=0, p=p)
  obs <- data.frame(x = c(rep(0,nz), 
                          pmax(rep(0,Tobs-1), x[1:(Tobs-1)])), 
                    y = c(rep(0,nz), 
                          x[2:Tobs]))
raw_plot <- ggplot(data.frame(time = 1:Tobs, x=x), aes(time,x)) + geom_line()
raw_plot
```

![plot of chunk obs](http://farm8.staticflickr.com/7407/8979192379_8862e88fe1_o.png) 



## Maximum Likelihood


```r
set.seed(12345)
estf <- function(p){ 
    mu <- f(obs$x,0,p)
    -sum(dlnorm(obs$y, log(mu), p[4]), log=TRUE)
}
par <- c(p[1]*rlnorm(1,0,.4), 
         p[2]*rlnorm(1,0,.3), 
         p[3]*rlnorm(1,0, .3), 
         sigma_g * rlnorm(1,0,.3))
o <- optim(par, estf, method="L", lower=c(1e-5,1e-5,1e-5,1e-5))
f_alt <- f
p_alt <- c(as.numeric(o$par[1]), as.numeric(o$par[2]), as.numeric(o$par[3]))
sigma_g_alt <- as.numeric(o$par[4])

est <- list(f = f_alt, p = p_alt, sigma_g = sigma_g_alt, mloglik=o$value)
```



Mean predictions


```r
true_means <- sapply(x_grid, f, 0, p)
est_means <- sapply(x_grid, est$f, 0, est$p)
```


## Non-parametric Bayes



```r
#inv gamma has mean b / (a - 1) (assuming a>1) and variance b ^ 2 / ((a - 2) * (a - 1) ^ 2) (assuming a>2)
s2.p <- c(5,5)  
d.p = c(10, 1/0.1)
```



Estimate the Gaussian Process (nonparametric Bayesian fit)


```r
gp <- gp_mcmc(obs$x, y=obs$y, n=1e5, s2.p = s2.p, d.p = d.p)
gp_dat <- gp_predict(gp, x_grid, burnin=1e4, thin=300)
```



Show traces and posteriors against priors


```r
plots <- summary_gp_mcmc(gp, burnin=1e4, thin=300)
```

![plot of chunk gp_traces_densities](http://farm6.staticflickr.com/5452/8979192641_6cec3355b6_o.png)
![plot of chunk gp_traces_densities](http://farm8.staticflickr.com/7459/8980386876_605218075e_o.png) 




```r
# Summarize the GP model
tgp_dat <- 
    data.frame(  x = x_grid, 
                 y = gp_dat$E_Ef, 
                 ymin = gp_dat$E_Ef - 2 * sqrt(gp_dat$E_Vf), 
                 ymax = gp_dat$E_Ef + 2 * sqrt(gp_dat$E_Vf) )
```



Parametric Bayesian Models
---------------------------

We use the JAGS Gibbs sampler, a recent open source BUGS
implementation with an R interface that works on most platforms.
We initialize the usual MCMC parameters; see `?jags` for details.

All parametric Bayesian estimates use the following basic parameters for the JAGS MCMC:



```r
y <- x 
N <- length(x);
jags.data <- list("N","y")
n.chains <- 6
n.iter <- 1e6
n.burnin <- floor(10000)
n.thin <- max(1, floor(n.chains * (n.iter - n.burnin)/1000))
n.update <- 10
```



We will use the same priors for process and observation noise in each model, 


```r
stdQ_prior_p <- c(1e-6, 100)
stdR_prior_p <- c(1e-6, .1)
stdQ_prior  <- function(x) dunif(x, stdQ_prior_p[1], stdQ_prior_p[2])
stdR_prior  <- function(x) dunif(x, stdR_prior_p[1], stdR_prior_p[2])
```


### Parametric Bayes of correct (Allen) model

We initiate the MCMC chain (`init_p`) using the true values of the
parameters `p` from the simulation.  While impossible in real data, this
gives the parametric Bayesian approach the best chance at succeeding.
`y` is the timeseries (recall `obs` has the $x_t$, $x_{t+1}$ pairs)


The actual model is defined in a `model.file` that contains an R function
that is automatically translated into BUGS code by *R2WinBUGS*.  The file
defines the priors and the model. We write the file from R as follows: 



```r
K_prior_p <- c(0.01, 40.0)
logr0_prior_p <- c(-3.0, 6.0)
logtheta_prior_p <- c(-6.0, 6.0)

bugs.model <- 
paste(sprintf(
"model{
  K     ~ dunif(%s, %s)
  logr0    ~ dunif(%s, %s)
  logtheta ~ dunif(%s, %s)
  stdQ ~ dunif(%s, %s)", 
  K_prior_p[1], K_prior_p[2],
  logr0_prior_p[1], logr0_prior_p[2],
  logtheta_prior_p[1], logtheta_prior_p[2],
  stdQ_prior_p[1], stdQ_prior_p[2]),

  "
  iQ <- 1 / (stdQ * stdQ);
  r0 <- exp(logr0)
  theta <- exp(logtheta)
  y[1] ~ dunif(0, 10)
  for(t in 1:(N-1)){
    mu[t] <- log(y[t]) + r0 * (1 - y[t]/K)* (y[t] - theta) / K 
    y[t+1] ~ dlnorm(mu[t], iQ) 
  }
}")
writeLines(bugs.model, "allen_process.bugs")
```


Write the priors into a list for later reference


```r
K_prior     <- function(x) dunif(x, K_prior_p[1], K_prior_p[2])
logr0_prior <- function(x) dunif(x, logr0_prior_p[1], logr0_prior_p[2])
logtheta_prior <- function(x) dunif(x, logtheta_prior_p[1], logtheta_prior_p[2])
par_priors  <- list(K = K_prior, deviance = function(x) 0 * x, 
                    logr0 = logr0_prior, logtheta = logtheta_prior,
                    stdQ = stdQ_prior)
```


We define which parameters to keep track of, and set the initial values of
parameters in the transformed space used by the MCMC.  We use logarithms
to maintain strictly positive values of parameters where appropriate.



```r
jags.params=c("K","logr0","logtheta","stdQ") # be sensible about the order here
jags.inits <- function(){
  list("K"= 8 * rlnorm(1,0, 0.1),
       "logr0"=log(2 * rlnorm(1,0, 0.1) ),
       "logtheta"=log(  5 * rlnorm(1,0, 0.1) ), 
       "stdQ"= abs( 0.1 * rlnorm(1,0, 0.1)),
       .RNG.name="base::Wichmann-Hill", .RNG.seed=123)
}

set.seed(1234)
# parallel refuses to take variables as arguments (e.g. n.iter = 1e5 works, but n.iter = n doesn't)
allen_jags <- do.call(jags.parallel, list(data=jags.data, inits=jags.inits, 
                                      jags.params, n.chains=n.chains, 
                                      n.iter=n.iter, n.thin=n.thin, 
                                      n.burnin=n.burnin, 
                                      model.file="allen_process.bugs"))

# Run again iteratively if we haven't met the Gelman-Rubin convergence criterion
recompile(allen_jags) # required for parallel
allen_jags <- do.call(autojags, list(object=allen_jags, n.update=n.update, 
                                     n.iter=n.iter, n.thin = n.thin))
```



#### Convergence diagnostics for Allen model

R notes: this strips classes from the `mcmc.list` object (so that we have list of matrices; objects that `reshape2::melt` can handle intelligently), and then combines chains into one array. In this array each parameter is given its value at each sample from the posterior (index) for each chain.  


```r
tmp <- lapply(as.mcmc(allen_jags), as.matrix) # strip classes the hard way...
allen_posteriors <- melt(tmp, id = colnames(tmp[[1]])) 
names(allen_posteriors) = c("index", "variable", "value", "chain")
ggplot(allen_posteriors) + geom_line(aes(index, value)) + 
  facet_wrap(~ variable, scale="free", ncol=1)
```

![plot of chunk allen-traces](http://farm6.staticflickr.com/5321/8979193249_a88f2311ea_o.png) 




```r
allen_priors <- ddply(allen_posteriors, "variable", function(dd){
    grid <- seq(min(dd$value), max(dd$value), length = 100) 
    data.frame(value = grid, density = par_priors[[dd$variable[1]]](grid))
})

ggplot(allen_posteriors, aes(value)) + 
  stat_density(geom="path", position="identity", alpha=0.7) +
  geom_line(data=allen_priors, aes(x=value, y=density), col="red") + 
  facet_wrap(~ variable, scale="free", ncol=3)
```

![plot of chunk allen-posteriors](http://farm9.staticflickr.com/8553/8979193499_c42e5dc822_o.png) 



Reshape the posterior parameter distribution data, transform back into original space, and calculate the mean parameters and mean function


```r
A <- allen_posteriors
A$index <- A$index + A$chain * max(A$index) # Combine samples across chains by renumbering index 
pardist <- acast(A, index ~ variable)
# pardist <- acast(allen_posteriors[2:3], 1:table(allen_posteriors$variable)[1] ~ variable) # NOT SURE WHY THIS FAILS 
# transform model parameters back first
pardist[,"logr0"] = exp(pardist[,"logr0"]) 
pardist[,"logtheta"] = exp(pardist[,"logtheta"])
colnames(pardist)[colnames(pardist)=="logtheta"] = "theta"
colnames(pardist)[colnames(pardist)=="logr0"] = "r0"
bayes_coef <- apply(pardist,2, posterior.mode) 
bayes_pars <- unname(c(bayes_coef["r0"], bayes_coef["K"], bayes_coef["theta"])) # parameters formatted for f
allen_f <- function(x,h,p) unname(RickerAllee(x,h, unname(p[c("r0", "K", "theta")])))
allen_means <- sapply(x_grid, f, 0, bayes_pars)
bayes_pars
```

```
[1] 0.13533 7.68891 0.05298
```

```r
head(pardist)
```

```
         K deviance      r0    theta    stdQ
170  7.458    33.73 0.29784 1.375401 0.05024
171  7.260    40.88 0.10384 0.005763 0.06147
172  8.062    35.60 0.33299 0.945562 0.05145
173  3.744    32.80 0.13850 7.776346 0.05265
174 10.100    41.82 0.05934 0.070421 0.05359
175  8.352    38.33 0.12159 0.066087 0.04874
```







## Parametric Bayes based on the structurally wrong model (Ricker)




```r
K_prior_p <- c(0.01, 40.0)
logr0_prior_p <- c(-6.0, 6.0)

bugs.model <- 
paste(sprintf(
"model{
  K    ~ dunif(%s, %s)
  logr0    ~ dunif(%s, %s)
  stdQ ~ dunif(%s, %s)", 
  K_prior_p[1], K_prior_p[2],
  logr0_prior_p[1], logr0_prior_p[2],
  stdQ_prior_p[1], stdQ_prior_p[2]),

  "
  iQ <- 1 / (stdQ * stdQ);
  r0 <- exp(logr0)
  y[1] ~ dunif(0, 10)
  for(t in 1:(N-1)){
    mu[t] <- log(y[t]) + r0 * (1 - y[t]/K) 
    y[t+1] ~ dlnorm(mu[t], iQ) 
  }
}")
writeLines(bugs.model, "ricker_process.bugs")
```



Compute prior curves


```r
K_prior     <- function(x) dunif(x, K_prior_p[1], K_prior_p[2])
logr0_prior <- function(x) dunif(x, logr0_prior_p[1], logr0_prior_p[2])
par_priors <- list(K = K_prior, deviance = function(x) 0 * x, 
                   logr0 = logr0_prior, stdQ = stdQ_prior)
```


We define which parameters to keep track of, and set the initial values of
parameters in the transformed space used by the MCMC.  We use logarithms
to maintain strictly positive values of parameters where appropriate.



```r
# Uniform priors on standard deviation terms
jags.params=c("K","logr0", "stdQ")
jags.inits <- function(){
  list("K"=10 * rlnorm(1,0,.5),
       "logr0"=log(1) * rlnorm(1,0,.5),
       "stdQ"=sqrt(0.05) * rlnorm(1,0,.5),
       .RNG.name="base::Wichmann-Hill", .RNG.seed=123)
}
set.seed(12345) 
ricker_jags <- do.call(jags.parallel, 
                       list(data=jags.data, inits=jags.inits, 
                            jags.params, n.chains=n.chains, 
                            n.iter=n.iter, n.thin=n.thin, n.burnin=n.burnin,
                            model.file="ricker_process.bugs"))
recompile(ricker_jags)
```

```
Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 251

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 251

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 251

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 251

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 251

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 251

Initializing model
```

```r
ricker_jags <- do.call(autojags, 
                       list(object=ricker_jags, n.update=n.update, n.iter=n.iter, 
                            n.thin = n.thin, progress.bar="none"))
```



#### Convergence diagnostics for parametric bayes Ricker model


```r
tmp <- lapply(as.mcmc(ricker_jags), as.matrix) # strip classes the hard way...
ricker_posteriors <- melt(tmp, id = colnames(tmp[[1]])) 
names(ricker_posteriors) = c("index", "variable", "value", "chain")

ggplot(ricker_posteriors) + geom_line(aes(index, value)) + 
  facet_wrap(~ variable, scale="free", ncol=1)
```

![plot of chunk ricker-traces](http://farm4.staticflickr.com/3794/8980387706_6747cf59d5_o.png) 



```r
ricker_priors <- ddply(ricker_posteriors, "variable", function(dd){
    grid <- seq(min(dd$value), max(dd$value), length = 100) 
    data.frame(value = grid, density = par_priors[[dd$variable[1]]](grid))
})
# plot posterior distributions
ggplot(ricker_posteriors, aes(value)) + 
  stat_density(geom="path", position="identity", alpha=0.7) +
  geom_line(data=ricker_priors, aes(x=value, y=density), col="red") + 
  facet_wrap(~ variable, scale="free", ncol=2)
```

![plot of chunk ricker-posteriors](http://farm3.staticflickr.com/2875/8979194039_857ac19403_o.png) 



Reshape posteriors data, transform back, calculate mode and corresponding function.  


```r
A <- ricker_posteriors
A$index <- A$index + A$chain * max(A$index) # Combine samples across chains by renumbering index 
ricker_pardist <- acast(A, index ~ variable)
ricker_pardist[,"logr0"] = exp(ricker_pardist[,"logr0"]) # transform model parameters back first
colnames(ricker_pardist)[colnames(ricker_pardist)=="logr0"] = "r0"
bayes_coef <- apply(ricker_pardist,2, posterior.mode) # much better estimates from mode then mean
ricker_bayes_pars <- unname(c(bayes_coef["r0"], bayes_coef["K"]))

ricker_f <- function(x,h,p){
  sapply(x, function(x){ 
    x <- pmax(0, x-h) 
    pmax(0, x * exp(p["r0"] * (1 - x / p["K"] )) )
  })
}
ricker_means <- sapply(x_grid, Ricker, 0, ricker_bayes_pars[c(1,2)])

head(ricker_pardist)
```

```
         K deviance       r0    stdQ
170  9.314    40.94 0.037042 0.05319
171 22.661    42.57 0.011806 0.05243
172 11.366    43.80 0.063853 0.05085
173 17.873    44.43 0.035388 0.05420
174 32.354    42.87 0.005756 0.05521
175 16.642    45.01 0.020490 0.06445
```

```r
ricker_bayes_pars
```

```
[1] 0.00836 8.28684
```



## Myers Parametric Bayes


```r
logr0_prior_p <- c(-6.0, 6.0)
logtheta_prior_p <- c(-6.0, 6.0)
logK_prior_p <- c(-6.0, 6.0)

bugs.model <- 
paste(sprintf(
"model{
  logr0    ~ dunif(%s, %s)
  logtheta    ~ dunif(%s, %s)
  logK    ~ dunif(%s, %s)
  stdQ ~ dunif(%s, %s)", 
  logr0_prior_p[1], logr0_prior_p[2],
  logtheta_prior_p[1], logtheta_prior_p[2],
  logK_prior_p[1], logK_prior_p[2],
  stdQ_prior_p[1], stdQ_prior_p[2]),

  "
  iQ <- 1 / (stdQ * stdQ);
  r0 <- exp(logr0)
  theta <- exp(logtheta)
  K <- exp(logK)

  y[1] ~ dunif(0, 10)
  for(t in 1:(N-1)){
    mu[t] <- log(r0)  + theta * abs(y[t]) - log(1 + pow(abs(y[t]), theta) / K)
    y[t+1] ~ dlnorm(mu[t], iQ) 
  }
}")
writeLines(bugs.model, "myers_process.bugs")
```




```r
logK_prior     <- function(x) dunif(x, logK_prior_p[1], logK_prior_p[2])
logr_prior     <- function(x) dunif(x, logr0_prior_p[1], logr0_prior_p[2])
logtheta_prior <- function(x) dunif(x, logtheta_prior_p[1], logtheta_prior_p[2])
par_priors <- list( deviance = function(x) 0 * x, logK = logK_prior,
                    logr0 = logr_prior, logtheta = logtheta_prior, 
                    stdQ = stdQ_prior)
```




```r
jags.params=c("logr0", "logtheta", "logK", "stdQ")
jags.inits <- function(){
  list("logr0"=log(rlnorm(1,0,.1)), 
       "logK"=log(10 * rlnorm(1,0,.1)),
       "logtheta" = log(2 * rlnorm(1,0,.1)),  
       "stdQ"=sqrt(0.5) * rlnorm(1,0,.1),
       .RNG.name="base::Wichmann-Hill", .RNG.seed=123)
}
set.seed(12345)
myers_jags <- do.call(jags.parallel, 
                      list(data=jags.data, inits=jags.inits, jags.params, 
                           n.chains=n.chains, n.iter=n.iter, n.thin=n.thin,
                           n.burnin=n.burnin, model.file="myers_process.bugs"))
recompile(myers_jags)
```

```
Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 370

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 370

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 370

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 370

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 370

Initializing model

Compiling model graph
   Resolving undeclared variables
   Allocating nodes
   Graph Size: 370

Initializing model
```

```r
myers_jags <- do.call(autojags, 
                      list(myers_jags, n.update=n.update, n.iter=n.iter, 
                           n.thin = n.thin, progress.bar="none"))
```


Convergence diagnostics for parametric bayes


```r
tmp <- lapply(as.mcmc(myers_jags), as.matrix) # strip classes the hard way...
myers_posteriors <- melt(tmp, id = colnames(tmp[[1]])) 
names(myers_posteriors) = c("index", "variable", "value", "chain")

ggplot(myers_posteriors) + geom_line(aes(index, value)) +
  facet_wrap(~ variable, scale="free", ncol=1)
```

![plot of chunk myers-traces](http://farm6.staticflickr.com/5466/8980388296_0c4bc86990_o.png) 





```r
par_prior_curves <- ddply(myers_posteriors, "variable", function(dd){
    grid <- seq(min(dd$value), max(dd$value), length = 100) 
    data.frame(value = grid, density = par_priors[[dd$variable[1]]](grid))
})

ggplot(myers_posteriors, aes(value)) + 
  stat_density(geom="path", position="identity", alpha=0.7) +
  geom_line(data=par_prior_curves, aes(x=value, y=density), col="red") + 
  facet_wrap(~ variable, scale="free", ncol=3)
```

![plot of chunk myers-posteriors](http://farm8.staticflickr.com/7368/8980388596_92e9431c35_o.png) 




```r
A <- myers_posteriors
A$index <- A$index + A$chain * max(A$index) # Combine samples across chains by renumbering index 
myers_pardist <- acast(A, index ~ variable)
### myers_pardist <- acast(myers_posteriors[2:3], 1:table(myers_posteriors$variable) ~ variable) 
myers_pardist[,"logK"] = exp(myers_pardist[,"logK"]) # transform model parameters back first
myers_pardist[,"logr0"] = exp(myers_pardist[,"logr0"]) # transform model parameters back first
myers_pardist[,"logtheta"] = exp(myers_pardist[,"logtheta"]) # transform model parameters back first
colnames(myers_pardist)[colnames(myers_pardist)=="logK"] = "K"
colnames(myers_pardist)[colnames(myers_pardist)=="logr0"] = "r0"
colnames(myers_pardist)[colnames(myers_pardist)=="logtheta"] = "theta"


bayes_coef <- apply(myers_pardist,2, posterior.mode) # much better estimates
myers_bayes_pars <- unname(c(bayes_coef[2], bayes_coef[3], bayes_coef[1]))
myers_means <- sapply(x_grid, Myer_harvest, 0, myers_bayes_pars)
myers_f <- function(x,h,p) Myer_harvest(x, h, p[c("r0", "theta", "K")])
head(myers_pardist)
```

```
    deviance        K    r0   theta    stdQ
170    44.97   0.8325 7.981 0.11231 0.04493
171    39.41   3.2984 4.337 0.11580 0.05258
172    40.55   7.4651 3.711 0.11318 0.04928
173    48.74 332.0344 2.826 0.12810 0.06829
174    39.99 202.2245 3.276 0.11083 0.04632
175    47.68  55.1607 3.829 0.09356 0.04547
```

```r
myers_bayes_pars
```

```
[1]  0.4414  3.4996 41.2481
```




### Phase-space diagram of the expected dynamics


```r
models <- data.frame(x=x_grid, 
										 GP=tgp_dat$y, 
										 True=true_means, 
                     MLE=est_means, 
										 Ricker=ricker_means, 
                     Allen = allen_means,
                     Myers = myers_means)
models <- melt(models, id="x")

# some labels
names(models) <- c("x", "method", "value")

# labels for the colorkey too
model_names = c("GP", "True", "MLE", "Ricker", "Allen", "Myers")
colorkey=cbPalette
names(colorkey) = model_names 
```




```r
plot_gp <- ggplot(tgp_dat) + geom_ribbon(aes(x,y,ymin=ymin,ymax=ymax), fill="gray80") +
    geom_line(data=models, aes(x, value, col=method), lwd=1, alpha=0.8) + 
    geom_point(data=obs, aes(x,y), alpha=0.8) + 
    xlab(expression(X[t])) + ylab(expression(X[t+1])) +
    scale_colour_manual(values=cbPalette) 
print(plot_gp)
```

![plot of chunk Figure1](http://farm4.staticflickr.com/3714/8980388856_800771282f_o.png) 



## Goodness of fit 

This shows only the mean predictions.  For the Bayesian cases, we can instead loop over the posteriors of the parameters (or samples from the GP posterior) to get the distribution of such curves in each case.  


```r
require(MASS)
step_ahead <- function(x, f, p){
  h = 0
  x_predict <- sapply(x, f, h, p)
  n <- length(x_predict) - 1
  y <- c(x[1], x_predict[1:n])
  y
}
step_ahead_posteriors <- function(x){
gp_f_at_obs <- gp_predict(gp, x, burnin=1e4, thin=300)
df_post <- melt(lapply(sample(100), 
  function(i){
    data.frame(time = 1:length(x), stock = x, 
                GP = mvrnorm(1, gp_f_at_obs$Ef_posterior[,i], gp_f_at_obs$Cf_posterior[[i]]),
                True = step_ahead(x,f,p),  
                MLE = step_ahead(x,f,est$p), 
                Allen = step_ahead(x, allen_f, pardist[i,]), 
                Ricker = step_ahead(x, ricker_f, ricker_pardist[i,]), 
                Myers = step_ahead(x, myers_f, myers_pardist[i,]))
  }), id=c("time", "stock"))
}

df_post <- step_ahead_posteriors(x)

ggplot(df_post) + geom_point(aes(time, stock)) + 
  geom_line(aes(time, value, col=variable, group=interaction(L1,variable)), alpha=.1) + 
  scale_colour_manual(values=colorkey, guide = guide_legend(override.aes = list(alpha = 1))) 
```

![plot of chunk Figureb](http://farm6.staticflickr.com/5345/8980389116_f84da2c41b_o.png) 




## Optimal policies by value iteration

Compute the optimal policy under each model using stochastic dynamic programming. We begin with the policy based on the GP model,



```r
MaxT = 1000
# uses expected values from GP, instead of integrating over posterior
#matrices_gp <- gp_transition_matrix(gp_dat$E_Ef, gp_dat$E_Vf, x_grid, h_grid)

# Integrate over posteriors 
matrices_gp <- gp_transition_matrix(gp_dat$Ef_posterior, gp_dat$Vf_posterior, x_grid, h_grid) 

# Solve the SDP using the GP-derived transition matrix
opt_gp <- value_iteration(matrices_gp, x_grid, h_grid, MaxT, xT, profit, delta, reward)
```



Determine the optimal policy based on the allen and MLE models


```r
matrices_true <- f_transition_matrix(f, p, x_grid, h_grid, sigma_g)
opt_true <- value_iteration(matrices_true, x_grid, h_grid, OptTime=MaxT, xT, profit, delta=delta)

matrices_estimated <- f_transition_matrix(est$f, est$p, x_grid, h_grid, est$sigma_g)
opt_estimated <- value_iteration(matrices_estimated, x_grid, h_grid, OptTime=MaxT, xT, profit, delta=delta)
```


Determine the optimal policy based on Bayesian Allen model


```r
matrices_allen <- parameter_uncertainty_SDP(allen_f, x_grid, h_grid, pardist, 4)
opt_allen <- value_iteration(matrices_allen, x_grid, h_grid, OptTime=MaxT, xT, profit, delta=delta)
```


Bayesian Ricker


```r
matrices_ricker <- parameter_uncertainty_SDP(ricker_f, x_grid, h_grid, as.matrix(ricker_pardist), 3)
opt_ricker <- value_iteration(matrices_ricker, x_grid, h_grid, OptTime=MaxT, xT, profit, delta=delta)
```



Bayesian Myers model


```r
matrices_myers <- parameter_uncertainty_SDP(myers_f, x_grid, h_grid, as.matrix(myers_pardist), 4)
myers_alt <- value_iteration(matrices_myers, x_grid, h_grid, OptTime=MaxT, xT, profit, delta=delta)
```



Assemble the data


```r
OPT = data.frame(GP = opt_gp$D, True = opt_true$D, MLE = opt_estimated$D, Ricker = opt_ricker$D, Allen = opt_allen$D, Myers = myers_alt$D)
colorkey=cbPalette
names(colorkey) = names(OPT) 
```




## Graph of the optimal policies


```r
policies <- melt(data.frame(stock=x_grid, sapply(OPT, function(x) x_grid[x])), id="stock")
names(policies) <- c("stock", "method", "value")

ggplot(policies, aes(stock, stock - value, color=method)) +
  geom_line(lwd=1.2, alpha=0.8) + xlab("stock size") + ylab("escapement")  +
  scale_colour_manual(values=colorkey)
```

![plot of chunk Figure2](http://farm3.staticflickr.com/2867/8980389346_39ac5232c4_o.png) 




## Simulate 100 realizations managed under each of the policies



```r
sims <- lapply(OPT, function(D){
  set.seed(1)
  lapply(1:100, function(i) 
    ForwardSimulate(f, p, x_grid, h_grid, x0, D, z_g, profit=profit, OptTime=OptTime)
  )
})

dat <- melt(sims, id=names(sims[[1]][[1]]))
dt <- data.table(dat)
setnames(dt, c("L1", "L2"), c("method", "reps")) 
# Legend in original ordering please, not alphabetical: 
dt$method = factor(dt$method, ordered=TRUE, levels=names(OPT))
```



```r
ggplot(dt) + 
  geom_line(aes(time, fishstock, group=interaction(reps,method), color=method), alpha=.1) +
  scale_colour_manual(values=colorkey, guide = guide_legend(override.aes = list(alpha = 1)))
```

![plot of chunk Figure3](http://farm4.staticflickr.com/3747/8980389788_3f5c38ac50_o.png) 




```r
Profit <- dt[, sum(profit), by=c("reps", "method")]
Profit[, mean(V1), by="method"]
```

```
   method     V1
1:     GP 24.908
2:   True 26.532
3:    MLE  4.420
4: Ricker 12.223
5:  Allen  6.429
6:  Myers  9.483
```



```r
ggplot(Profit, aes(V1)) + geom_histogram() + 
  facet_wrap(~method, scales = "free_y") + guides(legend.position = "none") + xlab("Total profit by replicate")
```

![plot of chunk totalprofits](http://farm6.staticflickr.com/5344/8979196259_4205b30818_o.png) 




```r
allen_deviance <- posterior.mode(pardist[,'deviance'])
ricker_deviance <- posterior.mode(ricker_pardist[,'deviance'])
myers_deviance <- posterior.mode(myers_pardist[,'deviance'])
true_deviance <- 2*estf(c(p, sigma_g))
mle_deviance <- 2*estf(c(est$p, est$sigma_g))

c(allen = allen_deviance, ricker=ricker_deviance, myers=myers_deviance, true=true_deviance, mle=mle_deviance)
```

```
  allen  ricker   myers    true     mle 
  34.55   43.35   41.25  -61.08 -287.60 
```
