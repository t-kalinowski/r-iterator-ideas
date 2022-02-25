
<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Proposal

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. Below is a proposal for what such a
mechanism might look like:

-   `for` gains the ability to accept a function, which it repeatedly
    calls until a sentinal is returned.

-   A new `as.iterator()` generic; methods are expected to return a
    function.

-   User can implement an `as.iterator()` method for their object, which
    is invoked by `for` before it begins iterating.

The equivalent of the following could be added to base R (with core
parts implemented in C):

``` r
as.iterator <- function(x) UseMethod("as.iterator")
as.iterator.function <- identity
as.iterator.default <- function(x) {
  # This default method tries to faithfully match the current behavior of `for`.
  # The intent is that if no `as.iterator()` method for an object is explicitly
  # defined, then there is no change in behavior.
  i <- 1L
  function() {
    if (i > length(unclass(x)))
      return(IteratorExhaustedSentinal)
    on.exit(i <<- i + 1L)
    .subset2(x, i)
  }
}

IteratorExhaustedSentinal <- new.env(parent = emptyenv())

`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  # same guard as in `lapply()`
  if (!is.vector(iterable) || is.object(iterable)) 
    iterable <- as.iterator(iterable)
  
  if (is.function(iterable)) {
    
    repeat {
      value <- iterable()
      if (identical(value, IteratorExhaustedSentinal))
        return(invisible())
      assign(x = var, value = value, envir = env)
      eval(body, env)
    }
    
  } else # not a function, dispatch to the current `for`
    eval.parent(as.call(list(base::`for`, as.name(var), iterable, body)))
  
}
```

With this approach, the users might write code like this:

``` r
SquaresSequence <- function(from = 1, to = 10) {
  out <- list(from = from, to = to)
  class(out) <- "SquaresSequence"
  out
}

as.iterator.SquaresSequence <- function(x) {
  index <- x$from
  function() {
    if(index > x$to)
      return(IteratorExhaustedSentinal)
    on.exit(index <<- index + 1)
    (x$from - 1L + index) ^ 2
  }
}

for (x in SquaresSequence(1, 10))
  print(x)
#> [1] 1
#> [1] 4
#> [1] 9
#> [1] 16
#> [1] 25
#> [1] 36
#> [1] 49
#> [1] 64
#> [1] 81
#> [1] 100
```

Reticulate support might look like:

``` r
as.iterator.python.builtin.object <- function(x) {
  iterator <- reticulate::as_iterator(x)
  function()
    reticulate::iter_next(iterator, completed = IteratorExhaustedSentinal)
}

for(x in reticulate::r_to_py(1:3))
  print(x)
#> 1
#> 2
#> 3
```

Then users could write also write code like this, where they don’t
define any generic methods and just define a function that could be
passed directly to `for`:

``` r
SquaresSequenceClosure <- function(from = 1, to = 10) {
  force(from); force(to)
  function() {
    if(from > to)
      return(IteratorExhaustedSentinal)
    on.exit(from <<- from + 1L)
    from ^ 2
  }
}

for (x in SquaresSequenceClosure(1, 10))
  print(x)
#> [1] 1
#> [1] 4
#> [1] 9
#> [1] 16
#> [1] 25
#> [1] 36
#> [1] 49
#> [1] 64
#> [1] 81
#> [1] 100
```

### Optional extensions

#### as.iterator.POSIXt

``` r
# current behavior, not desirable
(ct <- .POSIXct(c(1, 2, 3)))
#> [1] "1969-12-31 19:00:01 EST" "1969-12-31 19:00:02 EST"
#> [3] "1969-12-31 19:00:03 EST"
(lt <- as.POSIXlt(ct))
#> [1] "1969-12-31 19:00:01 EST" "1969-12-31 19:00:02 EST"
#> [3] "1969-12-31 19:00:03 EST"

for(x in ct)
  print(x)
#> [1] 1
#> [1] 2
#> [1] 3

for(x in lt)
  print(x)
#> [1] 1 2 3
#> [1] 0 0 0
#> [1] 19 19 19
#> [1] 31 31 31
#> [1] 11 11 11
#> [1] 69 69 69
#> [1] 3 3 3
#> [1] 364 364 364
#> [1] 0 0 0
#> [1] "EST" "EST" "EST"
#> [1] -18000 -18000 -18000
```

``` r
.as.iterator_via_length_and_extract <- function(x) {
  i <- 1
  function() {
    if(i > length(x))
      return(IteratorExhaustedSentinal)
    on.exit(i <<- i + 1)
    x[i]
  }
}

as.iterator.POSIXt <- .as.iterator_via_length_and_extract

# current behavior, not desirable
(ct <- .POSIXct(c(1, 2, 3)))
#> [1] "1969-12-31 19:00:01 EST" "1969-12-31 19:00:02 EST"
#> [3] "1969-12-31 19:00:03 EST"
(lt <- as.POSIXlt(ct))
#> [1] "1969-12-31 19:00:01 EST" "1969-12-31 19:00:02 EST"
#> [3] "1969-12-31 19:00:03 EST"

for(x in ct)
  print(x)
#> [1] "1969-12-31 19:00:01 EST"
#> [1] "1969-12-31 19:00:02 EST"
#> [1] "1969-12-31 19:00:03 EST"

for(x in lt)
  print(x)
#> [1] "1969-12-31 19:00:01 EST"
#> [1] "1969-12-31 19:00:02 EST"
#> [1] "1969-12-31 19:00:03 EST"
```
