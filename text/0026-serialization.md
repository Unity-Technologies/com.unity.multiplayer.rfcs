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

This feature also proposes a new way of defining messages that supports these goals.

# Motivation

[motivation]: #motivation

This is both a performance and architectural change to MLAPI. Currently, serialization is one of our biggest offenders for CPU usage, and we've created pools to avoid allocations. Additionally, sending a message passes through a number of steps that can be difficult to follow for an unfamiliar user - the message is first serialized (and serialized differently depending on whether it's going to another node or going to the inbound queue), placed into the MessageQueueHistoryFrame as a buffer, then later deserialized, modified, and reserialized in a different format. The serialization itself is also sometimes spread out throughout the code base - one function serializes some of it, then calls another function that adds more to it, and so on. All of this comes at the cost that users unfamiliar with the code base can have a difficult time following the path a message takes or understanding what is (or should be) in the buffer in order to debug it.

This feature will rethink the whole flow of serialization, from the beginning (defining messages) all the way through serializing, batching, sending, deserializing, and handling messages.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

## Defining Messages

Messages in MLAPI are represented in-memory using structs containing, as much as possible, only unmanaged POD. Each struct must implement the IMessage interface, which is defined as follows:

```C#
interface IMessage
{
    enum MessageType
    {
        //...
    }

    // Returns the type of the message, used in automatic message header generation
    MessageType GetMessageType();

    // Returns an upper bound on the message size.
    // Doesn't have to be exactly accurate to the serialized size
    // (over-estimating is preferred to using more CPU cycles for an exact result)
    // but must be AT LEAST the message size.
    // When sent with a non-fragmenting QoS, it also must be AT MOST (MTU - 6) bytes.
    int GetSizeUpperBound();

    // Serialize to a buffer
    void Serialize(ref FastBufferWriter writer, in NetworkContext context);
    
    // Handle the message
    void Handle(in NetworkContext context);
}
```

Additionally, each message should have a method on it as follows:

```c#
[MessageHandler(IMessage.MessageType.MyMessageType)]
static void Receive(ref FastBufferReader reader, in NetworkContext context);
```

This static method is responsible for constructing an instance of the received type and handling it. The `[MessageHandler]` attribute statically registers this handler to be responsible for this type via ILPP code generation, just like `[ServerRpc]` and `[ClientRpc] `

Handling will be put in its own method so that it's separate from deserialization, which is important in efficient handling of loopback messages.

The second parameter, `NetworkContext`, contains any data that would be necessary for the handling of the message, including:

- The NetworkManager instance associated with this message
- The client ID the message came from
- The network channel the message came in through
- The received message type (in case two messages share a common deserializer and need to switch on this)
- The timestamp at which the message was received

More information may be added to `NetworkContext` as necessary.

A sample message might look like this:

```c#
struct Message : IMessage
{
    private float f;
    private int i;
    private bool b;

    void GetType()
    {
        return IMessage.MessageType.Message;
    }

    void Serialize(ref FastBufferWriter writer)
    {
        writer.WriteValue(this);
    }

    int GetSizeUpperBound()
    {
        return UnsafeUtility.SizeOf<Message>();
    }

    void Handle(in NetworkContext context)
    {
        // Handler code
    }

    [MessageHandler(IMessage.MessageType.Message)]
    static void Receive(ref FastBufferReader reader, in NetworkContext context)
    {
        Message message;
        reader.ReadValue(message);
        message.Handle(context);
    }
}
```

## Serialization and Deserialization

The serialization and deserialization is done via `FastBufferWriter` and `FastBufferReader`. These are similar in interface to `NetworkReader` and `NetworkWriter`, having methods for serializing individual types and methods for serializing packed numbers, but in particular provide a high-performance method called `WriteValue()`/`ReadValue()` (for Writers and Readers, respectively) that can extremely quickly write an entire unmanaged struct to a buffer. There's a trade-off of CPU usage vs bandwidth in using this: Writing individual fields is slower (especially when it includes operations on unaligned memory), but allows the buffer to be filled more efficiently.

Taking an example class:

