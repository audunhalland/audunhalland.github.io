+++
title = "Testability: Reimagining OOP design patterns in Rust"
date = 2022-04-30
[taxonomies]
categories = ["Programming"]
tags = ["rust", "testing"]
+++

I'm going to look at design patterns that enables easy code testability. Most of the code
I write at work is written in a traditional object-oriented language, and is almost
exclusively using the pattern known as **Dependency Injection** (DI).

Quick outline:

1. A look into how DI is used in the field, in development environments considered more orthodox than Rust
2. A look at typical industry-standard Rust code with questions like _is it well tested?_ If not, _Is it easy to change that_?
3. Look at how good patterns from other languages can be brought to Rust, *without also taking the idioms that are foreign to Rust*. Take **only** the things tht actually provide value.

This is the first in a series of posts, with the end goal of presenting a full ergonomic design
pattern for Rust applications that enables easy testability everywhere.

### Testing terminology
We have some _application_ (a structured computer program) which is fairly modularized. The most interesting module of a modern program is the _function_.
We would like to verify that the functions do what they are supposed to do, so we write a _test program_ - a meta program that links
to the original program. The test program calls the functions of the original program with some inputs, and execute some assertions on
the outputs.

If things only were that simple! We forgot that an _application_ usually has very deep call graphs. If we test a very high level
function, it will usually call into layer upon layer of intermediate steps before returning an answer, and sometimes it will even perform I/O.

#### Unit tests
Functions are units that represents some computation. When we test a function, we'd like to test _only_ that function,
_the code within that function body_. We don't (necessarily) want to test everything else that the function directly or indirectly calls out to.

#### Business logic vs. utilities
I like to group functions into these two categories. Business logic is deep, but utilities are shallow. An example of a utility
in the context of an application, could be e.g. `HashMap`. If the function we are testing uses a HashMap, we just let it do so, because it's
not considered a Business Logic dependency, it's just a utility.

#### Inversion of control
Business logic will usually depend on _lower level_ business logic. It is typically this kind of dependency we want to "cut off" when
executing a unit test. In the test, we want to treat the _output_ of the lower level dependency (`B`) as _input_ to the tested function (`A`).

Could we just rewrite the function? Instead of _calling_ `B` directly for its output, `A` could just accept it
as an additional parameter? No, because `A` would then become a _utility_! There has to be _some code somewhere_ that connects
the output of `B` to the input of `A`. We would see that `A` is on the call stack while `B` is executing. Therefore, it must be modelled
as a function call.

So the control flow goes from `A` to `B` and then back to `A`, with `B`'s answer. But only in release mode. In the unit test
we'd like to just specify what `B`s output is, we don't want the call to happen for real. Something external to `A` needs to
specify whether the real call to `B` will happen or not. This is inversion of control (IoC).

There are different ways of implementing IoC. It can use function pointers, it can use method dispatch, it can use a special configuration parameter.

#### Dependency Injection
Dependency Injection (DI) is an implementation of IoC. It usually builds on some kind of method dispach. The `B` from before could be some
object named `b`, with the method `b`, so `A` could call `b.b()`, and since the method call is being dispatched, `A` has not hard-coded
its dependency. The _injection_ is to pass a `B` _object instance_ into `A`.

In modern field practice, dependencies are _code modules_ with _dispatch functionality_. What I mean by that is that one `B` have many methods that
can be called, instead of just one. In strictly object oriented languages like Java, every function is part of a class, and most of them
are typically methods. Java has one class per file, so it will feel natural to put _related methods_ inside _the same class_. Then it will also
feel natural to pass around instances of those classes to create dependency graphs.

I am not sure whether passing a function pointer would classify as DI.

#### Mocking
Mocking, broadly, is the practice of injecting _test doubles_ as surrogates for real dependencies when unit tests are executed.

Java/JVM, which I'm most familiar with, has various mock libraries (Mockito, mockk, etc) that utilize reflection and runtime bytecode generation
to create alternative implementations based on _concrete classes_ at _runtime_. That's right, dependencies do not need to be abstract
interfaces for this to work.

