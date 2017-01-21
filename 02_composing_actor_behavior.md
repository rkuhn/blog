Roland Kuhn, Jan 21, 2017

#Composing Actor Behavior

In my [previous post](01_my_journey_towards_understanding_distribution.md) I took you on a journey towards a better understanding of distributed computing. The journey ended—somewhat dissatisfactorily—with the insight that due to its inherent properties the Actor model is very well suited for describing distributed systems, but reasoning about it is more difficult than for π-calculus. As @nestmann has [pointed out in the comments](https://github.com/rkuhn/blog/pull/1#issuecomment-266908620) there is a connection between these aspects, π-calculus is not in the same “class of distributability” as the Actor model, Join-calculus, or the localized π-calculus.

Since I do not feel competent to contribute to the theoretical discourse on these topics, I have worked on a concrete implementation of the sub-actor concept mentioned at the end of the previous post. The result is an expressive toolbox for Actor behaviors that should combine well with an adaptation of @alcestes’ [lchannels](https://github.com/alcestes/lchannels) library: generating message classes that represent the succession of protocol steps in a session type.

##The basic abstraction

Since the term “sub-actor” is a bit awkward—also typographically—I will call the basic building block a _process_. Every process describes a sequence of operations (e.g. awaiting a message, querying the environment, etc.) and is hosted by an Actor. These Actors are not directly visible in the programming abstraction, the implementation comes from the library and contains an interpreter for multiple concurrent processes. A very simple first program illustrates the basic setup:

~~~scala
import akka.typed._
import akka.typed.ScalaProcess._

object FirstStep extends App {

  val main =
    OpDSL[Nothing] { implicit opDSL =>
      for {
        self  <- opProcessSelf
        actor <- opActorSelf
      } yield {
        println(s"Hello World!")
        println(s"My process reference is $self,")
        println(s"and I live in the actor $actor.")
      }
    }

  ActorSystem("first-step", main.toBehavior)
}
~~~

This code is taken from [the process demo project](https://github.com/rkuhn/akka-typed-process-demo/blob/58731693461899813e414f0e9d9a09fe580e62c6/src/main/scala/com/rolandkuhn/process_demo/FirstStep.scala) which contains a modified version of the the Akka Typed artifacts that contains the process DSL. Running this program with sbt looks like the following:

~~~
rk:akka-typed-process-demo rkuhn$ sbt run
[... snip ...]
[warn] Multiple main classes detected.  Run 'show discoveredMainClasses' to see the list

Multiple main classes detected, select one to run:

 [1] com.rolandkuhn.process_demo.AskPattern
 [2] com.rolandkuhn.process_demo.FirstStep
 [3] com.rolandkuhn.process_demo.HelloWorld
 [4] com.rolandkuhn.process_demo.Parallelism

Enter number: 2

[info] Running com.rolandkuhn.process_demo.FirstStep 
Hello World!
My process reference is Actor[typed://first-step/user/$!a#0],
and I live in the actor Actor[typed://first-step/user#0].
~~~

We see the result of the `println` statements in the `yield` block of the main process. The friendly greeting is nice, but more interesting are the two identities that are printed on the final lines: `opProcessSelf` is a handle for the input channel of the current process while `opActorSelf` is a handle for the main input channel of the Actor that hosts this process. As you can see, these two are in a parent–child relationship according to Akka `ActorRef` rules: the process has its own reference named `$!a` under the namespace of the parent. That parent in the example is the guardian actor for the ActorSystem, always named `/user`. The `#0` part to the right should disambiguate different incarnations of an Actor, but that is not yet fully implemented.

Processes are always constructed within a lexical scope defined by an `OpDSL` instance. This is necessary because a process has exactly one ingress point for messages and the type of these messages is fixed by the OpDSL type parameter; this parameter was `Nothing` in the example above because that process does not receive any messages. We demonstrate the use of different process context by building a slightly less trivial “hello world” example. First, we construct a server process that listens for `WhatIsYourName` messages and responds with the string `"Hello"`.

~~~scala
case class WhatIsYourName(replyTo: ActorRef[String])

val sayHello =
  OpDSL.loopInf[WhatIsYourName] { implicit opDSL =>
    for {
      request <- opRead
    } request.replyTo ! "Hello"
  }
~~~

The `OpDSL.loopInf` constructor will construct and run its argument in an indefinite sequence, in this case alternating between awaiting a request with `opRead` and sending a response to the `ActorRef` that was contained in the request. A slightly different formulation might use pattern matching within the for-comprehension, as shown in the definition of the second process we need for our “hello world”.

~~~scala
val theWorld =
  OpDSL.loopInf[WhatIsYourName] { implicit opDSL =>
    for {
      WhatIsYourName(replyTo) <- opRead
    } replyTo ! "World"
  }
~~~

The clou of the process abstraction is that the `sayHello` and `theWorld` values can be used as building blocks for larger behaviors by composing them. The main process we are building here will first obtain its own handle—prepared to receive strings from the two processes defined above—and then fork or spawn the two helpers. In each case a `WhatIsYourName` request is sent and the response is read. Finally, both responses are combined in a single output statement.

~~~scala
val main =
  OpDSL[String] { implicit opDSL =>
    for {
      self <- opProcessSelf

      hello    <- opFork(sayHello.named("hello"))
      _         = hello.ref ! WhatIsYourName(self)
      greeting <- opRead

      world <- opSpawn(theWorld.named("world"))
      _      = world ! MainCmd(WhatIsYourName(self))
      name  <- opRead

    } yield {
      println(s"$greeting $name!")
      hello.cancel()
    }
  }
~~~

Running [this process](https://github.com/rkuhn/akka-typed-process-demo/blob/58731693461899813e414f0e9d9a09fe580e62c6/src/main/scala/com/rolandkuhn/process_demo/HelloWorld.scala) will print the expected `Hello World!` to the console. The greeting is constructed from the inputs of two processes running concurrently to the main process:

* the one named “hello” is forked, which means that it is hosted by the same Actor as the main process; therefore this process must also be canceled at the end, otherwise its infinite loop would keep waiting for messages indefinitely
* the one named “world” is spawned as the main process of a real child actor, meaning that it can also be executed in parallel (given enough CPU cores) instead of sequentially sharing the CPU time allotted to the guardian actor with its main process; since the child Actor will need to also receive some internal management messages, we need to wrap the message destined for the main process in a `MainCmd` envelope.

So far we have discussed the basic operations of running process steps sequentially, concurrently, or in parallel. With one addition we will be able to formulate our own abstractions on top of this foundation.

##Sequential composition

The main concern with compositionality of typed processes has been discussed in the previous post under [the path towards compositionality](https://github.com/rkuhn/blog/blob/master/01_my_journey_towards_understanding_distribution.md#the-path-towards-compositionality): if a process is characterized by the type of messages it can receive, then doing two different activities in sequence implies having first one type and then another. Typestate (see [the paper from 1986](http://dl.acm.org/citation.cfm?id=10693) by R.E. Strom and S. Yemini) may be able to model such a transition, but no mainstream programming language includes this capability today. We work around this restriction by introducing `opCall`. This operation spawns the given process within the same host Actor, but instead of running concurrently the caller is suspended until the called process has run to completion and returned its computed value. The called process is free to use a differently typed `ActorRef` for itself, allowing interactions that do not affect the caller in any way—the called process is encapsulated just as a function call that only returns its final value.

Since it is so common to send a request and expect back a response, we demonstrate this feature by creating a new building block, the “ask” operation.

~~~scala
def opAsk[T, U](target: ActorRef[T], msg: ActorRef[U] => T): Operation[U, U] =
  OpDSL[U] { implicit opDSL =>
    for {
      self <- opProcessSelf
      _ = target ! msg(self)
    } yield opRead
  }
~~~

This operation assumes to be run in a process whose message type is `U` and it will produce a result of that same type. To do that, it obtains its own process handle, uses the given function to insert the handle into the request message, sends the request to the target `ActorRef`, and returns the result of awaiting the response message. Using this new operation our “hello world” example becomes quite a bit shorter.

~~~scala
val mainForName = WhatIsYourName andThen MainCmd.apply _

val main =
  OpDSL[Nothing] { implicit opDSL =>
    for {
      hello    <- opSpawn(sayName.named("hello"))
      greeting <- opCall(opAsk(hello, mainForName).named("getGreeting"))
      world    <- opFork(sayName.named("world"))
      name     <- opCall(opAsk(world.ref, WhatIsYourName).named("getName"))
    } yield {
      println(s"$greeting $name!")
      world.cancel()
    }
  }
~~~

Sending the first request to the spawned process in the child Actor is a little more complicated since the request needs to be wrapped additionally inside a `MainCmd` envelope—the Actor uses a set of internal messages for triggering the interpreter and this envelope allows messages to be forwarded to the main process after unwrapping them. Another formulation for that function value would be `(replyTo: ActorRef[String]) => MainCmd(WhatIsYourName(replyTo))`.

Running [this process](https://github.com/rkuhn/akka-typed-process-demo/blob/4dc1d1c3a197ebbf51c45fd189119baa2f2d1fcb/src/main/scala/com/rolandkuhn/process_demo/AskPattern.scala) does not print the expected greeting, though: instead it prints `hello user!`. The reason lies in the implementation of `sayName` that demonstrates the difference between spawning and forking—hence the “hello” comes from the child Actor’s name while the “user” comes from the guardian actor that hosts the forked process.

~~~scala
val sayName =
  OpDSL[WhatIsYourName] { implicit opDSL =>
    for {
      self <- opActorSelf
    } yield OpDSL.loopInf { _ =>
      for {
        request <- opRead
      } request.replyTo ! self.path.name
    }
  }
~~~

The trick here is that the `ActorRef` contains the main process’ name as the last segment of its `ActorPath`. This snippet also shows how to start an infinite server process after doing some initialization.

##Adding parallelism

The basic functionality for allowing parallel execution is already present in the form of `opSpawn` but that operator ignores the spawned process’ result value. In order to speed up lengthy computations we should be able to build an abstraction that can turn a list of process descriptions into a list of results, running computations in parallel. This is an interesting example because it corroborates the completeness of the provided process algebra.

~~~scala
def opParallel[T](processes: Process[_, T]*)(implicit opDSL: OpDSL): Operation[opDSL.Self, List[T]] = {

  def forkAll(self: ActorRef[(T, Int)], index: Int, procs: List[Process[_, T]])(implicit opDSL: OpDSL): Operation[opDSL.Self, Unit] =
    procs match {
      case x :: xs =>
        opFork(x.foreach(t => self ! (t -> index)))
          .flatMap(s => forkAll(self, index + 1, xs))
      case Nil => opUnit(())
    }

  opCall(OpDSL[(T, Int)] { implicit opDSL =>
    for {
      self    <- opProcessSelf
      _       <- forkAll(self, 0, processes.toList)
      results <- OpDSL.loop(processes.size)(_ => opRead)
    } yield results.sortBy(_._2).map(_._1)
  }.withMailboxCapacity(processes.size))
}
~~~

Constructing this abstraction comes with a bit more boilerplate because we will be using `opCall` to run the processes that oversees the parallel execution, awaiting the list of results. This `opCall` needs to know the process context it is embedded into in order to return the right type—it does not care about its self reference and can thus provide the type of the process that is constructed in the calling scope. Scala’s implicit arguments model exactly this situation (and Dotty’s [implicit function types](https://www.scala-lang.org/blog/2016/12/07/implicit-function-types.html) will make this boilerplate go away eventually), the implicit `OpDSL` instance conveys the type information through the type member `Self`.

The implementation of parallel execution proceeds in four steps:

* first obtain a self-reference capable of receiving pairs of result value and integer
* then fork all given processes, appending a step that sends the computed value together with the sequence index of that process to the self-reference
* use a loop constructor to read the right number of responses, resulting in an unordered list
* sort the list according to the original sequence and return only the result values

Calling the process that collects results from the parallel tasks requires us to think about a detail that has not played a role so far. Every process handle is backed by a message queue, where senders enqueue and `opRead` operations dequeue items. The default queue capacity is 1 because it is very common to not have multiple messages outstanding. Exceptions would be server processes or other processes that employ non-determinism in the form of receiving from multiple senders without coordination. The process we are building will need to be able to receive a number of messages equal to the size of the list of processes it is asked to query in parallel. Configuring the mailbox capacity (using `withMailboxCapacity`) or a few other options is the difference between an `Operation` (i.e. a sequence of steps) and a `Process`. 

Using this new abstraction we can make our “hello world” go parallel:

~~~scala
val main =
  OpDSL[Nothing] { implicit opDSL =>
    for {
      hello <- opSpawn(sayName.named("hello"))
      world <- opSpawn(sayName.named("world"))
      List(greeting, name) <- opParallel(
        opAsk(hello, mainToName).withTimeout(1.second),
        opAsk(world, mainToName).withTimeout(1.second)
      )
    } yield {
      println(s"$greeting $name!")
    }
  }
~~~

Parallelism itself is enabled by spawning the processes that compute the answers in child Actors, then we use `opParallel` to ask both of them, returning the ordered list of responses. One noteworthy aspect here is that we set a timeout of 1 second for each of the ask operations—if any one of these expires the whole Actor hosting these processes will fail. This choice of linked failure has been made because allowing concurrency (of a specifically restricted kind) within an Actor is already stretching it—allowing processes to fail independently would make Actors distributed on the inside, and that way lies madness.

##Current status and future plans

The process DSL presented in this blog post is currently an open [pull request](https://github.com/akka/akka/pull/22087) towards Akka, awaiting some naming discussions (in particular on the “op” prefix for the built-in operations). I am very happy with how the encoding has worked out and am reasonably confident that a very similar process DSL will soon land in an Akka release near you. If you want to play around with the current code, please clone the [akka-typed-process-demo](https://github.com/rkuhn/akka-typed-process-demo) repository. It also contains source jars for the embedded implementation jar. **For eclipse users: please ensure that you select a 2.12 Scala installation for compilation, otherwise you will encounter binary incompatibility issues.**

An aspect that is not discussed in this post is that the proposed DSL also contains operations that embed a type-indexed set of state monads within each Actor, allowing processes to keep state, possibly share it, and in a future version also persist it—this is the foreseen integration point with Akka Persistence. You can follow the discussion [on akka-meta](https://github.com/akka/akka-meta/issues/7).

The next step will be the integration of this process DSL with automatically derived protocol message types, allowing an already useful subset of the static protocol verification that is possible with the Scribble language. While not guaranteeing perfect safety, it will make many aspects of Actor interactions accessible to rigorous compile-time checks.

#Comments

Please leave comments [on the pull request](https://github.com/rkuhn/blog/pull/1) or [on specific lines](https://github.com/rkuhn/blog/pull/1/files).
