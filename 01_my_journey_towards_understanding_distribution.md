Roland Kuhn, Oct 18–Nov 20, 2016

# My journey towards understanding distribution

When asked why I like the Actor Model I usually say “because it models distribution exactly.” This means that it expresses what it means for computation to be distributed, without fluff and without hiding essential features. The purpose of this article is to persist something I learnt recently about this statement; I hope it is useful to others as well.

_Disclaimer: someone else has probably written down all the salient points in the eighties—apologies for not taking the time to research this aspect, this is the way I prefer to learn._

## Starting point: untyped Akka Actors

It was my privilege to be part of the rewrite of Akka between version 1.3 and 2.0. During this period we made several fundamental changes to how the toolkit works and what it guarantees. The golden rule for every change was “if it cannot be guaranteed to always work under distribution, then it must not be done.” We had an intuitive understanding of what it means to be distributed, something along the lines of

* sender and receiver of a message can be on systems that are far apart (in terms of communication latency), so knowing when something has been processed is not really meaningful, because
* all communication is unreliable: messages can be lost or delayed arbitrarily, and
* processes (Actors) can fail independently from each other, whether on the same machine or on different parts of a network.

One deeply ingrained mental reflex within the Akka team was formed back then, assuming that all consensus between Actors is problematic and should be avoided wherever possible. It takes a lot of time to reach consensus, and once it has been reached parts of the system can already be far ahead of what has been decided, potentially invalidating the result unless processes are designed to take propagation time into account.

What can we offer in terms of user API and features under these severe constraints? The Actor Model defines three features and implies a fourth (but we’ll come back to the implication later):

* sending messages
* changing behavior between messages (i.e. sequential processing)
* creating more Actors

Akka implements all these, although it deviates in terms of message delivery guarantees: instead of building in _reliable delivery_ we made that optional, arguing that we should leave it up to the user to decide which level of reliability is required—e.g. we think that without (redundant) persistence it would not really be reliable because a power outage can break the guarantee, but requiring a persistent storage just to run a few local Actors is definitely very heavy. The important constraint here is that user API have a 1:1 mapping to operational semantics, i.e. an ActorRef _must always_ behave in the same fashion, it cannot be made more or less reliable by way of configuration because that would not be obvious when looking at the expression `ref ! msg`.

Additional features offered by Akka are

* mandatory parental supervision (including bounding the child Actor’s lifetime by its parent’s)
* lifecycle monitoring a.k.a. _DeathWatch_

Going beyond what the Actor Model provides comes at a cost, it requires a certain level of coherency within the cluster of nodes that hosts the actors. Only with consensus on when to assume that a node has fatally failed can we guarantee that these features keep working with the same semantics under all conditions. Reading the mailing list makes it clear that this price is not to be underestimated. One recurring question is why nodes get kicked out the cluster and why they cannot come back later (explanation: once a node has been declared dead all supervision and death watch notification have been fired, so coming back from the dead would lead to Actors that misbehave like zombies).

## Vantage point: Akka Typed

The untyped nature of Actor interactions irked me from the very beginning. Sending a message is mediated by the `!` operator, essentially a function from Any to Unit—completely unconstrained and without feedback. The lack of feedback is a concession to modeling a distributed system, since all communication has a high price. The lack of typing constraints seems accidental, though, and we see the same problem again when looking at how an Actor is defined: it is a _partial_ function from Any to Unit, making every Actor a black box that may or may not do anything when you send a message to it. This gives Actors a lot of freedom, but it also makes static reasoning rather difficult—it feels a bit like injecting a bubble of JavaScript into the type-safe Scala world.

