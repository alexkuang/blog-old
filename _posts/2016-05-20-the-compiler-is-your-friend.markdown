---
layout: post
title: "The Compiler is Your Friend"
date: 2016-05-20 07:06:20 -0400
comments: true
categories:
- scala
---

One sentiment I see from people coming from dynamically typed languages (python, ruby) or even Java and C is a general
dismissiveness about static typing and compilers.  At best, the sentiment is "Oh, well it makes sure my input is an Int,
or a Float, or a Bool...  That's cool I guess, but I can do that with TDD".  At worst, static typing is seen as
something to fight against -- shackles that limit our creativity and bar us from what we *really* want to do and the
beautiful programs that we could write if only the type-checker got out of our way.

Personally, I think of the type-checker (and by extension the compiler, really) not only as a free suite of rigorous
tests built from first principles, but as a friend that is nice enough to correct me, the silly human, when I think I'm
making sense but I'm really not.  Sure, sometimes that friend is a bit dense (*ahem* Java, *ahem*) and can't quite
understand what I'm trying to say, but in the case of a language like Scala I find that the compiler is right more often
than not.  In fact, I had a whole giant wall of text geared up to talk about the value in using `Option` over `null`, in
using `Either` instead of exceptions, in capturing values without using Stringly-Typed data...  But then Li Haoyi beat
me to it with another addition to his wonderful Strategic Scala Style series: [Practical Type Safety](http://www.lihaoyi.com/post/StrategicScalaStylePracticalTypeSafety.html).
I highly recommend reading it (and the rest of the Strategic Scala Style series) before coming back.

Still here?  That post covers a lot of what I wanted to say, but I wanted to put some extra emphasis on one particular
topic...

<!-- more -->

### ADTs (!!!)

ADTs (Algebraic Data Types) are so, so good.  You can take a lot of features away in Scala and I could probably get by,
but ADTs -- Or at least, the closest Scala approximation -- are on the short list that you'd have to pry out of my cold
dead hands... Along with higher order functions and pattern matching, probably.

So, a quick review...  What are Algebraic Data Types?  ADTs are so named because their structure can be described in two
operations: Product, and Sum.

#### Product Types

Product types are present in one form or another in the vast majority of mainstream programming languages.  People may
know it as a struct in C, or a record, or a tuple, or a `case class` in Scala.  It's essentially a way of mashing
multiple types together into one type.  The reason it is called a product type is because the cardinality of the type
(i.e., the set of all possible values for it), is the product of the cardinality of the type's constituents.  Some quick
examples:

```scala
// Has 2 possible values: A(true), A(false)
case class A(b: Boolean)

// has (2 * 2 = 4) possible values: AA(true, true), AA(true, false), AA(false, true), AA(false, false)
case class AA(b1: Boolean, b2: Boolean)
```

#### Sum Types

Where product types express "this *and* that *and* the other thing", sum types (also commonly known as union types)
express "this *or* that *or* the other thing".  Sum types are so named because the cardinality of a sum type is the
sum of the cardinality of the type's consituents.  Unfortunately in Scala 2.x, there isn't direct support for sum types,
but they can be roughly approximated with `sealed trait`.  Some examples:

```scala
// Probably not how any of this is implemented in the actual standard lib but you get the idea  :)
sealed trait Boolean
final case object True extends Boolean
final case object False extends Boolean

sealed trait Option[T]
final case class Some(x: T) extends Option[T]
final case class None extends Option[_]

sealed trait Either[A, B]
final case class Left[A, B](a: A) extends Either[A, B]
final case class Right[A, B](b: B) extends Either[A, B]
```

One cheerful note is that proper union type support was announced for Dotty at ScalaDays 2016, so hurrah!  \*confetti\*

### So What?

At first glance, ADTs might seem simplistic -- kinda like fancied up named tuples.  But they are deceptively powerful,
because combining them effectively allows you to encode invariants about your program's logic and state in the
type system, and therefore leverage the compiler to keep you from violating those invariants and writing buggy software.

Which sounds like a bunch of abstract nonsense, so it's time for a concrete example!

### Modeling Real Life: Wrangling User Lists

Working in the marketing space, it is very common to deal with lists of users.  It can be a list of users from a mailing list, a list
of website visitors, or a list of people who have downloaded your white paper.  In any case, it is a list of users that you
have in your possession, and you want to follow them around the internet and serve them ads.  Such a list of users might
have attributes like an id and a human-readable name.  Since most folks aren't in the business of building exchanges,
it might also have a downstream platform to target (e.g., AdWords, or Facebook).  It might also have some different
states that keep track of whether the list's meta-data has been sent to the downstream platform, or whether it's been
archived or soft-deleted and when that state transition happened.

So one reasonable implementation at a data structure for such a user list might look something like:

```scala
sealed trait ListStatus
final case object Pending extends ListStatus
final case object Active extends ListStatus
final case object Archived extends ListStatus

sealed trait Platform
final case object Facebook extends Platform // And so on and so forth

case class UserList(
  id: Long,
  name: String,
  platform: Platform,
  status: ListStatus,
  createdTime: Long,
  archivedTime: Option[Long],
  downstreamId: Option[String] // i.e., a "foreign key" to the list on the downstream platform
)
```

At first glance, this seems like pretty reasonable code.  The various states and platforms are enumerated, and it seems
to carry all the information we want with decent naming and so on.  At the very least it's better than carting around a
tab-separated string, or something like `val list: (Long, String, ListStatus, Long, Option[Long], Option[String])`.

But there's still something a bit off here.  In particular, look around the optional fields.  With this current model,
it is possible to construct an internally inconsistent list:

```scala
val list = UserList(
  id = 1,
  name = "website visitors",
  platform = Facebook,
  status = Active,
  createdTime = System.currentTimeMillis,
  archivedTime = None,
  downstreamId = None // Wait, what?
)
```

The example above is internally inconsistent because the list's active status indicates that it should be onboarded to
the downstream platform, but for some reason we do not have a downstream id.  There are other "weird" cases like this
that are possible but do not make sense semantically -- for example, `archivedTime = Some(1)` with `status = Pending`.

This data structure is almost trivially simple, but there area already a good number of things that can go wrong,
especially once we take outside input and possibly complex business logic into account.  And here there's nothing
stopping us from constructing these degenerate cases other than our ability to keep everything in our heads (spotty even
at the best of times) and read the code *really really* carefully every time we work with this particular data structure
(good luck).

Another point to consider is what this does to the code that works with the data:

```scala
// Send users from our list to downstream platform so we can serve them ads
def onboardUsersToList(list: UserList, users: Seq[User]) {
  list.status match {
    case Active => {
      val platformId = list.downstreamId.getOrElse {
        // this should never happen
        throw new IllegalStateException("Downstream id not found for active list")
      }

      platformClient(list.platform).sendUsers(platformId, users)
    }
    case _ => throw new IllegalArgumentException("Cannot onboard users for non-active list")
  }
}
```

The above code is a conceptually simple method that takes users from our list and sends them to a downstream platform,
e.g.  Facebook.  It technically does what we want, but there is a lot of clutter introduced by the management of the
status.  The block under `list.downstreamId.getOrElse` is particularly bad, since it basically amounts to saying "well
we don't think we're wrong, but we technically could be wrong, so we have to sprinkle this boilerplate-ish error
handling into our business logic".  Apparently, this sort of thing is
[not](https://github.com/anthavio/anthavio-commons/blob/e1982362cb3671c85b79923b1b3199f6eee8d60f/src/main/java/net/anthavio/NeverHappenException.java)
[that](https://github.com/adi-bolb/pair-programming-match-making/blob/51496f4b854c56974324d3f97ac19b7e76ac9a54/src/main/java/org/findapair/ThisShouldNeverHappenException.java)
[uncommon](https://github.com/akestner/DockerPlugin/blob/42abd2aaf6c649827d4101250bee761d0f65d160/src/com/akestner/plugins/docker/exception/ShouldNotHappenException.java).

One possible remedy is to refactor `UserList` to encode some of the constraints into the type:

```scala
final case class UserListMetadata(id: Long, name: String, platform: Platform, createdTime: Long)

sealed trait UserList { metadata: UserListMetadata }

final case class PendingList(metadata: UserListMetadata) extends UserList
final case class ActiveList(metadata: UserListMetadata, downstreamId: String) extends UserList
final case class ArchivedList(metadata: UserListMetadata, archivedTime: Long) extends UserList
```

This lets us rewrite the above method as:

```scala
def onboardUsersToList(list: ActiveList, users: Seq[User]) {
  platformClient(list.metadata.platform).sendUsers(list.downstreamId, users)
}
```

At this point, one might protest, "That just makes you push the status validation somewhere else!  The first
implementation was just bad practice, you could easily have written this simple version with the old type by refactoring
to `def onboardUsersToList(downstreamId: String, users: Seq[User])`."  Yes, this is perfectly true.  The validation
still has to take place *somewhere* since the code will presumably be interacting with the outside world.  The
difference here is that the former implementation can only enforce cleanliness and correctness with malleable things
like documentation and best practice guidelines, whereas the latter implementation enforces it by simply *refusing to
compile until you fix it*.  The latter implementation also reduces the amount of "unforced" errors since it is
impossible to construct an `ActiveList` object without a `downstreamId`, whereas it was possible to accidentally create
a `UserList(status = Active, downstreamId = None)` previously.

### Stepping back

Hopefully the above example demonstrated a reasonable real-world case where ADTs can be really useful.  ADTs seem really
basic -- and in the vast world of type hackery they are only a starting point -- but you can do some surprisingly
powerful things with these fundamental building blocks.

There is one last point I want to make.  As a professional programmer (i.e., employed by a business to write code that
ostensibly generates some amount of profit), the goal of all this is not to encode the world at the type level.  The
goal is to reduce complexity and mental overhead.  There are some insane things possible once you go down the rabbit
hole of type hackery -- for example, [type-level quicksort](http://jto.github.io/articles/typelevel_quicksort/).  This
is all great fun and by no means a bad thing, but at some point it's important to step back and think, "Right, somebody
else actually has to read, maintain, and modify this code at some point.  *Maybe* this should not go into production."

This is especially important to keep in mind in a language like Scala, which provides us with plentiful amounts of
power to complicate things and shoot ourselves in the foot if we're not careful.
