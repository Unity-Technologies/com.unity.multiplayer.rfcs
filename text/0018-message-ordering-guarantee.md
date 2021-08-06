- Feature Name: `message_ordering_guarantee`
- Start Date: 2021-06-02
- RFC PR: [RFC#18](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/18)
- Issue: [MLAPI#700](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/issues/700)

# Summary

This feature adds a guarantee of ordering between all types of messages, that, unless the user requests otherwise, all messages sent over a **reliable sequenced** channel will be received and processed by the recipient process in the order they were sent by the sender process. It guarantees that MLAPI itself will not internally change the order of any messages, that they will all reach the transport level in the same order they were initiated/requested. This RFC does **not**, however, make any guarantees on ordering for messages sent on non-sequenced channels, nor does it provide any new enforcement to force any messages to be sent on sequenced channels.

# Motivation

Prior to implementation of this feature, there were four mechanisms by which messages are sent:

- Directly, via `InternalMessageSender.Send()`

- Through the internal message queue, via `RpcQueueContainer.AddToInternalMLAPISendQueue()`

- Through the RPC queue, via `RpcQueueProcessor.SendFrameQueueItem()`

- As batches, via the `RPCBatcher` class

The last two of these are mutually exclusive, determined by whether or not batching is enabled, which means that at any given time, there were three separate message delivery mechanisms in use.

The first mechanism, `InternalMessageSender.Send()`, was processed and placed on the wire immediately. The queueing systems (whichever two are in use) were batched up and delivered at the end of the frame, but the relative order of messages between the two queueing systems was not preserved - the RPC queue was always sent before the internal command queue.

Because there wasn't a strict order guarantee between messages, in order to ensure certain things happen before others (such as spawning objects before calling RPCs on them), certain internal commands were given specific stages in which they must be processed, and would always be processed in those stages. In order to resolve an immediate need, the internal message queue was created to allow those specific commands to specify the desired update stage in which to be executed; while this resolved the immediate need, it was still in the work-in-progress stage and thus provided an incomplete solution. Internal commands were inconsistent about which ones use the direct send and which use the internal queue - for example, `Spawn()` used the queue, while `ChangeOwnership()` sent directly. Thus, if you used those two on the same object in the same frame, the `ChangeOwnership()` command would be sent before the `Spawn()` command, causing it to be received and processed by the client before the object had been created.

# Guide-level explanation

To ensure that all commands sent arrive in the same order relative to each other, including relative order between commands and RPC calls (such as ensuring an ownership change arrives before or after an RPC whose function is dependent on the ownership), all messages will be passed through `MessageQueueProcessor` (formerly known as `RpcQueueProcessor`), which will determine based on whether batching is enabled whether to process it using `MessageQueueProcessor.SendFrameQueueItem()` or to pass it to the `MessageBatcher` (formerly `RpcBatcher`) class. Internal commands are wrapped in the same structure as RPCs, allowing them to specify the update stage at which they should be processed.

When sending any command or RPC, the UpdateStage for it to be processed by the recipient will **default to the current update stage on the sender,** as defined by `NetworkUpdateLoop.UpdateStage`, unless the sender overrides that value. What this means is that with default behavior, the recipient will process all messages (both internal commands and RPC messages) in the same order they were requested by the sender, unless the sender specifies otherwise, which in practice should mean that if it works on the sender side, it should work on the recipient side as well.

Example:

```C#
void FixedUpdate()
{
    GameObject obj = GameObject.Instantiate(prefab, location, rotation);
    NetworkBehavior networkBehavior = obj.GetComponent<NetworkBehavior>();
    networkBehavior.Spawn();
    // Requires server ownership at this point, ChangeOwnership() must not have been processed yet
    networkBehavior.SendAfterSpawnClientRPC();
    networkBehavior.ChangeOwnership(newOwnerClientId);
    // Requires client ownership, ChangeOwnership() must have been processed before this
    networkBehavior.SendAfterChangeOwnershipClientRpc();
}
```

In this example, without strict ordering, the `ChangeOwnership()` call may happen first, followed by `Spawn()`, followed thereafter by the two RPCs. Such an ordering would result in errors from the `ChangeOwnership()` call (due to the object not yet existing) and the second RPC (due to the ownership not having been changed). Ensuring that all messages arrive in the same order they were sent also ensures that they are called in the order required to process these four operations successfully.

Another example:

```C#
void FixedUpdate()
{
    SendFixedUpdateClientRpc();
}

void Update()
{
    SendUpdateClientRpc();
}
```

In this example, the client will execute `SendFixedUpdateClientRpc()` during the `FixedUpdate` stage, and `SendUpdateClientRpc()` during the `Update` stage - this ensures behavior between the sender and the recipient is somewhat more predictable, as logic that occurs on one side during `FixedUpdate` will implicitly occur on the other side during the same stage unless requested otherwise. 

Likewise, the same is true of internal commands:

```C#
GameObject m_obj;

void Update()
{
    m_obj = GameObject.Instantiate(prefab, location, rotation);
    NetworkBehavior networkBehavior = m_obj.GetComponent<NetworkBehavior>();
    networkBehavior.Spawn();
}

void LateUpdate()
{
    ulong newOwnerClientId = GetNewOwnerClientId();
    NetworkBehavior networkBehavior = m_obj.GetComponent<NetworkBehavior>();
    networkBehavior.ChangeOwnership(newOwnerClientId);
}
```

In this example, because `Spawn()` is called during `Update()` on the server, and `ChangeOwnership()` is called during `LateUpdate()` on the server, the same will be true on the client - the object will be spawned on the client during the `Update` stage, and its ownership changed during `LateUpdate`.

# Reference-level explanation

There are two main refactors involved in implementing this change:

1. The first and largest refactor is a change in the way internal messages are sent to ensure they use the same mechanism as RPCs, and the corresponding refactor of that mechanism to ensure it's sufficiently generic to support all message types.

2. The second, smaller refactor is changing the default value of the UpdateStage value in the send parameters so that it reads from the current update stage rather than defaulting to 0 (Update).

The vast majority of the work is in the former refactor, but the biggest user-facing change is the latter one.

In order to complete this refactor, a few things need to happen:

- All RPC-related classes will need to be renamed from Rpc\* to Message\*. This accounts for RpcFrameQueueItem, RpcQueueContainer, RpcQueueProcessor, RpcBatcher, and RpcQueueHistoryFrame.

- All internal commands must be sent through `RpcQueueProcessor.SendFrameQueueItem()`. As a consequence, this also means all internal commands must now serialize in a format that can be received via `RpcFrameQueueItem` structs.

- In order to minimize allocations, `RpcFrameQueueItem` will not actually be created on the sender side. Instead, the values necessary to serialize it will be passed as function parameters, just like they currently are in `BeginAddQueueItemToOutboundFrame` and `EndAddQueueItemToOutboundFrame`. Additionally, `RpcFrameQueueItem` has some `class` members within it that don't appear to be used on the sender side, so a new `SendFrameQueueItem` can be introduce containing only the used fields to reference queue data for send processing without needing to allocate those unused class members.

- `RpcQueueContainer.QueueItemType` now contains only three values: `ClientRpc`, `ServerRpc`, and `InternalCommand`. `CreateObject` and `DestroyObject` are removed, and are now a subcategory of `InternalCommand` <sup>[See Question #1]</sup>

- `MessageQueueProcessor`'s (formerly `RpcQueueProcessor`) `m_InternalMLAPISendQueue` is removed, along with all functions in `MessageQueueProcessor` and `MessageQueueContainer` that reference it.

- On the receiving end, the handling of all internal messages using the previous values from `NetworkConstants.cs` will instead handle a new `INTERNAL_COMMAND` constant, as it will need to pull the encapsulated message from the queue and process it before passing the wrapped content to the existing logic

- Since structs can't have parameterless constructors, `NetworkUpdateStage` will be adjusted to appear as follows:
  
  ```C#
  public enum NetworkUpdateStage : byte  
  {
      Unset = 0, // Default
      Immediate = 1, // Bypasses the queueing system
      Initialization = 2,  
      EarlyUpdate = 3,  
      FixedUpdate = 4,  
      PreUpdate = 5,  
      Update = 6, 
      PreLateUpdate = 7,  
      PostLateUpdate = 8  
  }
  ```
  
  The reason for this change is because the `UpdateStage` value in the struct will default to 0. If `Update` is represented by 0, then we won't be able to differentiate between the user not providing a value (in which case we need to set it to the value of `NetworkUpdateLoop.UpdateStage`) or the user intentionally providing a value of `Update` (in which case we _must not_ change it).

- An additional value is also added to `NetworkUpdateStage`, as seen above: `NetworkUpdateStage.Immediate`. When this value is set, both the sending and receiving sides bypass the message queue system entirely - messages are sent immediately by the sender, and processed immediately on receipt by the receiver. *Note, of course, that ordering guarantees are only provided between messages that have the same `UpdateStage` value - Two Immediate messages will be ordered correctly, but an Immediate sent after an Update will likely get processed before the Update.*
  
  This has the added benefit that RPCs can now also be sent using `NetworkUpdateStage.Immediate`, which exposes to users the ability to bypass the queueing system if they need to.

- `__beginSendServerRpc` and `__beginSendClientRpc` will then check for `*RpcParams.Send.UpdateStage == NetworkUpdateStage.Unset` and if true, will set `*RpcParams.Send.UpdateStage = NetworkUpdateLoop.UpdateStage` to trigger the recipient to execute the RPC at the same stage as the sender.

- Once this is done, all hard-coded stage requirements for internal commands (such as Spawn only processing during Update and Despawn only processing during PostLateUpdate) should be modified so that they occur at whatever stage they were called in, the same as the default behavior for other internal commands and RPCs.

- Because all messages now use the queuing system, the enum `RpcQueueContainer.QueueItemType` now becomes redundant with the constants in `NetworkConstants.cs`. Since enums generally provide better type-safety and extra functionality over constants, the redundancy is removed by eliminating `NetworkConstants.cs` and using the enum for all use cases. The enum is renamed to `MessageQueueContainer.MessageType`.

- Because internal messages are often not associated with a `NetworkObjectId` or a `NetworkBehaviorId`, the message header is adjusted such that `NetworkObjectId` and `NetworkBehaviorId` are now considered part of the message body, rather than the message header, for RPCs.
  
  The header format is now:
  
  ```C#
  MessageQueueContainer.MessageType MessageType; // 1 byte
  NetworkUpdateStage UpdateStage; // 1 byte
  ```
  
  As an added benefit, the current message parsing code logic that passes over the object and behavior IDs to read the update stage, then resets the position of the buffer, is no longer necessary, as the updateStage is right at the top of the buffer and can simply be read directly. The value can then be cached onto the `MessageFrameQueueItem` (now renamed to just `MessageFrameItem` since it will not always come from the queue thanks to `NetworkUpdateStage.Immediate`) and then the buffer position no longer needs to be reset, offering a small optimization to the message processing logic.

# Drawbacks

The biggest drawback of this implementation is the standardization of the packet format for all types of messages. This is *mostly* a positive thing from a standpoint of maintainability and ability to expand or refactor the system further in the future, but does potentially involve sending a few extra bytes that may not be required for all internal commands.

# Rationale and alternatives

The primary rationale here is to ensure predictable behavior on both sides of the network. One argument in favor is that our SDK allows for (and *should* allow for) `Spawn()` and `ChangeOwnership()` as separate operations, but it is non-obvious that a period of time exists after calling `Spawn()` during which calling `ChangeOwnership()` is an error. It is also not trivial for the code on the sender side to detect when this is the case in order to surface that error to the user. As a result, the lack of strong ordering between these two operations creates a pitfall for our users that is so well-hidden beneath branches and twigs as to make it nearly impossible to know it exists until you have already fallen into it. Ensuring that these two operations are both sent using the same mechanism, and thus both sent in a predictable order, removes this pitfall and makes the operation of `ChangeOwnership()` function as intuitively expected in all use cases.

Another rationale, and the reason for removing the internal message queue entirely instead of simply changing `ChangeOwnership()` to use the same internal queue mechanism as `Spawn()`, is that (in addition to allowing RPCs and internal commands to be interleaved rather than placed in separate queues) the existing RPC queue has a useful function in it that the internal message queue does not: the ability to specify the update stage at which the operation should be executed by the recipient. Being able to specify that stage is important to ensure order between RPCs and internal commands - if an object is spawned  during the `EarlyUpdate` stage on the server and an RPC is then invoked for that same stage, the current behavior of processing the `Spawn` operation during the `Update` stage would result in the order of execution on the client being different than the server *even though they appear in the correct order in the queue*. In this situation, it's important that the `Spawn` operation send its correct update stage, so that it will process during `EarlyUpdate` before the RPC is executed.

Finally, this change has the benefit of consolidating all of our messages into a single mechanism. This is not only good for maintainability, but as we look forward to future features of the snapshotting system, this consolidation should serve as a boon for any attempt to integrate command messages into the snapshotting system - rather than having to refactor or integrate three separate mechanisms, we will only need to deal with the one.

As **alternatives** go, there are four primary alternatives we discussed:

1. Make all internal commands send immediately, as does `ChangeOwnership()`. Doing this would remove the ability for Spawn and Despawn to have any control at all over which stage they are executed in on the client side, which is likely to resurface bugs this behavior previously fixed.

2. Make all internal commands use the internal send queue, as do `Spawn()` and `Despawn()`. This would work as a partial fix - it would allow internal commands to be ordered with each other, but would still leave a disconnect between the ordering of internal commands with respect to RPCs.

3. Perform part 1 of the refactor without part 2 - that is, consolidate the message sending, but leave the RPCs defaulted to the Update stage. This has less chance of causing user friction, at the cost of accepting the potential that the default update stage of `Update` for RPCs could create edge cases that are not intuitive to the user where they would be required to set the `UpdateStage` value in their send parameters. As using send parameters at all is somewhat more advanced than the most basic usage of RPCs, the assertion is made that the fewer surprises we can allow into the basic usage, the more intuitive that usage will be; the `UpdateStage` parameter will remain for advanced users, but an intelligent default value should diminish the need for users to interact with that parameter.

4. We discussed integrating this change into the current snapshotting system proposal. There are two reasons this isn't currently achievable:
   
   - The snapshotting system, as currently proposed, only sends delta value updates and doesn't have a mechanism for carrying commands, and
   
   - The snapshotting system, as currently proposed, sends all of its messages unreliably, which introduces the possibility of edge cases where dropped packets unexpectedly change the order of commands.
   
   However, as mentioned above, when the snapshotting system is better realized to convey these sorts of messages, the consolidation of the message sending logic into a single mechanism should still be of benefit to the integration with snapshotting, as it will reduce the number of different types of messaging logic the snapshotting system needs to integrate down to just one.

# Prior art

This modification is fairly MLAPI- and Unity-specific. I'm unaware of any existing implementations that attempt to solve this particular problem.

# Unresolved questions

- None

# Future possibilities

As mentioned above in the rationale section, there is a known intention to integrate commands into the snapshot system. While this RFC doesn't attempt to address that yet, the consolidation of message sending mechanisms can be seen as a first step toward realizing that.
