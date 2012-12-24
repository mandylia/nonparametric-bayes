










```r
f <- RickerAllee
p <- c(r = 3, K = 10, C=4)
K <- p[2]
allee <- p[3]
```


We use the model of Allen 



```r
sigma_g <- 0.05
sigma_m <- 0.2
z_g <- function() rlnorm(1, 0, sigma_g) #1+(2*runif(1, 0,  1)-1)*sigma_g #
x_grid <- seq(0, 1.5 * K, length=101)
h_grid <- x_grid
profit = function(x,h) pmin(x, h)
delta <- 0.01
OptTime = 50 # should be long enough that profit is greater sustainably harvesting
reward = profit(x_grid[length(x_grid)], x_grid[length(x_grid)]) + 1 / (1 - delta) ^ OptTime 
xT <- 0
```


With parameters `3, 10, 4`. 


```r
x_0_observed <- K
xT <- 0
set.seed(123)
```



```r
nz <- 1 # weight on 0,0 pt
Tobs <- 40
harvest <- sort(rep(seq(1, 2, length=5), 8))
x <- numeric(Tobs)
x[1] <- x_0_observed
for(t in 1:(Tobs-1))
  x[t+1] = z_g() * f(x[t], h=2, p=p)
plot(x)
```

![plot of chunk sim-with-harvest](figure/sim-with-harvest1.png) 

```r


obs <- data.frame(x=c(rep(0,nz), pmax(rep(0,Tobs-1),x[1:(Tobs-1)]-harvest[1:Tobs-1])),y=c(rep(0,nz),x[2:Tobs]))
plot(obs$x, obs$y)
```

![plot of chunk sim-with-harvest](figure/sim-with-harvest2.png) 


We simulate data under this model, starting from a size of `10`.  


We consider the observations as ordered pairs of observations of current stock size $x_t$ and observed stock in the following year, $x_{t+1}$.  We add the pseudo-observation of $0,0$.  Alternatively we could condition strictly on solutions passing through the origin, though in practice the weaker assumption is often sufficient. 


```r
estf <- function(p){ 
  mu <- f(obs$x,0,p)
  -sum(dlnorm(obs$y, log(mu), p["s"]), log=TRUE)
}
par = c(r = p[1] - 1, 
        C = p[2] - 1, 
        K = p[3] + 2, 
        s = sigma_g + abs(rnorm(1,0,.1)))
o <- optim(par, estf, method="L", lower=c(1e-3,1e-3,1e-3, 1e-3))
f_alt <- f
p_alt <- c(as.numeric(o$par[1]), as.numeric(o$par[2]), as.numeric(o$par[3]))
sigma_g_alt <- as.numeric(o$par[4])
p_alt
```

```
[1]  2.865 10.863  6.279
```

```r
sigma_g_alt
```

```
[1] 0.001
```




```r
s2.p <- c(50,50) #inv gamma has mean b / (a - 1) (assuming a>1) and variance b ^ 2 / ((a - 2) * (a - 1) ^ 2) (assuming a>2)
tau2.p <- c(50,50)
d.p = c(10, 1/0.01, 10, 1/0.01)  ## sum of gammas, shape a & rate b. gamma has mean a / b and variance a / b ^ 2
nug.p = c(10, 1/0.01, 10, 1/0.01)
s2_prior <- function(x) dinvgamma(x, s2.p[1], s2.p[2])
tau2_prior <- function(x) dinvgamma(x, tau2.p[1], tau2.p[2])
d_prior <- function(x) dgamma(x, d.p[1], scale = d.p[2]) + dgamma(x, d.p[3], scale = d.p[4])
nug_prior <- function(x) dgamma(x, nug.p[1], scale = nug.p[2]) + dgamma(x, nug.p[3], scale = nug.p[4])
beta0_prior <- function(x, tau = tau2.p[2] / (tau2.p[1] - 1)) dnorm(x, 0, tau)
beta = c(0)
```




```r
gp <- bgp(X=obs$x, XX=x_grid, Z=obs$y, verb=0,
          meanfn="constant", bprior="b0", BTE=c(2000,16000,2),
          m0r1=FALSE, corr="exp", trace=TRUE, 
          beta = beta, s2.p = s2.p, d.p = d.p, nug.p = nug.p, tau2.p = tau2.p,
          s2.lam = "fixed", d.lam = "fixed", nug.lam = "fixed", tau2.lam = "fixed")
```