### Java application architecture
Java was once purely Object Oriented, but has moved towards a more functional style in more recent years. Other, "secondary" JVM languages
have accelerated this move. But a typical modern Java (or Kotlin) program is still object oriented, and what I'm thinking of
is of course Dependency modelling, and all of the Domain-Driven-Design names of classes, like `FooController`, `FooService`
and `FooRepository`. There might be a setup like this:

```java
class FooController {
    FooService fooService;
}

// another file
class FooService {
    FooRepository fooRepository;
}
```

This is a dependency graph. The classes accept their dependencies as parameters in the constructor. In the running application,
they get instantiated once, and live for the rest of the program's lifetime.

This architectural style is ubiquitous, it's an industry standard and accepted best practice.

To recap:
* Business logic calls are _dispatched_ via a method call to through a reference to another class instance.
* Utilities are just called/instantiated direcly, no dispatch.

#### Java testing
Let's look at the consequence of this pattern in unit tests. We want to _mock_ dependencies.
We want to test a method of a class, and that class receives dependencies through its constructor.
There could be _many_ dependencies, and since we run in a test, it's common to mock out
those dependencies. For example:

```java
FooService service = new FooService(
    mock<BarService>(),
    mock<BazService>(),
    mock<QuxService>(),
    mock<WyfService>(),
);
```

I.e. we create 4 mock objects. So far so good, continuing with:

```java
int output = service.someMethodThatWeWantToTest(1, 2, 3);
// assertEquals(42, output);
```

Does that method call use all those 4 dependencies? We don't really know without looking at its implementation.
`FooService` could be a large class
with a hundred methods that are "somewhat related". We still had to mock 4 dependencies.

Of those 4 dependencies, what _methods_ of those dependencies are used within `someMethodThatWeWantToTest`
(There _could_ be a hundred methods in each)?
Also not easy to know without looking at the actual implementation.

What we have is something that works OK, but in many cases look like over-abstractions, in my view.
I'm talking about the whole Dependency vs. method _inside_ Dependency duality. In fact, it is the
_called methods_ which are the **real** dependencies, not the class that contains those methods.
The dependency to the class instance may be seen as a _hack_ to get method dispatch working.

But if Java didn't do it that way, there would be 10 times more classes with just a single method
in each. And if you wanted to have a method that calls 10 other methods, you'd need to pass
10 different dependency references into it. And those things aren't free!

We will see if it is possible to improve a bit on this when we come to the Rust section:

### Rust application architecture

Rust and the JVM languages have very different designs. In Java you can "hack" the language
during runtime to enable things that the language wasn't really designed to do. Trying to do something
like that with Rust would be close to crazy. Rust has no reflection, so it is much
more limited in the things it might be able to infer during runtime. Rust only does exactly what the
code says, with very few exceptions.

That is not to say we cannot have IoC in Rust, we very much can, and the go-to language feature for
that is `trait`. In fact we can have zero-cost IoC. So whatever testing design we come up with, it should
not have performance impact on the finished product. We only pay for what we use, and
a release build does not include the tests.

Altough we can use traits to achieve IoC, it's very much opt-in. A standard method call in Rust
is not dispatched. It's only dispatched if some kind of _generics_ are involved (a zero-cost
abstraction), and when `dyn` is used (which is not completely zero cost).

#### Testing philosophy in Rust
I think that Rust codebases have traditionally leaned more upon integration testing than unit testing.
The unit tests that I tend to see there, are usually tests for _utilities_. Unit-testing
utilities is easy.

Business logic seems to get less attention. I don't know the answer why, but it _might_ be because
there aren't any established design patterns.

#### Trying to port the OOP patterns directly

First, we can imagine something like this:

```rust
trait FooController {
    fn handle_something(&self);
}
```

This is an interface that we need to implement for a type.

```rust
impl FooController for FooControllerImpl {
    fn handle_something(&self) {
        // ... the implementation
    }
}
```

Trying to model DI could look something like this:

```rust
struct FooControllerImpl {
    foo_service: Box<dyn FooService>,
}
```

But I think there there are several code smells already.

1. It's not zero cost
    * It requires a heap allocation
    * It uses `dyn`, and therefore dynamic dispatch through a vtable.
2. It looks Object Oriented, which is _probably_ not the correct paradigm
3. It's a little verbose, tedious to write.
4. It inherits the many-methods-in-many-dependencies design from OOP

To mitigate `1.`, we could make a static dispatch solution:

