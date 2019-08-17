---
layout: post
title: "Thread Macro"
date: 2015-01-24 11:53:42 -0500
comments: true
categories:
- programming
- clojure
- totw
---

Let’s deviate from Scala for a bit and talk about clojure.  Or lisps in general, I guess.  A lot of the “kinda joking
except not really” quips that commonly float around on the internet are about the parentheses, as in how there are so
many of them.  For example, if you want to take a number `x` and add one, multiply by two, then add 3, the code might
naively look something like this:

```clojure
(+ (* (+ x 1) 2) 3)
```

Or perhaps like this:

```clojure
(+ 3 (* 2 (+ 1 x)))
```

Look at the parens!  Especially the consecutive 3 closing ones in the second variation.  For a sufficiently long chain
of functions, it can get pretty unreadable—especially with multiple arguments and whatnot.

Enter clojure’s thread macro.  The thread macro is a macro in the form of `(-> x & forms)`, and it “threads” `x` through
the `forms` as the first arg*.  Which sounds terribly confusing explained, so an example is probably better here.  Take
this snippet using the thread macro:

```clojure
;; add one, multiply by two, and add three
(-> x
  (+ 1)
  (* 2)
  (+ 3))
```

This desugars into `(+ (* (+ x 1) 2) 3)`, i.e. the first variation of the initial example above.  Personally, I find the
macro version much more readable since each call is on its own line, and it seems more expressive of applying a series
of functions to the initial x.

The thread macro is also useful for chaining together collection methods like `map`.  Since clojure doesn’t have
first-class OO support (instead favoring protocols and such), map exists as a regular function that takes the collection
as an arg, instead of as a method on a collection class.  So chaining together a bunch of ops on a vector might look
something like...

```clojure
;; add one to every number and filter for even numbers
(->> [1 2 3 4 5 6]
  (map #(+ 1 %))
  (filter even?))

;; Without the thread macro, would look like:
(filter even? (map #(+ 1 %) [1 2 3 4 5 6]))
```

The only difference is that in this case, ->> was used.  ->> is “thread last”, which is like -> (“thread first”), except
it inserts the expression at the end of the form.

This pattern also exists in other languages (especially those that don’t offer first-class OO, which allows fancy
`return self` type stuff), like Elixir’s pipe `|>` (in the spirit of the unix pipe) which is what prompted me to spread
the word about this:

```elixir
# double and add one to each element
[1, 2, 3]
|> Enum.map(fn x -> x * 2)
|> Enum.map(fn x -> x + 1)
```

The thread macro pattern doesn’t have as much of a place in Scala, since Scala has mechanisms like the collection
library and implicit conversions to help express similar logic in elegant ways.  But when I first read up on macros in
lisp, I spent some time scratching my head at the day-to-day practical uses until I found this and had my first
“ohhhhhhhhhhhh” moment.  In any case, hope this was mildly interesting!

\* - Well technically, as the second item in the form, which is effectively the first arg for functions... But that might
be a bit too lispy.

### Bonus

When I published this email to the internal list, it generated some discussion wherein I learned that there are other
neat features of the same sort like [doto](https://clojuredocs.org/clojure.core/doto), and that they're all just
various derivations of the K combinator.  Of course, googling k-combinators led to a pretty
[heavy looking wiki page](http://en.wikipedia.org/wiki/SKI_combinator_calculus), so I was referred to
http://combinators.info/ , which I have been trying to get through since.
