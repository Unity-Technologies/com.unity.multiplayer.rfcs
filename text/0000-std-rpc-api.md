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

Achieve better performance, unify & standardize RPC API, make RPC API extensible & future-proof and reduce boilerplate.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Multiplayer framework provides 2 main network constructs ([ServerRPC](#serverrpc) and [ClientRPC](#clientrpc)) to execute logic on either server-side or client-side. This concept often called as [Remote Procedure Call (RPC)](https://en.wikipedia.org/wiki/Remote_procedure_call) and has wide adoption across the industry.

## RPC Methods

A framework user (Unity developer) can declare multiple RPCs under a `NetworkBehaviour` and inbound/outbound RPC calls will be replicated as a part of its replication in a network frame.

### ServerRPC

A `ServerRPC` can be invoked by a client to be executed on the server.

Developer can declare a `ServerRPC` by marking a method with `[ServerRPC]` attribute and making sure to have `ServerRPC` postfix in the method name:

```cs
[ServerRPC]
void PingServerRPC(int framekey) { /* ... */ }
```

Developer can invoke a `ServerRPC` by making a direct function call with parameters:

```cs
void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        PingServerRPC(Time.frameCount); // Client -> Server
    }
}
```

Marking method with `[ServerRPC]` and putting `ServerRPC` postfix to the method name are required:

```cs
// Invalid, missing 'ServerRPC' postfix in the method name
[ServerRPC]
void Ping(int framekey) { /* ... */ }

// Invalid, missing [ServerRPC] attribute
void PingServerRPC(int framekey) { /* ... */ }
```

`[ServerRPC]` attribute and matching `...ServerRPC` postfix in the method name are there to make it crystal clear for RPC call sites to know when they are executing an RPC, it will be replicated and executed on the server-side, without necessarily jumping into original RPC method declaration to find out if it was an RPC, if so whether it is a ServerRPC or ClientRPC:

```cs
Ping(framekey); // Is this an RPC call?

PingRPC(framekey); // Is this a ServerRPC call or ClientRPC call?

PingServerRPC(framekey); // This is clearly a ServerRPC call
```

### ClientRPC

A `ClientRPC` can be invoked by the server to be executed on a client.

Developer can declare a `ClientRPC` by marking a method with `[ClientRPC]` attribute and making sure to have `CLientRPC` postfix in the method name:

```cs
[ClientRPC]
void PongClientRPC(int framekey) { /* ... */ }
```

Developer can invoke a `ClientRPC` by making a direct function call with parameters:

```cs
void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        PongClientRPC(Time.frameCount); // Server -> Client
    }
}
```

Marking method with `[ClientRPC]` and putting `ClientRPC` postfix to the method name are required:

```cs
// Invalid, missing 'ClientRPC' postfix in the method name
[ClientRPC]
void Pong(int framekey) { /* ... */ }

// Invalid, missing [ClientRPC] attribute
void PongClientRPC(int framekey) { /* ... */ }
```

`[ClientRPC]` attribute and matching `...ClientRPC` postfix in the method name are there to make it crystal clear for RPC call sites to know when they are executing an RPC, it will be replicated and executed on the client-side, without necessarily jumping into original RPC method declaration to find out if it was an RPC, if so whether it is a ServerRPC or ClientRPC:

```cs
Pong(framekey); // Is this an RPC call?

PongRPC(framekey); // Is this a ServerRPC call or ClientRPC call?

PongClientRPC(framekey); // This is clearly a ClientRPC call
```

### Execution Table

An RPC function **never** executes its body immediately since it's being a network construct. Even a `ServerRPC` called by a host (an instance that is a client and the server at the same time, aka listen server) will not be executed immediately but follow the regular network frame staging first.

||Server|Client|Host (Server+Client)|
|-:|:-:|:-:|:-:|
|ServerRPC Network Send|❌|✅|✅|
|ServerRPC Network Call|✅|❌|✅|
|ServerRPC Direct Call|❌|❌|❌|
|ClientRPC Network Send|✅|❌|✅|
|ClientRPC Network Call|❌|✅|✅|
|ClientRPC Direct Call|❌|❌|❌|

## RPC Options

Sometimes developer might want to control RPC's network execution (such as targeting specific subset of clients) and that is why we expose `ClientRPCOptions` and `ServerRPCOptions` structs to give better control over RPC network execution. RPC options will be specified per call basis at runtime (optionally) without touching RPC method signature so that we as framework developers could further extend RPC options in the future without touching the Standard RPC API necessarily which makes extensibility and future-proofing possible, also relatively easier.

### ClientRPCOptions

`ClientRPCOptions` can be put as the last parameter of a `ClientRPC` method signature to gain access to `ClientRPC` network executions options at runtime:

```cs
[ClientRPC]
void MyClientRPC(int framekey, ClientRPCOptions rpcOptions = default) { /* ... */ }

// Server -> Client #123, Client #456, Client #789
var framekey = Time.frameCount;
var targetClientIds = new ulong[] {123, 456, 789};
var clientRPCOptions = new ClientRPCOptions{TargetClientIds =  targetClientIds};
MyClientRPC(framekey, clientRPCOptions);

// Server -> Owner Client
MyClientRPC(Time.frameCount, new ClientRPCOptions {TargetClientIds = new[] {OwnerClientId}});
```

### ServerRPCOptions

`ServerRPCOptions` can be put as the last parameter of a `ServerRPC` method signature to gain access to `ServerRPC` network executions options at runtime:

```cs
[ServerRPC]
void MyServerRPC(int framekey, ServerRPCOptions rpcOptions = default) { /* ... */ }
```

## Serialization

Multiplayer framework has built-in serialization support for C# and Unity primitive types out-of-the-box, also with ability to further extend network serialization for user defined types implementing `INetworkSerializable` interface.

### C# Primitives

Arguments with `bool`, `char`, `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `string` types passed into an RPC method for its parameters will be serialized by built-in serialization code:

```cs
[ServerRPC]
void MyServerRPC(
    bool b1, char c8,
    sbyte i8, byte u8,
    short i16, ushort u16,
    int i32, uint u32,
    long i64, ulong u64,
    float s32, double d64,
    string str)
{
    // ...
}


MyServerRPC(
    true, 'N',
    127, 255,
    32767, 65535,
    2147483647, 4294967295,
    9223372036854775807, 18446744073709551615,
    123456.789f, 987654321.0,
    "netcode");
```

### Unity Primitives

Arguments with `Color`, `Color32`, `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Ray`, `Ray2D` types passed into an RPC method for its parameters will be serialized by built-in serialization code:

```cs
[ClientRPC]
void MyClientRPC(
    Color col, Color32 col32,
    Vector2 vec2, Vector3 vec3, Vector4 vec4,
    Quaternion quat,
    Ray ray, Ray2D ray2d)
{
    // ...
}


MyClientRPC(
    new Color(0.3f, 0.4f, 0.6f, 0.3f), new Color32(64, 128, 192, 255),
    Vector2.zero, Vector3.zero, Vector4.zero,
    Quaternion.identity,
    new Ray(transform.position, transform.forward), new Ray2D(transform.position, transform.forward));
```

### Static Arrays and Generic Collections

Static arrays and generic collections like `IEnumerable<T>`s and `IEnumerable<KeyValuePair<K, V>>`s will be serialized by built-in serialization code if their underlying type is either serialization supported types (e.g. `Vector3`) or if they implement `INetworkSerializable` interface.

For example, static array of `Vector3` is supported:

```cs
[ClientRPC]
void MyClientRPC(Vector3[] startPositions) { /* ... */ }
```

However, a static array of non-supported and non-`INetworkSerializable` types are not OK:

```cs
[ClientRPC]
void MyClientRPC(Material[] skinMaterials) { /* ... */ }

// Not OK! `Material` type has no built-in network serialization code and does not implement `INetworkSerializable`
```

Beyond that, `IEnumerable<T>` and `IEnumerable<KeyValuePair<K, V>>` will be supported as long as underlying `T`, `K`, and `V` types are supported:

```cs
[ClientRPC]
void MyClientRPC(IEnumerable<Vector3> startPositions) { /* ... */ }


var startPositions = new List<Vector3>();
// ...
MyClientRPC(startPositions);
```

```cs
[ClientRPC]
void MyClientRPC(IEnumerable<KeyValuePair<ulong, Vector3>> startPositionMap) { /* ... */ }


var startPositionMap = new Dictionary<ulong, Vector3>();
// ...
MyClientRPC(startPositionMap);
```

### INetworkSerializable

## Tooling

### Unity Editor

### Roslyn Analyzers

### Rider Unity Plugin

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we _not_ do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For framework, tools, and library proposals: Does this feature exist in other networking stacks and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other projects, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC. Please also take into consideration that Unity Multiplayer sometimes intentionally diverges from common multiplayer networking features.

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
