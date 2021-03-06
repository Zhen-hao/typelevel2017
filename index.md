---
title: Type-level Talk
author: Zhenhao Li
mode : selfcontained
knit : slidify::knit2slides
highlighter : highlight.js #highlight.js   #highlight  #prettify
framework: revealjs
hitheme : atom-one-dark  #solarized-light #rainbow #atom-one-dark #foundation #vs2015 #sunburst #zenburn #tomorrow
revealjs:
  theme: sky
  transition: concave #linear #concave
  center: "true"
url: {lib: "."}
bootstrap:
  theme: amelia
---

## The Power of Type Classes in Big Data ETL
### A Use Case of Combining Spark and Shapeless

[Zhenhao Li](https://www.linkedin.com/in/zhenhaoli/)
[<div><img width = '480px'   src='connecterra_black-01.png'></div>](http://www.connecterra.io/) 

<script src="http://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>

*** =pnotes

This is the title page.

--- 

### About Me


<section>

---

Data scientist and engineer at Connecterra since Feb 2017

---

1.5 year in consulting at Accenture

---

M.Sc. in logic and PhD-in-progress in mathematical logic 

---

B.Eng. in Software Engineer from China 

---

Two years experience with Scala

---
Learned about Shapeless in Dec 2016

</section>

--- &vertical

## Big data <small>@</small> [<img width = '460px'  src='connecterra_black-01.png'>](http://www.connecterra.io/) 
> - Our business is built on data and models
> - Each data science project has an ETL component  
> - We need flexible and extendable library for ETL


*** =pnotes

The boundary between ETL and feature engineering is blurred

--- &vertical

## Why Spark

> - Powerful API to write distributive algorithms 
> - Support for map-side reduce (RDD API only)
> - Good performance

***

### Performance Comparison

![plot of chunk unnamed-chunk-1](assets/fig/unnamed-chunk-1-1.png)


--- &vertical

## Why type level programming

> - Let the compiler help reduce programming errors
> - We want our code to be composable and reusable  
> - Libraries should be open to user-extensions

***

The task

```scala
case class Event(userID: Int, activityType: String, 
                timeStamp: Long, duration: Long)

val rawEvents: RDD[Event] = ???

def processSittingEvents(sittingEvents: RDD[Event]) = ???

def processWalkingEvents(walking: RDD[Event]) = ???

val sittingResult = 
    processSittingEvents(rawEvents.filter(_.activityType == "sitting"))

val walkingResult = 
    processWalkingEvents(rawEvents.filter(_.activityType == "walking"))
```

*** =pnotes

We are asked to produce aggregations for each type of activity. 
For "walking" we only care about the total duration of each user;
for "sitting" we want the total duration and the total event count of each user. 


***

Would be nice if we could write

```scala

case class Event(userID: Int, activityType: String, 
                timeStamp: Long, duration: Long)

val rawEvents: RDD[Event] = ???

def processEvents[T](rawEvents: RDD[Event]) = ???

val sittingResult = processEvents[Sitting](rawEvents)

val walkingResult = processEvents[Walking](rawEvents)

```

--- &vertical

### Singleton types

```scala
import shapeless.syntax.singleton._

val sittingWit = "sitting".witness
type Sitting = sittingWit.T

val walkingWit = "walking".witness
type Walking = walkingWit.T
``` 

*** =pnotes

A type is a set(collection) of values.
A singleton type is a type containing only one value. 
Hence the value is inferred from the type. 

***

### Type classes

```scala
trait Aggregator[T]{
    type Result
    type Representation
    def name: String
    def extract(rawEvents: RDD[Event]): RDD[Representation]
    def aggregate(internalReprRDD: RDD[Representation]): Result
    def process: RDD[Event] => Result = aggregate _ compose extract
}
```

***

### Type classes

```scala
object Aggregator{
    implicit def basicActivityAggregator[A <: String]
        (implicit wtA: Witness.Aux[A]): Aggregator[A] = new Aggregator[A]{
            type Result = RDD[(Int, Double)] //userID and total duration
            type Representation = (Int, Double)
            def name = wtA.value
            def extract(rawEvents: RDD[Event]) =
                rawEvents
                  .filter(_.activityType == name)
                  .map(e => (e.userID, e.duration))
            def aggregate(internalReprRDD: RDD[Representation]) =
                internalReprRDD
                  .aggregateByKey(0.0)(_ + _, _ + _)
    }
}
``` 

***

### Type classes

```scala
object AggregatorsWithCounts {
    import shapeless._
    def basicActivityAggregatorWithCounts[A <: String]
    (implicit wtA: Witness.Aux[A]): Aggregator[A] = new Aggregator[A]{
        type Result = RDD[(Int, (Double, Int))] //userID and total duration
        type Representation = (Int, (Double, Int))
        def name = wtA.value
        def extract(rawEvents: RDD[Event]) =
            rawEvents
              .filter(_.activityType == name)
              .map(e => (e.userID, (e.duration, 1)))
        val monoid = implicitly[Monoid[(Double, Int)]]
        import MonoidSyntax._
        def aggregate(internalReprRDD: RDD[Representation]) =
            internalReprRDD
              .aggregateByKey(monoid.zero)(_ |+| _, _ |+| _)
    }

}
``` 

***

### Type classes

```scala
def processEvents[T](rawEvents: RDD[Event])
    (implicit aggregator: Aggregator[T]): aggregator.Result
        = aggregator.process(rawEvents)
        
val rawEvents: RDD[Event] = ???

implicit val sittingAggregator: Aggregator[Sitting] 
            = AggregatorsWithCounts.basicActivityAggregatorWithCounts[Sitting]
val sittingResult2 = processEvents[Sitting](rawEvents)
val walkingResult2 = processEvents[Walking](rawEvents)
```

<small>[Full tutorial link](https://gist.github.com/Zhen-hao/26edb91668c286eed7a2864912a59c35)</small>

--- &vertical

### What we achieved in our ETL library

```scala
implicit val rawRDD: RDD[Event] = ???
val intctRDD = implicitly[IntercectionActivityRDD]

val sittingEatingResult: RDD[(Key, SimpleAggregation)]
    = intctRDD.intercectionAggregation[Key, Sitting, Eating]
```

***

### Under the hood

```scala
trait ActivityIntercectionMonoid[A <: String, B<: String] 
    extends Monoid[ActivityIntercection[A, B]] {
    implicit def orderingA: Ordering[ActivityWithType[A]]
    implicit def orderingB: Ordering[ActivityWithType[B]]
    implicit def orderingCombin: Ordering[CombinedActivity[A, B]]
}

object ActivityIntercectionMonoid{
    implicit def getActIntMonoid[A <: String, B<: String]
        (implicit wtA: Witness.Aux[A], wtB: Witness.Aux[B])
        = new ActivityIntercectionMonoid[A,B]{
            ???
        }
}
```


***

### Under the hood

```scala
trait KeyedActivityRDD[K, A] extends Serializable{
    implicit val keyedRDD: RDD[(K, A)]
    def aggregation[S](implicit classTagS: ClassTag[S], 
        lift: A => S, monoid: Monoid[S]): RDD[(K,S)]
}
object KeyedActivityRDD{
    implicit def apply[K, A](implicit classTagK: ClassTag[K], 
        classTagA: ClassTag[A], rdd: RDD[(K, A)])
        : KeyedActivityRDD[K,A] = new KeyedActivityRDD[K,A]{
        implicit val keyedRDD = rdd
        implicit def aggregation[S](implicit classTagS: ClassTag[S], 
        lift: A => S, monoid: Monoid[S]): RDD[(K,S)] ={
            import MonoidSyntax._
            keyedRDD.mapValues(lift)
                    .aggregateByKey(monoid.zero)(_ |+| _, _ |+| _)
        }
    }
}
```

***

### So we can do

```scala
case class SittingAggregation(totalDuration: Double, eventCount: Int)

implicit rawData: RDD[(K,A)] = ???
val result = implicitly[ActivityIntercectionMonoid[K,A]]
                .aggregation[SittingAggregation]
```


--- &vertical

# Conclusions

***

### Pros

> - Typelevel programming enables the compiler to catch programming errors  
> - Can help build a higher level API or DSL  
> - Separate how from what
> - Compostable abstractions

***

### Cons

> - Users need to appreciate types and compiling errors
> - Users must understand how implicits work
> - Danger of ambiguity errors when multiple processes exist sharing input or output data types.

*** 

## Thank you! 

<a href="#/2" class="image">
  <img src="cph.jpg">
</a>