We fit a Gaussian process with 


```r
V <- gp$ZZ.ks2
Ef = gp$ZZ.km
tgp_dat <- data.frame(x   = gp$XX[[1]], 
                  y   = gp$ZZ.km, 
                 ymin = gp$ZZ.km - 1.96 * sqrt(gp$ZZ.ks2), 
                 ymax = gp$ZZ.km + 1.96 * sqrt(gp$ZZ.ks2),
                 ymin2 = gp$ZZ.mean - 1.96 * sqrt(gp$ZZ.vark), 
                 ymax2 = gp$ZZ.mean + 1.96 * sqrt(gp$ZZ.vark) )

# ZZ.ks2, ZZ.vark, ZZ.s2
```



```r
true <- sapply(x_grid, f, 0, p)
est <- sapply(x_grid, f_alt, 0, p_alt)
models <- data.frame(x=x_grid, GP=tgp_dat$y, Parametric=est, True=true)
models <- melt(models, id="x")
names(models) <- c("x", "method", "value")
# plot
ggplot(tgp_dat)  + 
  geom_ribbon(aes(x,y,ymin=ymin,ymax=ymax), fill="gray80") +
  geom_ribbon(aes(x,y,ymin=ymin2,ymax=ymax2), fill="gray60") +
  geom_line(data=models, aes(x, value, col=method), lwd=2, alpha=0.8) + 
  geom_point(data=obs, aes(x,y), alpha=0.8) + 
  xlab(expression(X[t])) + ylab(expression(X[t+1])) +
  scale_colour_manual(values=cbPalette)
```

![plot of chunk gp-plot](figure/gp-plot.png) 





```r
hyperparameters <- c("index", "s2", "tau2", "beta0", "nug", "d", "ldetK")
posteriors <- melt(gp$trace$XX[[1]][,hyperparameters], id="index")
priors <- list(s2 = s2_prior, tau2 = tau2_prior, beta0 = beta0_prior, nug = nug_prior, d = d_prior, ldetK = function(x) 0)
prior_curves <- ddply(posteriors, "variable", function(dd){
  grid <- seq(min(dd$value), max(dd$value), length = 100)
  data.frame(value = grid, density = priors[[dd$variable[1]]](grid))
})
ggplot(posteriors) + 
  #geom_density(aes(value), lwd=2) +
  geom_histogram(aes(x=value, y=..density..), lwd=2) +
  geom_line(data=prior_curves, aes(x=value, y=density), col="red", lwd=2) +
  facet_wrap(~ variable, scale="free")
```

```
stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust
this.
```

```
stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust
this.
```

```
stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust
this.
```

```
stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust
this.
```

```
stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust
this.
```

```
stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust
this.
```

![plot of chunk gp-posteriors](figure/gp-posteriors1.png) 

```r
ggplot(prior_curves) + 
  geom_line(aes(x=value, y=density), col="red", lwd=2) +
  facet_wrap(~ variable, scale="free")
```

![plot of chunk gp-posteriors](figure/gp-posteriors2.png) 

```r
#ggplot(subset(posteriors, variable=="nug")) + geom_histogram(aes(x=value, y = ..density..), lwd=2) + stat_function(fun = nug_prior, col="red", lwd=2)
#ggplot(subset(posteriors, variable=="s2")) + geom_histogram(aes(x=value, y = ..density..), lwd=2) + stat_function(fun = s2_prior, col="red", lwd=2)
```



## The transition matrix of the inferred process


```r
X <- numeric(length(x_grid))
X[38] = 1
h <- 0
```



```r
F_ <- gp_F(h, Ef, V, x_grid)
xt1 <- X %*% F_
xt10 <- xt1
for(s in 1:OptTime)
  xt10 <- xt10 %*% F_
qplot(x_grid, xt10[1,]) + geom_point(aes(y=xt1[1,]), col="grey")
```

![plot of chunk gp-F-sim](figure/gp-F-sim.png) 



```r
F_true <- par_F(h, f, p, x_grid, sigma_g)
yt1 <- X %*% F_true
yt10 <- yt1
for(s in 1:OptTime)
  yt10 <- yt10 %*% F_true
qplot(x_grid, yt10[1,]) + geom_point(aes(y=yt1[1,]), col="grey")
```

![plot of chunk par-F-sim](figure/par-F-sim.png) 



