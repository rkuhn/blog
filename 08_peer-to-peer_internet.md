# A peer-to-peer Internet

_In this post I articulate something I [learnt yesterday](https://github.com/libp2p/rust-libp2p/issues/2657#issuecomment-1142310386) and explore its ramifications._

For a while I’ve been dreaming of a second Internet, co-existing with the current one, consisting not of myriads of clients connecting to numerous servers but of countless peers connecting to each other.
As social beings we have no trouble imagining such a system — it matches how we interact with our friends and neighbours, with staff at businesses, our co-workers, etc.
Some interactions prompt or are prompted by physical proximity, others are conducted over remote communcation.
One restriction we are intuitively aware of is that a successful conversation can only occur if the interlocutors speak the same language, both literally as well as figuratively.
And we tend to seek out certain persons when we have a concrete problem to solve.

## How does this translate to programming?

It would be natural to give each network participant — be that a mobile phone, a server in the cloud, or a thermometer — a cryptographic identity and call that `PeerId`.
Each peer would then maintain an “address book” associating names and purposes with `PeerId`s.
And there would be a form of neighbourhood discovery by which each peer can learn of other peers that are nearby, including some levels of indirect reachability (like a peer of mine could see other peers I cannot see with my own radio antenna).

In this world, a network application would want to deal in `PeerId`s: open a stream to that peer, receive datagrams from that peer, etc.
Whether the application implements video chat, document sharing, or a hiking tour guide doesn’t matter for our further deliberations.
The important conclusion from this generality is that in addition to `PeerId` we need the concept of a `ProtocolName`: the intended language to be spoken.
There would be a plethora of such names because there are many different and useful ways of implementing any of the aforementioned applications and their communication.
`ProtocolName`s would naturally employ namespacing and versioning, e.g. `/voice/rtp` or `/doc/automerge/v2`.

So if I wanted to make a voice call to a friend, I’d use an app that tries to open a signaling stream using the range of protocols supported locally.
The friend’s `PeerId` would be searched in the network neighbourhood or in a [DHT](https://en.wikipedia.org/wiki/Distributed_hash_table), a communication path found (like an IP address and port), and a low-level connection made.
Then both sides compare their lists of supported `ProtocolName`s and choose one from the intersection.
The application uses the negotiated protocol to stream the audio bits back and forth over the established connection.

## The state of the art

Contrast that with how network programming is done today: the application deals in IP addresses and port numbers, where the former is usually ephemeral and the latter is a very weak representation of a `ProtocolName` with no negotiation.
This established paradigm works well for clients connecting to servers at well-known coordinates, where IP addresses and port numbers are stable.

But it does not work well for peer-to-peer networking.
How should an app on a watch know that the peer I want to communicate with happens to be in the same private network right now?
Even though it costs far less to use the local network, such calls today are still routed via cloud nodes because that is much easier to program.
Local network awareness is being added to iOS and Android, but its growth is slow because each application needs to explicitly make provisions for local communication, using proprietary protocols.

One ongoing effort to improve this situation is [libp2p](https://libp2p.io/), which I’ve been using (and very slightly improving) for the past few years at [Actyx](https://developer.actyx.com/).

## But what about privacy?

If I use a single `PeerId` to make myself discoverable, then everyone else will be able to follow my (network) movements.
In the real world, following someone’s every move is considered inacceptable behaviour and is in many jurisdictions even punishable by law.
With the technical means sketched and implied above, it would become possible to stalk a person from anywhere on the planet without risk of being noticed (modulo possible but costly countermeasures).

Taking a look at the situation today, we find that cloud services perform the implicit function of hiding their users’ location.
Only when switching to a direct connection for real-time data streams can we see the current IP addresses of our interlocutors.
These addresses are in the vast majority of cases only temporarily associated with a mobile device or home network; ordinary people have no legal access to the real name behind a transport address.
The association with a specific person (e.g. via a username) is encapsulated by the cloud service.

We can transfer this scheme into the peer-to-peer world by relegating the aforementioned `PeerId` to become an ephemeral transport identity.
The personal identity may have the appearance and cryptographic structure of a `PeerId` and its location would be hidden by a relay service.
This has recently been implemented in [libp2p DCUtR](https://github.com/libp2p/specs/blob/master/relay/DCUtR.md), albeit for the reason of allowing connections to peers behind strict firewalls.
If two peers trust each other enough, a relay service could also facilitate a direct connection between them, e.g. by selectively disclosing the current transport identity.

## The path forward

I’ll keep experimenting with these concepts in the context of [rust-libp2p](https://github.com/libp2p/rust-libp2p).
The currently emerging plans for opening negotiated substreams to some `PeerId` sound very promising and should be complemented by a mechanism for sending and receiving unreliable datagrams.
In addition to these point-to-point facilities there is already the battle-tested [gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md) protocol for broadcasts.
Together with the infrastructure services of [identify](https://github.com/libp2p/specs/tree/master/identify), [kademlia](https://github.com/libp2p/specs/tree/master/kad-dht), etc. this should become a fertile ground for exploring peer-to-peer and local-first network software design.

