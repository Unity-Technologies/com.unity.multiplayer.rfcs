- Feature Name: `serializable-types`
- Start Date: 2020-11-06
- RFC PR: [RFC#2](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/2)
- Issue: [MLAPI#454](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/issues/454)

# Summary
[summary]: #summary

This RFC proposes a standard way to support both built-in and custom serialization flows. Proposed design could be further extended with additional built-in serialization support, templated custom serializers and others in the future.

# Motivation
[motivation]: #motivation

Type serialization is one of the most common areas in the network programming. We want to automate this process as much as we can, offer out-of-the-box features to make life easier.

MLAPI used to offer serialization support that is much less performant (due to boxing, runtime type checking etc.) and (arguably) less convenient. When [Standard RPC API](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/blob/master/text/0001-std-rpc-api.md) was introduced, it also did open a whole set of opportunities in this area. At the time of writing this RFC, this proposed design applies to the serialization of RPC parameters and is very likely to be used in other areas that use serialization such as network variables.

We want to offer built-in serialization support for most commonly used types and also support custom serialization for user-defined types with ease.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Serialization

Multiplayer framework has built-in serialization support for C# and Unity primitive types out-of-the-box, also with ability to further extend network serialization for user-defined types implementing `INetworkSerializable` interface.

### C# Primitives

`bool`, `char`, `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `string` types will be serialized by built-in serialization code.

```cs
[ServerRpc]
void FooServerRpc(int somenumber, string sometext) { /* ... */ }

void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        FooServerRpc(Time.frameCount, "hello, world");
    }
}
```

### Unity Primitives

`Color`, `Color32`, `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Ray`, `Ray2D` types will be serialized by built-in serialization code.

```cs
[ClientRpc]
void BarClientRpc(Color somecolor) { /* ... */ }

void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        BarClientRpc(Color.red);
    }
}
```

### Enum Types

A user-defined enum type will be serialized by built-in serialization code (with underlying integer type).

```cs
enum SmallEnum : byte
{
    A,
    B,
    C
}

enum NormalEnum // default -> int
{
    X,
    Y,
    Z
}

[ServerRpc]
void ConfigServerRpc(SmallEnum smallEnum, NormalEnum normalEnum) { /* ... */ }

void Update()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        ConfigServerRpc(SmallEnum.A, NormalEnum.X);
    }
}
```

### Static Arrays

Static arrays like `int[]` will be serialized by built-in serialization code if their underlying type is either one of serialization supported types (e.g. `Vector3`) or if they implement `INetworkSerializable` interface.

```cs
[ServerRpc]
void HelloServerRpc(int[] scores, Color[] colors) { /* ... */ }

[ClientRpc]
void WorldClientRpc(MyComplexType[] values) { /* ... */ }
```

### INetworkSerializable & BitSerializer

Complex user-defined types that implement `INetworkSerializable` interface will be serialized by user provided serialization code.

```cs
interface INetworkSerializable
{
    void NetworkSerialize(BitSerializer serializer);
}
```

An instance of `BitSerializer` will be passed into `INetworkSerializable::NetworkSerialize(BitSerializer)` method which can be used to easily serialize fields by reference.

```cs
struct MyComplexStruct : INetworkSerializable
{
    public Vector3 Position;
    public Quaternion Rotation;

    // INetworkSerializable
    public NetworkSerialize(BitSerializer serializer)
    {
        serializer.Serialize(ref Position);
        serializer.Serialize(ref Rotation);
    }
    // ~INetworkSerializable
}
```

All types with built-in serialization support will also be supported by `BitSerializer` with `BitSerializer::Serialize(...)` variant methods.

```cs
class BitSerializer
{
    bool IsReading { get; }

    // C# Primitives
    void Serialize(ref bool value) { /* ... */ }
    void Serialize(ref char value) { /* ... */ }
    // and other variants: sbyte, byte, short, ushort, int, uint, long, ulong, double, string

    // Unity Primitives
    void Serialize(ref Color value) { /* ... */ }
    // and other variants: Color32, Vector2, Vector3, Vector4, Quaternion, Ray, Ray2D

    // Enum Types
    void Serialize<TEnum>(ref TEnum value) where TEnum : Enum { /* ... */ }

