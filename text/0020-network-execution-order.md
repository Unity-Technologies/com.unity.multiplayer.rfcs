- Feature Name: NetworkExectionOrder
- Start Date: 2021-06-30
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0020](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0020)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

Execution order is a core part when writing gameplay loops. Unity has built in feature for execution order which we can leverage with MLAPI. This RFC introduces minor changes which allow us to access the Unity execution order.

This RFC is a step towards providing our users a `NetworkFixedUpdate` function which will replace `FixedUpdate` for `NetworkBehaviours`.

# Guide-level explanation

This is an implementation oriented (framework internal RFC). This has currently no effects on how our users interact with MLAPI. In the future some of our features will use script execution order to provide users with a guarnteed execution order.

# Reference-level explanation

The Unity execution order can be accessed via the mono importer with the following code:
```csharp
MonoImporter.GetExecutionOrder(MonoScript.FromMonoBehaviour(this));
```
`MonoImporter` is part of the UnityEditor namespasce so we can not use it at runtime. Because of that our `NetworkBehaviours` will cache their execution order. The implementation for this will be done in `Awake` of `NetworkObject`. It will iterate over all its `NetworkBehaviours` and assign an execution order to them based on the value it got from `MonoImporter`. The reason for why we don't do this in `NetworkBehaviour` is because a derived class might define its own `Awake` function and hide the underlying implementation. `NetworkObject` is sealed so that cannot happen.


The execution order will be stored in a `NetworkExecutionOrder` property which is publicly accessible. This property can be used future MLAPI features or also directly accessed by users if they need an execution order.
```csharp
public int NetworkExecutionOrder { get; } // this will be on NetworkBehaviour
```

# Rationale and alternatives

In some cases one might want to update execution order by code. For instance having a different execution order on the same NetworkBehaviour on the server vs the client could be desired. We could expose a property which allows to manually set the execution order of unspawned network objects. This might limit our ability to order some other events such as spawn calls in the future.
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
This design brings a few inconveniences with it. Mainly by using the default MLAPI spawn handlers it won't be possible to override the execution order before the object gets spawned. It is also unclear whether there would be a use-case for this.

# Prior art

# Unresolved questions

# Future possibilities

We might want to integrate execution order into `OnNetworkSpawn` and `OnNetworkDespawn` callbacks in the future.