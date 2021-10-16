XOr-Some Bayesian Power analysis
================
Polina Tsvilodub
9/14/2021

For the main xor-some experiment, we conduct a simulation-based Bayesian
power analysis (used in a (loose) sense of determining the required
sample size here). This power analysis aims at determining how many
participants are required in order to detect a theoretically motivated
conjunctive effect of the predictors prior, relevance and competence
with a confidence of at least .8. The simulations are based on an
assumed effect size *β* =  ± 0.15 for all predictors, and a region of
practical equivalence (ROPE) *δ* = 0.05 for judging evidence for the
directionality of a coefficient given the data. The simulations are
further based on the maximal desired model justified theoretically and
by the design: `rating = trigger * prior * competence * relevance + REs`

In general, the power analysis proceeds as follows:

-   the desired effect sizes of the critical predictors are set to
     ∣ 0.15∣
-   hypothetical experimental data is simulated by drawing samples from
    a Normal distribution with the parameters described above
-   the desired maximal Bayesian regression model is computed on the
    simulated data
-   the N of subjects for which the data is simulated is increased
    iteratively in steps of 10 starting at 50
-   for each N, the model is re-computed on the simulated samples for
    k=100 iterations (the more, the better the power estimate, but there
    are computational constraints)
-   for each iteration *i*<sub>*k*</sub>, the conjunctive hypothesis of
    interest is tested
-   the power for the given number of participants is calculated as the
    proportion of iterations for which the conjunctive hypothesis was
    credible (i.e., if the posterior probability
    *P*(*β*<sub>*X*</sub> &gt; *δ* ∣ *D*) is at least .95, for
    *δ* = 0.05 the parameter that defines our ROPE for the conjunction
    of all three predictor coefficients, for all six sub-hypotheses).
-   we aim to determine the number of participants for which that
    proportion is at least .8.

## Simulate initial data

We assume an effect size of 0.15 for the predictors prior, competence
and relevance, and, therefore, simulate data as coming from a Normal
distribution with *μ* =  ± .15 and *σ* = .05 (because this *σ* would
give us app. 95% of the data in the region we defined as a positive /
negative effect).

The target response variable is simulated via 𝒩(*μ*<sub>*t*</sub>, 1),
with
*μ*<sub>*t*</sub> = *β*<sub>0</sub> + *β*<sub>pri</sub> + *β*<sub>comp</sub> + *β*<sub>rel</sub> + *β*<sub>trigger</sub> + interactions.
The coefficients for interactions and the trigger effect are assumed to
be 0.

From each participant, we get four data points for xor and four data
points for some, one data point per condition (prior X competence X
relevance, mapped to triggers at random). For each condition (and for
each trigger), there are four stories for which the random intercept may
vary.

Below, there is a toy example how data would be simulated for one
subject (ignoring random effects for now).

