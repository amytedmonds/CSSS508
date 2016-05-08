CSSS 508, Week 7
===
author: Rebecca Ferrell
date: May 11, 2016
transition: rotate
width: 1100
height: 750


Vectorization
===
type: section


Non-vectorized example
===
incremental: true

We have a vector of numbers, and we want to add 1 to each entry.

```r
my_vector <- rnorm(1000000)
```

A `for` loop works but is relatively slow:

```r
for_start <- proc.time() # start the clock
new_vector <- rep(NA, length(my_vector))
for(position in 1:length(my_vector)) {
    new_vector[position] <- my_vector[position] + 1
}
(for_time <- proc.time() - for_start) # time elapsed
```

```
   user  system elapsed 
  2.055   0.040   2.146 
```


Vectorization wins
===
incremental: true

Recognize that we can instead use R's vector addition (with recycling):

```r
vec_start <- proc.time()
new_vector <- my_vector + 1
(vec_time <- proc.time() - vec_start)
```

```
   user  system elapsed 
  0.060   0.008   0.067 
```

```r
for_time / vec_time
```

```
    user   system  elapsed 
34.25000  5.00000 32.02985 
```

Vector/matrix arithmetic is implemented using fast, optimized functions that a `for` loop can't compete with.


Vectorization examples
===
incremental: true

* `rowSums`, `colSums`, `rowMeans`, `colMeans` give sums or averages over rows or columns of matrices/data frames


```r
(a_matrix <- matrix(1:12, nrow = 3, ncol = 4))
```

```
     [,1] [,2] [,3] [,4]
[1,]    1    4    7   10
[2,]    2    5    8   11
[3,]    3    6    9   12
```

```r
rowSums(a_matrix)
```

```
[1] 22 26 30
```


Back to Goofus example from last week
===
incremental: true

Remember when Goofus tried find the mean for each variable in the `swiss` data? The most Gallant solution is to just use `colMeans` without even thinking about pre-allocation or `for` loops:

```r
colMeans(swiss)
```

```
       Fertility      Agriculture      Examination        Education 
        70.14255         50.65957         16.48936         10.97872 
        Catholic Infant.Mortality 
        41.14383         19.94255 
```


More examples of vectorized functions
===
incremental: true

* `cumsum`, `cumprod`, `cummin`, `cummax` give back a vector with cumulative quantities (e.g. running totals)

```r
cumsum(1:7)
```

```
[1]  1  3  6 10 15 21 28
```

* `pmax` and `pmin` take a matrix or set of vectors, output the min or max for each **p**osition (after recycling):

```r
pmax(c(0, 2, 4), c(1, 1, 1), c(2, 2, 2))
```

```
[1] 2 2 4
```


Lab exercise: running totals
===

1. Use `rnorm` to randomly generate a vector of length 10 million

2. Using a `for` loop and timing your code, calculate another vector of length 10 million that gives the running total

3. Repeat using `cumsum` instead of a loop

Did you get the same result both ways? What's the difference in speed? Which is easier to read?


Writing your own functions
===
type: section


Examples of existing functions
===
incremental: true

* `mean`:
    + Input: a vector
    + Output: a single number
* `dplyr::filter`:
    + Input: a data frame, logical conditions
    + Output: a data frame with rows removed using those conditions
* `readr::read_csv`:
    + Input: file path, optionally variable names or types
    + Output: data frame containing info read in from file


Why write your own functions?
===
incremental: true

Functions can encapsulate repeated actions such as:

* Given a vector, compute some special summary stats
* Given a vector and definition of "gross" values, replace with `NA`
* Templates for favorite `ggplot`s used in reports

Advanced uses for functions (not covered in this class):

* Parallel processing
* Making custom packages containing your functions


Simple homebrewed function
===
incremental: true

Let's look at a function that takes a vector as input and outputs a named vector of the first and last elements:


```r
first_and_last <- function(x) {
    first <- x[1]
    last <- x[length(x)]
    return(c("first" = first, "last" = last))
}
```

Test it out:


```r
first_and_last(c(4, 3, 1, 8))
```

```
first  last 
    4     8 
```


More testing of simple function
===
incremental: true

What if I give `first_and_last` a vector of length 1?


```r
first_and_last(7)
```

```
first  last 
    7     7 
```

