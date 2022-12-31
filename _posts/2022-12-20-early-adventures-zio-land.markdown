---
layout: post
title:  "Early adventures into ZIO land"
date:   2022-12-20 01:01:53 +0000
categories: scala zio
---
# Early adventures into ZIO land

I've been writing Scala applications for more than a decade now, but I wouldn't call myself anywhere close to an expert in it.
I really do enjoy coding in Scala whenever I have a chance, although this doesn't happen very often these days, given that I spend a lot of time in the world of cloud infrastructure and architecture (more on that some other time, hopefully).

I also generally enjoy taking a functional approach to writing code, but I've never been a purist, I was looking for elegant but quick solutions.
I had heard about Cats Effects and ZIO for a couple of years, but I didn't get to look properly into what the buzz was about, until I saw that Daniel Ciocîrlan of [Rock the JVM](https://rockthejvm.com/) released a [course on ZIO](https://rockthejvm.com/p/zio).
Daniel's Scala courses, videos and articles are just excellent. I thoroughly recommend them to anyone who wants to learn about Scala in a quick and pragmatic way.
Better yet, this is a course about ZIO 2, which was released in June 2022, rather than about ZIO 1. And ZIO 2 I understand is a fairly big improvement over ZIO 1. Check out John De Goes' post https://degoes.net/articles/zio-2.0 for more details as to why. 
So, I subscribed to his ZIO course. And it was totally worth it! I got to learn about most of the ZIO features in a fun, quick and practical way.
And it really was a bit of a revelation for me, pretty much in the same way the Spring framework was a revelation when it came to save the Java world from the madness of EJBs :).

### What is ZIO 2 and why you should pay attention to it?

