<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Proposal

## Motivation

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. This desire comes up frequently in a
variety of contexts.

For example, when a dataset is too large to fit in memory, and must be
operated on it batches, it is desirable to able to write:

``` r
for (batch in lazy_loading_dataset()) ...
```

Or, when sequence elements are arriving at unknown times, potentially
forever, it would be desirable to be able to write:

``` r
for (x in socket_pool()) ...
```

In particular, there is a desire for `for` to iterate over sequences of:

-   lazily generated elements
-   unknown or infinite length
-   unknown order

In the reticulate package there is another strong motivator: the `for`
iteration protocol in Python is heavily used. For example, the Python
documentation for `tensorflow.data.Dataset` objects frequently shows the
`Dataset` object being passed directly to `for`, in, for example, a
training loop.

In most circumstances, R users can read simple examples in Python API
documentation and translate them almost verbatim to R using reticulate,
with little knowledge of Python required. However, when users want to
port a Python example using `for` to R, they must now be aware of
everything that Python's `for` is doing and are responsible for manually
constructing the iterator and managing it. An extensible `for` in R
would reduce the teaching burden for all R packages that wrap objects
that use the Python iterator protocol (like {tfdatasets}).

In general, giving R package authors the ability to define custom
iterables would reduce the teaching burden, because `for` is intuitively
easier to grasp for new users than some variation of `as_iterator()` and
`iter_next()`, and maybe even easier than `lapply()`.

Finally, the current behavior of `for` is somewhat idiosyncratic and
suboptimal for a small handful of object types. For example iterating
with `for` over:

-   `POSIXct` strips attributes and yields the bare underlying numeric.
-   `POSIXlt` strips attributes and iterates over the underlying list.
-   `factor` coerces to character and iterates over a character vector.
-   `numeric_version` strips attributes, yields length 3 integer
    vectors.
-   `environment` throws an error

Having a generic iteration protocol in R would provide a straightforward
mechanism to introduce intentional and narrow changes to the behavior of
`for` with these and other object types.

## Alternatives

This repository contains three alternatives for what an iteration
protocol in R might look like. Potential interfaces sketched out here
would allow users to pass any object to `for` that:

-   [Alternative
    1](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-1-iterate-generic.md):
    has a generic method with signature `iterate(x, state)`

-   [Alternative
    2](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-2-subset2-generic.md):
    has a `[[` method, or coercible to something that has a `[[` method
    with an `as.iterable()` generic.

-   [Alternative
    3](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-3-iterable-closure.md):
    is a function, or coercible to a function with an `as.iterator()`
    generic.

In general, all the approaches achieve the same outcome of allowing code
authors to define custom objects that can be passed to `for`; anything
enabled by one of the alternatives is enabled by all of them. The
primary differences are in the interface presented to users.

Each alternative aims to be narrow and conservative. To fully bake the
idea of a language-native iterator protocol however, there are some
additional tricky question that naturally emerge:

-   Should `as.list()` (and derivatives, like `lapply()`) invoke the
    iterator protocol? If not, should R provide a convenient way to
    materialize an iterable’s elements into a list, maybe with a
    convenient way to limit infinitely iterable objects?

-   Should `length()` return the number of elements an object will
    return when iterated over? If yes, what should `length()` return for
    objects where the iteration length is unknown? (e.g.,
    `length(tfdataset)` can return `NA` or `Inf`)

For the most part, the proposals here punt on these hard questions, and
stick to suggesting only the most narrow and conservative change.

## Comparison

### Alternative 1, `iterate(x, state)`

Pros:

-   Only introduces 1 new symbol.
-   Very simple implementation; low risk for unintended or subtle
    consequences.
-   `iterate()` seems easier to teach than `<<-` to new R authors.
-   Encourages authors to be intentional about state management.
-   Because it does not introduce an explicit `iterator` class, it
    sidesteps the thorny questions of what `length()` and `as.list()`
    should do for iterators.
-   Because the `state` object is exposed, is will be relatively easy
    for users to rewind and replay iterators (for objects where
    `iterate()` is a pure function).

Cons:

-   The iterable is prevented from being garbage collected during the
    duration of iteration.
-   Doesn't provide the nicest API for users who want to manually
    iterate over an object.
-   Invoking generic dispatch each iteration may introduce a performance
    penalty (though this could be mitigated by dispatching once and
    caching the method)

### Alternative 2, `[[`

Pros:

-   No new symbols would need to be introduced (in its most narrow
    implementation).
-   Many existing objects would gain the ability to be passed to `for`
    with no code changes.

Cons:

-   It seems to have high risk for unforeseen or subtle unintended
    consequences due to the many other uses of `[[`.
-   The concepts of subsetting and an 'out of bounds' condition don't
    naturally map to iterators and iterator exhaustion, especially when
    iteratring over sequences that are unordered, or when the iterable
    is stateful.
-   Existing conventions around `[[` would require that users
    communicate iterator exhaustion via a condition, not a sentinal. The
    need for `for` to setup a signal handler on each loop iteration
    might introduce a performance penalty.

### Alternative 3, `as.iterator()` returning closures

Pros:

-   It would feel familiar to R users who are accustomed to managing
    state in closures.
-   Simple implementation.
-   Provides a nice API for users to iterate over an object manually:  
    `i <- as.iterator(x); i(); i(); i(), i(), ...`

Cons:

-   A robust and ergonomic implementation would also include the
    introduction of an additional symbol or a magic phrase to indicate
    when an iterator is exhausted (e.g., an
    `iterator_exhausted_sentinal` or `iterator_exhausted_condition()`,
    or a faux typed simple condition with a magic message like
    `stop("IteratorExhausted")`).

-   Introduces a footgun by inviting code authors to capture
    environments which may contain large objects.

### Concluding remarks:

It is very possible to build a hybrid approach, that tries to provide
the best (and worst) of some combination of the alternatives. For
example, with Alternative 1, it is possible to build an `as.iterator()`
function atop `iterate()`.

``` r
as.iterator <- function(x, exhausted = NULL) {
  force(x); state <- NULL
  function(x) {
    step <- iterate(x, state)
    if(is.null(step))
      return(exhausted)
    state <<- step$state
    step$value
  }
}
```
