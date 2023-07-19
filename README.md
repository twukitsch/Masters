Running and Graphing glmers
================
TJ Wukitsch
July 2023

# BACKGROUND

These data are from [Wukitsch & Cain, 2021](10.1007/s00213-021-05805-y)
and are used here as an example of how to use `ggplot2` to graph
Generalized Linear Mixed Effects Models using R. In this case, we have
count data so we will use a Poisson model. The `ggplot2` library along
with a few others, were used to produce the images in the aforementioned
publication. Below is a comprehensive guide on how you can make ones
like it for your glmer models too!

## Understanding the Data

![](README_files/Experimental%20Structure.png)

These data originate from a portion of the study that had 2 independent
(output) variables: Total Aversive Responses & Total Hedonic Responses.
These variables are counts of the number of behaviors emitted in
response to an orally infused fluid and are the behavioral results of
the Taste Reactivity Test. Prior to undergoing Taste Reactivity testing,
adult and adolescent Long-Evans rats were allowed Intermittent
(voluntary) Access to drink Ethanol (IAE) Monday, Wednesday, and Friday.
The amount and pattern of drinking over time during IAE were dependent
(predictor) variables in some of our models in addition to their Age
group. Control animals received only water during IAE and were not
exposed to ethanol until Taste Reactivity testing. During Taste
Reactivity testing, 3 substances were orally infused: Water on the first
and last day of Taste Reactivity testing, Ethanol (at 5, 20, and 40% v/v
concentrations), and Sucrose (at .01, .1, and 1 molar concentrations).
We will focus on the Ethanol data here and want to treat all variables
as they are… No categorizing our numeric continua into factors. This
yields the following set of variables for our models:

**Independent (Outcome) Variables:**

- Total Aversive Responses (count data)
- Total Hedonic Responses (count data)

**Dependent (Predictor) Variables:**

- **Age** (during IAE period)
  - Adult
  - Adolescent
- Intermittent Ethanol Access (IAE) **Condition**
  - IAE (Had access to ethanol during IAE period)
    - **Ethanol Consumption** during IAE period *(A NESTED VARIABLE)*
  - CTRL (Did not have access to ethanol prior to Taste Reactivity
    Testing)
- **Concentration** of Ethanol
  - 5%
  - 20%
  - 40%

## Modeling Approach

This nested data structure leaves us with some options. We can either 1.
use a nested model or 2. perform two separate analyses of the same data
that answer different questions. It all depends on your audience. We
decided it was better for our audience to keep it straightforward and
use the two analysis approach. The first model will answer questions
about overall differences between our IAE and our CTRL groups, Age, and
Concentration. The second will be able to answer questions about
individual differences in ethanol consumption during IAE and how those
may interact with Age and Concentration to alter taste reactivity. Thus,
the formulas for our `glmer` models are as follows:

**Overall Models**

- **Independent (AKA Predicted AKA Outcome) Variables**
  - Total Aversive Responses
  - Total Hedonic Responses
- **Fixed Effect (AKA Main Effect) Variables (Full Factorial)**
  - Concentration
  - Age
  - Condition
- **Random Effect Variables**
  - Intercept (RatID)
  - (Slope of) Concentration

This yields the following conceptual formulas for the Overall models:

- $Aversive~Responses = exp(Age * Condition * Concentration + (Concentration | RatID))$
- $Hedonic~Responses = exp(Age * Condition * Concentration + (Concentration | RatID))$

**Ethanol Consumption Models**

We haven’t yet determined which measure of ethanol consumption to use
for the Ethanol Consumption Models. We could use the sum total of all
ethanol consumed during IAE (Total Ethanol), or we could use the
intercept (Mean Alcohol Consumed; MAC) and slope of a line that best
fits each individual’s drinking during IAE across time (Rate of Change;
RoC). We used `lmers` to calculate these values, but that is beyond the
scope of the present example. A quirk of adolescent drinking during IAE
was also of interest. Adolescents had very high drinking on the first
day. This created some non-monotonicity (non-linearity) in our data, so,
we wondered if removing the first day would impact the RoC
substantially. Thus, we had an additional model that did not include the
first day of drinking when calculating RoC and MAC (RoC3 and MAC3,
respectively). Our plan was to compare the models predicting responses
to orally infused ethanol and use the model with the best (lowest)
AIC/BIC scores for interpretation.

