- Feature Name: Authority
- Start Date: 2021-05-19
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0015](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0015)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

The authority concept is an abstraction layer for gameplay logic on top of MLAPI. It allows high level components to write game play code without the need of considering the underlying network model. Authority does not directly affect other parts of the framework.

# Motivation
[motivation]: #motivation

## Existing Networked Components

While developing our new `NetworkTransform` we already saw the need to introduce an authority switch. We currently have many other high level components which still face the same problem as `NetworkTransform`. An authority system will also help us introducing future components.

`NetworkTransform` has already an authority switch. NetworkNavMeshAgent and NetworkAnimator will need one too. Instead of solving the same problem over and over again we will solve it once.

## Network Physics

The most common and simple way to provide networked physics is to run the dynamic simulation only on the authoritative peer. For co-op games this will be the physics option we can provide to users

The network physics system will have network rigidbody components which use the authority system to decide whether to run the dynamic simulation on that rigidbody and to replicate the information to the right peers.

## Accessibility
Right now using MLAPI requires extensive knowledge of multiplayer concepts. Beginners often struggle with simple tasks like figuring out the correct way of moving a player object etc. The authority concept among other features will simplify this.

As a gameplay programmer with the authority system, I will have a simple implementation pattern I can follow and will not need to think about netcode topics such as network topologies.

A common problem with abstraction layers which aim for simplicity is that they are often not flexible enough for advanced users and promote anti patterns. The aim with the authority system is the opposite. Our goal is to provide users with more flexibility. More details about this in the next section.

## Flexibility & Multi Platform Support

The current API of the MLAPI is quite inflexible. With MLAPI our goal is to have a solid foundation which is flexible and extensible and we want to support core Unity features such as our multi platform support.

The way MLAPI currently works would often not allow such a switch without rewriting a majority of the components. The authority system will provide the necessary flexibility to switch to different authority models.

Being able to launch a multiplayer game on multiple platforms is important for a lot of our users. Different platforms have very different networking needs:
- Mobile : high latency, high jitter, almost no hackers
- PC: low latency, low jitter, many hackers
- Console: low latency, low jitter, almost no hackers

Based on those platforms conditions different authority modes might be more ideal for some of them. `Mobile` for instance could profit from `Owner Authority` while `PC` or `Console` might want server authority.

With the authority mode feature users will be able to:
1. To pivot or expand to a different game mode/genre while in development by just changing the authority mode on their NetworkObjects.
2. Build multiplayer games for multiple platforms by just writing gameplay code and switching the authority model based on the platform. We can even explore support of cross platform play with multiple authority models (PC player with a predictive server authority model + mobile player with client authority in the same match)

## CO-OP Games

There are very compelling arguments to be made that CO-OP games should use predictive gameplay features because it enhances the player experience. The biggest downside of using a predictive over client authoritative model is usually CPU cost on client machines but as performance and core counts rise on lower end devices this becomes much less of an issue. Our competitors are also exploring this space and plan to provide solutions for this space..

While we cannot build all the technology to support predicted games now. If we introduce a stable foundation which focuses on the gameplay simulation now we can have people migrate to a client side predicted model by just switching the authority mode to a predicted mode later. If we have users write code with our current API a major rewrite would be needed to switch to a prediction model.

## Modularity
Currently when developing with MLAPI it is very hard to develop modular components. Each component might have different logic and a list of expectations which are not compatible with each other.

If we want multiple components such as `NetworkAnimator` and `NetworkTransform` to work together we must ensure that they both use the same authority mode. The `Authority` mode is a way for a NetworkBehaviour to define how it works and what network model it supports. If we have each component implement it's own form of authority then cross component accessibility will be unclear. 

While the authority system will not fully solve this problem it will help developers create components which can be used in a composition based architecture. This allows a user to build reusable components which is a core feature of Unity when creating single player games.

## Why do this now?
We have existing components, samples and new features such as Network Physics which all benefit from the authority system.

