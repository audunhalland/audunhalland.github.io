+++
title = "Unimock 0.6: Mutation patterns"
date = 2024-04-26
[taxonomies]
categories = ["Programming"]
tags = ["rust", "testing", "unimock"]
+++

[Unimock 0.6](https://docs.rs/unimock/0.6/unimock/index.html) is just out, with an important change in design which makes it much more powerful than before.

The previous version (0.5) added support for parameter mutation, but it was quite limited.
The reason for the limitation was an inferior design that took a significant amount of trial and error to finally fix.
In hindsight it might seem obvious, but for some reason it wasn't to me.

_TL;DR_: Unimock's `answers` API is now based on an associated type instantiated to a `dyn Fn`.

## Background
Unimock allows developers to define flexible one-off verifiable trait implementations with very little code[^1]:

```rust
#[unimock(api = SomeTraitMock)]
trait SomeTrait {
    fn my_func(&self, input: i32) -> i32;
}

let u = Unimock::new(
    SomeTraitMock::my_func
        .next_call(matching!(42))
        .returns(10)
);

assert_eq!(u.my_func(42), 10);
```

It also supports invoking a user-supplied function to compute the output dynamically:

```rust
let u = Unimock::new(
    SomeTraitMock::my_func
        .next_call(matching!(42))
        .answers(|input| input * 2) // <-- note: The old 0.5 API
);

assert_eq!(u.my_func(42), 84);
```

But a problem appeared when I wanted to provide a mock integration for e.g. `Display`:

```rust
trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}
```

`Display` implementors supply behaviour by interacting with the mutable parameter `f`.
The return value is only used for reporting errors, and can be considered secondary.

As soon as the `answers` function needed to mutate parameters, it ran into problems with the type system.

## The old 0.5 model

Unimock's function model was loosely based on a trait like this:

```rust
trait MockFn {
    type Inputs<'i>; // 'i = lifetime of borrowed values in an inputs tuple
    type Output<'u>; // 'u = lifetime of output borrowed from Self
}
```

Using this model, it is quite easy to define a function bound like `Fn(F::Inputs<'i>) -> F::Output<'u>`.
But as soon as mutation is involved, this signature is not going to cut it, because the set of involved lifetimes is not finite anymore.

Version 0.5 ended up using a hack where _one_ parameter could be mutated.
That `&mut` parameter (if present) was excluded from the the `Inputs<'i>` tuple, and handled completely separately from all other inputs.
This meant that it wasn't possible to use the `matching!`-macro on that parameter.
The `matching` macro operates on immutable views of the function inputs, where a single, common lifetime parameter cuts it (because matching only reads things and does not return anything).

Given these limitations it became clear that this wasn't very _developer friendly_.

## 0.6: A better model for mutation patterns
After trying many experiments with several predefined possibly-used GAT-lifetimes, I was ready to give up after hitting Rust limitations like not being able to sepcify outlives-bounds involving higher-ranked lifetimes (`for<'a, 'b> where 'b: 'a`).

Then it occured to me that it's possible to take advantage of the compiler's builtin syntax to express exactly what is needed using _implied bounds_:

```rust
for<'a, 'b> Fn(&'a i32, &'b mut Foo<'_>) -> Bar<'a>
```

The `Fn` _syntax family_ expresses this perfectly, but what the `MockFn` trait needs is an associated _type_, not another trait bound (the `MockFn` is implemented for different function signatures!).

The answer to that is to use `dyn Fn`:

```rust
trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}

// #[unimock]-generated code:
mod DisplayMock {
    struct fmt;
}

impl MockFn for DisplayMock::fmt {
    // ...
    type AnswerFn = dyn (
        for<'u> Fn(
            &'u crate::Unimock,
            &mut core::fmt::Formatter<'_>
        ) -> core::fmt::Result
    ) + Send + Sync;
}
```

The upside is that all mutation patterns are now possible, but the downside is that `dyn Fn` is a type instead of a bound.

The unimock user has to supply a value of this type into the `answers` combinator:

```rust
// defined as:
impl .. {
    // note: F::AnswerFn (i.e. dyn Fn) must be an unsized type:
    fn answers(self, answer_fn: &'static F::AnswerFn) { .. }
}

let u = Unimock::new(
    DisplayMock::fmt
        .next_call(matching!(_))
        .answers(&|_, f| write!(f, "mocked!"))
);
```

This forces the user to pass a function reference rather than any closure that just happens to implement a given `Fn` signature like it did before.

Closure patterns are possible, and unimock supports this, but not in a generic way.
Closures have to be passed using a different combinator:

```rust
impl .. {
    // must take an Arc, because userland functions
    // can't accept unsized types:
    fn answers_arc(self, answer_fn: Arc<F::AnswerFn>) { .. }
}
```

I of course tried to make an abstraction that unites the two APIs, but failed to do so.
I believe the reason for this is that _dyn trait coercion_ can't be abstracted.

##### Footnotes

[^1]: For even more background, and a walk-through of Unimock's overall design: [How to write a type-level mock library in Rust](@/blog/how-to-write-a-type-level-mock-library-in-rust.md).
