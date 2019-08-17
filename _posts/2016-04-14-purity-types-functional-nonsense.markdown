---
layout: post
title: "Purity, Types, and Other Functional Nonsense"
date: 2016-04-14 07:12:23 -0400
comments: true
categories:
- totw
---

A while ago the topic of [lenses](http://fluffynukeit.com/how-functional-programming-lenses-work/) came up in one of the
chatrooms at work. This led to a discussion on the (ahem) suboptimal readability of advanced functional programming
code examples, and how a lot of the fancier constructs are by-products of being super hardcore about purity.  This led
to me writing a great big email to the team that generated a whole bunch of thoughtful discussion on FP, complexity, and
straw men with free lunches. I was also told I should transcribe to blog form, so here it is reproduced (mostly)
faithfully:

I’ve been digging into Haskell/etc a bunch lately and have hit some similar pain points. Typeclasses and
Haskell-specific syntax aside, I think one of the biggest mental shifts I have experienced is learning to reason based
on types alone. Full disclaimer: I am by no means an expert on any of this. But since I am (at the original time of
writing) sitting in an airport with a mildly disgruntled cat, I figure I may as well write up a blurb about my
experiences on purity, types, and what can follow from it.

<!-- more -->

### A Brave New World

So for the rest of this email, let’s suspend our disbelief and imagine that we have an “ideal functional” Scala that is
pure and strongly typed. This means no:

* `null`
* exceptions
* type-casting (e.g. `asInstanceOf`, `isInstanceOf`)
* side-effects (e.g. `println`, `cw.track(metric)`)
* Object (e.g. `anything.toString`)

### A simple example

Given the above, consider a function with this signature:

```scala
def nonsense[A](i: A): A
```

`nonsense` is parameterized on the type `A`, which could be any type. This means that the implementation of `nonsense` knows
literally nothing about its input—It could be a number, a list, an Option, a String, or anything else we can think of
passing in. Because we’re living in this imaginary world where null, etc do not exist, `nonsense` literally cannot do
anything with its argument except return it.

Following this logic, we can deduce that `nonsense` is just the identity function based on its type signature alone. It
doesn’t matter if it is called `nonsense` or `foo` or `id`, or if its argument is named `i`, `j`, `k`. We don’t even have to look at
the implementation. Given the type `A => A`, this function literally cannot do anything other than return its argument, or
else it would not have compiled. (Again, this is assuming no `null`, `println`, etc).

Extending this slightly, this function:

```scala
def c[A](i: A, j: A): A
```

can only have two possible implementations based on its type: `def c[A](i: A, j: A) = i`, or `def c[A](i: A, j: A) = j`.
Likewise, `def c2[A, B](i: A, j: B): B` can only `=> j`, and so on.

### A structured example

Now consider the function:

```scala
def nonsense2[A](i: List[A]): List[A]
```

Without looking at the function name, the implementation, and so on, what can we deduce about `nonsense2` in our imaginary
magical world? *The output never contains any element that was not in the input*. That is, the output could be the input
itself, it could be input reversed, it could be the input sliced up—but you will never have a situation where e.g.
`nonsense2(List(1)) == List(2)` or `nonsense2(List(1)) == List(List("XYZ"))`. To put it another way, just based on the type
signature of `List[A] => List[A]`, we know that the function may do something to the `List` structure but nothing will
happen to the individual `A`s--or else the code would not have compiled.

This type of reasoning can be applied to basically any function to derive information about it (guess what this function
is: `def fff[A, B](f: A => List[B], i: List[A]): List[B]`). By maintaining strict typing and purity, we give up the
convenience of things like side-effects and run-time type-matching. But in exchange, we get increased ability to reason
about (and have a sufficiently smart compiler prove) certain properties of our code, regardless of potentially
misleading/outdated evidence such as function names and comments. The big idea here is to 1) shift as much of the
“thinking” to the compiler as possible to reduce the potential for human error, and 2) lock as much down during
compile-time as possible to reduce the potential surface area of run-time errors.

### Real life

Sadly, real life is messy and not everything can be proved at compile time. Even in `fff` above, the implementation could
be e.g. `flatMap` or `flatMap andThen reverse` or just `Nil`.

As another example, consider:

```scala
def intNonsense(i: Int): Int
```

Based on the types, we can say that the potential output of `intNonsense` is constrained to the size of the set of
possible Ints; i.e., there are 2^32 possible outputs. Other than that though, we can’t say much about the function’s
implementation(*). It could double its input, add 1 to its input, negate its input, negate its input but only if it’s
even, and so on.