By introducing the authority system now we can ensure that all our systems and components provide a stable and extensible foundation which later can be used to support many different game genres. Without the concept of an authority mode we will risk needing to rewrite major part of the MLAPI such as our physics or animation systems.

Soon we will also need to introduce other systems such as a command stream system for more competitive games. These systems are based on the concept of authority so we need to have that in place first.

While the internals of the authority system and it's scope might change over time we plan to introduce a stable core API for authority which is extensible and will not need to be changed later on.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

NetworkObjects have an `Authority` switch. The Authority switch can be used when creating a `NetworkPrefab` to decide which authority model the `NetworkObject` should use.

Currently there are two `Authority` modes:
- Owner: The owner of the `NetworkObject` has authority over it.
- Server: The server always has authority over the `NetworkObject.

![dropdown NetworkObject](./0015-authority/dropdown-ownership-networkobject.png)

We might add new authority modes in the future.

Having Authority over a `NetworkObject` does the following
- There is a `HasAuthority` property on the `NetworkObject` and each `NetworkBehaviour` which will be set accordingly.
- Built in MLAPI components such as `NetworkTransform` or `NetworkAnimator` will synchronize the state of the authority to other peers (by using the correct NetworkVariable permissions internally)
- NetworkVariables with `ReadPermission = NetworkVariablePermission.Authority` can be read from. (write permission also exists)

Having Authority over a `NetworkObject` **does not** change MLAPI features such as how NetworkVariables or RPCs are sent/received etc. In the future we might have authority modes such as `Predicted` which will change this. But this is not part of the authority mode but will be implemented by the specific feature like `prediction`.

## HasAuthority
The `HasAuthority` property can be accessed to check if we have authority over a NetworkObject. The following code example moves the NetworkObject to the `m_TargetPosition` but the movement is always driven by the authority. So this script could be used in both a client authoritative and server authoritative game.

```csharp
public class SampleClass : NetworkBehaviour{
    
    [ServerRpc(RequireOwnership = false)]
    public void SetPositionServerRpc(Vector3 position)
    {
        SetPosition(position)
    }

    public void SetPosition(Vector3 position)
    {
        transform.Position = position;
    }

    public void Update()
    {
        if(HasAuthority)
        {
            if(IsServer)
            {
                SetPosition(m_TargetPosition);
            }
             else
            {
                SetPositionServerRpc(m_TargetPosition);
            }
        }
    }
}
```

## NetworkVariables
NetworkVariables will get an additional write and read permission. `NetworkVariablePermission.Authority` can be used to make a NetworkVariable only writable/readable by the authority.
This can be used to write code which works for any authority mode. For instance like this.
```csharp
public class SampleClass : NetworkBehaviour{
    
    NetworkVariable<bool> m_MyInput = new NetworkVariable<bool>(
        new NetworkVariableSettings()
        {
            ReadPermission = NetworkVariablePermission.Authority,
            WritePermission = NetworkVariablePermission.OwnerOnly
        });

    public void Update()
    {
        // Owner client gives inputs into network variable.
        if (IsOwner)
        {
            m_MyInput.Value = Input.GetKeyDown(KeyCode.Space);
        }

        // The authority which could either be the owner or the server based on the authority mode uses the input to move.
        if(HasAuthority)
        {
            if(m_MyInput.Value)
            {
                m_Rigidbody.AddForce(transform.up * m_Thrust);
            }
        }
    }
}
```
*The authority system allows NetworkVariables to be flexibly used as inputs.*

## Changing Authority

The `Authority` mode of a `NetworkObject` can be changed, on the server only, by calling `networkObject.ChangeAuthority(Authority.Server);`.

Which peer has authority also changes if the ownership of an object changes when in `owner` authority mode. This means the `HasAuthority` property should never be cached but always accessed directly.

> The reason why this is a server only operation is mainly tech debt. Changing ownership or authority change the underlying synchronization protocol. This is a non-trivial operation which affects many other systems such as RPCs or snapshots. Currently MLAPIs support for this is very limited. We have multiple race conditions in our code base when changing `Ownership` on a NetworkObject already. To support client authoritative changes to ownership or authority we will need to address our underlying issues first.

## OnGainedAuthority / OnLostAuthority

`OnGainedAuthority` and `OnLostAuthority` functions can be overridden to get informed about changes to the `HasAuthority` status. `OnGainedAuthority` gets called when `HasAuthority` switches from false to true. `OnLostAuthority` gets called when it switches to false from true
```csharp
public override void OnGainedAuthority()
{
    Debug.Log("Authority Gained :)");
}

