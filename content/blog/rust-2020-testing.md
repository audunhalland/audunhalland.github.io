+++
title = "Rust 2020: Testing"
date = 2019-11-29
[taxonomies]
categories = ["Programming"]
tags = ["rust"]
+++

_originally posted on [Knowitlabs](https://knowitlabs.no/rust-2020-testing-4ab3d80112ba)_

Unit testing. Becoming a master of any programming language totally depends on also mastering its testing tools. Having tests in the code base in the first place increases the overall functional code quality, but when we go a step further and write isolated tests, this also enforces various desirable constraints on our code structure. The call graphs/external dependency graphs of components that are tested in isolation are much easier to reason about. The result is better overall code structure.

Before our team chose to develop a new application using Rust, I had some experience using various degrees of test driven development styles in Python, Spring Boot (JUnit) and node.js (jest). I am by no means an expert at test writing, having adopted this style of development only the last 2–3 years. Projects I worked on before that did not rely as much on TDD, but more on “stunt programming” — let’s just say that I try to be a good professional now.

In this article I’ll write a little about the way I currently think about testing, and then about implementing these ideas in Rust. TL;DR: Not everything works as smoothly as when working with higher level languages, maybe unsurprisingly.


### Code decoupling

To be able to unit test our code (in isolation!) we can follow the design principle of writing loosely coupled code. The code components simply have to be loosely coupled to its dependencies in order for the latter to be controlled in isolated tests. I will get back to why loose coupling is a little more inconvenient to achieve in Rust than in higher-level languages. But first there are different degrees of loose coupling I want to mention, and they will hopefully help to illustrate my points later in the article: 1) Loose coupling for runtime reasons. 2) Loose coupling for testtime reasons. 3) Loose coupling for loose coupling reasons (I will not discuss 3) because I think it is absurd).

I personally think that the way components are decoupled should be done with its primary intent in mind. A de-coupling that enables a runtime abstraction in release mode should be syntactically and semantically more decoupled than a decoupling that just enables dependency control during test.

Let’s look at some decoupling techniques that I know of. Because this article is about testing, I’m thinking about all of these as means to control dependencies during test time.

#### Higher order functions

If I want to test a function or procedure `A` in isolation that depends on another function `B`, I can pass `B` as a parameter to `A`. In production code I call `A(B)`. In the test I call `A(mocked_B)`, now with full control over that dependency.

#### Object oriented “interface”

If A has more than one logical dependency, the signature of the function will have more arguments, and the code will not look as clean. If some of the dependencies are fairly cohesive, we can group several of the dependencies together in an object oriented interface: `B.foo()` and `B.bar()`, where the actual implementation of foo and bar is determined by the concrete type of B, which will be a so-called mock object at test time. Good test tools provide us with ways to quickly configure these mocked functions (or methods), e.g. verifying how many times they are called in the test, recording their input arguments for later verfication or configuring return values.

Within my team it has been debated whether the best style is to exclusively depend on abstract interfaces, everywhere. My personal view is that this should be determined by the _runtime intent_. In high level languages this is usually no problem, because method dispatch is usually dynamic dy default. This means that any or most concrete types or classes are easily sub-classable, and thus the coupling to such a class by name is not really that strong under the hood. All python/Java/Kotlin/JS languages support this easily: Creating mocks of concrete types.


#### Module mocks

There may be of course be many, but Jest is the only testing framework I have seen where it is possible (and fully supported) to mock out entire code modules. You write your module-under-test with all external module dependencies in-place, all imported very directly. In JavaScript, the value of an imported code module is really an object with methods on it, so it’s really no surprise that this can work. Module mocking in jest works by reconfiguring which file the module is loaded from, and the overridden module is active for all tests defined in the same test file. Because a module import is a global “destructive”/permanent operation, each Jest test suite is executed in a sandboxed environment to avoid interfering with other module mock configurations (or non-mocks) elsewhere in the code base.

This technique has the most implicit decoupling. It does not clutter business logic with phony quasi-abstraction-objects (service objects or what you may happen to call them). Another advantageous pattern that can emerge from this is that as more code is tested, modules will sometimes need to be refactored to be able to mock at the correct level for each test. The end result might be better overall module cohesion.

## Rust

Rust is a different beast compared to the aforementioned languages. Being a system language focused on bare metal performance, abstractions do not “come for free”. At least not the ones I’m interested in here. First of all, all function calls are by default dispatched statically. This means that code calling another function directly is compiled into calling the exact implementation of that function, and not by a variable or table lookup. This is true also when calling regular methods of concrete types — struct methods are not overridable.

The two first techniques mentioned above can be applied perfectly fine in Rust, but there is a lot more for the developer to consider (and write!) in each case. The first thing to consider when introducing an abstraction is whether to use static or dynamic method dispatch. Static dispatch is related to monomorphization and generics. It means that a function A calling into an unknown function B is only lazily compiled — only when a call to some concrete B is known does the compiler emit code for A calling that exact B. This behaviour is optional, and we may choose to do dynamic dispatch instead, either by using a function pointer or by using a trait object, a way to call trait methods using a virtual lookup table passed along with the data pointer to the object instance.

So the way to express an abstraction in Rust is to use a trait. Trait mocking in Rust today is fairly developer friendly, but not as friendly as in dynamic-dispatch-languages. What I’ve been doing so far is to introduce a new trait everywhere I need test isolation, and use the crate mockall to autogenerate a mocked implementation that I instantiate in my test. Not very far from how java/mockito or jest works, except that, as mentioned, we always need to be very explicit about the abstraction taking place:

