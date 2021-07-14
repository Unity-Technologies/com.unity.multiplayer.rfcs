# NetworkFixedUpdate & NetworkTick Order
[feature]: #feature

- Start Date: `2021-07-14`
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0021](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0021)
- SDK PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/0000)
# Summary
[summary]: #summary

A `NetworkFixedUpdate` event function will be introduced for `NetworkBehaviour`. `NetworkFixedUpdate` gets called once per network tick and provides a conventient way to run logic during each network tick.

The `Tick` event of the `NetworkTickSystem` will be replaced with a data structure which allows to subscribe delegates in a specific order. This will allows us to control in which order we execute things during the network tick. `NetworkFixedUpdate` will use this feature as well to allow users to specify an execution order.

# Motivation
[motivation]: #motivation

As explained in the [NetworkTime RFC](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/14) time in a networked game does not always steadily advance like regular `Time.time` in Unity. Because of that our network tick does not match the fixed tick of the Unity Engine which means `FixedUpdate` cannot be used in a networked context and we need a replacement. We add `NetworkFixedUpdate` which is an event function which gets called once per network tick and replaces `FixedUpdate` for network objects.

Running code at a fixed rate is necessary for some systems like Physics in addition if values are always only changed on a fixed rate we can provide interpolation to smooth out bad networking conditions. This means we will also need to locally alias (interpolate) our state in Update to reduce visual stuttering. This is a commonly used practice in game engines and physics systems. This RFC will not cover aliasing / interpolation but `NetworkFixedUpdate` will allow us to implement this in the future.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`NetworkFixedUpdate` should be preferably used on `NetworkBehaviours` instead of `FixedUpdate`. `NetworkFixedUpdate` will be called once per network tick so changing a NetworkVariable once per `NetworkFixedUpdate` will result in all changes being transmitted over the network. `FixedUpdate` does not have that guarantee because two fixed update steps could be executed during a single network tick.

Using `NetworkFixedUpdate` will result in more smooth interpolation of NetworkTransforms and other components (In the future, not now).

Here is an example of how to use `NetworkFixedUpdate`.
```csharp
[DefaultExecutionOrder(42)] // Script execution order is supported by NetworkFixedUpdate
public class SampleNetworkBehaviour : NetworkBehaviour
{
    public override void NetworkFixedUpdate()
    {
        // Your code here instead of in FixedUpdate

        var deltaTime = NetworkManager.LocalTime.FixedDeltaTime; // Use this instead of Time.fixedDeltaTime
    }
}
```

## Execution Order

`NetworkFixedUpdate` supports execution order (both by setting it in the inspector window or by using the `DefaultExecutionOrder` attribute). There is one difference between Unity execution order and MLAPI execution order. We restrict execution order to a range of [-20000, 20000]. Values outside that range will be clamped to the range and a warning will be logged to the console.

## Tick Event
Advanced users can manually subscribe functions to the tick event which gets called once per NetworkTick. The following code can be used to register a function to the tick event:

```csharp
private void Awake()
{
    short executionOrder = 42;
    NetworkManager.NetworkTickSystem.Tick.RegisterFunction(MyFunction, executionOrder);
}

private void MyFunction()
{
    Debug.Log("Tick");
}
```
There is also a `DeregisterFunction` function for deregistering a function from the event which similar to c# event should always be called on dispose/destroy.

The tick only supports functions with 0 parameters which no return parameter (Delegates of type `Action`).

## But what if I want to use Update / FixedUpdate?

Update / FixedUpdate can be used for gameplay code but by doing so users will end up with the following issues:

- Jitter when RTT changes and the network time needs to correct itself.
- No guarantee that the built in interpolation (which we will have soon<sup>TM</sup>) of MLAPI will work smoothly.
- Additional delay of `~1/framerate` as NetworkVariables will be sent during the next frame before the next Update. (This is already the case currently in MLAPI, maybe we will fix this in the future when we rethink the network frame)
- Some other future features such as network physics or prediction might not work.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## NetworkFixedUpdate

`NetworkBehaviour` will have a virtual `NetworkFixedUpdate` function. In `OnNetworkSpawnInternal` this function will be subscribed to the `NetworkManager.TimeSystem.Tick` event. In `OnNetworkDespawnInternal` it will be unsubscribed again. This will cause `NetworkFixedUpdate` to be called once per tick on all `NetworkBehaviours` on a spawned `NetworkObject`.

For runtime performance reasons we will do a check whether `NetworkFixedUpdate` has been overridden and not subscribe to the tick event if it hasn't been overridden.

## Ordered Events
We will introduce a new `SortedEvent` type which allows to register delegates with a execution order (short). This type is essentially a thin wrapper around `SortedDictionary<short, HashSet<Action>>`. The sorted event type will expose a `RegisterFunction` and `DeregisterFunction` functions. Both of them take an execution order so when a function get unregistered the same execution order must be provided which was used to register the function.

The `NetworkTickSystem's` `Tick` event will be replaced with this `SortedEvent` type.

## Network Tick Stages

The order in the `Tick` event gives us more control over execution order during the network tick. We will use this to order internal systems which run during the network tick which allows us to decouple them from NetworkManager. A `NetworkTickStage` enum backed by a short will be added to the `NetworkTickSystem` and internal MLAPI systems should use this enum when subscribing to the tick event. The current implementation uses the following tick stages:

