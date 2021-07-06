# NetworkObject Parenting
[feature]: #feature

- Start Date: `2021-05-26`
- RFC PR: [#17](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/17)
- SDK PR: [#855](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/855)

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

## Opt-out

Before we dig deeper into this feature proposal, it needs to be said that this feature is going to be behind a bool flag that can be toggled on the `NetworkObject` inspector UI, which will be enabled by default but experienced developers can opt-out from it and wouldn't be running any of this logic therefore they could to implement a solution on their own as well.

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
  ├─ Head
  ├─ Body
  ├─ Arms
  │  ├─ LeftArm
  │  │  └─ LeftHand (NetworkObject)
  │  └─ RightArm
  │     └─ RightHand (NetworkObject)
  └─ Legs
Axe (NetworkObject)
```

So, let's try a few moves!

### Root/Axe → RightHand/Axe

This is a **valid** move because `Axe (NetworkObject)` is being moved under `RightHand (NetworkObject)`. We know about their `NetworkObjectId`s and it will be replicated across the network to the clients by the server.

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

This is an **invalid** move because `Axe (NetworkObject)` is being moved under `Body` which is _not_ a `NetworkObject`. It does _not_ have a `NetworkObjectId` and it can _not_ be replicated and synced on the clients.

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

### RightHand/Axe → SceneRoot/Axe

This is a **valid** move because `Axe (NetworkObject)` is being moved to the scene root (no parent). Even though there is no `NetworkObjectId` to sync, empty/null parent _can_ be synced across the network on the clients.

Our up-to-date hierarchy is now looking like this:

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
  │               [from] ───┐
  └─ Legs                   ├ OK
Axe (NetworkObject) [to] <──┘
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We'll utilize [`MonoBehaviour.OnTransformParentChanged()`](https://docs.unity3d.com/ScriptReference/MonoBehaviour.OnTransformParentChanged.html) under `NetworkObject` to catch `transform.parent` changes.

We'll store 3 additional state variables in `NetworkObject`:

```cs
bool m_IsReparented; // did parent change compared to initial scene hierarchy?
ulong? m_LatestParent; // who (NetworkObjectId) is our latest (current) parent if we changed our parent?
Transform m_CachedParent; // who (Transform) was our previously assigned parent?
```

We'll also add another new virtual method into `NetworkBehaviour`:

```cs
/// <summary>
/// Gets called when the parent NetworkObject of this NetworkBehaviour's NetworkObject has changed
/// </summary>
virtual void OnNetworkObjectParentChanged(NetworkObject parentNetworkObject) { }
```

There are 2 main codepaths we need to consider when sychronizing `NetworkObject` parenting:

1. At Object Spawn
    - Client spawns objects including static scene objects and dynamic spawned objects on join.
    - We serialize `NetworkObject`s with their payloads (such as `NetworkBehaviour`s etc.)
    - We will also write `m_IsReparented` and `m_LatestParent` fields to sync on the client-side
2. During Gameplay
    - When a valid `NetworkObject` reparenting happens during networked gameplay on the server-side, it will be replicated across the network to the connected clients to sync
    - We will write `m_IsReparented` and `m_LatestParent` fields into a `NetworkBuffer` and send that over to all connected clients with `PARENT_SYNC` message type on `MLAPI_INTERNAL` channel

Transform parent synchronization will rely on initial formation of transforms in the scene hierarchy being identical on all standalone instances (note: this is pre-this-RFC in MLAPI, not introduced by this RFC or feature).

# Drawbacks
[drawbacks]: #drawbacks

## Limiting Non-Networked NetworkObject Transform Parenting

Rules outlined above are applied and enforced even while not networking (not hosting or connected). More specifically, if you were to try reparenting a `NetworkObject` under a non-`NetworkObject`, that'd be invalid and reverted even though you are not hosting or connected to a server.

This is due to several limitations caused by current MLAPI design and resolving these issues are not in the scope of this proposed feature work.

If framework were to allow any arbitrary `NetworkObject` parenting which might also include invalid moves, player would be desynced upon arrival/join/connect to the server. We might try to keep local changes in a buffer/cache but we still cannot guarantee those moves would be synced without issues on connect — so, we play safe instead of half-working approach here and still apply rules and prevent `NetworkObject` parenting to desync while not networking.

Having said that, we could expect this limitation to change and potentially no longer exist if we were to address fundamental `NetworkObject` & `NetworkBehaviour` architecture in MLAPI.

## Implementing A High-Level Concept In A Low-Level Context

Both scene transform hierarchy and transform reparenting are higher-level and even arguably non-network concepts but they happened to appear in MLAPI's low-level design primitives. Network framework does not necessarily need to know about the scene hierachy, transforms and other systems in order to synchronize state of network entities (`NetworkObject`s) over the network. However, existing MLAPI architecture forms an hierarchy based on the scene transform hierarchy but it does not fully mirror it. We also have other bespoke `NetworkBehaviour` derived types such as `NetworkTransform`, `NetworkAnimator`, `NetworkRigidbody` etc. which are not considered as core parts of the MLAPI framework but rather supplied utilities/modules. This (re)parenting feature would be implemented at much higher-level as `NetworkTransform`/`NetworkRigidbody` if we were to isolate MLAPI core primitives from higher-level systems and concepts. This might change later in the roadmap or in future releases but FWIW, we still believe delivering `NetworkObject` reparenting out-of-the-box serves some customers and saves their time to better spend on the game they are crafting.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Given the limited time and scope, we think we cannot re-architect MLAPI fundamentals to support an approach where `NetworkObject` and `NetworkBehaviour` components are simply unaware of scene and transform hierarchy but they are plus parenting to be provided by a higher-level construct outside MLAPI core such as `NetworkTransform`. We wanted to move forward with what we already have.

# Prior art
[prior-art]: #prior-art

Both UNet and Unreal does not offer a similar solution out-of-the-box and mostly leave it up to developers to roll their own. This option gives enough freedom to developers so that they could address their specific needs in their project in the given context. We might favor this approach but ultimately, product team wanted to offer something that works out-of-the-box, mostly targeting beginners and non-experts.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Why can't we queue up all the changes to a buffer in non-networked gameplay and simply sync all of them on connect/join?
    - Technical MLAPI limitations, see details above in guide & reference level explanation sections.
- Why `NetworkObject` parenting can't be client-driven?
    - Similar to "Ownership" model we have, we trust server to be the source of truth and we simply rely on server-side logic for both ownership and here for parenting too.

# Future possibilities
[future-possibilities]: #future-possibilities

Potentially, we'll rework low-level core parts of MLAPI which would involve `NetworkObject`, `NetworkBehaviour` etc. and that'd also throw away all scene transform hierarchy related logic out and bump this user-need/feature to a higher-level construct like `NetworkTransform` in the future.