ZIO is a framework centered around the idea of "[effect systems](https://en.wikipedia.org/wiki/Effect_system)" and the [IO monad](https://en.wikipedia.org/wiki/Monad_(functional_programming)#IO_monad_(Haskell)).
Designing an application based on this idea confers some pretty nice advantages that have become more obvious to me as I progressed through the course, but it does fundamentally change the way one should organise and think about their code.
In some way, to me, it seems that programming with an effects framework like ZIO is like building a complex pipeline (your actual program) by interconnecting small specialised pipes (the effects) that may have an input (another effect's output) and an output (the effect's return type **A**).
These pipes can [fail in expected ways](https://zio.dev/reference/error-management/types/failures/) (the effect's error type **E**) or in unexpected ways ([Defects](https://zio.dev/reference/error-management/types/defects)) and the failures should be explicitly modelled and handled in the pipeline design.
The pipes (effects) can also optionally have specific features that they require in order to function, so they must be provided by the program. These constitute the effect's environment (the type **R**). You can think of these as smart electronic chips that are required by the individual pipe to actually do what it is supposed to.
This pipe architecture doesn't do anything until data starts flowing through it. That is until we "run" it with the "effects engine".

So there you have it. In a nutshell this is what a `ZIO[R, E, A]` is about. Of course, I'm over-simplifying things, there is a lot more depth packed into this apparently basic construct.

But this blog is not meant to be a ZIO tutorial, so let's talk about some practical experience with it :)

### The application

Fairly recently I got to work on a relatively simple component, that was meant to handle a message processing pipeline that I could sum up this way:
_Consume a JSON message from AWS Kinesis ~~> Parse the JSON and extract an S3 location ~~> Fetch the S3 object from that location ~~> POST the object to an HTTP server and read the result_
Nothing fancy here.

The component does need to handle a fairly high and sustained rate of processed messages per second (in the thousands).
There are some implicit constraints:
* there shouldn't be more Kinesis application consumers (effectively JVM processed) than there are Shards
* the files that are sent via the POST request can be of various sizes, sometimes in megabytes, so I decided it was wise to design the application to stream directly from S3 to the HTTP server (not buffering the files in memory)
* because of the number of parallel requests per second, I decided to go for a non-blocking I/O approach
* the latency doesn't matter, but the throughput needs to be high

Note: I developed this component with Scala 3.2.1, JDK 17/19 and ZIO 2.0.2 and it is run on a EKS cluster in AWS.

### Error management

The first ZIO feature I got hooked on was [error management](https://zio.dev/reference/error-management/).
Previously I would have used constructs like Try, Either, Option to handle errors.
These constructs are great and they make functional composition of errors much more elegant than using try/catch blocks (which are actually not composable at all). But ZIO's error handling mechanisms goes to the next level. They provide a domain-driven, unified, functional, explicit, type-safe way of dealing with failures. Handling errors with ZIO for my application felt like a breeze and the code felt easy to read and crucially, easier to test. Just make sure to follow the "[Best practices](https://zio.dev/reference/error-management/best-practices/algebraic-data-types)" section from the ZIO reference guide. Based on this, I think that one of the most important early decisions you need to take is to **_model your domain errors_**.

Intermezzo: I mention things that "compose well" and "functional composition". What does it mean and why is it a big deal? Coming back to the "pipes" analogy - we want to construct a program from smaller pipes, each having a specific role and we want to be able to join them together neatly so that the information can flow through this architecture correctly. These pipes are reusable (because the effects just describe code, they don't actually run it) and they can be reused and composed with other pipes in whatever ways we see fit. Mathematically functions compose well and this is the model that the effects take as well. And this is because Effect instances (as in ZIO[R, E, A]), just like functions, _do not have side effects_. It's only the code they wrap that may have side effects (like reading from a socket or modifying a file, or even printing something on the console).

### Dependency injection
Most JVM developers would be familiar with the concept of "dependency injection" which was made popular by the Spring framework. The main idea is that when you assemble the application (at the top level), you loosely couple its components, so that you can easily swap different implementations for various situations. This is also called "_inversion of control_".

ZIO also provides its own ability to wire together an application, but it does so "natively", using the `ZLayer[-RIn, +E, +ROut]`, which has many similarities to `ZIO[R, E, A]`.

I must admit that in the beginning I wasn't quite sure about using this feature in my application, primarily because I didn't feel the need to use an IoC framework for a very long time. However, with most of the libraries in the ZIO ecosystem there is no getting away from understanding and using `ZLayers` (and `ZEnvironment`). And fairly quickly I actually got to like it. I think that the feature that contributed mostly to that was [Automatic layer construction](https://zio.dev/reference/di/automatic-layer-construction/). In terms of what it achieves it is similar to the [Spring Autowired](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/annotation/Autowired.html) annotation, but I find the ZIO approach more elegant because the developer experience is better - there is no metaprogramming involved and everything is statically type-checked. You even get really nice compiler messages when you forget to provide some required layers.

Manual compositions seems to be nice as well, although I haven't used it much so far.

It takes a bit of getting used to, especially understanding sometimes unexpected environment requirements, but the compiler always points you into the right direction, because these layers always compose in a functionally correct way.

Here are some "gotchas" that tripped me for a while until I understood how things should work:

##### Using [ZIO.provideSomeLayer](https://javadoc.io/static/dev.zio/zio_3/2.0.5/zio/ZIO.html#provideSomeLayer-fffff1e1) or [Spec.provideSomeLayer](https://javadoc.io/doc/dev.zio/zio-test_3/latest/zio/test/Spec.html#provideSomeLayer-5d8)
Sometimes you don't want to provide all your environment dependency "at the end of the world" (i.e. at the root of your application) and you need to split your environment so that some of it is provided in one place and the rest of it in another place. I couldn't find many examples of how this should be used, but as usually you can find the ultimate information in the API docs.
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



Talk about:
* error handling: before (Try), after (ZIO[R, E, A])
* the ZIO course (Daniel Ciocarlan)
* functional composition 
* resource management (`ZIO.acquireWith()`)
* dependency injection
  * orderly shutdown
* streaming
* debugging - ".debug"; sometimes stacktraces don't make sense, but this seems to improve with newer ZIO releases (e.g. 2.0.2 vs. 2.0.5)
* testing
* pitfalls
  * all effects have to compose eventually into a single effect, runnable from `run`
  * passing the Environment to a ZIO/ZStream parameter in a method call - ZEnvironment - is this the right solution?
  * time in tests - not particularly obvious what the default behaviour is. The `@@ TestAspect.withLiveClock` annotation
  * meaningful stack traces (for the most part)
* ecosystem
  * zio-aws; special mention for zio-kinesis; mention the issue with kinesis-stream & zio-http
  * sttp support for zio
  * a diverse ecosystem (https://zio.dev/ecosystem/) seems to have appeared out of nowhere - I have a feeling it's because of the strong foundations. Admittedly, some integrations listed there seem very raw/basic.
  * Application performance measuring and instrumentation: ZIO + Opentracing + DataDog
* IntelliJ support: the ZIO plugin seems to suggest good refactoring + it allows testing integration - I had a very quick reply from one of the project maintainers when the early access release for ZIO 2 bumped into an error
* For me ZIO has been quite a revelation and it seems to be like the next big thing in the same way that Spring was for the Java world
* Caveats:
  * I haven't run it in production yet (to come soon in the new year)
  * I haven't done any profiling yet. I will need to figure out how to do it with the DataDog APM
