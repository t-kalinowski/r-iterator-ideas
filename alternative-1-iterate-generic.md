
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

(This implementation is very similar to the Julia approach).

The equivalent of the following could be added to base R (with core
parts implemented in C):

``` r
iterate <- function(x, state = NULL) UseMethod("iterate")

iterate.default <- function(x, state = NULL) {
  # This default method tries to faithfully match the current behavior of `for`.
  # The intent is that if no `iterate()` method for an object is explicitly
  # defined, then there is no change in behavior.

  if(is.null(state)) # start of iteration
    state <- 1L
  if(state > length(unclass(x))) # end of iteration
    return(NULL)
  list(value = .subset2(x, state), 
       state = state + 1L)
}

`for` <- function(var, iterable, body) {
  var <- as.character(substitute(var))
  body <- substitute(body)
  env <- parent.frame()
  
  step <- list(value = NULL, state = NULL)
  repeat {
    step <- iterate(iterable, step$state)
    # In a real implementation, we would likely cache the method and try to
    # avoid having to S3 dispatch each iteration.
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
  if(identical(value, sentinal))
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
introduced, `base::iterate()`). Since `iterate()` is a new generic, it
would allows for changes in behavior to be made with intention
individually for each class type.

## Optional extensions

### `iterate.function()`

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

### `as.iterator()`

It’s possible to expose an `as.iterator()` user interface, for users
wanting to manually iterate over an object

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

### `iterate.environment()`

This would allow users to pass environemtns to `for`. However, it opens
the thorny question of what is best to do if the environment is modified
in place while being iterated over. This example implementation allows
for symbols to be removed or modified while iteration is ongoing, but
new symbols added will not be included as part of iteration.

``` r
iterate.environment <- function(x, state = NULL) {
  if(is.null(state)) # start of iteration
    state <- list(names = names(x), idx = 1L)

  repeat {
    if (state$idx > length(state$names)) # end of iteration
      return(NULL)
    
    name <- state$names[[state$idx]]
    state$idx <- state$idx + 1L
    
    if(!exists(name, envir = x, inherits = FALSE)) # removed from env
      next
    
    return(list(x[[name]], state))
  }
}
```

``` r
e <- list2env(list(a = 1, b = 2, c = 3), parent = emptyenv())
for (el in e)
  print(el)
#> [1] 3
#> [1] 2
#> [1] 1
```

### `iterate.POSIXt()`

Currently, `for` strips the class of `POSIXct` and returns a bare
numeric, or iterates over the underlying list of `POSIXlt`. Neither of
these seem like desirable or useful behaviors, and it might be good for
R to also provide an `iterate.POSIXt` method that yields elements of the
supplied `POSIX` type.

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
.iterate_via_length_and_extract <- function(x, state = NULL) {
  if(is.null(state))
    state <- 1L
  if(state > length(x))
    NULL
  else
    list(x[state], state + 1L)
}

iterate.POSIXt <- .iterate_via_length_and_extract

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

### `iterate.factor()`

`for` currently coerces `factor` objects to a character vector. This
seems not terrible (at least it’s not the underlying integer), though it
might be preferable to yield a length-1 factor with a levels attribute.

``` r
base::`for`(x, factor(letters[1:3]), 
  print(x))
#> [1] "a"
#> [1] "b"
#> [1] "c"

iterate.factor <- .iterate_via_length_and_extract

for(x in factor(letters[1:3]))
  print(x)
#> [1] a
#> Levels: a b c
#> [1] b
#> Levels: a b c
#> [1] c
#> Levels: a b c
```

### `iterate.package_version()`

Again, attributes are stripped, and this seems undesirable.

``` r
for(x in package_version(c("1.1.1", "2.2.2"))) 
  str(x)
#>  int [1:3] 1 1 1
#>  int [1:3] 2 2 2

iterate.numeric_version <- .iterate_via_length_and_extract

for(x in package_version(c("1.1.1", "2.2.2"))) 
  str(x)
#> Classes 'package_version', 'numeric_version'  hidden list of 1
#>  $ : int [1:3] 1 1 1
#> Classes 'package_version', 'numeric_version'  hidden list of 1
#>  $ : int [1:3] 2 2 2
```

### `iterate.array`

It is likely that users will be tempted to define their own methods for
base classes like `array` (e.g., if they want row-wise iteration), and
it might be good to get ahead of that by disallowing it. For example,
adding a check to `R CMD check` ensuring that no CRAN packages can
define an `iterate` method for objects of type `array`, `data.frame`,
`numeric`, and so on.