Of length 0?


```r
first_and_last(numeric(0))
```

```
first 
   NA 
```

Maybe we want it to be a little smarter.


Checking inputs
===
incremental: true

Let's make sure we get an error message is the vector is too small:


```r
smarter_first_and_last <- function(x) {
    if(length(x) == 0L) { # specify integers with L
        stop("The input has no length!")
    } else {
        first <- x[1]
        last <- x[length(x)]
        return(c("first" = first, "last" = last))        
    }
}
```


Testing the smarter function
===
incremental: true


```r
smarter_first_and_last(numeric(0))
```

```
Error in smarter_first_and_last(numeric(0)): The input has no length!
```

```r
smarter_first_and_last(c(4, 3, 1, 8))
```

```
first  last 
    4     8 
```


Cracking open functions
===
incremental: true

If you type the function name without any parentheses or arguments, you can see its guts:


```r
smarter_first_and_last
```

```
function(x) {
    if(length(x) == 0L) { # specify integers with L
        stop("The input has no length!")
    } else {
        first <- x[1]
        last <- x[length(x)]
        return(c("first" = first, "last" = last))        
    }
}
```


Anatomy of a function
===
incremental: true

* Name: what you assign the function to so you can use it later
    + Can have "anonymous" (no-name) functions
* Arguments (aka inputs, parameters): things the user passes to the function that affects how it works
    + e.g. `x` or `na.rm` in `my_new_func <- function(x, na.rm = FALSE) {...}`
    + `na.rm = FALSE` is example of setting a default value: if user doesn't say what `na.rm` is, it'll be `FALSE`
    + `x`, `na.rm` values won't exist in R outside of the function
* Body: the guts!
* Return value: the output thing inside `return()`. Could be a vector, list, data frame, another function, or even nothing
    + If unspecified, will be the last thing calculated (maybe not what you want?)
    

Example: reporting quantiles
===
incremental: true

Maybe you want to know more detailed quantile information than `summary` gives you with interpretable names. Here's a starting point:


```r
quantile_report <- function(x, na.rm = FALSE) {
    quants <- quantile(x, probs = c(0.01, 0.05, 0.10, 0.25, 0.5, 0.75, 0.90, 0.95, 0.99), na.rm = na.rm)
    names(quants) <- c("Bottom 1%", "Bottom 5%", "Bottom 10%", "Bottom 25%", "Median", "Top 25%", "Top 10%", "Top 5%", "Top 1%")
    return(quants)
}
quantile_report(rnorm(10000))
```

```
  Bottom 1%   Bottom 5%  Bottom 10%  Bottom 25%      Median     Top 25% 
-2.25847948 -1.64396367 -1.26526868 -0.67515473  0.00667206  0.69270116 
    Top 10%      Top 5%      Top 1% 
 1.30913774  1.65679599  2.25174944 
```


lapply: list + applying functions
===

`lapply` **apply**s a function over a **l**ist of any kind (e.g. a data frame), and returns a list. This is a lot easier than preparing a `for` loop!


```r
lapply(swiss, FUN = quantile_report)
```

```
$Fertility
 Bottom 1%  Bottom 5% Bottom 10% Bottom 25%     Median    Top 25% 
    38.588     47.580     56.240     64.700     70.400     78.450 
   Top 10%     Top 5%     Top 1% 
    84.600     90.670     92.454 

$Agriculture
 Bottom 1%  Bottom 5% Bottom 10% Bottom 25%     Median    Top 25% 
     4.190     15.650     17.360     35.900     54.100     67.650 
   Top 10%     Top 5%     Top 1% 
    76.820     84.810     87.952 

$Examination
 Bottom 1%  Bottom 5% Bottom 10% Bottom 25%     Median    Top 25% 
      3.00       5.00       6.00      12.00      16.00      22.00 
   Top 10%     Top 5%     Top 1% 
     26.00      30.40      36.08 

$Education
 Bottom 1%  Bottom 5% Bottom 10% Bottom 25%     Median    Top 25% 
      1.46       2.00       3.00       6.00       8.00      12.00 
   Top 10%     Top 5%     Top 1% 
     23.20      29.00      43.34 

$Catholic
 Bottom 1%  Bottom 5% Bottom 10% Bottom 25%     Median    Top 25% 
    2.2052     2.4480     2.8320     5.1950    15.1400    93.1250 
   Top 10%     Top 5%     Top 1% 
   99.0000    99.6140    99.8666 

$Infant.Mortality
 Bottom 1%  Bottom 5% Bottom 10% Bottom 25%     Median    Top 25% 
    12.778     15.600     16.420     18.150     20.000     21.700 
   Top 10%     Top 5%     Top 1% 
    23.680     24.470     25.818 
```