``` r
# ignore group-level effects for now; the effects will be sampled based on pilot data

# simulate toy data set for one subject 
d_init <-
  # create the 8 conditions
  tibble(prior = rep(c(0, 1), each = 4),
         comp = rep(c(0, 0, 1, 1), times = 2),
         rel = rep(c(0,1), times = 4),
         main_type = as.factor(sample(rep(c("xor", "some"), times=4)))) %>% 
  rowwise() %>%
  # sample effects based on condition
  mutate(
    subj_ID = 1, 
    intercept = 0,
    prior_obs = ifelse(prior == 0, rnorm(1, sd = 0.05), 
                       rnorm(1, -0.15, 0.05)),
    comp_obs = ifelse(comp == 0, rnorm(1, sd = 0.05), 
                       rnorm(1, 0.15, 0.05)),
    rel_obs = ifelse(rel == 0, rnorm(1, sd = 0.05), 
                       rnorm(1, 0.15, 0.05)),
    # sample response
    # what do we do about the interactions? can they safely be ignored?
    target = rnorm(1, 
                   mean = (intercept + prior_obs + comp_obs + rel_obs), 
                   sd = 1) # how to set / determine this properly?
                           # MF: irrelvant; we should z-score anyway exactly like we 
                           #     do for the real analysis 
                           #     this also means that choices of mean and SD for 
                           #     distributions to sample coefficents for are no that relevant
  )
d_init
```

    ## # A tibble: 8 × 10
    ## # Rowwise: 
    ##   prior  comp   rel main_type subj_ID intercept prior_obs comp_obs rel_obs
    ##   <dbl> <dbl> <dbl> <fct>       <dbl>     <dbl>     <dbl>    <dbl>   <dbl>
    ## 1     0     0     0 xor             1         0   -0.0205   0.0310  0.0954
    ## 2     0     0     1 some            1         0    0.0529  -0.0629  0.203 
    ## 3     0     1     0 xor             1         0   -0.0547   0.0834 -0.0116
    ## 4     0     1     1 xor             1         0    0.0287   0.0989  0.184 
    ## 5     1     0     0 some            1         0   -0.0863   0.0843 -0.0205
    ## 6     1     0     1 some            1         0   -0.215    0.0499  0.159 
    ## 7     1     1     0 xor             1         0   -0.0881   0.0897 -0.0566
    ## 8     1     1     1 some            1         0   -0.129    0.160   0.158 
    ## # … with 1 more variable: target <dbl>

## Compute initial model

Next, the seed model is computed on the initial simulated dataset. This
model will only be updated in further steps when the model is re-fit.

``` r
# fit model on simulated data for first time  
model_init <- brm(target ~ prior*comp*rel*main_type + 
                    (1 + prior + competence + relevance + main_type || submission_id) +
                   (1 | title),
                  data = d_init,
                  cores = 4,
                  contol = list(adapt_delta = 0.95),
                  iter = 3000)
model_init
```

## Build pipeline for simulating more data and fitting models

First, add random effects to the simulation based on random effects
estimated on pilot data.

``` r
# fit model on pilot data to get random effects estimates
pilot1 <- read_csv("./../data/pilots/pilot2_critical_zScore_wide_tidy.csv") %>%
  select(-competence_wUtt, -relevance_wUtt)
pilot2 <- read_csv("./../data/pilots/pilot3_critical_zScore_wide_tidy.csv")

d_both <- pilot1 %>% mutate(pilot = 1) %>% 
  rbind(., pilot2 %>% mutate(pilot = 2)
        )
# Full model 
d_both <- d_both %>% mutate(
  main_type = as.factor(main_type)
)
# some = 0, xor = 1
contrasts(d_both$main_type) 
# try to fit maximal model with brm
model_SI <- brm(target ~ prior*competence*relevance* main_type +
                   (1 + prior + competence + relevance + main_type || submission_id) +
                   (1 | title),
                 data = d_both,
                 control = list(adapt_delta = 0.95),
                 cores = 4,
                 iter = 3000)

# Get the by-story estimates
# check if this code can be less ugly
story_rand_int <- c(ranef(model_SI)[['title']][1:(length(ranef(model_SI)[['title']])/4)])

# get by-subj random intercepts and slopes
# TODO

# idea: sample from the vectors of observed estimates when simulating 
# (across predictors (slopes for pri vs rel vs comp for subjects) 
# and story types (comp X pri X rel condition of story))
#### --> Is this okay? 
```

Build helper function for simulating data for a given number of
subjects. \[stuff from here on is not finished yet\]

