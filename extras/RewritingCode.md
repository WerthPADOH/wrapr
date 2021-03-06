Let X=X in R
================
John Mount, Win-Vector LLC
2017-11-03

Our article ["Let’s Have Some Sympathy For The Part-time R User"](http://www.win-vector.com/blog/2017/08/lets-have-some-sympathy-for-the-part-time-r-user/) includes two points:

-   Sometimes you have to write parameterized or re-usable code.
-   The methods for doing this should be easy and legible.

The first point feels abstract, until you find yourself wanting to re-use code on new projects. As for the second point: I feel the [`wrapr`](https://winvector.github.io/wrapr/) package is the easiest, safest, most consistent, and most legible way to achieve maintainable code re-use in [`R`](https://cran.r-project.org).

In this article we will show how `wrapr` makes code-rewriting even easier with its new `let x=x` automation.

![Let X=X](411gJqs4qlL.jpg)

There are *very* important reasons to choose a package that makes things easier. One is debugging:

> Everyone knows that debugging is twice as hard as writing a program in the first place. So if you're as clever as you can be when you write it, how will you ever debug it?
> <p/>
> [Brian Kernighan, *The Elements of Programming Style*, 2nd edition, chapter 2](https://en.wikiquote.org/wiki/Brian_Kernighan)

Let's take the monster example from ["Let’s Have Some Sympathy For The Part-time R User"](http://www.win-vector.com/blog/2017/08/lets-have-some-sympathy-for-the-part-time-r-user/).

The idea was that perhaps one had worked out a complicated (but useful and important) by-hand survey scoring method:

``` r
suppressPackageStartupMessages(library("dplyr"))
library("wrapr")

d <- data.frame(
  subjectID = c(1,                   
                1,
                2,                   
                2),
  surveyCategory = c(
    'withdrawal behavior',
    'positive re-framing',
    'withdrawal behavior',
    'positive re-framing'
  ),
  assessmentTotal = c(5,                 
                      2,
                      3,                  
                      4),
  stringsAsFactors = FALSE
)

scale <- 0.237

d %>%
  group_by(subjectID) %>%
  mutate(probability =
           exp(assessmentTotal * scale)/
           sum(exp(assessmentTotal * scale))) %>%
  arrange(probability, surveyCategory) %>%
  mutate(isDiagnosis = row_number() == n()) %>%
  filter(isDiagnosis) %>%
  ungroup() %>%
  select(subjectID, surveyCategory, probability) %>%
  rename(diagnosis = surveyCategory) %>%
  arrange(subjectID)
```

    ## # A tibble: 2 x 3
    ##   subjectID           diagnosis probability
    ##       <dbl>               <chr>       <dbl>
    ## 1         1 withdrawal behavior   0.6706221
    ## 2         2 positive re-framing   0.5589742

The presumption is that the above pipeline is considered reasonable (but long, complicated, and valuable) `dplyr`, and our goal is to re-use it on new data that may not have the same column names as our original data.

We are making the huge simplifying assumption that you have studied the article and the above example is now familiar.

The question is: what to do when one wants to process the same type of data with different column names? For example:

``` r
d <- data.frame(
  PID = c(1,                   
          1,
          2,                   
          2),
  DIAG = c(
    'withdrawal behavior',
    'positive re-framing',
    'withdrawal behavior',
    'positive re-framing'
  ),
  AT = c(5,                 
         2,
         3,                  
         4),
  stringsAsFactors = FALSE
)

print(d)
```

    ##   PID                DIAG AT
    ## 1   1 withdrawal behavior  5
    ## 2   1 positive re-framing  2
    ## 3   2 withdrawal behavior  3
    ## 4   2 positive re-framing  4

The new table has the following new column definitions:

``` r
subjectID       <- "PID"
surveyCategory  <- "DIAG"
assessmentTotal <- "AT"
isDiagnosis     <- "isD"
probability     <- "prob"
diagnosis       <- "label"
```

We could "reduce to a previously solved problem" by renaming the columns to names we know, doing the work, and then renaming back (which is actually a service that [`replyr::replyr_apply_f_mapped()`](https://winvector.github.io/replyr/reference/replyr_apply_f_mapped.html) supplies).

In ["Let’s Have Some Sympathy For The Part-time R User"](http://www.win-vector.com/blog/2017/08/lets-have-some-sympathy-for-the-part-time-r-user/) I advised editing the pipeline to have obvious stand-in names (perhaps in all-capitals) and then using [`wrapr::let()`](https://winvector.github.io/wrapr/reference/let.html) to perform symbol substitution on the pipeline.

[Dr. Nina Zumel](https://ninazumel.com) has since pointed out to me: if you truly trust the substitution method you can use the original column names and adapt the original calculation pipeline as is (without alteration). Let's try that:

``` r
let(
  c(subjectID = subjectID,
    surveyCategory = surveyCategory, 
    assessmentTotal = assessmentTotal,
    isDiagnosis = isDiagnosis,
    probability = probability,
    diagnosis = diagnosis),
  d %>%
    group_by(subjectID) %>%
    mutate(probability =
             exp(assessmentTotal * scale)/
             sum(exp(assessmentTotal * scale))) %>%
    arrange(probability, surveyCategory) %>%
    mutate(isDiagnosis = row_number() == n()) %>%
    filter(isDiagnosis) %>%
    ungroup() %>%
    select(subjectID, surveyCategory, probability) %>%
    rename(diagnosis = surveyCategory) %>%
    arrange(subjectID))
```

    ## # A tibble: 2 x 3
    ##     PID               label      prob
    ##   <dbl>               <chr>     <dbl>
    ## 1     1 withdrawal behavior 0.6706221
    ## 2     2 positive re-framing 0.5589742

That works! All we did was: paste the original code into the block and the adapter did all of the work, with no user edits of the code.

It is a bit harder for the user to find which symbols are being replaced, but in some sense they don't really need to know (it is `R`'s job to perform the replacements).

`wrapr` has a new helper function [`mapsyms()`](https://winvector.github.io/wrapr/reference/mapsyms.html) that automates all of the "`let x = x`" steps from the above example.

<iframe width="560" height="315" src="https://www.youtube.com/embed/r4_qdFyVnv8?start=31" frameborder="0" allowfullscreen>
</iframe>
`mapsyms()` is a simple function that captures variable names and builds a mapping from them to the names they refer to in the current environment. For example we can use it to quickly build the assignment map for the let block, because the earlier assignments such as "`subjectID <- "PID"`" allow `mapsyms()` to find the intended re-mappings. This would also be true for other cases, such as re-mapping function arguments to values. Our example becomes:

``` r
print(mapsyms(subjectID,
              surveyCategory, 
              assessmentTotal,
              isDiagnosis,
              probability,
              diagnosis))
```

    ## $subjectID
    ## [1] "PID"
    ## 
    ## $surveyCategory
    ## [1] "DIAG"
    ## 
    ## $assessmentTotal
    ## [1] "AT"
    ## 
    ## $isDiagnosis
    ## [1] "isD"
    ## 
    ## $probability
    ## [1] "prob"
    ## 
    ## $diagnosis
    ## [1] "label"

This allows the solution to be re-written and even wrapped into a function in a very legible form with very little effort:

``` r
computeRes <- function(d,
                       subjectID,
                       surveyCategory, 
                       assessmentTotal,
                       isDiagnosis,
                       probability,
                       diagnosis) {
  let(
    mapsyms(subjectID,
            surveyCategory, 
            assessmentTotal,
            isDiagnosis,
            probability,
            diagnosis),
    d %>%
      group_by(subjectID) %>%
      mutate(probability =
               exp(assessmentTotal * scale)/
               sum(exp(assessmentTotal * scale))) %>%
      arrange(probability, surveyCategory) %>%
      mutate(isDiagnosis = row_number() == n()) %>%
      filter(isDiagnosis) %>%
      ungroup() %>%
      select(subjectID, surveyCategory, probability) %>%
      rename(diagnosis = surveyCategory) %>%
      arrange(subjectID)
  )
}

computeRes(d,
           subjectID       = "PID",
           surveyCategory  = "DIAG",
           assessmentTotal = "AT",
           isDiagnosis     = "isD",
           probability     = "prob",
           diagnosis       = "label")
```

    ## # A tibble: 2 x 3
    ##     PID               label      prob
    ##   <dbl>               <chr>     <dbl>
    ## 1     1 withdrawal behavior 0.6706221
    ## 2     2 positive re-framing 0.5589742

The idea is: instead of having to mark what instances of symbols are to be replaced (by quoting or de-quoting indicators), we instead declare what symbols are to be replaced using the `mapsyms()` helper.

[`mapsyms()`](https://winvector.github.io/wrapr/reference/mapsyms.html) is a stand-alone helper function (just as "[`:=`](https://winvector.github.io/wrapr/reference/named_map_builder.html)" is). It works *not because* it is some exceptional corner-case hard-wired into other functions, but because `mapsyms()`'s reasonable semantics happen to synergize with `let()`'s reasonable semantics. `mapsyms()` behaves as a replacement target controller (without needing any cumbersome direct quoting or un-quoting notation!).