Since version 2.4 Akka ships with [Akka Typed](http://doc.akka.io/docs/akka/current/scala/typed.html), the third incarnation of the wish to improve the situation by restricting the type of messages accepted by an Actor, allowing the compiler to reject clearly incorrect programs. In essence, an Actor’s definition is now given by a _total_ function from some input message type to the next behavior (restricted to be of the same type). Correspondingly, it becomes possible and prudent to parameterize the Actor reference by the same message type, rejecting invalid inputs.

~~~scala
// pseudo-Scala syntax with dotty extensions
type ActorRef[-T] = T => Unit
type Behavior[T]  = (T | Signal) => Behavior[T] // of course this is cyclic, so it needs a trait
~~~

This change inspired many cleanups in internals and auxiliary features, but it also opened up possibilities of expressing more than just static Actor types: by including appropriately typed ActorRefs in messages the types occurring during a conversation between Actors can evolve as time passes, going through different protocol steps.

~~~scala
case class Authenticate(token: Token, replyTo: ActorRef[AuthResponse])

sealed trait AuthResponse
case class AuthSuccess(session: ActorRef[SessionCommand]) extends AuthResponse
case class AuthFailure(reason: String) extends AuthResponse
~~~

Modeling a protocol like this and exposing only an `ActorRef[Authenticate]` to clients does not only inhibit them from sending the entirely wrong message type, it also expresses the dependency of the session availability upon successful authentication—without having an `ActorRef[SessionCommand]` the compiler will not accept the sending of such messages.

## Tangent: Protocols

The previous example works by using different message types for each protocol step, which can get unwieldy after a while. Another caveat is that the number of messages sent at each step is not statically verified, the client could send multiple times, perhaps even going back to a previous protocol step by retaining that step’s ActorRef. And of course this scheme would break down as soon as protocols involve cycles that reuse the same type at different times.

What is necessary to get a grip on these problems is to describe multi-step protocols in terms of their shape. One promising approach is called [_Session Types_](http://groups.inf.ed.ac.uk/abcd/), but not all questions have been answered here. For example it remains problematic to express the linearity of the process (i.e. the inability to go back in time and use previously invalidated knowledge) within programming languages such that the result is comprehensible to mere humans. One approximation is presented by Alceste Scalas’ [lchannels library](http://alcestes.github.io/lchannels/instructions.html).

## The path towards compositionality

The formulation of Actors consciously focuses on a single entry point for messages. This entry point can be rebound in untyped Akka using `context.become(...)`, or it is the result of each message processing in Erlang or Akka Typed. The consequence for composing an Actor from different behavior pieces (i.e. making it do different things with different interlocutors) is that all messages come in via this one ingress point and must be demultiplexed to reach their correct destination within the internal logic. This is mildly annoying in untyped Actors, but it can be downright frustrating for Akka Typed, requiring casts to formulate a behavior that accepts both `Authenticate` and `SessionCommand` but only exposes the former to the public, for example. Strongly typed logic requires principled means of composition, this seems to be universally true whether composing pure functions or distributed computations.

Alex Prokopec’s [presentation at ScalaDays 2016](https://www.youtube.com/watch?v=7lulYWWD4Qo&index=11&list=PLLMLOC3WM2r7kLKJPHKnyJgdiBGWaKlJf) in Berlin was a transformative experience for me in that it showed a way out of this dilemma. It is the nature of the Actor Model to designate different Actor identities (their references) for different purposes. We can use this to build a bigger entity that can talk a different protocol with each of its interlocutors. Creating independent Actors has the downside of losing internal consistency—_Actors are isolated islands of sanity in a sea of distributed chaos_—so the trick is to virtualize the Actor and create multiple ingress points, each with their own identity. Where Alex uses stream processing semantics à la RxJava I was immediately attracted by the idea of using π-calculus for the internal composition of these compound Actors.

The first version of a [possible session DSL](https://github.com/akka/akka/pull/21212/files#diff-b8845b35e24817f4231a4f11d7a86865R203) on top of Akka Typed was quickly created based on a monadic description of the sequential and concurrent composition of primitive actions and calculations.

~~~scala
val server = toBehavior(for {
  backend ← initialize
  server ← register(backend)
} yield run(server, backend))

private def initialize: Process[ActorRef[BackendCommand]] = {
  val getBackend = channel[Receptionist.Listing[BackendCommand]](1)
  actorContext.system.receptionist ! Receptionist.Find(BackendKey)(getBackend.ref)
  for (listing ← readAndSeal(getBackend)) yield {
	if (listing.addresses.isEmpty) timer((), 1.second).map(_ ⇒ initialize)
	else unit(listing.addresses.head)
  }
}

...
~~~

The core abstraction is a Process that eventually computes a value of a given type. `flatMap` or `map` is used for sequential composition and there is a `fork(process)` action that is used to create concurrent threads of execution. The complete resulting Process is evaluated within a single Actor (the `toBehavior` function wraps it in a suitable interpreter), reacting to inputs as they become available and asked for: the `readAndSeal` operation suspends the process until a message is available on the `getBackend` channel created as part of the `initialize` Process.

The primitives offered by this library sketch match the actions and composition features of π-calculus:

* channel creation
* sending
* receiving
* sequence
* choice
* parallelization

Things started looking really good, a world of nicely composable and reusable behavior pieces began building itself in my imagination.

## The trough (yes, of disillusionment)

My dream universe crumbled when I asked myself what would happen whenever a crucial message—one that unlocks the next protocol step for another piece of Process—were to not arrive, for whatever reason. Making delivery reliable does not fix the problem that other Actors can fail independently, and with the writing end of a channel being an ActorRef it would be entirely reasonable to depend on remote systems to make progress—location transparency is a very strong semantic promise. The fix would of course be to place an upper bound on the waiting time for a receive operation, but then a local Process would need to fail. Assuming that local processes would coordinate also via channels, independent failure of processes would imply that channels can be orphaned—this is a resource safety problem that would require (distributed) GC to be solved. (Channels can also be orphaned due to programmer error, of course, so avoiding a fatal resource leak seems prudent in any case.)

Another thing I realized was that in a π-calculus expression on paper we can spot and eliminate dead processes that cannot possibly make progress anymore because they are waiting to send or receive along a channel that is not known to any other process. This kind of dead Process elimination is not practically possible in an implementation based on opaque Scala closures.

But the most severe difficulty is that the defining feature of π-calculus, namely the ability to send channels around, proves extremely challenging to implement in practice. The sending side is trivially solved by exposing it as an ActorRef. For the receiving side it would be necessary to enable message sends to be delivered to a set of readers whose only defining characteristic is that they are currently in possession of a reference to the channel and ready to receive, and it would need to be guaranteed that only exactly one of the readers actually gets the message.

This kind of global coordination is what we eschew in Akka, at least for the basic primitives, since it is so expensive. The premise is that the basic solution should be scalable without practical limits, in principle infinitely. We strive to get as close to this ideal as possible. Not everybody needs infinite scalability and there are valid cases where a sequentially consistent database is the right solution, but that does not keep us from pushing the envelope.

## Climbing up the ridge

It would be possible to “fix” channel usage by not allowing the receiving end to be serializable (meaning that it cannot be sent across the network) and throwing an exception if a receive operation is attempted from the wrong Actor’s context. This would be ugly, not only because it deviates fundamentally from π-calculus but also because it is bad practice to offer non-total functions as user API whose function depends on circumstances that are invisible in the code or types.

Selecting which channel to read from is convenient for us humans, it matches how we interact as well: we walk around and talk to different people in a sequence we choose in order to reach our goals. Actors are forced to deal with whatever message comes in next, which in real life would correspond to getting distracted all the time. Unfortunately the freedom to select the channel is precisely what proved problematic above, so we now turn our attention to alternative approaches.

One alternative was presented by Alex with his Reactors. The API for a channel allows transformations and other reactions to be attached in stream-processing fashion. This is already better, but it still allows a channel reference to be passed to another Reactor and wreak havoc by receiving from the wrong context.

**This made me realize the advantage of having the receive operation as an _implicit_ property of the API, something that is not freely accessible by user code. This is precisely how the Actor Model avoids this pitfall, it defines three kinds of actions that can be taken _in response to a message_, but it does not permit the actor to actively ask for a message to come in.**

So the other alternative is … Actors. Each channel is created from a behavior that describes how it will react to incoming messages, including the ability to change behavior betwixt them. This means that continuing a conversation with another Actor is done by creating a channel with the continuation and sending that back to the interlocutor. This is precisely how Carl Hewitt and Gul Agha have envisioned and advertised it from the very beginning, albeit without the inherent concurrency.

## Vantage point: sub-Actors

The improved model could be described as an implicit packaging of channel creation with Process creation and the removal of the argument to `read` operations—reading only has access to the single input channel of a Process. Sequential composition could choose to reuse the same channel, parallel composition would need to communicate results via previously established continuation channels if appropriate.

The advantage over bare Akka Typed would be the removal of boilerplate code to create and install continuation behaviors, plus the reuse of a single Actor as a scheduling unit makes this fine-grained usage feasible without incurring forbiddingly high overhead in terms of Actor creation and inter-thread messaging. It would mean that step-wise definition of behavioral processes within Actors can conveniently be written down, reused, and composed.

## Have we reached the top?

In the beginning I declared that I have learnt something about the statement “Actors model distribution exactly.” The learning consists in realizing just how exactly this model fits to the problem: there does not seem to be any room between the features of the Actor Model and the semantics of distributed systems. In particular, while for example the Wikipedia page on process calculi states that π-calculus and the Actor Model can be seen as duals, I do no longer think that this is true—to me it seems that π-calculus offers too rich a feature set in order to allow infinitely scalable implementations. This aspect triggered some more research, in particular about the [expressiveness of asynchronous π-calculus](https://arxiv.org/abs/cs/9809008) (thanks to Chris Meiklejohn for the pointer!); I also encountered a very helpful [FAQ about π-calculus](https://cs.cmu.edu/~wing/publications/Wing02a.pdf) that explains its intended use. My conclusion is that the tools of π-calculus are interesting for formal description and verification of protocols—where not all possible expressiveness of the calculus is actually used—and that it is not suitable as direct inspiration for end-user API.

On the other hand I did not find a true formalization of the Actor Model into a calculus in the sense that it becomes mathematically tractable in a similar fashion to other calculi, with equivalence and congruence relations and all the nice theorems that follow. It might well be that instead of such a formalization we need to derive constraints on an Actor’s behavior from external protocol descriptions, lifting them to the source level by using code generation (e.g. by encoding the whole session as Alceste has done, or by generating suitably linked message classes, depending on how much safety can be achieved with a reasonable end-user API). Or we need to extract the Actor’s actions in an abstract behavior tree that can be represented as π-calculus processes, to be analyzed externally. My main conceptual difficulty is that the primitive action of sending a message unreliably and with arbitrary delay maps to a non-trivial π process, leading to combinatorial explosions in terms of reduction possibilities; but I am not (yet?) ready to abandon the goal of having the most basic construct be efficiently implementable even for infinitely scalable systems.

My current takeaway is that we should first try out composable sub-Actors and any other such model that others can think up. And then we take it further from there.

_For the second part please see [Composing Actor Behavior](02_composing_actor_behavior.md)_

# Comments

Please leave comments [on the pull request](https://github.com/rkuhn/blog/pull/1) or [on specific lines](https://github.com/rkuhn/blog/pull/1/files).

---
_Writing space sponsored by [BAYMARKETS](http://baymarkets.com/) in Stockholm (tack så mycket!)_