```rust
struct FooControllerImpl<F> {
    foo_service: F,
}
```

But Instead I really want to break this properly apart and try to lose all these "wannabe-class" things.

#### Modules

In Java, a code module/file always equals a `class` which somehow needs to be instantiated. Rust just
has normal code modules, and you usually group together related types, impls, traits and functions
and put them in the same module. You don't instantiate a Rust module.

#### Functions or methods?
In Rust, it is not always clear whether you should make some computation a _function_ or a _method_.

If it is very clear what the _subject_ is that you are operating on, you should probably create a method.
But a design like the following will appear contrived to many Rust developers:

```rust
struct FooServiceImpl {
    bar_service: ?
}

impl FooServiceImpl {
    fn do_something(&self) {
        self.bar_service.do_something_else();
    }
}

struct BarServiceImpl {}

impl BarServiceImpl {
    fn do_something_else(&self) {
    }
}
```

Types (`struct`, `enum`) should be used to represent the _data types_ in the domain of the
application. A type like `FooServiceImpl` is no such thing, and my view is that these have no place
in a Rust program. I once developed a Rust application at work where I used a Service-oriented
design, and I wasn't fond of the end result at all. It felt very unnatural to work with.
These OOP patterns just don't tend to fit well with Rust.

I'd now like to introduce a design based on combining functions and traits, to achieve _static, zero-cost dependency injection_.

### zero-cost DI with functions and traits: The _"deps-pattern"_
The idea is that some function `a` has a generic parameter called `deps` that declares all its
dependencies as a union of trait bounds.

An abstract computation that can be depended upon is declared as a trait:

```rust
trait B {
    fn b(&self);
}
```

The function which depends on `B`, looks like this:

```rust
fn a(deps: &impl B) {
    deps.b();
}
```

(`impl B` is a shorthand notation, it means being generic over some type that implements the trait `B`)

Expanding a bit, let's write out the full declarations of both `A`  and `B`:

```rust
trait A {
    fn a(&self);
}

fn a(deps: &impl B) {
    deps.b();
}

trait B {
    fn b(&self);
}

fn b<T>(deps: &T) { // (T is unbounded, as it's not used)
    unimplemented!()
}
```

You will notice that the `fn a(&self)` in `A` and `fn a(deps: &impl B)` have similar signatures.
The difference between them is that the trait exports _itself_ as a dependency, while the function is
the implementation of that, and declares its _own_ dependencies.

There's still something missing in this picture though, we don't have a working application yet!
There is nothing that makes `a` _actually_ call `b`.

To get there, we need a _type_ which will implement both traits `A` and `B`, with those
implementations acting as the final "linking stage", wiring together  `A` with `a` and `B` with `b`:

```rust
struct Application {
    // "state" the app needs to operate.
    // configuration, connection pools, etc
}

impl A for Application {
    fn a(&self) {
        // call the _function_ a, defined above.
        // This compiles, because Application
        // also implements trait B.
        a(self)
    }
}

impl B for Application {
    fn b(&self) {
        // call the _function_ b
        b(self)
    }
}
```

Neither of the global functions `a` nor `b` depend on the Application directly, just on
their immediate dependencies.

The `Application` struct holds all the data the application needs to operate.

_Leaf nodes_ of the dependency graph will often require access to application state.
Examples of leaf operations could be performing a database operation, other kinds
of I/O or provide some configuration parameter:

```rust
trait GetApiUrl {
    fn get_api_url(&self) -> &str;
}

impl GetApiUrl for Application {
    fn get_api_url(&self) -> &str {
        self.config.api_url
    }
}

trait FetchTodo {
    fn fetch_todo(&self, id: u32) -> Todo;
}

fn fetch_todo(deps: &impl GetApiUrl, id: u32) -> Todo {
    let url = deps.get_api_url();
    some_http_library.get(format!("{url}/posts/{id}"))
}

impl FetchTodo for Application {
    fn fetch_todo(&self, id: u32) -> Todo {
        fetch_todo(self, id)
    }
}
```

An example showing multiple dependencies:

```rust
trait A {
    fn a(&self);
}
trait B {
    fn b(&self);
}
trait C {
    fn c(&self);
}

fn a(deps: &(impl B + C)) {
    deps.b();
    deps.c();
}
```