- **Independent (AKA Predicted AKA Outcome) Variables**
  - Total Aversive Responses
  - Total Hedonic Responses
- **Fixed Effect (AKA Main Effect) Variables (Full Factorial where
  appropriate)**
  - Concentration
  - Age
  - Ethanol Consumption
    - Total Ethanol Consumed (grams ethanol/kg bodyweight; during IAE)
    - MAC & RoC (Consumption Pattern during IAE)
    - MAC3 & RoC3 (Consumption Pattern starting at day 3 \[the second
      day of drinking\])
- **Random Effect Variables**
  - Intercept (RatID)
  - (Slope of) Concentration

This yields the following conceptual formulas for the Ethanol
Consumption models:

- Aversive Responses
  - $Aversive~Responses = exp(Concentration * Age * TotalEthanolConsumed + (Concentration | RatID))$
  - $Aversive~Responses = exp(Concentration * Age * [MAC + RoC] + (Concentration | RatID))$
  - $Aversive~Responses = exp(Concentration * Age * [MAC3 + RoC3] + (Concentration | RatID))$
- Hedonic Responses
  - $Hedonic~Responses = exp(Concentration * Age * TotalEthanolConsumed + (Concentration | RatID))$
  - $Hedonic~Responses = exp(Concentration * Age * [MAC + RoC] + (Concentration | RatID))$
  - $Hedonic~Responses = exp(Concentration * Age * [MAC3 + RoC3] + (Concentration | RatID))$

*Note: the brackets ‘\[\]’ are intended to indicate that MAC and RoC
(and MAC3 and RoC3) do not interact with each other in the models above.
This will become clearer when we specify the formula in long-form within
the coding for the model in a later step.*

# Coding Approach Goals

Now that we understand our data and our modeling approach, we need to
consider our order of operations and goals for our data. We want an
organized workspace that we can come back to and modify if needs change
or if we have additional questions. We want to organize our models and
any additional analyses into smaller sets of code, so we can more easily
navigate and troubleshoot. And, finally, we want to have some pretty
graphs that get our point across and clearly represent the data well.
This gives us 4 goals…

**GOALS:**

1.  Organize and Setup Data Workspace & Structures
2.  Model Data (Generalized Linear Mixed Effects Models) & Organize
    Output Objects
3.  Perform Comparisons, Obtain Trend Info., & Organize Output objects
4.  Create Publication-Quality Nested Multi-panel Graphs with `ggplot2`
    & Organize Output objects

# 1.1 Workspace Setup

## Install and Load Required Packages

