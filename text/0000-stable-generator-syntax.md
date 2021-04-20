- Feature Name: stable-generators-syntax
- Start Date: 2021-04-XX
- RFC PR: [rust-lang/rfcs#](https://github.com/rust-lang/rfcs/pull/)
- Rust Issue: [rust-lang/rust#](https://github.com/rust-lang/rust/issues/)

# Summary
[summary]: #summary

This is an RFC for the syntax of an existing nightly feature,
generators (which are a subset of coroutines). This RFC is a follow up to
[rust-lang/rfcs#2033](https://github.com/rust-lang/rfcs/pull/2033) and a
companion to rust-lang/rfcs#XXXX. The intention here is to build concensus
around the feature's syntax only, ignoring the specifics of the implementation's
semantics.

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

This RFC aims to answer the following questions:

* What's the actual syntax for generators?
* Should new keywords be introduced to the language?
* How verbose should it be?
* Should annotations at the item level be needed?

This RFC *explicitly* skips over the implementation and stabilization of
generalized coroutines, as these have outstanding open questions that are yet to
be resolved that *aren't* necessary for the most common use-case of writing
synchronous or asynchronous iterables.

## Status quo

### Alternatives

For the examples below, we show how to write code that is equivalent to the
expression `start..end` or compositions of multiple iterables.

#### `Iterator::from_fn`

For simple `Iterator` implementations, the trait provides [`from_fn`] which
addresses the simplest cases, but that doesn't compose well or allow users to
write complex combinators.

[`from_fn`]: https://doc.rust-lang.org/std/iter/fn.from_fn.html

```rust
let start = 0;
let end = 3;
let mut current = start;
let counter = std::iter::from_fn(move || {
    let prev = current;
    current += 1;
    if prev < end {
        Some(prev)
    } else {
        None
    }
});
```

#### Custom `Iterator` implementation

Somebody wanting to write this code without relying on syntactic sugar can
implement a new `struct` and implement `Iterator` directly:

```rust
struct MyRange {
    start: usize,
    end: usize,
}

impl Iterator for MyRange {
    type Item = usize;

    fn next(&mut self) -> Option<Self::Item> {
        if self.start >= self.end {
            return None;
        }
        let prev = self.start;
        self.start += 1;
        Some(prev)
    }
}

fn main() {
    let mut iter = MyRange { start: 0, end: 3 };
    for _ in iter {}
}
```

This gives the user the most flexibility, but it imposes the user with writing a
lot of boilerplate that grows quickly as the underlying algorithm becomes more
complex. Furthermore, this problem is exacerbated when implementing `Stream`
because of the need to interact with `Pin`:

```rust
#![feature(async_stream)]

struct MyRange {
    start: usize,
    end: usize,
}

impl Stream for MyRange {
    type Item = usize;

    fn poll_next(
        mut self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Option<Self::Item>> {
        if self.start >= self.end {
            return Poll::Ready(None);
        }
        let prev = self.start;
        self.start += 1;
        Poll::Ready(Some(self.count))
    }
}
```

In these simple examples, the implementation requirements aren't onerous, but
when writing combinators that consume multiple `Stream`s and create a single
output `Stream`, the code quickly becomes complex.

#### Current generators syntax

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

To interact with with these generators, users need to interact with the
`Generator` trait and `Pin`, as it doesn't use the `Iterator` or `IntoIterator`
as its interface:

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

#### gen-iter

[gen-iter] is a nightly-only crate that exposes a macro-by-example that allows
the user to write:

```rust
#![feature(generators)]
#![feature(conservative_impl_trait)]

use gen_iter::gen_iter;

fn my_range(start: usize, end: usize) -> impl Iterator<Item = u64> {
    gen_iter!({
        let mut current = start;
        loop {
            let prev = current;
            current += 1;
            yield prev;
            if prev >= end {
                break;
            }
        }
    })
}

for i in my_range(0, 3) {
    println!("{}", i);
}
```

[gen-iter](https://docs.rs/gen-iter/0.2.0/gen_iter/)

#### Propane

[Propane] is a nightly-only crate that exposes proc-macros that allows the user
to write `Iterator`s and `Stream`s by annotating a function or inline in an
expression:

```rust
#[propane::generator]
fn foo(start: usize, end: usize) -> usize {
    for n in start..end {
        yield n;
    }
}

fn main() {
    let mut iter = propane::gen! {
        for x in 0..3 {
            yield x;
        }
    };
    let mut iter = foo(0, 3);
    for _ in iter {}
}
```

[Propane]: https://github.com/withoutboats/propane

#### async-stream

[async-stream] is similar to propane, providing a proc-macro to write `Stream`s
exclusively, ignoring `Iterator`s:

```rust
async fn foo() {
    let s = stream! {
        for i in 0..3 {
            yield i;
        }
    };

    pin_mut!(s); // needed for iteration

    while let Some(value) = s.next().await {
        println!("got {}", value);
    }
}
```

[async-stream](https://github.com/tokio-rs/async-stream)

#### Other languages

The intention with this syntax is to be as familiar as possible to existing Rust
programmers as well as to people coming from other languages. Because of this,
it is informative to look at what comparable features look like in other
languages.

C#

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

Python

```python
def my_range(start, end):
    for i in range(start, end):
        yield i
```

JavaScript

```javascript
function* my_range(start, end) {
  var current = start;
  while (current < end) {
    yield current;
    current = current + 1;
  }
}

const gen = my_range(0, 3);

console.log(gen.next().value); // 0
console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
```

## Proposed syntax

It is desireable for the final syntax to mimic existing patterns, both within
Rust, as well as from other languages.

Assuming a new keyword `gen` or `generator` would be introduced, the following
syntax is put forward:

```rust
fn main() {
    let mut my_range = generator || {
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
    let mut my_range = async generator move |end| {
        for i in start..end {
            yield i;
        }
    };
}
```

Generators are "unnameable types" that conform to the unstable `Generator`
trait, similar to closures and the `Fn*()`-family of traits. We do *not* propose
to stabilize the `Generator` trait, but to make it trivial to use the resulting
generator anywhere an `IntoIterator` is accepted:

```rust
fn main() {
    let start = 0;
    let mut my_range = generator move |end| {
        for i in start..end {
            yield i;
        }
    };
    for i in my_range (3) {
        println!("{}", i);
    }
}
```

This new syntax should also fit with potential extensions to the language for
more ergonomic interaction with `Stream`s:

```rust
async fn main() {
    let start = 0;
    let mut my_async_range = async generator move |end| {
        for i in start..end {
            yield i;
        }
    };
    for await i in my_async_range(3) {
        println!("{}", i);
    }
}
```

Alternatively, we could keep the current nightly behavior where in order to
create a generator we "just" use `yield` inside of a closure:

```rust
let mut my_async_range = async move |end| {
    for i in start..end {
        yield i;
    }
};
```

By introducing a new keyword and keeping a way for arguments to be passed to a
generator before usage, the ability to write generators in item context becomes
possible syntactically:

```rust
generator my_range(start: usize, end: usize) yields usize {
    for i in start..end {
        yield i;
    }
}
```

`yield` is an already reserved keyword, which can be accepted only inside of
`generator` items and closures. Under this design `generator` would have to be
introduced as a new keyword. In the example above, `yields` would also have to
be introduced.

*Note: All of these sytnax suggestions are a work-in-progress.*

### No generalized coroutines

It might have been noted in all the prior examples that there's no surface
syntax to return values into the generator at the `yield` point, nor a way to
express the generator returning a distinct value at the end with its own type.
This is on purpose. Generalized coroutines still have open questions around
their implementation, but the most common use case of representing iterators
and streams do not require either of these two features. We make an effort to
keep the syntax open for the addition of them, but they are left as follow up
work.

##### Open Questions

* Are we closing the door on `yield`-return and return values?
* Will a potential future generalized coroutine syntax be needed?
* Should `Stream` implementations have their own syntax, beyond `async` header?

##### Tests


# How We Teach This
[how-we-teach-this]: #how-we-teach-this

Generators are a common feature in multiple languages. New documentation will
be written explaining how to write `Iterator`s and `Stream` combinators using
the new syntax.

# Drawbacks
[drawbacks]: #drawbacks

Generalized coroutines are a strict super-set of this feature. Focusing only on
the most pressing use-case might cause syntactic inconsistencies if we ever
introduce the more general feature.

# Alternatives
[alternatives]: #alternatives

*To be updated after T-lang design meeting.*

# Unresolved questions
[unresolved]: #unresolved-questions

*To be updated after T-lang design meeting.*