More usable lapply output with sapply
===

A downside to `lapply` is that lists are not always natural to work with. `sapply` will **s**implify the list output by converting each list element to a column in a matrix:


```r
sapply(swiss, FUN = quantile_report)
```

```
           Fertility Agriculture Examination Education Catholic
Bottom 1%     38.588       4.190        3.00      1.46   2.2052
Bottom 5%     47.580      15.650        5.00      2.00   2.4480
Bottom 10%    56.240      17.360        6.00      3.00   2.8320
Bottom 25%    64.700      35.900       12.00      6.00   5.1950
Median        70.400      54.100       16.00      8.00  15.1400
Top 25%       78.450      67.650       22.00     12.00  93.1250
Top 10%       84.600      76.820       26.00     23.20  99.0000
Top 5%        90.670      84.810       30.40     29.00  99.6140
Top 1%        92.454      87.952       36.08     43.34  99.8666
           Infant.Mortality
Bottom 1%            12.778
Bottom 5%            15.600
Bottom 10%           16.420
Bottom 25%           18.150
Median               20.000
Top 25%              21.700
Top 10%              23.680
Top 5%               24.470
Top 1%               25.818
```


apply
===
incremental: true

There's also a function just called `apply` that works over matrices or data frames. You tell it whether to apply the function to each row (`MARGIN = 1`) or column (`MARGIN = 2`).


```r
apply(swiss, MARGIN = 2, FUN = quantile_report)
```

```
           Fertility Agriculture Examination Education Catholic
Bottom 1%     38.588       4.190        3.00      1.46   2.2052
Bottom 5%     47.580      15.650        5.00      2.00   2.4480
Bottom 10%    56.240      17.360        6.00      3.00   2.8320
Bottom 25%    64.700      35.900       12.00      6.00   5.1950
Median        70.400      54.100       16.00      8.00  15.1400
Top 25%       78.450      67.650       22.00     12.00  93.1250
Top 10%       84.600      76.820       26.00     23.20  99.0000
Top 5%        90.670      84.810       30.40     29.00  99.6140
Top 1%        92.454      87.952       36.08     43.34  99.8666
           Infant.Mortality
Bottom 1%            12.778
Bottom 5%            15.600
Bottom 10%           16.420
Bottom 25%           18.150
Median               20.000
Top 25%              21.700
Top 10%              23.680
Top 5%               24.470
Top 1%               25.818
```


Example: discretizing continuous data
===
type: incremental

Maybe you often want to bucket variables in your data into groups based on quantiles:

| Person | Income | Income Bucket |
|:------:|-------:|--------------:|
|    1   |   8000 |             1 |
|    2   | 103000 |             3 |
|    3   |  12000 |             1 |
|    4   |  52000 |             2 |
|    5   | 150000 |             3 |
|    6   |  45000 |             2 |


Bucketing function
===
incremental: true

There's already a function in R called `cut` that does this, but you need to tell it cutpoints or the number of buckets. Let's make a convenience function that calls `cut` using quantiles for splitting and returns an integer:


```r
bucket <- function(x, quants = c(0.333, 0.667)) {
    # set low extreme, quantile points, high extreme
    new_breaks <- c(min(x)-1, quantile(x, probs = quants), max(x)+1)
    # labels = FALSE will return integer codes instead of ranges
    return(cut(x, breaks = new_breaks, labels = FALSE))
}
```

Trying out the bucket
===


```r
rando_data <- rnorm(100)
bucketed_rando_data <- bucket(rando_data, quants = c(0.05, 0.25, 0.5, 0.75, 0.95))
plot(x = bucketed_rando_data, y = rando_data,
     main = "Buckets and values")
```

<img src="slides_week_7-figure/unnamed-chunk-20-1.png" title="plot of chunk unnamed-chunk-20" alt="plot of chunk unnamed-chunk-20" width="1100px" height="330px" />


