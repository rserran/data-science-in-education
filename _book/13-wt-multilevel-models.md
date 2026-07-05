# Walkthrough 7: Multilevel models {#c13}

## Topics emphasized

- Transforming data
- Modeling data
- Communicating results

## Functions introduced

- `fastDummies::dummy_cols()`
- `lme4::lmer()`
- `performance::icc()`

## Vocabulary

- dummy coding
- hierarchical linear model  
- intra-class correlation
- multilevel model  

## Chapter overview

The purpose of this walkthrough is to explore students' performance in online courses. While this and the analysis in [Walkthrough 1/Chapter 7](#c07) focus on the time students spent in the course, this walkthrough focuses on the effects of attending a particular course. 

To do that, you'll use multilevel models, which will help you account for students who shared classes. While the conceptual details underlying multilevel models are complex, they address a basic problem: How might you include variables like classes and student grouping levels in the model? 

Note that while using multilevel models is accessible through R, some of the concepts remain challenging. In these cases, you can start by running the model with data you've already collected. Later, you can revisit the technical details in this chapter to help you go deeper with the analyses.

### Background

Using multilevel models will help you account for the way individual students are grouped together into higher-level units, like classes. Multilevel models do something different from a simple linear regression like the ones described in [Walkthrough 1/Chapter 7](#c07) and [Walkthrough 4/Chapter 10](#c10). They estimate the effect of being a student in a particular group. A multilevel model uses a different way to standardize the estimates for each group based on how systematically different the groups are, relative to the effect on the dependent variable.

Though these conceptual details are complex, fitting them is fortunately straightforward and will be familiar if you have used R's `lm()` function. Let's get started.

### Data source

We'll use the same data source on students' motivation in online science classes that we used in [Walkthrough 1](#c07).

### Methods

Does the amount of time students spend on a course depend on the specific course they're in? Does the amount of time students spend on a course affect the points they earn? There are a number of ways to approach these questions. One way is to use a linear model.

To do this, you'll assign codes to the groups so you can include them in the model. You'll use a technique called "dummy coding". Dummy coding means transforming a variable with multiple categories into new variables, where each variable indicates the presence and absence of each category.

## Load packages

We will load the tidyverse and a few other packages for using multilevel models: 
{lme4} [@R-lme4] and {performance} [@R-performance].

If you haven't already, you'll need to install {lme4}, {performance}, and {fastDummies} to do the rest of this walkthrough. If needed, you can review the [Packages](#c06p) section of the [Foundational Skills](#c06) chapter for an overview of installing packages.

The remaining packages, ({tidyverse}, {sjPlot}, and {dataedu}), are used in other chapters. But if you have not installed these before, you can install them using the `install.packages()` function. 


``` r
library(tidyverse)
library(fastDummies)
library(sjPlot)
library(lme4)
library(performance)
library(dataedu)
```

## The role of dummy codes

Before we import our data, let's learn about dummy coding. In this discussion, you'll learn how dummy coding works using the {fastDummies} package. In this section, you'll use base R and data.frame instead of the {tidyverse} and tibbles.

Start by looking at the `penguins` data that comes loaded with the {tidyverse} using the handy `glimpse()` function.


``` r
glimpse(penguins)
```

```
## Rows: 344
## Columns: 8
## $ species     <fct> Adelie, Adelie, Adelie, Adelie, Adelie, Adelie, Adelie, Ad…
## $ island      <fct> Torgersen, Torgersen, Torgersen, Torgersen, Torgersen, Tor…
## $ bill_len    <dbl> 39.1, 39.5, 40.3, NA, 36.7, 39.3, 38.9, 39.2, 34.1, 42.0, …
## $ bill_dep    <dbl> 18.7, 17.4, 18.0, NA, 19.3, 20.6, 17.8, 19.6, 18.1, 20.2, …
## $ flipper_len <int> 181, 186, 195, NA, 193, 190, 181, 195, 193, 190, 186, 180,…
## $ body_mass   <int> 3750, 3800, 3250, NA, 3450, 3650, 3625, 4675, 3475, 4250, …
## $ sex         <fct> male, female, female, NA, female, male, female, male, NA, …
## $ year        <int> 2007, 2007, 2007, 2007, 2007, 2007, 2007, 2007, 2007, 2007…
```

As you can see above, the `species` variable is a factor. Recall that factor data types are categorical variables. They associate a row with a category, or level, of that variable. So how do you consider factor variables in your model? `species` seems to be made up of words such as "Adelie".

A common way to approach this is through dummy coding, where you create new variables for each of the possible values of `species`. These new variables have a value of 1 when the row is associated with that level.

Put the {fastDummies} package to work on this task. How many possible values are there for `species`? You can check with the `levels` function: 


``` r
levels(penguins$species)
```

```
## [1] "Adelie"    "Chinstrap" "Gentoo"
```

The function `dummy_cols()` returns a `tibble` where specified columns (such as `species`) are given dummy attributes.

Please note that the code below will trigger a warning. A warning will run the code but alert you that something should be changed. This warning is because of an outdated parameter in the `dummy.data.frame()` function that hasn't been updated. R 3.6 and above triggers a warning when this happens. This is a good reminder that packages evolve (or don't), so you'll need to be aware of any changes when using them for analysis.


``` r
d_penguins <- penguins %>% 
  dummy_cols("species")

glimpse(d_penguins)
```

```
## Rows: 344
## Columns: 11
## $ species           <fct> Adelie, Adelie, Adelie, Adelie, Adelie, Adelie, Adel…
## $ island            <fct> Torgersen, Torgersen, Torgersen, Torgersen, Torgerse…
## $ bill_len          <dbl> 39.1, 39.5, 40.3, NA, 36.7, 39.3, 38.9, 39.2, 34.1, …
## $ bill_dep          <dbl> 18.7, 17.4, 18.0, NA, 19.3, 20.6, 17.8, 19.6, 18.1, …
## $ flipper_len       <int> 181, 186, 195, NA, 193, 190, 181, 195, 193, 190, 186…
## $ body_mass         <int> 3750, 3800, 3250, NA, 3450, 3650, 3625, 4675, 3475, …
## $ sex               <fct> male, female, female, NA, female, male, female, male…
## $ year              <int> 2007, 2007, 2007, 2007, 2007, 2007, 2007, 2007, 2007…
## $ species_Adelie    <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1…
## $ species_Chinstrap <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0…
## $ species_Gentoo    <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0…
```

`d_penguins` has _all_ of the original variables and three additional variables that `penguins` did not have. Specifically, `d_penguins` has one variable for each of the three species of penguin, with a zero if that particular observation is not associated with that species and a one if it is. 

You can see this easiest if you select the `species` variable and the three new dummy-coded variables. Look at the first 5 rows to examine and check:


``` r
d_penguins %>%
  select(species, species_Adelie, species_Chinstrap, species_Gentoo) %>%
  slice_head(n = 5)
```

```
##   species species_Adelie species_Chinstrap species_Gentoo
## 1  Adelie              1                 0              0
## 2  Adelie              1                 0              0
## 3  Adelie              1                 0              0
## 4  Adelie              1                 0              0
## 5  Adelie              1                 0              0
```

For example, the first row of the data frame above has a `1` for `species_Adelie` and `0` for the other two dummy-coded variables, which indicates that this row corresponds to an observation of an Adelie penguin.

Now that we have a basic understanding of how dummy codes work, it's time to explore how to use them in the model. When fitting models in R that include factor variables, R displays coefficients for all but one level in the model output. The factor level that's not explicitly named is called the "reference group". The reference group is the level that all other levels are compared to.

Why can't R explicitly name every level of a dummy-coded column? It has to do with how the dummy codes are used for comparison of groups. The purpose of the dummy code is to show how different the dependent variable is for all of the observations in a group. Go back to the `penguins` dataset. Consider all the penguins that are in the "Adelie" group. To represent how different those penguins are, they have to be compared to another group of penguins. In R, you'd compare all the penguins in the "Adelie" group to the reference group of penguins. The reference group of penguins would be the group that is not explicitly named in the model output.

However, if every level of species is dummy-coded, there would be no single group to compare to. For this reason, one group needs to be selected as the reference group.

## Import data 

Let's return to the online science class data. You'll be using the same dataset you used in [Chapter 7](#c07). Load that dataset now from the {dataedu} package. 


``` r
dat <- dataedu::sci_mo_processed
```

To wrap up our discussion about factor variables, levels, and dummy codes, let's look at how many classes are represented in the `course_id` variable. These classes will be the factor levels you'll use in the model soon. Use the `count()` function to see how many courses there are: 


``` r
dat %>% 
  count(course_id)
```

```
## # A tibble: 26 × 2
##    course_id         n
##    <chr>         <int>
##  1 AnPhA-S116-01    43
##  2 AnPhA-S116-02    29
##  3 AnPhA-S216-01    43
##  4 AnPhA-S216-02    17
##  5 AnPhA-T116-01    11
##  6 BioA-S116-01     34
##  7 BioA-S216-01      7
##  8 BioA-T116-01      2
##  9 FrScA-S116-01    70
## 10 FrScA-S116-02    12
## # ℹ 16 more rows
```

## Analysis

### Regression (linear model) analysis with dummy codes

Before you fit the model, let's talk about the dataset. You will keep the variables you used in the last set of models - `TimeSpent` and `course_id` - as independent variables. Recall that `TimeSpent` is the amount of time in minutes that a student spent in a course and `course_id` is a unique identifier for a particular course. In this walkthrough, we'll predict students' final grade rather than the `percentage_earned` variable that we created in [Chapter 7](#c07).

Since you will be using the final grade variable a lot, rename it to make it easier to type.


``` r
dat <- 
  dat %>% 
  rename(final_grade = FinalGradeCEMS)
```

Now you can fit the model. Save the model object to `m_linear_dc`, where the `dc` stands for dummy code. Later you'll be working with `course_id` as a factor variable, so you can expect to see `lm()` treat it as a dummy coded variable. The model output will include a reference variable for `course_id` that all other levels of `course_id` will be compared against. 


``` r
m_linear_dc <- 
  lm(final_grade ~ TimeSpent_std + course_id, data = dat)
```

The output from the model will be long. This is because each course in the `course_id` variable will get its own line in the model output. You can see that using `tab_model()` from {sjPlot}:


``` r
tab_model(m_linear_dc,
          title = "Table 13.1")
```

<table style="border-collapse:collapse; border:none;">
<caption style="font-weight: bold; text-align:left;">Table 13.1</caption>
<tr>
<th style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm;  text-align:left; ">&nbsp;</th>
<th colspan="3" style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm; ">final grade</th>
</tr>
<tr>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  text-align:left; ">Predictors</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">Estimates</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">CI</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">p</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">(Intercept)</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">73.20</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">67.20&nbsp;&ndash;&nbsp;79.20</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">TimeSpent std</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">9.66</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">7.91&nbsp;&ndash;&nbsp;11.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;1.59</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;10.88&nbsp;&ndash;&nbsp;7.70</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.737</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;9.05</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;17.44&nbsp;&ndash;&nbsp;-0.67</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.034</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;4.51</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;16.41&nbsp;&ndash;&nbsp;7.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.457</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">7.24</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;6.34&nbsp;&ndash;&nbsp;20.82</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.296</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;3.56</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;12.67&nbsp;&ndash;&nbsp;5.55</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.443</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;14.67</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;31.61&nbsp;&ndash;&nbsp;2.26</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.089</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">9.18</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;18.84&nbsp;&ndash;&nbsp;37.20</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.520</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">12.02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">4.33&nbsp;&ndash;&nbsp;19.70</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.002</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;3.14</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;17.36&nbsp;&ndash;&nbsp;11.08</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.665</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">3.51</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;5.43&nbsp;&ndash;&nbsp;12.46</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.441</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">5.23</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;14.98&nbsp;&ndash;&nbsp;25.43</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.612</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">9.92</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">2.41&nbsp;&ndash;&nbsp;17.43</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.010</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">7.37</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;2.70&nbsp;&ndash;&nbsp;17.45</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.151</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">2.38</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;25.65&nbsp;&ndash;&nbsp;30.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.868</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">15.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;2.92&nbsp;&ndash;&nbsp;33.72</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.099</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">8.12</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;12.08&nbsp;&ndash;&nbsp;28.33</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.430</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">4.06</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;5.67&nbsp;&ndash;&nbsp;13.79</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.413</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">2.02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;9.89&nbsp;&ndash;&nbsp;13.93</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.739</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;18.75</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;57.86&nbsp;&ndash;&nbsp;20.36</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.347</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;6.41</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;15.04&nbsp;&ndash;&nbsp;2.22</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.145</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;2.76</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;13.47&nbsp;&ndash;&nbsp;7.95</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.613</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;2.05</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;16.97&nbsp;&ndash;&nbsp;12.87</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.787</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">15.35</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">6.99&nbsp;&ndash;&nbsp;23.71</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">5.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;6.01&nbsp;&ndash;&nbsp;16.82</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.353</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">20.73</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;7.23&nbsp;&ndash;&nbsp;48.70</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.146</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm; border-top:1px solid;">Observations</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left; border-top:1px solid;" colspan="3">573</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">R<sup>2</sup> / R<sup>2</sup> adjusted</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">0.252 / 0.216</td>
</tr>

</table>



Those are a lot of effects. The model estimates the effects of being in each class, accounting for the time students spent on a course and the class they were in. You know this because the model output includes the time spent (`TimeSpent_std`) variable and subject variables (like `course_id[AnPhA-S116-02]`). 

If you count the number of courses, you see that there are 25, not 26. One has
been automatically selected as the reference group, and every other class's
coefficient represents how different each class is from it. The intercept's
value of 73.20 represents the number of percentage points that students in the reference
group class are estimated to earn. `lm()` automatically picks the first level of the `course_id` variable as the reference group when it is converted to a factor. In this case, the course associated with course ID `course_idAnPhA-S116-01`, a first-semester physiology course, is picked as the reference group.

What if you want to pick another class as the reference variable? For example, say you want `course\_idPhysA-S116-01` (the first section of the physics class offered during this semester and year) to be the reference group. You can do this by using the `fct_relevel()` function, which is part of the {tidyverse} suite of packages. Note that before using `fct_relevel()`, the variable `course_id` was a character data type, which `lm()` coerced into a factor data type when including it as a predictor variable. Using `fct_relevel()` will explicitly convert `course_id` to a factor data type. It's important to note that the actual value of the variable is in square brackets in the output, whereas `course_id` is the variable name; in the output, these are combined to make it easier to tell what the values represent (e.g., "PhysA-S116-01" is an ID for a course).

Now use `fct_relevel()` and `mutate()` to re-order the levels within a factor, so that the "first" level changes:


``` r
dat <-
  dat %>%
  mutate(course_id = fct_relevel(course_id, "PhysA-S116-01"))
```

Now fit the model again with the newly releveled `course_id` variable. Give it a different name, `m_linear_dc_1`: 


``` r
m_linear_dc_1 <- 
  lm(final_grade ~ TimeSpent_std + course_id, data = dat)

tab_model(m_linear_dc_1,
          title = "Table 13.2")
```

<table style="border-collapse:collapse; border:none;">
<caption style="font-weight: bold; text-align:left;">Table 13.2</caption>
<tr>
<th style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm;  text-align:left; ">&nbsp;</th>
<th colspan="3" style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm; ">final grade</th>
</tr>
<tr>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  text-align:left; ">Predictors</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">Estimates</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">CI</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">p</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">(Intercept)</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">88.55</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">82.83&nbsp;&ndash;&nbsp;94.27</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">TimeSpent std</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">9.66</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">7.91&nbsp;&ndash;&nbsp;11.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;15.35</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;23.71&nbsp;&ndash;&nbsp;-6.99</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;16.94</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;26.20&nbsp;&ndash;&nbsp;-7.67</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;24.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;32.77&nbsp;&ndash;&nbsp;-16.04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;19.86</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;31.71&nbsp;&ndash;&nbsp;-8.01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;8.11</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;21.64&nbsp;&ndash;&nbsp;5.42</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.240</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;18.91</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;27.72&nbsp;&ndash;&nbsp;-10.09</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;30.02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;46.80&nbsp;&ndash;&nbsp;-13.24</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;6.17</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;34.09&nbsp;&ndash;&nbsp;21.75</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.664</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;3.33</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;10.76&nbsp;&ndash;&nbsp;4.10</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.379</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;18.49</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;32.58&nbsp;&ndash;&nbsp;-4.39</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.010</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;11.84</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;20.59&nbsp;&ndash;&nbsp;-3.08</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.008</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;10.12</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;30.32&nbsp;&ndash;&nbsp;10.08</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.326</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;5.43</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;12.62&nbsp;&ndash;&nbsp;1.75</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.138</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;7.97</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;17.85&nbsp;&ndash;&nbsp;1.90</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.113</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;12.97</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;40.89&nbsp;&ndash;&nbsp;14.95</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.362</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.05</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;18.15&nbsp;&ndash;&nbsp;18.25</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.996</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;7.22</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;27.47&nbsp;&ndash;&nbsp;13.02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.484</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;11.29</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;20.98&nbsp;&ndash;&nbsp;-1.60</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.022</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;13.33</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;25.16&nbsp;&ndash;&nbsp;-1.49</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.027</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;34.10</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;73.17&nbsp;&ndash;&nbsp;4.97</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.087</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;21.76</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;30.29&nbsp;&ndash;&nbsp;-13.23</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;18.11</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;28.66&nbsp;&ndash;&nbsp;-7.56</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;17.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;32.22&nbsp;&ndash;&nbsp;-2.58</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.021</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;9.94</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;21.16&nbsp;&ndash;&nbsp;1.28</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.082</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">5.39</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">&#45;22.55&nbsp;&ndash;&nbsp;33.32</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">0.705</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm; border-top:1px solid;">Observations</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left; border-top:1px solid;" colspan="3">573</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">R<sup>2</sup> / R<sup>2</sup> adjusted</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">0.252 / 0.216</td>
</tr>

</table>



You can now see that "PhysA-S116-01" is no longer listed as an independent variable. Every coefficient listed in this model is in comparison to the new reference variable, "PhysA-S116-01". You also see that `course_id` is now recognized as a factor data type. 

Dummy codes are used in nearly every case where you need to fit a model with variables that are factors. You've already seen one benefit of using R functions like `lm()`, or the `lme4::lmer()` function discussed later. These functions automatically convert character data types into factor data types. 

For example, imagine you include a variable for courses that has values like "mathematics", "science", "english language" (typed like that!), "social studies", and "art" as an argument in `lm()`. `lm()` will automatically dummy-code these for you. Note that you'll need to decide if you want to use the default reference group or if you should use `fct_relevel()` to pick a different one. 

Lastly, it's worth noting that there may be some situations where you don't want a single factor level to act as a reference group. In such cases, no intercept is estimated. This can be done by passing a -1 as the first value after the tilde, as follows:


``` r
# specifying the same linear model as the previous example, but using a "-1" to indicate that there should not be a reference group
m_linear_dc_2 <- 
  lm(final_grade ~ -1 + TimeSpent_std + course_id, data = dat)

tab_model(m_linear_dc_2,
          title = "Table 13.3")
```

<table style="border-collapse:collapse; border:none;">
<caption style="font-weight: bold; text-align:left;">Table 13.3</caption>
<tr>
<th style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm;  text-align:left; ">&nbsp;</th>
<th colspan="3" style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm; ">final grade</th>
</tr>
<tr>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  text-align:left; ">Predictors</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">Estimates</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">CI</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">p</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">TimeSpent std</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">9.66</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">7.91&nbsp;&ndash;&nbsp;11.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">88.55</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">82.83&nbsp;&ndash;&nbsp;94.27</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">73.20</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">67.20&nbsp;&ndash;&nbsp;79.20</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">71.61</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">64.38&nbsp;&ndash;&nbsp;78.83</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">64.15</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">58.12&nbsp;&ndash;&nbsp;70.17</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">68.69</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">58.35&nbsp;&ndash;&nbsp;79.04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idAnPhA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">80.44</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">68.20&nbsp;&ndash;&nbsp;92.67</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">69.64</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">62.89&nbsp;&ndash;&nbsp;76.40</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">58.53</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">42.74&nbsp;&ndash;&nbsp;74.32</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idBioA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">82.38</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">55.04&nbsp;&ndash;&nbsp;109.72</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">85.22</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">80.46&nbsp;&ndash;&nbsp;89.98</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">70.06</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">57.18&nbsp;&ndash;&nbsp;82.94</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">76.71</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">70.08&nbsp;&ndash;&nbsp;83.34</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S116&#45;04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">78.43</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">59.08&nbsp;&ndash;&nbsp;97.78</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">83.12</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">78.72&nbsp;&ndash;&nbsp;87.52</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">80.57</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">72.51&nbsp;&ndash;&nbsp;88.64</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">75.58</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">48.23&nbsp;&ndash;&nbsp;102.92</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-S216&#45;04</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">88.60</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">71.31&nbsp;&ndash;&nbsp;105.89</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idFrScA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">81.32</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">61.94&nbsp;&ndash;&nbsp;100.71</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">77.26</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">69.49&nbsp;&ndash;&nbsp;85.03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">75.22</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">64.88&nbsp;&ndash;&nbsp;85.56</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S116&#45;03</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">54.45</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">15.80&nbsp;&ndash;&nbsp;93.10</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>0.006</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">66.79</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">60.50&nbsp;&ndash;&nbsp;73.07</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-S216&#45;02</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">70.44</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">61.57&nbsp;&ndash;&nbsp;79.31</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idOcnA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">71.15</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">57.48&nbsp;&ndash;&nbsp;84.81</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-S216&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">78.60</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">68.94&nbsp;&ndash;&nbsp;88.27</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">course_idPhysA-T116&#45;01</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">93.93</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">66.60&nbsp;&ndash;&nbsp;121.27</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm; border-top:1px solid;">Observations</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left; border-top:1px solid;" colspan="3">573</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">R<sup>2</sup> / R<sup>2</sup> adjusted</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">0.943 / 0.940</td>
</tr>

</table>



### A deep-dive into multilevel models

Let's discuss multilevel models more by exploring nuances of using them with education data. 

*Dummy coding variables and ease of interpretation* 

Using multiple levels is a trade-off between accounting for multiple variables and ease of interpretation. A technique like dummy coding is a helpful strategy for working with multiple groups of predictors. In this walkthrough, we estimated the effects of being in one of the five online science courses. Dummy coding can go even further by accounting for multiple course sections for each subject. But consider the challenge of interpreting the effects, where each class and section becomes its own line in the model output. It gets complicated interpreting these effects relative to the intercept.

*Multilevel models and the assumption of independent data points* 

Including a group in your model helps you meet the assumption of independent data points. Linear regression models assume that each data point is not correlated with another. This is what is meant by the "assumption of independence" or of "independently and identically distributed" (*i.i.d.*) residuals (Field, Miles, & Field, 2012).

A linear regression model that considers students in different sections (e.g., for an introductory life science class and in different laboratory sections) as a single sample will assume that the outcome for each student is not correlated with the outcome of another student. This is a tough assumption, considering that students in the same section may perform similarly. Factors include having the same instructor, common scheduling, or students studying together.

Adding a section group to the model helps us meet the assumption of independent data points by considering the effect of being in a particular section. Generally speaking, analysts have the goal of accounting for students sharing a class. This is very different from determining the effect of any one particular class on the outcome.

*Regularization* 

So far we've learned that multilevel models help us meet the assumption of independent data points by considering groups in the model. They do this by estimating the effect of being a student in each group, but with a key distinction from linear models: instead of determining how different the observations in a group are from those in the reference group, the multilevel model "regularizes" the difference based on how systematically different the groups are. 

You may also see the term "shrink" to describe this. The term "shrinkage" is used because the group-level estimates (e.g., for classes) obtained through multilevel modeling can never be larger than those from a linear regression model. As described earlier, when there are groups included in the model, a regression effectively estimates the effect for each group independent of the others.

Groups comprising individuals who are consistently higher or lower than individuals on average are not regularized very much. Their estimated differences are close to the estimate from a multilevel model. But groups with only a few individuals or a lot of variability within individuals are regularized more. The way that a multilevel model does this "regularizing" is by considering the groups to be samples from a larger population of classes.  

Regularization relates to Bayesian approaches to data analysis. These approaches are different from the statistical approaches you may have learned in quantitative methods courses. The key idea of the Bayesian approach is that it naturally incorporates prior information — what we already know or expect before seeing the data. 

This is directly connected to regularization in multilevel models: the estimates for each group are "pulled" toward the overall average, much like a Bayesian analysis pulls estimates toward a prior. In fact, under certain conditions, the estimates from a multilevel model are mathematically equivalent to those from a Bayesian model. 

The Bayesian approach also has a notable difference from more familiar statistical approaches: instead of p-values, we can make direct probability statements about our estimates — for example, stating that there is a 95% probability that the effect of being in a particular class falls within a specific range.

Diving deep into Bayesian methods is beyond the scope of this work, but you will estimate a Bayesian version of a multilevel model near the end of this chapter. There are excellent resources on Bayesian modeling in R, particularly the wonderful @bayesrulesbook. Those interested in the use of Bayesian methods in educational research may also find @kubsch2021beyond helpful.

*Intra-class correlation coefficient*

Multilevel models are very common in educational research because they help account for students taking the same classes or going to the same school (see Raudenbush & Bryk, 2002). Using multilevel models means that the assumption of independence can be addressed. It means that individual coefficients for classes do not need to be included, though they are accounted for in the model. 

So what's the most useful way to report the importance of groups in a model? It's usually in the form of the *intra-class correlation coefficient* (ICC), which explains the proportion of variation in the dependent variable that the groups explain. Smaller ICCs, like ICCs with values of 0.05, representing 5% of the variation in the dependent variable, mean that the groups are not very important. Larger ICCs, like ICCs with values of 0.10 or larger, suggest that groups are important. When groups are important, not including them in the model may ignore the assumption of independence. 

### Multilevel model analysis

Fortunately, for all the complicated details, multilevel models are relatively
easy to use in R. You'll need a new package for this next example. One of the most common for
estimating these types of models is {lme4}. You'll use `lme4::lmer()` very similarly to the `lm()` function, but you'll pass it an additional argument for the *groups* included in the model. This model is often referred to as a "varying intercepts" multilevel model. The difference between the groups is the effect of being a student in a class: the intercepts between the groups vary.

Now you can fit the multilevel model using the `lmer()` function:


``` r
m_course <- 
  lmer(final_grade ~ TimeSpent_std + (1|course_id), data = dat)
```

You'll notice something that you didn't see using `lm()`. You used a new term (`(1|course_id)`) to model the group (in this case, courses). With `lmer()`, these group terms are in parentheses and on the right side of the bar. `|course_id` tells `lmer()` that courses are groups in the data and they should be included in the model. The `1` on the left side of the bar tells `lmer()` that you want varying intercepts for each group. 1 is used to denote the intercept.

If you're familiar with Bayesian methods, you'll appreciate a connection here (@gelman2006data). Regularizing in a multilevel model takes data across all groups into account when generating estimates for each group. The data for all of the classes can be interpreted as a Bayesian prior for the group estimates.

There's more you can do with `lmer()`. For example, you can include different effects for each group in the model output, so that each group has its own slope. To explore techniques like this and more, we recommend the book by West, Welch, and Galecki [2014], which provides an excellent walkthrough on how to specify varying slopes using `lmer()`. 

## Results

Let's view the results using the `tab_model()` function from {sjPlot} again.


``` r
tab_model(m_course,
          title = "Table 13.4")
```

<table style="border-collapse:collapse; border:none;">
<caption style="font-weight: bold; text-align:left;">Table 13.4</caption>
<tr>
<th style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm;  text-align:left; ">&nbsp;</th>
<th colspan="3" style="border-top: double; text-align:center; font-style:normal; font-weight:bold; padding:0.2cm; ">final grade</th>
</tr>
<tr>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  text-align:left; ">Predictors</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">Estimates</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">CI</td>
<td style=" text-align:center; border-bottom:1px solid; font-style:italic; font-weight:normal;  ">p</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">(Intercept)</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">75.63</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">72.41&nbsp;&ndash;&nbsp;78.84</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; ">TimeSpent std</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">9.45</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  ">7.74&nbsp;&ndash;&nbsp;11.16</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:center;  "><strong>&lt;0.001</strong></td>
</tr>
<tr>
<td colspan="4" style="font-weight:bold; text-align:left; padding-top:.8em;">Random Effects</td>
</tr>

<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">&sigma;<sup>2</sup></td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">385.33</td>
</tr>

<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">&tau;<sub>00</sub> <sub>course_id</sub></td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">38.65</td>

<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">ICC</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">0.09</td>

<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">N <sub>course_id</sub></td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">26</td>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm; border-top:1px solid;">Observations</td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left; border-top:1px solid;" colspan="3">573</td>
</tr>
<tr>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; text-align:left; padding-top:0.1cm; padding-bottom:0.1cm;">Marginal R<sup>2</sup> / Conditional R<sup>2</sup></td>
<td style=" padding:0.2cm; text-align:left; vertical-align:top; padding-top:0.1cm; padding-bottom:0.1cm; text-align:left;" colspan="3">0.170 / 0.246</td>
</tr>

</table>



For `lmer()` models, `tab_model()` provides the output, including fit statistics, coefficients and their standard errors and estimates. There are two things to note about this output:

1.  *p*-values are not automatically provided, due to debates in the wider field about how to calculate the degrees of freedom for coefficients^[ Run `?lme4::pvalues` to see a discussion of the issue and solutions. The lmerTest is helpful as an easy solution, though you may prefer some of the recommendations available in `?lme4::pvalues` because the technique lmerTest uses has known issues.]

2.  In addition to the coefficients, there are also estimates for how much variability there is between the groups.

A common way to understand variability at the group level is to calculate the *intra-class* correlation. This value is the proportion of the variability in the outcome (the *y*-variable) accounted for solely by the groups identified in the model. There is a useful function in the {performance} package for doing this.

You can install the {performance} package by typing this code in your Console: 


``` r
install.packages("performance")
```

After that, try this function: 


``` r
icc(m_course)
```

```
## # Intraclass Correlation Coefficient
## 
##     Adjusted ICC: 0.091
##   Unadjusted ICC: 0.076
```

This shows that 9.1% of the variability in the percentage of points students earned can be explained by the class they are in. The adjusted ICC is typically reported. This value is the proportion of the variability in the dependent variable explained by the groups (courses). See the documentation for `icc()` for details on the interpretation of the conditional ICC.

### Adding additional levels

Now add additional levels. The data you're using is all from one school, so you cannot estimate a "two-level" model. Imagine, however, that you have student data from 230 classes instead of 26 and that these classes were from 15 schools. 

You could estimate a two-level, varying intercepts (two groups with effects) model similar to the model above, but with another group added for the school. The model will automatically account for the way that the classes are nested within the schools (Bates, Maechler, Bolker, & Walker, 2015).

You don't have a variable containing the name of different schools. But if you did, you could fit the model like this, where `school_id` is the variable containing different schools: 


``` r
# this model would specify a group effect for both the course and school
m_course_school <- 
  lmer(final_grade ~ TimeSpent + (1|course_id) + (1|school_id), data = dat)
```

If you estimated this model (and then use the `icc()` function), you would see two ICC values representing the proportion of the variation in the dependent variable explained by the course and the school. Note that as long as the courses are uniquely labeled, it's not necessary to explicitly nest the courses within schools.

The {lme4} package was designed for complex multilevel models so you can add more levels. For more on advanced multilevel techniques like these, see @west2014linear.

## Conclusion

In this walkthrough, the groups in our multilevel model are classes. But multilevel models can be used for other cases where data is associated with a common group. For example, if students respond to quizzes over time, then the multiple quiz responses for each student are "grouped" within students. In such a case, you'd specify students as the "grouping factor" instead of courses. Moreover, multilevel models can include multiple groups even if the groups are of very different kinds. For example, this would be the case if students from multiple classes responded to multiple quizzes.

Note that the groups in multilevel models do not need to be nested. They can
also be "crossed", as may be the case for data from teachers in different schools who attended different teacher preparation programs. Not every teacher in a school attended the same teacher preparation program, and graduates from every teacher preparation program are
highly unlikely to all teach in the same school.

Finally, as noted earlier, multilevel models have similarities to the Bayesian
methods which are becoming more common among educational data scientists. There is much more that can be done with multilevel models; we have more
recommendations in the [Additional Resources](#c18) chapter.
