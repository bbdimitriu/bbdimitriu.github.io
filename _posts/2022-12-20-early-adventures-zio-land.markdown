---
layout: post
title:  "Early adventures into ZIO land"
date:   2022-12-20 01:01:53 +0000
categories: scala zio
toc: true
---

* TOC
{:toc}

## How I got interested in ZIO
I've been writing Scala applications for more than a decade now, but I wouldn't call myself anywhere close to an expert in it.
I really do enjoy coding in Scala whenever I have a chance, although this doesn't happen very often these days, given that I spend a lot of time in the world of cloud infrastructure and architecture (more on that some other time, hopefully).

I also generally enjoy taking a functional approach to writing code, but I've never been a purist, I was looking for elegant but quick solutions.
I had heard about Cats Effects and ZIO for a couple of years, but I didn't get to look properly into what the buzz was about, until I saw that Daniel Ciocîrlan of [Rock the JVM](https://rockthejvm.com/) released a [course on ZIO](https://rockthejvm.com/p/zio).
Daniel's Scala courses, videos and articles are just excellent. I thoroughly recommend them to anyone who wants to learn about Scala in a quick and pragmatic way.
Better yet, this is a course about ZIO 2, which was released in June 2022, rather than about ZIO 1. And ZIO 2 I understand is a fairly big improvement over ZIO 1. Check out John De Goes' post [https://degoes.net/articles/zio-2.0](https://degoes.net/articles/zio-2.0) for more details as to why. 
So, I subscribed to his ZIO course. And it was totally worth it! I got to learn about most of the ZIO features in a fun, quick and practical way.
And it really was a bit of a revelation for me, pretty much in the same way the Spring framework was a revelation when it came to save the Java world from the madness of EJBs :).

## What is ZIO 2 and why you should pay attention to it?

