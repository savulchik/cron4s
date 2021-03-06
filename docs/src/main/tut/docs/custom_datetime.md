---
layout: docs
title: "Custom DateTime"
---

## Custom DateTime

As already mentioned in [Built-in libs](builtin_libs.html), **cron4s** comes with support for some DateTime libraries
out of the box. But implementations of DateTime are many, both in the JVM and in JavaScript and adding support for all
of them as a _batteries included_ is impossible. Also, asking all the potential users to stick to a very limited range
of libraries to be able to use **cron4s** is quite unrealistic.

To solve this, **cron4s** has a means to allow users to provide with ability to _plug-in_ their favorite library and
get all the profit that the library provides with very little effort. This is due to **cron4s**'s typeclass-driven
design and, in fact, there is nothing in the internals of the library tied to the _built-in_ lib support, all of them
are implemented the same way as we are going to see right now.

### `IsDateTime` typeclass

All comes down to one single (and very simple) typeclass: `cron4s.datetime.IsDateTime`. Providing an implicit instance
of this trait is enough to be able to use your favourite library.

As an example, we are going to implement support for a very dummy version of a _time_ object:

```tut:silent
case class MyTime(seconds: Int, minutes: Int, hour: Int)
```

Now that we have a way of representing time, we need to provide an instance for the `IsDateTime` typeclass:

```tut:silent
import cron4s._
import cron4s.datetime.IsDateTime

implicit object MyDateInstance extends IsDateTime[MyTime] {

  def supportedFields(myTime: MyTime): List[CronField] =
    List(CronField.Second, CronField.Minute, CronField.Hour)
        
  def get[F <: CronField](myTime: MyTime, field: F): Option[Int] = field match {
    case CronField.Second => Some(myTime.seconds)
    case CronField.Minute => Some(myTime.minutes)
    case CronField.Hour   => Some(myTime.hour)
    case _                => None
  }
  
  def set[F <: CronField](myTime: MyTime, field: F, value: Int): Option[MyTime] = field match {
    case CronField.Second => Some(myTime.copy(seconds = value))
    case CronField.Minute => Some(myTime.copy(minutes = value))
    case CronField.Hour   => Some(myTime.copy(hour = value))
    case _                => None
  }

}
```

That should be enough. To be sure that all works we are going to use it with a CRON expression:

```tut
val Right(cron) = Cron("* 5,15,30 6-12 * * *")
val myTime = MyTime(59, 15, 10)
cron.next(myTime)
cron.timePart.next(myTime)
```

### Testing

Running a series of expressions in a REPL is not enough to be 100% confident that our implementation is correct.
**cron4s** provides with the `cron4s-testkit` module which includes laws and base test classes to help in this task.
To start using it, just include it in your dependencies:

```scala
libraryDependencies += "com.github.alonsodomin.cron4s" %% "cron4s-testkit" % "x.y.z" % Test
```

The first thing to to is to provide an implementation of the `cron4s.testkit.DateTimeTestKitBase` trait, this
is meant to instruct **cron4s** on how to create arbitrary instances of your date time object:
 
```tut:silent
import cron4s.CronUnit
import cron4s.testkit.DateTimeTestKitBase
import org.scalacheck.{Arbitrary, Gen}

trait MyTimeTestBase extends DateTimeTestKitBase[MyTime] {
  import CronUnit._

  override implicit lazy val arbitraryDateTime: Arbitrary[MyTime] = Arbitrary {
    for {
      second <- Gen.choose(Seconds.min, Seconds.max)
      minute <- Gen.choose(Minutes.min, Minutes.max)
      hour   <- Gen.choose(Hours.min, Hours.max)
    } yield MyTime(second, minute, hour)
  }

  def createDateTime(seconds: Int, minutes: Int, hours: Int, dayOfMonth: Int, month: Int, dayOfWeek: Int): MyTime =
    MyTime(seconds, minutes, hours)

}
```

Also, to be able to perform assertions, **cron4s** needs an instance of the `scalaz.Equal` type class:

```tut:silent
import scalaz.Equal

implicit val myTimeEq: Equal[MyTime] = Equal.equalA[MyTime]
```

Now define a spec for your `IsDateTime` instance (along with your other test sources):

```tut:silent
import cron4s.testkit.IsDateTimeTestKit

class MyTimeSpec extends IsDateTimeTestKit[MyTime]("MyTime") with MyTimeTestBase
```

This will test that your implementation of the `IsDateTime` typeclass abides by its laws. To be fully covered is also
recommended to verify that the different CRON expressions work correctly with your date time library. In this case
we will be defining a class extending from `cron4s.testkit.DateTimeCronTestKit`:

```tut:silent
import cron4s.testkit.DateTimeCronTestKit

class MyTimeCronSpec extends DateTimeCronTestKit[MyTime] with MyTimeTestBase
```

And that's all, now those tests will be run along any other test sources in your project.