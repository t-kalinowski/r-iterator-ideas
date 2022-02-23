
<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Proposal

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. Below is a proposal for what such a
mechanism might look like:

-   R provides an `iterate()` generic.

-   User implements a `iterate()` method for their object, which is
    called with arguments `iterate(x, state)`. `iterate()` is expected
    to return a list of length 2: `list(next_elem, next_state)`.

-   `for` repeatedly calls `iterate(x, state)` until the `next_state`
    returned is `NULL`.

(This implementation is very similar to the Julia approach)

The equivalent of the following could be added to base R (with core
parts implemented in C):

``` r
iterate <- function(x, state = NULL) UseMethod("iterate")

iterate.default <- function(x, state = NULL) {
  if(is.null(state)) # start of iteration
    state <- 1L
  if(state > length(x)) # end of iteration
    return(list(NULL, NULL)) 
  list(x[[state]], state + 1L)
}

`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  state <- NULL
  repeat {
    c(value, state) %<-% iterate(iterable, state)
    if(is.null(state)) return(invisible())
    assign(x = var, value = value, envir = env)
    eval(body, env)
  }
}

envir::import_from(zeallot, `%<-%`) 
# tuple unpacking assignment operator
# just an implementation detail, not part of the proposal
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
    return(list(NULL, NULL))
  list((x$from - 1L + state) ^ 2, state + 1L)
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
    list(NULL, NULL)
  else
    list(value, state)
}

for(x in reticulate::r_to_py(1:3))
  print(x)
#> 1
#> 2
#> 3
```

We could also provide a default `iterate()` method for functions. This
would allow users to pass functions directly to `for`. User defined
functions would be responsible for eventually signaling an
`iterator_exhausted` condition (or running forever).

``` r
iterate.function <- function(x, state = NULL) {
  if(is.null(state))
    state <- x
  value <- NULL
  tryCatch(value <- state(),
           iterator_exhausted = function(e) state <<- NULL)
  list(value, state)
}

iterator_exhausted_condition <- function(call = sys.call()) {
  structure(class = c("iterator_exhausted", "error", "condition"),
            list(message = "iterator exhausted", call = call))
}
```

Then users could write code like this:

``` r
SquaresSequenceClosure <- function(from = 1, to = 10) {
  force(from); force(to)
  function() {
    if(from > to)
      stop(iterator_exhausted_condition())
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
