* Feature Name: `network collections / variables revision`
* Start Date: 2021-08-11
* RFC PR: TBA
* Issue: TBA0

# Summary
[summary]: #summary
This RFC describes some upcoming revisions to the NetworkCollections (e.g. `NetworkSet`, `NetworkDictionary`, `NetworkList`) as well as the underlying `NetowrkVariable` system.

# Motivation
[motivation]: #motivation

The current programming model allows for unsafe, race-condition-prone operations.  Its design also in its current form is not readily adaptable to future snapshot plans.  Lastly, the goals of the library to be a performant netcode system are in conflict with some of its lack of limitations


# Guide-Level Explanation
[guide-level]: #guid-level

## Permission Changes

### Current Design
Today, there is one `NetworkVariable` type.  One can instantiate it with separate read and write permissions:

* `ServerOnly`
* `OwnerOnly`
* `Everyone`


### New Design
In the new design, we bifurcate and offer 2 types of Network Variables.

* `NetworkVariable` - only the server can write to these
* `ClientNetworkVariable` - only the owning client can write to these (not even the server)

Then, for each, we have 2 read permission options:

* `Everyone` - all clients and the server will be synchronized
* `OwnerOnly` - only the owning client and server will be synchronized

**Notes:** 

* We expect to continue to refine the way clients communicate state to the server via input passing or other schemes.  The `ClientNetworkVariable` scheme was chosen to be a simple and familiar way to pass state to the server in this period while we develop this mechanism further.  It is intentionally isolated in its own class so that if we have a different mechanism later we can deprecate and replace the type vs mutate the core `NetworkVariable` type.

* As we move towards the snapshot tick-based model we do not plan to snapshot `ClientNetworkVaraible` data.

* The `Everyone` mode was intionally removed.  Not only did this have the same issues as the client-writable mode, but also it led to race conditions when multiple clients and the server could write to a variable.

* The collection network variable types (`NetworkSet`, `NetworkDictionary` and `NetworkList`) intentionally do not have a client-writable mode.

## Allowable NetworkVariable types
Today, there is no restriction on what can be stored as `T` in `NetworkVariable<T>`.  To enable targeted performance gains, future jobification and more, `T` will be constrained to being  `unmanaged`.  Most notably, `class` types may no longer be backed.

## Change in network variable writing interface

The NetworkVariable class has a `Value` getter/setter.  Previously, one had to be careful about changing the elements of classes backed by network variables.  But with this change, since only unmanaged types may be stored in all the network variable types, one cannot do patterns like this:

```
public struct TestStruct : INetworkSerializable
{
    public uint SomeFloat;
    public bool SomeInt;

    public void NetworkSerialize(NetworkSerializer serializer) { ... }
}

NetworkVariable<TestStruct> structVar = new NetworkVariable<TestStruct>(new TestStruct());
structVar.Value.SomeFloat = 1.23f; // <<------ ERROR, can't modify temporary value
```

Instead, if you want to update a value of a struct, you must assign `Value` a new copy of the struct.

```
var tmpVal = structVar.Value;
tmpVal.SomeFloat = 1.23f;
structVar.Value = tmpVal;  // Dirtying properly called here
```
 
## Container Operations
`NetworkList`, `NetworkSet` and `NetworkDictionary` will continue to support all the operations that `IList`, `ISet` and `IDictionary`.  This was because these classes used the (managed) corresponding containers.  However as we move to using native containers for the backing storage, we may use interfaces that more match what those containers can / can't do.

## Container Max Sizes
Before this change a user could simply declare a container and add an arbitrary amount of elements.  This creates at least 2 complications; one, it works against the upcoming goals of having MTU-sized snapshot packets.  Second, having network variables be variable length gets in the way of performance-oriented, allocation-free schemes we have planned.  To support this, a user must specify a maxmimum capacity when instantiating a container.

# Drawbacks

- Having two types to enable Client writing support makes things a bit more complicated
- If one wanted a scenario where a client and a server both could write to the same variable it was very easy, but it was also very inconsistent
- The new "`Value` get operation equates so dirtying" operation will have more overhead, especially if users do many more reads than writes on a network variable.

# Unresolved questions
* the final interface for the collections is still being developed

 
# Future possibilities / Work to do
* The internal serialization system will be improved to employ other serialization work going on now
* The batching scheme for updating groups of changes in a conatiner hasn't been studied deeply enough but appears to have room for optimization