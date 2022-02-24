
<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Proposal

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. Below is a proposal for what such a
mechanism might look like:

-   R `for` gains the ability to accept a function, which it repeatedly
    calls until a sentinal is returned.

-   R also introduces an `as.iterator()` generic, which is expected to
    return a function.

-   User can implement an `as.iterator()` method for their object, which
    is invoked by `for` before it begins iterating.

The equivalent of the following could be added to base R (with core
parts implemented in C):

``` r
as.iterator <- function(x) UseMethod("as.iterator")
as.iterator.function <- identity
as.iterator.default <- function(x) {
  i <- 1L
  function() {
    if (i > length(x))
      return(IteratorExhaustedSentinal)
    on.exit(i <<- i + 1L)
    x[[i]]
  }
}

IteratorExhaustedSentinal <- new.env(parent = emptyenv())

`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  if (!is.vector(iterable) || is.object(iterable)) 
    iterable <- as.iterator(iterable)
  
  if (!is.function(iterable))
    return(eval.parent(as.call(list(
      base::`for`, as.name(var), iterable, body))))
  
  repeat {
    value <- iterable() 
    if(identical(value, IteratorExhaustedSentinal))
      return(invisible())
    assign(x = var, value = value, envir = env)
    eval(body, env)
  }
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