* Feature Name: `interest-management`
* Start Date: 2020-05-11
* RFC PR: TBA
* Issue: TBA0

# Summary
[summary]: #summary
This RFC proposes an interest management system to help MLAPI support greater scale in games.  It is sometimes conceptualized as an 'AOI' (Area of Interest) system, but goes beyond that in also providing support for later elements such as prioritization.

# Motivation
[motivation]: #motivation

In the existing MLAPI Interest system, every networked object (i.e. `NetworkedObject`) stores a list of all clients that can see that object (in the variable `observers`).  In the default case, every object stores a list of all connected clients.  This creates some scalability problems

* Every client has to be checked against every object
* Storage: every object has to store a list of all clients
* Updating: every time a client joins / leaves, every object must be scanned and modified

For example, a leading FPS game begins with around 100 clients and 50,000 replicated objects.  With the current scheme, one would need to store 20 MB of client / object data and do 50,000*100 = 5 million client/object checks each tick.

## Desired Outcome

What we instead would like to achieve:

* We want to minimize the computational per-frame work.  As the library walks through its connected clients, we want a system that will efficiently answer the question 'which objects should this connection see'
* Therefore, we desire an interface that takes in a connection and returns the objects for that client
* We want the way this question is answered to be user-extensible to support all kinds of different scenarios.  Domain decomposition / geometric / AOI is very common, but we also want developers to be able to come up with creative, game-specific **schemes** that have nothing to do with spatial relationships
* We want the aforementioned schemes to be composable.  For instance, a user should be able to have both a AOI scheme and a "always synchronize players who are in the same cohort" scheme and have the results from each efficiently combine
* Each scheme should be free to decide how to implement relationships; so long as the scheme can take in a client and return a set of objects it should be free to come up with that answer using whatever storage and computation it wants
* Each scheme can therefore decide at what cadence it does its work.  A scheme can choose to dynamically come up with an answer every frame.  Or, it could return cached results each time except every 30th frame.
* We also want a means for code outside the scheme to be able to trigger an update.  That is, there are times when it's best for the scheme itself to know when it's time to do things like the cache/compute decision, but then there are other times when game code outside the Interest System knows better that it's time for the scheme to adjust itself

# Guide-Level Explanation

The interest system consists of 3 main components:

## Components
### 1. `InterestManager`

There is one and only one `InterestManager` per `NetworkManager`.  It is defined / setup / torn down just like the many other "managers" in `NetworkManager`.  It has the following jobs:


* It maintains a list of `InterestNodes` which implement the 'schemes' described above
* Via the `QueryFor()` method, it is the entry point for getting the "which objects for this client" answer.  It is designed such that, if a user does not implement any schemes it produces the same result as if there was no interest system (e.g. all clients get all objects)
* It listens to all MLAPI object spawns / de-spawns so that it can tell associated the `InterestNodes` (see below) it manages about them.  Logic for this lives in `AddObject()` and `RemoveObject()`, which are linked to via `NetworkSpawnManager's`  `SpawnNetworkObjectLocally()` and `OnDespawnObject()` methods respectively

### 2. `InterestNode`

The core interest scheme functionality happens here.  Its jobs are to:

* return its results in for the per-connection queries that come in from the `InterestManager`
* respond to `AddObject` / `RemoveObject` from `InterestManager`.  How it responds to this is totally up to the node, as illustrated in examples below
* respond to `UpdateObject` calls when came code wants to make specific triggers to the node
* Multiple `InterestNode`s can be instantiated of the same kind with different settings.  For instance, one could define a Radius-based node with a value of 10 for shotgun shell casings (only interesting if you're very close by) but 100 for rockets.

### 3. `InterestSettings`
This is the least-developed component and doesn't have any settings in it as of this writing, but it is a placeholder for per-`InteresNode` settings that are not specific to any particular `InterestNode`.  For later when prioritization is added this is where we can place data that prioritization system uses when considering results from this node.


## How the `InterestNode`, `InterestManager` and Prefabs relate & connect

Let's explore this by way of an example


relationship between nodes and their objects through 

default, pass-through