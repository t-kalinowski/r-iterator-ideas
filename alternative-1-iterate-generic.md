
<!-- README.md is generated from README.Rmd. Please edit that file -->

# `iterate()` generic with explicit state

Inspired by
[Julia](https://docs.julialang.org/en/v1/manual/interfaces/#man-interface-iteration),
this approach proposes a new generic:

``` r
iterate <- function(x, state = NULL) UseMethod("iterate")
```

`iterate()` returns `NULL` if the iteration is complete or otherwise
returns a list of length 2:
`list(value = cur_value, state = next_state)`.

Then `for` can be implemented as:

``` r
`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  step <- list(value = NULL, state = NULL)
  repeat {
    # In a real implementation, would likely cache the method to avoid 
    # S3 dispatch on every iteration
    step <- iterate(iterable, step$state)
    if(is.null(step)) return(invisible())

    assign(x = var, value = step$value, envir = env)
    eval(body, env)
  }
}
```

We can provide a `default` method that tries to faithfully match the
current behavior of `for`. This ensures that there’s no change in
behaivour unless an `iterate()` method is explicitly defined.

``` r
iterate.default <- function(x, state = NULL) {

  if(is.null(state)) # start of iteration
    state <- 1L
  if(state > length(unclass(x))) # end of iteration
    return(NULL)
  list(
    value = .subset2(x, state), 
    state = state + 1L
  )
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
  list(
    value = (x$from - 1L + state) ^ 2, 
    state = state + 1L
  )
}

for (x in SquaresSequence(5, 10))
  print(x)
#> [1] 25
#> [1] 36
#> [1] 49
#> [1] 64
#> [1] 81
#> [1] 100
#> [1] 121
#> [1] 144
#> [1] 169
#> [1] 196
```

### Sequence of unknown length

``` r
SampleSequence <- function(max) {
  structure(
    list(max = max),
    class = "SampleSequence"
  )
}

iterate.SampleSequence <- function(x, state = NULL) {
  if(is.null(state))
    state <- 0L
  if (state > x$max)
    return(NULL)
  
  val <- abs(rnorm(1))
  list(
    value = val, 
    state = state + val
  )
}

for (x in SampleSequence(2)) {
  print(x)
}
#> [1] 1.400044
#> [1] 0.2553171
#> [1] 2.437264
```

### POSIXt

A simple modification to the default iterator (from `.subset2()` to
`[[`) makes it possible to selectively change the behavior of key S3
classes:

``` r
iterate.POSIXt <- function(x, state = NULL) {
  if(is.null(state)) # start of iteration
    state <- 1L
  if(state > length(x)) # end of iteration
    return(NULL)
  list(
    value = x[[state]], 
    state = state + 1L
  )
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

### `iterate.environment()`

An environment method would allow users to pass environments to `for`.
Environments are mutable so there are some challenges if the environment
changes during implementation. This implementation works by keeping
track of which elements have already been processed.

``` r
iterate.environment <- function(x, state = NULL) {
  if(is.null(state)) {
    seen <- character()
  } else { 
    seen <- state$seen  
  }
  
  left <- setdiff(names(x), seen)
  if (length(left) == 0) {
    NULL
  } else {
    this <- left[[1]]
    list(
      value = x[[this]],
      state = list(seen = c(seen, this))
    )
  }
}

e <- list2env(list(a = 1, b = 2, c = 3), parent = emptyenv())
for (el in e)
  print(el)
#> [1] 3
#> [1] 2
#> [1] 1
```

### Reticulate

``` r
iterate.python.builtin.object <- function(x, state = NULL) {
  if(is.null(state))
    state <- reticulate::as_iterator(x)

  sentinal <- environment()
  value <- reticulate::iter_next(state, completed = sentinal)
  if(identical(value, sentinal))
    NULL
  else
    list(value = value, state = state)
}

for(x in reticulate::r_to_py(1:3))
  print(x)
```

## Iterator functions

A `function` method for `iterate` would make it possible for users to
define iterators as stateful functions:

``` r
iterate.function <- function(x, state = NULL) {
  if (is.null(state)) # start of iteration
    state <- x
  
  value <- state()
  if (identical(value, IteratorExhaustedSentinal))
    NULL
  else
    list(value = value, state = state)
}

IteratorExhaustedSentinal <- new.env(parent = emptyenv())
```

Here we’ve chosen to indicate exhaustion by returning a sentinel; an
alternative approach would be to signal a custom condition.

Then users could write code like this:

``` r
SquaresSequenceIterator <- function(from = 1, to = 10) {
  force(from); force(to)
  function() {
    if(from > to) {
      IteratorExhaustedSentinal
    } else {
      on.exit(from <<- from + 1L)
      from ^ 2
    }
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

It would also be possible to expose the opposite transformation,
converting an object into an iterator:

``` r
as.iterator <- function(x, exhausted = NULL) {
  force(x); state <- NULL
  function() {
    step <- iterate(x, state)
    if(is.null(step))
      return(exhausted)
    state <<- step$state
    step$value
  }
}

it <- as.iterator(SquaresSequence())
it(); it(); it()
#> [1] 1
#> [1] 4
#> [1] 9
```
