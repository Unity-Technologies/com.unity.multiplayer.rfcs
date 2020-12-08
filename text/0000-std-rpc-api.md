- Feature Name: `std-rpc-api`
- Start Date: 2020-11-02
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

Existing MLAPI [**Convenience** and **Performance** RPCs](https://mlapi.network/wiki/messaging-system/) are sub-optimal in terms of required boilerplate code, API design and runtime performance.

This RFC proposes a new **Standard** RPC API that excels at minimal boilerplate, API design/clarity, extensibility (future-proofing) and runtime performance.

The initial implementation of this standard would be kept minimal and further-extended with additional features down the line as it will also support future extensions and enhancements without changing the standard API necessarily.

# Motivation
[motivation]: #motivation

Achieve better performance, unify & standardize RPC API, make RPC API extensible & future-proof and reduce boilerplate code.

Also a simple internal performance benchmark came with these results:

    Local RPC Send Performance Test (524280 calls)
    Convenience -> 4361ms
    Performance -> 2607ms
    Standard -> 2205ms
    Local RPC Recv Performance Test (524280 calls)
    Convenience -> 9626ms
    Performance -> 5923ms
    Standard -> 1935ms

We're looking at **up to 5.24x better performance** (compared to convenience API vs standard API receive performance).

Beyond that, on an internal project, we saw roughly **%80 less boilerplate code** compared to performance RPCs and standard RPCs which is also a huge win.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Multiplayer framework provides 2 main network constructs ([ServerRpc](#serverrpc) and [ClientRpc](#clientrpc)) to execute logic on either server-side or client-side. This concept often called as [Remote Procedure Call (RPC)](https://en.wikipedia.org/wiki/Remote_procedure_call) and has wide adoption across the industry.

## RPC Methods

A framework user (Unity developer) can declare multiple RPCs under a `NetworkBehaviour` and inbound/outbound RPC calls will be replicated as a part of its replication in a network frame.

A method turned into an RPC is no longer a regular method, it will have its  own implications on direct calls and in the network pipeline (see [Execution Table](#execution-table)).

### ServerRpc

A `ServerRpc` can be invoked by a client to be executed on the server.

Developer can declare a `ServerRpc` by marking a method with `[ServerRpc]` attribute and making sure to have `ServerRpc` suffix in the method name.

```cs
[ServerRpc]
void PingServerRpc(int framekey) { /* ... */ }
```

Developer can invoke a `ServerRpc` by making a direct function call with parameters:

```cs
void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        PingServerRpc(Time.frameCount); // Client -> Server
    }
}
```

Marking method with `[ServerRpc]` and putting `ServerRpc` suffix to the method name are required, otherwise it will prompt error messages:

```cs
// Error: Invalid, missing 'ServerRpc' suffix in the method name
[ServerRpc]
void Ping(int framekey) { /* ... */ }

// Error: Invalid, missing [ServerRpc] attribute on the method
void PingServerRpc(int framekey) { /* ... */ }
```

`[ServerRpc]` attribute and matching `...ServerRpc` suffix in the method name are there to make it crystal clear for RPC call sites to know when they are executing an RPC, it will be replicated and executed on the server-side, without necessarily jumping into original RPC method declaration to find out if it was an RPC, if so whether it is a ServerRpc or ClientRpc:

```cs
Ping(framekey); // Is this an RPC call?

PingRpc(framekey); // Is this a ServerRpc call or ClientRpc call?

PingServerRpc(framekey); // This is clearly a ServerRpc call
```

### ClientRpc

A `ClientRpc` can be invoked by the server to be executed on a client.

Developer can declare a `ClientRpc` by marking a method with `[ClientRpc]` attribute and making sure to have `ClientRpc` suffix in the method name.

```cs
[ClientRpc]
void PongClientRpc(int framekey) { /* ... */ }
```

Developer can invoke a `ClientRpc` by making a direct function call with parameters:

```cs
void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        PongClientRpc(Time.frameCount); // Server -> Client
    }
}
```

Marking method with `[ClientRpc]` and putting `ClientRpc` suffix to the method name are required, otherwise it will prompt error messages:

```cs
// Error: Invalid, missing 'ClientRpc' suffix in the method name
[ClientRpc]
void Pong(int framekey) { /* ... */ }

// Error: Invalid, missing [ClientRpc] attribute on the method
void PongClientRpc(int framekey) { /* ... */ }
```

`[ClientRpc]` attribute and matching `...ClientRpc` suffix in the method name are there to make it crystal clear for RPC call sites to know when they are executing an RPC, it will be replicated and executed on the client-side, without necessarily jumping into original RPC method declaration to find out if it was an RPC, if so whether it is a ServerRpc or ClientRpc:

```cs
Pong(framekey); // Is this an RPC call?

PongRpc(framekey); // Is this a ServerRpc call or ClientRpc call?

PongClientRpc(framekey); // This is clearly a ClientRpc call
```

### Reliability

RPCs are reliable by default which means they are guaranteed to be executed on the remote side. However, sometimes developer might want to opt-out reliability, which is often the case for non-critical events such as particle effects, sounds effects etc.

Reliability configuration can be specified for both `ServerRpc` and `ClientRpc` methods at compile time:

```cs
[ServerRpc]
void MyReliableServerRpc() { /* ... */ }

[ServerRpc(IsReliable = false)]
void MyUnreliableServerRpc() { /* ... */ }

[ClientRpc]
void MyReliableClientRpc() { /* ... */ }

[ClientRpc(IsReliable = false)]
void MyUnreliableClientRpc() { /* ... */ }
```

An RPC call made without active connection will be dropped and will not be queued for send automatically. Both reliable and unreliable RPC calls have to be made when there is an active network connection established between a client and the server. Also reliable RPC calls made during connection will be dropped on disconnect as well.

### Execution Table

An RPC function **never** executes its body immediately since it's being a network construct. Even a `ServerRpc` called by a host (an instance that is a client and the server at the same time, aka listen server) will not be executed immediately but follow the regular network frame staging first.

||Server|Client|Host (Server+Client)|
|-:|:-:|:-:|:-:|
|ServerRpc Network Send|❌|✅|✅|
|ServerRpc Network Call|✅|❌|✅|
|ServerRpc Direct Call|❌|❌|❌|
|ClientRpc Network Send|✅|❌|✅|
|ClientRpc Network Call|❌|✅|✅|
|ClientRpc Direct Call|❌|❌|❌|

## RPC Params

### ServerRpc Params

```cs
struct ServerRpcSendParams
{
}

struct ServerRpcReceiveParams
{
    ulong SenderId;
}

struct ServerRpcParams
{
    ServerRpcSendParams Send;
    ServerRpcReceiveParams Receive;
}

[ServerRpc]
void AbcdServerRpc(int framekey) { /* ... */ }

[ServerRpc]
void XyzwServerRpc(int framekey, ServerRpcParams rpcParams = default) { /* ... */ }
```

### ClientRpc Params

```cs
struct ClientRpcSendParams
{
    ulong[] TargetClientIds;
}

struct ClientRpcReceiveParams
{
}

struct ClientRpcParams
{
    ClientRpcSendParams Send;
    ClientRpcReceiveParams Receive;
}

[ClientRpc]
void AbcdClientRpc(int framekey) { /* ... */ }

[ClientRpc]
void XyzwClientRpc(int framekey, ClientRpcParams rpcParams = default) { /* ... */ }
```

## Serialization

Instances of [Serializable Types](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/2) passed into an RPC as parameters will be serialized and replicated to the remote side.

A quick and simple example:

```cs
[ClientRpc]
void WelcomeClientRpc(string motd, Vector3 spawnPoint, ClientRpcOptions rpcOptions = default) { /* ... */ }

// Server -> Owner Client
WelcomeClientRpc("Greetings!", Vector3.zero, new ClientRpcOptions {TargetClientIds = new[] {OwnerClientId}});
```

## Tooling

Tooling is one of the most important parts in overall developer experience, especially when we as framework developers are introducing a brand-new network construct, in this case RPCs.

### Unity Editor Diagnostics

In short, developers will see error and warning messages in the Unity Editor in Console window if there was a problem with RPC method definition, parameters types on RPC method signature etc.

Some of those messages will be warning messages which will not prevent compile and/or build.

Some of those messages will be error messages which will prevent compile and/or build.

Descriptive messages, message types, when messages should appear and other details are very much dependent on the implementation of this RPC API specification, therefore they will be decided while implementing the feature.

### Roslyn Unity Analyzers

While [Unity Editor Diagnostics](#unity-editor-diagnostics) messages will be covering in-Unity-Editor experience, Roslyn Analyzers will cover in-IDE experience.

[Roslyn Analyzers](https://devblogs.microsoft.com/dotnet/write-better-code-faster-with-roslyn-analyzers/) will help developers to catch issues with their RPC netcode within the IDE without jumping back to Unity Editor, which we believe will improve overall developer experience. Roslyn Analyzers is a compiler feature, Roslyn compiler feature in fact, which will analyze written code and feed back to IDE so that we could see both command-line errors/warnings and in-IDE hints/popups.

There is already an [**Analyzers for Unity** repository owned and maintained by Microsoft](https://github.com/microsoft/Microsoft.Unity.Analyzers) that provides some insights for Unity developers &mdash; we could be influenced by this repository and create a Unity owned one or further extend existing Microsoft owned project by actively collaborating with them.

### Rider Unity Plugin

Beyond [Unity Editor Diagnostics](#unity-editor-diagnostics) and [Roslyn Analyzers](#roslyn-unity-analyzers), we could also strengthen our tight Rider IDE integration by either contributing to existing [repository owned and maintained by JetBrains](https://github.com/JetBrains/resharper-unity) or create a Unity owned one.

Plugin could recognize RPC API and provide additional hints, checks, visuals and other features to enhance developer experience.

### Strategy

We will be committed to deliver [Unity Editor Diagnostics](#unity-editor-diagnostics) as of our main goal, and could further extend to other tooling opportunities by our official commitment and/or community engagements.

In an ideal world, one might have Unity Editor + Roslyn Compiler + Rider IDE tools that gives in-Editor and in-IDE feedback and better visuals &mdash; hopefully that'd make everybody's life much easier.

**One important note to plug here is that, [Unity Editor Diagnostics](#unity-editor-diagnostics) are not optional, they are essential and fundamentally checking against invalid RPC code and prevent builds while also providing contextful error/warning messages.**

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Implementation of this feature will be relying on IL Post-Processing which will increase compile and build time in Unity Editor. However, we will explore replacing the IL components if and when [Roslyn Source Generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/) support arrives in Unity Editor.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
  - Compared to MLAPI's existing templated delegates approach, this RFC proposes much more performant and cleaner API exposed to framework users. This API is also more familiar to [UNET RPC API](https://docs.unity3d.com/Manual/UNetActions.html) which would make existing UNET users' life easier while onboarding.
- What other designs have been considered and what is the rationale for not choosing them?
  - There was an ongoing discussion around whether or not to enforce `...ServerRpc` and `...ClientRpc` suffixes on method names tied to `[ServerRpc]` and `[ClientRpc]` attributes but what we are advising here is to also consider RPC call sites and make it obvious even in the call sites that the RPC method is in fact an RPC method and it's either `ClientRpc` or `ServerRpc` (see [ServerRpc](#serverrpc) and/or [ClientRpc](#clientrpc) sections for further details and examples). This approach also realized by [UNET Remote Actions](https://docs.unity3d.com/Manual/UNetActions.html) in the past but had its own issues with naming (`Command`, `ClientRpc`, `TargetRpc` names are confusing and ambiguous). We could weaken this enforcement by case-insensitive naming so that both `MyServerRpc` and `MyServerRPC` would be OK (open discussion). However, both call-site clarity and existing UNET API gave us more confidence towards this approach at the end.
- What is the impact of not doing this?
  - This is an API breaking change. If we were to make this change later, it would be harder to rollout in public. It would require API upgrader, cause some frustrations and issues in customers' codebases.
  - If we **never** do this change, we're leaving quite a lot of potential goodies on the table (see [Summary](#summary) and [Motivation](#motivation) sections).

# Prior art
[prior-art]: #prior-art

## Unreal RPCs

Unreal Engine offers similar RPC functionality on Actors ([documentation](https://docs.unrealengine.com/en-US/InteractiveExperiences/Networking/Actors/RPCs/index.html)).

```cpp
UFUNCTION(Client)
void ClientRPCFunction();

UFUNCTION(Server)
void ServerRPCFunction();

UFUNCTION(NetMulticast)
void MulticastRPCFunction();
```

### Validation Before Execution

Unreal provides a way to check RPC being executed before its execution, and it disconnects the caller immediately if validation fails. This approach might be useful to catch very basic cheats and packet attacks but might also perceived as a very strong punishment. However, it could still give us an idea and impression to discover ways to implement validation checks programmatically.

```cpp
// Header

UFUNCTION(Server, WithValidation)
void SomeRPCFunction(int32 AddHealth);


// Source

bool SomeRPCFunction_Validate(int32 AddHealth)
{
    if (AddHealth > MAX_ADD_HEALTH)
    {
        // This will disconnect the caller
        return false;
    }

    // This will allow the RPC to be called
    return true;
}

void SomeRPCFunction_Implementation(int32 AddHealth)
{
    Health += AddHealth;
}
```

## Photon RPCs

Photon PUN offers similar RPC functionality ([documentation](https://doc.photonengine.com/en-us/pun/v2/gameplay/rpcsandraiseevent)).

```cs
[PunRPC]
void ChatMessage(string a, string b)
{
    Debug.Log(string.Format("ChatMessage {0} {1}", a, b));
}
```

### MessageInfo as a Parameter

Photon provides `PhotonMessageInfo` over RPCs to give more context in the executing RPC body. This could be very useful for [`ServerRpc`](#serverrpc)s when multiple clients want to execute the same RPC on the server. Server &rarr; Clients is clear, server executes the RPC on the client but in the Clients &rarr; Server scenario, ServerRpc body can't tell who was the instigator. There is still a chance to provide RPC context information via a protected member on `NetworkBehaviour` which gets set/unset before/after RPC execution but direct access to that information on the method callstack would be better approach (IMHO). We could discover similar approach and repurpose our existing magical last parameter `XXXRPCOption` _(what if we abstract that magical last parameter to be either RPC option or RPC info based on the execution context?)_.

```cs
[PunRPC]
void ChatMessage(string a, string b, PhotonMessageInfo info)
{
    // the photonView.RPC() call is the same as without the info parameter.
    // the info.Sender is the player who called the RPC.
    Debug.LogFormat("Info: {0} {1} {2}", info.Sender, info.photonView, info.SentServerTime);
}
```

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