Example: removing bad data values
===
type: incremental

Let's say we have data where impossible values occur:


```r
(school_data <- data.frame(school = letters[1:10], pct_passing_exam = c(0.78, 0.55, 0.91, -1, 0.88, 0.81, 0.90, 0.76, 999, 999), pct_free_lunch = c(0.33, 999, 0.25, 0.05, 0.12, 0.09, 0.22, -13, 0.21, 999)))
```

```
   school pct_passing_exam pct_free_lunch
1       a             0.78           0.33
2       b             0.55         999.00
3       c             0.91           0.25
4       d            -1.00           0.05
5       e             0.88           0.12
6       f             0.81           0.09
7       g             0.90           0.22
8       h             0.76         -13.00
9       i           999.00           0.21
10      j           999.00         999.00
```


Function to remove extreme values
===
incremental: true

* Input: a vector `x`, cutoff for `low`, cutoff for `high`
* Output: a vector with `NA` in the extreme places


```r
remove_extremes <- function(x, low, high) {
    x_no_low <- ifelse(x < low, NA, x)
    x_no_low_no_high <- ifelse(x_no_low > high, NA, x)
    return(x_no_low_no_high)
}
remove_extremes(school_data$pct_passing_exam, low = 0, high = 1)
```

```
 [1] 0.78 0.55 0.91   NA 0.88 0.81 0.90 0.76   NA   NA
```


dplyr::mutate_each, summarize_each
===

`dplyr` functions `summarize_each` and `mutate_each` take an argument `funs()`. This will apply our function to every variable (besides `school`) to update the columns in `school_data`:


```r
library(dplyr)
school_data %>%
    mutate_each(funs(remove_extremes(x = ., low = 0, high = 1)),
                -school)
```

```
   school pct_passing_exam pct_free_lunch
1       a             0.78           0.33
2       b             0.55             NA
3       c             0.91           0.25
4       d               NA           0.05
5       e             0.88           0.12
6       f             0.81           0.09
7       g             0.90           0.22
8       h             0.76             NA
9       i               NA           0.21
10      j               NA             NA
```


mutate_each in data downloading demo
===

In the data downloading demo, I had this block of code to fix ID values that were mangled like `"12345678.000000"`:


```r
CA_OSHPD_util <- CA_OSHPD_util %>%
    # translating the regular expression pattern:
    # \\. matches the location of the period.
    # 0+ matches at least one zero and possibly more following that period.
    # replacement for period + 0s is nothing (empty string)
    mutate_each(funs(gsub(pattern = "\\.0+",
                          x = .,
                          replacement = "")),
                # variables to fix
                FAC_NO, HSA, HFPA)
```


Standard and non-standard evaluation
===

`dplyr` uses what is called **non-standard evaluation** that lets you refer to "naked" variables (no quotes around them) like `FAC_NO, HSA, HFPA`. There are **standard evaluation** versions of `dplyr` functions that use the quoted versions instead, which can sometimes be more convenient. These end in an underscore (`_`).

Example converting character data to dates from the data downloading demo:

```r
yearly_frames[[i]] <- yearly_frames[[i]] %>%
    # cnames is a character vector of var names
    # 4th and 5th variables are strings to become dates
    mutate_each_(funs(mdy), cnames[4:5])
```

Anonymous functions in dplyr
===

You can skip naming your function in `dplyr` if you won't use it again. Code below will return the mean divided by the standard deviation for each variable in `swiss`:


```r
swiss %>%
    summarize_each(funs( mean(., na.rm = TRUE) / sd(., na.rm = TRUE) ))
```

```
  Fertility Agriculture Examination Education  Catholic Infant.Mortality
1  5.615134    2.230597    2.066884  1.141785 0.9865478         6.846766
```


Anonymous functions in lapply
===

Like with `dplyr`, you can use anonymous functions in `lapply`, but a difference is you'll need to have the `function` part at the beginning:


```r
lapply(swiss, function(x) mean(x, na.rm = TRUE) / sd(x, na.rm = TRUE))
```

```
$Fertility
[1] 5.615134

$Agriculture
[1] 2.230597

$Examination
[1] 2.066884

$Education
[1] 1.141785

$Catholic
[1] 0.9865478

$Infant.Mortality
[1] 6.846766
```