``` r
# TOFINISH
# N = number of subjects to simulate
get_new_data <- function(d, N) { # d probably unnecessary
    
      data <-
  # create the 8 conditions, manually, for double-checking
        tibble(prior = rep(c(0, 0, 0, 0, 1, 1, 1, 1), times = N),
         comp = rep(c(0, 0, 1, 1, 0, 0, 1, 1), times = N),
         rel = rep(c(0, 1, 0, 1, 0, 1, 0, 1), times = N),
         main_type = rep(sample(c("xor", "some", "xor", "some", "xor", "some", "xor", "some")), times=N),
         # assign subject ID
         subj_ID = rep(1:N, each = 8),
         # prepare parameters 
         intercept = 0,
         # sample random effects
         story_int = sample(story_rand_int, N*8, replace = T)
         # TODO: analogously sample by-subj effects
         ) %>% 
  rowwise() %>%
  # sample effects based on condition
  mutate(
    prior_obs = ifelse(prior == 0, rnorm(1, sd = 0.05), 
                       rnorm(1, -0.15, 0.05)),
    comp_obs = ifelse(comp == 0, rnorm(1, sd = 0.05), 
                       rnorm(1, 0.15, 0.05)),
    rel_obs = ifelse(rel == 0, rnorm(1, sd = 0.05), 
                       rnorm(1, 0.15, 0.05)),
    # sample response
    # what do we do about the interactions? can they safely be ignored?
    target = rnorm(1, 
                   mean = (intercept + story_int + prior_obs + comp_obs + rel_obs), 
                   sd = 1) # how to set / determine this properly?
  )
    
    
}
```

Build helper function for simulating and computing regression model for
k iterations (tracked as seeds). Also extract contrasts of interest
here.

``` r
# TODO:
# just copied over old stuff below

# simulate data and update
sim_data_fit <- function(seed, N) {
  set.seed(seed)
  
  # possibly add more participants
  data <- get_new_data(d, N)
  
  # mutate main_type to factor
  
  # update model fit with new data
  update(model_init,
         newdata = data,
         seed = seed) %>% 
    # extract posterior draws
    spread_draws(b_Intercept, b_syntax_dev1, b_trial_dev1, `b_syntax_dev1:trial_dev1`) %>%
    # extract contrasts of interest, especially effect of syntax by-trial 
  mutate(critical_subj = b_Intercept + b_syntax_dev1 - b_trial_dev1 - `b_syntax_dev1:trial_dev1`,
         critical_pred = b_Intercept - b_syntax_dev1 - b_trial_dev1 + `b_syntax_dev1:trial_dev1`,
         syntax_critical = critical_subj - critical_pred, # subject vs pred syntax
         filler_subj = b_Intercept + b_syntax_dev1 + b_trial_dev1 + `b_syntax_dev1:trial_dev1`,
         filler_pred = b_Intercept - b_syntax_dev1 + b_trial_dev1 - `b_syntax_dev1:trial_dev1`,
         syntax_filler = filler_subj - filler_pred) %>% # subject vs predicate syntax
  select(b_Intercept, b_syntax_dev1, b_trial_dev1, `b_syntax_dev1:trial_dev1`, critical_subj, critical_pred, syntax_critical, filler_subj, filler_pred, syntax_filler) %>%
  gather(key, val) %>%
  group_by(key) %>%
    # compute summary statistics 
  summarise( 
    mean = mean(val),
    lower = quantile(val, probs = 0.025),
    upper = quantile(val, probs = 0.975)
  )
##########  
  # use stuff below here
  spread_draws(b_Intercept, b_prior, b_competence, b_relevance,
                          b_main_typexor, `b_prior:competence`,
                          `b_prior:relevance`, `b_competence:relevance`,
                          `b_prior:main_typexor`, `b_competence:main_typexor`,
                          `b_relevance:main_typexor`, `b_prior:competence:relevance`, 
                          `b_prior:competence:main_typexor`, `b_prior:relevance:main_typexor`,
                          `b_competence:relevance:main_typexor`, 
                          `b_prior:competence:relevance:main_typexor`) %>% 
  mutate(
    prior_xor = b_prior + `b_prior:main_typexor`,
    prior_some = b_prior,
    competence_xor = b_competence + `b_competence:main_typexor`,
    competence_some = b_competence,
    relevance_xor =  b_relevance + `b_relevance:main_typexor`,
    relevance_some = b_relevance
  ) -> model_SI_posteriors

# check P that the effects are positive / negative / no effect present
posterior_hypotheses <- model_SI_posteriors %>% 
  select(prior_xor, prior_some, 
         competence_xor, competence_some,
         relevance_xor, relevance_some) %>%
  gather(key, val) %>%
  group_by(key) %>% mutate(positive = mean(val > 0.05),
                           negative = mean(val < -0.05),
                           no = mean(val %>% between(-0.05, 0.05))) %>%
  summarise(positive_eff = mean(positive),
            negative_eff = mean(negative),
            no_eff = mean(no))
}
```

