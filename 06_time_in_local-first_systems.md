# Time in Local-First Systems

A lot has been written about the two ways of keeping track of time in distributed systems.
The first way — chosen either by the careless or the particularly resourceful — is to use physical time, measured by clocks that advance at a fixed rate.
Although broken by general relativity in principle, they work quite well if the system in question is constrained to this planet.
The second way — the way you learnt about in your distributed systems lecture — is to use logical time, first proposed by Leslie Lamport.
Here, clock ticks are driven by the communication between network nodes as well as the actions being performed on those nodes.

While the above seems to cover all (i.e. both) angles, there is indeed a crack between them.
This post explores use-cases that defy or evade all solutions known to me.

## Enter local-first cooperation

In [local-first cooperation](https://www.local-first-cooperation.org/) we strive for autonomous nodes that work together with nearby peers when and if such peers are available.
The characteristic of such networks is that there is no central authority that controls the nodes, meaning that for example the device clocks cannot be trusted across devices.
And there is also no expectation of stable or long-lasting communication relationships: nodes form fleeting associations to exchange information and work together.
In fact, many use-cases like remote maintenance workers or logistics vehicles require that individual devices are fine with being isolated over long stretches of time.
No peers, and no internet either.

## Confusion ensues

This is problematic for logical clocks.
Imagine two nodes working together from time to time and doing relevant things in between their information exchanges.
The logical clock of the one device may advance at a much smaller rate than the other, for example because more is happening on that other device or it is in contact with other nodes that also drive the clock forward.
Whenever the two nodes synchronise, the first node’s clock would make a large jump forward due to [Lamport’s algorithm](https://en.wikipedia.org/wiki/Lamport_timestamp).

Since the purpose of a clock is to generate timestamps from it and then sorting events by those timestamps, it is obvious that the first node’s events would tend to be sorted at the beginning while the second node’s events would mostly be sorted after those.
In theory, this should be fine because sorting by Lamport timestamps only guarantees correct sorting between causally related events, and the events produced during a network partition are unrelated — they are concurrent.
In practice, this makes it hard to write business logic that makes sense of the merged event history, though.
The two nodes might be mobile devices used by persons, and these persons might have communicated some of the facts recorded, for example by shouting or by a phone call.
Or a single person may have seen the two devices while wifi was down, and some of the events were created by that person on the first device only because the second device showed a relevant piece of status information.

So the first problem is that causal links may be formed outside the system, and those causal links will not be recorded and thus also not be respected within the system.

## We’re addicted to time

You’re addicted to something _exactly if_ you cannot stop using that thing even if you want to — you have no choice regarding this thing.
Time itself is of this quality, our whole existence is defined by the linear forward progression of time, so each one of us is trapped by the concept of time.

One of the consequences is that we frequently need and even want to measure the passage of time, we are compelled to quantify it.
Therefore, we cannot disregard physical clocks, as they are the only mechanism we have to measure time.
On the other hand we have not yet created clocks that we can trust across computing device boundaries.
We have developed sophisticated mechanisms to keep clocks synchronised in a continuously functioning network, and we have created foundational clocks that demand massive operational effort to keep running, but we do not yet have a cheap and portable solution to the problem in the general case.

This is the reason why the event trace of a distributed system may be difficult to represent in terms of temporal reporting.
Consider that you receive some information on your mobile phone (e.g. about a parcel delivery) and you tap a button to confirm it (i.e. to open the delivery box for the courier).
There is a clear causality between these two events, but if one of the clocks is off by a few seconds and your button tap happens quickly then the measured time interval between the two events may turn out to be negative.

How do you report this?
If you round up all negative time intervals to zero, then the sum of time intervals along a causality chain no longer matches the time that passed from beginning to end.
And if you report the computed difference in timestamps, the result is nonsensical.

Note how there is no option to report the actually correct real-world time period it took: we simply have no clock that could measure it.
We can only choose between two incorrect options.

## How can we fix this?

Here are a bunch of non-solutions:

- require device clocks to be in sync before permitting information to flow

  This is not a solution because the information itself usually carries timestamps of past events.
  Requiring the devices to sync would only produce a false sense of safety, negative reported time intervals are not prevented.
  And depending on how much hassle this synchronisation presents for the user, the network may become unusable and thus effectively useless.

- artificially advance Lamport clocks to approximate a physical clock

  This sounds enticing at first because causality will be mostly in line with physical time, but lagging clocks will still timestamp events such that they are sorted too early.
  And synchronising with a clock that is way ahead will switch the system again into pure Lamport mode because a correct clock will not make the same jump ahead.

- record as much causality information as possible and sort concurrent events by physical timestamp

  This is probably the best we can do, but it only makes it somewhat less challenging to interpret concurrent events.
  Causally related events may still describe a timeline along which the physical clock jumps backwards, which can happen when events come from different devices or when the physical clock of one device needed a significant backwards correction.

In summary, as far as I can see there is no real solution to this problem.
Under the constraints that we can trust neither our physical clocks nor our ability to communicate, we are left without a mechanism for synchronisation.

## So what can we do?

Within the constraints of pure local-first cooperation the only thing we can do is to record causality within event traces as best we can.
The interpretation of physical timestamps in these traces will always be problematic as they are open to deliberate or accidental tampering.

If we weaken the constraints, there are a few options:

1. Define and use **trusted physical clocks** that can be used to obtain qualified timestamps; this reduces the autonomy of network nodes, in particular their ability to record facts while they don’t have access to communication.
2. Employ a **consensus protocol** like Raft or Paxos to manage one centralised event trace with its timings; there probably are some subtleties in the “timings” part, nodes would effectively need to synchronise their physical clocks based on the central ledger.
3. Restrict deployment to **highly available network infrastructure** and pay the operational cost required for that; in this case any resulting negative time intervals will be anomalies due to operational failure, to be minimised by the standard operating procedures.

Next to that, there is a grey area that I’ll explore more in the future.
Instead of providing a rock solid infrastructure solution to this problem, we may also restrict the application designer so that issues caused by clock skew become less important.
The general idea is that if two devices intensely cooperate, they may reasonably be expected to communicate frequently enough so that their event traces are nicely ordered by causality links.
And then we may use physical device clocks only to measure the relative passage of time since the most recent communication.
A network of devices using such a scheme may well exhibit clock drift relative to UTC, for example due to uncertainties on message propagation delays.
But at least for networks of limited size this may well be good enough.