So what’s a programmer to do?

(*) - Without some fancy pants type-level probably-church-encoded something-something-something sort of programming, but
that’s a whole ‘nother can of worms that I haven’t gotten remotely close to opening...

### Tests!

One natural reaction would be to start writing some unit tests to make sure this function does what we think it does. So
we start writing some tests:

```scala
assert(intNonsense(1) == 1)
assert(intNonsense(2) == 2)
assert(intNonsense(0) == 0)
assert(intNonsense(-1) == -1)
```

And from that we may think that `intNonsense == identity`. Except that with the above suite, the following could still
happen, and we would be wrong:

```scala
def intNonsense(i: Int) = if (i > 20000) -1 else i
```

We could also test the totality of the mapping between input and output to intNonsense. But even for a simple `Int =>
Int` function, that would be a total of 2^32 possible inputs, and I feel like even the most determined programmer would
give up after a couple hundred. Or maybe not, and more power to you. :)

One way to help with this is properties-based testing, which I think has been discussed before. Props-based testing
frameworks come in various levels of sophistication, but the basic idea is that instead of writing tests for a single
case, you write a general “property” for your function and the framework will generate a whole bunch of kinda-random
input to throw at it. As an example in [ScalaCheck](https://scalacheck.org/), a property for string concatenation might
look something like:

```scala
property("startsWith") = forAll {
  (a: String, b: String) => (a+b).startsWith(a)
}
```

This is not perfect, but it still lends itself to far more robustness-per-effort when compared to individual unit tests.
In some cases, it is even more readable as far as communicating what you want to be true about your code.

Of course, props-based testing can be used with the most imperative mutable-state-having code, but when combined with
pure compiler-friendly code it can end up really reducing the potential for unexpected errors.

### Disclaimers

Wrapping up, a couple of disclaimers.

1) If we take the idea of using types/etc in this way another step further, we might draw the conclusion that function
names and documentation are relatively meaningless next to what you can prove with the compiler. This is one argument
I’ve heard re: scalaz’s lack of concern about approachability in its API naming + docs. “We don’t need a docstring, from
the types there is only one possible implementation of this function, so it’s self-explanatory”. If we take this idea
all the way to the extreme, we might even argue that concrete names are harmful since they propagate impressions about
the code that can potentially be outdated or false. This should sound familiar to anyone who has read, say, Tony Morris.

FWIW, I tend to come down on the side of “why not both”? I love the idea of catching more mistakes at build-time by
fully leveraging a smart compiler and better tests, and I do agree (at least in principle) that maintaining purity also
makes composability easier. But at the same time, I don’t really see the harm in calling an argument `zero` to declare its
intentions, regardless of whether we can fully prove it in the code. To play devil’s advocate again though, I do agree
that sometimes names are vestigial and you have to depend on the types. For example, I’ve tried and failed to think of a
better name than `f` for the argument `A => List[B]` in `flatMap`.

2) Approaching this from another angle, one can also write code using nothing but the types, or what I like to call
“playing type jigsaw.” IMO it’s pretty fun to just write out a bunch of type signatures and almost-kinda-auto-pilot your
way through until it compiles—but it can get really meta, and I have only had limited experience in playing around with
this style of programming. The furthest I have taken it is trying to reconstruct twitter’s
[Stitch](https://www.youtube.com/watch?v=VVpmMfT8aYw) as a natural transformation to `Future`, and most of the other
stuff I’ve played with has been small and isolated—think Project Euler style problems. Which is to say that I’ve
explored this stuff on the side, but in practice for a large enough system I have no idea if it’s e.g. just trading one
type of complexity for another. Though intuitively I feel like enterprise code (i.e., large systems, many
inputs/outputs, lots of complex logic) is where leveraging a strongly typed functional style with a smart compiler could
really be useful in the long run.

### Takeaways

Funnily enough, the most widely actionable takeaway from all of this is completely unrelated to types and naming and all
of that. The one thing I feel that everyone can benefit from immediately is to use more quickcheck/properties-based
testing. While it is not perfect, it sure beats enumerating things on a case-by-case basis. Plus--I forget where I read
this--“nothing scares bugs out of your code like having something feed it pathologically bad values”.

Aside from that... I am not immediately hopping onto the HARDCORE PURITY train, but hopefully this at least gives some
food for thought as far as another approach to writing and thinking about code.
