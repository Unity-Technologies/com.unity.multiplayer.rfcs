# Serialization

[feature]: #feature

- Start Date: `2021-07-12`
- RFC PR: [#26](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/26)
- SDK PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/0000)

# Summary

[summary]: #summary

This feature describes our approach to serialization. The goals of the serialization feature are that it is:

- Fast
- Allocation-free
- Consistent
- Debuggable
- Transparent (that is, easy for end users to follow and understand)
- Maintainable
- Future-ready (i.e., for jobs)

# Motivation

[motivation]: #motivation

This is both a performance and architectural change to MLAPI. Currently, serialization is one of our biggest offenders for CPU usage, and we've created pools to avoid allocations.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The serialization and deserialization is done via `FastBufferWriter` and `FastBufferReader`. These are similar in interface to the current `NetworkReader` and `NetworkWriter`, having methods for serializing individual types and methods for serializing packed numbers, but in particular provide a high-performance method called `WriteValue()`/`ReadValue()` (for Writers and Readers, respectively) that can extremely quickly write an entire unmanaged struct to a buffer. There's a trade-off of CPU usage vs bandwidth in using this: Writing individual fields is slower (especially when it includes operations on unaligned memory), but allows the buffer to be filled more efficiently.

Taking an example struct:

```C#
struct Message
{
    private float f;
    private bool b;
    private int i;

    void Serialize(ref FastBufferWriter writer);
}
```

Serialize can be implemented in two ways:

```C#
void Serialize(ref FastBufferWriter writer)
{
    writer.WriteSingle(f);
    writer.WriteBool(b);
    writer.WriteInt32(i);
}
```

```C#
void Serialize(ref FastBufferWriter writer)
{
    writer.WriteValue(this);
}
```

The former will create more efficiently packed data in the message, and can be even further optimized by using `WriteSinglePacked()` and `WriteInt32Packed()`, but it has two downsides: first, that it involves more method calls and more instructions, making it slower; and second, that it creates a greater opportunity for the serialize and deserialize code to become misaligned, since they must contain the same operations in the same order.

The latter can also be improved by optimizing the struct itself for alignment and optimizing the size of the values themselves, changing the definition to:

```c#
struct Message
{
    private float f;
    private short i;
    private bool b;

    void Serialize(ref FastBufferWriter writer);
}
```

This will naturally result in a more efficiently-packed buffer, though still not quite as efficient as using packed values.

You can also use a hybrid approach if you have a few values that will need to be packed and several that won't:

```C#
struct Message
{
    struct Embedded
    {
        private byte a;
        private byte b;
        private byte c;
        private byte d;
    }
    public Embedded embedded;
    public float f;
    public short i;

    void Serialize(ref FastBufferWriter writer)
    {
        writer.WriteValue(embedded);
        writer.WriteSinglePacked(f);
        writer.WriteInt32Packed(i);
    }
}
```

This allows the four bytes of the embedded struct to be rapidly serialized as a single action, then adds the compacted data at the end, resulting in better bandwidth usage than serializing the whole struct as-is, but better performance than serializing it one byte at a time.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## FastBufferWriter and FastBufferReader

FastBufferWriter and FastBufferReader are replacements for the current NetworkWriter and NetworkReader. They have much the same interface, but there are some critical differences:

- FastBufferWriter and FastBufferReader are **structs**, not **classes**. This means they can be constructed and destructed without allocations if necessary.
- FastBufferWriter and FastBufferReader both wrap `NativeArray<byte>`, allowing us to take advantage of extremely fast `Allocator.Temp` and `Allocator.TempJob` allocations to create temporary buffers.
- Neither FastBufferReader nor FastBufferWriter inherits from nor contains a `Stream`. Any code that currently operates on streams will have to be rewritten.
- FastBufferReader and FastBufferWriter are heavily optimized for speed, using aggressive inlining and unsafe code to achieve the fastest possible buffer storage and retrieval.
- FastBufferReader and FastBufferWriter use unsafe typecasts and UnsafeUtil.MemCpy operations on `byte*` values, achieving native memory copy performance with no need to iterate or do bitwise shifts and masks.
- FastBufferReader and FastBufferWriter are intended to make data easier to debug - one such thing to support will be a `#define MLAPI_FAST_BUFFER_UNPACK_ALL` that will disable all packing operations to make the buffers for messages that use them easier to read.

