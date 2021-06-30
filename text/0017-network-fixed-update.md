- Feature Name: NetworkFixedUpdate
- Start Date: 2021-06-30
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0021](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0021)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

A `NetworkFixedUpdate` event function will be introduced for `NetworkBehaviour`. `NetworkFixedUpdate` gets called once per network tick and provides a conventient way to run logic during each network tick.

# Motivation
[motivation]: #motivation

As explained in the [NetworkTime RFC](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/14) time in a networked game does not always steadily advance like regular `Time.time` in Unity. Because of that our network tick does not match the fixed tick of the Unity Engine `FixedUpdate`. `NetworkFixedUpdate` is an event function which gets called once per network tick and replaces `FixedUpdate` for network objects.

Running code at a fixed rate is necessary for some systems like Physics in addition if values are always only changed on a fixed rate we can provide interpolation to smooth out bad networking conditions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`NetworkFixedUpdate` should be preferably used on `NetworkBehaviours` instead of `FixedUpdate`. `NetworkFixedUpdate` will be called once per network tick so changing a NetworkVariable once per `NetworkFixedUpdate` will result in all changes being transmitted over the network. `FixedUpdate` does not have that guarantee because two fixed update steps could be executed during a single network tick.

Using `NetworkFixedUpdate` will result in more smooth interpolation of NetworkTransforms and other components.

Here is an example of how to use `NetworkFixedUpdate`.
```csharp
public class SampleNetworkBehaviour : NetworkBehaviour
{
    public override void NetworkFixedUpdate()
    {
        // Your code here instead of in FixedUpdate

        var deltaTime = NetworkManager.LocalTime.FixedDeltaTime; // Use this instead of Time.fixedDeltaTime
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`NetworkBehaviour` will have a virtual `NetworkFixedUpdate` function. In `OnNetworkSpawnInternal` this function will be subscribed to the `NetworkManager.TimeSystem.Tick` event. In `OnNetworkDespawnInternal` it will be unsubscribed again. This will cause `NetworkFixedUpdate` to be called once per tick on all `NetworkBehaviours` on a spawned `NetworkObject`.

For runtime performance reasons we will do a check whether `NetworkFixedUpdate` has been overridden and not subscribe to the tick event if it hasn't been overriden.

# Drawbacks
[drawbacks]: #drawbacks

Adding another event based update function incurs quite a bit of a performance overhead especially if there is a large quantity of `NetworkObjects`. In most cases this should not be an issue since `FixedUpdate` should be replaced with `NetworkFixedUpdate` and only in very rare cases both of them would need to be on the same `NetworkBehaviour.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could not write this functionality as a built in part of NetworkBehaviour and instead Users could write code which manually subscribes to the `NetworkManager.TimeSystem.OnTick` event if they need this functionality.

# Prior art
[prior-art]: #prior-art

## Bolt

Bolt uses the regular Unity `FixedUpdate` function for gameplay code. It does that by polling at the begin of `FixedUpdate` (order -10'000) and flushing at the end of `FixedUpdate` (order 10'000). This pattern caused a lot of issues for bolt because of a lack of control over when ticks happen.

## DOTS Netcode

DOTS Netcode has the concept of a `GhostPredictionSystemGroup` which runs a set of gameplay systems once per network tick and multiple times during a rollback/reconciliation. Introducing `NetworkFixedUpdate` will allow us to implement a similar pattern but for `GameObjects`.

## UNet / Mirror

UNet and Mirror do not have an equivalent to this.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

- We will expose ways to users for controlling the execution orders of `NetworkFixedupdate`
- Prediction will be possible by predictively running the `NetworkFixedUpdate` function on a subset of NetworkObjects.
- We will introduce a network physics system which will run the physics simulation once after `NetworkFixedUpdate` each tick.