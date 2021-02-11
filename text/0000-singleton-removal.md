- Feature Name: `singleton-remove`
- Start Date: 2021-02-11
- RFC PR: [RFC#11](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/11)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

This change removes the `NetworkingManager.Singleton` static, as well as the majority of other static state in MLAPI. 

# Motivation
[motivation]: #motivation

The purpose of the change is to allow multiple instances of the MLAPI Networking Stack to co-exist in the same process. This enables:
- in-process servers. This allows developers to architect single-player and host-mode versions of their multiplayer game identically to the dedicated-server version of the same game.
- integration testing. Both games that use MLAPI and MLAPI itself can write full server/client tests that run entirely in the scope of a single process. 
- clients talking to multiple servers simultaneously. This is expected to be a requirement for MMO support. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

MLAPI supports having multiple `NetworkingManager`s running concurrently. There are a few things to keep in mind when doing so:
- The simplest case is to only have one `NetworkingManager` per scene. This is what you do to support an in-process server: load a "server" version of your scene, and then call `StartServer` on the `NetworkingManager` in it. All "server" physics calculations must then be performed via the `PhysicsScene` associated with the server scene.
- If you have multiple `NetworkingManager`s in one scene (for example, to talk to two different servers at once), then you should take care that all placed `NetworkedObject`s be wired to the `NetworkingManager` they are associated with via a serialized field.
- `NetworkedObject`s spawned by a `NetworkingManager` will always be bound to the `NetworkingManager` that created them, and be placed in the same scene as their parent `NetworkingManager`.
- If launching an in-process server, be sure to only call `StartClient` after the `OnServerStarted` callback has fired. 
- Currently only the ENet Transport is supported. UNet shares a common static at the Unity level (`NetworkTransport`), which prevents it from being fully separated.
- We have removed the old `NetworkingManager.Singleton` method. For migrators, if you are writing code that in a component sitting on a `NetworkedObject`, you can simply get the `NetworkedObject`s `NetManager` instead. For other code, you will need to adopt your own strategy (for example, finding the `NetworkingManager` in the same scene as yourself via [`GameObject.FindWithTag`](https://docs.unity3d.com/ScriptReference/GameObject.FindWithTag.html)).
- Note that the MLAPI Profiler doesn't support profiling multiple `NetworkingManager`s simultaneously. If run when multiple `NetworkingManager`s are active, it will profile the first `NetworkingManager` it finds.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`NetworkingManager.Singleton` is removed. `SpawnManager`, `InternalMessageSender`, `InternalMessageHandler`, `NetworkSceneManager`, and `BitReaderPool` have all been made non-static, and hang off the `NetworkingManager` that owns them. 

`NetworkedObject`s have a `NetManager` field that is stamped on them at their moment of creation (when spawned by MLAPI), or should already be set in editor (for placed objects). If still unset by the time `NetworkedObject.Start` runs, the `NetworkedObject` takes the first `NetworkingManager` it finds in the same scene as itself.

A few things have been left static. `BitWriterPool` doesn't make use of `NetworkingManager` at all, and has consequently been left shared. The `LogLevel` has also been left static, as it is a bother to thread through into some of the places where it is used (however, I can see the argument for biting the bullet and doing so, as it would be nice to see whether log lines were generated from the server or client, respectively).

For editor code, there is a new static method: `NetworkingManagerEditor.GetAnyNetworkingManager` that just wraps a [`FindObjectOfType`](https://docs.unity3d.com/ScriptReference/Object.FindObjectOfType.html) call. This is the replacement for editor code that needed to operate on "the NetworkingManager" (back when there was only one). 

# Drawbacks
[drawbacks]: #drawbacks

Singletons are convenient. For code that operates in the context of a `NetworkedObject`, this isn't a loss, but for any other code, it requires more thought. For games that have multiple `NetworkingManager`s (particularly in the same scene), there is no recourse to making users think harder about _which_ `NetworkingManager` they want to interact with. However, for games that don't make use of multiple `NetworkingManager`s, it creates a pain point with no upside. I've considered if we should just leave the `NetworkingManager.Singleton` in place (with the first `NetworkingManager` to claim it "winning"), just to maintain continuity for users that don't need this feature. However, there is a risk of this creating bugs and confusion for users who do start down the path of multiple `NetworkingManager`s.

This also makes the editor facilities feel a bit more halfbaked. It would be nice if the MLAPIProfiler could provide feedback for all the `NetworkingManager` instances that are running, but this will entail more work. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There is no alternative design that covers all use cases of removing MLAPI singleton dependence. An architecturally similar approach to supporting host-based games is to have the client start up a separate, out-of-process server, but this will increase the minspec of the Host machine, because it effectively doubles its memory requirements. There is no good solution to talking to multiple servers simultaneously via MLAPI. It may be possible to live without this feature when writing MMOs, as a common pattern is to have the client only talk to a single gateway server in production; however, forcing this pattern onto development is undesirable, as configuring a local gateway server can be nontrivial. 

# Prior art
[prior-art]: #prior-art

There is no specific prior art to cite, but avoiding static global state is a common software development recommendation, as it reduces system isolation and flexibility.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should we maintain a `NetworkingManager.Singleton` for the convenience of people not using this feature?
- Should we force `NetworkLog` to need a `NetworkingManager` context? This makes it less convenient, but it means the logs could append `[SERVER]` or `[CLIENT]` tags. 
- Are there any "must-do" changes to how I've handled MLAPIProfiler in particular, or is the current compromise of profiling the first-found `NetworkingManager` acceptable?

# Future possibilities
[future-possibilities]: #future-possibilities

This is a fairly self-contained feature (really somewhere between a feature and a code refactor). While it enables new use patterns mentioned above, I don't have anything specific to bring up here.