```C#
struct Message : IMessage
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
struct Message : IMessage
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
struct Message : IMessage
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

## Messages

When sending messages, instead of working directly with buffers, messages are created on the stack as struct instances, populated with data, and then serialized to the buffer from there. There are a few reasons for doing it this way:

- The primary reason is readability and maintainability. Putting all the code related to serializing, deserializing, and handling a message in one place makes it easy to understand how that message works. Building the message as a struct, which should match the network transfer format as closely as possible, makes it easy to understand what the data going across the wire will look like.
- The secondary reason is performance - structs can be created on the stack with no heap allocations, and with FastBufferWriter and FastBufferReader, they can be rapidly serialized as whole units.
- The third reason is easier debugging - for situations where a lot of packed integers are being used, a preprocessor flag can be used to switch between optimal byte packing and simple whole-struct serialization. At debug time, when there's no dynamic data involved, the buffer in a reader or writer can be simply cast to the desired type by adding a watch of `*(MessageType*)reader.GetUnsafePtrAtCurrentPosition()`
- The fourth reason is to improve the interface by which messages are sent - instead of needing a Begin...() to add header information and an End...() to perform sending logic, MessageQueueContainer now simply has `public void SendMessage<TMessageType>(in TMessageType message) where TMessageType: IMessage{}`, which can handle both responsibilities, as well as message batching responsibilities, all in one location.

## Message Header

Each message sent will be prepended with a standardized message header, which is also represented as a struct with the following data:

```c#
struct MessageHeader
{
    // Sized 1 byte
    public MessageType MessageType;
    // Sized 1 byte
    public NetworkUpdateStage UpdateStage;
    // Sized 2 bytes
    public short MessageSize;
    // Sized 1 byte
    public NetworkChannel NetworkChannel
}
```

Since the largest value is 2 bytes in size, the struct should be aligned to a 2 byte alignment, which should make the struct packed and the total header size equal to exactly 5 bytes, which can be sent and received using `WriteValue` and `ReadValue` without any wasted bandwidth.

In addition to the message header, there's also a batch header - the batch header is simply 2 bytes containing the total size of the batch.

## Sending Messages

As mentioned above, the interface for sending message changes from "begin add item, populate buffer, end add item" to "create struct, pass struct to SendMessage". However, there are more changes beyond just the interface to the way messages are sent:

Each client object holds on it a fixed-size `SendQueue` of FastBufferWriters associated with that client (fixed-size so as to avoid allocations, but large enough to reasonably hold all the messages that would need to be sent in a single frame, and with the size adjustable by game teams if it turns out to still be too small). At the start of each frame, this queue size is `0`, and the writer in index 0 persists from frame to frame, wrapping a `NativeArray<byte>` allocated with `Allocator.Persistent`. When we need to write a message, if the writer in `SendQueue[queueSize]` doesn't have enough remaining capacity to hold the message we're attempting to write, we increment the queue size and create a new writer wrapping a new `NativeArray<byte>` allocated with `Allocator.TempJob`. This allocator is much faster than GC allocation, approaching the speed of stack allocation, and doesn't create an object that's garbage collected (instead, it must be manually deallocated).

In the parameter list when sending to multiple recipients, we also have multiple ways in which we can incur allocation overhead. Currently we have our sending functions taking `ulong[] clientIds` as a parameter. In a lot of situations, this is resulting in extra allocations to create this list, particularly when we only have one client ID. To reduce the allocation overhead, this parameter will be replaced with `IReadOnlyList<ulong>` as a generic parameter, which will allow any indexable array type (`List<ulong>`, `ulong[]`, etc) to be used for this parameter. When new lists need to be made (for example, for a host filtering out its own local client ID), it can be made without allocations by creating a `NativeArray<ulong>` with `Allocator.Temp` and then passed in using a wrapper class that allows it to implement `IReadOnlyList<T>` (it currently implements all required methods, but doesn't name the interface in its implementation list, so a simple wrapper class can provide that and pass those things through). This will allow `NativeArray<ulong>` to be passed and used without any boxing allocations that would be incurred if we passed it as `IEnumerable<T>`. (As a note: `foreach` on an `IEnumerable<T>` will incur boxing allocations if the return type is a value type - i.e., a struct - so `IReadOnlyList<T>` is preferable because it allows iteration by index, which will not incur allocations.)

Additionally, there is a separate method for sending to a single client vs sending to multiple clients, so sending to a single client will have no requirement to create a list from a single value. As detailed below, the single and multiple sending behaviors are sufficiently different as to warrant separate implementations, anyway. 

There are three special cases for how the sending logic serializes the message.

### Sending to a single remote client

The first thing the sending logic does is call the `GetSizeUpperBound()` method on the message and compare the result against the size left in the buffer. If the buffer doesn't have enough space left for the message, the first two bytes of the writer are overwritten with the value of writer.Position, and then the writer's buffer is placed onto the client's send queue and a new buffer allocated with `Allocator.TempJob`.

**If at any time, `GetSizeUpperBound()` returns a value larger than `MTU - sizeof(MessageHeader) - sizeof(short)` when sending with a non-fragmenting QoS, an exception is thrown and the message is not sent.**

The sending logic then creates a `MessageHeader` storing the MessageType and UpdateStage, and then seeks forward in the buffer an amount equal to `sizeof(MessageHeader)` to reserve space for the header, while storing the original position.

The sending logic then calls the `Serialize()` method on the message to populate the buffer.

Finally, the logic stores the actual size of the message onto the message header, seeks back to the header position, writes the header to the buffer, and then seeks once more to the end of the buffer (the value of `writer.Position` at the end of the call to `Serialize()`)

### Sending to multiple remote clients

Sending to remote clients is a bit more complex because each client has its own FastBufferWriter, and each may or may not be full enough to trigger a batch to be sent.

In this case, `GetSizeUpperBound()` isn't called. Instead, a `NativeArray<byte>` is constructed using `Allocator.Temp` - a bump allocator that effectively operates the same way as allocating stack memory - and a new FastBufferWriter is created wrapping that NativeArray (which does not cause an allocation, as FastBufferWriter is a struct). The data is then serialized once to that writer instead of an existing client's writer.

Afterward, a `MessageHeader` is created and fully initialized with the size of the serialized data, then each requested client destination is iterated. Since we know the actual size of the data, we can skip the call to `GetSizeUpperBound()` in this case (which is meant to be a more efficient alternative to creating a temp buffer and copying), and we can use the actual size to determine whether or not to enqueue the current batch and create a new one. We then write the header (which is the same for all clients) and then copy the serialized data (using `UnsafeUtility.MemCpy`) into the destination buffer.

### Sending to a local client (loopback)

The local client case is the most complicated case, and has some special logic for it. When an item is destined for a LoopBack destination, what happens is based on the current UpdateStage as compared to the UpdateStage requested for the message.

- If the update stage is the current stage, then message.Handle() is called immediately. If this was the only destination, no serialization is performed at all - the message is simply handled on the spot.
- If the update stage is a later stage than the current stage, a new NativeArray is created with Allocator.TempJob, the message is serialized to it, and its data is placed in the incoming queue for the **current** frame. (See "Receiving Messages" below for elaboration on how the incoming queue works.)
- If the update stage is an earlier stage than the current stage, a new NativeArray is created with Allocator.TempJob, the message is serialized to it, and its data is placed in the incoming queue for the **next** frame

When the message can't be handled immediately, the use of `Allocator.TempJob` provides an extremely efficient allocation of the NativeArray that can last long enough for it to be handled before it must be deallocated. Based on benchmarks, `Allocator.TempJob` is nearly as fast as `Allocator.Temp`, and orders of magnitude faster than a standard allocation.

When serializing messages for loopback, the logic follows the above two cases depending on whether there is only one destination or multiple - if there is only one, the data is serialized directly to the newly created array via a FastBufferWriter; when there are multiple destinations, it's copied into this array from the one allocated with `Allocator.Temp`, so there is still only one serialization call.

When `Allocator.TempJob` is used, it must be disposed, so this NativeArray is added to the receiver dispose queue for the relevant frame to ensure it is properly disposed. (See "Receiving Messages" below for more information on the receiver dispose queue.)

### Buffer Overflow Protection

Before writing anything to a `FastBufferWriter`, a `VerifyCanWrite()` method must be called to verify that the amount of bytes you intend to write are available in the buffer. This should be called as infrequently as possible - and in fact, in our network code, this is called before calling `Serialize()` using the result of `GetSizeUpperBound()` (or using the full MTU size when serializing to a temporary buffer). This must be called before writing anything in order to avoid buffer overflow.

This is enforced in editor and development builds only by keeping track of how much space has been verified valid for write and throwing an exception if the user tries to write past that point. After each serialization, we reset the watermark to the current position (as `GetSizeUpperBound()` is allowed to be larger than the actual serialized data) so that further writes after the end of serialization become disallowed.

In production builds, this bounds checking is disabled; however, one extra piece of validation is provided and remains enabled during production builds: when attempting to convert a `FastBufferWriter` to an array or native array, it will check that its current `Position` has not exceeded its `Capacity`, and if it has, it will **terminate the application completely** (an exception is not acceptable in this case, as once a buffer overflow has occurred, memory in unknown areas of the application is already corrupted). When we're serializing messages, we also invoke that check willfully after each call to `message.Serialize()` in order to catch such corruption as quickly as possible.

### Channels/QoS

The way channels work is going to be rethought - the `NetworkChannel` value sent to the transport will represent a direct one-to-one mapping between a single `NetworkChannel` and a single QoS value. The current `NetworkChannel` values will be lofted up to a higher level and will be represented as a byte-sized value in the message header. This allows us to be a little more intelligent about how we handle network channels - specifically because it allows us to be able to to tell when two channels are behaviorally identical and treat them the same at the transport level.

Since messages sent with different QoS guarantees have different behavior requirements, the data for different QoS guarantees cannot be stored in the same buffer. To handle this, each client object stores its current QoS on its object next to the buffer. If any message is sent for a QoS other than the one currently stored, the batch is completed and a new batch is started for the new QoS (i.e., a new NativeArray is added to the send queue).

There are two reasons for doing it this way:

The first is to avoid allocations. Storing a separate writer for each channel would result in needing to keep a dictionary of writers that can be allocated on demand.

The second is to avoid regressions on message ordering. If each QoS is allowed to have its own buffer, then messages created in order A, B, C, D could be sent in order D, A, B, C, if D is sent using a different QoS than A, B, and C. Flushing on QoS change ensures we don't deliver messages to the transport in a different order than the order requested by the user. 

### Fragmenting QoS Behaviors

The handling of message sizes is based on the QoS for the channel currently being used to send the message. As mentioned earlier, on non-fragmenting channels, it is an error (raising a nice, loud exception) to attempt to send any single message larger than the MTU size. Non-fragmenting channels, however, use a larger value, which is the largest size of fragmented message the transport is able to handle (i.e., in the current UNet implementation, this would be ConnectionConfig::FragmentSize * 64 - a standard interface must be created for retrieving this value). This will allow us to support user needs for sending large messages, up to this limit.

In general, we should strive to not use fragmenting QoS for internal messages at all. Fragmenting QoS is less optimized/slower and we have seen issues with UNet where the overall throughput capacity of the fragmenting QoS is far lower than non-fragmenting QoS, and is also quite unpredictable, even when sending messages small enough to not need to be fragmented at all. As such, we should avoid using fragmenting QoS anywhere, and also not default any user-generated messages to fragmenting, requiring the user to explicitly request fragmenting QoS if they need it.

As a small bonus for containing all internal messages within non-fragmenting channels, not only will reliable messages be transmitted more optimally, but we also lower the burden on any user who wishes to implement their own transport: fragmenting QoS is by far the most complex to implement, so if we don't use it, we can document that QoS as optional, letting the user avoid implementing it unless they themselves need it.

### End of Frame

At the end of the frame, each client is iterated once to determine if its writers contain unsent messages. It first iterates the `SendQueue` up to the queue size to see if any writers exist in it, and for each writer, if `writer.Position != 2` (reserved batch header), it's finalized (the first two bytes updated to contain the batch size) and also sent to the transport. The writer's `Dispose()` method is then called (unless it's the persistent writer at index 0, which is not disposed until the client disconnects), and once all writers have been sent, the queue size is reset to 0.

## Receiving Messages

Incoming messages are processed through an incoming message queue, which contains items of this type:

```C#
struct IncomingMessageItem
{
	MessageType MessageType;
	FastBufferReader Reader;
	ulong SenderId;
	NetworkChannel ReceivingChannel;
	float Timestamp;
}
```

A separate queue of this type exists for each update stage, and exists as a simple array with a fixed size. If it gets full, we begin discarding messages to avoid allocating to resize the array (however, the array is reasonably sized - and, like the send queue, this size is adjustable by the user - so as to ensure that this will only happen in the case of a malicious actor sending a DoS attack). At the point of receiving, we construct a new `NativeArray<byte>` using `Allocator.TempJob` and copy the received data into it (this one-time copy is to avoid any assumptions about whether or not the data in the array segment will change between receiving and processing it). Then we create a `FastBufferReader` from that array segment and read two bytes from it to determine the total size of the message batch. We then iterate the following steps until we've read the full amount of data indicated in the batch size header:

- Read a value of type `MessageHeader`
- Use `header.UpdateStage` to pick the correct incoming message queue
- Set the value of the array item at the queue's current position to:
  - MessageType = header.MessageType
  - Reader = reader (since `FastBufferReader` is a struct, this will create a copy of the reader pointed at the current position in the buffer)
    - We then call `Reader.SetCapacity(header.MessageSize)` to mark the boundaries of the message so we can read it safely without any buffer overreads.
  - SenderId = clientId
  - ReceivingChannel = networkChannel
  - Timestamp = receiveTime
- Increment the queue's current position

Finally, an additional queue exists: The dispose queue. Like the incoming message queues, this queue is a fixed-size array of `NativeArray<byte>` objects sized to the combined size of all update queues. Once each buffer has been processed, it's added to the dispose queue, where it will be disposed at the end of the frame after all messages pointing into it have been handled.

As a small security measure, since we know there is an absolute upper bound of MTU size, the following will be rejected:

- Any batch with a size > `(MTU - sizeof(short))`
- Any message whose size when compared to its position in the batch would put it beyond the `MTU` size boundary
- Any attempt to read any field with a dynamic size (i.e., a string, a list, etc) in `FastBufferReader` that would exceed the buffer's marked boundary point

We're avoiding making `FastBufferReader` do bounds checking on every single read operation because this would involve a lot of overhead. However, it's still important that bounds checking be done - otherwise a malicious user could send us a message with an empty body, and we could then attempt to read past the end of the buffer even without reading any dynamically-sized data. `FastBufferReader` will provide a `VerifyCanRead()` function on it that should be called to validate that space exist to read memory in blocks - for example, before reading a set of 10 int64 values, the message processing code should call `reader.VerifyCanRead(UnsafeUtility.SizeOf(long) * 10)` as a single operation.

To enforce this, in editor and developer builds, some checking code will be included that will throw an exception if any amount of data is read from the buffer beyond the point in the buffer for which `VerifyCanRead()` has validated. This code will be stripped out in production builds, but should ensure all of our reads are safe while also keeping them fast since any failure to verify readability before reading will result in the message read getting aborted.

***For uses outside of incoming message processing, this enforcement can be disabled by passing `safe = false` in `FastBufferReader`'s constructor.*** 

`VerifyCanRead()` will be implemented by simply throwing an exception if the requested size is too large, which will be handled in the code that processes the incoming message queue. Any `MessageHandler` callback that throws `MessageOutOfBoundsException` will be ignored, and the code will move onto the next message in the incoming message queue.

### Processing Messages

At each stage of the network update loop, we iterate through the fixed-size queue for that stage, checking each queue item. At that point, the proper `[MessageHandler]`-tagged method to deserialize and handle the method is discovered via a simple array lookup on the MessageType field, and it's called with a NetworkContext composed of the network manager plus the appropriate values from the IncomingMessageItem. Array lookups are fast, and the MessageHandlerAttribute allows this array to be automatically populated via codegen, eliminating any need to manually maintain this list.

## FastBufferWriter and FastBufferReader

FastBufferWriter and FastBufferReader are replacements for the current NetworkWriter and NetworkReader. They have much the same interface, but there are some critical differences:

- FastBufferWriter and FastBufferReader are **structs**, not **classes**. This means they can be constructed and destructed without allocations if necessary.
- FastBufferWriter and FastBufferReader both wrap `NativeArray<byte>`, allowing us to take advantage of extremely fast `Allocator.Temp` and `Allocator.TempJob` allocations to create temporary buffers.
- Neither FastBufferReader nor FastBufferWriter inherits from nor contains a `Stream`. Any code that currently operates on streams will have to be rewritten.
- FastBufferReader and FastBufferWriter are heavily optimized for speed, using aggressive inlining and unsafe code to achieve the fastest possible buffer storage and retrieval.
- FastBufferReader and FastBufferWriter are intended to make data easier to debug - one such thing to support will be a `#define MLAPI_FAST_BUFFER_UNPACK_ALL` that will disable all packing operations to make the buffers for messages that use them easier to read.

