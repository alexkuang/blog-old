---
layout: post
title: "Clojure First Impressions"
date: 2013-12-20 14:05
comments: true
categories:
- programming
- clojure
---

After achieving some measure of familiarity with Scala, and with newfound copious amounts of free time, I decided I
wanted to see more of what the functional world had to offer.  The obvious choices were Haskell and Clojure; but while
Haskell has the upper hand in functional purity and a crazy advanced type system, I like to think I'm a pragmatic guy at
heart and Clojure seemed more practical.  I haven't worked with it too extensively, but my experience so far can be
summarized by two words: Simple and composable.

#### The language

Clojure is a refreshingly simple language.  Despite my last foray into a Lisp being about half a decade ago, the
learning curve was much gentler than I'd expected.  Maybe it's because I was already in a functional programming
mindset, but the straightforward syntax and [abundance](http://clojure-doc.org/)
[of](http://clojure.org/getting_started) [documentation](http://clojure.org/cheatsheet) probably helped.  And on a
completely subjective level: `iDislikeCamelCase`, and `clojure-case-is-pretty-neat`.

#### The ecosystem

Of course, the overall enjoyability of using a language doesn't depend solely on the core language, but also the
libraries and toolchain available.  Most of the libraries I've seen keep in line with the design of the language: super
lightweight, super simple, super composable, and as a result super easy to ramp up on and use.  Theoretically that
should just describe all good library design in general, but I feel like the clojure community takes it especially to
heart.

Compojure, for example, chose to implement its url
[destructuring](https://github.com/weavejester/compojure/wiki/Destructuring-Syntax) to closely follow the destructuring
available in stock Clojure `let`s expressions.  I can't help but draw the comparison to Scala, where I'd be more likely
to find that url decomposition exists only in the form of an exotic DSL.  Another huge example for me is the difference
between the simplicity of the Clojure build tool Leiningen and the craziness of Scala's SBT.  Sorry SBT--You work very
well, but I'd rather not have to google what the `<++=` operator does every time I touch the build.

#### With vim

One of my original reasons for leaning clojure was its close integration with
[LightTable](http://www.chris-granger.com/lighttable/).  As it turns out, the functionality I liked could be
replicated in vim with [fireplace.vim](https://github.com/tpope/vim-fireplace)'s quasi-insta-repl and insta-doc, due in
no small part to leiningen and nrepl's awesomeness.
[Rainbow parentheses](https://github.com/kien/rainbow_parentheses.vim) is also pretty cool, and has been useful enough
that I will probably keep it on even when I don't have to deal with the hardcore levels of parens in Lisps:

![rainbow-parens](/images/rainbow-parens.png)

#### Overall

If programming languages could be graded on usability, Clojure would get full marks.  It has been a breath of fresh air
after dealing with the crazy complexity in Scala.  Undoubtedly working through the latter had a part in making the
former much easier, and Scala will always have a place with me, but for now I find myself slowly joining the rest of the
Clojure bandwagon.