```rust
#[cfg_attr(test, mockall::automock)]
trait SomethingIWantToMock {
    fn mockable();
}

struct ConcreteSomethingUsedInProduction {}

impl SomethingIWantToMock for ConcreteSomethingUsedInProduction {
    fn mockable() { ... }
}

#[test]
fn test_something() {
    let mock_something = MockSomethingIWantToMock::new();
    // test some code using that dependency
}
```

In Rust, making code dynamic dispatch is less verbose than making it static dispatch. This is in some ways counterintuitive, because in other areas of the language it appears to be a concious decision that less verbose code is simpler and therefore more efficient. Heap allocation is one example, it is always very apparent that a heap allocation will take place: Box<MyThing> instead of MyThing. A heap allocation is more expensive to type and execute. It seems concious that there is no short hand for this. First consider dynamic dispatch:

```rust
fn a(b: &dyn B) {
    b.b();
}
```

where B is the trait a is abstract over. Now the static dispatch version of the above:

```rust
fn a<T: B>(b: T) {
    b.b();
}

// or alternatively
fn a<T>(b: T) where T: B {
    b.b();
}
```

To make the function abstract with static dispatch (not losing runtime performance), we needed to introduce two new names. First `B` for the trait itself, and then the name `T` that represents its concrete type when a is monomorphized over it.

Generic code can be a lot of fun to write, but I wouldn’t like it if my codebase is full of it just for test isolation purposes. I want my business logic to be clear and readable when it does concrete things — generic code mostly belongs in utilities. And in principle, I don’t want to make all my code dynamic dispatch just because I want to test it. For all I know the code could be a really performance critical hot path in my application. And I did choose to write it in Rust in the first place.

### Module mocks in Rust

How nice would it be to have module mocks in Rust? My very first thought was that it sounds technically infeasible. All tests in Rust run in a thread pool. In Rust it is forbidden to mutate global state, with good reason. There’s no sandboxing environment like in Jest. All mocked module functions would need to implicitly take a hidden argument — or the whole test executable would need to be “monomorphized” for each individual test case, effectively leading to N copies of the program where N is the number of tests…

But it turns out you can still do this, using the nightly compiler! Enter Mocktopus, a module-level mocking library. Using a macro to transform a module definition, mocktopus turns any public function in that module into a mockable by rewriting its definition into first checking (at call time) if a function pointer has been registered, which is stored in a thread-local lookup table. The macro only takes effect in a test build, not in debug or release. I think this technique could fit well with our project, but trying to be somewhat serious we will not depend on experimental nightly-only Rust features.

Unfortunately, my current solution is that I do not try to make my tests isolated where it is not strictly needed for correctness or where it would lead to awkward-to-read business logic code. The consequence of this is that many places in the code I essentially test the same thing over and over again. But this is also due to a completely different (slight) shortcoming in the Rust testing ecosystem: The basic assertion macros.

### Assertion macros

My test assertions are exclusively variations of `assert_eq!` and `assert_ne!`. There are assertion utility libraries that one could pull in, but none of them seem to be very widely used, there is no clear adopted industry standard on top of std::assert. Let’s go through some very simple code to highlight the slight inconvenience:

```rust
struct Answer {
    foo: Vec<String>,
    bar: Vec<String>
}

fn compute_answer(some_input: &str) -> Answer {
    Answer {
        foo: some_input.split("."),
        bar: external_module::compute_bar(some_input)
    }
}
```

Testing this function in isolation is the same as checking if the input string is split correctly. The test of external_module belongs in a different file, so we should ignore the value of bar in our test. But:

```rust
#[test]
fn test() {
    assert_eq!(
        compute_answer("input.string"),
        Answer {
            foo: vec!["input".to_string(), "string".to_string()],
            bar: vec!["?"]
        }
    );
}
```

using `assert_eq!` I have to compare the full struct. Of course I could compare the member(s) explicitly:

```rust
#[test]
fn test() {
    let aswer = compute_answer("input.string");
    assert_eq!(
        answer.foo,
        vec!["input".to_string(), "string".to_string()],
    );
}
```

but it’s not as elegant as having a single matching expression for the whole type that I’m testing. But what if there was a macro for matching only what I care about? Let’s try the assert_matches! macro from the matches crate, a macro that makes it possible to assert that something matches a pattern similar to a match expression:

```rust
#[test]
fn test() {
    assert_matches!(
        compute_answer("input.string"),
        Answer {
            foo: ["input", "string"], // Does not compile
            bar: _, // Actually works!
        }
    );
}
```

AFAIK it’s not possible to match a struct deeply like this, because directly matching a Vec to a slice pattern does not work. Only things that are already slices may be pattern matched, and the members of Answer are not slices but Vecs, and not coerced.

My point with looking at assertion macros? Even if I might not be fully able to isolate everything my module-under-test does, I hoped that it would be straight-forward and elegant to at least only match the part of the output that I care about. It is a pattern I often use in jest: `expect(actual).toMatchObject({…})`.

## Conclusion

I actually haven’t had a hard time at all “selling in” Rust for projects that would normally have been written in JVM or node.js. We’ve all been there complaining about the unstoppable memory-hunger and crazy slow Boot of Spring Boot. We are making web applications, and the problem space is usually in the high-level domain. Developers working with domains of a similar level are spoiled with good testing tools because their traditional languages of choice are also high level. Rust finally gives us a viable way to solve high level problems using a low level language—runtime-bloat-be-gone. But Rusts current testing story will be harder to sell to the TDD purists.

For 2020, I think the Rust community should make awesome, intuitive, complete, and well documented testing a priority. I don’t know how. But Rust has already managed to overcome so many unbelievable obstacles—just the feeling of using a high level language when you’re not really — this blows my mind. Now let’s make testing a killer product too.