```r
transition <- melt(data.frame(x = x_grid, gp = xt1[1,], parametric = yt1[1,]), id="x")
ggplot(transition) + geom_point(aes(x,value, col=variable))
```

![plot of chunk F-sim-plot](figure/F-sim-plot.png) 





```r
matrices_gp <- gp_transition_matrix(Ef, .01*V, x_grid, h_grid)
opt_gp <- find_dp_optim(matrices_gp, x_grid, h_grid, OptTime, xT, profit, delta, reward=reward)
```



```r
matrices_true <- f_transition_matrix(f, p, x_grid, h_grid, sigma_g)
opt_true <- find_dp_optim(matrices_true, x_grid, h_grid, OptTime, xT, profit, delta=delta, reward = reward)
```



```r
matrices_estimated <- f_transition_matrix(f_alt, p_alt, x_grid, h_grid, sigma_g_alt)
opt_estimated <- find_dp_optim(matrices_estimated, x_grid, h_grid, OptTime, xT, profit, delta=delta, reward = reward)
```




```r
policies <- melt(data.frame(stock=x_grid, 
                            GP = x_grid[opt_gp$D[,1]], 
                            Parametric = x_grid[opt_estimated$D[,1]],
                            True = x_grid[opt_true$D[,1]]),
                  id="stock")
names(policies) <- c("stock", "method", "value")
policy_plot <- ggplot(policies, aes(stock, stock - value, color=method)) +
  geom_line(lwd=2, alpha=0.8) + 
  xlab("stock size") + ylab("escapement")  +
  scale_colour_manual(values=cbPalette)
policy_plot
```

![plot of chunk policy_plot](figure/policy_plot.png) 




```r
z_g = function() rlnorm(1, 0, sigma_g)
z_m = function() 1+(2*runif(1, 0,  1)-1) * sigma_m
```



```r
m <- sapply(1:OptTime, function(i) opt_gp$D[,1])
opt_gp$D <- m
mm <- sapply(1:OptTime, function(i) opt_true$D[,1])
opt_true$D <- mm
mmm <- sapply(1:OptTime, function(i) opt_estimated$D[,1])
opt_estimated$D <- mmm
```





```r
set.seed(1)
sim_gp <- lapply(1:100, function(i) ForwardSimulate(f, p, x_grid, h_grid, K, opt_gp$D, z_g, profit=profit))
set.seed(1)
sim_true <- lapply(1:100, function(i) ForwardSimulate(f, p, x_grid, h_grid, K, opt_true$D, z_g, profit=profit))
set.seed(1)
sim_est <- lapply(1:100, function(i) ForwardSimulate(f, p, x_grid, h_grid, K, opt_estimated$D, z_g, profit=profit))
```




```r
dat <- list(GP = sim_gp, Parametric = sim_est, True = sim_true)
dat <- melt(dat, id=names(dat[[1]][[1]]))
dt <- data.table(dat)
setnames(dt, c("L1", "L2"), c("method", "reps")) 
```




```r
ggplot(dt) + 
  geom_line(aes(time, fishstock, group=interaction(reps,method), color=method), alpha=.1) +
  scale_colour_manual(values=cbPalette, guide = guide_legend(override.aes = list(alpha = 1)))
```

![plot of chunk sim-fish](figure/sim-fish.png) 




```r
ggplot(dt) +
  geom_line(aes(time, harvest, group=interaction(reps,method), color=method), alpha=.1) +
  scale_colour_manual(values=cbPalette, guide = guide_legend(override.aes = list(alpha = 1)))
```

![plot of chunk sim-harvest](figure/sim-harvest.png) 



```r
profits <- dt[, sum(profit), by = c("reps", "method")]
means <- profits[, mean(V1), by = method]
sds <- profits[, sd(V1), by = method]
yield <- cbind(means, sd = sds$V1)
yield
```

```
       method     V1    sd
1:         GP  70.72 2.604
2: Parametric  69.70 4.180
3:       True 111.07 3.560
```


<p>Myers RA, Barrowman NJ, Hutchings JA and Rosenberg AA (1995).
&ldquo;Population Dynamics of Exploited Fish Stocks at Low Population Levels.&rdquo;
<EM>Science</EM>, <B>269</B>.
ISSN 0036-8075, <a href="http://dx.doi.org/10.1126/science.269.5227.1106">http://dx.doi.org/10.1126/science.269.5227.1106</a>.
