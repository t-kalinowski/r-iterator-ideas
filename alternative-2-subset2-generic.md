
<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Ideas (`[[` + signal, optionally with `as.iterable()`)

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. Below is a proposal for what such a
mechanism might look like:

-   User implements `[[`, and optionally `as.iterable()`.

-   User is expected to call `stop(out_of_bounds_condition())` in `[[`
    when the iterable is exhausted.

-   `for` repeatedly calls `x[[i]]` with incrementing `i` until the
    condition is encountered.

The equivalent of the following could be in base R (with core parts
implemented in C):

``` r
as.iterable <- function(x) UseMethod("as.iterable")
as.iterable.default <- identity

`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()

  if (!is.vector(iterable) || is.object(iterable))
    iterable <- as.iterable(iterable)
  
  i <- 1L
  exhausted <- FALSE
  repeat {
    tryCatch({
      value <- iterable[[i]]
    },
    out_of_bounds = function(cond) {
      exhausted <<- TRUE
    },
    error = function(e) {
      # Not everything raises a typed signal yet, so we need to 
      # accept simple errors too
      if (e$message == "subscript out of bounds")
        exhausted <<- TRUE
      else
        stop(e)
    })
    if (exhausted) return(invisible())
    assign(x = var, value = value, envir = env)
    i <- i + 1L
    eval(body, env)
  }
}

out_of_bounds_condition <- function(call = sys.call()) {
  structure(class = c("out_of_bounds", "error", "condition"),
            list(message = "subscript out of bounds", call = call))
}

# support for functions
as.iterable.function <- function(x) {
  # Not strictly necessary, just a guardrail here so we wouldn't have to part
  # with our old friend: "object of type 'closure' is not subsettable"
  times_called <- 0L
  iterator <- function() {
    on.exit(times_called <<- times_called + 1L)
    x()
  }
  class(iterator) <- "iterable_function"
  iterator
}

`[[.iterable_function` <- function(x, i) {
  if(environment(x)$times_called + 1L != i)
    warning("Something is wrong")
  x() # x is expected to eventually call stop(signal_out_of_bounds())
}
```

With this approach, the users might write code like this:

``` r
SquaresSequence <- function(from = 1, to = 10) {
  out <- list(from = from, to = to)
  class(out) <- "SquaresSequence"
  out
}

`[[.SquaresSequence` <- function(x, i) {
  if (i > x$to)
    stop(out_of_bounds_condition())
  (x$from - 1L + i) ^ 2
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


SquaresSequenceClosure <- function(from = 1, to = 10) {
  force(from); force(to)
  function() {
    from <<- from + 1L
    if(from > to)
      stop(out_of_bounds_condition())
    from ^ 2
  }
}

for (x in SquaresSequenceClosure(1, 10))
  print(x)
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

Reticulate support might look like this:

``` r
as.iterable.python.builtin.object <- function(x) {
  iterator <- reticulate::as_iterator(x)
  class(iterator) <- unique(c("reticulate_iterable", class(iterator)))
  iterator
}

`[[.reticulate_iterable` <- function(x, i) {
  sentinal <- environment()
  val <- reticulate::iter_next(x, completed = sentinal)
  if(identical(val, sentinal))
    stop(out_of_bounds_condition())
  val
}

for(x in reticulate::r_to_py(1:3))
  print(x)
#> 1
#> 2
#> 3
```