public override void OnLostAuthority()
{
    Debug.Log("Authority Lost :(");
}
```

### SupportedAuthorityModes Attribute
Some NetworkBehaviours might not support all authority modes. This can be indicated with the SupportedAuthorityModes attribute like this:
```csharp
[SupportedAuthorityModes(Authority.Owner | Authority.Server)]
public class SampleNetworkBehaviour : NetworkBehaviour{}
```

Setting the supported authority modes will not prevent code or the inspector from assigning a wrong authority. It will just log a warning whenever the authority gets changed to a value which the behaviour does not support.

Supported authority modes is mainly a tool to signal intent on how a component should be used.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Why is Authority a mode?
Why is authority not something like ownership which you give to a client. The reason for that is because authority is a relation between multiple peers. In an authoritative model a set of peers pass inputs to the authoritative peer for processing.
> Note with client authority the peer which collects the inputs is the same peer as the one processing them.

Because we are a star topology we have some technical limitations. For instance we could not have one client generate the inputs but pass it to another client which holds authority (because this would break the encapsulation principle of time explained in the NetworkTime RFC)(also this does not really have any use cases).

Because of the above we will only support authority models where the owner of an object collects inputs and sends them to the authoritative peer which has to be either the owner or the server.

Accepting inputs from multiple peers is something we will not support with the current proposed design. The authority mode system allows for further extension to support those modes.

## Ownership vs Authority
This RFC will introduce some rules and responsibilities for each of the two systems.

IsOwner:
- Only one peer can own a NetworkObject.
- The `owner` of a NetworkObject can use `ServerRpcs` on its `NetworkBehaviour` by default.
- The `owner` is responsible for collecting inputs and redirecting them to the authority.
- If the `owner` wants to pass inputs to the `authority` this can be done by writing to NetworkVariables with `WritePermission = Owner` and `ReadPermission = Authority`.
- When the owner peer disconnects, the object it owns might get destroyed when `DontDestroyWithOwner` is set to false.

HasAuthority:
- The `authority` should run gameplay code such as physics, movement, path finding, animation etc.
- The `authority` should update NetworkVariables which get used by other clients to visualize the game state.

IsServer:
- The server has permission to change ownership and authority of NetworkObjects.
- `IsServer` should be avoided for gameplay related code. HasAuthority should be used instead.

## NetworkObject Authority

NetworkObject will get an Authority Property and a way to query for authority with `HasAuthority`:
```csharp
public Authority Authority => m_Authority;

