# R Iterator Proposal

## Motivation

R currently lacks a way for users to customize what happens when objects
are passed to `for`. This desire comes up frequently in a variety of
contexts:

-   If a dataset is too large to fit in memory, and needs to be
    processed in batches, it'd be nice to write:

    ``` r
    for (batch in lazy_loading_dataset()) ...
    ```

-   If reading events from a stream (potentially forever), it'd be nice
    to write:

    ``` r
    for (x in socket_pool()) ...
    ```

In particular, there is a desire for `for` to iterate over sequences of:

-   Lazily generated elements.
-   Unknown length.
-   Infinite length.

Giving R package authors the ability to define custom iterables would
reduce the teaching burden, because `for` is intuitively easier to grasp
for new users than some variation of `as_iterator()` and `iter_next()`,
and maybe even easier than `lapply()`.

Additionally, when considering extensions to `for`, it would be
appealing to improve the somewhat idiosyncratic and suboptimal behaviour
with a small handful of S3 classes. For example iterating with `for`
over:

-   `POSIXct` strips attributes and yields the bare underlying numeric.
-   `POSIXlt` strips attributes and iterates over the underlying list.
-   `factor` coerces to character and iterates over a character vector.
-   `numeric_version` strips attributes, yielding length 3 integer
    vectors.
-   `environment` throws an error.

This document explores ways in which we might extend `for` to handle new
types of sequences, while also providing a mechanism to introduce
intentional and narrow changes to the behavior of `for` with existing S3
classes.

### Reticulate

<!-- HW: I moved this to a separate section because I'm not sure how motivating this will be to R-Core folks -->

In the [reticulate](https://rstudio.github.io/reticulate/) package,
which makes it easy to use Python code from R, there is another strong
motivator. Many important Python classes make heavy use of the `for`
iteration protocol. For example, the Python documentation for
`tensorflow.data.Dataset` objects frequently shows the `Dataset` object
being passed directly to `for`, in, for example, a training loop.

In most circumstances, R users can read simple examples in Python API
documentation and translate them almost verbatim to R using reticulate,
with little knowledge of Python required. However, when users want to
port a Python example using `for` to R, they need to be aware of
precisely what Python's `for` does and manually construct and manage the
underlying iterator. An extensible `for` in R would reduce the teaching
burden for all R packages that wrap objects that use the Python iterator
protocol (like {tfdatasets}).

## Alternatives

This repository implements three possible approaches to a general
iteration protocol:

<!-- HW: I think it would be good to give these names, rather than numbers -->

-   [Alternative
    1](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-1-iterate-generic.md):
    introduces a new iterator generic with an explicit state object,
    inspired by Julia's approach.

-   [Alternative
    2](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-2-subset2-generic.md):
    has a `[[` method, or coercible to something that has a `[[` method
    with an `as.iterable()` generic.

-   [Alternative
    3](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-3-iterable-closure.md):
    proposes a stateful function called repeatedly until it signals
    exhaustion.

All the approaches achieve the same outcome of allowing developers to
define S3 classes with custom `for` behavior; anything enabled by one of
the alternatives is enabled by all of them. The primary differences is
in the interface presented to developers.

### Open issues

-   Should `as.list()` (and derivatives, like `lapply()`) invoke the
    iterator protocol? If not, should R provide a convenient way to
    materialize an iterable’s elements into a list, maybe with a
    convenient way to limit infinitely iterable objects?

-   Should `length()` return the number of elements an object will
    return when iterated over? If yes, what should `length()` return for
    objects where the iteration length is unknown? (e.g.,
    `length(tfdataset)` can return `NA` or `Inf`)

## Comparison

### Alternative 1, `iterate(x, state)`

Pros:

-   Only introduces 1 new function.
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

-   Potential for unforeseen or subtle unintended consequences due to
    the many other uses of `[[`.
-   The concepts of subsetting and an 'out of bounds' condition don't
    naturally map to iterators and iterator exhaustion, especially when
    iterating over sequences that are unordered, or when the iterable is
    stateful.
-   Would need to introduce new `out_of_bounds` condition type and
    retrofit it to existing base `[[` implementations.
-   Existing conventions around `[[` would require that users
    communicate iterator exhaustion via a condition, not a sentinel. The
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