A core benefit of `NativeArray<byte>` is that it offers access to the allocation scheme of `Allocator.TempJob`. This uses a special type of allocation that is nearly as fast as stack allocation and involves no GC overhead, while being able to persist for a few frames. In general we will never need them for more than a frame, but this does give us a very efficient option for creating buffers as needed, which avoids the need to use a pool for them. The only downside is that buffers created this way must be manually disposed after use, as they're not garbage collected.

Of note, FastBufferReader and FastBufferWriter are **fixed size** buffers. They cannot grow.

## Bounds Checking

For performance reasons, FastBufferReader and FastBufferWriter **do not do bounds checking** on each write. Rather, they require the use of specific bounds checking functions - `VerifyCanRead(int amount)` and `VerifyCanWrite(int amount)`, respectively. This is to improve performance by allowing you to verify the space exists for the data being read or written in blocks, rather than doing that check on every single operation.

**In editor mode and development builds**, however, calling these functions records a watermark point, and any attempt to read or write past the watermark point will throw an exception. This ensures these functions are used properly, while avoiding the performance cost of per-operation checking in production builds.

## Bitwise Reading and Writing

Writing values in sizes measured in bits rather than bytes comes with a cost - first, it comes with a cost of having to track bitwise lengths and convert them to bytewise lenghts, and second, it comes with a cost of having to remember to add padding after your bitwise writes and reads to ensure the next bytewise write or read functions correctly, and to make sure the buffer length includes any trailing bits.

To address that, `FastBufferReader` and `FastBufferWriter` do not, themselves, have bitwise operations. When needed, however, you can create a `BitwiseWriter` or `BitwiseReader` instance, which is an IDisposable to ensure that no unaligned bits are left at the end - from the perspective of `FastBufferReader` and `FastBufferWriter`, only bytes are allowed. `BitwiseWriter` and `BitwiseReader` operate directly on the underlying buffer, so calling non-bitwise operations within a bitwise context is an error (and will raise an exception in non-production builds).

```
FastBufferWriter writer = new FastBufferWriter(256, Allocator.TempJob);
using(var bitWriter = writer.EnterBitwiseContext())
{
	bitWriter.WriteBit(a);
	bitWriter.WriteBits(b, 5);
} // Dispose automatically adds 3 more 0 bits to pad to the next byte.
```

## INetworkSerializable

A new `BufferSerializer` class wrapping `FastBufferReader` and `FastBufferWriter` will replace the existing `NetworkSerializer` to provide the same two-way interface as the existing `NetworkSerializer`, and is rewritten as a **ref struct** so that no allocation overhead is incurred in calling the `INetworkSerializable`'s `Serialize()` method. A **ref struct** is chosen instead of a normal struct to avoid the pitfalls associated with having mutable structs and forgetting to pass them with the ref keyword - we can manage this with `FastBufferReader` and `FastBufferWriter` because those are internal and we can rely on our ability to understand the implications of using them as structs, but our end users may be less experienced and may inadvertently pass this forward to calls that do not take the value by ref. A `ref struct` is always passed by reference rather than copied, but does have the disadvantage of being limited to the stack frame - so we can't use it for `FastBufferWriter` and `FastBufferReader` because we persist those.

`BufferSerializer` has an additional advantage (or disadvantage, depending on your viewpoint) compared to `FastBufferWriter` and `FastBufferReader`: since it's intended to be used by our end users, it performs bounds checking on every operation and throws exceptions if attempting to read or write out of bounds. This, combined with the `IsReader` if-checks, will make `BufferSerializer` slower than `FastBufferReader` and `FastBufferWriter` - but for advanced users, we can expose `FastBufferReader` and `FastBufferWriter` and allow the user to use them directly if they choose (with all appropriate warnings about them delivered).

# Drawbacks

[drawbacks]: #drawbacks

There are certainly some good reasons not to take this approach:

