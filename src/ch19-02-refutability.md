## Refutability: Whether a Pattern Might Fail to Match

Patterns come in two forms: refutable and irrefutable. Patterns that will match
for any possible value passed are _irrefutable_. An example would be `x` in the
statement `let x = 5;` because `x` matches anything and therefore cannot fail
to match. Patterns that can fail to match for some possible value are
_refutable_. Here are some examples:

<!-- BEGIN INTERVENTION: 3c29eb2d-cbe9-4a2c-99b8-aa5c6467c8b4 -->
* In the expression `if let Some(x) = a_value`, then `Some(x)` is refutable. If the value in the `a_value` variable is `None` rather than
`Some`, the `Some(x)` pattern will not match. 
* In the expression `if let &[x, ..] = a_slice`, then `&[x, ..]` is refutable. If the value in the `a_slice` variable has zero elements, the `&[x, ..]` pattern will not match.
<!-- END INTERVENTION: 3c29eb2d-cbe9-4a2c-99b8-aa5c6467c8b4 -->

Function parameters, `let` statements, and `for` loops can only accept
irrefutable patterns because the program cannot do anything meaningful when
values don’t match. The `if let` and `while let` expressions and the
`let...else` statement accept refutable and irrefutable patterns, but the
compiler warns against irrefutable patterns because, by definition, they’re
intended to handle possible failure: the functionality of a conditional is in
its ability to perform differently depending on success or failure.

In general, you shouldn’t have to worry about the distinction between refutable
and irrefutable patterns; however, you do need to be familiar with the concept
of refutability so you can respond when you see it in an error message. In
those cases, you’ll need to change either the pattern or the construct you’re
using the pattern with, depending on the intended behavior of the code.

Let’s look at an example of what happens when we try to use a refutable pattern
where Rust requires an irrefutable pattern and vice versa. Listing 19-8 shows a
`let` statement, but for the pattern, we’ve specified `Some(x)`, a refutable
pattern. As you might expect, this code will not compile.

<Listing number="19-8" caption="Attempting to use a refutable pattern with `let`">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch19-patterns-and-matching/listing-19-08/src/main.rs:here}}
```

</Listing>

If `some_option_value` were a `None` value, it would fail to match the pattern
`Some(x)`, meaning the pattern is refutable. However, the `let` statement can
only accept an irrefutable pattern because there is nothing valid the code can
do with a `None` value. At compile time, Rust will complain that we’ve tried to
use a refutable pattern where an irrefutable pattern is required:

```console
{{#include ../listings/ch19-patterns-and-matching/listing-19-08/output.txt}}
```

Because we didn’t cover (and couldn’t cover!) every valid value with the
pattern `Some(x)`, Rust rightfully produces a compiler error.

If we have a refutable pattern where an irrefutable pattern is needed, we can
fix it by changing the code that uses the pattern: instead of using `let`, we
can use `if let`. Then if the pattern doesn’t match, the code will just skip
the code in the curly brackets, giving it a way to continue validly. Listing
19-9 shows how to fix the code in Listing 19-8.

<Listing number="19-9" caption="Using `let...else` and a block with refutable patterns instead of `let`">

```rust
{{#rustdoc_include ../listings/ch19-patterns-and-matching/listing-19-09/src/main.rs:here}}
```

</Listing>

We’ve given the code an out! This code is perfectly valid now. However,
if we give `if let` an irrefutable pattern (a pattern that will always
match), such as `x`, as shown in Listing 19-10, the compiler will give a
warning.

<Listing number="19-10" caption="Attempting to use an irrefutable pattern with `if let`">

```rust
{{#rustdoc_include ../listings/ch19-patterns-and-matching/listing-19-10/src/main.rs:here}}
```

</Listing>

Rust complains that it doesn’t make sense to use `if let` with an irrefutable
pattern:

```console
{{#include ../listings/ch19-patterns-and-matching/listing-19-10/output.txt}}
```

For this reason, match arms must use refutable patterns, except for the last
arm, which should match any remaining values with an irrefutable pattern. Rust
allows us to use an irrefutable pattern in a `match` with only one arm, but
this syntax isn’t particularly useful and could be replaced with a simpler
`let` statement.

Now that you know where to use patterns and the difference between refutable
and irrefutable patterns, let’s cover all the syntax we can use to create
patterns.

{{#quiz ../quizzes/ch18-02-refutability.toml}}