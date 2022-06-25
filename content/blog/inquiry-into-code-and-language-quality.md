+++
title = "An inquiry into code and language quality"
date = 2022-06-25
[taxonomies]
categories = ["Programming"]
tags = ["quality"]
+++

I recently had a very small dispute with someone about _code quality_.
Afterwards, I wasn't sure if I fully understood the concept, so I wanted to write something down to organize my thoughts around this topic.

In my experience, people, including myself, will name this code or that code as _bad_ or _good_ quality, often without much thought behind it.
This could be because _quality_ could be a measure of whether we subjectively like what we see, or not.
But I suspect that it may be possible to speak about _objective quality_ in the context of computer code.

I also find it amusing to argue with people on what are good and bad programming languages. More on that towards the end.

## Objective quality
I'll try to define some terminology.
A _computer program_ is made from a _code base_.
A code base produces different computer programs at different points in time, because the code base is being changed by humans over time.
A code base at a point in time is usually referred to as just *(the) code*.

Code and code bases is by and for humans, programs are for computers.
Code bases may be very large or very small.
Compilers and interpreters are programs that implement a programming language, whose job is to turn code into computer programs.

Based on this, we can suggest the most important _code quality_:

#### 1. The code can be turned into a valid computer program.
This can be done using a compiler or an interpreter. Failing to execute a single program
instruction from the resulting program will mean that this quality is not present.

Code having this quality is simply _better_ than code lacking this quality.

Moving on..

#### 2. Executing the program produced from the code does what the programmer intended.
This quality is related to the absence of bugs in the program.
If the code looks like it might do one thing, but then the program does a different thing, it will lead to a decrease in general quality, because of a decrease in this _specifc_ quality.

#### 3. It is possible to find the source of a bug by just reading the code.
This quality assumes some human skill, but that's necessary for writing code that verify as valid computer programs in the first place.

If this quality is present in code, it is possible to read through the code lines and find problems without using other tools like debuggers.
It is related to _code readablility_, but that's more subjective and not the same thing.

This quality is about completeness, and code being more excplicit than implicit.
The code should do what it looks like it does, not more, and not less.
It shouldn't try to hide any details.
It seems somewhat related to the previous quality:
Code that has a fewer number of bugs, tends to be more explicit than code that contains more bugs.
There could be several reasons for that, but I can think of two, both of which relate to the previously mentioned qualities:

* The code base has accumulated bugfixes over time, because bugs have been found and the code base has since been corrected over and over again.
  Code is usually less explicit before a bug fix than after: Most bugfixes involve _adding some more code_ (though this is not always true).
* The code needs to be valid computer program in order to be useful at all, and the compiler/interpreter _required_ the programmer to be explicit.

_Explicit_ programs that are valid are better then _implicit_ programs that are valid.

But there is nothing about _code quality_ in itself that make any of these mentioned qualities related to each other.

## Programming language quality
A programming language is required for turning code into a computer program. The programming language either accepts or rejects code validity.

Are we allowed to say that some programming languages have better quality than others?

Let's try.

#### 1. A programming language that rejects more programs has better quality than one that rejects less

This is trivially false, because a language that accepts no programs is not a good quality programming language.

#### 2. A programming language has good quality if it objectively rejects low quality code

I think we can say that a programming language that rejects more buggy programs than another one is a better programming language (if
all other things are equal).

If this is true, using a higher quality programming language will lead to higher quality code.

To illustrate: I believe that many developers like to use linters that are able detect common mistakes.
So let's compare a permissive language with and without a linter.
With the linter turned on, the developer would likely write fewer bugs to begin with, i.e. higher quality code.
Therefore, if we pretend that the language + linter were combined together to form a new language, that language would have higher quality than the original permissive language isolated.

## Conclusion for now
Yes, I do think we can speak of both objective quality in code, and objective quality of languages.
Of course, I didn't cover every possible aspect of code and language here, I was just trying to find some examples from which
we can state something objective.