- The proposed serializers use unsafe code to maximize speed (we could use NativeArray's built-in methods to gain its safety checks, but would lose considerable performance in so doing)
- The proposed design uses fixed-size buffers and requires manual bounds checking, which may make it a little more difficult to use.

However...

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- This approach achieves extremely fast performance
- This approach avoids memory allocations at the core layer except by way of `NativeArray`'s `Allocator.Temp` and `Allocator.TempJob`, which are orders of magnitude faster.

# Prior art

[prior-art]: #prior-art

This RFC is largely inspired from personal past experience on multiple projects discovering the fastest and most effective ways to serialize data with minimal allocations and minimal copies, applied within the current MLAPI framework.

Below are some comparisons to some of our main competitors in the C#/Unity arena.

- **Unity Netcode:**
  The Unity Netcode implementation of serialization has a lot in common with this one in the underlying nuts and bolts - the implementation of DataStreamWriter and DataStreamReader is very similar to the proposed implementation of FastBufferWriter and FastBufferReader; both use raw `byte*` values under the hood and perform their operations in "unsafe" code blocks.

  Where FastBufferReader and FastBufferWriter differ is primarily at the interface level, where the FastBuffer options provide the `WriteValue<T>()` helpers for any `unmanaged` T, and in how the two handle going over capacity: the DataStream options perform checks on each operation, while the FastBuffer options hoist those checks up to the user level to allow them to be batched, and provides editor/development build-only safeguards to ensure the user is using those checks correctly. At the cost of a little more overhead on the user-side of any given serialization implementation, this allows us to reduce the number of branches we execute, which results in faster code.

  By comparison to FastBufferWriter and FastBufferReader, DataStreamWriter and DataStreamReader are very bare-bones: they contain methods for serializing primitive types, packed types, individual bits, and bytes, but contain no helpers for serializing more complex types. FastBufferWriter and FastBufferReader add built-in support for certain higher level types, such as INetworkSerializable, as well as any type that fits the `unmanaged` requirement, such as structs.

- **UNet/HLAPI:**
  UNet's source repository seems to no longer exist, so I am basing this information on a fork that I hope is accurate to the state of HLAPI prior to the repository being removed.

  The serializers in UNet use `NetworkReader` and `NetworkWriter` classes very similar to our existing ones, except rather than containing a `Stream`, they contain a simple `byte[]` that they write to one byte at a time. These classes are not allocation-free, but HLAPI does not appear to use a pool. An example for serializing a ulong:

  ```C#
  public void Write(ulong value)
  {
      m_Buffer.WriteByte8(
          (byte)(value & 0xff),
          (byte)((value >> 8) & 0xff),
          (byte)((value >> 16) & 0xff),
          (byte)((value >> 24) & 0xff),
          (byte)((value >> 32) & 0xff),
          (byte)((value >> 40) & 0xff),
          (byte)((value >> 48) & 0xff),
          (byte)((value >> 56) & 0xff));
  }
  ```
  
  Each of call to `WriteByte*()` will have an additional call to `WriteCheckForSpace()`, which verifies there's enough space and resizes and copies the array if necessary, resulting in significant method call overhead and possible allocations. By comparison, FastBufferWriter aggressively inlines all of its writes, and writing any unmanaged type will always be done with a single assignment operation of that type followed by a single position increment (excepting special operations like packed writes that need to do transformation operations). Because we have very clear constraints on how much data can be sent to the transport for one message, we can make allocate buffers that fill those constraints and avoid any overhead of needing to resize them - we simply never will resize them. We also move our capacity checking to outside the write calls, so capacity can be checked just once for a series of several writes, dramatically reducing the overhead of capacity validation.
  
- **Mirror:**
  
  Mirror is not allocation-free - its NetworkReader and NetworkWriter classes must be instantiated. As a result, they do much the same that we do: they create PooledNetworkReader and PooledNetworkWriter classes and get and release them from a pool. By comparison, using mutable structs for FastBufferReader and FastBufferWriter involves no allocations at all, and they cannot be leaked.
  
  Mirror's NetworkReader and NetworkWriter are built around a managed byte[] array, and each write checks for capacity and grows the array if necessary, then performs copies using Array.ConstrainedCopy() when calling WriteBytes(). When calling other functions, such as WriteULong(), it writes one byte at a time as follows:
  
  ```c#
  public static void WriteULong(this NetworkWriter writer, ulong value)
  {
      writer.WriteByte((byte)value);
      writer.WriteByte((byte)(value >> 8));
      writer.WriteByte((byte)(value >> 16));
      writer.WriteByte((byte)(value >> 24));
      writer.WriteByte((byte)(value >> 32));
      writer.WriteByte((byte)(value >> 40));
      writer.WriteByte((byte)(value >> 48));
      writer.WriteByte((byte)(value >> 56));
  }
  ```
  
  Each of those calls to `WriteByte()` will have an additional call to `EnsureCapacity()`, which verifies there's enough space and resizes and copies the array if necessary, resulting in significant method call overhead and possible allocations. This is taken even further than UNet's implementation - where UNet would check for 8 bytes of space in a single check during `WriteByte8()`, Mirror checks for space before writing each and every individual byte. The same comments apply here as the ones I made discussing UNet.
  

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- There are currently no unresolved questions.

# Future possibilities

[future-possibilities]: #future-possibilities

There are no specific future features that are likely to fall out of this work.
