---
layout: post
title: "Testing Play without FakeApplication"
date: 2013-12-07 14:37
comments: true
categories: [play, scala, testing]
---

The testability of our application is very dear to us. We are currently using Play-Scala on a large project for almost a year now. There is very little documentation about different ways to test a play app without having to depend on the framework provided helpers like `FakeApplication`. 

The `FakeApplication` helper does its job as expected by starting a mock application in a test which depends on presence of a running app. It is a great utility to test your routes and controllers. But there are other variants of integration tests that merely read from a config file but required us to start a `FakeApplication`, which was puzzling. One can argue that reading from a config file makes it an integration test just like testing a route so why not do that. But running a `FakeApplication` for the large number of tests we had accumulated has some downsides:

* It adds verbosity because each single test must be wrapped inside `WithApplication` helper that internally creates a `FakeApplication`, starts it and stops it after that that test has finished
* There is a performance overhead in starting and stopping the application each time. Given a large amount of tests we have, it increased the test run time significantly

The reason a module needs a running application in the first place is because of dependency on `Play.current` which provides an implicit instance a play app. For example, consider a module which reads all settings from a config file and captures them into named variables[1]:

``` scala
trait PlayConfigComp {

  lazy val playConfig = new PlayConfig

  class PlayConfig {

    import Play.current
    private val underlying = Play.configuration.underlying

    val dateFormatString = underlying.getString("publishing.date.format")
    val publishingBatchSize = underlying.getInt("publishing.batch.size")
    ....
  }
}
```

Due to the way it is implemented, `Play.configuration` requires an implicit instance of a running app which is provided by importing Play.current. Now duing a test run for another module which depends on `PlayConfigComp` (a very common scenario) if we do not have a running app, Play.current will throw an error[2]

Before we actually write a test, lets create a module which depends on `PlayConfigComp`. For example, a service which converts a `Date` between its `String` and `Long` representations must get the date-format from the `PlayConfigComp` like so:

``` scala
trait DateServiceComp extends PlayConfigComp {

  lazy val dateUtil = new DateService

  class DateService {

    def parse(date: String) = dateFormat.parse(date).getTime
    def format(timeInMillis: Long) = dateFormat.format(timeInMillis)

    private def dateFormat = new SimpleDateFormat(playConfig.dateFormatString)
  }
}
```

To be able to test the `DateServiceComp`, Play framework provides a nice abstraction called `WithApplication`. It can be mixed in with each test to handle creation, starting and stopping of the a mock application. The tests look like this:

``` scala
"date service" should {
  
  "parse with a fake app" in new WithApplication with DateServiceComp {

     dateService.parse("2013-05-15 00:30:00") mustEqual 1368570600000L

  }

  "format with a fake app" in new WithApplication with DateServiceComp {

     dateService.format(1368570600000L) mustEqual "2013-05-15 00:30:00"

  } 
}
```

Can you imagine what will happen if we had 10 tests instead of just 2 for the date service and if we have dozens of such services? We will be adding that much boilerplate and it will also impact the total run time for tests. 

So how do we solve this? To answer that, we must remember that the main reason we have to have a running play application in tests is our direct dependency on the `Play.current`. Which gives us a hint that if we factor out the dependency on the `Play` singleton in a separate module and stub out that module, we can achieve what we want. In our case, the extracted module looks like this:

``` scala
trait PlayAppComp {

  lazy val playApp = new PlayApp

  class PlayApp {

    implicit def app = Play.current

    def isProd = Play.isProd
    def configuration = Play.configuration
    def getFile(path: String) = Play.getExistingFile(path).getOrThrow(s"File not found at $path.")

    def getConfig(path: String) = ConfigFactory.parseFile(getFile(path))
  }
}
```

Because we have a separate module, we take this opportunity to encapsulate not only the troublesome `Play.current` and `Play.configuration` but also a few other methods on `Play` singleton that transitively depend on `Play.current`. Now `PlayConfigComp` can depend on the new `PlayAppComp` without requiring implicit instance of `Play.current`:

``` scala
trait PlayConfigComp extends PlayAppComp {
  
  lazy val playConfig = new PlayConfig
  
  class PlayConfig {

    private val underlying = playApp.configuration.underlying
    
    val dateFormatString = underlying.getString("publishing.date.format")
    val publishingBatchSize = underlying.getInt("publishing.batch.size")
    ....
  }
}
```

For the tests, lets stub out `PlayAppComp`. Stub also gives us the flexibility to override other methods. We specify a different conf file (`test.conf`) instead of default `application.conf` and set `isProd` to false[3]. These are added benefits of the refactoring.

``` scala
trait StubPlayAppComp extends PlayAppComp {

  override lazy val playApp = new StubPlayApp

  class StubPlayApp extends PlayApp {

    override def isProd = false
    override def configuration = Configuration(getConfig("conf/test.conf"))
    override def getFile(path: String) = new File(path)
  }
}
```

Using this stub, we can finally run our tests without having to start a fake app! Also, note that we can now create the entire test assembly exactly once per suite and reuse across tests. This also reduces boilerplate and also helps in better compilation times which becomes an issue while using Scala on a large code base (topic for yet nother post, I guess).

``` scala
"date service" should {

  object Assembly extends DateServiceComp with StubPlayAppComp
  import Assembly._
  
  "parse with a fake app" in {

    dateService.parse("2013-05-15 00:30:00") mustEqual 1368570600000L

  }

  "format with a fake app" in {

    dateService.format(1368570600000L) mustEqual "2013-05-15 00:30:00"

  } 
}
```

If you are a Play-Scala developer who ecnoutered the same issue, I hope this blog post gives your one possible solution.There are a lot of other topics around a Play-Scala testing like compile-time reduction, sbt-config settings and dependency resoltuon that I hope to write about in next blog posts.


[1] The reason we structured the code as shown above (a class encapsulated inside a trait) is because we are using Scala's inbuilt name based dependency resolution mechanism. We could have very well used a DI framework like Guice or Spring, but this approach is very flexible and deserves a separate blog post.

[2] One way to solve it could be to convert these tests into pure unit tests by mocking PlayConfig which is what we did in cases where it made sense. The objective here is how to show how to get around the problem where we do want to read from a real config file.

[3] `isProd` is used by the application in a few places to conditionally change behaviour for production mode. 