- [-20'000, 20'000] => Reserved tick range for user code
- -32'678 (min value) => `NetworkTickStage.Initialization`: System initialization which is not dependent on ordering.
- 32'678 (max value) => `NetworkTickStage.End`: Cleanup stage at the end of the tick. Runs `BufferManager.CleanBuffer` currently.
- 20'100 => `NetworkTickStage.Physics`: Reserved for the future when we need to run physics after the tick
- 21'000 => `NetworkTickStage.NetworkBehaviourUpdate`: Runs `NetworkBehaviourUpdater.NetworkBehaviourUpdate` which updates `NetworkVariables` at the end of the tick and sends changes out (will probably be replaced with snapshot system)

The NetworkManager will register the internal functions to the tick system when MLAPI gets started and deregister them on shutdown.

*(Network Game Update Loop diagram is taken from the Network Game Update Loop proposal and made by Noel Stephens)*
![Network Tick STages](0021-network-fixed-update/tickStages.png)

## Execution Order of NetworkFixedUpdate

For simplicity `NetworkFixedUpdate` will use the built in script execution order which our users are already using for regular Unity code.

Because ScriptExecutionOrder can only be accessed in the editor it will be cached inside each `NetworkBehaviour` component as a serialize field which is hidden in the inspector. In OnValidate we will update the execution order like this:
```csharp
public void OnValidate()
{
#if UNITY_EDITOR
    m_NetworkExecutionOrder = MonoImporter.GetExecutionOrder(MonoScript.FromMonoBehaviour(this));
#endif
}
```

# Drawbacks
[drawbacks]: #drawbacks

The way we do ordering works great if we expect the tick to be sequentially executed on the main thread. It is not a great design if we want to jobify / parallelize parts of the network tick. If we want that a design similar to DOTS ECS with `UpdateBefore` / `UpdateAfter` options makes more sense as it can allow us to run many stages at the same time in parallel.

Adding another event based update function incurs quite a bit of a performance overhead especially if there is a large quantity of `NetworkObjects`. In most cases this should not be an issue since `FixedUpdate` should be replaced with `NetworkFixedUpdate` and only in very rare cases both of them would need to be on the same `NetworkBehaviour.

The way we do ordering might bring some complications when implementing ordering for OnNetworkSpawn/OnNetworkDespawn. The main problem identified are:
- Global ordering of spawn functions can be confusing because we will only guarantee this for objects which are spawned during the same tick.
- The implementation for having globally ordered spawning is a bit more complicated and performs worse then other (per object) alternatives.
- If we decide that ordering works on a non-global level for spawn/despawn this inconsistency might confuse users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could not write this functionality as a built in part of NetworkBehaviour and instead Users could write code which manually subscribes to the `NetworkManager.TimeSystem.OnTick` event if they need this functionality.

## Setting ExecutionOrder at Runtime

We could consider allowing setting execution order at runtime.

In some cases one might want to update execution order by code. For instance having a different execution order on the same NetworkBehaviour on the server vs the client could be desired. We could expose a property which allows to manually set the execution order for networking purposes. This might limit our ability to order some other events such as spawn calls in the future.

```csharp
public int NetworkExecutionOrder
{
    get => m_NetworkExecutionOrder;
    set 
    {
        if(IsSpawned)
        {
            throw new SpawnStateException($"Cannot change {nameof(NetworkExecutionOrder)} on spawned {nameof(NetworkObject)}")
        }
        m_NetworkExecutionOrder = value;
    }
}
```
# Prior art
[prior-art]: #prior-art

## Bolt

Bolt uses the regular Unity `FixedUpdate` function for gameplay code. It does that by polling at the begin of `FixedUpdate` (order -10'000) and flushing at the end of `FixedUpdate` (order 10'000). This pattern caused some issues for Bolt because of a lack of control over when ticks happen.

## DOTS Netcode

DOTS Netcode has the concept of a `GhostPredictionSystemGroup` which runs a set of gameplay systems once per network tick and multiple times during a rollback/reconciliation. Introducing `NetworkFixedUpdate` will allow us to implement a similar pattern but for `GameObjects`.

## UNet / Mirror

UNet and Mirror do not have an equivalent to this.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Why is this not a Player Loop?

That's a good question. We could consider having this a a Player Loop in a similar way to how `FixedUpdate` works. In that case the `NetworkTickSystem` would just provide this tick loop with the information of how many ticks to process and then the player loop stage would execute that many `NetworkFixedUpdates`.

One thing to consider is that in the future we will need the ability to run a network tick during prediction. This tick will most likely be run from inside our prediction / reconciliation system. With the current implementation we can simply use another NetworkTickSystem specific to prediction and subscribe all relevant NetworkObjects to that system. If we implement this as a PlayerLoop it is unclear how we could implement this.


# Future possibilities
[future-possibilities]: #future-possibilities

- Prediction will be possible by predictively running the `NetworkFixedUpdate` function on a subset of NetworkObjects.
- We will introduce a network physics system which will run the physics simulation once during a late network tick stage each tick.
