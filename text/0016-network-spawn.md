- Feature Name: `network-spawn`
- Start Date: 2021-05-25
- RFC PR: [RFC#16](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/16)
- Issue: [MLAPI#864](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/issues/864)

# Summary
[summary]: #summary

Rename `NetworkStart` to `OnNetworkSpawn` and introduce `OnNetworkDespawn` function.

# Motivation
[motivation]: #motivation

- Many users try to subscribe to events in `NetworkStart` but then have no place to unsubscribe. As a user I want to know when my NetworkObject gets spawned and despawned. The proposed API will make this accessible.
- `NetworkStart` might lead people to think that it is a similar initialization function as `Start` or `OnEnable` which is not true. The new wording will be more direct.
- This will also make pooling/addressable support etc. much more straight forward because we decouple object life cycle from network lifecycle.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`NetworkStart` has been renamed to `OnNetworkSpawn`. `OnNetworkSpawn` gets called when a server spawns an object on the server and gets called when a client replicates the spawn of a NetworkObject.

`OnNetworkDespawn` gets called when the NetworkObject gets despawned. On the server this will be called when the object gets despawned with `NetworkObject::Despawn` or destroyed. On the client this will be called when a NetworkObject gets despawned by the server.

`OnNetworkSpawn` and `OnNetworkDepawn` are great places to subscribe to network related events such as OnValueChanged from NetworkVariable.
```csharp
public class Example : NetworkBehaviour
{
    private NetworkVariableFloat m_Variable = new NetworkVariableFloat();

    public override void OnNetworkSpawn()
    {
        m_Variable.OnValueChanged += MyValueChangeFunction;
    }

    public override void OnNetworkDespawn()
    {
        m_Variable.OnValueChanged -= MyValueChangeFunction;
    }

    private void MyValueChangeFunction(float prev, float next)
    {
        Debug.Log($"previous {prev}, next {next}");
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Both the internal `InternalNetworkStart` and virtual `NetworkStart` will be renamed to the new naming convention. In addition we will introduce the same two functions "`InternalNetworkDespawn`, `OnNetworkDespawn`" to NetworkBehaviour.

The internal function will be used so that NetworkBehaviours can subscribe to internal MLAPI events.

`OnNetworkDespawn` will also be called on the server when a `NetworkObject` gets destroyed because that also counts as a despawn. 

The implementation of this will work by mirroring what's already there for `NetworkStart`. The following will be introduced to `NetworkObject` to call the despawn event functions on all child NetworkBehaviours:
```csharp
internal void InvokeBehaviourNetworkDespawn()
{
    for (int i = 0; i < ChildNetworkBehaviours.Count; i++)
    {
        ChildNetworkBehaviours[i].InternalOnNetworkDespawn();
        ChildNetworkBehaviours[i].OnNetworkDespawn();
    }
}
```

MLAPI already has internal code which runs on every despawn. We will leverage that code. The function doing this is `NetworkSpawnManager::OnDestroyObject` (this will be renamed to `OnDespawnObject` because that's actually what this function does. Inside this function there is a line which does: `networkObject.IsSpawned = false;` after that line the following will be added: `networkObject.InvokeBehaviourNetworkDespawn();`. This runs the despawn event function at the correct point in time which is before any PrefabHandlers run.

# Drawbacks
[drawbacks]: #drawbacks

Existing users will need to rename `NetworkStart` to `OnNetworkSpawn` (breaking change) besides that no drawbacks have been found.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could have just introduced `OnNetworkDespawn` and kept the naming of `NetworkStart`.

# Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

This change will allow us to implement future internal systems. Being able to subscribe and unsubscribe to events when a NetworkObject gets spawned or despawned will be crucial for many future components.