public bool HasAuthority
{
    get
    {
        if (IsSpawned) // This is needed because else we don't know whether we are a server.
        {
            return IsOwner ? Authority == Authority.Owner : NetworkManager.IsServer && Authority == Authority.Server;
        }
        throw new SpawnStateException($"Cannot get authority for not spawned {nameof(NetworkObject)}");
    }
}
```

There will also be a function to change authority. Similar to ownership this will use the `NetworkSpawnManager` to send authority changes to all clients.
```csharp
public void ChangeAuthority(Authority authority)
{
//.....
}
```

## NetworkVariable Authority

We introduce a `NetworkVariablePermission.Authority` permission. This is a very simple change to the NetworkVariable we just add the following to the read/write queries.

This permission is needed so that authority is not just a gameplay code feature but can also be enforced onto NetworkVariables so that server authoritative games can remain authoritative

```csharp
public bool CanClientRead(ulong clientId)
{
    switch (Settings.ReadPermission)
    {
    //....
        case NetworkVariablePermission.Authority:
            return m_NetworkBehaviour.HasAuthority;
    //.....
    }
    return true;
}
```

## OnGainedAuthority / OnLostAuthority

Will be implemented by listening to authority mode changes internally and then checking whether the resulting `HasAuthority` flag is different.

These have similar usability as `OnGainedOwnership/OnLostOwnership`, in most cases they are not necessary but can be helpful.

## SupportedAuthorityModes Attribute

We will introduce a supported authority modes attribute. The main purpose for this attribute is to signal compatibility with other components.

The attribute is optional, if no attribute is added to a component all authority modes are considered to be supported. The attribute will not strongly enforce anything but just display warnings in the console and on NetworkObject if a mode is not supported.

```csharp
    [AttributeUsage(AttributeTargets.Class)]
    public class SupportedAuthorityModesAttribute : Attribute
    {
        private Authority m_AuthorityModes;

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public static bool SupportsAuthority(SupportedAuthorityModesAttribute attribute, Authority authority)
        {
            // Enum.HasFlag but faster
            return (((byte)attribute.m_AuthorityModes & (int)authority) != 0);
        }

        public SupportedAuthorityModesAttribute(Authority authorityModes)
        {
            m_AuthorityModes = authorityModes;
        }
    }
