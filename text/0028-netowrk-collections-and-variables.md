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

Today, the library has 3 levels of permissions

### Old: Read & Write Permissions

* `ServerOnly`
* `OwnerOnly`
* `Everyone`


### New: Read Permissions
* `Everyone`
* `ServerOnly`

Going forward, Network Variables will only be (directly) writable by the server.  Previously, a client could change state on the server by writing to its local network variable instance.  As we move towards the snapshot tick-based model this free-form mechanism becomes intractable to support at least in its current form.  With this change, when a client wants to cause a state change it will need to use a different means such as sending an RPC.  

Hence, with this release there are only *Read* permissions.  Either a variable can be read by everyone, or the specific owner of the variable.  The server can always read all variables.

The `Everyone` mode has also been removed.  Not only did this have the same issues as the client-writable mode, but also it led to race conditions when multiple clients and the server could write to a variable.

## Change in network variable writing interface

Currently the NetworkVariable class has a `Value` getter/setter.  When the `get` method is called, the internal, current value is returned.  When the `set` method is called the intention is that the value change is wrapped by code to dirty the variable and cause it to be updated.  This works great for scalar types, e.g.

```
NetworkVariable<float> someVar = new NetworkVariable<float>();
someVar.Value = 1.23;

```

But not so great for struct access.

```
public class TestClass : INetworkSerializable
{
    public uint SomeFloat;
    public bool SomeInt;

    public void NetworkSerialize(NetworkSerializer serializer) { ... }
}

NetworkVariable<TestClass> TheClass = new NetworkVariable<TestClass>(new TestClass());
TheClass.Value.SomeBool = false;
TheClass.Value.SomeInt = k_TestUInt;
```

This code in fact causes nothing to happen on the server as it isn't in fact calling `set`.  Users must remember to call `TheClass.SetDirty(true)`.  

Moreover, the above code doesn't work at all with struct member access.  With structs, the only way to change value is to assign a new copy.

Instead, with this RFC network variables will no longer have a `set` on the `Value` property.  The `get` property will return the value by reference and mark the variable as dirty.  There will be mitigation around avoiding the fact that dirty-ing will happen more often to avoid unnecessary updates.

## Allowable NetworkVariable types
Today, there is no restriction on what can be stored as `T` in `NetworkVariable<T>`.  To enable targeted performance gains, future jobification and more, `T` will be constrained to being  `unmanaged`. 


## Container Operations
Before this change, `NetworkList`, `NetworkSet` and `NetworkDictionary` supported all the operations that `IList`, `ISet` and `IDictionary` supported.  This was because these classes used the (managed) corresponding containers.  As we move to using native containers, we may use interfaces that more match what those containers can / can't do. 

## Container Max Sizes
Before this change a user could simply declare a container and add an arbitrary amount of elements.  This creates at least 2 complications; one, it works against the upcoming goals of having MTU-sized snapshot packets.  Second, having network variables be variable length gets in the way of performance-oriented, allocation-free schemes we have planned.  To support this, a user must specify a maxmimum capacity when instantiating a container.

# Drawbacks
The old method where clients could simply write to a network variable is simpler, especially when trying to employ a client-authoritative scheme.  One way to think of network programming is that each design is a combination of tradeoffs around ease of use, synchronization consistency and performance.  While we seek to maximize all of them, the pre-RFC design maximizes ease of use at the expense of consistency (different clients & the server can have different values at different times with no way to link them) and performance (network variables rely on being ordered & reliable).  The upcoming snapshot system addresses consistency and performance, but at least at this point, it is intractable to do that with the current (extremely easy) interface on NetworkVariables.

The new "`Value` get operation equates so dirtying" operation will have more overhead, especially if users do many more reads than writes on a network variable.

# Unresolved questions
* the final interface for the collections is still being developed

 
# Future possibilities / Work to do
* The internal serialization system will be improved to employ other serialization work going on now
* We may try and replace the `INetworkVariable` interface scheme with a base class
* The batching scheme for updating groups of changes in a conatiner hasn't been studied deeply enough but appears to have room for optimization