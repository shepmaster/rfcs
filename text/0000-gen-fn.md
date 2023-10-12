- Feature Name: `gen-fn`
- Start Date: 2023-10-10
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add `gen {}` blocks to the language. These implement `Iterator` by `yield`ing
elements. This is simpler and more intuitive than creating a custom type and
manually implementing `Iterator` for that type, which requires writing an
explicit `Iterator::next` method body. This is a change similar to adding `async
{}` blocks that implement `Future` instead of having to manually write futures
and their state machines.

Furthermore, add `gen fn` to the language. `gen fn foo(arg: X) -> Y` desugars to
`fn foo(arg: X) -> impl Iterator<Item = Y>`.

# Motivation
[motivation]: #motivation

The main motivation of this RFC is to reserve a new keyword in the 2024 edition.
The feature used by the keyword described here should be treated as an e-RFC for
experimentation on nightly. I would like to avoid discussion of the semantics
provided here, deferring that discussion until during the experimental
implementation work.

Writing iterators manually can be very painful. Many iterators can be written by
chaining `Iterator` methods, but some need to be written as a `struct` and have
`Iterator` implemented for them. Some of the code that is written this way
pushes people to avoid iterators and instead execute a `for` loop that eagerly
writes values to mutable state. With this RFC, one can write the `for` loop
and still get a lazy iterator of values.

As an example, here are multiple ways to write an iterator over something that contains integers,
only keep the odd integers, and multiply all of them by 2:

```rust
// `Iterator` methods
fn odd_dup(values: impl Iterator<Item = u32>) -> impl Iterator<Item = u32> {
    values.filter(|value| value.is_odd()).map(|value| value * 2)
}

// `struct` and manual `impl`
fn odd_dup(values: impl Iterator<Item = u32>) -> impl Iterator<Item = u32> {
    struct Foo<T>(T);
    impl<T: Iterator<Item = u32>> Iterator<Item = u32> for Foo<T> {
        type Item = u32;
        fn next(&mut self) -> Option<u32> {
            loop {
                let value = self.0.next()?;
                if value.is_odd() {
                    return Some(x * 2)
                }
            }
        }
    }
    Foo(values)
}

// `gen block`
fn odd_dup(values: impl Iterator<Item = u32>) -> impl Iterator<Item = u32> {
    gen {
        for value in values {
            if value.is_odd() {
                yield value * 2;
            }
        }
    }
}

// `gen fn`
gen fn odd_dup(values: impl Iterator<Item = u32>) -> u32 {
    for value in values {
        if value.is_odd() {
            yield value * 2;
        }
    }
}
```

