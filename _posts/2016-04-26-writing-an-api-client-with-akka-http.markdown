---
layout: post
title: "Writing an API Client with akka-http"
date: 2016-04-26 08:00:00 -0400
comments: true
categories:
- scala
- stockfighter
---

[Stockfighter](https://www.stockfighter.io) is a CTF (short for Capture the Flag) game that I first heard about at
Microconf 2015, but haven't gotten a chance to play up until very recently.  I plan on posting more about my impressions
of the game later, but very shortly: it is a series of programming challenges based on the concept of stock exchanges
and ways to manipulate stock exchanges.  Along with the web UI, a public json [API](https://starfighter.readme.io/) is
exposed as a mechanism for interacting with the game.  There did not seem to be any Scala clients floating around, so
I took this as a chance to play around with [akka-http](http://doc.akka.io/docs/akka/2.4.4/scala/http/index.html).

<!-- more -->

### Scaffolding

After some quick googling and reading of tutorials, it looked like the basic structure of an http client would be
something like this:

```scala
package stockfighter.client

import akka.actor.ActorSystem
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer

class TradingApiClient(apiKey: String)(implicit val system: ActorSystem = ActorSystem()) {

  implicit val materializer = ActorMaterializer()

  // TODO
  // def endpoint(): Response = {
  // }
}
```

A couple of things to note here.  First off, `akka-http` is a part of the (increasingly pervasive) akka ecosystem, so
obviously actors have to be involved.  The tutorials either had the `ActorSystem` at the top level or had the
client itself be an `Actor`, but I felt like a regular class would suffice so I compromised by passing the system in as
a parameter with a default.  The `ActorMaterializer` is completely new to me, since I am coming from
[spray](http://spray.io/documentation/1.1.2/) ~1.1 and have missed out on a lot of the latest reactive-buzzwordy
developments.

I'm still not sure I grok it completely, but my understanding is that akka-http is backed completely by reactive
streams, which the client constructs as lazy descriptions of computations.  When the computations are run, the
`ActorMaterializer` spins up the actors to do the actual work.  In any case, I thought about putting the Materializer in
the constructor as well, but the fact that it takes an implicit `ActorSystem` as an argument makes it fairly awkward to
have both `ActorSystem` and `ActorMaterializer` live as constructor params with defaults.  I can think of a few ways to
deal with this, but for a quickie client I decided to just in-line the materializer and move on.

### Making a request

The Stockfighter API ships with a heartbeat/status endpoint, i.e., "is the service up?"  The endpoint lives at
`https://api.stockfighter.io/ob/api/heartbeat` and returns a response in the following format:

```json
{
  "ok": true,
  "error": ""
}
```

This seemed like as good a starting point as any, in that it's a fairly simple endpoint with a simple response type,
but still complex enough to test a full request flow with some common functionality like serialization/deserialization.

As it turns out, it took a decent amount of time and a lot of reading to get to a basic implementation:

```scala
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.model.Uri.Path
import akka.stream.scaladsl.{ Source, Sink }

// constructor boilerplate elided

def apiIsUp: Future[HttpResponse] = {
  // Fancier DSL: `Path.singleSlash / "ob" / "api" / "heartbeat"`
  val source = Source.single(HttpRequest(uri = Uri(path = Path("/ob/api/heartbeat"))))
  val flow = Http().outgoingConnectionHttps("api.stockfighter.io")

  source.via(flow).runWith(Sink.head)
}
```

Executive summary: `akka-http` leverages the concept of reactive streams that seems to be the new hot thing lately.
Streams are essentially fancied up functions, and consist of three parts: the `Source`, the `Flow`, and the `Sink`.  The
`Source` describes the input, which can be single element (`Source.single`), an iterator (`Source.iterator`), a `Future`
(`Source.fromFuture`), etc.  The  `Flow` is the description of a computation to run on the data from the `Source`.  The
`Sink` describes what to do with data after it has been run through the flow: push it into a `queue` (`Sink.queue[T]`),
`fold` over it (`Sink.fold`), and so on.  You can combine things in all sorts of different ways--the above uses `via`
and `runWith`, but there's also `viaMat`, `run`, and any number of other fancy combinators.

What it boils down to here is that the `Source` is an http request, the `Flow` describes how to send that request, and
`runWith(Sink.head)` runs the flow and returns a future of the response.  Phew...

### Serialization/Deserialization

For serialization/deserialization, `akka-http` provides its own `Marshal`/`Unmarshal`.  For json, the default option is
to lean on `akka-http`'s predecessor, `spray`--Or more specifically, `spray-json`:

```scala
// ADT describing the response
case class ApiStatus(ok: Boolean, error: String)


import spray.json._
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport

trait TradingApiSerialization extends SprayJsonSupport {
  // One of the built-in spray-json auto-formatters
  implicit val ApiStatusFormat = jsonFormat2(ApiStatus)
}


// Previous imports elided
import akka.http.scaladsl.unmarshalling.Unmarshal

// N.B. mixing in TradingApiSerialization to get the automatic conversions
class TradingApiClient(apiKey: String)(implicit val system: ActorSystem = ActorSystem()) extends TradingApiSerialization {

  implicit val materializer = ActorMaterializer()

  def apiIsUp: Future[ApiStatus] = {
    val source = Source.single(HttpRequest(uri = Uri(path = Path("/ob/api/heartbeat"))))
    val flow = Http().outgoingConnectionHttps("api.stockfighter.io").mapAsync(1) { r =>
      Unmarshal(r.entity).to[ApiStatus]
    }

    source.via(flow).runWith(Sink.head)
  }
}
```

The Serialization trait is normal procedure for `spray-json`, and the `SprayJsonSupport` provided by `akka-http` just
provides an implicit conversion that links the `Unmarshal(r.entity)` together with the `jsonFormat` for the entity.  The
big wart here is actually the `mapAsync(parallelism = 1)`, which is a result of `Unmarshall(...).to[T]` returning a
`Future[T]`.  I didn't dig too deeply into this, but based on some quick googling the general consensus seems to be that
the use of `Future` here is a way of handling lazy/streaming responses.  Whatever the case, I could not find an
alternative API for this so `mapAsync(1)` seemed to be the least of the evils--another choice would have been something
like `.map { r => Await.result(Unmarshal(r.entity).to[ApiStatus], Duration.Inf) }` but that seems even clunkier.

### Error handling

The above code still has the flaw that if the server responds with e.g. 404, it will throw an exception and the client
will be SOL.  This is not so much an issue for the heartbeat endpoint, but Stockfighter is nice enough to enumerate a
bunch of its common errors for us so why not add in some minimal handling via
[Either](http://danielwestheide.com/blog/2013/01/02/the-neophytes-guide-to-scala-part-7-the-either-type.html)?

```scala
// Type alias for readability's sake
type TradingApiResult[T] = Either[ApiError, T]

sealed trait ApiError
case class NotFound(error: String) extends ApiError
case class Unauthorized(error: String) extends ApiError
case class UnexpectedStatusCode(status: StatusCode) extends ApiError


import akka.http.scaladsl.unmarshalling.{ Unmarshal, Unmarshaller }
import akka.http.scaladsl.model.StatusCodes

// Constructor/etc elided

// `um` is provided by the previously mentioned `SprayJsonSupport`
// This is a prevalent theme in akka-related code: IMPLICITS, IMPLICITS EVERYWHERE.  Fun fact: this also requires
// an implicit ActorSystem and ActorMaterializer floating around!
private def deserialize[T](r: HttpResponse)(implicit um: Unmarshaller[ResponseEntity, T]): Future[TradingApiResult[T]] = {
  r.status match {
    case StatusCodes.OK => Unmarshal(r.entity).to[T] map Right.apply
    case StatusCodes.Unauthorized => Future(Left(Unauthorized(r.entity.toString)))
    case StatusCodes.NotFound => Future(Left(NotFound(r.entity.toString)))
    case _ => Future(Left(UnexpectedStatusCode(r.status)))
  }
}

// Use it in the API call!
def apiIsUp: Future[TradingApiResult[ApiStatus]] = {
  val source = Source.single(HttpRequest(uri = Uri(path = Path("/ob/api/heartbeat"))))
  val flow = Http().outgoingConnectionHttps("api.stockfighter.io").mapAsync(1) { r =>
    deserialize[ApiStatus](r)
  }

  source.via(flow).runWith(Sink.head)
}

// Repeat for rest of the endpoints
```

`apiIsUp` should now return an `Either[ApiError, ApiStatus]` unless something really bad (dare I say, *exceptional*?) happens.

### TODOs

The above is a nice start, but a few big TODOs stand out to me before I go on and toss this onto github.

First and foremost... Tests!  Testing libraries like this is always tricky since they're essentially all integration-y
glue code, but I have always been a big fan of the [vcr](https://github.com/vcr/vcr) gem in Ruby.  As far as I know the
closest thing in Scala is [betamax](https://github.com/betamaxteam/betamax), which I have not used but would like to.
(I know, I know--Not writing test firsts?  What about TDD?  BAD DEVELOPER!  \*rolls up newspaper\*)

Another big thing for me is domain modeling.  The built-in json deserialization is fine for working with row-level data,
but the plain case class format leaves a bit to be desired as far as robust data modeling.  As a simple example:

```json
// A simplistic "order request"
{ price: 0, qty: 0, direction: "buy" } // "buy" or "sell"
```

```scala
// naive translation:
OrderRequest(price: Int, qty: Int, direction: String)

// Preferable:
sealed trait OrderDirection
case object Buy extends OrderDirection
case object Sell extends OrderDirection

BetterOrderRequest(price: Int, qty: Int, direction: OrderDirection)
```

I haven't decided whether it would be better to add another step to the pipeline (e.g., `mapJsonToDomainObject`) or to
roll custom spray `JsonFormat`s to do this.

Lastly: websockets.  In [theory](http://doc.akka.io/docs/akka/2.4.4/scala/http/client-side/websocket-support.html)
websockets are supported, but the documentation is even sparser than for http clients and I haven't quite figured it out
yet--especially since the deserialization provided in `SprayJsonSupport` does not seem to work with the types used in
the websocket API.

### Overall Impressions

So far, my impression of `akka-http` is by and large the same as my impression of `spray`.  Actors (and now reactive
streams) provide a lot of power and performance in exchange for non-trivial complexity.  In my experience this tradeoff
is generally worth it for server-side/business application code, but lugging around an ActorSystem/etc ends up feeling
very clunky for a simple http client.  It doesn't help that the -client libraries seem to be the red-headed
step-children of both ecosystems.

The documentation feels consistent with the general API design.  That is: it tries to look simple for the most basic
use-cases, but in reality there is a lot of implicit stuff floating around.  It was basically a pre-requisite for me to
go digging for not only how streams worked conceptually but all the varied APIs that need to be used to link everything
together before I could unpack the examples in the client tutorials.  For example, while playing with the
[websocket tutorial](http://doc.akka.io/docs/akka/2.4.4/scala/http/client-side/websocket-support.html) I tried to switch
the `Sink.foreach` with a `Sink.queue` and got the following:

```
[error] (...)/TradingApiClient.scala:118: type mismatch;
[error]  found:
akka.stream.scaladsl.Sink[Nothing,akka.stream.scaladsl.SinkQueue[Nothing]]
[error]  required:
akka.stream.Graph[akka.stream.SinkShape[akka.http.scaladsl.model.ws.Message],akka.stream.scaladsl.SinkQueue[akka.http.scaladsl.model.ws.Message]]
[error]     outgoing.via(webSocketFlow).runWith(Sink.queue)
```

It's not the end of the world as I worked out the need for a type parameter (i.e., `Sink.queue[Message]`) but there a
lot of examples like this where the errors and tutorials are not exactly intuitive.  I can see this being a huge
deterrent to folks who are new to the ecosystem, to the concepts, or to Scala in general who will hit a wall and think,
"Wow, all this and I can't even open up a websocket/execute a json `POST`/etc?"  Or even worse--the example code will be
cargo-culted in by a harried developer on a deadline and carried on as the software version of the [five monkeys](http://c2.com/cgi/wiki?TheFiveMonkeys).
(This is not to say I could do any better.  Documentation and API design are some of the most underrated hard problems
in software today, IMO. :)

All my nitpicking aside, there is a lot to like about `akka-http`.  In exchange for all the effort involved in learning
about reactive streams and how to work with them, they provide a nice construct for abstracting away concerns like
back-pressure management.  This frees up developers to concentrate on the actual flow of the data.  The resulting code
is also quite clean and generally easy to follow, despite the time it took to actually get to that point.  In other
words, it trades off learning curve and ease of intuition for API comprehensiveness and composability.  `akka-http` is
especially nice on the server, where performance is a bigger concern.  I've built a couple of internal webservices with
`spray` previously, and it's always been fairly performant without excessive tuning on my part.  In addition, I've found
the concept of Directives and the server-side routing DSL to be quite nice to work with in the past.

Overall I would recommend `akka-http` unreservedly for writing web services and business applications.  My experience
with it on the server side has been quite good.  I would also use it again on the client side, mostly because there
don't seem to be any better options.  I had looked into some alternatives, but e.g. play-ws has the same overloaded
baggage problem and dispatch is like the poster-child of unintelligible symbolic operators (and seems unmaintained to
boot).  So until a better http client surfaces in the Scala ecosystem, one could do a lot worse.
