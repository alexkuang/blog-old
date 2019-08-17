---
layout: post
title: "Fibonacci Stream"
date: 2013-10-17 07:15
comments: true
categories:
- programming
- scala
---

I was going to test out some of the codeblock functionality in octopress, but as it turns out the fancy stuff I wanted
to test is [not available in master](https://groups.google.com/forum/#!topic/octopress/y1IlHmFYydQ) and I'm far too lazy
to sync to 2.1 at the moment.  As such, everyone will have to settle for this plain jane snippet:

```scala
// e.g., fibStream.take(5)
def fibStream: Stream[BigInt] = {
  def loop(prev1: BigInt, prev2: BigInt): Stream[BigInt] = prev1 #:: loop(prev2, prev1 + prev2)
  loop(0,1)
}
```

Nothing fancy, just one of many possible variations of the fibonacci sequence.  This one is done via a `Stream`,
which is Scala's memoized lazily evaluated list.  ([More here](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream)).

Will probably write more about the why/how of the nested loop `def` (mostly for my own reference), but for now I think
the codeblock looks pretty neat.  It's still a shame I didn't get to play with funny line numbers/highlighting stuff
though.
