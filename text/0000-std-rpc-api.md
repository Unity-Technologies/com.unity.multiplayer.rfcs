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

The multiplayer framework provides 2 main network constructs ([ServerRpc](#serverrpc) and [ClientRpc](#clientrpc)) to execute logic on either server-side or client-side. This concept is often called [Remote Procedure Call (RPC)](https://en.wikipedia.org/wiki/Remote_procedure_call) and has wide adoption across the industry.

## RPC Methods

A typical framework user (Unity developer) can declare multiple RPCs under a `NetworkBehaviour` and inbound/outbound RPC calls will be replicated as a part of its replication in a network frame.

A method turned into an RPC is no longer a regular method, it will have its  own implications on direct calls and in the network pipeline (see [Execution Table](#execution-table)).

### ServerRpc

A `ServerRpc` can be invoked by a client to be executed on the server.

Developers can declare a `ServerRpc` by marking a method with `[ServerRpc]` attribute and making sure to have `ServerRpc` suffix in the method name.

```cs
[ServerRpc]
void PingServerRpc(int somenumber, string sometext) { /* ... */ }
```

Developers can invoke a `ServerRpc` by making a direct function call with parameters:

```cs
void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        PingServerRpc(Time.frameCount, "hello, world"); // Client -> Server
    }
}
```

Marking method with `[ServerRpc]` attribute and putting `ServerRpc` suffix to the method name are required, otherwise it will prompt error messages:

```cs
// Error: Invalid, missing 'ServerRpc' suffix in the method name
[ServerRpc]
void Ping(int somenumber, string sometext) { /* ... */ }

// Error: Invalid, missing [ServerRpc] attribute on the method
void PingServerRpc(int somenumber, string sometext) { /* ... */ }
```

`[ServerRpc]` attribute and matching `...ServerRpc` suffix in the method name are there to make it crystal clear for RPC call sites to know when they are executing an RPC, it will be replicated and executed on the server-side, without necessarily jumping into original RPC method declaration to find out if it was an RPC, if so whether it is a ServerRpc or ClientRpc:

```cs
Ping(somenumber, sometext); // Is this an RPC call?

PingRpc(somenumber, sometext); // Is this a ServerRpc call or ClientRpc call?

PingServerRpc(somenumber, sometext); // This is clearly a ServerRpc call
```

### ClientRpc

A `ClientRpc` can be invoked by the server to be executed on a client.

Developers can declare a `ClientRpc` by marking a method with `[ClientRpc]` attribute and making sure to have `ClientRpc` suffix in the method name.

```cs
[ClientRpc]
void PongClientRpc(int somenumber, string sometext) { /* ... */ }
```

Developers can invoke a `ClientRpc` by making a direct function call with parameters:

```cs
void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        PongClientRpc(Time.frameCount, "hello, world"); // Server -> Client
    }
}
```

Marking method with `[ClientRpc]` attribute and putting `ClientRpc` suffix to the method name are required, otherwise it will prompt error messages:

```cs
// Error: Invalid, missing 'ClientRpc' suffix in the method name
[ClientRpc]
void Pong(int somenumber, string sometext) { /* ... */ }

// Error: Invalid, missing [ClientRpc] attribute on the method
void PongClientRpc(int somenumber, string sometext) { /* ... */ }
```

`[ClientRpc]` attribute and matching `...ClientRpc` suffix in the method name are there to make it crystal clear for RPC call sites to know when they are executing an RPC, it will be replicated and executed on the client-side, without necessarily jumping into original RPC method declaration to find out if it was an RPC, if so whether it is a ServerRpc or ClientRpc:

```cs
Pong(somenumber, sometext); // Is this an RPC call?

PongRpc(somenumber, sometext); // Is this a ServerRpc call or ClientRpc call?

PongClientRpc(somenumber, sometext); // This is clearly a ClientRpc call
```

### Reliability

RPCs are reliable by default which means they are guaranteed to be executed on the remote side. However, sometimes developers might want to opt-out reliability, which is often the case for non-critical events such as particle effects, sounds effects etc.

Reliability configuration can be specified for both `ServerRpc` and `ClientRpc` methods at compile-time:

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

Reliable RPCs will be received on the remote end in the same order as they are fired but this in-order guarantee only applies to RPCs on the same `NetworkObject`. Different `NetworkObject`s might have reliable RPCs called but executed in different order compared to each other. To put more simply, in-order reliable RPC execution is guaranteed per `NetworkObject` basis only.

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

Both `ServerRpc` and `ClientRpc` methods can be configured either by `[ServerRpc]`/`[ClientRpc]` attributes at compile-time and/or `ServerRpcParams`/`ClientRpcParams` at runtime.

Developers can put `ServerRpcParams`/`ClientRpcParams` as the last parameter (optionally) and they could be used for a consolidated space for `XXXRpcReceiveParams` and `XXXRpcSendParams`.

The network framework will inject `XXXRpcReceiveParams` when invoked by network receive handling (framework code) and will consume `XXXRpcSendParams` when invoked by RPC send call (user code).

### ServerRpc Params

```cs
struct ServerRpcSendParams
{
}

struct ServerRpcReceiveParams
{
    // who sent the RPC?
    ulong SenderClientId;
}

struct ServerRpcParams
{
    ServerRpcSendParams Send;
    ServerRpcReceiveParams Receive;
}


// Both ServerRPC methods below are fine, `ServerRpcParams` is completely optional

[ServerRpc]
void AbcdServerRpc(int somenumber) { /* ... */ }

[ServerRpc]
void XyzwServerRpc(int somenumber, ServerRpcParams rpcParams = default) { /* ... */ }
```

### ClientRpc Params

```cs
struct ClientRpcSendParams
{
    // who are the target clients?
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


// Both ClientRPC methods below are fine, `ClientRpcParams` is completely optional

[ClientRpc]
void AbcdClientRpc(int framekey) { /* ... */ }

[ClientRpc]
void XyzwClientRpc(int framekey, ClientRpcParams rpcParams = default) { /* ... */ }
```

## Serialization

Instances of [Serializable Types (RFC &rarr;)](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/2) passed into an RPC as parameters will be serialized and replicated to the remote side.

## Backward-Compatibility

This API change is not backward compatible but future iterations over this API will be backward compatible most of the time as we will be using existing `XXXRpcSendParams` and `XXXRpcReceiveParams` for future additions. One of the main goals with this RPC API is also making it as future-proof as possible and prevent from continuous API breaking changes.

Framework registers RPC methods statically by their deterministic [method signature](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/methods#method-signatures) hash which means as long as method signature stays the same, backward-compatibility will be possible but if method signature changes, same method with the same name under same namespace and type will not be compatible with its predecessor.

## Cross-Compatibility

In this section, we will be focusing on the Standard RPC API's cross-compatibility only, not the framework as a whole.

A method marked as RPC will be statically registered with its assembly-scoped method signature hash.

A typical assembly-scoped method signature sample:

```
Game.dll / System.Void Shooter::PingServerRpc(System.Int32,MLAPI.Messaging.ServerRpcParams)
```

- `Game.dll` &rarr; Assembly
- ` / ` &rarr; Separator
- `System.Void Shooter::PingServerRpc(System.Int32,MLAPI.Messaging.ServerRpcParams)` &rarr; Method signature
  - `System.Void` &rarr; Return type
  - `Shooter` &rarr; Enclosing type
  - `::` &rarr; Scope resolution operator
  - `PingServerRpc` &rarr; Method name
  - `(System.Int32,MLAPI.Messaging.ServerRpcParams)` &rarr; Params with types (no param names)

An RPC signature will be turned into 32-bit integer using [xxHash](http://xxhash.com) (XXH32) non-cryptographic hash algorithm.

As expected, RPC signature therefore its hash will be changed if assembly, return type, enclosing type, method name and/or any method param type changes (but names of method parameters can be changed as they are not a part of the method signature).

A change in the RPC signature will lead into different send/receive codepath with different serialization code and execute a different method body. Previous version of the RPC method will not be executed by the new RPC method with the new signature.

### Cross-Build Compatibility ✅

As long as RPC method signature kept the same, it will be compatible between different builds.

### Cross-Version Compatibility ✅

As long as RPC method signature kept the same, it will be compatible between different versions.

### Cross-Project Compatibility ❌

Since project name or any project-specific token is not being a part of RPC signature, it is possible to have the exact same RPC method signature defined in different builds and they are not necessarily going to be compatible with each other.

## Deprecation of Return Values

MLAPI supports RPC return values on convenience RPCs.

Example:

```cs
public IEnumerator MyRpcCoroutine()
{
    RpcResponse<float> response = InvokeServerRpc(MyRpcWithReturnValue, Random.Range(0f, 100f), Random.Range(0f, 100f));

    while (!response.IsDone)
    {
        yield return null;
    }

    Debug.LogFormat("The final result was {0}!", response.Value);
}

[ServerRPC]
public float MyRpcWithReturnValue(float x, float y)
{
    return x * y;
}
```

This RFC also drops this feature and similar functionality can be achived as following:

```cs
void MyRpcInvoker()
{
    MyRpcWithReturnValueRequestServerRpc(Random.Range(0f, 100f)), Random.Range(0f, 100f)));
}

[ServerRpc]
void MyRpcWithReturnValueRequestServerRpc(float x, float y)
{
    MyRpcWithReturnValueResponseClientRpc(x * y);
}

[ClientRpc]
void MyRpcWithReturnValueResponseClientRpc(float result)
{
    Debug.LogFormat("The final result was {0}!", result);
}
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