## Test H of interest

Run the simulations and test the hypothesis of interest for each
iteration and compute the proportion of successful runs.

``` r
# TODO
test_conjunction_of_all_hypotheses <-  function(posterior_hypotheses) {
  posterior_hypotheses %>% 
    mutate(hypothesis_true = case_when(
      key == 'competence_some' ~ positive_eff > 0.95,
      key == 'competence_xor'  ~ positive_eff > 0.95,
      key == 'prior_some' ~ negative_eff > 0.95,
      key == 'prior_xor'  ~ negative_eff > 0.95, 
      key == 'relevance_some' ~ positive_eff > 0.95,
      key == 'relevance_xor'  ~ positive_eff > 0.95
    )) %>% 
    pull(hypothesis_true) %>% all()
}

# again old stuff below

# add iterating over Ns
sim1 <-
  tibble(seed = 1:100) %>% 
  mutate(tidy = map(seed, sim_data_fit, 23)) %>% 
  unnest(tidy)

# map each iter to test_conjunction_of_all_hypotheses
sim1 %>%
  filter(key == "syntax_critical") %>%
  mutate(check_syntax = ifelse(lower > 0, 1, 0)) %>%
  summarise(power_syntax = mean(check_syntax))
```

# MF’s crowbar ignoramus approach

``` r
# create fake data
create_fake_data <- function(N = 100, seed = 1) {
  set.seed(seed)
  beta_rel  <- 0.15
  beta_comp <- 0.15
  beta_pri  <- 0.15
  map_df(
    1:N, 
    function(i) {
      tibble(
        subj    = rep(str_c("subj_",i), times = 8),
        trigger = rep(c('or', 'some'), times= 4), 
        rel     = rnorm(8),
        comp    = rnorm(8),
        pri     = rnorm(8)
      ) %>% 
        group_by(subj) %>% 
        mutate(
          rel     = rel-mean(rel) / sd(rel),
          comp    = comp-mean(comp) / sd(comp),
          pri     = pri-mean(pri) / sd(pri)
        ) %>% 
        ungroup() %>% 
        mutate(
          target  = rnorm(8, beta_rel * rel + beta_comp * comp + beta_pri * pri)
        )
    }
  )
}
initial_fake_data <- create_fake_data(N=100, seed = 3)

# initial fit
fit_initial <- brm(target ~ trigger * rel * comp * pri, initial_fake_data)

# main results
N <- 300
k <-100

results <- map_dbl(
  1:k, 
  function(i) {
    fake_data <- create_fake_data(N=N, seed = i)
    fit <- update(fit_initial, newdata = fake_data, seed = i, cores = 4)
    lower_bounds <- rbind(
      aida::summarize_sample_vector(tidybayes::tidy_draws(fit) %>% select(b_rel), "rel"),
      aida::summarize_sample_vector(tidybayes::tidy_draws(fit) %>% select(b_comp), "comp"),
      aida::summarize_sample_vector(tidybayes::tidy_draws(fit) %>% select(b_pri), "pri")
    ) %>% pull("|95%")
    min(lower_bounds)
  }
)

message("Effective power for N=", N, " is:" , mean(results > 0.05))
```