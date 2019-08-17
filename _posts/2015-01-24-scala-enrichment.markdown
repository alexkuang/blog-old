---
layout: post
title: "Scala Enrichment"
date: 2015-01-24 11:52:37 -0500
comments: true
categories:
- programming
- scala
- totw
---

Douglas Crockford recently gave a tech talk at work, where he casually endorsed Scala during the Q&A at the end.  Given
that and the fact that I've been hearing increasing mentions of it in the company at large, I figured this week I'd plug
a neat feature of Scala and a recent use case where I found it extremely handy.

### The Feature

Anyone who's worked with The AWS SDK (or just Java code in general) will be familiar with the builder pattern.
Recently, I was writing some code to automate setup of CloudWatch alarms for a DynamoDB table.  The alarm request
started looking something like:

```scala
val req = new PutMetricsAlarmRequest()
  .withNamespace("DynamoDB")
  .withDimensions(dimensionsForTableName)
  .withStatistic("Sum")
  .withComparisonOperator(GreaterThanThreshold)
  .withThreshold(thresholdNumber)
  .withMetricName(throttledMetricName)
  .withEvaluationPeriods(periods)
  .withPeriod(periodDuration.toSeconds.toInt)
  .withAlarmActions(idMappingSNS)
  .withAlarmName(alarmNameFor(tableName, throttledMetric, throttledThreshold))

cloudwatchClient.putMetricAlarm(req)
```

Which is serviceable, but less than ideal given that I had multiple alarms of roughly the same nature.  Plus, I felt
like it left something to be desired in terms of readability and communicating the intent of the alarm in a succinct
way.  Enter `implicit class`s and the "enrichment"* pattern!  Basically, it lets us turn the above into something more
like this:

```scala
val req = new PutMetricAlarmRequest()
  .forTable(tableName)
  .triggerOnSumGreaterThan(throttledMetric, throttledThreshold)
  .afterEvaluationPeriods(evaluationPeriodDuration, evaluationPeriods)
  .withAlarmActions(idMappingSNS)
  .withAlarmName(alarmNameFor(tableName, throttledMetric, throttledThreshold))

cloudwatchClient.putMetricAlarm(req) /** Can still pass req back into AWS API! */
```

With the addition of this:

```scala
implicit class MetricAlarmRequestHelper(req: PutMetricAlarmRequest) {
  def forTable(tableName: String) = {
    req.withNamespace("DynamoDB").withDimensions(tableMetricDimensions(tableName).asJava)
  }

  def triggerOnSumGreaterThan(metricName: String, threshold: Int) = {
    req.withStatistic("Sum").withComparisonOperator(GreaterThanThreshold).withThreshold(threshold).withMetricName(metricName)
  }

  def afterEvaluationPeriods(periodDuration: Duration, periods: Int) = {
    req.withEvaluationPeriods(periods).withPeriod(periodDuration.toSeconds.toInt)
  }
}
```

<!-- more -->

### How It Works

One of the coolest--and probably most confusing--keywords in scala is `implicit`, which can refer to many different
things.  For now, let's limit the discussion to implicit conversions.  A grossly oversimplified tl;dr is that there can
be some def of the form:

```scala
implicit def foo2bar(foo: Foo): Bar = { … }
```

And as long as that def is in scope, the code will convert anything of type Foo into type Bar without having to re-write
that logic or call some conversion method.  For more information on implicits in general, see this excellent answer by
Daniel Sobral, who is basically the Jon Skeet of the Scala world:
http://stackoverflow.com/questions/5598085/where-does-scala-look-for-implicits/5598107#5598107

Extending the use of implicits, that means that if you do something like, for the above use-case:

```scala
class MetricAlarmRequestHelper(req: PutMetricAlarmRequest) = { /** Same function defs as above */ }

implicit def vanillaRequest2Helper(req: PutMetricAlarmRequest) = new MetricAlarmRequestHelper(req)
implicit def helper2vanillaRequest(helper: MetricAlarmRequestHelper) = helper.req
```

then scala will be able to magically convert from the vanilla request to the helper for use in your client code, and
then from the helper back to the vanilla request for passing to other parts of the Amazon API. `implicit class` is just
short-hand introduced in Scala 2.10 that does the above for you in one convenient construct that makes things even more
concise.  For more info, see the scala docs: http://docs.scala-lang.org/overviews/core/implicit-classes.html

Beyond use cases like wrapping builders, this kind of enrichment using `implicit` can be extremely powerful, especially
for extending functionality where it's not practical to alter the original code.  Though as with all advanced features
of anything, it's probably best not to go overboard.  :)

### Naming

Quick bit of bonus trivia...  When the pattern first rose into prominence, it was known colloquially as "pimp my class".
Then folks got all up in arms about the political correctness of the word "pimp", so now the common term is
"enrichment".
