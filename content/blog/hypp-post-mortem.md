+++
title = "Hypp post mortem"
date = 2022-04-27
[taxonomies]
categories = ["Programming"]
tags = ["rust", "hypp"]
+++

Last year I spent a lot of time trying to make a proof-of-concept Rust GUI/web framework
called [Hypp](https://github.com/audunhalland/hypp). It takes inspiration from
Svelte, and uses a complex procedural macro to define components.

I wanted to design a syntax that is a hybrid of HTML and Rust code,
and I think it succeds and some things and fails at others.

A code example taken from its `tests` directory:

```rust
component! {
    Toggle(prop1: bool, prop2: &str) {
        toggled: bool,
    }

    fn handle_click(&mut self) {
        *self.toggled = !*self.toggled;
    }

    <div>
        if prop1 {
            <p>"yep"</p>
        }
    </div>
    <div>
        <button onClick={Self::handle_click}>
            if toggled {
                "Toggled"
            } else {
                "Not toggled"
            }
        </button>
    </div>
}
```

Everything related to one reusable component is written within the proc macro `component`,
which enables the macro to make some wild optimizations.

First, an analysis of what this code means. At first we see a construct like
`Name(arguments...) { state_variables... }`. That's the name of the component,
its parameters, and its internal state. Then there's an `fn` definition which
is a callback function that can be used within the _body_, which is
the last main syntax element. The body is a markup template language, and
will look familiar to many.

### The nice features in Hypp that I actually got working
#### Nice JSX-like syntax featuring Rust keywords in markup templates
The markup syntax supports `if`, `for` and `match`! It only needs to look at a
few things to know what kind of "element" something is. A `<`
starts some markup element expression. A known Rust keyword
starts some "evaluation" (conditional or repetition).
A `{` will start a string-evaluation expression. This means
that there is no awkward syntax to "escape out" to reach
evaluation mode, but it also requires _literal text_ to be
`"double-quoted"`. In my opinion that syntax works out very nicely,
the code appears clean and readable.

#### Update optimizations and data-flow analysis
When you create a new reactive GUI framework, you absolutely need to have all
the cool optimizations as selling points! Most optimizations in web frameworks
are of the form _don't update DOM if nothing changed_. I went for an architecture
where I wanted to avoid comparing new/old model values as much as possible.
Along with each passed parameter to a _redraw_-like instruction, Hypp sends
along whether some parameter has changed from the parent component to the child
component. The child component then tracks data flow from its input definitions
out to its leaf nodes where it passes those parameters on.

So a Hypp component makes a quick check at the start of its _update_ procedure
to see which parts of its markup tree needs to be updated. Some parts of the
tree will be _constant_, i.e. they don't depend on any variables. These parts
of the tree will only be created once, then never touched again. _Variable_
parts will get patched in and out between these constant parts.

#### Server side rendering
From the start I designed Hypp with multiple backends in mind. I think that
turned out well.

#### Use of Generic Associated Types
Hypp is GAT-heavy, and I utilized it to such a degree that I stumbled upon
several bugs in the Rust compiler, I even [fixed one](https://github.com/rust-lang/rust/pull/89341)!

In the end though, I think I refactored the code so the fix wasn't necessary for Hypp after all.
But I can definitely say that Hypp directly lead to actual improvements in the Rust language!

### What didn't work so well
#### To much macroifization
I don't like that everything has to be inside one big macro. But I think it needs to,
to be able to see everything that's part of the component at the same time. There
is a lot of analysis going on. All the conditionals and loops need to be translated
into a state machine, the component instances need to store all of this, along with
its input parameters. It needs to store its last input parameters in case it gets
a direct signal that it needs to update (e.g. an event happened like a button click).
In that case it may have to send those parameters to _child components_ that were
potentially invisible (not instantiated) before that event.

The macro generates a _lot_ of Rust code, more than I would have thought at first.
I put in significant effort to reduce every possible code duplications. Somehow
I don't like that the component looks small and elegant expressed in surface syntax,
but compiles down to a monstrosity.

#### Event propagation
I never got this part right. The problems with event handling can be found quite
easily in Hypp's Todo example app. It gets into a never-ending update loop.

At this point I lost the motivation. It didn't think it was fun anymore, there
were too much refactoring of everything everywhere just to make tiny adjustments.

#### Loose ends and scope of project
Hypp started out as a syntax experiment, and it escalated. I find meta-programming
too interesting.

A project like this can never really be "done" because its
scope is very hard to define. To take it somewhere remotely close to production quality
is way too big a task for a hobby project developed during the night.

It was (almost too much) fun while it lasted. My next projects will try have a more
focused scope.
