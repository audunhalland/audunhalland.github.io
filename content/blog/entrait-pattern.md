+++
title = "The entrait pattern"
date = 2022-05-28
[taxonomies]
categories = ["Programming"]
tags = ["rust", "testing"]
+++


[Last time](/blog/testability-reimagining-oop-design-patterns-in-rust/) I presented an idea for a design
pattern for Rust applications, that can help with testing of business logic. In short, we need a way to
turn dependencies into _inputs_ when we run tests, so that we can test function bodies as _units_.

The last blog post presented some workable ideas, which suffered a bit from being too verbose to write by hand. This time
I will write about a macro that I named
[`entrait`](https://docs.rs/entrait/0.3.0-beta.0/entrait/index.html),
which removes this boilerplate. _The entrait pattern_ is a certain code style and design technique
that is used together with the macro.

To start from the beginning: The main problem we would like to tackle is _unit testing_ of business logic:

```rust
fn my_function() -> i32 {
    your_function() + some_other_function()
}
```

We would like to be able to write a test for `my_function` that does not depend on any
implementation details from `your_function` or `some_other_function`. Instead we'd like to
treat these functions differently only when we're testing: We want to explicitly specify
what these functions are _returning_, as another kind of input to the function we are _testing_.

## Entrait
_Entrait_ is just a word that I might have invented (not sure!).
It means to _put/enclose something in a trait_, and this is just what the macro does.
When you have written a regular
function, you can annotate it with `entrait` to automatically generate a _single-method trait_ based on its signature:

```rust
#[entrait(MyFunction)]
fn my_function() {
}
```

The entrait macro operates in _append only_ mode, so the original function is not changed in any way and is outputted verbatim.
The generated code that gets appended, includes the trait definition:

```rust
trait MyFunction {
    fn my_function(&self);
}
```

along with a generic implementation for [`implementation::Impl`](https://docs.rs/implementation/latest/implementation/struct.Impl.html):

```rust
impl<T> MyFunction for ::implementation::Impl<T>
    where T: Sync
{
    fn my_function(&self) {
        my_function() // invoking our original function
    }
}
```

This is the very basics of entrait.


## Specifying dependencies
The basic entrait usage is not that interesting in itself. The pattern becomes more interesting when we
hook up different traits, and make one function depend on another set of functions.

Consider a Rust _method_, which has a special `self`-receiver as its first argument. Entraited functions
work in a similar way, but instead of the first parameter being a self-receiver, it specifies _dependencies_.
The dependencies of an entraited function is a set of traits. We can specify them in the following way:

```rust
#[entrait(MyFunction)]
fn my_function(
    deps: &(impl YourFunction + SomeOtherFunction),
    some_arg: i32
) {
    deps.your_function(some_arg);
    deps.some_other_function();
}
```

The macro understands the type syntax of the first parameter, and will adjust its default
implementation bounds (for `Impl<T>`) accordingly. The only thing we need to know for now, is that `deps` will be
a reference to some type on which we can call the methods `your_function` and `some_other_function`.

Using the dependency notation, we can easily build up complex directed dependency graphs with very little code:

```rust
#[entrait(Foo)]
fn foo(deps: &impl Bar, arg: i32) -> i32 {
    deps.bar()
}

#[entrait(Bar)]
fn bar(deps: &(impl Baz + Qux), arg: i32) -> i32 {
    deps.baz(arg) + deps.qux(arg)
}

#[entrait(Baz)]
fn baz(deps: &impl Qux, arg: i32) -> i32 {
    deps.qux(arg) * 2
}

#[entrait(Qux)]
// this function has no dependency bounds:
fn qux<T>(_: &T, arg: i32) -> i32 {
    arg * arg
}
```

## Generics and application state
What we have created so far, is generic on two different levels. First of all,
the function

```rust
#[entrait(MyFunction)]
fn my_function(deps: &impl YourFunction) {
    // ...
}
```

is generic:
1. Because the `deps` parameter can be _any type_ that implements `YourFunction`.
2. Beacuse the trait implementation provided out of the box, `impl<T> MyFunction for implementation::Impl<T>`,
   is defined for _any_ `T`.

The first kind of genericness is covered in the next section, and is related to real vs. fake
implementations and _mocking_.
The second kind of generic parameter, the `T` in `Impl<T>`, is intended to be a _placeholder_ for
the type that we will choose to represent the _state_ of our concrete application.

An application often needs e.g. configuration parameters, connection pools, caches, various data it needs
to operate correctly. An entraited function with generic dependencies can receive any `T` as application state:
```rust
let application: Impl<bool> = Impl::new(true);
application.my_function();
```

Somewhere, usually deep down in our dependency graph, we would like to perform concrete operations on our
chosen application state, for example _borrowing data_ from it. Entrait lets you do that very easily, and
the trick is just to make `deps` be a reference to that concrete type:

```rust
#[entrait(FetchStuffFromApi)]
// still generic:
fn fetch_stuff_from_api(deps: &impl GetApiUrl) -> Stuff {
    some_http_lib::get(deps.get_api_url())
}

struct AppConfig {
    api_url: String,
}

#[entrait(GetApiUrl)]
// concrete:
fn get_api_url(config: &AppConfig) -> &str {
    &config.api_url
}
```

What will happen now, is that the trait `GetApiUrl` will only be implemented for `Impl<AppConfig>`.
This means that `fetch_stuff_from_api`, which depends on a type that implements `GetApiUrl`, in practice
will inherit that same bound. As soon as we introduced a concrete leaf, our whole application became concretized!

## Testing and mocking
So far, we have seen some constructs that enable some degree of abstraction when designing applications.
The way the `deps` parameter specifies bounds as a set of traits is a manifestation of the
[dependency inversion principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle).

The whole point of this in the first place, was to be able to write a unit test for a function.
The function we started with was a `my_function(..) -> i32`
which sums toghether the outputs of `your_function` and `some_other_function`.
Let's write this using entrait:

```rust
#[entrait(MyFunction)]
fn my_function(
    deps: &(impl YourFunction + SomeOtherFunction)
) -> i32 {
    deps.your_function() + deps.some_other_function()
}
```

To test that this function works, we should be able to give it two numbers, e.g. `1` and `2`, and check
that it in fact produces the number `3`. If it did that, we can assume that it performed addition correctly.
We need a way to say that in the test, `your_function` should have the output value of `1`, and `some_other_function`
should have the output value of `2`.

This is where mocking enters the picture. We need _mock implementations_ of the traits `YourFunction` and `SomeOtherFunction`,
and crucially, it must be _one type_ that implements both.

## Unimock
Enter `unimock`, a new mock crate that is designed to be the testing companion to `entrait`. _Uni_ means _one_,
and the core idea is that unimock exports one struct, `Unimock`, which acts as an additional implementation target for
your traits.

In entrait, unimock support is opt-in. The basic entrait usage does _not_ generate a mock implementation:

```rust
use entrait::*;

#[entrait(YourFunction)]
fn your_function() -> i32 { todo!() }
```

The mock implementation is added when entrait is imported from an alternative path:

```rust
use entrait::unimock::*;

#[entrait(YourFunction)]
fn your_function() -> i32 { todo!() }
```

When we write it like that, `YourFunction` will get _two_ generated implementations:

1. for `implementation::Impl<T>`
2. for `unimock::Unimock`

If we entrait `your_function` and `some_other_function` like this, with unimock implementations,
we can easily test `my_function`:

```rust
use unimock::*;

#[test]
fn my_function_should_add_two_numbers() {
    let unimock = mock([
        your_function::Fn::each_call(matching!())
            .returns(1)
            .in_any_order(),
        some_other_function::Fn::each_call(matching!())
            .returns(2)
            .in_any_order(),
    ]);

    assert_eq!(3, my_function(&unimock));
}
```

### Deeper integration tests with entrait and unimock
A testing pattern seen in various OOP languages is mocking out business
logic at an _arbitrary distance_ from the direct function being tested.
I think in some circles this might be referred to as _integration testing_.
Sometimes a deeper integration test is the best strategy for a particular problem.

The real advantage, the _crux_ if you will, about the entrait pattern, is that
both unit and integration tests (of arbitrary depth!) become very much a reality.

Recall that our entraited functions are just ordinary, generic functions:

```rust
fn my_function(deps: &(impl YourFunction + SomeOtherFunction)) -> i32 {
    deps.your_function() + deps.some_other_function()
}
```

`deps` can be a reference to any type that implements `YourFunction` and `SomeOtherFunction`.
`Unimock` can be that type, and we pass it into deps to unit test the function.

What happens when `Unimock::your_function()` is called? Unimock must be configured before it's used.
For example, it can match input patterns in order to find some value to return.

But unimock has another mode, which is called _unmocking:_ Instead of returning a pre-configured value,
it can be instructed to _not mock_, but _call some implementation_ instead. Because we used the entrait
pattern, that implementation is right in our hands, it's that original, handwritten generic function.

There are two main ways to configure a unimock instance:

1. `unimock::mock(clauses)` - Every interaction must be declared up front. If not, you'll get a panic.
2. `unimock::spy(clauses)` - Every interaction is _unmocked_ by default.

A `Unimock` created with `unimock::spy` is an alternative implementation of your entire entraited application.
With that kind of setup, you can start at the other end, i.e. instead of specifying the value of each
dependency to your unit test, you instead say which interfaces to _mock out_. Subtractive instead of additive mocking.

There is one kind of function that cannot be automatically unmocked though. It's those functions that have non-generic
`deps` that live at the leaf nodes of our dependency graph. Those will continue to produce panics
unless they are explicitly mocked.

## Testing an application's external interface
Consider a REST API where we would like to test the interface of the API without invoking the
application's inner business logic. REST handlers in Rust are usually `async fn`s passed to some web framework.
I will present a simple example using `Axum` here:

```rust
async fn my_handler<A>(
    Extension(app): Extension<A>
) -> ResponseType
    where A: SomeEntraitMethod + Sized + Clone + Send + Sync + 'static
{
    app.some_entrait_method()
}
```

We can make a generic Axum `Router` using the same trait bounds, but duplicating these trait bounds
for a lot of different endpoints sounds a bit tedious. A solution to that could be to group related
handlers together in a generic struct, and have the handlers as static methods of that struct:

```rust
struct MyApi<D>(std::marker::PhantomData<D>);

impl<D> MyApi<D>
where
    D: SomeEntraitMethod + SomeOtherEntraitMethod + Sized + Clone + Send + Sync + 'static
{
    pub fn router() {
        Router::new()
            .route("/api/foo", get(Self::foo))
            .route("/api/bar", get(Self::bar))
    }

    async fn foo(
        Extension(deps): Extension<D>
    ) -> ResponseType {
        deps.some_entrait_method().await
    }

    async fn bar(
        Extension(deps): Extension<D>
    ) -> ResponseType {
        deps.some_other_entrait_method().await
    }
}
```

### entrait and `async`
The last example used `async fn`, but async functions in traits are not supported out of the box in current Rust.
The way around that for now is to use `#[async_trait]`. Entrait supports this by accepting a keyword argument list:

```rust
#[entrait(Foo, async_trait=true)]
async fn foo() {
}
```

This has been designed in an opt-in way to make it more visible that we are paying the cost of heap allocation
for every function invocation. When Rust one day supports `async fn` in traits natively, you should be able
to remove this opt-in feature and things should then work in the same manner as synchronous functions.

## Conclusion and further reading
The entrait pattern and related crates are in an experimental state. `unimock` depends on `#[feature(generic_associated_types)]`, so
you will need a nightly Rust compiler to be able to use it (at the time of writing).

What I'm hoping to achieve with this blog post is some feedback on the ideas and current implementation. I'm hoping
some people will find the time to try it out, and maybe find flaws in the design that I might be able to improve.

This blog post does not go into great depth about `unimock` or `entrait`. At the time of writing, both crates are
released on `crates.io`, but only as beta versions, because of the dependency on the nightly compiler.
You will hopefully find much more useful information if you visit respective rustdoc pages for each crate:

| github | crates.io | docs.rs |
|--------------------------------------|---------------------------------|-----------------------------------------------------------------|
| [entrait](https://github.com/audunhalland/entrait) | [0.3.0-beta.0](https://crates.io/crates/entrait/0.3.0-beta.0) | [docs](https://docs.rs/entrait/0.3.0-beta.0/entrait/index.html) |
| [unimock](https://github.com/audunhalland/unimock) | [0.2.0-beta.0](https://crates.io/crates/unimock/0.2.0-beta.0) | [docs](https://docs.rs/unimock/0.2.0-beta.0/unimock/index.html) |
| [implementation](https://github.com/audunhalland/implementation) | [0.1](https://crates.io/crates/implementation) | [docs](https://docs.rs/implementation)|

My next plan for entrait is to develop a full-fledged example application. I will likely put it in the `examples/` directory in the entrait repository.
Meanwhile, you can take a look at the [tests/](https://github.com/audunhalland/entrait/tree/main/tests) directory, which already contains a handful of
good examples.

At least I hope that you found some of this to be interesting to read. I surely had an interesting time developing it and writing about it!