    // Static Arrays
    void Serialize(ref bool[] array) { /* ... */ }
    void Serialize(ref Color[] array) { /* ... */ }
    void Serialize<TEnum>(ref TEnum[] array) where TEnum : Enum { /* ... */ }
    // and other arrays of built-in supported type variants
}
```

`BitSerializer` will both serialize and deserialize fields based on its serialization mode indicated by `IsReading` flag using its internal `BitReader` and `BitWriter` instances.

```cs
class BitSerializer
{
    BitReader m_Reader;
    BitWriter m_Writer;

    bool IsReading { get; }

    BitSerializer(BitReader reader)
    {
        IsReading = true;
        m_Reader = reader;
    }

    BitSerializer(BitWriter writer)
    {
        IsReading = false;
        m_Writer = writer;
    }

    void Serialize(ref int value)
    {
        if (IsReading)
        {
            value = m_Reader.ReadInt32Packed();
        }
        else
        {
            m_Writer.WriteInt32Packed(value);
        }
    }

    // ...
}
```

#### Conditional Serialization

As the developer has more control over serialization of a struct, one might implement conditional serialization at runtime.

We will explore more advanced use-cases with the examples below:

##### Example: Array

```cs
public struct MyCustomStruct : INetworkSerializable
{
    public int[] Array;

    public void NetworkSerialize(BitSerializer serializer)
    {
        // Length
        int length = 0;
        if (!serializer.IsReading)
        {
            length = Array.Length;
        }

        serializer.Serialize(ref length);

        // Array
        if (serializer.IsReading)
        {
            Array = new int[length];
        }

        for (int n = 0; n < length; ++n)
        {
            serializer.Serialize(ref Array[n]);
        }
    }
}
```

Reading:

- (De)serialize `length` back from the stream
- Iterate over `Array` member `n=length` times
- (De)serialize value back into `Array[n]` element from the stream

Writing:

- Serialize `length=Array.Length` into stream
- Iterate over `Array` member `n=length` times
- Serialize value from `Array[n]` element into the stream

`BitSerializer.IsReading` flag is being utilized here to determine whether or not to set `length` value to prepare before writing into the stream — on the flip side, we use it to determine whether or not to create a new `int[]` instance with `length` size to set `Array` before reading values from the stream.

##### Example: Move

```cs
public struct MyMoveStruct : INetworkSerializable
{
    public Vector3 Position;
    public Quaternion Rotation;

    public bool SyncVelocity;
    public Vector3 LinearVelocity;
    public Vector3 AngularVelocity;

    public void NetworkSerialize(BitSerializer serializer)
    {
        // Position & Rotation
        serializer.Serialize(ref Position);
        serializer.Serialize(ref Rotation);
        
        // LinearVelocity & AngularVelocity
        serializer.Serialize(ref SyncVelocity);
        if (SyncVelocity)
        {
            serializer.Serialize(ref LinearVelocity);
            serializer.Serialize(ref AngularVelocity);
        }
    }
}
```

Reading:

- (De)serialize `Position` back from the stream
- (De)serialize `Rotation` back from the stream
- (De)serialize `SyncVelocity` back from the stream
- Check if `SyncVelocity` is set to `true`, if so:
- (De)serialize `LinearVelocity` back from the stream
- (De)serialize `AngularVelocity` back from the stream

Writing:

- Serialize `Position` into the stream
- Serialize `Rotation` into the stream
- Serialize`SyncVelocity` into the stream
- Check if `SyncVelocity` is set to `true`, if so:
- Serialize `LinearVelocity` into the stream
- Serialize `AngularVelocity` into the stream

Unlike [the Array example above](#example-array), we do not use `BitSerializer.IsReading` flag to change serialization logic but the value of a serialized flag itself. If `SyncVelocity` flag is set to `true`, both `LinearVelocity` and `AngularVelocity` will also be serialized into the stream — otherwise when it is set to `false`, we will leave `LinearVelocity` and `AngularVelocity` with default values.

#### Recursive Nested Serialization

It is possible to recursively serialize nested members with `INetworkSerializable` interface down in the hierachy tree.

Let's have a look at the example below:

```cs
public struct MyStructA : INetworkSerializable
{
    public Vector3 Position;
    public Quaternion Rotation;

    public void NetworkSerialize(BitSerializer serializer)
    {
        serializer.Serialize(ref Position);
        serializer.Serialize(ref Rotation);
    }
}

