- Feature Name: stable-generators
- Start Date: 2021-04-XX
- RFC PR: [rust-lang/rfcs#](https://github.com/rust-lang/rfcs/pull/)
- Rust Issue: [rust-lang/rust#](https://github.com/rust-lang/rust/issues/)

# Summary
[summary]: #summary

This is an RFC for partially stabilizing an existing nightly feature,
generators (which are a subset of coroutines). This RFC is a follow up to
[rust-lang/rfcs#2033](https://github.com/rust-lang/rfcs/pull/2033). The
intention here is to build concensus around the feature's specifics.

# Motivation
[motivation]: #motivation

This feature is the underlying mechanism behind `async`/`await` state machines,
and has been available to use on the nighlty channel for 4 years now. The lack
of stable generators burdens users' with the need to implement `Iterator`s and
`AsyncIterator`s (`Stream`s) combinators by hand, which can be difficult,
particularly when it comes to `AsyncIterator`s due to the need to understand
`Pin` in order to be productive, which is a common source of confusion amongst
users. As part of the Async Vision document, `Pin` and manually implemented
combinators have been identified as a common source of frustration. Generators
are a common feature available in languages like C#, JS and Python. Stabilizing
this feature would be a huge boon to productivity and make significant
improvement in the ease of use of `async`/`await`.

There are two main parts to the stabilization of this feature that need to be
addressed:

* What's the actual syntax for generators? Should new keywords be introduced to
  the language? How verbose should it be? Should annotations at the item level
  be needed?

* What are the stabilized semantics of generators? How will they behave in
  synchronous and asynchronous scopes? Are changes from their current behavior
  needed? If so, how extensive?

I am *explicitly* pushing forward a desire to *not* implement the more general
feature, coroutines, as these have outstanding open questions that are yet to be
resolved but that aren't necessary for the most common use-case of writing
synchronous or asynchronous iterables.

For simple `Iterator` implementations, the trait provides [`from_fn`] which
addresses the simplest cases, but that doesn't compose well or allow users to
write complex combinators.

[`from_fn`]: https://doc.rust-lang.org/std/iter/fn.from_fn.html

```rust
let start = 0;
let end = 3;
let mut current = start;
let counter = std::iter::from_fn(move || {
    let prev = current ;
    current += 1;
    if prev < end {
        Some(prev)
    } else {
        None
    }
});
```

### Generators syntax

The current nightly-only syntax mimics closures and is only available in
expression context:

```rust
#![feature(generators)]

fn main() {
    let ret = "foo";
    let mut gen = move || {
        yield 1;
        return ret
    };
}
```

These expressions look like closures with no header differentiating them from
regular closures, except for the existence of a yield statement within them.
This can be seen as undesirable as it requires users to inspect the totality of
closure bodies to confirm what the expression actually is. In order to cater to
the `Iterator` use-case specifically, the `return` value isn't necessary.

Assuming a new keyword `gen` or `generator` would be introduced, the following
syntax is put forward:

```rust
fn main() {
    let mut gen = generator || {
        for i in 0..3 {
            yield i;
        }
    };
}
```

Because we want this same syntax to be used for `Stream`s, this new keyword
should compose well with existing and planned related keywords:

```rust
fn main() {
    let start = 0;
    let mut gen = async generator move |end| {
        for i in start..end {
            yield i;
        }
    };
}
```

Generators are "unnameable types" that conform to the unstable `Generator`
trait, similar to closures and the `Fn*()`-family of traits. We do *not* propose
to stabilize the `Generator` trait, but make it trivial to use the resulting
generator anywhere an `IntoIterator` is accepted:

```rust
fn main() {
    let start = 0;
    let mut gen = generator move |end| {
        for i in start..end {
            yield i;
        }
    };
    for i in gen(3) {
        println!("{}", i);
    }
}
```

This is in contrast with the current usage that requires people to interact with
`Pin` and doesn't use `Iterator` as its interface:

```rust
#![feature(generators, generator_trait)]

use std::ops::{Generator, GeneratorState};
use std::pin::Pin;

fn main() {
    let mut generator = || {
        for i in 0..3 {
            yield i;
        }
    };

    for _ in 0..3 {
      match Pin::new(&mut generator).resume(()) {
          GeneratorState::Yielded(val) => {
              println!("{}", val);
          }
          _ => panic!("unexpected return from resume"),
      }
    }
    match Pin::new(&mut generator).resume(()) {
        GeneratorState::Complete(()) => {}
        _ => panic!("unexpected return from resume"),
    }
}
```

This new syntax should also fit neatly with potential extensions to the language
for more ergonomic interaction with `Stream`s:

```rust
async fn main() {
    let start = 0;
    let mut gen = async generator move |end| {
        for i in start..end {
            yield i;
        }
    };
    for await i in gen(3) {
        println!("{}", i);
    }
}
```

By introducing a new keyword and a way for arguments to be passed to a generator
before usage, the hability to write generators in item context becomes possible:

```rust
generator gen(start: usize, end: usize) yields usize {
    for i in start..end {
        yield i;
    }
}
```

`yield` is an already reserved keyword, which can be accepted only inside of
`generator` items and closures. Under this design `generator` would have to be
introduced as a new keyword.

*Note: All of these sytnax suggestions are a work-in-progress.*

The intention with this syntax is to be as familiar as possible to existing Rust
programmers and be familiar to people coming from other languages.

```cs
public class Range
{
    public static System.Collections.Generic.IEnumerable<int> Range(int start, int end)
    {
        for (int i = start; i < end; i++)
        {
            yield return i;
        }
    }
}
```

```python
def my_range(start, end):
    for i in range(start, end):
        yield i
```

------

To reiterate where we are at this point, here's some of the highlights:

* One of Rust's roadmap goals for 2021 is pushing Rust's usage on the server.
* A major part of this goal is going to be implementing async/await syntax for
  Rust with futures.
* The async/await syntax has a relatively straightforward syntactic definition
  (borrowed from other languages) with procedural macros.
* The procedural macro itself can produce optimal futures through the usage of
  *stackless coroutines*

Put another way: if the compiler implements stackless coroutines as a feature,
we have now achieved async/await syntax!

### Features of stackless coroutines

At this point we'll start to tone down the emphasis of servers and async I/O
when talking about stackless coroutines. It's important to keep them in mind
though as motivation for coroutines as they guide the design constraints of
coroutines in the compiler.

At a high-level, though, stackless coroutines in the compiler would be
implemented as:

* No implicit memory allocation
* Coroutines are translated to state machines internally by the compiler
* The standard library has the traits/types necessary to support the coroutines
  language feature.

Beyond this, though, there aren't many other constraints at this time. Note that
a critical feature of async/await is that **the syntax of stackless coroutines
isn't all that important**. In other words, the implementation detail of
coroutines isn't actually exposed through the `#[async]` and `await!`
definitions above. They purely operate with `Future` and simply work internally
with coroutines. This means that if we can all broadly agree on async/await
there's no need to bikeshed and delay coroutines. Any implementation of
coroutines should be easily adaptable to async/await syntax.

# Detailed design
[design]: #detailed-design


##### Open Questions - coroutines

* Should we delay introduction of generators in favor of generalized
  coroutines?

##### Open Questions - async/await

* `Pin` behavior on the actual `impl Stream for Generator`

##### Tests - Basic usage

* Coroutines which don't yield at all and immediately return results
* Coroutines that yield once and then return a result
* Creating a coroutine which closes over a value, and then returning it
* Returning a captured value after one yield
* Destruction of a coroutine drops closed over variables
* Create a coroutine, don't run it, and drop it
* Coroutines are `Send` and `Sync` like closures are wrt captured variables
* Create a coroutine on one thread, run it on another

##### Tests - Basic compile failures

* Coroutines cannot close over data that is destroyed before the coroutine is
  itself destroyed.
* Coroutines closing over non-`Send` data are not `Send`

##### Test - Interesting control flow

* Yield inside of a `for` loop a set number of times
* Yield on one branch of an `if` but not the other (take both branches here)
* Yield on one branch of an `if` inside of a `for` loop
* Yield inside of the condition expression of an `if`

##### Tests - Panic safety

* Panicking in a coroutine doesn't kill everything
* Resuming a panicked coroutine is memory safe
* Panicking drops local variables correctly

##### Tests - Debuginfo

* Inspecting variables before/after yield points works
* Breaking before/after yield points works

Suggestions for more test are always welcome!

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

Generators are a common feature in multiple languages. New documentation will
be written explaining how to write `Iterator`s and `Stream` combinators using
the new syntax.

# Drawbacks
[drawbacks]: #drawbacks

# Alternatives
[alternatives]: #alternatives

# Unresolved questions
[unresolved]: #unresolved-questions

*To be updated after T-lang design meeting.*