Iterators created with `gen` return `None` once they `return` (implicitly at the end of the scope or explicitly with `return`).
See [the unresolved questions][#unresolved-questions] for whether `gen` iterators are fused or may behave strangely after having returned `None` once.
Under no circumstances will it be undefined behavior if `next` is invoked again after having gotten a `None`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## New keyword

Starting in the 2024 edition, `gen` is a keyword that cannot be used for naming any items or bindings. This means during the migration to the 2024 edition, all variables, functions, modules, types, ... named `gen` must be renamed.

## Returning/finishing an iterator

`gen` block's trailing expression must be of the unit type or the block must diverge before reaching its end.

### Diverging iterators

For example, a `gen` block that produces the infinite sequence `0, 1, 0, 1, 0, 1, ...`, will never return `None`
from `next`, and only drop its captured data when the iterator is dropped.

```rust
gen {
    loop {
        yield 0;
        yield 1;
    }
}
```

If a `gen` block panics, the behavior is very similar to `return`, except that `next` unwinds instead of returning `None`.

## Error handling

Within `gen` blocks, the `?` operator desugars differently from how it desugars outside of `gen` blocks.
Instead of returning the `Err` variant, `foo?` yields the `Err` variant and then `return`s immediately afterwards.
This creates an iterator with `Iterator::Item`'s type being `Result<T, E>`.
Once a `Some(Err(e))` is produced via `?`, the iterator returns `None` on the subsequent call to `Iterator::next`.

`gen` blocks do not need to have a trailing `Ok(x)` expression.
Returning from a `gen` block will make the `Iterator` return `None`, which needs no value.
Instead, all `yield` operations must be given a `Result`.

The `?` operator on `Option`s will `yield None` if it is `None`, and require passing an `Option` to all `yield` operations.

## Fusing

Like `Generators`, `Iterator`s produced by `gen` panic when invoked again after they have returned `None` once.
This can probably be fixed by special casing the generator impl if `Generator::Return = ()`, as we can trivially
produce infinite values of the unit type.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
## New keyword

In the 2024 edition we reserve `gen` as a keyword. Previous editions need to use `k#gen` to get the same features.

## Error handling

`foo?` in `gen` blocks desugars to

```rust
match foo.branch() {
    ControlFlow::Break(err) => {
        yield R::from_residual(err);
        return;
    },
    ControlFlow::Continue(val) => val,
}
```

which will stop iteration after the first error. This is the same behaviour that `collect::<Result<_, _>>()` performs
on any iterator over `Result`s

## Implementation

This feature is mostly implemented via existing generators, we'll just need some desugarings and then lots of work to get good diagnostics.

### Gen fn

`gen fn` desugars to the function itself, with its return type replaced by `impl Iterator<Item = $ret>` and its body wrapped in a `gen` block.
So a `gen fn`'s "return type" is in fact its iterator's `yield` type.

A `gen fn` captures all lifetimes and generic parameters into the `impl Iterator` return type (just like `async fn`).
If you want more control over your captures, you'll need to use type alias impl trait when that becomes stable.

Just like all other uses of `impl Trait`, auto traits are revealed without being specified.

### Gen blocks

`gen` blocks are effectively the same as an unstable generator

* without arguments,
* with an additional check forbidding holding borrows across `yield` points,
* and an automatic `Iterator` implementation.

We'll probably be able to modularize the generator impl and make it more robust (on the impl and diagnostics side) for the `gen` block case, but I believe the initial implementation should just be a HIR lowering to a generator and wrapping that generator in `std::iterator::from_generator`.

# Drawbacks
[drawbacks]: #drawbacks

It's another language feature for something that can already be written entirely in user code.

In contrast to `Generator`, `gen` blocks that produce `Iterator`s cannot hold references across `yield` points.
See also https://doc.rust-lang.org/std/iter/fn.from_generator.html, which has an `Unpin` bound on the generator it takes
to produce an `Iterator`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
## Keyword

We could also use `iter` as a keyword. I would prefer `iter` in because I connect generators with a more powerful
scheme than just plain `Iterator`s. The `Generator` trait can do everything that `iter` blocks and `async` blocks can do, and more. I believe connecting the `Iterator`
trait with `iter` blocks is the right choice, but that would require us to carve out many exceptions for this keyword,
as `iter` is used for module names and method names everywhere (including libstd/libcore). It may not be much worse than `gen` (see also [#unresolved-questions]).

One argument for `iter` is also that we may want to use `gen` for full on generators in the future.

## 2021 edition

We could allow `gen` blocks on the 2021 edition via `k#gen {}` syntax.
We can allow `gen fn` on all editions.

## Do not do this

The alternative is to keep adding more helper methods to `Iterator`. It is already rather hard for new Rustaceans to get a hold of all the options they have on `Iterator`.
Some such methods would also need to be very generic (not an `Iterator` example, but https://doc.rust-lang.org/std/primitive.array.html#method.try_map on arrays is something
that has very complex diagnostics that are hard to improve, even if it's nice once it works).

Users can use crates like [`genawaiter`](https://crates.io/crates/genawaiter) instead, which work on stable and give you `gen!` blocks that behave pretty mostly
like `gen` blocks, but don't have compiler support for nice diagnostics or language support for the `?` operator.

## `return` statements `yield` one last element

Similarly to `try` blocks, trailing expressions could yield their element.

But then have no way to terminate iteration, as `return` statements would similarly have to have a
value that needs to get `yield`ed before terminating iteration.

We could do something magical where returning `()` terminates the iteration, so

```rust
gen fn foo() -> i32 {
    42
}
```

could be a way to specify `std::iter::once(42)`. The issue I see with this is that

```rust
gen fn foo() -> i32 {
    42; // note the semicolon
}
```

would then not return a value.

Furthermore this would make it unclear what the behaviour of

```rust
gen fn foo() {}
```

is supposed to be, as it could be either `std::iter::once(())` or `std::iter::empty::<()>()`


# Prior art
[prior-art]: #prior-art

## Python

Python has `gen fn`: any function that uses `yield` internally.
These work pretty much like the `gen` functions proposed in this PR. The main difference is that raising an
exception automatically passes the exception outwards, instead of yielding an `Err()` element.

```python
def odd_dup(values):
    for value in values:
        if is_odd(value):
            yield value * 2
```

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Keyword

Should we use `iter` as a keyword instead, as we're producing `Iterator`s.
We can also use `gen` like proposed in this RFC and later extend its abilities to more powerful generators.

[playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=efeacb803158c2ebd57d43b4e606c0b5)

```rust
#![feature(generators)]
#![feature(iter_from_generator)]

fn main() {
    let mut it = std::iter::from_generator(|| {
        yield 1
    });

    assert_eq!(it.next(), Some(1));
    assert_eq!(it.next(), None);
    it.next(); // panics
}
```

## Panicking

What happens when a `gen` block that panicked gets `next` called again? Do we need to poison the iterator?

## Fusing

Should we make `gen` blocks fused? Right now they'd panic (which is what the generator impl does):

## Contextual keyword

Popular crates (like `rand`) have methods called `gen` (https://docs.rs/rand/latest/rand/trait.Rng.html#method.gen). If we forbid those, we are forcing those crates to make a major version bump when they update their edition, and we are requiring any users of those crates to use `r#gen` instead of `gen` when calling that method.

We could instead choose to use a contextual keyword and only forbid

* bindings,
* field names (due to destructuring bindings),
* enum variants,
* and type names

to be `gen`. This should avoid any parsing issues around `gen` followed by `{` in expressions.

# Future possibilities
[future-possibilities]: #future-possibilities

## `yield from` (forwarding operation)

Python has the ability to `yield from` an iterator.
Effectively this is syntax sugar for looping over all elements of the iterator and yielding them individually.
There are infinite options to choose from if we want such a feature, so I'm just going to list the general ideas below:

### Do nothing, just use loops

```rust
for x in iter {
    yield x
}
```

### language support

we could do something like postfix `yield` or an entirely new keyword, or...

```rust
iter.yield
```

### stdlib macro

We could add a macro to the standard library and prelude, the macro would just expand to the for loop + yield.

```rust
yield_all!(iter)
```

## Full on `Generator` support

We already have a `Generator` trait on nightly that is much more powerful than the `Iterator`
API could possibly be.

1. it uses `Pin<&mut Self>`, allowing self-references in the generator across yield points
2. it has arguments (`yield` returns the arguments passed to it in the subsequent invocations)

Similar (but def not the same) to ideas around `async` closures, I think we could argue for `Generators` to be `gen` closures,
while `gen` blocks are the simpler concept that has no arguments and just captures variables.

Either way, support for full `Generator`s should (in my opinion) be discussed and implemented separately,
as there are many more open questions around them than around just a simpler way to write `Iterator`s.

## `async` interactions

We could support using `await` in `async gen` blocks, similar to how we support `?` being used within `gen` blocks.
This is not possible in general due to the fact that `Iterator::next` takes `&mut self` and not `Pin<&mut self>`, but
it should be possible if no references are held across the `await` point, similar to how we disallow holding
references across `yield` points in this RFC.


## self-referential `gen` blocks

There are a few options forward:

* Add a separate trait for pinned iteration that is also usable with `gen` and `for`
    * downside: very similar traits for the same thing
* backwards compatibly add a way to change the argument type of `Iterator::next`
    * downside: unclear if possible
* implement `Iterator` for `Pin<&mut G>` instead of for `G` directly (whatever `G` is here, but it could be a `gen` block)
    * downside: the thing being iterated over must now be pinned for the entire iteration, instead of for each invocation of `next`.

## `try` interactions

We could allow `try gen fn foo() -> i32` to actually mean something akin to `gen fn foo() -> Result<i32, E>`.
Whatever we do here, it should mirror whatever `try fn` will mean in the future.
