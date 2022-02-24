<!-- README.md is generated from README.Rmd. Please edit that file -->

# R Iterator Proposal

R doesn’t currently have a mechanism for users to customize what happens
when objects are passed to `for`. This repository contains a few
alternatives for what such a mechanism might look like.

Potential interfaces sketched out here would allow users to pass any
object to `for` that:

1.  has a generic method with signature: `iterate(x, state)`

2.  has a `[[` method, or coercible to something that has a `[[` method
    with an `as.iterable()` generic.

3.  is a function, or coercible to a function with an `as.iterator()`
    generic.

In general, all the approaches achieve the same outcome of allowing code
authors to define custom objects that can be passed to `for`; anything
enabled by one of the alternatives is enabled by all of them. The
primary differences are in the interface presented to users.

Each proposal is also very narrow and conservative, no breaking changes
to R would be introduced. To fully bake the idea of a language-native
iterator protocol however, there are some additional tricky question
that naturally emerge:

-   Should `as.list()` (and derivatives, like `lapply()`) invoke the
    iterator protocol? If not, should R provide a convenient way to
    materialize an iterable’s elements into a list, maybe with a
    convenient way to limit infinitely iterable objects?

-   Should `length()` return the number of elements an object will
    return when iterated over? If yes, what should `length()` return for
    objects where the iteration length is unknown?

For the most part, the proposals here punt on these hard questions, and
stick to suggesting only the most narrow and conservative change.

## Comparison

### Alternative 1 (`iterate(x, state)`):

Pros:

-   Only introduces 1 new symbol.
-   Very simple implementation; low risk for unintended or subtle
    consequences.
-   `iterate()` seems easier to teach than `<<-` to new R users.
-   Encourages authors to be intentional about state management.
-   Because it does not introduce an explicit `iterator` class, it
    sidesteps the thorny questions of what `length()` and `as.list()`
    should do for iterators.

Cons:

-   The iterable is prevented from being garbage collected during the
    duration of iteration.

### Alternative 2 (`[[`):

Pros:

-   No new symbols would need to be introduced (in it's most narrow
    implementation).
-   Many existing objects would gain the ability to be passed to `for`
    with no code changes.

Cons:

-   It seems to have high risk for unforeseen or subtle unintended
    consequences due to the many other uses of `[[`.
-   The concepts of subsetting and an 'out of bounds' condition don't
    naturally map to stateful iterators and iterator exhaustion--feels
    like a forced API.

### Alternative 3 (`as.iterator()` returning closures):

Pros:

-   It would feel familiar to R users who are accustomed to managing
    state in closures.

Cons:

-   A robust and ergonomic implementation would also include the
    introduction of an additional symbol or a magic phrase to indicate
    when an iterator is exhausted (e.g., an
    `iterator_exhausted_sentinal` or `iterator_exhausted_condition()`,
    or a faux typed simple condition with a magic message like
    `stop("IteratorExhausted")`).

-   Introduces a footgun by inviting code authors to capture
    environments which may contain large objects.

## Recomendations

-   Alternative 1: Recommended and preferred.
-   Alternative 2: Not recommended
-   Alternative 3: Acceptable, but not preferred.

It is my (Tomasz's) opinion that alternative 1 (`iterate()`) strikes the
best balance between:

-   minimizing new api surface and new symbols added,
-   being easy to understand, teach, and implement methods for,
-   maximizing the flexibility users gain with the change, and
-   minimizing the risk of unforeseen or subtle unintended consequences.
