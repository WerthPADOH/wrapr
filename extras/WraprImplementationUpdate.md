`wrapr` Implementation Update
=============================

Introduction
------------

The [development version](https://github.com/WinVector/wrapr) of our [`R`](https://cran.r-project.org) helper function `wrapr::let()` has switched from string-based substitution to abstract syntax tree based substitution (AST based subsitution, or language based substitution).

I am looking for some feedback from `wrapr::let()` users already doing substantial work with `wrapr::let()`. If you are already using `wrapr::let()` please test if the current [development version of `wrapr`](https://github.com/WinVector/wrapr) works with your code. If you run into problems: I apologize and please file a GitHub issue.

The substitution modes
----------------------

The development version of `wrapr::let()` now has three substitution implementations:

-   Language substitution (`subsMethod='langsubs'` the new default). In this mode user code is captured as an abstract syntax tree (or parse tree) and substitution is performed only on nodes known to be symbols.
-   String substitution (`subsMethod='stringsubs'`, the CRAN current default). In this mode user code is captured as text and then string replacement on word-boundaries is used to substitute in variable re-mappings.
-   Substitute substitution (`subsMethod='subsubs'`, new to the development version). In this mode substitution is performed by `R`'s own `base::substitute()`.

The semantics of the three methods can be illustrated by showing the effects of substituting the variable name "`y`" for "`X`" in the somewhat complicated block of statements:

``` r
  {
    d <- data.frame("X" = "X", X2 = "XX", d = X*X, .X = X_)
    X <- list(X = d$X, X2 = d$"X", v1 = `X`, v2 = ` X`)
  }
```

This block a lot of different examples and corner-cases.

#### Language substitution (`subsMethod='langsubs'`)

``` r
library("wrapr")

let(
  c(X = 'y'), 
  {
    d <- data.frame("X" = "X", X2 = "XX", d = X*X, .X = X_)
    X <- list(X = d$X, X2 = d$"X", v1 = `X`, v2 = ` X`)
  },
  eval = FALSE, subsMethod = 'langsubs')
```

    ## {
    ##     d <- data.frame(y = "X", X2 = "XX", d = y * y, .X = X_)
    ##     y <- list(y = d$y, X2 = d$y, v1 = y, v2 = ` X`)
    ## }

Notice the substitution replaced all symbol-like uses of "`X`", and only these (including some that were quoted!).

The reason I need more testing on this method is `R` language structures are fairly complicated, so there may be some corner cases we want additional coding for. For example treating `d$"X"` the same as `d$X` is in fact handled by some special-case code.

#### String substitution (`subsMethod='stringsubs'`)

``` r
let(
  c(X = 'y'), 
  {
    d <- data.frame("X" = "X", X2 = "XX", d = X*X, .X = X_)
    X <- list(X = d$X, X2 = d$"X", v1 = `X`, v2 = ` X`)
  },
  eval = FALSE, subsMethod = 'stringsubs')
```

    ## expression({
    ##     d <- data.frame(y = "y", X2 = "XX", d = y * y, .y = X_)
    ##     y <- list(y = d$y, X2 = d$y, v1 = y, v2 = ` y`)
    ## })

Notice string substitution has a few flaws: it went after variable names that appeared to start with a word-boundary (the cases where the variable name started with a dot or a space). Substitution also occurred in some string constants (which as we have seen could be considered a good thing).

These situations are all avoidable as both the code inside the `let`-block and the substitution targets are chosen by the programmer, so they can be chosen to be simple and mutually consisitent. We suggest "`ALL_CAPS`" style substitution targets as they jump out as being macro targets. But, of course, it is better to have stricter control on substitution.

Think of the language substitution implementation as a lower-bound on a perfect implementation (cautious, with a few corner cases to get coverage) and string substitution as an upper bound on a perfect implementation (aggressive, with a few over-reaches).

#### Substitute substitution (`subsMethod='subsubs'`)

``` r
let(c(X = 'y'), 
    {
      d <- data.frame("X" = "X", X2 = "XX", d = X*X, .X = X_)
      X <- list(X = d$X, X2 = d$"X", v1 = `X`, v2 = ` X`)
    },
    eval = FALSE, subsMethod = 'subsubs')
```

    ## {
    ##     d <- data.frame(X = "X", X2 = "XX", d = y * y, .X = X_)
    ##     y <- list(X = d$y, X2 = d$X, v1 = y, v2 = ` X`)
    ## }

Notice `base::substitute()` doesn't re-write left-hand-sides of argument bindings. This is why I originally didn't consider using this implementation. Re-writing left-hand-sides of assignments is critical in expressions such as `dplyr::mutate( RESULTCOL = INPUTCOL + 1)`.

Conclusion
----------

`wrapr::let()` when used prudently is already a safe and powerful tool. That being said it will be improved by switching to abstract syntax tree based substitution. I'd love to do that in the next release, so it would helpful to collect more experience using the current development version prior to that.

Appendix `wrapr::let()` for beginners
-------------------------------------

Obviously `wrapr::let()` is not interesting if you have no idea why you would need it or how to use it.

`wrapr::let()` is designed to solve a single problem when programming in `R`: when in writing code you don't yet know the name of a variable or a `data.frame` column you are going to use. That is: the name of the column will only be available later when your code is run.

For example suppose we have a `data.frame` "`d`" defined as follows:

``` r
d <- data.frame(x = c(15, 30, 40))
print( d )
```

    ##    x
    ## 1 15
    ## 2 30
    ## 3 40

If we know the name of the column we can access it as follows:

``` r
print( d$x )
```

    ## [1] 15 30 40

If we don't know the name of the column (such as would be the case when writing a function) we write code like the following:

``` r
getColumn <- function(df, columnName) {
  df[[columnName]]
}

print( getColumn(d, 'x') )
```

    ## [1] 15 30 40

This works because `R` takes a lot of trouble to supply parametric interfaces for most use cases.

We only run into trouble if code we are trying to work with strongly prefers non-parametric (also called non-standard-evaluation or NSE) interfaces.

A popular package that heavily emphasizes NSE interfaces is `dplyr`. It is the case that `dplyr` supplies its own methods to work around NSE issues ("underbar methods" and `lazyeval` for `dplyr` `0.5.0`, and `rlang`/`tidyeval` for `dplyr` `0.7.0`). However, we will only discuss `wrapr::let()` methodology here.

The issue is: it is common to write statements such as the following in `dplyr`:

``` r
suppressPackageStartupMessages(library("dplyr"))

d %>% mutate( v2 = x + 1 )
```

    ##    x v2
    ## 1 15 16
    ## 2 30 31
    ## 3 40 41

If we were writing the above in a function it would plausible that we would not know the name of the desired result column or the name of the column to add one to. `wrapr::let()` lets us write such code easily:

``` r
library("wrapr")

addOneToColumn <- function(df, 
                           result_column_name, 
                           input_column_name) {
  let(
    c(RESULT_COL = result_column_name,
      INPUT_COL = input_column_name),
    
    df %>% mutate( RESULT_COL = INPUT_COL + 1 )
    
  )
}

d %>% addOneToColumn('v2', 'x')
```

    ##    x v2
    ## 1 15 16
    ## 2 30 31
    ## 3 40 41

Again, writing the function `addOneToColumn()` was the goal. The issue was that in such a function we don't know what column names the user is going to end up supplying. We work around this difficulty with `wrapr::let()`

All the user needs to keep in mind is: `wrapr::let()` takes two primary arguments:

-   The aliases mapping concrete stand-in names (here shown in `ALL_CAPS` style) to the variables holding the actual variable names.
-   The block of code to re-write and execute (which can be a single statement, or a larger block with braces).