ZIO is a framework centered around the idea of "[effect systems](https://en.wikipedia.org/wiki/Effect_system)" and the [IO monad](https://en.wikipedia.org/wiki/Monad_(functional_programming)#IO_monad_(Haskell)).
Designing an application based on this idea confers some pretty nice advantages that have become more obvious to me as I progressed through the course, but it does fundamentally change the way one should organise and think about their code.
In some way, to me, it seems that programming with an effects framework like ZIO is like building a complex pipeline (your actual program) by interconnecting small specialised pipes (the effects) that may have an input (another effect's output) and an output (the effect's return type **A**).
These pipes can [fail in expected ways](https://zio.dev/reference/error-management/types/failures/) (the effect's error type **E**) or in unexpected ways ([Defects](https://zio.dev/reference/error-management/types/defects)) and the failures should be explicitly modelled and handled in the pipeline design.
The pipes (effects) can also optionally have specific features that they require in order to function, so they must be provided by the program. These constitute the effect's environment (the type **R**). You can think of these as smart electronic chips that are required by the individual pipe to actually do what it is supposed to.
This pipe architecture doesn't do anything until data starts flowing through it. That is until we "run" it with the "effects engine".

In a nutshell this is what a `ZIO[-R, +E, +A]` is about. Of course, I'm over-simplifying things, there is a lot more depth packed into this apparently basic construct.

But this blog is not meant to be a ZIO tutorial, so let's talk about some practical experience with it :)

## The application

Fairly recently I got to work on a relatively simple component, that was meant to handle a message processing pipeline that I could sum up this way:
_Consume a JSON message from AWS Kinesis ~~> Parse the JSON and extract an S3 location ~~> Fetch the S3 object from that location ~~> POST the object to an HTTP server and read the result_.
Nothing fancy here.

The component does need to handle a fairly high and sustained rate of processed messages per second (in the thousands).
There are some implicit constraints:
* there shouldn't be more Kinesis application consumers (effectively JVM processed) than there are Shards
* the files that are sent via the POST request can be of various sizes, sometimes in megabytes, so I decided it was wise to design the application to stream directly from S3 to the HTTP server (not buffering the files in memory)
* because of the number of parallel requests per second, I decided to go for a non-blocking I/O approach
* the latency doesn't matter, but the throughput needs to be high

Note: I developed this component with Scala 3.2.1, JDK 17/19 and ZIO 2.0.2 and it is run on a EKS cluster in AWS.

## Error management

The first ZIO feature I got hooked on was [error management](https://zio.dev/reference/error-management/).
Previously I would have used constructs like Try, Either, Option to handle errors.
These constructs are great and they make functional composition of errors much more elegant than using try/catch blocks (which are actually not composable at all).
But ZIO's error handling mechanisms goes to the next level.
They provide a domain-driven, unified, functional, explicit, type-safe way of dealing with failures.
Handling errors with ZIO for my application felt like a breeze and the code felt easy to read and crucially, easier to test.
Just make sure to follow the "[Best practices](https://zio.dev/reference/error-management/best-practices/algebraic-data-types)" section from the ZIO reference guide.
Based on this, I think that one of the most important early decisions you need to take is to **_model your domain errors_**.

Intermezzo: I mention things that "compose well" and "functional composition". What does it mean and why is it a big deal?
Coming back to the "pipes" analogy - we want to construct a program from smaller pipes, each having a specific role and we want to be able to join them together neatly so that the information can flow through this architecture correctly.
These pipes are reusable (because the effects just describe code, they don't actually run it) and they can be reused and composed with other pipes in whatever ways we see fit.
Mathematically functions compose well and this is the model that the effects take as well. And this is because Effect instances (as in `ZIO[-R, +E, +A]`), just like functions, _do not have side effects_.
It's only the code they wrap that may have side effects (like reading from a socket or modifying a file, or even printing something on the console).

## Dependency injection
Most JVM developers would be familiar with the concept of "dependency injection" which was made popular by the Spring framework.
The main idea is that when you assemble the application (at the top level), you loosely couple its components, so that you can easily swap different implementations for various situations.
This is also called "_inversion of control_".

ZIO also provides its own ability to wire together an application, but it does so "natively", using `ZLayer[-RIn, +E, +ROut]`, which has many similarities to `ZIO[-R, +E, +A]`.

I must admit that in the beginning I wasn't quite sure about using this feature in my application, primarily because I didn't feel the need to use an IoC framework for a very long time.
However, with most of the libraries in the ZIO ecosystem there is no getting away from understanding and using `ZLayers` (and `ZEnvironment`).
And fairly quickly I actually got to like it.
I think that the feature that contributed mostly to that was [Automatic layer construction](https://zio.dev/reference/di/automatic-layer-construction/).
In terms of what it achieves it is similar to the [Spring Autowired](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/annotation/Autowired.html) annotation, but I find the ZIO approach more elegant because the developer experience is better.
There is no metaprogramming involved and everything is statically type-checked.
You even get really nice compiler messages when you forget to provide some required layers.

Manual compositions seems to be nice as well, although I haven't used it much so far.

It takes a bit of getting used to, especially understanding sometimes unexpected environment requirements, but the compiler always points you into the right direction, because these layers always compose in a functionally correct way.

As you may expect, the ZIO runtime also manages the lifecycle of the layers, creating and destroying them (if required) automatically in the appropriate order
`ZLayer.scoped` wrapping `ZIO.acquireRelease` is very useful in these situations, as in the example below, in which a [Testcontainers](https://www.testcontainers.org/) [Localstack](https://www.testcontainers.org/modules/localstack/) container is wrapped in a ZLayer and it is started and stopped by ZIO automatically:
```scala
private val localStackContainerLayer: ZLayer[Any, Nothing, LocalStackV2Container] =
  ZLayer.scoped {
    ZIO.acquireRelease(ZIO.succeed {
      val localStackContainer = LocalStackV2Container(services = Seq(Service.S3, Service.KINESIS, Service.DYNAMODB, Service.CLOUDWATCH), tag = "1.3.0")
      localStackContainer.start()
      localStackContainer
    })(container => ZIO.succeed(container.stop()))
  }
```
In fact `ZIO.acquireRelease` and the other [resource management]((https://zio.dev/reference/resource/scope)) features are themselves very powerful constructs. 

Here are some "gotchas" that tripped me for a while until I understood how things should work:

### Using [ZIO.provideSomeLayer](https://javadoc.io/static/dev.zio/zio_3/2.0.5/zio/ZIO.html#provideSomeLayer-fffff1e1) or [Spec.provideSomeLayer](https://javadoc.io/doc/dev.zio/zio-test_3/latest/zio/test/Spec.html#provideSomeLayer-5d8)
Sometimes you don't want to provide all your environment dependency "at the end of the world" (i.e. at the root of your application) and you need to split your environment so that some of it is provided in one place and the rest of it in another place.
I couldn't find many examples of how this should be used, but as usually you can find the ultimate information in the API docs.
This was the case when I was writing a test suite where all the tests were supposed to share a set of layers, but also each test had its own layers that needed to be provided. Something like this:
```scala
trait Database

trait Logging

trait Connection

trait S3

trait Tracing

val databaseLayer: ZLayer[Any, Nothing, Database] = ???
val loggingLayer: ZLayer[Any, Nothing, Logging] = ???
val connectionLayer: ZLayer[Any, Nothing, Connection] = ???
val s3Layer: ZLayer[Any, Nothing, S3] = ???
val tracingLayer: ZLayer[Any, Nothing, Tracing] = ???

override def spec = suite("Stub Ripper Spec")(
  test("Test that uses a database") {
    // ... the effect under test with type ZIO[Database & Connection & Tracing & Logging, Nothing, String]
  }.provideSomeLayer(databaseLayer >+> connectionLayer), // here we compose two layers together to have a combined output of type ZLayer[Any, Nothing, Database & Connection]
  test("Test that uses S3") {
    // ... the effect under test with type ZIO[S3 & Tracing & Logging, Nothing, String]
  }.provideSomeLayer(s3)
).provideShared(
  tracingLayer,
  loggingLayer
)
```
The same works with plain ZIO effects rather than ZIO test specs.

### Providing the environment to an effect somewhere in the program
When using the STTP HTTP client with the ZIO backend I had a situation where I had to provide the environment for a ZStream[S3, Nothing, Byte] to make it a ZStream[Any, Nothing, Byte], because that's what the STTP API works with.
I suppose this is a deficiency of STTP, because it should allow the use of a generic environment `R` for the ZStream to be POST-ed, but it's the way the API is at the moment.
And I couldn't use `ZIO#provideLayer(s3Layer)`, because I didn't have a reference to the `s3Layer` in that part of the code.

It took me a while to figure out how to achieve that, but in the end the solution is relatively simple: I had to use `ZIO.environment[S3]`
See the example below, in which I tried to reduce to the essence of the problem as much as possible:
```scala
import sttp.capabilities.WebSockets
import sttp.capabilities.zio.ZioStreams
import sttp.client3.{Response, SttpBackend, basicRequest}
import sttp.model.Uri
import zio.*
import zio.s3.S3
import zio.stream.ZStream

object SttpPostExample extends ZIOAppDefault {

  private def program = {
    val streamWithoutS3Environment: ZStream[S3, Nothing, Byte] = ??? // <- we construct a ZStream that requires an `S3` layer

    for {
      s3Environment <- ZIO.environment[S3] // extract an environment with the requirements of the stream
      streamWithCompleteEnvironment: ZStream[Any, Nothing, Byte] = streamWithoutS3Environment.provideEnvironment(s3Environment) // <- provide all the environment requirements to the ZStream
      response <- Uploader.upload(uri = ???, streamWithCompleteEnvironment)
    } yield response
  }

  override def run = {
    val s3Layer: ZLayer[Any, Nothing, S3] = ???
    val sttpLayer: ZLayer[Any, Nothing, SttpBackend[Task, ZioStreams & WebSockets]] = ???

    program.provide(s3Layer, sttpLayer)
  }
}

object Uploader {

  def upload(uri: Uri, content: ZStream[Any, Nothing, Byte]): ZIO[SttpBackend[Task, ZioStreams], Throwable, Response[Either[String, String]]] =
    for {
      backend <- ZIO.service[SttpBackend[Task, ZioStreams]]
      response <- backend.send(basicRequest.post(uri).streamBody(ZioStreams)(content)) // !!! The STTP API requires a ZStream[Any, Nothing, Byte]
    } yield response
}
```

## Streaming with `ZStream`
For streaming, I had two requirements in my program, described below.
ZStream has very similar APIs to the ZIO type, so most of the compositional constructs come naturally.

### Streaming data from an AWS Kinesis stream
For this purpose I used the [zio-kinesis](https://github.com/svroonland/zio-kinesis) library because it has an API built on top of [zio-streams](https://zio.dev/reference/stream/) and it offers the same high-level consumer features as the official [KCL library](https://docs.aws.amazon.com/streams/latest/dev/kcl2-standard-consumer-java-example.html).
The library is of a good quality from what I can see so far, so I didn't have any issues in getting it to work as I needed.
The shards are processed in parallel between them, but by default the data within a shard is processed sequentially.
However, because I didn't care about the processing order even for elements within the same shard, I could easily parallelise the processing of each individual record, as in the following code:
```scala
def getKinesisStream[R, E, T](recordHandler: Record[String] => ZIO[R, E, T]) =
  Consumer
    .shardedStream(
      streamName = "my_stream",
      applicationName = "my_application",
      deserializer = Serde.asciiString,
      workerIdentifier = "my_worker_identifier", // make sure you use a unique name for each worker
      initialPosition = Consumer.InitialPosition.Latest
    )
    // we always want to process each shard in parallel, that's why we use a "safe" value of `Int.MaxValue` in `flatMapPar`
    .flatMapPar(Int.MaxValue) { case (_, shardStream, checkpointer) =>
      shardStream
        // processing the elements from the same shard in parallel too, because we don't care about the order
        // the level of parallelism here affects how many concurrent requests Cape server will receive
        // the maximum number of parallel requests will be `numberOfShards * myStreamProcessingParallelism`
        .mapZIOParUnordered(myStreamProcessingParallelism)(record =>
          recordHandler(record)
            .ignore // here we don't care here if the element scanning succeeded or failed, we just don't want to fail the stream
            .as(record)) // transforming the stream back to a stream of Records so that we can checkpoint it
        .tap(tuple => checkpointer.stage(tuple))
        .viaFunction(
          checkpointer.checkpointBatched[R](nr = kinesisConfig.checkpointBatchSize, interval = kinesisConfig.checkpointDuration)
        )
    }
```
The only thing that was a bit trickier here was deciding how to test the above, code.
The type of stream above is `ZStream[Any, Throwable, Unit]`, because of the checkpointer transformation at the end.
This means that I cannot just put some records into the Kinesis stream and use the stream above to read them on the other side of the pipe.
But there is at least one easy solution: make the `recordHandler` function add at the end the element that it processes to a [ZIO Queue](https://zio.dev/reference/concurrency/queue) and then the elements can be de-queued in the test to do te assertions. 

### Streaming a file from an AWS S3 location and sending it over to an HTTP server using a POST request
I had no problems in this area. I have used the [zio-s3](https://github.com/zio/zio-s3) library to obtain a ZStream reference to an S3 object and then I passed that reference to an [STTP client with a ZIO backend](https://sttp.softwaremill.com/en/latest/backends/zio.html) when I needed to make the POST request.
It works well, it is non-blocking and the API is very simple:
```scala
val objectLocation = ??? // the URI of the S3 object
val myObjectStream: ZStream[Any, Throwable, Byte] = 
  ZStream.fromZIO(ZIO.attempt(new URI(objectLocation))).flatMap(uri => getObject(uri.getHost, uri.getPath.substring(1)))
  
for {
  backend <- ZIO.service[SttpBackend[Task, ZioStreams]] // the backend is provided from the environment
  response <- backend.send(
    basicRequest.post(capeUri)
      .streamBody(ZioStreams)(myObjectStream)
      .readTimeout(scala.concurrent.duration.Duration(capeConfig.capeReadTimeoutMs.toLong, TimeUnit.MILLISECONDS))
      .headers(Map(
        "Header1" -> "value1",
        "Header2" -> "value2"
      ))
  )
} yield response
```

## Testing
For the most part, the zio-testing is very straightforward to use, but this is one area where I had some issues for a while, not because the ZIO testing framework if buggy, but because I misunderstood the documentation and because it was a bit difficult to find examples that matched my requirements.

First, I found it a bit odd how the tests in a suite are structured, all in a single call to the `suite` method, but I got used to it and it didn't make things more difficult.
```scala
override def spec = suite("My suite")(
  test("Test 1") {
    for {
      o1 <- effect1
      o2 <- effect2
    } yield assertTrue(o1 == o2)
  }.provideSomeLayer(myLayer)
    @@ TestAspect.before(someSetupToRunBeforeTest())
    @@ TestAspect.timeout(20.seconds))
```
The main thing that caused me some headaches is the fact that by default in ZIO tests _"Calls to sleep and methods derived from it will semantically block until the clock time is set/adjusted to on or after the time the effect is scheduled to run"_.
This is actually documented clearly [here](https://zio.dev/reference/test/services/clock), but in my test I didn't really need this behaviour and I wanted the time to pass as in real life.
So it took me some time to figure out why my test, which uses some zio-streams test utility classes that call `ZIO.sleep`, got stuck.

Changing this behaviour to the one I expected was pretty simple: I had to add the `@@ TestAspect.withLiveClock` aspect to the respective test.
It would have helped if this behaviour was more clearly stated somewhere at the beginning of the testing reference documentation.

Otherwise, in general, I consider the design of the testing framework quite neat and pleasant to use.

## Debugging
Debugging can be done in various ways. Most of the practices described in the ZIO guide: https://zio.dev/guides/tutorials/debug-a-zio-application/.

In general, I found exception stack traces to be useful, although I have bumped into a situation in which I couldn't understand much out of an exception (see [this](https://github.com/svroonland/zio-kinesis/issues/797)).
I have also noticed the rendering of the exception stacktrace getting better in between releases.

One important thing that I'm still missing at the moment, which I found really useful with the synchronous/blocking JVM applications, is the ability to easily request a Fiber dump.
I've been many times into a situation where the application gets stuck and I have no idea why, especially if it's some code from some third party library.
With blocking applications one can request a thread dump with `kill -3 <PID>` or `jstack <PID>` and can find out where the blockage is.
With non-blocking applications this approach is mostly useless, of course.

ZIO does have the ability to request a dump of all fibers using [Fiber.dumpAll](https://zio.dev/api/zio/fiber$#dumpAll(implicittrace:zio.Trace):zio.ZIO[Any,java.io.IOException,Unit]), but I couldn't find a way to trigger it externally from the application.
There is [this ticket](https://github.com/zio/zio/issues/2263), which has been open for a while, but it looks like it got de-prioritised.
So, unless I'm missing some better approach to achieve this, I think that this is something really important to be addressed by the ZIO devs.
Perhaps with the advent of [Project Loom](https://openjdk.org/projects/loom/) this issue will be neatly solved, given how well positioned the ZIO 2 engine is to take advantage of the new Virtual Threads.
But I wish there was an easier way available now.

## Logging

The usual libraries I reach out for when I need to do logging in my Scala application are `logback`, `slf4j` and occasionally `scala-logging`.
With ZIO you will want to do effectful logging, therefore you need to use [zio-logging](https://zio.dev/zio-logging/).

In my case I needed the application to log in JSON format to `stdout` (because we run it in ephemeral Kubernetes pods and logs are collected with FluentBit)
Initially, probably due to inertia, I rushed to configure Logback + SLF4J with `logstash-logback-encoder` (for the JSON format).
However, due to an issue I encountered (and which I didn't manage to get to the bottom of), I realised that I don't in fact need any of this.
ZIO Logging can do JSON format to stdout just as well:
```scala
import zio.logging.*

object MainZIO extends ZIOAppDefault {
  
  override val bootstrap: ZLayer[ZIOAppArgs, Any, Any] = Runtime.removeDefaultLoggers >>> consoleJson(LogFormat.default)

  override def run = ZIO.logError("Some error")
}
```
Also, the nice thing about using this approach is that now in the logs under the `thread` attribute I can actually see the Fiber ID (e.g. `"thread":"zio-fiber-36"`) instead of the Thread name where the fiber actually runs.
This is quite nice because you get more information about how things run in parallel or sequentially. 

## Profiling
In my team, we use the DataDog APM for instrumenting, tracing and profiling applications our applications.
So I wanted to use the same thing for my new ZIO application.
Fortunately, things seemed to work quite seamlessly here, with the help of the [zio-telemetry OpenTracing](https://zio.dev/zio-telemetry/opentracing) library.
DataDog APM [supports OpenTracing](https://docs.datadoghq.com/tracing/trace_collection/custom_instrumentation/java/) for custom instrumentation, for the cases where it doesn't automatically support some libraries, like STTP.

Here's what I needed to do to get custom defined spans:
* Added the following dependencies to `build.sbt`: 
```
"dev.zio" %% "zio-opentracing" % "3.0.0-RC1",
"io.opentracing" % "opentracing-util" % "0.33.0"
```
* The code goes roughly like this:
```scala
import io.opentracing.Tracer
import io.opentracing.util.GlobalTracer
import zio.telemetry.opentracing.OpenTracing

val defaultTracerLayer: ULayer[Tracer] = ZLayer.fromZIO(ZIO.succeed(GlobalTracer.get))
val program: ZIO[Any, Nothing, Any] = ???

override def run: URIO[Any, ExitCode] =
  program.provide(
    OpenTracing.live(),
    defaultTracerLayer
  )

def program: ZIO[OpenTracing, Nothing, Any] =
  ZIO.serviceWithZIO[OpenTracing] { tracing =>
    import tracing.aspects.*
    (for {
      httpResponse <- makeSomeHttpRequest @@ span("my_http_request")
      responseCode = httpResponse.code
      customerId = httpResponse.customerId
      _ <- saveToDb(customerId) @@ span("save_to_db") @@ tag("customer_id", customerId) @@ tag("response_code", responseCode)
    } yield response) @@ root("process_customer_request")
  }

```
The `import tracing.aspects.*` seems a bit unusual there, but it's not a big deal. Perhaps the API can be improved a bit in this area. 

## The ZIO ecosystem

I must say that I am totally impressed with the [sprawling ZIO ecosystem](https://zio.dev/ecosystem/) given how young the framework itself is.
For my use case I found a library for pretty much all aspects of my application.

Aside from [zio-kinesis](https://github.com/svroonland/zio-kinesis), [sttp](https://sttp.softwaremill.com/) and [zio-s3](https://zio.dev/zio-s3/), which I have mentioned above, I'd also bring up:
* [zio-aws](https://zio.dev/zio-aws/), which is a low-level interface, but a must-have if you need to interact with AWS services
* [zio-json](https://zio.dev/zio-json/) is also very easy to use if you work with JSON (and who doesn't these days? :) )
* the [ZIO Intellij plugin](https://plugins.jetbrains.com/plugin/13820-zio-for-intellij) can be quite helpful and it is a must if you are going to run tests from within the IDE

I must also mention the fact that before I found [sttp](https://sttp.softwaremill.com/), I actually tried to use [zio-http](https://zio.dev/ecosystem/community/zio-http), but due to [this issue](https://github.com/svroonland/zio-kinesis/issues/797) I was not able to get it to work together with the latest version of zio-kinesis, so I gave up on it.

This actually brings me to an important point: there are sometimes very annoying incompatibilities between libraries that are built for different minor versions of ZIO (e.g. 2.0.2 vs 2.0.5).
And if you're not careful about these dependencies you will get some very cryptic error messages. For example:
`java.lang.Error: Defect in zio.ZEnvironment: Set(SttpBackend[=λ %A → ZIO[-Any,+Throwable,+A],+{package::WebSockets & ZioStreams}]) statically known to be contained within the environment are missing`.
So this is another area that requires improvements.

Overall though, I believe that the already existing diversity of supporting libraries is a testimony to how strong the foundations are, so kudos to the ZIO devs!

## Final thoughts and looking ahead
My journey with ZIO so far has been quite exciting. Admittedly, I haven't run my application in anger in production just yet and there is no significant complexity involved in my use case.

But I believe that I have already touched on most of the usual aspects of the development lifecycle (writing the code, testing, deployment, debugging, profiling, etc) and ZIO had an idiomatic approach to most of these aspects.

I have been quite lucky that I was able to use it in a small greenfield application, and I think that this is the best way to make a start with it.
So I'm looking forward to new use cases that can benefit from the elegance and safety of effect systems and ZIO in particular.

One new ZIO feature that I'd be interested to try is [zio-direct](https://github.com/zio/zio-direct), which promises to make the program structure easier to understand, especially for developers that don't have much experience with functional programming, chained `flatMaps` and the Scala "for comprehension".

I'm also looking forward to better runtime compatibility between minor releases, because at the moment it can cause some puzzling situations, with seemingly no way out.

I hope that this blog was useful for getting people interested into functional programming, effect systems and of course, ZIO, and please reach out to me on [Twitter](https://twitter.com/dimitriu_bogdan) if you'd like to know more about my experience with it.

Have fun!

Bogdan