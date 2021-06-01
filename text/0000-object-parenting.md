- Feature Name: `object-parenting`
- Start Date: 2021-05-26
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

This RFC proposes a way to reparent `NetworkObject`s to other `NetworkObject`s at runtime while networking.

The approach we are presenting here is limited by current MLAPI architecture and its internal structure. We will touch on these limitations and reasons below.

# Motivation
[motivation]: #motivation

We want to offer `NetworkObject` reparenting solution within MLAPI to help developers synchronizing `transform` parent-child relationships of `NetworkObject`s.

It is often not easy to grok the parenting in networking, also your choice of networking framework would have its limitations you don't necessarily know up until trying to implement and cover the edge cases. Here, we want to provide a solution that plays nicely with current MLAPI limitations and we expect this "NetworkObject Parenting" concept to evolve and change over time when MLAPI matures.

With this RFC implemented, we expect developers to reparent their `NetworkObject`s at runtime during networked gameplay and synchronize across the network (server to clients) out-of-the-box within certain limits and conditions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Rules

Let's list a few basic `NetworkObject` reparenting rules below.

### Only A Server (or A Host) Can Reparent

Similar to Ownership, only the server (or host, which is both a server and a client at the same time) can control reparenting of a `NetworkObject` in the network.

Clients however, can send RPCs to server and execute a logic server-side that ultimately makes server to reparent a `NetworkObject`.

### Only Reparenting Under A NetworkObject (Or To The Root) Is Valid

A `NetworkObject` can only be reparented under another `NetworkObject` (`GameObject` with `NetworkObject` component attached). Only exception is moving a `NetworkObject` to the root of the scene hierarchy.

This is simply due to the fact that MLAPI would not be able to identify & locate new parent on the remote-side if it was a non-`NetworkObject` parent. Again, except moving it to the root because we could identify no parent (root) scenario without `NetworkObject` identification or scene hierarchy traversal.

### Only Reparenting During Networking Is Valid

A `NetworkObject` can only be reparented while networking, in other terms you can only reparent while listening/running as a server.

If we were to allow moves while not networking, we would be desynced immediately when we switch to networking. Also reparenting a `NetworkObject` under a non-`NetworkObject` while not networking would sound valid but that would not be replicable on the remote-side since MLAPI does not cover full scene hierarchy synchronization (and this might be a good thing, hence server vs client scene hierarchies).

In short, we have to keep initial `NetworkObject` formation in the scene hierarchy identical between instances so that we could rely on initial states to be in sync as reference points.

### Invalid Reparenting Will Move NetworkObject Back To Its Original Location

If an invalid/unsupported `NetworkObject` parenting happens, MLAPI will immediately pop it back to its previous location to keep things in sync and also will print relevant error/warning messages to indicate the issue.

## Moves

We will assume that our initial scene hierarchy is looking like this:

```
Sun
Tree
Camera
Player (NetworkObject)
  |_ Head
  |_ Body
  |_ Arms
  | |_ LeftArm
  | | |_ LeftHand (NetworkObject)
  | |_ RightArm
  |   |_ RightHand (NetworkObject)
  |_ Legs
Axe (NetworkObject)
```

So, let's try a few moves!

### Root/Axe → RightHand/Axe

This is a **valid** move because `Axe (NetworkObject)` is being moved under `RightHand (NetworkObject)`. We know about their `NetworkObjectId`s and it will be replicated across the network to clients by the server.

Now, our hierarchy is looking like this:

```
Sun
Tree
Camera
Player (NetworkObject)
  ├─ Head
  ├─ Body
  ├─ Arms
  │  ├─ LeftArm
  │  │  └─ LeftHand (NetworkObject)
  │  └─ RightArm
  │     └─ RightHand (NetworkObject)
  │        └─ Axe (NetworkObject) [to] <──┐
  └─ Legs                                 ├ OK
                                [from] ───┘
```

### RightHand/Axe → Body/Axe

This is an **invalid** move because `Axe (NetworkObject)` is being moved under `Body` which is _not_ a `NetworkObject`. It does _not_ have a `NetworkObjectId` and it can _not_ be replicated and synced on clients.

So, we tried to do this but it did _not_ succeed:

```
Sun
Tree
Camera
Player (NetworkObject)
  ├─ Head
  ├─ Body
  │                               [to] <──┐
  ├─ Arms                                 │
  │  ├─ LeftArm                           │
  │  │  └─ LeftHand (NetworkObject)       ├ INVALID
  │  └─ RightArm                          │
  │     └─ RightHand (NetworkObject)      │
  │        └─ Axe (NetworkObject) [from] ─┘
  └─ Legs
```

We'd get an error message in the logs similar to this:

```
Invalid parenting, NetworkObject moved under a non-NetworkObject parent
```

### RightHand/Axe → Scene Root

todo

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we _not_ do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For framework, tools, and library proposals: Does this feature exist in other networking stacks and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other projects, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC. Please also take into consideration that Unity Multiplayer sometimes intentionally diverges from common multiplayer networking features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the Unity Multiplayer as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the Unity Multiplayer in your proposal. Also consider how the this all fits into the roadmap for the project and the team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
