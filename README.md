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

## Recommendations

Alternative 2 (`[[`) is not recommended. It seems to have high risk for
unforeseen or subtle unintended consequences due to the many other uses
of `[[`.

Alternative 3 (`as.iterator()` returning closures) would feel natural to
R users who are accustomed to managing state in closures. However, an
ergonomic implementation would also include the introduction of an
additional symbol or a magic phrase to indicate when an iterator is
exhausted (e.g., an `iterator_exhausted_sentinal` or
`iterator_exhausted_condition()`, or a faux typed simple condition with
a magic message like `stop("IteratorExhausted")`). By comparison,
alternative 1 would only introduce 1 new symbol: `iterate()`.

Additionally, `iterate()` seems like it would be easier to teach than
`<<-` to new R users.

It is my (Tomasz's) opinion that alternative 1 (`iterate()`) strikes the
best balance between:

-   minimizing new api surface and new symbols added,
-   being easy to understand, teach, and implement methods for,
-   maximizing the flexibility users gain with the change, and
-   minimizing the risk of unforeseen or subtle unintended consequences.