```

At runtime modes get cached in a static lookup table `Dictionary<Type, Authority>` to avoid reflection in the hot path.

Supported Authority mode just changes some minor details in NetworkObjectEditor to display components with not supported authority and after changing authority it will check whether the behaviour is still supported. In addition it checks foreach NetworkBehaviour whether the authority mode is still allowed when authority gets changed and if not logs a warning to the console.

# Drawbacks
[drawbacks]: #drawbacks

## Not deeply enough integrated
The authority system is very loosely integrated into MLAPI. It does not change how NetworkVariables and RPCs work etc. While there are many upsides to keeping authority as a separate abstraction layer it could also be a missed opportunity depending on the direction we want to take MLAPI.

With a more direct integration we could completely decouple gameplay code and network code with an abstraction layer. As a user I would no longer need to know whether I'm a client or a server, I would no longer need to learn network concepts such as RPCs or NetworkVariable because we can replace them with more simple simulation based concepts.

## NetworkObject Meta Header System

The proposed implementation adds more complexity to NetworkObject by introducing a new message type. Authority mode is a very concept to ownership both are meta data which we attach to a NetworkObject at spawn time and can be changed at runtime. 

A consolidated system for handling this kind of meta information could potentially change how we think about ownership or authority and allow us to implement the authority system with much less code changes needed.

I think it is very likely that we will need to implement such a feature soon. The recently proposed ACL system could be one solution to this. This means the authority feature in this RFC introduces tech debt because we likely will need to. Replacing the internal implementation of the authority system should be quite easy as it has no dependencies on other MLAPI features.

## Spawn Header Size
Authority increases the spawn meta header size of a `NetworkObject` by 1 bit. Increasing the spawn header size is very costly because spawning is an operation which changes the protocol. If we decide to implement spawning with a pessimistic transfer algorithm this would add 1 bit to the bandwidth used repeatedly for many ticks.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Networking frameworks with a lack of a gameplay abstraction layer such as UNet have failed many times in the past. As our competitors are moving to more gameplay and simulation focused solutions we should too. The authority system is one of many stepping stones to get us there.

We need the authority system to solve some of the core issues which UNet had. More details about this can be found in the `Motivation` section.

There are other ways in which the authority system could have been implemented. For instance a permission system where any client could have been the peer providing inputs and any other client could have been the authoritative peer would be another option. We were not able to find many real world use cases where this would be useful so that's why the authority mode was chosen. In the future we might consider storing authority internally as a set of owners instead of an authority mode but then most likely we would still want to expose authority as an authority mode for ease of use.

# Prior art
[prior-art]: #prior-art

## UNet

UNet also had the concept of [NetworkAuthority](https://docs.unity3d.com/Manual/UNetAuthority.html). There was a `Local Player Authority` checkbox on NetworkIdentity (MLAPI equivalent of NetworkObject). The UNet authority feature was very different from MLAPI's in UNet the authority was the one who was able to call commands (equivalent of server rpc). In UNet SyncVars (equivalent to NetworkVariable) where always controlled by the server, a client could never change them.

The UNet authority system has a lot of problems. The NetworkTransform component was using authority because it used RPCs. Many user made components could not support authority because they wanted to have client driven SyncVars.

This system is very limited and not really `authority` it is more what we have in MLAPI with `ownership`. It looks like the MLAPI `ownership` model has been inspired by the UNet `authority` model. The MLAPI `authority` model will improve MLAPI and allow users to do things which were not possible in UNet.

## DOTS Netcode
DOTS Netcode has a ghost mode which can be interpolated or predicted. The interpolated mode matches our 'server' mode. There is no mode for client authority. The predicted mode is something we do not support yet but can do so in the future.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Per NetworkBehaviour Authority

Per NetworkBehaviour Authority could give users more fine grained control over authority. More details to this approach can be found in the appendix section.

Adding this feature would increase the complexity of the code and it is not sure whether there are

## Naming
We might want to consider a different name for authority. Some candidates would be `AuthorityMode`, `ControlMode` or `GhostMode`(to match DOTS). Similar `HasAuthority` might be `HasControl`.

## Snapshots and Interest Systems
There are a lot of unknowns on how our snapshot and interest system will work with `NetworkVariablePermission.Authority`. We do not plan to resolve this unresolved question in the authority RFC as it only introduces an additional `NetworkVariablePermission`. The snapshot / interest RFCs will have to answer this question.

## AuthorityChanged Event
Do we need an AuthorityChanged event? We already have `OnAuthorityGained` and `OnAuthorityLost` events but would we need an additional to event to signal if the authority mode itself changed?

## Default NetworkVariable Permissions

The default write permission for a NetworkVariable is currently `NetworkVariablePermission.ServerOnly` and the default write permission is `NetworkVariablePermission.Everyone`. We could consider changing the default write permission to `NetworkVariablePermission.Authority`. It is not clear whether that would simplify or complicate the onboarding experience of MLAPI.

The problem we face with the current MLAPI is that the authority default for components such as NetworkTransform is the owner but for NetworkVariable is the server. So if we want to unify this then there will be a breaking change. We either:
- Set owner as default => NetworkVariables will behave differently when we set the default permission to NetworkVariablePermission.Authority
- Set server as default => Components like NetworkTransform which are currently owner authoritative by default will now be server authoritative by default.

## Shared/Distributed Authority

The RFC does not cover authority modes where multiple peers have authority over an object. It also does not cover prediction. The proposed authority system can be extended to support more authority modes. To support shared/distributed authority we might need an ACL entry to indicate which peers have authority over an object

# Future possibilities
[future-possibilities]: #future-possibilities

## Prediction
The authority feature is a first step towards the goal of supporting client side prediction. When using client side prediction we need precises control over what's being run on the predicted client and run the same on the server. The `HasAuthority` flag will allow us to do this by setting it to true on both the predicting client and the server.

## Commands
Commands in multiplayer games are inputs which are created by the owner/controller of an object and then sent to the authority for execution. The authority mode feature will allow us to implement commands in the future.

## RPCs

Right now RPCs can only be send to the server/clients. This means that a lot of time boilerplate must be introduced to write authoritative concepts.
```csharp
if(HasAuthority)
{
    SetPosition(m_TargetPosition);
}
else
{
    SetPositionServerRpc(m_TargetPosition);
}
```

In the future we could consider introducing an `AuthorityRpc/CommandRpc` or similar which sends an RPC to the authority. The reason why such a concept is not introduced in this RPC is mainly scope. When we look at command streams in the future both NetworkVariables and RPCs might get extended to better support them with less boilerplate. But to implement command streams we first need to agree on the concept of authority.

# Appendix (Per behaviour authority)

We could decide to also allow per behaviour authority. We decided against this approach because of the complexity involved and the lack of real world use cases. We might reconsider this in the future.

## Guide-level explanation

### NetworkBehaviour Authority

Per default `NetworkBehaviour`s use the same `Authority` mode as their `NetworkObject`. If the `NetworkObject` changes `Authority` all `NetworkBehaviour`s belonging to it will change to that's because the use the `NetworkBehaviour` specific `Authority` mode called `FromNetworkObject`.

`NetworkBehaviour Authority` is an advanced feature which can be used to change the authority of individual components. For most games this should not be necessary and mixing `authority` might lead to unexpected results.

There are two ways to change the `Authority` of a `NetworkBehaviour`

#### 1. In Editor
By default there is only a `Authority` dropdown on `NetworkObject` but not on `NetworkBehaviour`. The `AllowAuthorityOverride` attribute can be used on custom `NetworkBehaviour`s to make the dropdown appear.

#### 2. Runtime
This also need the `AllowAuthorityOverride`. Only the server can change authority by code. On the server you can call `myNetworkBehaviour.ChangeAuthority(Authority.Server);` to change authority. The change will automatically replicated to all clients.

```csharp
[AllowAuthorityOverride]
public class PlayerMovement : NetworkBehaviour
{
}
```

![dropdown NetworkBehaviour](./0015-authority/dropdown-ownership-networkbehaviour.png)

If runtime authority override is wanted but the inspector dropdown is not wanted it can be disabled in the attribute like this:
```csharp
[AllowAuthorityOverride(ShowDropdown = false)]
public class SampleNetworkBehaviour : NetworkBehaviour{}
```

The Unity high level components such as `NetworkTransform` and `NetworkAnimator` will not allow for authority override. If a user wants to use them with authority override they should derive from them and add the attribute.

## Reference-level explanation


### NetworkBehaviour Authority
By default NetworkBehaviours do not sync authority. Only NetworkBehaviours with the `AllowAuthorityOverride` will serialize their authority. This allows us to save bandwidth by accessing compile time information.


NetworkBehaviour exposes an authority override property. This property usually doesn't need to be used.
```csharp
public Authority AuthorityOverride { get; }
```

The check for `HasAuthority` is also a bit different:
```csharp
public bool HasAuthority
{
    get
    {
        if (NetworkObject.IsSpawned)
        {
            Authority authority = AuthorityOverride == Authority.FromNetworkObject ? NetworkObject.Authority : AuthorityOverride;
            return IsOwner ? authority == Authority.Owner : NetworkManager.IsServer && authority == Authority.Server;
        }
        throw new SpawnStateException($"Cannot get authority for not spawned {nameof(NetworkObject)}");
    }
}
```

Serialization works similar just uses a different wrapper function:
```csharp
internal void ChangeAuthority(NetworkBehaviour networkBehaviour, Authority authority)
{
    var networkObject = networkBehaviour.NetworkObject;
    if (networkObject.IsSpawned == false)
    {
        throw new SpawnStateException("Object is not spawned");
    }

    // convert flags to index
    var serialized = new BitField32((uint)authority).CountLeadingZeros();
    ChangeAuthority(networkObject.NetworkObjectId, networkBehaviour.NetworkBehaviourId, serialized, 2);
}
```

### AllowAuthorityOverride Attribute
Per component authority is a feature which can be useful in some rare situations. The `AllowAuthorityOverride` allows a developer to override the network authority of the network object. This is disabled by default because many components could not support with other components on the same  

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class AllowAuthorityOverrideAttribute : Attribute
{
    public bool ShowDropdown { get; set; } = true;

}
```
The `AllowAuthorityOverride` attribute serves three purposes:

1. Allows code to change authority of a network behaviour.
2. Allows users to add a per component authority dropdown but reduces UX clutter by not showing an authority dropdown on any NetworkBehaviour by default.
3. Reduce bandwidth. With per component authority we would need to synchronize the authority of each object. But with this attribute we can only sync the ones which have the attribute. 
