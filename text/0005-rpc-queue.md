# RPC Queue
[feature]: #feature

- Start Date: `2020-11-16`
- RFC PR: [#5](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/5)
- SDK PR: [#413](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/413)

# Summary
[summary]: #summary

Ingress and egress data flow in netcode architectures frequently include synchronization mechanisms that provide further control over when and how messages/packets are processed or sent. Placing RPCs into queues allows for the refinement of the invocation(inbound) and notification(outbound) process and provides the opportunity to develop more advanced netcode systems that could coalesce messages into specific categories/groups and/or provide additional processing opportunities for more complicated netcode architectures.

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

![](0005-rpc-queue/RPCQueueOutboundHigh.png)
1. A user defined RPC within a NetworkedBehaviour derived child is initiated.
2. The ILPP generated code invokes the BeginSendXXXXRpc ( XXXX could be either Server or Client )
3. RPCQueueContainer initializes a new RPC entry in the current QueueHistoryFrame.  It then returns the stream writer to the ILPP generated code
4. RPC ILPP generated code writes serialized parameters of the RPC via the stream writer
5. RPC ILPP generated code invokes EndSendXXXXRpc method where the writer used to write serialized parameters is returned back to the RPCQueueContainer
6. The RPCQueueContainer finalizes the serialized RPC Queue entry
7. Later in the player loop (game update loop) the RPCQueueContainer processes through all outbound RPCs via RPCQueueProcessing and they are sent individually or batched.

![](0005-rpc-queue/RPCQueueInboundHigh.png)

1. The NetworkingManager enumerates through all current transport events (in the case of UNET) and adds any received RPC messages to the current inbound QueueHistoryFrame via the RPCQueueContainer (Network PreUpdate Stage).
2. Later in the PlayerLoop (Network FixedUpdate stage) the RpcQueueContainer enumerates through all inbound RPCs and as long as they are valid will invoke each RPC, NetworkedObject relative, within the associated NetworkedBehaviour component instance initiated by the sender.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Outbound RPC Data Flow Pipeline
![](0005-rpc-queue/OutboundDataFlowPipeline.png)

### Inbound RPC Data Flow Pipeline
![](0005-rpc-queue/InboundDataFlowPipeline.png)

### RPC Queue Classes
![](0005-rpc-queue/RPCQueueClasses.png)

- **RPCQueueContainer:** Manages inbound and outbound RPC queues.  Both inbound and outbound queues are byte arrays exposed as streams (i.e. currently BitStreams) and contained within a QueueHistoryFrame.  The number of QueueHistoryFrames is defined by calling the Initialize method and passing the maximum number of history frames.  The total number of frames is MaxFrameHistory + 1, where the additional frame is considered the “current frame” (i.e. it will always maintain the exact number of frames in history).
- **RPCQueueProcessing:** Currently, this class is instantiated by the RPCQueueContainer during its initialization.  It is highly probable that this class will get absorbed into the RPCQueueContainer.
- **QueueHistoryFrame:** Container class for handling the management of an RPC queue.  One QueueHistoryFrame instance per data flow pipeline (i.e. inbound and outbound yields two QueueHistoryFrames).

# Drawbacks
[drawbacks]: #drawbacks

This approach does include additional processing overhead and there could be some minor refactoring depending upon any future transport related work that might occur.  This iteration of the RPC Queue would require some refactoring in order to become fully compatible with the Unity Job System, however there were several earlier adjustments already made (i.e. pre-allocating a byte stream per inbound and outbound QueueHistoryFrame) that took this into consideration in order to make any such refactoring less time intensive.

For MMO related netcode architectures, the RPC Queue system could be further leveraged to provide IPC between server instances, client batching could need to account for relay servers, and there could be additional levels of refactoring for this type of functionality.  

On the downside, the QueueHistoryFrame currently stores all inbound and outbound RPCs into a contiguous byte stream for the current frame which requires copying each entry from the QueueHistoryFrame into a new byte stream for inbound RPC processing while providing an ArraySegment for outbound RPCs.  Currently, the inbound RPCs are copied into a single FrameQueueItem (one is created for each QueueHistoryFrame) that should only be used at the time it is invoked but never stored for future reference.  In order for a user to store a FrameQueueItem, they would need to create a copy and store it using their own method.  This all could lead to additional memory allocations and frame time consumed, which could lead to performance degradation under heavy loads (i.e. 25k-30k RPCs per frame).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Unet provides the ability to switch from immediate to queued mode which does provide a transport layer queue, however this setting is applied to every outbound packet and does not provide the ability to extract specific packet types from the buffer.  Unet’s queued buffering only helps reduce the network stack load, and Unet is slated to become deprecated.  Since there were other framework related design factors taken into consideration (i.e. batched RPCs, invocation at different update stages, etc.) and MLAPI could theoretically support other custom transports, it made sense to provide queueing capabilities at the framework layer.

 
Framework queuing allows additional levels of control over RPCs as is outlined under the motivation section above.   It also provides additional opportunities for Rpc rollback capabilities, network update loop stage specific invocation, and batching RPCs into target client id buckets prior to being sent in order to better optimize packet data distribution and utilization. Comparatively, transport layer packet buffering/queueing is “packet data agnostic” and does not provide the same range of functionality as batching at the framework layer.

# Prior art
[prior-art]: #prior-art

N/A

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

    Any additional future considerations not currently addressed in the original RFC.  

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
    
    Since MLAPI will be evoling and the way data ingresses and egresses could be further optimized/improved, those additional RPC Queue design considerations (if any) will be further defined in new RFCs. This RFC is for the initial framework of RPC queueing.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

    Unity Job system integration, memory allocation (per frame) that occurs outside of the scope of the RPC Queue system (but falls within the RPC send, receive, and invocation realm), RPC message batching, and future MLAPI UTP integration.   


# Future possibilities
[future-possibilities]: #future-possibilities

This provides the opportunity to place RPC's into batched packets (containing more than 1 RPC per packet) prior to being sent as well as it offers the ability to add additional features to specify at which point in the Network Update Loop system an RPC could get invoked.

The RPC Queue and associated data structures could be refactored to become more Unity Job System friendly, but this would most likely require unified adjustments across RPC batching, processing, and any other MLAPI framework related system that utilizes the RPC Queue system.  However, the reduced processing benefits gained from handling the sorting and queueing of RPCs into respective outbound or inbound buckets via the Job system could help reduce the overall frame-time cost associated with the RPC Queue system and be worth the time and effort.
