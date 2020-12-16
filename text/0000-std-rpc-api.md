- Feature Name: `std-rpc-api`
- Start Date: 2020-11-02
- RFC PR: [RFC#1](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/1)
- Issue: [MLAPI#406](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/issues/406)

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

An RPC call made without an active connection will be dropped and will not be queued for send automatically. Both reliable and unreliable RPC calls have to be made when there is an active network connection established between a client and the server. Also reliable RPC calls made during connection will be dropped on disconnect as well.

### Execution Table

||Server|Client|Host (Server+Client)|
|-:|:-:|:-:|:-:|
|ServerRpc Send|❌|✅|✅|
|ServerRpc Execute|✅|❌|✅|
|ClientRpc Send|✅|❌|✅|
|ClientRpc Execute|❌|✅|✅|

An RPC function **never** executes its body immediately since the function call really is a stand-in for a network transmission. Even a `ServerRpc` called by a host (an instance that is a client and the server at the same time, aka listen-server) will not be executed immediately but instead, follow the regular network frame staging first and queued-up to be executed locally in the next network frame.

Structure of a typical ServerRpc:

```cs
[ServerRpc]
void MyServerRpc(int somenumber, string somestring)
{
    // Network Send Block (framework-code)
    // Network Return Block (framework-code)
    // RPC Method Body (user-code)
}
```

Pseudo-code sample of a ServerRpc:

```cs
[ServerRpc]
void MyServerRpc(int somenumber, string somestring)
{
    // --- begin: injected framework-code
    if (NetworkSend())
    {
        // this block will be executed if:
        //   - called from user-code on client
        //   - called from user-code on host

        var writer = NetworkCreateWriter();
        writer.WriteInt32(1234567890); // RPC method signature hash
        writer.WriteInt32(somenumber);
        writer.WriteChars(somestring);
        NetworkSendRpc(writer);
    }

    if (NetworkReturn())
    {
        // this block will be executed if:
        //   - called from user-code
        //   - called from framework-code on client

        return;
    }
    // --- end: injected framework-code

    print($"MyServerRpc: {somenumber}");
}
```

## RPC Params

Both `ServerRpc` and `ClientRpc` methods can be configured either by `[ServerRpc]`/`[ClientRpc]` attributes at compile-time and/or `ServerRpcParams`/`ClientRpcParams` at runtime.

Developers can put `ServerRpcParams`/`ClientRpcParams` as the last parameter (optionally) and they could be used for a consolidated space for `XXXRpcReceiveParams` and `XXXRpcSendParams`.

The network framework will inject the corresponding `XXXRpcReceiveParams` to what the user had declared in code when invoked by network receive handling (framework code) and will consume `XXXRpcSendParams` when invoked by RPC send call (user code).

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

An RPC signature will be turned into a 32-bit integer using [xxHash](http://xxhash.com) (XXH32) non-cryptographic hash algorithm.

As expected, RPC signature therefore its hash will be changed if assembly, return type, enclosing type, method name and/or any method param type changes (but names of method parameters can be changed as they are not a part of the method signature).

A change in the RPC signature will lead into a different send/receive codepath with different serialization code and execute a different method body. Previous versions of the RPC method will not be executed by the new RPC method with the new signature.

### Cross-Build Compatibility ✅

As long as the RPC method signature is kept the same, it will be compatible between different builds.

### Cross-Version Compatibility ✅

As long as the RPC method signature is kept the same, it will be compatible between different versions.

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

In short, developers will see error and warning messages in the Unity Editor in the Console window if there was a problem with RPC method definition, parameters types on RPC method signature etc.

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

**One important note to plug here is that, [Unity Editor Diagnostics](#unity-editor-diagnostics) are not optional, they are essential and fundamentally checking against invalid RPC code and prevent builds while also providing contextual error/warning messages.**

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Since we already covered lots of high-level details above in the [Guide-level explanation](#guide-level-explanation) section, we'll cover just a few concepts which are a little bit subtle when it comes down to implementing this RFC.

## RPC vs non-RPC methods

When an RPC method is invoked via direct method call, it'll be making 2 main decisions:

1. Should it replicate the RPC call with parameters?
2. Should it execute the RPC body or return early?

These check blocks will be injected into the method body right before every other instructions in the body. You can have a quick look at [Execution Table](#execution-table) section to have a rough idea about execution flow.

A non-RPC method would just execute the body without having anything before its first instruction.

## RPC is just a middleware

Currently, an RPC is a part of its enclosing `NetworkBehaviour` and it'll be replicated as a part of it.

An RPC call would not begin/end network frame and/or begin/end network buffer. RPC will be asking its `NetworkBehaviour` to give it something to write into then write into it (e.g. `BeginSendServerRpc` &rarr; RPC write &rarr; `EndSendServerRpc`) which means it has its own RPC method signature hash written but no other header or footer before/after network buffer. Managing network buffer, stream, writer etc. is entirely up to `NetworkBehaviour` itself and implementation should comply with this approach to give full control and flexibility over to `NetworkBehaviour`. In fact, that would allow us to queue, batch, filter (...) RPCs on the `NetworkBehaviour` side when an RPC invocation happens.

Basically, RPC is just a middleware that wants to write into its `NetworkBehaviour` network buffer when invoked.

## Managing ILPP overhead (today)

ILPP is not free in the Unity Editor, it will increase compile and build time for Unity users, therefore we need to implement ILPP as minimal and clever as possible to reduce ILPP processing time overhead.

Previously, UNET tried to replace all RPC call-sites via ILPP which meant that it had to iterate over every single method to check potential RPC call sites, traversing the entire assembly from top to bottom. That's not an optimal and clever way of doing it, also it doesn't scale well. 1000 RPCs meant 1000 iterations over the entire assembly, which as you might expect, doesn't scale up well, it scales exponentially.

Current implementation of this RFC however, finds RPC methods via reflection metadata by searching for methods with either `[ServerRpc]` or `[ClientRpc]` attributes, then inject extra ILPP-gen'd blocks into the RPC method body itself without touching call sites at all. This approach scales linearly and better compared to UNET's approach.

One other crucial topic to mention here is that we do not IL-gen brand-new stub methods for RPCs because we still want people to be able to have a fairly standard debugging experience. We also rewrite PDBs too which makes debugger to not break, developers are going to be able to put breakpoints in the method body, see local variables etc., we are not changing method call context at all.

## RPCs will be registered statically

There is no API or support for dynamic RPC registration being proposed with this RFC (that could be a separate RFC on itself if wanted and justified). Currently, all RPC methods will be registered on their enclosing `NetworkBehaviour` derived types' static constructor.

## RPC signatures (and hashes) must be stable

We do not want to break cross-version, cross-build compatibility so that we are proposing to have deterministic and stable RPC method signatures and hashes (see [Cross-Compatibility](#cross-compatibility) section for more).

This will even also allow deeper debugging options where you could match RPC signature hashes between different builds, network packets etc. and/or use framework's RPC signature hashing code to reverse-engineer unknown RPC IDs by brute-forcing based on RPC method's previous revisions.

In simple terms, let's try not to break consistency, determinism and debuggability if we can.

# Drawbacks
[drawbacks]: #drawbacks

Implementation of this feature will be relying on IL Post-Processing which will increase compile and build time in Unity Editor. However, we will explore replacing the IL components if and when [Roslyn Source Generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/) support arrives in Unity Editor.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
  - Compared to MLAPI's existing templated delegates approach, this RFC proposes much a more performant and cleaner API exposed to framework users. This API is also more familiar to [UNET RPC API](https://docs.unity3d.com/Manual/UNetActions.html) which would make existing UNET users' life easier while onboarding.
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

Unreal provides a way to check RPC being executed before its execution, and it disconnects the caller immediately if validation fails. This approach might be useful to catch very basic cheats and packet attacks but might also be perceived as a very strong punishment. However, it could still give us an idea and impression to discover ways to implement validation checks programmatically.

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

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - Multi-project (cross-project) setup support
    - Sharing a common library between projects?
    - Using weakly-types `string` names to invoke RPCs?
    - Implementing CrossProject RPC component just for this purpose?
  - Weakening `...ServerRpc`/`...ClientRpc` suffix enforcement
    - Should they really be hard compile errors or just soft warnings?
    - Should we allow suffix syntax configuration?
    - Should we allow suffix, prefix or none configuration?
  - Instantiation of complex types on RPC receive
    - Should we allow framework users to pool instances? (object-pool)
    - Should we allow non-default constructible types? (class-factory)

# Future possibilities
[future-possibilities]: #future-possibilities

- We might want to have RPCs independent from `NetworkBehaviour`s (free-standing RPC functions/events)
- We might follow-up on [Unresolved questions](#unresolved-questions) to address with further implementations
- We will probably follow-up with more RPC Send/Recv params fields (AOI filtering etc.)
- With more and more [Serializable Types](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/2), RPC API will be more an more convenient and pleasing to use
- RPC batching, queuing, filtering and other framework internals would be implemented without API-breaking changes
