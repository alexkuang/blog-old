---
layout: post
title: "A Neophyte's Take on Scala"
date: 2013-12-09 18:12
comments: true
categories:
- programming
- scala
---
A [blog post](http://overwatering.org/blog/2013/12/scala-1-star-would-not-program-again/) has recently been making the
rounds in all the usual programming fora--/r/programming, hackernews, and so on--even eliciting a
[response](https://groups.google.com/forum/#!topic/scala-debate/153H3Ya4Nxk) from Scala's creator himself.  There's a
lot of discussion both productive and inflammatory out there already, but I thought I'd jot down my perspective on the
common points as someone who: 1. picked scala up relatively recently (2.9.2 and 2.10), 2. has worked with it for a few
months on the side, and 3. comes from a background of dynamic languages.

#### Long Compile Times
I can definitely see where this is coming from.  Scala's compile times can be pretty long, though I had pegged that as
the cost of using a compiled language anyway.  I, at least, felt a marked increase in times upgrading from `2.9.2` to
`2.10`, presumably from the slew of new features like macros.  However, it seems that `2.11` will thankfully be
[focused](http://java.dzone.com/articles/state-scala-2013) on compiler optimizations.  In the meantime, I've found that
judicious use of some of Scala's heavier features (such as implicits) keeps the compile times manageable.
[JRebel](http://zeroturnaround.com/software/jrebel/) has been a godsend as well for hot-reloading in incremental
compilation, and they do offer a personal Scala license for free.  Without JRebel my life as a scala user would have
been a lot harder.

#### Opaque Syntax
I agree completely here.  Scala's flexibility in parsing + its allowance of symbolic method names obviously allows for a
lot of flexibility.  This allows the construction of all sorts of DSLs; whether these DSLs end up being expressive or
just plain confusing, is another thing altogether.  In this case, I'd have to say that just because something is
possible does not mean it should be done all the time.

<!-- more -->

The original post mentions that their build script's syntax seemed to be all over the place; I'm assuming they were
referring to the de facto scala build tool, [sbt](http://www.scala-sbt.org/).  While it's great in a lot of ways, I
personally am not a big fan of their heavy use of symbolic names either.  Just as an example, a relatively simple build
file for one of my personal projects already has all of the following:

```scala
libraryDependencies ++= Seq( ... )

testOptions in Test += (...)

resourceGenerators in Compile <+= (...)
```

All of them are obviously for adding to some sort of sequence, but still different enough to trip you up.  And this is
still relatively shallow territory--I haven't even mentioned the insanity that comes with more advanced libraries, like
the [fish operator](http://stackoverflow.com/questions/14472310/pronounceable-names-for-scalaz-operators).

In Odersky's google groups thread, he mentions a proposal for requiring an equivalent alphabetical alias for all
symbolic methods.  I hope that goes through.  If nothing else, it'll make the methods easier to google.

#### Documentation
As with any budding technology, documentation tends to lag behind development.  However, language and core library
documentation has improved immensely even in the time since I picked up Scala, so many kudos to the typesafe team for
that.  However, the third-party libraries still have a way to go; as helpful and educational as it may be, source code
should _not_ be the go-to method for research.

#### Types and Case Classes
Here's where I have to respectfully disagree completely with the original post.  I _love_ types and case classes.  Being
able to encode logic in your types that would otherwise have ended up as boilerplate-y edge case checking, _and_ having
the compiler do it for you instead of maintaining the extra tests, is a win in my book.  Coming from a dynamic web
background, having a compiler do some of these checks for me was a breath of fresh air, especially since it doesn't come
at the cost of extreme clunkiness a la Java.

And case classes / [algebraic data types](http://learnyouahaskell.com/making-our-own-types-and-typeclasses) are the
bee's knees.  Between the `copy` method and the free pattern matching, I don't see how case classes do anything but get
rid of repetition.  Forgive the simplistic example:

```scala
case class Calculator(brand: String, model: String, version: Int)

// Extraction, conditionals, and assignment, all done succinctly
def calcType(calc: Calculator) = calc match {
  case Calculator("hp", "20B", 1) => "financial"
  // ...
  case Calculator(brand, model, version) => "Calculator: %s %s %s is of unknown type".format(brand, model, version)
}

// No custom methods/etc needed
def upgrade(calc: Calculator) = calc.copy(version = calc.version + 1)
```

#### Overall impressions
Despite its flaws, I've really enjoyed working with Scala and it will probably remain a part of my toolbox.  Its
flexibility and power makes it a great gen-purpose language, especially in its type system and ability to combine
functional programming with OO.  It sits in the JVM, which means interoperability with the existing mammoth Java
ecosystem comes with just a little bit of glue code.  And more importantly, it means that if I should pick up Clojure in
the future, interoperability with THAT will not be too far off either, knock on wood.

That said, it's not by any means a perfect language.  The same flexibility that makes it great to use can often result
in convoluted APIs, and it's very easy to shoot yourself in the foot.  Living in the JVM also means the language has to
deal with things like type erasure; that results in some annoying limitations, especially if you're looking to use scala
in a more advanced functional capacity (see [scalaz](https://github.com/scalaz/scalaz)).  And lastly...  Scala, at least
to me, is one of those tools that feels natural once you're used to it, but requires a significant mental shift to fully
grok--I wouldn't want to be plopped into scala on a fresh project with a one month deadline.  Ideally, there should be
some better documentation to help bridge that gap.

Luckily, if that mailing list thread is any indication, the community is aware of these shortcomings.  And as the
language and ecosystem mature, they will hopefully become less and less of a problem.