#### Generalizing the implementations
The drawback with having a type like `Application` used everywhere is that each trait implementation needs
to be aware of the final type that's going to be used. This would make it a bit harder to modularize
an application into multiple crates, and `Application` should architecturally exist in one of the most downstream
crates.

Take a look at `A for Application` again:

```rust
impl A for Application {
    fn a(&self) {
        a(self)
    }
}
```

This implementation contains no code that actually depends on `Application`. It just calls the generic function `a`.
The only reason `Application` is used here is that we need something to dispatch from in the first place. We should _generalize_ these
implementations, so they can live in upstream crates!

In order to be able to mock our trait _and_ use its real implementation, `A` must be implemented for two distinct types. Implementing
it specifically for a mock type and generally for every other type would be disallowed by Rust coherence rules.

What we need is a _Generic Implementation-Providing Smart Pointer™_.

```rust
struct ImplRef<'t, T: ?Sized>(&'t T);
```

Every crate would depend on a common micro-crate that just provides this type.
The impl of function `a`, as the trait `A`, depending on trait `B`, would then look like:

```rust
impl<'t, T> A for ImplRef<'t, T>
    where ImplRef<'t, T>: B,
        T: Sync
{
    fn a(&self) {
        a(self)
    }
}
```

It's implemented for _any_ `T`! But what is `T`? `T` is the immutable state  of the appliction.

Now we can also write an `AsImpl` trait with an `as_impl` method that has a blanket impl for any type, so we could write this:

```rust
struct AppState {
    // ... things go here
};

fn entry_point(state: AppState) {
    state
        .as_impl() // -> ImplRef<'_, AppState>
        .a(); // Call trait method `A::a`, which again calls `fn a`
}
```

`AppState` now gets generically "tunneled" through the system from the entry point
to dependency leaf nodes. We can "dig out" the AppState at the other side, keeping
the rest of the application generic:

```rust
trait INeedAppState {
    fn i_need_app_state(&self);
}

// non-generic function
fn i_need_app_state(app_state: &AppState) {
    // lowest level of business logic,
    // bottom of the call stack.
    // From here, only utilities are used.
    // e.g. a HTTP library
}

impl<'t> INeedAppState for ImplRef<'t, AppState> {
    fn i_need_app_state(&self) {
        i_need_app_state(self)
    }
}
```

Having this impl in the leaf position, requires all of the application
to depend on `AppState`. But `ImplRef` can be _projected_, so we could
call into sub-crates of our application:

```rust
fn i_need_app_state(app_state: &AppState) {
    // Call into the "storage" module:
    // This is now a new sub-entry-point.
    app_state
        .storage
        .as_impl()
        .fetch_something(); // call into a deps-pattern subsystem
}
```

#### Real-world backend application
Integrating the "deps-pattern" into a real world backend will be very easy.

We'll use an [Axum](https://docs.rs/axum/latest/axum/) handler as an example:

```rust
let app_state = Arc::new(AppState {
    /* all the required configuration and state, etc */
});

let axum_app = Router::new()
    .route("/", get(handler_as_entry_point))
    .layer(Extension(app_state));

async fn handler_as_entry_point(
    Extension(state): Extension<Arc<AppState>>,
) {
    state
        .as_impl()
        .my_top_level_business_logic()
        .await
}
```

Note that in an `async` context, the deps-pattern wouldn't be free, because
futures would need to be boxed since they are defined through a trait.
But fortunately, that is [soon about to change](https://github.com/rust-lang/rust/issues/91611) for the better.

### Conclusion: Evaluating the deps-pattern

We have managed to flatten our earlier dependency graph (which was based on
an _actual_ in-memory graph of references) to something where the dependency
graph is just a compile-time concept.

We have gotten rid of unnecessary and contrived types from the application,
and can continue to model it with normal functions. We can have _high level_
business logic depend on _low level_ business logic, and we have Inversion of Control
since it's all based on trait bounds. And (almost) everything should be zero cost.

In unit tests, we can provide alternative implementations of the traits
depended upon.

_There is one great disadvantage though._ Can you guess it? It's the verbosity and all the boilerplate code.
The final pattern that I present will solve that isssue. Yes,
it will involve the use of macros! And yes, that will be presented in the next post,
because this is the end of this one.