public struct MyStructB : INetworkSerializable
{
    public int SomeNumber;
    public string SomeText;

    // nested member with `INetworkSerializable` interface
    public MyStructA StructA;
    
    public void NetworkSerialize(BitSerializer serializer)
    {
        serializer.Serialize(ref SomeNumber);
        serializer.Serialize(ref SomeText);

        // serialize `StructA` into the same stream using `serializer`
        StructA.NetworkSerialize(serializer);
    }
}
```

If we were to serialize `MyStructA` alone, it would serialize `Position` and `Rotation` into the stream via `BitSerializer`.

However, if we were to serialize `MyStructB`, it would serialize `SomeNumber` and `SomeText` into the stream, then serialize `StructA` by calling `MyStructA`'s `void NetworkSerialize(BitSerializer)` method which serializes `Position` and `Rotation` into the same stream.

**Note:** Technically, there is no hard-limit on how many `INetworkSerializable` fields you can serialize down the tree hierachy but in practice, there are some memory and bandwidth boundaries you'll need to watch out for.

**Pro-tip:** You can [conditionally serialize](#conditional-serialization) in [recursive nested serialization](#recursive-nested-serialization) scenario and make use of both features! :)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## INetworkSerializable

`INetworkSerializable` interface will not enforce anything other than `NetworkSerialize(BitSerializer)` method.

```cs
interface INetworkSerializable
{
    void NetworkSerialize(BitSerializer serializer);
}
```

## BitSerializer

`BitSerializer` is the main aggregator that implements serialization code for built-in supported types and holds `BitReader` and `BitWriter` instances internally.

```cs
class BitSerializer
{
    BitReader m_Reader;
    BitWriter m_Writer;

    bool IsReading { get; }

    BitSerializer(BitReader reader)
    {
        IsReading = true;
        m_Reader = reader;
    }

    BitSerializer(BitWriter writer)
    {
        IsReading = false;
        m_Writer = writer;
    }

    void Serialize(ref int value)
    {
        if (IsReading) value = m_Reader.ReadInt32Packed();
        else m_Writer.WriteInt32Packed(value);
    }

    void Serialize(ref Vector3 value)
    {
        if (IsReading) value = m_Reader.ReadVector3Packed();
        else m_Writer.WriteVector3Packed(value);
    }

    unsafe void Serialize<TEnum>(ref TEnum value) where TEnum : unmanaged, Enum
    {
        if (sizeof(TEnum) == sizeof(int))
        {
            if (IsReading)
            {
                int intValue = m_Reader.ReadInt32Packed();
                value = *(TEnum*)&intValue;
            }
            else
            {
                TEnum enumValue = value;
                m_Writer.WriteInt32Packed(*(int*)&enumValue);
            }
        }
        else if (sizeof(TEnum) == sizeof(byte))
        {
            // ...
        }

        // ...
    }

    // ...
}
```

## RPC & ILPP Changes

At the time of writing this RFC proposal, `NetworkBehaviour.__beginSendServerRpc` and other internal RPC methods are returning and consuming `BitWriter` instances but they should return and consume `BitSerializer` instances constructed with `BitWriter` and `BitReader` instead. IL injected into RPC method bodies should change and use `BitSerializer` instead of `BitWriter` and generated static RPC handler methods should also use `BitSerializer` instead of `BitReader`.

# Drawbacks
[drawbacks]: #drawbacks

N/A

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
  - There has been no alternative discussed.
- What other designs have been considered and what is the rationale for not choosing them?
  - N/A
- What is the impact of not doing this?
  - This is an essential feature that we need to support ASAP.

# Prior art
[prior-art]: #prior-art

## Unreal's `FArchive`

Unreal Engine has [`FArchive`](https://docs.unrealengine.com/en-US/API/Runtime/Core/Serialization/FArchive/index.html) which looks quite similar to `BitSerializer` proposed above. We may or may not get some ideas from there while implementing this RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
  - N/A
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
  - N/A
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - N/A

# Future possibilities
[future-possibilities]: #future-possibilities

## Templated Custom Serializers

Currently, there is no way to swap `BitSerializer` with something else. It is still possible to manually serialize types down to `byte[]` and use `BitSerializer.Serialize(ref byte[] value)` API but it would be much better to have full control over serializer. This RFC considers this as a potential improvement for the future.
