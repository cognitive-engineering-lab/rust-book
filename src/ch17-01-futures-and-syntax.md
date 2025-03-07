## Futures and the Async Syntax

The key elements of asynchronous programming in Rust are *futures* and Rust’s
`async` and `await` keywords.

A *future* is a value which may not be ready now, but will become ready at some
point in the future. (This same concept shows up in many languages, sometimes
under other names like “task” or “promise”.) Rust provides a `Future` trait as a
building block so different async operations can be implemented with different
data structures, but with a common interface. In Rust, we say that types which
implement the `Future` trait are futures. Each type which implements `Future`
holds its own information about the progress that has been made and what "ready"
means.

The `async` keyword can be applied to blocks and functions to specify that they
can be interrupted and resumed. Within an async block or async function, you can
use the `await` keyword to wait for a future to become ready, called *awaiting a
future*. Each place you await a future within an async block or function is a
place that async block or function may get paused and resumed.

Some other languages also use `async` and `await` keywords for async
programming. If you are familiar with those languages, you may notice some
significant differences in how Rust does things, including how it handles the
syntax. That is for good reason, as we will see!

Most of the time when writing async Rust, we use the `async` and `await`
keywords. Rust compiles them into equivalent code using the `Future` trait, much
like it compiles `for` loops into equivalent code using the `Iterator` trait.
Because Rust provides the `Future` trait, though, you can also implement it for
your own data types when you need to. Many of the functions we will see
throughout this chapter return types with their own implementations of `Future`.
We will return to the definition of the trait at the end of the chapter and dig
into more of how it works, but this is enough detail to keep us moving forward.

That may all feel a bit abstract. Let’s write our first async program: a little
web scraper. We will pass in two URLs from the command line, fetch both of them
concurrently, and return the result of whichever one finishes first. This
example will have a fair bit of new syntax, but don’t worry. We will explain
everything you need to know as we go.

### Our First Async Program

To keep this chapter focused on learning async, rather than juggling parts of
the ecosystem, we have created the `trpl` crate (`trpl` is short for “The Rust
Programming Language”). It re-exports all the types, traits, and functions you
will need, primarily from the [`futures`][futures-crate] and [`tokio`][tokio]
crates.

- The `futures` crate is an official home for Rust experimentation for async
  code, and is actually where the `Future` type was originally designed.

- Tokio is the most widely used async runtime in Rust today, especially (but
  not only!) for web applications. There are other great runtimes out there,
  and they may be more suitable for your purposes. We use Tokio under the hood
  for `trpl` because it is good and widely used.

In some cases, `trpl` also renames or wraps the original APIs to let us stay
focused on the details relevant to chapter. If you want to understand what the
crate does, we encourage you to check out [its source code][crate-source]. You
will be able to see what crate each re-export comes from, and we have left
extensive comments explaining what the crate does.

Go ahead and add the `trpl` crate to your `hello-async` project:

```console
$ cargo add trpl
```

Now we can use the various pieces provided by `trpl` to write our first async
program. We will build a little command line tool which fetches two web pages,
pulls the `<title>` element from each, and prints out the title of whichever
finishes that whole process first.

<Listing number="17-1" file-name="src/main.rs" caption="Defining an async function to get the title element from an HTML page">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-01/src/main.rs:all}}
```

</Listing>

In Listing 17-1, we define a function named `page_title`, and we mark it with
the `async` keyword. Then we use the `trpl::get` function to fetch whatever URL
is passed in, and, and we await the response by using the `await` keyword. Then
we get the text of the response by calling its `text` method and once again
awaiting it with the `await` keyword. Both of these steps are asynchronous. For
`get`, we need to wait for the server to send back the first part of its
response, which will include HTTP headers, cookies, and so on. That part of the
response can be delivered separately from the body of the request. Especially if
the body is very large, it can take some time for it all to arrive. Thus, we
have to wait for the *entirety* of the response to arrive, so the `text` method
is also async.

We have to explicitly await both of these futures, because futures in Rust are
*lazy*: they don’t do anything until you ask them to with `await`. (In fact,
Rust will show a compiler warning if you do not use a future.) This should
remind you of our discussion of iterators [back in Chapter 13][iterators-lazy].
Iterators do nothing unless you call their `next` method—whether directly, or
using `for` loops or methods like `map` which use `next` under the hood. With
futures, the same basic idea applies: they do nothing unless you explicitly ask
them to. This laziness allows Rust to avoid running async code until it is
actually needed.

> Note: This is different from the behavior we saw when using `thread::spawn` in
> the previous chapter, where the closure we passed to another thread started
> running immediately. It is also different from how many other languages
> approach async! But it is important for Rust. We will see why that is later.

Once we have `response_text`, we can then parse it into an instance of the
`Html` type using `Html::parse`. Instead of a raw string, we now have a data
type we can use to work with the HTML as a richer data structure. In particular,
we can use the `select_first` method to find the first instance of a given CSS
selector. By passing the string `"title"`, we will get the first `<title>`
element in the document, if there is one. Since there may not be any matching
element, `select_first` returns an `Option<ElementRef>`. Finally, we use the
`Option::map` method, which lets us work with the item in the `Option` if it is
present, and do nothing if it is not. (We could also use a `match` expression
here, but `map` is more idiomatic.) In the body of the function we supply to
`map`, we call `inner_html` on the `title_element` to get its content, which is
a `String`. When all is said and done, we have an `Option<String>`.

Notice that Rust’s `await` keyword goes after the expression you are awaiting,
not before it. That is, it is a *postfix keyword*. This may be different from
what you might be used to if you have used async in other languages. Rust chose
this because it makes chains of methods much nicer to work with. As a result, we
can change the body of `page_title` to chain the `trpl::get` and `text`
function calls together with `await` between them, as shown in Listing 17-2:

<Listing number="17-2" file-name="src/main.rs" caption="Chaining with the `await` keyword">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-02/src/main.rs:chaining}}
```