## INetworkSerializable

A new `BufferSerializer` class wrapping `FastBufferReader` and `FastBufferWriter` will replace the existing `NetworkSerializer` to provide the same two-way interface as the existing `NetworkSerializer`, and is rewritten as a **ref struct** so that no allocation overhead is incurred in calling the `INetworkSerializable`'s `Serialize()` method. A **ref struct** is chosen instead of a normal struct to avoid the pitfalls associated with having mutable structs and forgetting to pass them with the ref keyword - we can manage this with `FastBufferReader` and `FastBufferWriter` because those are internal and we can rely on our ability to understand the implications of using them as structs, but our end users may be less experienced and may inadvertently pass this forward to calls that do not take the value by ref. A `ref struct` is always passed by reference rather than copied, but does have the disadvantage of being limited to the stack frame - so we can't use it for `FastBufferWriter` and `FastBufferReader` because we persist those.

`BufferSerializer` has an additional advantage (or disadvantage, depending on your viewpoint) compared to `FastBufferWriter` and `FastBufferReader`: since it's intended to be used by our end users, it performs bounds checking on every operation and throws exceptions if attempting to read or write out of bounds. This, combined with the `IsReader` if-checks, will make `BufferSerializer` slower than `FastBufferReader` and `FastBufferWriter` - but for advanced users, we can expose `FastBufferReader` and `FastBufferWriter` and allow the user to use them directly if they choose (with all appropriate warnings about them delivered).

