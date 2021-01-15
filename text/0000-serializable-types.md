- Feature Name: `serializable-types`
- Start Date: 2020-11-06
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

_RFC draft for **Serializable Types** (to be updated and properly proposed)_

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Serialization

Multiplayer framework has built-in serialization support for C# and Unity primitive types out-of-the-box, also with ability to further extend network serialization for user defined types implementing `INetworkSerializable` interface.

### C# Primitives

`bool`, `char`, `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `string` types will be serialized by built-in serialization code.

### Unity Primitives

`Color`, `Color32`, `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Ray`, `Ray2D` types will be serialized by built-in serialization code.

### Enum Types

A user-defined enum type will be serialized by built-in serialization code (with underlying integer type).

### Static Arrays

Static arrays like `int[]` will be serialized by built-in serialization code if their underlying type is either one of serialization supported types (e.g. `Vector3`) or if they implement `INetworkSerializable` interface.

### NetworkObject & NetworkBehaviour

`NetworkObject` and `NetworkBehaviour` instances will be serialized by built-in serialization code if instances are not `null` and `.IsSpawned == true`. Ids of spawned `NetworkObject` and `NetworkBehaviour` instances will be resolved by actively running `NetworkManager` therefore those ids will be the links between local and remote instances. Also, those ids will be used when serializing `NetworkObject` and `NetworkBehaviour` instances as a part of an RPC call.

### INetworkSerializable

Complex user-defined types that implements `INetworkSerializable` interface will be serialized by user provided serialization code:

```cs
struct MyStruct : INetworkSerializable
{
    public Vector3 Position;
    public Quaternion Rotation;

    // INetworkSerializable
    public void NetworkRead(BitReader reader)
    {
        Position = reader.ReadVector3Packed();
        Rotation = reader.ReadRotationPacked();
    }

    public void NetworkWrite(BitWriter writer)
    {
        writer.WriteVector3Packed(Position);
        writer.WriteRotationPacked(Rotation);
    }
    // ~INetworkSerializable
}
```

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
