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
    // but must be AT LEAST the message size and AT MOST (MTU - 6) bytes.
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
}
```

Since the largest value is 2 bytes in size, the struct should be aligned to a 2 byte alignment, which should make the struct packed and the total header size equal to exactly 4 bytes, which can be sent and received using `WriteValue` and `ReadValue` without any wasted bandwidth.

In addition to the message header, there's also a batch header - the batch header is simply 2 bytes containing the total size of the batch.

## Sending Messages

As mentioned above, the interface for sending message changes from "begin add item, populate buffer, end add item" to "create struct, pass struct to SendMessage". However, there are more changes beyond just the interface to the way messages are sent:

Each client object holds on it a fixed-size `SendQueue` of FastBufferWriters associated with that client (fixed-size so as to avoid allocations, but large enough to reasonably hold all the messages that would need to be sent in a single frame, and with the size adjustable by game teams if it turns out to still be too small). At the start of each frame, this queue size is `0`, and the writer in index 0 persists from frame to frame, wrapping a `NativeArray<byte>` allocated with `Allocator.Persistent`. When we need to write a message, if the writer in `SendQueue[queueSize]` doesn't have enough remaining capacity to hold the message we're attempting to write, we increment the queue size and create a new writer wrapping a new `NativeArray<byte>` allocated with `Allocator.TempJob`. This allocator is much faster than GC allocation, approaching the speed of stack allocation, and doesn't create an object that's garbage collected (instead, it must be manually deallocated).

In the parameter list when sending to multiple recipients, we also have multiple ways in which we can incur allocation overhead. Currently we have our sending functions taking `ulong[] clientIds` as a parameter. In a lot of situations, this is resulting in extra allocations to create this list, particularly when we only have one client ID. To reduce the allocation overhead, this parameter will be replaced with `IReadOnlyList<ulong>` as a generic parameter, which will allow any indexable array type (`List<ulong>`, `ulong[]`, etc) to be used for this parameter. When new lists need to be made (for example, for a host filtering out its own local client ID), it can be made without allocations by creating a `NativeArray<ulong>` with `Allocator.Temp` and then passed in using a wrapper class that allows it to implement `IReadOnlyList<T>` (it currently implements all required methods, but doesn't name the interface in its implementation list, so a simple wrapper class can provide that and pass those things through). This will allow `NativeArray<ulong>` to be passed and used without any boxing allocations that would be incurred if we passed it as `IEnumerable<T>`. (As a note: `foreach` on an `IEnumerable<T>` will incur boxing allocations if the return type is a value type - i.e., a struct - so `IReadOnlyList<T>` is preferable because it allows iteration by index, which will not incur allocations.)

Additionally, there is a separate method for sending to a single client vs sending to multiple clients, so sending to a single client will have no requirement to create a list from a single value. As detailed below, the single and multiple sending behaviors are sufficiently different as to warrant separate implementations, anyway. 

There are three special cases for how the sending logic serializes the message.

### Sending to a single remote client

The first thing the sending logic does is call the `GetSizeUpperBound()` method on the message and compare the result against the size left in the buffer. If the buffer doesn't have enough space left for the message, the first two bytes of the writer are overwritten with the value of writer.Position, and then the writer's buffer is placed onto the client's send queue and a new buffer allocated with `Allocator.TempJob`.

**If at any time, `GetSizeUpperBound()` returns a value larger than `MTU - sizeof(MessageHeader) - sizeof(short)`, an exception is thrown and the message is not sent.**

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

Because FastBufferWriter and FastBufferReader do not check for buffer overflow on each and every write, a check is done after the call to `Serialize` - if `writer.Position` has exceeded the size of the buffer, an error is logged **and the application is immediately terminated**. Throwing an exception in this case is not acceptable, as a buffer overflow could potentially have corrupted memory anywhere in the application and is likely to cause unacceptable behavior and potentially expose security risks - the application may not continue running in this case.

### Channels

Since messages sent on different channels have different behavior requirements, the data for different channels cannot be stored in the same buffer. To handle this, each client object stores its current channel on its object next to the buffer. If any message is sent for a channel other than the one currently stored, the batch is completed and a new batch is started for the new channel (i.e., a new NativeArray is added to the send queue).

There are two reasons for doing it this way:

The first is to avoid allocations. Storing a separate writer for each channel would result in needing to keep a dictionary of writers that can be allocated on demand.

The second is to avoid regrem*(  ssions on message ordering. If each channel is allowed to have its own buffer, then messages created in order A, B, C, D could be sent in order D, A, B, C, if D is sent using a different channel than A, B, and C. Flushing on channel change ensures we don't deliver messages to the transport in a different order than the order requested by the user. 

### End of Frame

At the end of the frame, each client is iterated once to determine if its writers contain unsent messages. It first iterates the `SendQueue` up to the queue size to see if any writers exist in it, and for each writer, if `writer.Position != 2` (reserved batch header), it's finalized (the first two bytes updated to contain the batch size) and also sent to the transport. The writer's `Dispose()` method is then called (unless it's the persistent writer at index 0, which is not disposed until the client disconnects), and once all writers have been sent, the queue size is reset to 0.

## Receiving Messages

Incoming messages are processed through an incoming message queue, which contains items of this type:

```
struct IncomingMessageItem
{
	MessageType MessageType;
	FastBufferReader Reader;
	ulong SenderId;
	NetworkChannel ReceivingChannel;
	float Timestamp;
}
```

A separate queue of this type exists for each update stage, and exists as a simple array with a fixed size. If it gets full, we begin discarding messages to avoid allocating to resize the array (however, the array is reasonably sized - and, like the send queue, this size is adjustable by the user - so as to ensure that this will only happen in the case of a malicious actor sending a DoS attack). At the point of receiving, we construct a new `NativeArray<byte>` using `Allocator.TempJob` and copy the received data into it (this one-time copy is to avoid any assumptions about whether or not the data in the array segment will change between receiving and processing it). Then we create a FastBufferReader from that array segment and read two bytes from it to determine the total size of the message batch. We then iterate the following steps until we've read the full amount of data indicated in the batch size header:

- Read a value of type `MessageHeader`
- Use `header.UpdateStage` to pick the correct incoming message queue
- Set the value of the array item at the queue's current position to:
  - MessageType = header.MessageType
  - Reader = reader (since FastBufferReader is a struct, this will create a copy of the reader pointed at the current position in the buffer)
  - SenderId = clientId
  - ReceivingChannel = networkChannel
  - Timestamp = receiveTime
- Increment the queue's current position

Finally, an additional queue exists: The dispose queue. Like the incoming message queues, this queue is a fixed-size array of `NativeArray<byte>` objects sized to the combined size of all update queues. Once each buffer has been processed, it's added to the dispose queue, where it will be disposed at the end of the frame after all messages pointing into it have been handled.

### Processing Messages

At each stage of the network update loop, we iterate through the fixed-size queue for that stage, checking each queue item. At that point, the proper `[MessageHandler]`-tagged method to deserialize and handle the method is discovered via a simple array lookup on the MessageType field, and it's called with a NetworkContext composed of the network manager plus the appropriate values from the IncomingMessageItem. Array lookups are fast, and the MessageHandlerAttribute allows this array to be automatically populated via codegen, eliminating any need to manually maintain this list.

## FastBufferWriter and FastBufferReader

FastBufferWriter and FastBufferReader are replacements for the current NetworkWriter and NetworkReader. They have much the same interface, but there are some critical differences:

- FastBufferWriter and FastBufferReader are **structs**, not **classes**. This means they can be constructed and destructed without allocations if necessary.
- FastBufferWriter and FastBufferReader both wrap `NativeArray<byte>`, allowing us to take advantage of extremely fast `Allocator.Temp` and `Allocator.TempJob` allocations to create temporary buffers.
- Neither FastBufferReader nor FastBufferWriter inherits from nor contains a `Stream`. Any code that currently operates on streams will have to be rewritten.
- FastBufferReader and FastBufferWriter are heavily optimized for speed, using aggressive inlining and unsafe code to achieve the fastest possible buffer storage and retrieval.

# Drawbacks

[drawbacks]: #drawbacks

There are certainly some good reasons not to take this approach:

- The proposed serializers use unsafe code to maximize speed (we could use NativeArray's built-in methods to gain its safety checks, but would lose considerable performance in so doing)
- The proposed design is restrictive and necessitates coding in a style that avoids allocations
- The proposed design places fixed limits on various values: the size of the send queue, the number of items that may be received per frame, etc.
- The proposed design uses fixed-size receive queues, which will use memory even when empty (though the size of the ReceiveQueueItem is relatively small, so the memory use should be reasonable)
- The proposed design expects the user to accurately create a `GetSizeUpperBound()` method for each message type
- The proposed design encourages (though it does not require) messages to be unmanaged types

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

The implementation of FastBufferReader and FastBufferWriter is also similar to DOTS Netcode's DataStream class, which also uses unsafe native pointers to data stored within `NativeArray`s to maximize throughput on their reads and writes.

*TODO: Research is in progress to investigate how some of our competitors handle serialization, but I am confident this approach will be a solid solution to MLAPI's specific needs and desires around serialization.*

# Unresolved questions

[unresolved-questions]: #unresolved-questions

I do not currently have any unresolved questions around this RFC.

# Future possibilities

[future-possibilities]: #future-possibilities

I believe this RFC represents a fairly comprehensive plan for handling serialization in a high-performance, zero-allocation approach. Aside from the possibility of incorporating jobification (which is an open question - not to be answered by this RFC - as to determine what parts of our codebase can find value from jobification and whether it will be beneficial or detrimental to move serialization to jobs after this work is done - the cost of the context switch to a job thread may actually be more expensive than the serialization itself after these changes), I have no current thoughts on future work to add to this.