## Custom Messages

Custom messages as they are, using buffers, no longer need to exist. Since the message handling code is part of the message definition, users have the freedom to define their own messages. Our internal code doesn't need to know anything about the messages themselves in order to send or receive them, so users can create any custom message they want the same way we define our own internal messages, and send them the same way we send ours, and their message handler will be magically invoked when the message is received. The only gotcha is the MessageType enum - since users can't extend the enum itself, we will provide a MessageType.Custom that's always at the end of the enum, and users can operate as such:

```C#
enum CustomMessageType
{
	MyMessage1,
	MyMessage2,
	Etc,
}

struct Message : IMessage
{
	MessageType GetMessageType()
	{
		return (MessageType)(MessageType.Custom + (int)CustomMessageType.MyMessage1);
	}
	
	// Other funcs and data...
}
```

The casting isn't the most elegant interface, but it still provides great extensibility.

# Drawbacks

[drawbacks]: #drawbacks

There are certainly some good reasons not to take this approach:

- The proposed serializers use unsafe code to maximize speed (we could use NativeArray's built-in methods to gain its safety checks, but would lose considerable performance in so doing)
- The proposed design is restrictive and necessitates coding in a style that avoids allocations
- The proposed design places fixed limits on various values: the size of the send queue, the number of items that may be received per frame, the size of the send buffer in both fragmenting and non-fragmenting QoS, etc.
- The proposed design uses fixed-size receive queues, which will use memory even when empty (though the size of the ReceiveQueueItem is relatively small, so the memory use should be reasonable)
- The proposed design expects the user to accurately create a `GetSizeUpperBound()` method for each message type
- The proposed design encourages (though it does not require) messages to be unmanaged types
- Since the proposed design puts the message handling inside the message definition itself, and many messages have to interact with other systems (SpawnManager, etc), the proposed design does add one more place where NetworkManager has to be passed around in order to avoid using it as a singleton.

However...

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- This approach achieves extremely fast performance
- This approach avoids memory allocations at the core layer except by way of `NativeArray`'s `Allocator.Temp` and `Allocator.TempJob`, which are orders of magnitude faster, and are only used in situations where incoming data needs to be moved from memory we don't own to memory we do, or where we need a temporary buffer to avoid reserializing.
- This approach makes our message formats clean, easy to understand, and easy to debug.
- This approach gives each message its own "home" in the form of the struct that contains it, making it easy to find all the code that serializes, deserializes, and processes the message.
- This approach serializes data exactly once, removing a current step wherein data is deserialized, changed, and reserialized

Ultimately, this approach not only increases the performance of MLAPI, but also reduces the barrier of entry for new developers and increases the readability and transparency of our codebase - all of which will have the effect of increasing our users' ability to feel confident in MLAPI.

# Prior art

[prior-art]: #prior-art

This RFC is largely inspired from personal past experience on multiple projects discovering the fastest and most effective ways to serialize data with minimal allocations and minimal copies, applied within the current MLAPI framework.

Below are some comparisons to some of our main competitors in the C#/Unity arena.

- **Unity Netcode:**
  The Unity Netcode implementation of serialization has a lot in common with this one in the underlying nuts and bolts - the implementation of DataStreamWriter and DataStreamReader is very similar to the proposed implementation of FastBufferWriter and FastBufferReader; both use raw `byte*` values under the hood and perform their operations in "unsafe" code blocks.

  Where FastBufferReader and FastBufferWriter differ is primarily at the interface level, where the FastBuffer options provide the `WriteValue<T>()` helpers for any `unmanaged` T, and in how the two handle going over capacity: the DataStream options perform checks on each operation, while the FastBuffer options hoist those checks up to the user level to allow them to be batched, and provides editor/development build-only safeguards to ensure the user is using those checks correctly. At the cost of a little more overhead on the user-side of any given serialization implementation, this allows us to reduce the number of branches we execute, which results in faster code.

  By comparison to FastBufferWriter and FastBufferReader, DataStreamWriter and DataStreamReader are very bare-bones: they contain methods for serializing primitive types, packed types, individual bits, and bytes, but contain no helpers for serializing more complex types. FastBufferWriter and FastBufferReader add built-in support for certain higher level types, such as INetworkSerializable.

  Another difference between Unity Netcode and the proposal here is that Unity Netcode separates the command struct/class definition from the serialization code - rather than having a command struct with the proper serialization and deserialization functions on it, Unity Netcode has a separate struct to contain the serialization code. We're opting not to go this route primarily because it would disallow encapsulation in our message structs - any data that has to be serialized can't be private in this approach.

  In terms of the code around serialization and deserialization - the infrastructure that invokes them - it should be obvious to say that the Unity Netcode library is not entirely comparable to this proposal, primarily because it's built around jobs and designed to operate in parallel. This is something we'd like to do, as well, but are not tackling as part of this proposal. 

- **UNet/HLAPI:**
  UNet's source repository seems to no longer exist, so I am basing this information on a fork that I hope is accurate to the state of HLAPI prior to the repository being removed.

  UNet's approach to defining messages bears some surface similarities to this proposal: you define it by inheriting a MessageBase class and implementing Serialize and Deserialize methods. There are a few differences, though.

  One difference at the interface level is that UNet's deserialization is a non-static method - meaning UNet will create a version of your struct and ask you to populate it, as opposed to giving you control over the creation of the struct as this proposal does. In practice this may not actually matter from the user perspective, and there could be arguments made to adjust our approach to match this. The big advantage we get from the proposed approach, though, is that the networking code doesn't actually have to know anything about the messages - it simply invokes a callback indexed on the message type and allows that callback to do the rest. This makes it very easy to expand the system to support new messages, as indicated in the custom messages section.

  Another difference at the interface level is that UNet's implementation does not associate the message type with the class in a static way (as this proposal does via the `GetMessageType()` method), which means the correct message type has to be specified manually at each send location. This is a minor detail, but does present an opportunity for error that this design mitigates.

  At the implementation level, one difference is that UNet does not require users to implement their own serialization methods - if the methods are omitted, UNet fills them in automatically via codegen. This is helpful, but it does come at a cost of transparency in being able to tell exactly what the code is going to be doing. This proposal does not recommend codegen due to the presence of `WriteValue()` - since unmanaged types can be trivially written and read with a single method call,  a serializer can be trivially written for those types:

  ```c#
  public void Serialize(FastBufferWriter writer) => writer.WriteValue(this);
  ```

  We could consider using a codegen solution; however, since this solution is primarily intended for internal messages, I believe it's better for us to write our own serialization code directly, because that allows our users to better understand exactly what we're doing, whereas codegen creates a black box that our users will struggle to understand (and that we ourselves may indeed struggle to understand when debugging).

  The serializers in UNet use `NetworkReader` and `NetworkWriter` classes very similar to our existing ones, except rather than containing a `Stream`, they contain a simple `byte[]` that they write to one byte at a time. An example for serializing a ulong:

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

  UNet does not appear, from what I can tell, to support any form of message batching functionality.

  One interesting thing to note is that UNet actually does avoid using fragmented QoS unless a message is large enough to need it - they don't actually provide an option to request a fragmented QoS, instead only providing Reliable and Unreliable as options, and they send the message as fragmented automatically if it exceeds the fragmentation size. Fragmentation cannot be disabled, resulting in the associated costs always being incurred.

- **Mirror:**
  Mirror uses Weaver to generate its serialization code. In usage, that results in the serialization itself feeling similar to this proposal when directly reading and writing structs, except that it's able to handle more complex types automatically as well (such as types that use strings or dynamically sized arrays). However, while convenient, this doesn't come without its costs:

  - The code that reads and writes data is completely inaccessible to the user, as Weaver injects it directly into the dll after compiling.
  - There's little to no transparency as to the format that's actually sent across the wire.
  - There's no ability to pack integers or make fields optional to optimize bandwidth usage
    - You can write your own custom serialization code as extension functions and Weaver will use those instead of serializing, but when doing so, you're ultimately falling back on the same behavior as our current code and UNet's code, with all the same drawbacks

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

  Mirror also does support message batching functionality. However, a brief analysis of the code shows this involves a series of buffer pool lookups (or allocations if it's empty) and copies in forming each batch: when sending a message, first it's serialized using MessagePacking.Pack using a PooledNetworkWriter, then the result of that is sent to the batcher, which makes a copy of it by creating another PooledNetworkWriter and writing the result to it before enqueuing it onto a Queue (which may potentially incur an allocation to make space in the queue). Then later, the queue is iterated, each PooledNetworkWriter in the queue is pulled out and its data is then copied into yet another PooledNetworkWriter, before finally passing it to the transport (which then makes another copy of it before finally delivering it to the network).

  By comparison, this proposal does the writing directly to the final buffer from the beginning, leaving space in it for any header information that can be filled in necessary with no copies required. Our buffers are created using NativeArrays, which allow for very fast creation, and we never make any copies of them ourselves - the transport may potentially do so, but we do not.

  Mirror uses the KCP protocol as defined here: https://www.programmersought.com/article/11114836523/ Determination of protocol is a transport-level feature, so this proposal doesn't suggest a specific protocol, but this is mentioned because the KCP protocol performs fragmentation automatically. As a result, Mirror, like UNet, provides only two QoS options: Reliable or Unreliable, and fragmentation is something the user cannot disable, resulting in the associated costs always being incurred.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- There are currently no unresolved questions.

# Future possibilities

[future-possibilities]: #future-possibilities

One thing that may be beneficial in the future is to create a separation at the transport level between NetworkChannel and NetworkDelivery - NetworkChannel would become an MLAPI SDK concept instead of a transport concept, and would be added to the message header for each message, while the Transport's API would be changed so sending is done using the NetworkDelivery value instead of the NetworkChannel value. This would allow for more intelligent batching - right now, even if two channels have the same NetworkDelivery behavior, we can't put them in the same buffer because that would wipe out the information about which channels are used for which messages, since that's encoded at the transport level for the whole batch. With this change, we could have one send queue per delivery mechanism - one shared queue for ReliableSequenced, UnreliableSequenced, and ReliableFragmentedSequenced so that they all remain sequenced relative to each other in the buffers, and then separate queues for Reliable and Unreliable. We would then only ever have to interrupt a buffer before it's full when switching between the three Sequenced delivery types, while Reliable and Unreliable would always be able to go in the same buffer, and different channels of the same delivery type would also be able to share buffers without losing channel identifier information on the receiving side.

Other than that, I believe this RFC represents a fairly comprehensive plan for handling serialization in a high-performance, zero-allocation approach. Aside from the possibility of incorporating jobification (which is an open question - not to be answered by this RFC - as to determine what parts of our codebase can find value from jobification and whether it will be beneficial or detrimental to move serialization to jobs after this work is done - the cost of the context switch to a job thread may actually be more expensive than the serialization itself after these changes), I have no current thoughts on future work to add to this.