Example: ggplot templates
===

Let's say you have a particular way you like your charts:


```r
library(gapminder); library(ggplot2)
ggplot(gapminder %>% filter(country == "Afghanistan"),
       aes(x = year, y = pop / 1000000)) +
    geom_line(color = "firebrick") +
    xlab(NULL) + ylab("Population (millions)") +
    ggtitle("Population of Afghanistan since 1952") +
    theme_minimal() + theme(text = element_text(family = "Times"), plot.title = element_text(hjust = 0, size = 20))
```

* How could we make this flexible for any country?
* How could we make this flexible for any `gapminder` variable?

Example of desired chart
===

<img src="slides_week_7-figure/unnamed-chunk-29-1.png" title="plot of chunk unnamed-chunk-29" alt="plot of chunk unnamed-chunk-29" width="1100px" height="660px" />

Another example of desired chart
===

<img src="slides_week_7-figure/unnamed-chunk-30-1.png" title="plot of chunk unnamed-chunk-30" alt="plot of chunk unnamed-chunk-30" width="1100px" height="660px" />


Making country flexible
===

We can have the user input a character string for `cntry` as an argument to the function to get subsetting and the title right:

```r
gapminder_lifeplot <- function(cntry) {
    ggplot(gapminder %>% filter(country == cntry),
       aes(x = year, y = lifeExp)) +
    geom_line(color = "firebrick") +
    xlab(NULL) + ylab("Life expectancy") +
    ggtitle(paste0("Life expectancy in ", cntry, " since 1952")) +
    theme_minimal() + theme(text = element_text(family = "Times"), plot.title = element_text(hjust = 0, size = 20))
}
```
 

Testing out life expectancy plot function
===


```r
gapminder_lifeplot(cntry = "Turkey")
```

<img src="slides_week_7-figure/unnamed-chunk-32-1.png" title="plot of chunk unnamed-chunk-32" alt="plot of chunk unnamed-chunk-32" width="1100px" height="440px" />

Making y value flexible
===

Now let's allow the user to say which variable they want plotted on the y-axis. First, let's think about how we can get the right labels for the axis and title. A named character vector to serve as a "lookup table" inside the function would work:


```r
y_axis_label <- c("lifeExp" = "Life expectancy",
                  "pop" = "Population (millions)",
                  "gdpPercap" = "GDP per capita, USD")
title_text <- c("lifeExp" = "Life expectancy in ",
                "pop" = "Population of ",
                "gdpPercap" = "GDP per capita in ")
# example use:
y_axis_label["pop"]
```

```
                    pop 
"Population (millions)" 
```

aes_string
===

`ggplot` is usually looking for "naked" variables, but we can tell it to take them as quoted strings using `aes_string` instead of `aes`, which is handy when making functions:

```r
gapminder_plot <- function(cntry, yvar) {
    y_axis_label <- c("lifeExp" = "Life expectancy", "pop" = "Population (millions)", "gdpPercap" = "GDP per capita, USD")[yvar]
    title_text <- c("lifeExp" = "Life expectancy in ", "pop" = "Population of ", "gdpPercap" = "GDP per capita in ")[yvar]
    ggplot(gapminder %>% filter(country == cntry) %>% mutate(pop = pop / 1000000), aes_string(x = "year", y = yvar)) + geom_line(color = "firebrick") + xlab(NULL) + ylab(y_axis_label) + ggtitle(paste0(title_text, cntry, " since 1952")) + theme_minimal() + theme(text = element_text(family = "Times"), plot.title = element_text(hjust = 0, size = 20))
}
```


Testing out the gapminder plot function
===


```r
gapminder_plot(cntry = "Turkey", yvar = "pop")
```

<img src="slides_week_7-figure/unnamed-chunk-35-1.png" title="plot of chunk unnamed-chunk-35" alt="plot of chunk unnamed-chunk-35" width="1100px" height="440px" />


debugging
===

Something not working as hoped? Try using `debug` on a function, which will show you the world as perceived from inside the function:


```r
debug(gapminder_plot)
```

Then when you've fixed your problem, use `undebug` so that you won't go into debug mode every time you run it:


```r
undebug(gapminder_plot)
```



Homework
===
type: section


Homework
===

DUE IN TWO WEEKS
