* Feature Name: `rpc-queue`
* Start Date: 2020-11-01
* RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
* Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

Ingress and egress data flow in netcode architectures frequently include synchronization mechanisms that provide further control over when and how messages/packets are processed or sent.   Placing RPCs into queues allows for the refinement of the invocation(inbound) and notification(outbound) process and provides the opportunity to develop more advanced netcode systems that could coalesce messages into specific categories/groups and/or provide additional processing opportunities for more complicated netcode architectures.

# Motivation
[motivation]: #motivation

There are several advantages to a queue based architecture vs writing directly to the OS network stack every time a section of game logic/code determines it should send some form of data (whether RPCs, state updates, etc).

1. Unifying outbound transmission allows us to feed RPCs, variable syncs, etc. through a unified interest management pipeline (including AOI)
2. Unifying outbound transmission helps developers keep state changes from different mechanisms in sync.  For example, in MLAPI today RPCs are sent immediately, but network variables update on a heartbeat.  According toit is not uncommon for MLAPI devs to employ workarounds to sew up this situation.
3. There are opportunities for consolidation / batching when messages go out as a group
4. Within the game loop process, directly sending messages at the time they are created can create an abundance of processing overhead ([Foong et al.](http://www.nanogrids.org/jaidev/papers/ispass03.pdf)) which can eat into the game developers’ precious game loop processing time (16ms) which should be allocated towards the developer(s) game logic and not the processing of network packets. However, there is also a balance between performance when it comes to large packets and latency between packets.   Larger packets that can consist of several fragmented packets within a reliable transmission protocol framework often have a slower point to point delivery time than smaller packets.  As such, there is a need (in netcode game development) to provide a mechanism for controlling when network messages are transmitted.  This control can result in the coalescing of specific network messages into groups where it can be determined when and how a network message is going to be delivered.
5. Providing a queue for inbound messages provides an alternate level of control over when the message will be processed and in the case of RPCs provides the ability to synchronize RPC oriented messages that rely upon any tickrate based synchronization methodologies/implementations.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### RPC Queue Use Cases

Currently, the RPC Queue’s primary purpose is to provide additional control over when RPC messages are sent and upon being received when they are invoked (near future feature RPC Invocation Stage).  Additionally, RPC Queues provide the opportunity to place multiple RPC messages into a single outbound packet (near future feature RPC Message Batching) that reduces the total packets received and sent.

![](0000-rpc-queue/RCPQueueOutboundHigh.png)
1. A user defined RPC within a NetworkedBehaviour derived child is initiated.
2. The ILPP generated code invokes the BeginSendXXXXRpc ( XXXX could be either Server or Client )
3. RPCQueueContainer initializes a new RPC entry in the current QueueHistoryFrame.  It then returns the stream writer to the ILPP generated code
4. RPC ILPP generated code writes serialized parameters of the RPC via the stream writer
5. RPC ILPP generated code invokes EndSendXXXXRpc method where the writer used to write serialized parameters is returned back to the RPCQueueContainer
6. The RPCQueueContainer finalizes the serialized RPC Queue entry
7. Later in the player loop (game update loop) the RPCQueueContainer processes through all outbound RPCs via RPCQueueProcessing and they are sent individually or batched.



![](0000-rpc-queue/RCPQueueInboundHigh.png)

1. The NetworkingManager enumerates through all current transport events (in the case of UNET) and adds any received RPC messages to the current inbound QueueHistoryFrame via the RPCQueueContainer (Network PreUpdate Stage).
2. Later in the PlayerLoop (Network FixedUpdate stage) the RpcQueueContainer enumerates through all inbound RPCs and as long as they are valid will invoke each RPC, NetworkedObject relative, within the associated NetworkedBehaviour component instance initiated by the sender.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Outbound RPC Data Flow Pipeline
![](0000-rpc-queue/OutboundDataFlowPipeline.png)

### Inbound RPC Data Flow Pipeline
![](0000-rpc-queue/InboundDataFlowPipeline.png)

### RPC Queue Classes
![](0000-rpc-queue/RPCQueueClasses.png)

- **RPCQueueContainer:** Manages inbound and outbound RPC queues.  Both inbound and outbound queues are byte arrays exposed as streams (i.e. currently BitStreams) and contained within a QueueHistoryFrame.  The number of QueueHistoryFrames is defined by calling the Initialize method and passing the maximum number of history frames.  The total number of frames is MaxFrameHistory + 1, where the additional frame is considered the “current frame” (i.e. it will always maintain the exact number of frames in history).
- **RPCQueueProcessing:** Currently, this class is instantiated by the RPCQueueContainer during its initialization.  It is highly probable that this class will get absorbed into the RPCQueueContainer.
- **QueueHistoryFrame:** Container class for handling the management of an RPC queue.  One QueueHistoryFrame instance per data flow pipeline (i.e. inbound and outbound yields two QueueHistoryFrames).

# Drawbacks
[drawbacks]: #drawbacks

This approach does include additional processing overhead and there could be some refactoring depending upon any future transport related work that might occur.  

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
Since there are no current discussions on batching specific packet types together at the transport layer (not recommended) and while Unet provides the ability to switch from immediate to queued mode there were other framework related design factors taken into condiseration (i.e. batched RPCs, invocation at different update stages, etc.) that made this the current best path to take.


# Prior art
[prior-art]: #prior-art



# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

This provides the opportunity to place RPC's into batched packets (containing more than 1 RPC per packet) prior to being sent as well as it offers the ability to add additional features to specify at which point in the Network Update Loop system an RPC could get invoked.  