</Listing>

With that, we have successfully written our first async function! Before we add
some code in `main` to call it, let’s talk a little more about what we have
written and what it means.

When Rust sees a block marked with the `async` keyword, it compiles it into a
unique, anonymous data type which implements the `Future` trait. When Rust
sees a function marked with `async`, it compiles it into a non-async function
whose body is an async block. Thus, an async function’s return type is the type
of the anonymous data type the compiler creates for that async block.

Thus, writing `async fn` is equivalent to writing a function which returns a
*future* of the return type. When the compiler sees a function like `async fn
page_title` in Listing 17-1, it is equivalent to a non-async function defined
like this:

```rust
# extern crate trpl; // required for mdbook test
use std::future::Future;
use trpl::Html;

fn page_title(url: &str) -> impl Future<Output = Option<String>> + '_ {
    async move {
        let text = trpl::get(url).await.text().await;
        Html::parse(&text)
            .select_first("title")
            .map(|title| title.inner_html())
    }
}
```

Let’s walk through each part of the transformed version:

* It uses the `impl Trait` syntax we discussed back in the [“Traits as
  Parameters”][impl-trait] section in Chapter 10.
* The returned trait is a `Future`, with an associated type of `Output`. Notice
  that the `Output` type is `Option<String>`, which is the same as the the
  original return type from the `async fn` version of `page_title`.
* All of the code called in the body of the original function is wrapped in an
  `async move` block. Remember that blocks are expressions. This whole block is
  the expression returned from the function.
* This async block produces a value with the type `Option<String>`, as described
  above. That value matches the `Output` type in the return type. This is just
  like other blocks you have seen.
* The new function body is an `async move` block because of how it uses the
  `name` argument. (We will talk about `async` vs. `async move` much more later
  in the chapter.)
* The new version of the function has a kind of lifetime we have not seen before
  in the output type: `'_`. Because the function returns a `Future` which refers
  to a reference—in this case, the reference from the `url` parameter—we need to
  tell Rust that we mean for that reference to be included. We do not have to
  name the lifetime here, because Rust is smart enough to know there is only one
  reference which could be involved, but we *do* have to be explicit that the
  resulting `Future` is bound by that lifetime.

Now we can call `page_title` in `main`. To start, we will just get the title for
a single page. In Listing 17-3, we follow the same pattern we used for getting
command line arguments back in Chapter 12. Then we pass the first URL
`page_title`, and await the result. Since the value produced by the future is an
`Option<String>`, we use a `match` expression to print different messages to
account for whether the page had a `<title>`.

<Listing number="17-3" file-name="src/main.rs" caption="Calling the `page_title` function from `main` with a user-supplied argument">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-03/src/main.rs:main}}
```

</Listing>

Unfortunately, this does not compile. The only place we can use the `await`
keyword is in async functions or blocks, so Rust will not let us mark `main` as
`async`.

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-03
cargo build
copy just the compiler error
-->

```text
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:6:1
  |
