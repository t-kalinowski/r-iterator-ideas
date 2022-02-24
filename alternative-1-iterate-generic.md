
<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Proposal: `iterate()` generic.

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. Below is a proposal for what such a
mechanism might look like:

-   R provides a generic with signature: `iterate(x, state)`.

-   User implements a `iterate()` method for their object, which is
    called with arguments `iterate(x, state)`, and expected to return a
    list of length 2: `list(next_elem, next_state)`.

-   `for` repeatedly calls `iterate(x, state)` until `NULL` is returned.

(This implementation is very similar to the Julia approach)

The equivalent of the following could be added to base R (with core
parts implemented in C):

``` r
iterate <- function(x, state = NULL) UseMethod("iterate")

iterate.default <- function(x, state = NULL) {
  if(is.null(state)) # start of iteration
    state <- 1L
  if(state > length(x)) # end of iteration
    return(NULL) 
  list(value = x[[state]], state = state + 1L)
}

`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  step <- list(value = NULL, state = NULL)
  repeat {
    step <- iterate(iterable, step$state)
    if(is.null(step)) return(invisible())
    names(step) <- c("value", "state")
    assign(x = var, value = step$value, envir = env)
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

iterate.SquaresSequence <- function(x, state = NULL) {
  if(is.null(state))
    state <- 1L
  if (state > x$to)
    return(NULL)
  list(value = (x$from - 1L + state) ^ 2, 
       state = state + 1L)
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
iterate.python.builtin.object <- function(x, state = NULL) {
  if(is.null(state))
    state <- reticulate::as_iterator(x)

  sentinal <- environment()
  value <- reticulate::iter_next(state, completed = sentinal)
  if(identical(sentinal, value))
    NULL
  else
    list(value, state)
}

for(x in reticulate::r_to_py(1:3))
  print(x)
#> 1
#> 2
#> 3
```

The above is a narrow but flexible change (Only one new symbol would be
introduced, `base::iterate()`).

One minor advantage of this approach over the others explored is that,
because the `state` object is exposed, is will be relatively easy for
users to rewind and replay iterators (for objects where `iterate` is a
pure function).

An optional extension might be for the `base` namespace to also provide
a `iterate()` method for functions. This would allow users to pass
functions directly to `for`.

In this scenario, user defined functions would be responsible for
indicating when an iterator function is exhausted, either by signaling a
condition or returning a sentinal (or running forever). For example,
with a sentinal based approach:

``` r
iterate.function <- function(x, state = NULL) {
  if (is.null(state)) # start of iteration
    state <- x
  
  value <- state()
  if (identical(value, IteratorExhaustedSentinal))
    NULL
  else
    list(value, state)
}

IteratorExhaustedSentinal <- new.env(parent = emptyenv())
```

Then users could write code like this:

``` r
SquaresSequenceIterator <- function(from = 1, to = 10) {
  force(from); force(to)
  function() {
    if(from > to)
      return(IteratorExhaustedSentinal)
    on.exit(from <<- from + 1L)
    from ^ 2
  }
}

for (x in SquaresSequenceIterator(1, 10))
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