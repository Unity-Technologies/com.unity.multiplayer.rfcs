# Client Network Variables
[feature]: #feature

- Start Date: `2021-12-12`
- RFC PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- SDK PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/0000)

# Summary
[summary]: #summary

Today, on the server may write to `NetworkVariables`.  This RFC introduces a bifurcation where in addition to the existing (server) NetworkVariables, we now have ClientNetworkVariables, which clients may write to.

# Motivation
[motivation]: #motivation

To enable users to more simply communicate to the server (especially input) without having to use RPCs, this feature is proposed.  It is an extension of the existing `NetworkVariables` differing mainly in that clients can write to them.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Today, one adds a (server) `NetworkVariable` by declaring one in a `NetworkBehaviour` like so:

```
public class NetworkVariableTest : NetworkBehaviour
{
    public NetworkVariable<int> SomeSeverVar = new NetworkVariable<int>();

    // ...

    void Update()
    {
        if (NetworkManager.isServer)
        {
            SomeServerVar.Value = 123; // OK
        }

        if (NetworkManager.isClient)
        {
            // InvalidOperationException: Client can't write to NetworkVariables
            // SomeServerVar.Value = 456; // ERROR
        }

    }
}
```

Note, writing to the `SomeServerVar` in the server case succeeds (and is replicated to all clients), but fails with indicated exception in the comment.

This RFC proposes a dedicated `ClientNetworkVariable` type that will work like this:

```
public class NetworkVariableTest : NetworkBehaviour
{
    public NetworkVariable<int> SomeSeverVar = new NetworkVariable<int>();
    public ClientNetworkVariable<int> SomeClientVar = new ClientNetworkVariable<int>();

    // ...

    void Update()
    {
        if (NetworkManager.isServer)
        {
            SomeServerVar.Value = 123; // OK
            SomeClientVar.Value = 456; // ERROR!
        }

        if (NetworkManager.isClient)
        {
            SomeServerVar.Value = 123; // ERROR!
            SomeClientVar.Value = 456; // OK
        }

    }
}
```

When the `SomeClientVar` code is executed...
- The `SomeClientVar` for the client on which the code ran is immediately udpated
- The client then communicates this variable update change to the server
- The server then broadcasts this to the other clients
- Note, unlike pre-1.0 versions of NGO when clients could write to network variables, the server can **not** write to client network variables.

## Private Mode

So that developers can choose to replicate the `ClientNetworkVariable` just on the server but **not** to the other clients, a `OwnerOnly` mode is avaiable.

For example, one might want to use the `ClientNetworkVariable` to transmit a "cast spell" input command to just the server (vs. today where an RPC is required) so that it can react to the input and then update the other clients accordingly (e.g., update the clients differently whether the user actually has manna or not).  To do this, one declares the `ClientNetworkVariable` with the `OwnerOnly` permission, e.g.


```
public class NetworkVariableTest : NetworkBehaviour
{
    public ClientNetworkVariable<int> SomePrivateClientVar = 
        new ClientNetworkVariable<int>(NetworkVariableReadPermission.OwnerOnly);

    // ...

    void Update()
    {
        if (NetworkManager.isClient)
        {
            // only this server and the sending client will know about this
            SomePrivateClientVar.Value = 456; 
        }

    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


# Drawbacks
[drawbacks]: #drawbacks

Why might we *not* do this?
See the "Future possibilities" for a more detailed explanation, but we likely will need to change how this system works when moving on to a snapshotted model in future major releases.  In the current system there is no canonical "on tick 123 these variables had values {...}", and so having the client broadcast values at some arbitrary time to be received by clients at some aribitrary time is not disriptive.  As a result users used to this model may have to adapt to something different in the next major release.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This feature is a building block one can use for client-authoritative features.  However it intentionally leaves the "local manifestation" element to the user.  That is, while a user can define the state and the SDK will sync it, it is up to the user whether to take that `ClientNetworkVariables` state and manifest it, say, in a character's position OR transmit it as a command to the server and wait for the server's updates.

In considering the feature and talking with developers the common feedback was this was the level of involvement they wanted from the SDK.  Moreover, while we felt reasonably confident that we could define variable use case, we were less certain we could define a full client authority system with local manifestation given all the different use cases.  Note however such additions could be added later, building on this system. 

Technial advantages of this approach:
- programmer interface is just like before
- we were able to fully re-use the existing implementation 

If we don't adopt this approach:
- users will have to continue to use RPC's to transmit input either as a single client -> server transmission or a combination of client -> server and then server -> other clients.

# Future possibilities
[future-possibilities]: #future-possibilities

## What happens in a Server Authoritative, Snapshot-based, Eventually-consisten world?
In the current (1.0) release, network variables are all communicated in a reliable networking mode.  However, in future major releases we are likely to switch to an unreliable, server-authoritative, eventually-consistent, tick-based model for Server NetworkVariables.  It is less clear what we will do with `ClientNetworkVariables` however. 

- we may formalize input passing with a dedicated system
- we may bifurcate, and continue to have this mechanism, however Client and Server network variables will be "unlinked"; that is, one might use ClientNetworkVariables to transmit non-gameplay-critical, non-tick-correlated datat (e.g. transmitting players' cursors to each other)