6 | async fn main() {
  | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

The reason is that async code needs a *runtime*: a Rust crate which manages the
details of executing asynchronous code. A program's `main` function can
initialize a runtime, but it is not a runtime itself. (We will see more about
why this is a bit later.) Every async program in Rust has at least one place
where it sets up a runtime and executes the futures.

Most languages which support async bundle a runtime with the language. Rust does
not. Instead, there are many different async runtimes available, each of which
makes different tradeoffs suitable to the use case they target. For example, a
high-throughput web server with many CPU cores and a large amount of RAM has
very different different needs than a microcontroller with a single core, a
small amount of RAM, and no ability to do heap allocations. The crates which
provide those runtimes also often supply async versions of common functionality
like file or network I/O.

Here, and throughout the rest of this chapter, we will use the `run` function
from the `trpl` crate, which takes a future as an argument and runs it to
completion. Behind the scenes, calling `run` sets up a runtime to use to run the
future passed in. Once the future completes, `run` returns whatever value the
future produced.

We could pass the future returned by `page_title` directly to `run`. Once it
completed, we would be able to match on the resulting `Option<String>`, the way
we tried to do in Listing 17-3. However, for most of the examples in the chapter
(and most async code in the real world!), we will be doing more than just one
async function call, so instead we will pass an `async` block and explicitly
await the result of calling `page_title`, as in Listing 17-4.

<Listing number="17-4" caption="Awaiting an async block with `trpl::run`" file-name="src/main.rs">

<!-- should_panic,noplayground because mdbook does not pass args -->

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch17-async-await/listing-17-04/src/main.rs:run}}
```

</Listing>

When we run this, we get the behavior we might have expected initially:

```console
{{#include ../listings/ch17-async-await/listing-17-04/output.txt}}
```

Phew: we finally have some working async code! This now compiles, and we can run
it. Pick a couple URLs and run the command line tool. You may discover that some
sites are reliably faster than others, while in other cases which site “wins”
varies from run to run. Let’s briefly turn our attention back to how futures
work.

Each *await point*—that is, every place where the code uses the `await`
keyword—represents a place where control gets handed back to the runtime. To
make that work, Rust needs to keep track of the state involved in the async
block, so that the runtime can kick off some other work and then come back when
it is ready to try advancing this one again. This is an invisible state machine,
as if you wrote something like this:

```rust
# extern crate trpl; // required for mdbook test
enum PageTitleFuture<'a> {
    GetAwaitPoint {
        url: &'a str,
    },
    TextAwaitPoint {
        response: trpl::Response,
    },
}
```

Writing that out by hand would be tedious and error-prone, especially when
making changes to code later. Instead, the Rust compiler creates and manages the
state machine data structures for async code automatically. If you’re wondering:
yep, the normal borrowing and ownership rules around data structures all apply.
Happily, the compiler also handles checking those for us, and has good error
messages. We will work through a few of those later in the chapter!

Ultimately, something has to execute that state machine. That something is a
runtime. (This is why you may sometimes come across references to *executors*
when looking into runtimes: an executor is the part of a runtime responsible for
executing the async code.)

Now we can understand why the compiler stopped us from making `main` itself an
async function back in Listing 17-3. If `main` were an async function, something
else would need to manage the state machine for whatever future `main` returned,
but main is the starting point for the program! Instead, we use the `trpl::run`
function, which sets up a runtime and runs the future returned by `page_title`
until it returns `Ready`.

> Note: some runtimes provide macros to make it so you *can* write an async main
> function. Those macros rewrite `async fn main() { ... }` to be a normal `fn
> main` which does the same thing we did by hand in Listing 17-5: call a
> function which runs a future to completion the way `trpl::run` does.

Let’s put these pieces together and see how we can write concurrent code, by
calling `page_title_for` with two different URLs passed in from the command line
and racing it.

<Listing number="17-5" caption="" file-name="src/main.rs">

<!-- should_panic,noplayground because mdbook does not pass args -->

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch17-async-await/listing-17-05/src/main.rs:all}}
```

</Listing>

In Listing 17-5, we begin by calling `page_title` with both of the user-supplied
URLs. We save the futures produced by calling `page_title` as `title_fut_1` and
`title_fut_2`. Remember, these don’t do anything yet, because futures are lazy,
and we have not yet awaited them. Then we pass the futures  `trpl::race`, which
returns a value to indicate which of the futures passed to it finishes first.

Either future can legitimately “win,” so it does not make sense to return a
`Result`. Instead, `race` returns a type we have not seen before,
`trpl::Either`. The `Either` type is somewhat like a `Result`, in that it has
two cases. Unlike `Result`, though, there is no notion of success or failure
baked into `Either`. Instead, it uses `Left` and `Right` to indicate “one or the
other”.

```rust
enum Either<A, B> {
    Left(A),
    Right(B)
}
```

The `race` function returns `Left` if the first argument finishes first, with
that future’s output, and `Right` with the second future argument’s output if
*that* one finishes first. This matches the order the arguments appear when
calling the function: the first argument is to the left of the second argument.

We also update `page_title_for` to return the same URL passed in. That way, if
the page which returns first does not have a `<title>` we can resolve, we can
still print a meaningful message. With that information available, we wrap up by
updating our `println!` output to indicate both which URL finished first and
what the `<title>` was for the web page at that URL, if any.

You have built a small working web scraper now, which we could extend in a bunch
of different directions. More importantly, you have learned the basics of
working with futures, so we can now dig into even more of the things we can *do*
with async.

{{#quiz ../quizzes/async-01-futures-and-syntax.toml}}

[impl-trait]: ch10-02-traits.html#traits-as-parameters
[iterators-lazy]: ch13-02-iterators.html
<!-- TODO: map source link version to version of Rust? -->
[crate-source]: https://github.com/rust-lang/book/tree/main/packages/trpl
[futures-crate]: https://crates.io/crates/futures
[tokio]: https://tokio.rs