If you haven’t already downloaded and installed Rtools, please install
[`Rtools`](https://cran.r-project.org/bin/windows/Rtools/) and restart
R. The C++ compiler is required for some of the packages that you are
about to install.

The following code will check and remind you if you haven’t yet
installed Rtools. Then it will check to see if the package is already
downloaded and installed. If so, it will load the package. If not, it
will install the package from CRAN and then load the package once the
installation is complete. You may be prompted to allow some of the
downloads to take place so please remain at your device. I recommend
making a separate R script like this for every project. As you add
packages that you need, be sure to add them to your setup script and
save time and guess work.

``` r
# Load Packages ####

# See if rtools is installed. It has necessary C++ compiler for installs of some of the libraries that are required.
if (Sys.which("make") == "") {  # Check to see if "make" command from rtools is found in system's PATH. If an empty string is retuned, rtools isn't installed.
  message("Rtools is not found. Please install Rtools from https://cran.r-project.org/bin/windows/Rtools/ and restart R.")
} else { #rtools installed == TRUE
  message("Rtools is installed.")
  
  # Load package or install MASS and load if not present
  if (!require(MASS)) {
    install.packages("MASS")
    library(MASS)
  }
  
  # Load package or install lattice and load if not present
  if (!require(lattice)) {
    install.packages("lattice")
    library(lattice)
  }
  
  # Load package or install boot and load if not present
  if (!require(boot)) {
    install.packages("boot")
    library(boot)
  }
  
  # Load package or install car and load if not present
  if (!require(car)) {
    install.packages("car")
    library(car)
  }
  
  # Load package or install emmeans and load if not present
  if (!require(emmeans)) {
    install.packages("emmeans")
    library(emmeans)
  }
  
  
  # Load package or install lme4 and load if not present
  if (!require(lme4)) {
    install.packages("lme4")
    library(lme4)
  }
  
  # Load package or install zoo and load if not present
  if (!require(zoo)) {
    install.packages("zoo")
    library(zoo)
  }
  
  # Load package or install tidyr and load if not present
  if (!require(tidyr)) {
    install.packages("tidyr")
    library(tidyr)
  }
  
  # Load package or install ggplot2 and load if not present
  if (!require(ggplot2)) {
    install.packages("ggplot2")
    library(ggplot2)
  }
  
  # Load package or install lmerTest and load if not present
  if (!require(lmerTest)) {
    install.packages("lmerTest")
    library(lmerTest)
  }
  
  # Load package or install dplyr and load if not present
  if (!require(dplyr)) {
    install.packages("dplyr")
    library(dplyr)
  }
  
  # Load package or install multcomp and load if not present
  if (!require(multcomp)) {
    install.packages("multcomp")
    library(multcomp)
  }
  
  # Load package or install foreign and load if not present
  if (!require(foreign)) {
    install.packages("foreign")
    library(foreign)
  }
  
  # Load package or install msm and load if not present
  if (!require(msm)) {
    install.packages("msm")
    library(msm)
  }
  
  # Load package or install effects and load if not present
  if (!require(effects)) {
    install.packages("effects")
    library(effects)
  }
  
    # Load package or install effects and load if not present
  if (!require(gridExtra)) {
    install.packages("gridExtra")
    library(gridExtra)
  }

  # Load package or install patchwork and load if not present
  if (!require(patchwork)) {
    install.packages("patchwork")
    library(patchwork)
  }

} # end rtools installed == TRUE
```

## Set your working directory

Using a working directory allows for relative referencing as opposed to
absolute (aka static) referencing. Instead of changing every single
absolute reference to a folder on one computer throughout all your code,
you just need to change the working directory at the beginning and
load/save from there. For example, say I want to reference the
Experimental Structure.png from above. Instead of an absolute reference
path like “C:/Users/name/Documents/ABHV/README_files/Experimental
Structure.png”, a relative reference path would be
“README_files/Experimental Structure.png” Because our working directory
is already set to the ABHV folder that contains the README_files folder,
we dont need the extra fluff. This results in code that not only saves
time because it is shorter, but is much more flexible for a variety of
situations like using the code on another computer. Always try to use
relative referencing once you have changed your working directory to
make your life (and anyone’s life that you share you code with) much
easier.

``` r
# Set your working directory to your project folder and check that it is correct. 
setwd("C:/Users/[yourname]/Documents/ABHV")
getwd()
```

# 1.2 Workspace Overview

To keep an organized and uncluttered workspace, I use sets of
hierarchical lists to store related objects. We have the following
object categories with most analyses:

- data
- models
- comparisons (abbv. “compars”)
- plots

In this specific set of analyses we have a number of needs that
determine the objects in each category.

## Data Storage

The full data requires various sub-setting to ensure the correct
substances are being assessed independently. For example, we want to
make sure that controls that never received access to ethanol are not
included in an analysis or mean calculation when it would be
inappropriate to do so. Thus, my data list object will look like this
(with abbreviated names for easier calling in the code itself):

- Data
  - Raw
  - Ethanol
    - With Controls
    - No Controls
  - Sucrose
    - With Controls
    - No Controls
  - water
    - With Controls
    - No Controls

To do this we generate a list called `data` and begin populating it with
subsets of the original data. In the process, it will be easier to make
some changes now because subsets of data inherit certain attributes from
their parent data sets. Therefore, during this step, we will also adjust
the contrast coding and scale of some of our variables to help us avoid
problems later. We adjust our categorical (factor) variables’ contrast
coding attribute because we want our `glmer` models’ categorical
variables to be coded like an ANOVA to ease interpretation of the
effects. In other words, we want our model’s intercept to be at the
Grand Mean. This is just a personal preference of mine because I am used
to it. This requires us to recode our categorical variables using
sum-to-zero aka contrast coding instead of dummy coding or other types.
Briefly, one group is assigned +1 and the other is assigned -1 and they
sum to zero.

``` r
# Create our main data object
data <- list()

    # Read in data file 
    data$raw <- read.csv("ABHV2018.csv",
                         na.strings="\"\"", #with blanks/ N/As set to blank ("\"\"")
                         stringsAsFactors = TRUE) # categorical/character data as "factor"
    data$raw$RatID <- as.factor(data$raw$RatID) # Convert RatID to factor type because it is not a numeric variable although it seems like one.
      
    # Change to Contrast Sum Coding (sum-to-zero). 
      # This attribute will be inherited for all data frames subset from this one,
      # so it is a good idea to perform this step now, instead of many times later.
    
      # adjust Age to contrast coding
      contrasts(data$raw$Age)=contr.sum(2)
      contrasts(data$raw$Age)
      # adjust Condition to contrast coding
      contrasts(data$raw$Condition)=contr.sum(2)
      contrasts(data$raw$Condition)
      
        
  ## Subset Ethanol Data ####
      
    # With Controls
    data$eth$ctrl <- subset(data$raw, Substance == "Ethanol")
        # Rescale to avoid issues with eigen values later
        data$eth$ctrl$recoded.conc <- car::recode(data$eth$ctrl$Concentration, "5 =.05; 20 =.20; 40 =.40")
      
    # No Controls
      # Subset rats that could drink ethanol from rats that never had the opportunity
      data$eth$no.ctrl <- subset(data$eth$ctrl, Condition != "CTRL")
    
    # Uncomment the 2 lines below to view your data frames in RStudio
    #View(data$eth$ctrl)
    #View(data$eth$no.ctrl)
    
  ## Subset Sucrose Data ####
    # With Controls
    data$suc$ctrl <- subset(data$raw, Substance == "Sucrose")
      # Rescale -- to avoid issues with eigen values later
      data$suc$ctrl$molarity <- car::recode(data$suc$ctrl$Concentration, ".34=.01; 3.4=.1; 34=1")
    
    # No Controls
      # Subset rats that could drink ethanol from rats that never had the opportunity
    data$suc$no.ctrl <- subset(data$suc$ctrl, Condition != "CTRL")
    
    #Uncomment the 2 lines below to view your data frames in RStudio
    #View(data$suc$ctrl)
    #View(data$suc$no.ctrl)
  
  ## Subset Water Data ####
    # With Controls
    data$h2o$ctrl <- subset(data$raw, Substance == "Water1" | Substance == "Water2")
      # Get rid of the 4 levels of Substance inherited from the original dataset
      data$h2o$ctrl$Substance <- factor(data$h2o$ctrl$Substance)
      # Recode this for clarity and easy use as a label in future graphs
      data$h2o$ctrl$trial <- dplyr::recode(data$h2o$ctrl$Substance,
                                          "Water1" = "Trial 1",
                                          "Water2" = "Trial 2")
      # Adjust contrasts to sum-to-zero now that there are 2 factors
      contrasts(data$h2o$ctrl$Substance)=contr.sum(2)
      contrasts(data$h2o$ctrl$Substance)

      
    # No Controls
      # Subset rats that could drink ethanol from rats that never had the opportunity
    data$h2o$no.ctrl <- subset(data$h2o$ctrl, Condition != "CTRL")      
    
    #Uncomment the 2 lines below to view your data frames in RStudio
    #View(data$h2o$ctrl)
    #View(data$h2o$no.ctrl)
```

Now we have our data list object with the following hierarchical
structure:

- `data`
  - `data$raw`
  - `data$eth`
    - `data$eth$ctrl`
    - `data$eth$no.ctrl`
  - `data$suc`
    - `data$suc$ctrl`
    - `data$suc$no.ctrl`
  - `data$h2o`
    - `data$h2o$ctrl`
    - `data$h2o$no.ctrl`

And we have also performed our coding adjustments. However, we still
need to further prepare our data to avoid issues during analysis such as
problems with variance inflation. If we mean-center the continuous
variables that will be predictors in an `lmer` or `glmer` model, the
variance inflation factor tends to remain within a tolerable range and
shouldn’t affect results. Thus we mean-center the variables we need to
below and save our workspace, so we can come back to this point later if
we mess something up.

``` r
  ## Mean Centering Variables ####
    
    # Ethanol
      # center Concentration to avoid issues with variance inflation factor (VIF) tolerances
      data$eth$ctrl$c.conc <- data$eth$ctrl$recoded.conc - mean(data$eth$ctrl$recoded.conc)
      # center TOTAL.ETOH.Swap.Consumed..g.kg.
      data$eth$no.ctrl$c.totale <- data$eth$no.ctrl$TOTAL.ETOH.Swap.Consumed..g.kg. - mean(data$eth$no.ctrl$TOTAL.ETOH.Swap.Consumed..g.kg.)
      # center Concentration 
      data$eth$no.ctrl$c.conc <- data$eth$no.ctrl$recoded.conc - mean(data$eth$no.ctrl$recoded.conc)
      
      # center MAC and ROC
      data$eth$no.ctrl$c.MAC <- data$eth$no.ctrl$MAC - mean(data$eth$no.ctrl$MAC)
      data$eth$no.ctrl$c.ROC <- data$eth$no.ctrl$ROC - mean(data$eth$no.ctrl$ROC)
      data$eth$no.ctrl$c.MAC3 <- data$eth$no.ctrl$MAC3 - mean(data$eth$no.ctrl$MAC3)
      data$eth$no.ctrl$c.ROC3 <- data$eth$no.ctrl$ROC3 - mean(data$eth$no.ctrl$ROC3)
    
    
    # Sucrose
      # Center molarity
      data$suc$ctrl$c.molarity <- data$suc$ctrl$molarity - mean(data$suc$ctrl$molarity)
      data$suc$no.ctrl$c.molarity <- data$suc$no.ctrl$molarity - mean(data$suc$no.ctrl$molarity)
      # center TOTAL.ETOH.Swap.Consumed..g.kg.
      data$suc$no.ctrl$c.totale <- data$suc$no.ctrl$TOTAL.ETOH.Swap.Consumed..g.kg. - mean(data$suc$no.ctrl$TOTAL.ETOH.Swap.Consumed..g.kg.)
    
    
    # Water
      # center TOTAL.ETOH.Swap.Consumed..g.kg.
      data$h2o$no.ctrl$c.totale <- data$h2o$no.ctrl$TOTAL.ETOH.Swap.Consumed..g.kg. - mean(data$h2o$no.ctrl$TOTAL.ETOH.Swap.Consumed..g.kg.)
    
    # Save the workspace
    save.image("ABHV_workspace.RData")
```

## Model and Comparison Storage

### Model Data

To accommodate and store each list object returned by our models, we
need another hierarchical list object (`models`) with a similar
structure to the data above. However, there is more nuance because we
are performing analyses for hedonic and aversive responses and, for each
of those, we have analyses that involve different variables (Overall
Model, Total Ethanol, MR, and MR3). So we will need a slightly different
list structure that accommodates each formula we discussed in the
background above. And, since we are comparing models based on their
ability to predict reactions to the substance of most interest to us
(Ethanol), we will have less analyses for Sucrose and Water. In the end
it should look something like this:

- Models
  - Ethanol
    - Aversive
      - Overall
      - Total Ethanol
      - Mean Alcohol Consumed & Rate of Change
      - Mean Alcohol Consumed & Rate of Change excluding day 3
    - Hedonic
      - Overall
      - Total Ethanol
      - Mean Alcohol Consumed & Rate of Change
      - Mean Alcohol Consumed & Rate of Change excluding day 3
  - Sucrose
    - Aversive
      - Overall
      - Total Ethanol
    - Hedonic
      - Overall
      - Total Ethanol
  - Water
    - Aversive
      - Overall
      - Total Ethanol
    - Hedonic
      - Overall
      - Total Ethanol

### Comparison Data

Our comparison data will look a bit different. The overall structure of
our comparison list object (`compars`) will be the same as the models;
however, we won’t need to make comparisons for variables that aren’t
significant. Therefore, we will just have objects in `compars` based on
the analysis they came from and the name of the fixed effect/interaction
they are from. For example the structure might look something like:

- Comparisons
  - Ethanol
    - Aversive
      - Overall
        - Condition
      - Total Ethanol
        - Age
        - Age\*Concentration (interaction)

We will discuss the organization of `plots` and related variables when
we reach that portion so it makes more sense. But we will need our
`models` and `compars` lists for the next steps, so lets make those now.

``` r
# List setup ####

## Models ####
models <- list()

## Comparisons ####
compars <- list()
```

# 2.1 Modeling

To quickly recap: This is count data that has some missing data due to
attrition and it has a continuous predictor variable that represents our
repeated measure (Concentration) since each animal reacted to each
concentration of each substance. Therefore a Poisson Generalized Linear
Mixed Effects Regression (glmer with a Poisson link function) is
appropriate for analysis because it can handle these issues/needs much
better than a lot of other models. As discussed before, there is some
nesting in the structure of the data and, for reasons related to
understanding for the audience the publication, we opted to do two
separate analyses on this data set instead of properly nesting it. We
will use ethanol as an illustrative example because it also has two
models that I wanted to compare.

For convenience, I will restate our variables:

**Independent (Outcome) Variables:**

- Total Aversive Responses (count data)
- Total Hedonic Responses (count data)

**Dependent (Predictor) Variables:**

- **Age** (during IAE period)
  - Adult
  - Adolescent
- Intermittent Ethanol Access (IAE) **Condition**
  - IAE (Had access to ethanol during IAE period)
    - **Ethanol Consumption** during IAE period *(A NESTED VARIABLE)*
  - CTRL (Did not have access to ethanol prior to Taste Reactivity
    Testing)
- **Concentration** of Ethanol
  - 5%
  - 20%
  - 40%

Now that we have our variables and out storage objects ready to go, we
can begin running our models. Remember that our continuous variables
will be centered variables (e.g. `c.conc` for Concentration)

## Ethanol Aversive Responses

``` r
# Ethanol vs Control (Overall)

  ## Ethanol Aversives GLMER (with EtOH vs CTRL)####
    models$eth$avers$overall <- glmer(Total.Aversive ~ c.conc*Age*Condition # predictors: full factorial fixed effects of centered concentration, age, and condition
                                      + (c.conc|RatID), # and the random effects of the intercept (RatID) and the slope of concentration
                                      data = data$eth$ctrl,
                                      family = poisson)
    # Get a summary of our output
    summary(models$eth$avers$overall)
```

Now we need to check to see if our assumption of normally distributed
residuals is correct using a simple plot.

The blue line is a normal distribution. The black line is our residual
distribution. From the graph we can see the residuals are slighly
kurtotic (pointier and a bit thinner than a normal curve) but mostly
normal. I would say that the assumption of normality has been met. Now,
on to interpretation. I have my summary from before, but I need to
compare my significant variables. `emmeans` is a convenient package for
this. I can look at the means from the model and compare my two levels
of my Condition variable to see which direction the difference is in and
how large the difference between means is.

``` r
  # Compare significant variables
  compars$eth$avers$overall <- list() # Create new comparison list for overall model
  compars$eth$avers$overall$condition <- emmeans(models$eth$avers$overall, ~ Condition) # Perform comparison of Condition with emmeans
  
  # We dont need a comparison from c.conc because we have a direction AND a magnitude on a continuous variable! WOO!
  
  summary(compars$eth$avers$overall$condition, type = "response") # get summary in the numerical space of the original variable not the log/Poisson space
  
  #Save Workspace
  save.image("ABHV_workspace.RData")
```

I document these numbers, save my workspace and then move on to the next
model.

``` r
# Ethanol Group Only: Total Ethanol Consumed (total.e)
  
  # predictors: full factorial fixed effects of centered concentration, age, and total ethanol consumed during drinking phase of study
  models$eth$avers$total.e <- glmer(Total.Aversive ~ c.conc*Age*c.totale 
                                    + (c.conc|RatID), # and the random effects of the intercept (RatID) and the slope of concentration
                                    data=data$eth$no.ctrl,
                                    family=poisson)

  # Model did not converge. Grab Theta and fixed effects data from model and put into temporary object
  ss <- getME(models$eth$avers$total.e,c("theta","fixef"))
  # Extend the number of iterations with optimizer and update model
  models$eth$avers$total.e <- update(models$eth$avers$total.e,start=ss,control=glmerControl(optCtrl=list(maxfun=2e6))) 
  
  # Model converged, get summary of output
  summary(models$eth$avers$total.e)
  # Remove temporary object to declutter workspace
  remove(ss)
```

Again, before I begin interpreting, I need to check to see if our
assumption of normally distributed residuals is correct using a plot of
probability density.

Again, the blue line is a normal distribution. The black line is our
residual distribution. From the graph we can see the residuals are,
again, slightly kurtotic but mostly normal. I would say that the
assumption of normality has been met her as well. Now, on to
interpretation. I need to look at the means from the model and compare
my two levels of my Condition variable to see which direction and how
large the difference is.

``` r
    # Compare significant variables
    compars$eth$avers$total.e <- list() # Create new comparison list for total.e model
    compars$eth$avers$total.e$age<- emmeans(models$eth$avers$total.e, ~ Age) # Perform comparison of Condition with emmeans
    summary(compars$eth$avers$total.e$age, type = "response") # get summary in the numerical space of the original variable not the log/Poisson space
    
      #Save Workspace
  save.image("ABHV_workspace.RData")
```

Copy down the data you need, save, rinse, and repeat…

``` r
  # predictors: note that these are not the full factorial fixed effects. I'm not interested in the higher level interactions here.
models$eth$avers$MR1 <-glmer(Total.Aversive ~ c.conc+Age+c.MAC+c.ROC
                 + c.conc:Age
                 + c.conc:c.MAC
                 + c.conc:c.ROC
                 + Age:c.MAC
                 + Age:c.ROC
                 + c.conc:Age:c.MAC
                 + c.conc:Age:c.ROC
                 + (c.conc|RatID), data=data$eth$no.ctrl, family=poisson)

# Model did not converge, used code below to extend # of iterations and start from where the previous model left off.
ss <- getME(models$eth$avers$MR1,c("theta","fixef"))
models$eth$avers$MR1 <- update(models$eth$avers$MR1,start=ss,control=glmerControl(optCtrl=list(maxfun=2e6)))
summary(models$eth$avers$MR1)
remove(ss)

# Check variance inflation factor (VIF)
vif(models$eth$avers$MR1)

# Compare with the previous model
AIC(models$eth$avers$MR1, models$eth$avers$total.e)
BIC(models$eth$avers$MR1, models$eth$avers$total.e)
```

Again, check normality of residuals

Similar to previous residuals. No planned comparisons here. This model
was not selected for interpretation (considerably higher BIC), so I
didn’t perform any. Then we repeat for the 4th model which excludes the
first day of drinking from calculations of MAC and RoC.

``` r
  # predictors: note that these are not the full factorial fixed effects. I'm not interested in the higher level interactions here.
models$eth$avers$MR3 <-glmer(Total.Aversive ~ c.conc+Age+c.MAC3+c.ROC3
                 + c.conc:Age
                 + c.conc:c.MAC3
                 + c.conc:c.ROC3
                 + Age:c.MAC3
                 + Age:c.ROC3
                 + c.conc:Age:c.MAC3
                 + c.conc:Age:c.ROC3
                 + (c.conc|RatID), data=data$eth$no.ctrl, family=poisson)

# Model did not converge, used code below to extend # of iterations and start from where the previous model left off.
ss <- getME(models$eth$avers$MR3, c("theta","fixef"))
models$eth$avers$MR3 <- update(models$eth$avers$MR3,start=ss,control=glmerControl(optCtrl=list(maxfun=2e9)))
remove(ss)
summary(models$eth$avers$MR3)

vif(models$eth$avers$MR3)
AIC(models$eth$avers$MR3, models$eth$avers$MR1, models$eth$avers$total.e)
BIC(models$eth$avers$MR3, models$eth$avers$MR1, models$eth$avers$total.e)
```

Again, check normality of residuals

And again, things look pretty normal. This one also has no planned
comparisons and was not selected for the same reason as the previous.
The remaining model runs are not shown here for brevity.

# Plotting/Graphing GLMERs with ggplot2

In the end, our specific lists of objects with abbreviations looks like
this:

- data
  - raw
  - ethanol
    - with controls
    - no controls
  - sucrose
    - with controls
    - no controls
  - water
    - with controls
    - no controls
- models
  - ethanol
    - aversive
      - overall
      - total ethanol
      - MR (Mean Alcohol Consumed & Rate of Change)
      - MR3 (Mean Alcohol Consumed & Rate of Change excluding day 3)
    - hedonic
      - overall
      - total ethanol
      - MR (Mean Alcohol Consumed & Rate of Change)
      - MR3 (Mean Alcohol Consumed & Rate of Change excluding day 3)
  - sucrose
    - aversive
      - overall
      - total ethanol
      - MR (Mean Alcohol Consumed & Rate of Change)
    - hedonic
      - overall
      - total ethanol
      - MR (Mean Alcohol Consumed & Rate of Change)
  - water
    - aversive
      - overall
      - total ethanol
      - MR (Mean Alcohol Consumed & Rate of Change)
    - hedonic
      - overall
      - total ethanol
      - MR (Mean Alcohol Consumed & Rate of Change)
- comparisons
- plots

## Including Plots

You can also embed plots, for example:

Note that the `echo = FALSE` parameter was added to the code chunk to
prevent printing of the R code that generated the plot.
