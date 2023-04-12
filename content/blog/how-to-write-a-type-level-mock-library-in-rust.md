+++
title = "How to write a type-level mock library in Rust"
date = 2023-04-11
[taxonomies]
categories = ["Programming"]
tags = ["rust", "testing", "unimock"]
+++

[Unimock 0.5](https://docs.rs/unimock/latest/unimock/) is just out, and I wanted to reflect on how it came to be, how its design emerged and various implementation challenges along the way.

## Why unimock exists
Rust already has a number of mocking solutions, like the popular [mockall](https://docs.rs/mockall/latest/mockall/). Why another one?

The idea behind unimock comes from the observation that Rust traits form some kind of graph:

```rust
// There is a supertrait relationship between Foo and Bar:
trait Foo: Bar {}
```

```rust
trait A {}
trait B {}

// A and B are related via the fact that some type T needs to implement both:
fn func<T>(t: T) where T: A + B {}
```

To be maximally flexible, a trait mocking library needs to support all combinations of trait bounds.
If we wanted to write a test implementation of `Foo`, we'd also have to write one for `Bar`.
If we want to test `func`, the test type needs to implement both `A` and `B`.

Ultimately, there is only one possibility: All the traits need to be implemented by _the same type_!

This challenge first appeared while I experimented with [entrait](https://docs.rs/entrait/latest/entrait/), which is entirely made up of a trait graph.
I quickly hit a road block when trying to use `mockall` as a test tool, because it uses one type per mocked trait.
The next sections are modelled as a walkthrough on how a universal Unimock-like mocker can be (and is!) implemented, and some of the design challenges are discussed.

By _"type-level" mocker_, I mean a library that relies more on generics, traits and bounds than lots of macro-generated custom code blocks.
Unimock, although it comes with a fairly complex procedural macro that does trait parsing and analysis, is a type-level mocker.
The macro's job is to figure out what types to define!

## Implementing a simple mocker from scratch
Let's start with a simple trait.

```rust
trait Foo {
    fn foo(&self) -> i32;
}
```

We'd like to mock it! We want to start out with a type that can return a configurable `i32`:

```rust
struct Mocker(i32);

impl Foo for Mocker {
    fn foo(&self) -> i32 {
        self.0
    }
}
```

We've created a simple mock library!
But now let's reuse that type for another trait:

```rust
trait Bar {
    fn bar(&self) -> String;
}
```

Now the `Mocker` needs two fields: An `i32` and a `String`.
Going on like this can't scale forever, so we need to look at some more dynamic solutions.

```rust
struct Mocker {
    return_values: HashMap<TypeId, Box<dyn Any>>,
}

impl Foo for Mocker {
    fn foo(&self) -> i32 {
        let ret = self.return_values.get(&TypeId::of::<i32>()).unwrap();
        ret.downcast_ref::<i32>().unwrap().clone()
    }
}
```

This looks more dynamic and using `Any` and `TypeId` seems like a good idea, but there are two problems:
1. All methods that return `i32` has to return the same value.
2. All return types need to be `'static` in order to implement `Any`.

We have to be able to control each trait method individually from any other trait method.
Rust trait methods are not types, and cannot implement any traits (i.e. `Any`).

This can be solved by defining a new type per method:

```rust
// module which represents the trait
mod FooMock {
    // struct which represents the method
    pub struct foo;
}
```

And defining each method's return type through a new trait with an associated type:

```rust
// 'static makes this automatically implement Any
trait MockFn: 'static {
    type Output;
}

impl MockFn for FooMock::foo {
    type Output = i32;
}

struct Mocker {
    return_values: HashMap<TypeId, Box<dyn Any>>,
}

impl Foo for Mocker {
    fn foo(&self) -> i32 {
        let return_value = self
            .return_values
            .get(&TypeId::of::<FooMock::foo>())
            .unwrap();
        return_value.downcast_ref::<i32>().unwrap().clone()
    }
}
```

What's the point of the `Output` associated type? It's not used at all in `impl Foo for Mocker`.

Before the mocker can be used, the user needs to specify which return values each method will have.
It would not be a very good idea if the interface to configure the mocker exposed `Box<dyn Any>`.
So we'll design the configuration API around this associated type, and box the return value internally:

```rust
impl Mocker {
    pub fn should_return<F: MockFn>(&mut self, value: F::Output)
        where F::Output: 'static
    {
        self.return_values.insert(TypeId::of::<F>(), Box::new(value));
    }
}
```

Now we have implemented two phases of the mocking lifecycle: _Configuration_ and _interaction_.
The two phases are galvanically isolated but still type safe by the use of `dyn Any`.

There is still a problem that return values must be `'static`.
This can be solved by using an internal `enum` with two variants:
1. A `Box<Any>` representing a `'static` return value
2. A `Box<Any>` representing a _static closure_ that returns `F::Output` directly (and postpone the problem with what it actually references with its non-static lifetime).

### Returning references
The return values currently supported are owned types, including `&'static T`.
There is no way to put non-static references into that hash map.

What kind of references could we expect to support?
The archetypal pattern is likely some variation of this with the elided self-lifetime:

```rust
trait Foo {
    fn foo(&self) -> &str;
}
```

To support this, the mocker has to store an owned version of the return value and return a reference to it.
This already indicates that the configuration and interaction phases do not necessarily use the same types.
We could configure the mocker with a `String` and then return a `&str` of it later.

```rust
trait MockFn {
    type Output;
    type Response: AsRef<Self::Output>;
}
```

This is a start, but now _all_ outputs must be references of the response, that is not what we wanted.
What we need is a way to tell the mocker the "general category" of output: Owned or Borrowed.
Let's introduce a new trait:

```rust
trait Respond {
    type Type: 'static;
}
```

The `Type` is the owned version of the type that is allowed to be stored in the hash map.

```rust
struct Owned<T>(PhantomData<T>);

struct Borrowed<T: ?Sized + 'static>(PhantomData<T>);
```

These are the two current variants.
To wire things up, we need yet another trait:

```rust
trait Output<'a, R: Respond> {
    type Type;

    fn from_response(response: R::Type) -> Self::Type;
}
```

Let's start by implementing `Owned`:

```rust
impl<T: 'static> Respond for Owned<T> {
    type Type = T;
}

impl<'a, T: 'static> Output<'a, Self> for Owned<T> {
    type Type = T;

    fn from_response(response: R::Type) -> Self::Type {
        response
    }
}
```

These implementations express that an `Output::Type` is an "identity transformation" from `Respond::Type` in the case of `Owned<T>`.
Notice the `'a` lifetime introduced in the `Output` trait.
This lifetime represents borrowing from the mocker, and we will need it when implementing `Borrowed`.

But first, we have to change `MockFn` to use our new traits, and now we'll need a Generic Associated Type:

```rust
trait MockFn {
    type Response: Respond;
    type Output<'a>: Output<'a, Self::Response>;
}
```

The bounds on these types illustrate how `Response` and `Output` are connected.
The output must be an Output that can be constructed from the response, and the output is allowed to be a reference.

Now over to `Borrowed`:

```rust
impl<T: ?Sized + 'static> Respond for Borrowed<T> {
    type Type = Box<dyn std::borrow::Borrow<T> + Send + Sync>;
}
```

The type stored in the hash map is any type from which we can `Borrow` a `T`.
For example, `String` in the case of `&str`. (If we want `Mocker` to be thread safe we need Send and Sync bounds.)

```rust
impl<'a, T: ?Sized + 'static> Output<'a, Self> for Borrowed<T> {
    type Type = &'a T;

    fn from_response(
        response: <Self as Respond>::Type,
    ) -> Self::Type {
        panic!("Dang, cannot return borrow of a local")
    }
}
```

Here is the next spanner in the works.
We want to borrow the `Respond::Type`, but can't, since it is being passed by-value into the `from_response` function.
Now you might object that the `from_response` design is wrong.
Why is it passed an owned response when it could just borrow directly from the mocker's hash table?
Some paragraphs ago we said that the hash table can contain one of two things: An owned response and a closure that produces an owned response.
So we need to handle the ephemeral owned response anyway: Yes, returning a reference to something just produced by a function.

Finding a (safe) solution to this problem was not easy.
I needed some kind of data structure that can convert `T`'s to `&'self T`'s.
It needs to use interior mutability, or else we can't get `&T`'s out.
Once a `T` has been added to the structure, that `T` can't ever be touched until `drop`, or else the `&T` would be invalid.
The end solution involves a [once_cell](https://docs.rs/once_cell/latest/once_cell/) chain, the code can be found in the unimock repo.

In Unimock, there are even more variants of `Respond`/`Output`, the most notable one is called `Mixed<T>`.
`Mixed` is used in Owned/Borrowed tree-like situations, for example with the following signatures:

```rust
trait Foo {
    fn f1(&self) -> Option<&str>;
    fn f2(&self) -> (String, &String);
}
```

### Observing parameters
So in addition to outputs, functions also have inputs.

In a typical mocking situation we want to test some code that calls into a trait.
The trait implementation would like to check that it receives some expected parameter value, and react accordingly.
"Reacting accordingly" is to be understood as "returning some specific output".

This parameter observation business is also something that happens at interaction time, but is specified up front at configuration time.

Let's extend the `MockFn` trait with some inputs:

```rust
trait MockFn {
    type Inputs<'i>;
    // ..
}
```

When the function has multiple inputs it will use a tuple.
The `'i` lifetime is quite handy.
As long as the parameters are immutable, the same lifetime can be used for all of them.
(`&mut` on the other hand, can not be modelled this way. I won't get into this in this article, but it's related to [unexpected higher-ranked lifetime error in GAT usage](https://github.com/rust-lang/rust/issues/100013))

So, the mocker receives some `Inputs` and wants to produce an `Output`.
The user has to control which inputs map to which output.
It's not possible to just put the inputs in any kind of table, because that would require trait bounds.
Instead the user needs to supply a function that receives a reference to the inputs, and returns whether it is a "match" or not.

Something like:

```rust
|inputs| matches!(inputs, (1, "2", 3.0))
```

This closure type can be converted into to `Box<Any>` and stored together with the return values in the hash map.

Except, it's not that easy.
This solution "works", but is quite hard to debug when the user _expects_ a match that for some reason doesn't at runtime.
The mocker could panic with a message that it found no response for the given inputs, but it would be handy to see _why_ there was a mismatch.
(In mocking terminology, a proper mock is something you _expect_ to be called.
If it isn't, that's an error.)

Another annoying issue with the `matches!` approach is the lack of [Deref patterns](https://github.com/rust-lang/rust/issues/87121).
I.e. if one of the inputs is of type `String`, it can't be matched with a `"string literal"`.
This is one of the most annoying missing features in Rust for me personally.
So what I wanted for Unimock was a higher level macro for input matching, that encapsulates the closue syntax (it produces a closure) and has some plumbing that works around some of the missing Deref patterns issues.

The macro is called `matching!`:

```rust
matching!(1, "2", 3.0)
```

It basically expands to a pattern match on the arguments, along with useful diagnostics in the case of an unsuccessful match.
Every string literal is matched using `AsRef<str>`.

The input matching was not the hardest part to design.
Now let's move to the part that is most visible to users.

## The configuration phase
The configuration phase is what happens first (at the top of the test) and really ties all the parts toghether.
It's also the API that users have to suffer through actually using, API ergonomics should be in the front seat!

First some general observations about the mock configuration phase.

* The mocker is either configured or it's not configured.
   This implies some kind of builder pattern, where you are either in the configuration phase or interaction phase, enforced by type state.
* People like to write reusable code. It should be easy to factor out parts of the configuration phase to be reused many times.
* It should not be too verbose.
* It should not be too cryptic.
* Not too much `rustfmt` indentation whitespace.
* `rustfmt` indentation should visually group things in a logical way.

Let's start with a `MockBuilder` first:

```rust
MockBuilder::new()
    .add(FooMock::foo)
    .inputs(matching!(1))
    .returns(2)
    .next()
    .add(..)
    // ...
    .build()
```

This API is not that nice because there is no grouped indentation for one "unit of mock".
It's also too verbose for my taste, needing method calls like `.add` and `.inputs` which don't add information.

Let's look at defining an API for a "unit of mock" first.
Remember the `struct foo` inside `mod FooMock`, the one that implements `MockFn`.
Let's try to put a helper method in `MockFn`:

```rust
trait MockFn {
    fn mock() -> MockFnBuilder { .. }
}

FooMock::foo::mock()
    .inputs(matching!(1))
    .returns(2);
```

Even better:
```rust
trait MockFn {
    fn mock(matching: impl MatchingFn) -> MockFnBuilder<Self> { .. }
}

FooMock::foo::mock(matching!(1))
    .returns(2);
```

Even better (with the help of `rustfmt`):
```rust
trait MockFn {
    fn mock(self, matching: impl MatchingFn) -> MockFnBuilder<Self> { .. }
}

FooMock::foo
    .mock(matching!(1))
    .returns(2);
```

Integrating this into the universal mocker builder:
```rust
MockBuilder::new()
    .mock(FooMock::foo
        .mock(matching!(1))
        .returns(2)
    )
    .mock(..)
    .build()
```

It's a bit better than where we started, but not quite there.
I still don't like the boilerplate-y `.mock` (or `.add`) call that's required here.
Also there's a `.build()` call that's very Builder Pattern, and really doesn't convey anything.

What I'd like is just _one function call_ with one parameter that just returns the mocker.
This can be done using an array!

```rust
struct Mocker { .. }

impl Mocker {
    pub fn new(configs: impl IntoIterator<Item = MockFnBuilder<?>>) { .. }
}
```

Except that `MockFnBuilder` has to be a generic type because it needs to operate on the corresponding `MockFn`'s `Inputs` and `Response` types.

So if we can't use that, maybe the `MockFnBuilder<F: MockFn>` can be converted to a `DynMockFnBuilder` before being passed to the array?
That would still require one extra line per mock config:

```rust
Mocker::new([
    FooMock::foo
        .mock(matching!(1))
        .returns(2)
        .into_dyn() // <-- :(
])
```

Sometimes I wish that Rust could do implicit type conversion..

But not today.
What's nice about the array approach is that the genericity of each `MockFnBuilder<T>` ends with each `.into_dyn()`, and the compiler has an easier job because each array item has the same type (it has to!).
The obvious "solution" to missing implicit type conversion is.. _tuples_.
With tuples we'll end up with a big unique type for every unique mock configuration, the compiler will have a harder time, but this is Ergonomic (and elegant) API design!

```rust
Mocker::new((
    MockFoo::foo
        .mock(matching!(1))
        .returns(2),
    MockBar::bar
        .mock(matching!("string"))
        .returns(3),
))
```

I'm starting to like this!

Now the problem is the corny grammars: "Mock matching 1 returns 2".
I think there's too much use of the word "mock" everywhere.

To get a hint of what the helper function is going to be called, we can analyze the type-state in the mock builder:

1. `MockFoo::foo` a trait method
2. `.mock(matching!(something))` describe the function call
3. `.returns(something)` describe the response

After step 2 we have described a function call, so the "keyword" could include the word "call".
In addition, it would be nice if it sounded as a general rule.
"Each time this function is called with these parameters, it should return this".
`each_call` is nice.
We'll also support `next_call` and `some_call` (the Unimock documentation explains the difference).

```rust
Mocker::new((
    MockFoo::foo
        .each_call(matching!(1))
        .returns(2),
    MockBar::bar
        .next_call(matching!("string"))
        .returns(3)
))
```

And what about `Mocker::new`?
It needs to accept a generic with a trait bound as its only parameter.
That trait will be called `Clause`, and must of course be implemented for tuples of up to N elements.
Using a trait like this makes it easy to factor out common clauses to helper functions.
As long is the trait is implemented for both tuples and the elements inside the tuples, it's possible to build arbitrarily deep clause trees.

I think we have a good enough configuration API now, although all the details are not described here.
These are (very roughly) the steps that were taken before the Unimock API ended up the way it looks now.


## Mock the ecosystem!
Unimock before 0.5 was intended to be used in application development for locally defined traits.
I only recently came to think of rust traits as living inside a big interconnected graph.

So unimock 0.5 introduces mocking of upstream crates from [core](https://doc.rust-lang.org/stable/core/index.html) and [std](https://doc.rust-lang.org/std/).
The most notable ones are `Display`, `Debug`, `Read`, `Write` and `Seek`.

I'm not sure where to to take unimock next.
Should unimock depend on all kinds of third party crates or should it be the other way around?
It's not yet clear to me what's the ideal solution.

I'll close off this post by posting a new fun test case from the repo, demonstrating how `Display` and `Write` works:

```rust
use unimock::mock::core::fmt::DisplayMock;
use unimock::mock::std::io::WriteMock;
use unimock::*;

#[test]
fn test() {
    // All the clauses here use `next_call`
    // and therefore MUST happen in the specified order:
    let mocker = Unimock::new((
        DisplayMock::fmt
            .next_call(matching!())
            .mutates(|f, _| write!(f, "hello {}", "unimock")),
        // NOTE: `write!` calls `write_all` which
        // is a default method that implicitly
        // gets re-routed into `write`:
        WriteMock::write
            .next_call(matching!(eq!(b"hello ")))
            .returns(Ok(6)),
        WriteMock::write
            .next_call(matching!(eq!(b"unimock")))
            .returns(Ok("uni".len())),
        WriteMock::write
            .next_call(matching!(eq!(b"mock")))
            .returns(Ok("mock".len())),
    ));

    write!(&mut mocker.clone(), "{mocker}").unwrap();
}
```