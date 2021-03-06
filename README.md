
# R Iterator Proposal

## Motivation

R currently lacks a way for users to customize what happens when objects
are passed to `for`. This desire comes up frequently in a variety of
contexts:

-   If a dataset is too large to fit in memory, and needs to be
    processed in batches, it’d be nice to write:

    ``` r
    for (batch in lazy_loading_dataset()) ...
    ```

-   If reading events from a stream (potentially forever), it’d be nice
    to write:

    ``` r
    for (x in socket_pool()) ...
    ```

In general, it would be convenient if `for` could iterate over:

-   Lazily generated sequences.
-   Sequences of unknown length.
-   Sequences of infinite length.

Additionally, it would be appealing to improve the somewhat
idiosyncratic handling of key S3 classes provided by base. A
particularly problematic example is the handling of POSIXt objects:

``` r
dt <- .POSIXct(c(1, 2))
for (x in dt) {
  str(x)
}
#>  num 1
#>  num 2
for (x in as.POSIXlt(dt)) {
  str(x)
}
#>  num [1:2] 1 2
#>  int [1:2] 0 0
#>  int [1:2] 18 18
#>  int [1:2] 31 31
#>  int [1:2] 11 11
#>  int [1:2] 69 69
#>  int [1:2] 3 3
#>  int [1:2] 364 364
#>  int [1:2] 0 0
#>  chr [1:2] "CST" "CST"
#>  int [1:2] -21600 -21600
```

But similar problems exist elsewhere, for example, with Dates and
factors.

This document explores ways in which we might extend `for` to handle new
types of sequences, while also providing a mechanism to introduce
intentional and narrow changes to the behavior of `for` with existing S3
classes.

### Reticulate

<!-- HW: I moved this to a separate section because I'm not sure how motivating this will be to R-Core folks -->

We are personally also strongly motivated by the
[reticulate](https://rstudio.github.io/reticulate/) package, which makes
it easy to use Python code from R. Many important Python classes make
heavy use of the `for` iteration protocol. For example, the Python
documentation for `tensorflow.data.Dataset` objects frequently shows the
`Dataset` object being passed directly to `for`, in, for example, a
training loop.

In most circumstances, R users can read simple examples in Python API
documentation and translate them almost verbatim to R using reticulate,
with little knowledge of Python required. However, when users want to
port a Python example using `for` to R, they need to be aware of
precisely what Python’s `for` does and manually construct and manage the
underlying iterator. An extensible `for` in R would reduce the teaching
burden for all R packages that wrap objects that use the Python iterator
protocol (like
[tfdatasets](https://tensorflow.rstudio.com/reference/tfdatasets/)).

## Alternatives

This repository implements three possible approaches to a general
iteration protocol:

<!-- HW: I think it would be good to give these names, rather than numbers -->

-   [Alternative
    1](https://github.com/t-kalinowski/r-iterator-ideas/blob/main/alternative-1-iterate-generic.md):
    introduces a new iterator generic with an explicit state object,
    inspired by Julia’s approach.

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

## Open issues

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
-   Because the `state` object is exposed, is will be relatively easy
    for users to rewind and replay iterators (for objects where
    `iterate()` is a pure function).

Cons:

-   The iterable can’t be garbage collected during the duration of
    iteration.
-   Doesn’t provide the nicest API for users who want to manually
    iterate over an object.
-   Some additional work is needed to avoid the performance penalty of
    performing method dispatch each iteration.

### Alternative 2, `[[`

Pros:

-   In the most narrow implementation, no new symbols are needed.
-   `out_of_bounds` condition might be useful in other circumstances.

Cons:

-   Potential for unforeseen or unintended consequences from changing
    behaviour of `[[`.
-   The concepts of subsetting and an ‘out of bounds’ condition don’t
    naturally map to iterators and iterator exhaustion, especially when
    iterating over sequences that are unordered, or when the iterable is
    stateful.
-   Would need to introduce new `out_of_bounds` condition type and
    retrofit it to existing base `[[` implementations.
-   Small performance penalty introduced by adding a condition handler
    to `for`.

### Alternative 3, `as.iterator()` returning closures

Pros:

-   Simple implementation.
-   Very lightweight to create a new iterator, especially for users to
    accustomed to managing state in closures.
-   Provides a nice API for users to iterate over an object manually:  
    `i <- as.iterator(x); i(); i(); i(); i(); ...`

Cons:

-   Requires an additional sentinel value to indicate iterator
    exhaustion.

-   Potential risk for capturing environments that may contain large
    objects.
