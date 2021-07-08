# RTT computation. A mechanism to compute and expose round-trip times
[feature]: #feature

- Start Date: `2021-07-08`
- RFC PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- SDK PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/0000)

# Summary
[summary]: #summary

This would add an API for game code to read measured round-trip times, both on the server and client. The round-trip time exposed would be the high-level RTT: How much time goes by between the moment a high-level system writes something to the network and the moment it gets a response from the other machine. 

This is distinct from transport-level RTT which captures only the latency at the network level.

This RFC provides an API to read the RTT, and a mechanism for high-level systems to drive it. It doesn't describe the first usage of this system.

# Motivation
[motivation]: #motivation

High-level systems, like snapshotting, interpolation, and client-side prediction often need to know when an action performed locally could/will be surfaced on the other machines. Game code might also be interested in the overall latency a game sees.   

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When game code needs to read latency it checks in `public NetworkManager.ConnectedClientsList[index].GetRtt();`. This new function returns

```
public struct Rtt
{
    public double best;
    public double average;
    public double worst;
    public int sampleCount;
}
```

The members of this struct describe, for a given client, the RTT. `best` shows the best RTT measured, `average` shows the average RTT measured, and `worst` shows the worst RTT measured. `sampleCount` indicates how many samples were used to get those values. A count of 0 indicates no data is available. In this case, the other fields would be 0.

`NetworkConfig` gets the following extra members:

`public int RttSize = 64;` to control the number of slots for RTT computation. This number states the maximum number of messages to track for RTT computation. It should be set just above the maximum expected number of messages in-flight. This default should be a good default, for most game.

`public int RttSamples = 5;` to control the number of latency measurements to keep and average out. This is the max `sampleCount` you'd get out of `GetRtt()`. This default should be a good default, for most game.

The `NetworkClient` type also gets renamed to `NetworkConnection` and `NetworkManager` gets a new member: `public readonly NetworkConnection ServerConnection`

This way, the client machines have an available class to use, when they want to read the RTT to the server. Namely: `NetworkManager.ServerConnection.GetRtt();`

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The section above described how the game code would read RTTs. In order to measure RTT, a high-level system must tell when messages are sent and when corresponding replies are received.

The proposed API would be in `NetworkConnection`, formerly `NetworkClient`; 

```
internal void NotifySend(int key, double when);
internal void NotifyAck(int key, double when);
```

Using those two methods, high-level system would notify the RTT system when they send a message and when they receive the matching response. For example, SnapshotSystem could do:

- `NotifySend(42, 1001.35);` when it sends message 42.
- `NotifySend(43, 1001.86);` when it sends message 43.
- `NotifyAck(42, 1001.89);` when it receives a message acknowledging 42.
- `NotifySend(44, 1001.91);` when it sends message 44.
- `NotifySend(45, 1001.99);` when it sends message 45.
- `NotifyAck(43, 1002.01);` when it receives a message acknowledging 43.

At this point, if game code were to call `GetRtt()`, they would get
- best RTT of 0.15
- worst RTT of 0.54
- average RTT of 0.345 
- sampleCount of 2

Notice that unacknowledged messages don't contribute to RTT computation. Neither do lost messages. Messages that ended up being retransmitted by lower-level transport would end-up with higher measured latency. This is by design and a feature. From game code point-of-view, this latency exists and affects gameplay. As such, it must be measured and reported. It is another way in which transport RTT and high-level RTT differ.  
 
# Drawbacks
[drawbacks]: #drawbacks

This seems like a very basic feature a developer would expect. I don't see any reason we would not do this.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

In order to come up with this RFC, we also considered:

- An alternative would be to leave each high-level system implement their own RTT computations. For example, SnapshotSystem could internally track its own RTT and use the data to drive interpolation and client-side prediction. This would hide the feature and result in a more monolithic solution.
- Another alternative is to let game code compute the RTT themselves, and possibly feed the RTT they measure into the input prediction code.

Neither is very interesting.

# Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Instead of 

`NetworkConnection`, formerly `NetworkClient` offering; 

```
internal void NotifySend(int key, double when);
internal void NotifyAck(int key, double when);
```

we could have

```
internal void NotifySend(int key);
internal void NotifyAck(int key);
```

and let the RTT computation part read the system time itself. It makes for a simpler API, but it lacks the ability of the high-level system of stating the time at which something happened. For example, SnapshotSystem might know when a message was received but not report it right away. Adding it here, for completeness sake.

# Future possibilities
[future-possibilities]: #future-possibilities

