# Actors are a low-level tool

A few days ago Sergey Bykov published an article on [why he doesn’t use the term Actor anymore](https://docs.temporal.io/blog/sergey-the-curse-of-the-a-word/).
I’ve known Sergey for a few years now, we met at several conferences, were in program committees together and we’re both on the Reactive Foundation’s advisory council.
But funny enough, our first contact was precisely one of those discussions he mentions, leading to [an in-depth comparison between Akka and Orleans](https://github.com/akka/akka-meta/blob/master/ComparisonWithOrleans.md).

Since May 2015 my own understanding of distributed computing has evolved as well.
The first step was to recognise just [how precisely the Actor Model characterises distributed systems](https://github.com/rkuhn/blog/blob/master/01_my_journey_towards_understanding_distribution.md).
But after that I started building programming tools for automating high-level workflows on the factory shop floor, and I realised that Actors by themselves are not all that useful.
They are too low-level.

## Actors are a concurrency and distribution _primitive_

Implementations of the Actor Model offer an API that is rather small and seemingly simple: a message receiver is run in a loop and there is a handle for sending messages to it.
Since this API is larger than — say — that of a mutex, we are tricked into believing that an Actor is a higher-level construct.
This impression is corroborated by the fact that Actor runtimes employ mutexes or atomic variables under the hood.

But the impression is still incorrect.
An Actor is a programming primitive quite like a mutex or a promise/future, but it has two parts that are easily conflated.
The first part is the description of the message processing loop, which often is used as a concurrency control structure (processes “one message at a time”).
The second part is the Actor reference, the handle that allows sending messages.
This is the part that makes an Actor useful in a distributed setting, since Actor references can usually be sent across the network.

As I argue in the article linked above, the resulting package allows exactly the expression of distributed programs.
A distributed system is built from a group of Actors, each one being one primitive building block.
Designing and implementing Actors therefore requires a comprehensive understanding of distributed systems, the API forces the programmer to take a corresponding viewpoint.

This is really nice and powerful if you want to write a library that solves some problem using a distributed system: you get to work with the real thing, gloves off, hands dirty, but you’re in full control.
As an end-user API for people from a business background this is less suitable, and we’ll get back to how this observation surfaced in Akka.

## One Actor is always local

Going back to the original definition, an Actor is an entity living somewhere on a network node, tied to and identified by its mailbox.
It takes one message out of the mailbox, processes it, then starts over.
The Actor is created at some point in time and it may choose to become “inert” at a later point in time (which is equivalent to being stopped and re-routing its mailbox into Nirvana) — in other words, an Actor has a linear lifecycle.

Saying that one particular Actor is distributed does not make much sense because according to the rules it can only process one message at a time anyway.
Actors are building blocks for distributed systems, one Actor is not even a system, let alone a distributed one.

This presents another piece of evidence that the Actor Model doesn’t really solve high-level problems.
Business use-cases often require distributed systems for redundancy and fail-over, so that the resulting business solution has the resilience it requires.
Business entities can therefore not have a 1:1 relationship with Actors, such an entity will need to be an abstraction over a group of Actors that live in different locations.

## How do I tell my local CPU how to run my Actor?

Given that I have designed an Actor as a solution to one of my problems, how do I write that down?
The design will describe the accepted messages, the state managed by the Actor, and the logic that determines what to do with each received message.
The first two parts are types and data structures while the last part is a procedure.

Taking a step back, what does an _atomic integer_ require of me?
I only need to provide an initial value and then I can use the methods provided, like `get_and_add` or `compare_exchange`.
In case of a _mutex_ I use the provided constructor and then I can `lock` and `unlock` it, dividing my program into regions inside and outside of the exclusive zone.
One more level up, a _future_ is a handle for a value that may be provided at a later point in time.
In order to use it I need to describe a computation or some external resource and then my program uses callbacks to consume that result when it becomes available.

The funny thing is, `async`/`await` has been added to many programming languages as a tool for working with futures, but what this language construct allows you to write is the definition of an asynchronous _procedure_.
This is exactly what we need when describing how an Actor should act.
If you want to form a mental model of how an Actor works, my recommendation is to picture an asynchronous loop consuming messages from a queue.
The Actor reference is nothing but the sending side of that queue.

As most of my daily work is done in Rust nowadays, here’s how that could look:

```rust
async fn pong(mut mailbox: Mailbox<SPSC<Ping>>) -> Result<(), SenderGone> {
  let mut count = 10;
  while count > 0 {
    let Ping { mut reply } = mailbox.next().await?;
    reply.send(count);
    count -= 1;
  }
  Ok(())
}
```

The compiler will turn this into a state machine that suspends when it hits `.await`, keeps track of how many iterations remain, and returns “success of unit” when done.
This state machine implements the `Future` contract, so that the function call result of `pong(mailbox)` can be spawned as a task on a futures runtime.
I mention this here to make it dead obvious that each Actor will need a CPU to run on whenever a message needs to be processed.
The corresponding complexity of providing this infrastructure is another reason why I consider Actors as low-level tools.

## High-level business logic requires other abstractions

This whole article was sparked by Sergey’s post, which is mainly about a higher-level — and much more useful — programming abstraction.
He describes Orleans “grains” as cloud-native objects, as persistent entities with a business meaning.
A grain just exists somewhere in a silo, which is a cluster of cloud nodes in some computing center.
The important part is that a grain implements some workflow, it describes and defines an object in the virtual space — which may well have close ties to an object or process in the real world.
The programmer is freed from the concerns of when and where to schedule the evaluation of a grain or how to ensure the persistence of its state.

Akka added the [`PersistentActor`](https://doc.akka.io/docs/akka/current/persistence.html) API for the very same reasons, this API is a close cousin of Orleans’ grain.
While some design details and choices are different, their _raison d’être_ is the same: Actors are too low-level, so there exists an obvious but non-trivial extension package that presents the programmer with a more comprehensive tool.
Of course this larger package has already made some choices, it restricts the design space for the programmer, but that is exactly the reason why it is more useful.

At Actyx I recently blogged about [Local Twins](https://developer.actyx.com/blog/2021/04/29/partial-connectivity-ux), which is another example of this kind.
The design goal here is to offer replicated business logic with 100% availability in a peer-to-peer network; the logic always progresses as long as there is only a single device it can run on.
While Actors are certainly a helpful underlying primitive, Local Twins are far more useful to application programmers since they include ready-made choices for handling persistence, domain modeling, and distributed conflict resolution.

## Conclusion

Sergey looked at this topic from the perspective of teaching Orleans while using Actor vocabulary, which creates a number of difficulties.
My take is that Actors were never meant to be a high-level abstraction in the realm of business logic.
The short summary would be that we come to the same conclusion for very similar reasons, but use different paths to get there.

The Actor Model is a precise characterisation of what each individual part of a distributed system can do.
Business entities and workflows, on the other hand, describe the resulting behaviour that an underlying distributed system should achieve.
Until we as an industry have gained an understanding of the link between individual Actors and the whole system’s emergent behaviour, we will have to assume that no single concept can be stretched over this whole range without breaking.
We will thus continue to need higher-level abstractions to describe the business purpose, as well as low-level abstractions like Actors, futures, mutexes, sockets, etc. for the technical implementation.
