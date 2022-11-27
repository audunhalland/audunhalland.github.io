+++
title = "Unimock 0.4"
date = 2022-11-24
[taxonomies]
categories = ["Programming"]
tags = ["rust", "testing", "unimock"]
+++

[Unimock](https://github.com/audunhalland/unimock) ([docs](https://docs.rs/unimock/latest/unimock/)) is a trait mocking library.
Its defining feature is that all generated mock implementations are implemented for the same type ([`Unimock`](https://docs.rs/unimock/latest/unimock/struct.Unimock.html)).
This design allows using a mock object where the type of the generic value is constrained by several trait bounds at the same time (e.g. `T: Foo + Bar`).

Other features:

* Declarative verification of calls up front (executed at `drop`-time)
* Argument matching through pattern matching or `Eq`
* Partial mocking (though this needs to be manually integrated with some "canonical implementation")
* Versatile support for different types of return values, including borrowed values
* Largely implemented via generics, the macro expansions keep the size of generated code to an absolute minimum
* Safe Rust™

## Version 0.4
Version 0.4 contains many improvements.
First and foremost, it introduces more internal traits in order to expose a simpler yet more powerful API to end users.

### Mock construction
There is now less boilerplate involved when instantiating a new mock object:

```rust
let unimock = Unimock::new((
    FooMock::foo
        .next_call(matching!(1))
        .returns(Some("string")),
    BarMock::bar
        .next_call(matching!(2))
        .returns(Ok(42)),
));
```

`Unimock::new` accepts anything that implements `Clause`, which includes tuples up to 16 elements.
This eliminates the manual type-erasure step needed in version 0.3 (having to call `.in_order()` on each terminal clause).
There is a tradeoff though: Since the tuple elements all have different types, the compiler has to do more work.
In this case I do think that improved ergonomics are worth it.

### More versatile return types
In version 0.3, it wasn't possible to specify an output value for a function returning `-> Option<&T>`.
This type is really a mix between an outer owned value and an inner borrowed value.

Version 0.4 introduces new abstractions to be able to represent this.
It includes specific support for `Option<&T>` and `Result<&T, E>`.
The longer term plan is to be able to `#[derive]` these capabilities for custom user-defined types.

Example:

```rust
#[unimock(api = FooMock)]
trait Foo {
    fn foo(&self, arg: &str) -> Option<&String>;
}

let u = Unimock::new(
    FooMock::foo
        .next_call(matching!("a"))
        .returns(Some("b".to_string()))
);
```

here, unimock automatically converts the `Option<String>` into an `Option<&String>` internally.

### Improved diagnostics on matching! failures
Unimock will now try to diagnose what exactly went wrong when a mismatch happens.

Example

```rust
#[unimock(api = FooMock)]
trait Foo {
    fn foo(&self, arg: &str);
}

let u = Unimock::new(FooMock::foo.next_call(matching!("a")).returns(()));

<Unimock as Foo>::foo(&u, "b");
```

Running this code will produce the following terminal ouput:
```
thread 'foo::mismatch' panicked at 'Foo::foo("b"): Method invoked in the correct order (1), but inputs didn't match Foo::foo("a") at tests/foo.rs:42. 
Pattern mismatch for input #0 (actual / expected):
Diff < left / right > :
<"b"
>"a"
```

Except only better:
Unimock uses [pretty_assertions](https://docs.rs/pretty_assertions/latest/pretty_assertions/) to write this diff to the terminal, so diffs are are actually printed with colored output.

Note that the `matching!()` macro uses pattern matching by default.
Printing a `Debug`-based diff on a pattern mismatch might not produce a good diff output, since patterns could be much smaller than the actuall value due to spreads or other variations in syntax.

`matching!()` now also supports matching using `Eq`!
Just write `matching!(eq!(arg0), eq!(arg1))` to match against full values instead of patterns.
If the values implement `Debug`, you will likely get a high quality diff.

### GATs to reduce the amount of generated code
Unimock uses a lot lifetime-generic data type abstractions.
Although these constructs could be expressed in older versions of Rust,
    Generic Associated Types allow unimock to be much more compact,
    greatly reducing the amount of generated code.

### Other changes

See the [unimock changelog](https://github.com/audunhalland/unimock/blob/main/CHANGELOG.md#040---2022-11-20) for other changes in 0.4.

## The future of unimock
I intend to continue to support unimock, and attempt to actively dogfood it for internal projects at work as much as possible.

Unimock is still a project in early development, and I continue to try and find use cases that demonstrate its potential shortcomings.

A very recent experiment I did was to try and generate a mock implementation for `std::io::Read` (and `Write`).
There I hit a new roadblock, because of the signature of [`Read::read_vectored`](https://doc.rust-lang.org/std/io/trait.Read.html#method.read_vectored).
In this signature, the object receives/borrows a parameter with an internal lifetime bound to `'self`.
Unimock's traits aren't yet able to express this.

Some more upcoming changes to core traits are to be expected, for unimock to be able to work with more interesting function signatures.