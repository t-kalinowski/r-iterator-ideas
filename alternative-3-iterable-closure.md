
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Stateful iterator functions

This approach is centered around the idea of a stateful function that
you call repeatedly to get the next value. This is similar to Python’s
generators (where you repeatedly call `nextElem(it)`), but in a
functional programming language it feels more natural to return a
function that you call repeatedly.

The implementation begins with a new `as.iterator()` generic:

``` r
as.iterator <- function(x) UseMethod("as.iterator")
```

We then provide a default method that mimics the existing behavior of
`for`:

``` r
as.iterator.default <- function(x) {
  i <- 1L
  function() {
    if (i > length(unclass(x))) {
      IteratorExhausted
    } else {
      on.exit(i <<- i + 1L)
      .subset2(x, i)
    }
  }
}

IteratorExhausted <- new.env(parent = emptyenv())
```

(Like approach 2, this could instead signal a condition. However, a
sentinel object is a little easier to work with and avoids the (small)
overhead of setting up a condition handler. Unlike approach two, which
overloads an existing function, there’s no need for backward
compatiblity forcing us to use a signal here.)

And a method so that users can easily create new iterators with a
function:

``` r
as.iterator.function <- identity
```

Then `for` creates an iterator and repeatedly calls it until it’s
exhausted:

``` r
`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  iterator <- as.iterator(iterable)
  repeat {
    value <- iterator()
    if (identical(value, IteratorExhausted))
      return(invisible())
    
    assign(x = var, value = value, envir = env)
    eval(body, env)
  }
}

for (x in 1:5) {
  str(x)
}
#>  int 1
#>  int 2
#>  int 3
#>  int 4
#>  int 5
```

## Examples

### Calculation on demand

Unlike the other approaches, there’s no need to create an S3 class. We
just need a function factory:

``` r
SquaresSequence <- function(from = 1, to = 10) {
  force(from); force(to)
  function() {
    if(from > to) {
      IteratorExhausted
    } else {
      on.exit(from <<- from + 1L)
      from ^ 2
    }
  }
}

for (x in SquaresSequence(5, 10))
  print(x)
#> [1] 25
#> [1] 36
#> [1] 49
#> [1] 64
#> [1] 81
#> [1] 100
```

### Sequence of unknown length

``` r
SampleSequence <- function(max) {
  force(max)
  sum <- 0
  
  function() {
    if (sum > max) {
      IteratorExhausted
    } else {
      val <- abs(rnorm(1))
      sum <<- sum + val
      val
    }
  }
}

for (x in SampleSequence(2)) {
  print(x)
}
#> [1] 1.400044
#> [1] 0.2553171
#> [1] 2.437264
```

### Reticulate

``` r
as.iterator.python.builtin.object <- function(x) {
  iterator <- reticulate::as_iterator(x)
  function() {
    reticulate::iter_next(iterator, completed = IteratorExhausted)
  }
}

for(x in reticulate::r_to_py(1:3))
  print(x)
#> 1
#> 2
#> 3
```

### POSIXt

We can give POSIXt (and other S3 classes) more reasonable behavior by
selectively overriding the default method to use `length(x)` instead of
`length(unclass(x))` and `[[` instead of `.subset2()`.

``` r
as.iterator.POSIXt <- function(x) {
  i <- 1L
  function() {
    if (i > length(x)) {
      IteratorExhausted
    } else {
      on.exit(i <<- i + 1L)
      x[[i]]
    }
  }
}

dt <- .POSIXct(c(1, 2), tz = "UTC")
for (x in dt) {
  str(x)
}
#>  POSIXct[1:1], format: "1970-01-01 00:00:01"
#>  POSIXct[1:1], format: "1970-01-01 00:00:02"
for (x in as.POSIXlt(dt)) {
  str(x)
}
#>  POSIXlt[1:1], format: "1970-01-01 00:00:01"
#>  POSIXlt[1:1], format: "1970-01-01 00:00:02"
```
