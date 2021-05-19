- Feature Name: network-time
- Start Date: 2021-04-30
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0014](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0014)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

This RFC proposes a new network time system to replace the old time system present in MLAPI.

# Motivation
[motivation]: #motivation

We need a time system to address the problem of running multiple peers connected to each other in controlled manner while keeping latency in mind. The time system will act as a basis to build other systems such as snapshots, commands and prediction on top.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Deprecations / Removals

### NetworkManager Settings

`Receive Tickrate`, `Network Tick Interval Sec`, `Max Receive Events Per Tick Rate`, `Event Tickrate` settings in NetworkManager are getting removed.

![tickrate deprecations](0014-network-time/tickrate-deprecations.png)

- `Network Tick Interval Sec` will now be given via the new `Tickrate` field.
- `Receive Tickrate` is being removed. There is no replacement for this. MLAPI will now poll the receive queues at the beginning of every tick.
- `Event Tickrate` will now be given via the new `Tickrate` field.
- `Max Receive Events Per Tick Rate` will be removed. There will be no functionality to restrict the amount of events in MLAPI anymore as we move from an event to a snapshot based system.

The default value for `Tickrate` will be `60` instead of `64`. A tickrate of 60 is more consistent with industry standard tickrates. Note that this rate is different than the default tickrate for FixedUpdate in Unity which is 50.

### NetworkTime

`NetworkTime` will be removed from MLAPI. As explained later in the RFC there are some flaws in the concept of a single `NetworkTime` so we will be replacing it with other concepts instead. For a quick fix of existing code instead of:
```csharp
var time = NetworkManager.Singleton.NetworkTime;
```
use
```csharp
var time = NetworkManager.Singleton.PredictedTime.Time;
```

## New Features

### NetworkTime

This RFC will introduce new features which can be used to interact with time in a networked manner.

### Server Time
A client receives regular state updates from the server containing data about the game state. This state could for instance be a position. Because time passes as packets travel between the server and clients this state is in a different position in time then the state of a locally controlled player character. `ServerTime.Time` can be used to interact with logic in this time space.

For instance the following code could be used to displaying a moving platform which is aligned with the positions of all player objects from other players or server controlled NPCs:

```csharp
public class MovingPlatform : MonoBehaviour
{
    // Note moving in Update is only recommended for visual-only, non-networked objects.
    public void Update(){
        var positionX = Mathf.PingPong(NetworkManager.Singleton.ServerTime.Time)
        transform.position = new Vector3(positionX, 0, 0);
    }
}
```
<img src="0014-network-time/platform_1.gif" width="640" height="360">

### Predicted Time
When moving your own player character locally, the character is ahead of time compared to all other NetworkObjects. This is the case because any changes to state will first need to be sent to the server and then back to all clients. `PredictedTime.Time` can be used to access this time. Most of the time `PredictedTime` should be used instead of `ServerTime` when implementing gameplay code.

Example of using Predicted Time:

```csharp
public class NetworkChat : NetworkBehaviour
{
    [ServerRpc(RequireOwnership = false)]
    SendChatMessageServerRpc(string message, float time)
    {
        Debug.Log($"Received message: {message} at time: {time}");
    }

    public void Update()
    {
        if (Input.GetKeyDown(KeyCode.Enter))
        {
            SendChatMessageServerRpc("Hello World!", NetworkManager.Singleton.PredictedTime.Time);
        }
    }
}
```
*note: this is a stupid example :)*

### Time on the Host/Server
Since there is 0 latency when making a change to an object on the Host/Server `PredictedTime` will always be the same as `ServerTime` on the Server/Host. When implementing gameplay code using `PredictedTime` is recommended as it allows to potentially re-use the code on a client.


### Network Ticks

MLAPI sends, receives and processes data at fixed tick intervals. It is important that gameplay code runs at a the same rate as the fixed tick. This is the case because the state which is synchronized will always be associated with a specific tick. To interpolate state correctly on another peer those tick values are used. This is similar to how you should only move RigidBodies in `FixedUpdate`.

MLAPI exposes multiple APIs to allow developers to do so.

- `NetworkManager.Singleton.NetworkTimeSystem.OnNetworkTick` can be subscribed to to run code every tick.
- Code running in Update or FixedUpdate could check for the current tick using `NetworkManager.Singleton.NetworkTime.PredictedTime.Tick`.
- In the future we will expose more convenient ways to run logic during the network tick via a `NetworkFixedUpdate` in `NetworkBehaviour`.

### INetworkTimeProvider

While the built in `NetworkTimeSystem` provides a good starting point for a variety of games, sometimes more control over time is needed. `INetworkTimeProvider` is an interface which allows to override the root calculation of all network time. Note that this is an advanced feature, a faulty implementation of `INetworkTimeProvider` can result in a variety of issues.

Here is an example implementation of a very simple custom `INetworkTimeProvider`:
```csharp
public class SampleNetworkTimeProvider : INetworkTimeProvider
{
    public bool AdvanceTime(ref NetworkTime predictedTime, ref NetworkTime serverTime, float deltaTime)
    {
        serverTime += deltaTime;
        predictedTime = serverTime + 0.1f;
        return true;
    }

    public void InitializeClient(ref NetworkTime predictedTime, ref NetworkTime serverTime)
    {
        predictedTime = serverTime + 0.1f;
    }
}
```

This `INetworkTimeProvider` is a very simple provider which increments `ServerTick` and `PredictedTick` at a consistent rate while running the `PredictedTick` 100 ms ahead of the `ServerTick`. This just acts as an example implementation and is not a recommended way to implement a `INetworkTimeProvider`

### Buffering

When calculating time MLAPI adds additional buffering to delay messages further to smooth out the data stream. The default buffer values are `1f / Tickrate` when the client receives and `2f / Tickrate` when the server receives.

Buffering will only apply on NetworkVariable changes. RPCs, NetworkObject spawns etc. are unaffected.

Buffer time can be manually changed in code to for instance use different buffer sizes for different platforms. `NetworkManager.Singleton.NetworkTimeSystem.ClientBufferTime` and `NetworkManager.Singleton.NetworkTimeSystem.ServerBufferTime` can be used to adjust the time. Note these settings should only be adjusted on a client. The server does not apply any buffering.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Multiplayer and Time

There are different ways to define time in multiplayer scenarios. A multiplayer game connects multiple peers together where each peer has a different game engine time `Time.time` and potentially a different system time. This means there is a lack of a consistent time systems between peers and we need to introduce our our concept.

![peers connected to each other](0014-network-time//npeers.png)

### Networking Topologies

Networking topology affects how we think about the concept of time. In a p2p model where each client broadcasts state to all other clients there might be a need to establish a common time between all clients.

MLAPI always uses a listen server or `star topology` where all clients connect to a single server/host. This server contains and broadcasts state to all other clients. It does not necessarily hold authority over the state.

The possibilities of expressing times boil down to the following

- 1. **Shared Time** A common concept of a shared time between all peers. This time could be used to ensure that all peers can display something at the same real world time.

- 2. **Local Time** As a single peer I can have my own local time and base my gameplay around that time.

- 3. **Receive Time** When a peer receives state from another peer, the receive time is based on the `local time` that peer had when sending the message. This time can be used to smoothly interpolate received data.

The star topology allows us to reduce the amount of receive time each peer needs to keep track of. Since a client only receives state from the server it only has a single receive time, the `Server Time`. In addition it has one `local time` which it uses to send state to the server. The server can then map that time to `server time` when relaying the state to other clients. That way not all clients need to know about the time of other clients.

![star topology time simplification](0014-network-time/startopology.png)

### Simplifying Star Topology Time

When a client sends state to the server the server needs to somehow convert the `local time` of the client into `server time`. This is problematic in a few ways:
- The server must somehow keep track of all the times of every single connected client.
- The difference between the `Server Time` and the `local time` of a client is based on `RTT`. `RTT` can vary a lot between individual packets.
- Since the server is in control of the conversion from `local times` to `server time` it must let the client know about changes. The notifications for this changes will be delayed by `RTT / 2` or more if packets are dropped. In addition the server does not know much about a clients situation so forcing a change onto a client could lead into oscillation of the offset between `server time` and `client time`

To solve all the problems listed above a different way of handling `local time` will be used in MLAPI. In this new model the server will not deal with any time conversions. It will send out state in `server time` and expects to receive state from clients in `server time` as well.

How can the client send state so that it arrives in `server time` on the server? The client has to adjust `local time` so that it matches `server time` on arrival on the server. This is done by predicting the time at which the packet will arrive on the server. The client knows about the time when the last server packet arrived. To predict its `local time` it can add the `RTT` + a bit of buffering to that time and use that as the new `local time`. This concept of adjusting `local time` to the server is what we call `predicted time` from now on.


![predicted time](0014-network-time/predictedtime.png)

### The Elevator Problem

An example problem which will be solved by this new time system is the `elevator problem`. The elevator problem works in the following way.

- There is a platform moving up and down in a constant pattern.
- A local client authoritative or predicted player must be able to walk on the platform.
- Player objects from other players might also walk on the platform.

<img src="0014-network-time/platform_1.gif" width="640" height="360">

*A platform in server space and a server controlled non player character (NPC)*

Example code of moving a platform in server time:
```csharp
public class MovingPlatform : MonoBehaviour
{
    [SerializeField]
    private Vector3 m_Point1;
    [SerializeField]
    private Vector3 m_Point2;
    [SerializeField]
    private float m_Duration;

    // This is just for illustration purposes, not recommended to use for gameplay code
    void Update()
    {
        transform.position = Vector3.Lerp(m_Point1, m_Point2, Mathf.PingPong((NetworkManager.Singleton.ServerTime.Time) / m_Duration, 1));
    }
}
```

When adding a client authoritative player character to a platform the result is the following:

<img src="0014-network-time/platform_2.gif" width="640" height="360">

*RTT = 200ms, `Own Player` = aligned, `Server NPC` = aligned, `Other Player` = 200ms behind*

This is less than ideal since other players will be a full RTT behind the platform. The problem here is that the player character moves in `predicted time` but the platform is in `server time`. This causes the player movement to look correct locally but wrong on all the other clients because once the player movement arrives on the server, the platform is already at another place.

Another way of approaching this could be to run the platform in predicted time. Here is an example code of moving a platform in predicted time:
```csharp
// This is just for illustration purposes, not recommended to use for gameplay code
void Update()
{
    transform.position = Vector3.Lerp(m_Point1, m_Point2, Mathf.PingPong((NetworkManager.Singleton.PredictedTime.Time) / m_Duration, 1));
}
```

After applying the changes, it will look like this:

<p float="left">
<img src="0014-network-time/platform_3.1.gif" width="45%">
<img src="0014-network-time/platform_3.2.gif" width="45%">
</p>

_**Client** RTT = 200ms, `Own Player` = aligned, `Server NPC` = 200ms behind, `Other Player` = 200ms behind_\
_**Host** RTT = 200ms, `Own Player` = aligned, `Server NPC` = aligned, `Other Player` = aligned_

It might seem like predicting the platform only makes the situation worse because now on clients also the server controlled objects are delayed. But there is an important difference to the last model. On the Host all objects are perfectly aligned to the platform because they are in the same time space.

Using our knowledge we acquired from the last two approaches we can come to the conclusion that if we can manage to run a platform in the same time space as all objects on it, they will be displayed correctly. This means we should predict a platform if our player comes close to it but we don't want to predict a platform which is not close to our player and contains objects from other players.

The elevator problem is also present when running a game with server authority but only if prediction is involved. Without prediction, everything will run in `server time` which means `server time` can just be used to display everything perfectly aligned.

When predicting a player object everything it interacts with such as close platforms have to be predicted. If they don't get predicted, the player's prediction will often deviate from the server's prediction which will result in a correction rollback. In some special cases like strictly horizontally moving platforms movement can still work fine without predicting the platform.

## Network Time Struct

A new struct will be introduced to represent network time. Our time system needs to work with ticks in mind which means a floating point value is not sufficient.

```csharp
public struct NetworkTime
{
    private int m_TickRate;
    private float m_TickInterval;

    private int m_Tick;
    private float m_TickDuration;

    public int Tick => m_Tick;

    public int TickRate => m_TickRate;

    public NetworkTime(int tickRate)
    {
        m_TickRate = tickRate;
        m_TickInterval = 1f / m_TickRate; // potential floating point precision issue, could result in different interval on different machines
        m_TickDuration = 0;
        m_Tick = 0;
    }

    public NetworkTime(int tickRate, int tick, float tickDuration = 0f)
        : this(tickRate)
    {
        Assert.IsTrue(tickDuration < 1f / tickRate);

        m_Tick = tick;
        m_TickDuration = tickDuration;
    }

    public NetworkTime(int tickRate, float time)
        : this(tickRate)
    {
        this += time;
    }
}
```

The `NetworkTime` struct will give a user access to similar API functionality as the Unity `Time` API.

```csharp
public struct NetworkTime
{
    //.....
    public float Time => m_Tick * m_TickInterval + m_TickDuration;

    public float FixedTime => m_Tick * m_TickInterval;

    public float FixedDeltaTime => m_TickInterval;

    public int Tick => m_Tick;

    public int TickRate => m_TickRate;
    //....
}
```

NetworkTime will also provide +- operators for `NetworkTime` and `float` like this:

```csharp
public static NetworkTime operator +(NetworkTime a, NetworkTime b)
{
    Assert.AreEqual(a.TickRate, b.TickRate, $"NetworkTimes must have same TickRate to add.");

    int tick = a.Tick + b.Tick;
    float tickDuration = a.m_TickDuration + b.m_TickDuration;

    if (tickDuration > a.m_TickInterval)
    {
        tick++;
        tickDuration -= a.m_TickInterval;
    }

    return new NetworkTime(a.TickRate, tick, tickDuration);
}

public static NetworkTime operator +(NetworkTime a, float b)
{
    a.m_TickDuration += b;

    var deltaTicks = Mathf.FloorToInt(a.m_TickDuration * a.m_TickRate);
    a.m_TickDuration %= a.m_TickInterval;

    a.m_Tick += deltaTicks;

    return a;
}
```

Access to the `NetworkTime` can either be done over the `NetworkManager` for convenience:
```csharp
NetworkTime predictedTime = NetworkManager.Singleton.PredictedTime;
NetworkTime serverTime = NetworkManager.Singleton.ServerTime;
```
or by accessing the `PredictedTime` and `ServerTime` properties of the `NetworkTimeSystem` directly.

## Architecture

![architecture](0014-network-time/architecture.png)

- **INetworkTimeProvider** Open interface, in control of how time advances relative to Time.deltaTime.
- **NetworkTimeSystem** Closed system to keep track of time, convert them to ticks. Also providers helper functions and APIs to interact with network time.
- **NetworkManager** Executes logic for each network tick. Advances the `NetworkTimeSystem`.

## INetworkTimeProvider

INetworkTimeProvider is the interface used to drive time. This interface allows the MLAPI to be open about how a game approaches time. The `INetworkTimeProvider` decides how time is initialized and advanced.

```csharp
public interface INetworkTimeProvider
{
    /// <summary>
    /// Called once per frame to advance time.
    /// </summary>
    /// <param name="predictedTime">The predicted time to advance.</param>
    /// <param name="serverTime">The server time to advance.</param>
    /// <param name="deltaTime">The real delta time which passed.</param>
    /// <returns>true if advancing the the time succeeded; otherwise false if there was a hard correction.</returns>
    public bool AdvanceTime(ref NetworkTime predictedTime, ref NetworkTime serverTime, float deltaTime);

    /// <summary>
    /// Called once on clients only to initialize time.
    /// </summary>
    /// <param name="predictedTime">The predicted time to initialize. In value is serverTime.</param>
    /// <param name="serverTime">the server time to initialize. In value is a time matching the tick of the initial received approval packet.</param>
    public void InitializeClient(ref NetworkTime predictedTime, ref NetworkTime serverTime);
}
```

`AdvanceTime` is called once each update to advance times to new values. The `INetworkTimeProvider` has full control over how time gets advanced. MLAPI will come with two implementations of it.

### ServerNetworkTimeProvider

`ServerNetworkTimeProvider` will be used by the server for a constant time update rate. The server has no need to speed up or slow down it's time.
```csharp
public class ServerNetworkTimeProvider : INetworkTimeProvider
{
    public bool AdvanceTime(ref NetworkTime predictedTime, ref NetworkTime serverTime, float deltaTime)
    {
        predictedTime += deltaTime;
        serverTime += deltaTime;
        return true;
    }

    public void InitializeClient(ref NetworkTime predictedTime, ref NetworkTime serverTime)
    {
        predictedTime = serverTime;
    }
}
```

### ClientNetworkTimeProvider

`ClientNetworkTimeProvider` will be used as the default time provider for non-server peers. The implementation focuses on providing a stable value even under bad networking conditions by sacrificing latency.

The `AdvanceTime` function will do the following

1. Advance time but multiply the delta by a time scale to catch up or slow down if needed.
```csharp
predictedTime += deltaTime * m_PredictedTimeScale;
serverTime += deltaTime * m_ServerTimeScale;

```
2. Calculate the ideal target time for the client
```csharp
float targetServerTime = lastReceivedSnapshotTime - TargetClientBufferTime;
float targetPredictedTime = lastReceivedSnapshotTime + rtt + TargetServerBufferTime;
```
`targetPredictedTime` is currently just based on RTT. This is not ideal and not very precises which means we need to apply a lot of buffering to smooth out the data stream which introduces more latency. With the command + ack system we will improve this to instead base the `targetPredictedTime` on the size of the command queue of the server.

In addition RTT is currently taken from the `NetworkTransport` which is not ideal. In the future when we have a snapshot ack system we will leverage that system to calculate RTT more accurately and use an exponential moving average or similar to smooth the RTT value.

3. Check whether there is a need for a hard reset/catchup

```csharp
bool reset = false;
m_PredictedTimeScale = 1f;
m_ServerTimeScale = 1f;

// reset because too large predicted offset?
if (predictedTime.FixedTime < targetPredictedTime - k_HardResetThreshold || predictedTime.FixedTime > targetPredictedTime + k_HardResetThreshold)
{
    reset = true;
}

// reset because too large server offset?
if (serverTime.FixedTime < targetServerTime - k_HardResetThreshold || serverTime.FixedTime > targetServerTime + k_HardResetThreshold)
{
    reset = true;
}

// Always reset both times to not break simulation integrity.
if (reset)
{
    predictedTime = new NetworkTime(predictedTime.TickRate, targetPredictedTime);
    serverTime = new NetworkTime(serverTime.TickRate, targetServerTime);
    return false;
}
```

4. If no hard reset was needed adjust time scale to catch up or slow down
```csharp
// Adjust predicted time scale
if (Mathf.Abs(targetPredictedTime - predictedTime.FixedTime) > k_CorrectionTolerance)
{
    m_PredictedTimeScale += targetPredictedTime > predictedTime.FixedTime ? k_AdjustmentRatio : -k_AdjustmentRatio;
}

// Adjust server time scale
if (Mathf.Abs(targetServerTime - serverTime.FixedTime) > k_CorrectionTolerance)
{
    m_ServerTimeScale += targetServerTime > serverTime.FixedTime ? k_AdjustmentRatio : -k_AdjustmentRatio;
}

return true;
```

Most of the values such as correction tolerance and adjustment ration will be configurable by the developer if needed. In addition ClientNetworkTimeProvider will provide a mode where it only does hard resets and no soft catch ups. This is needed for certain types of games like rhythm game where background music needs to be synced to gameplay.


## NetworkTimeSystem

```csharp
public class NetworkTimeSystem
{
    private INetworkTimeProvider m_NetworkTimeProvider;
    private int m_TickRate;

    private NetworkTime m_PredictedTime;
    private NetworkTime m_ServerTime;

    /// <summary>
    /// Delegate for invoking an event whenever a network tick passes
    /// </summary>
    /// <param name="time">The predicted time for the tick.</param>
    public delegate void NetworkTickDelegate(NetworkTime time);

    /// <summary>
    /// Gets invoked before every network tick.
    /// </summary>
    public event NetworkTickDelegate OnNetworkTick = null;

    /// <summary>
    /// Gets invoked during every network tick. Used by internal components like <see cref="NetworkManager"/>
    /// </summary>
    internal event NetworkTickDelegate OnNetworkTickInternal = null;

    /// <summary>
    /// Creates a new instance of the <see cref="NetworkTimeSystem"/>.
    /// </summary>
    /// <param name="config">The network config</param>
    /// <param name="isServer">true if the system will be used for a server or host.</param>
    public NetworkTimeSystem(NetworkConfig config, bool isServer)
    {
        m_TickRate = config.TickRate;

        if (isServer)
        {
            m_NetworkTimeProvider = new ServerNetworkTimeProvider();
        }
        else
        {
            m_NetworkTimeProvider = new ClientNetworkTimeProvider(this);
        }

        m_PredictedTime = new NetworkTime(config.TickRate);
        m_ServerTime = new NetworkTime(config.TickRate);
    }
}
```

`NetworkTimeSystem` does not have dependencies on other non `timing` related MLAPI components. `NetworkTimeSystem` does not automatically advance time but must be advanced by calling the `AdvanceNetworkTime` function.

```csharp
/// <summary>
/// Called each network loop update during the <see cref="NetworkUpdateStage.PreUpdate"/> to advance the network time.
/// </summary>
/// <param name="deltaTime">The delta time used to advance time. During normal use this is <see cref="Time.deltaTime"/>.</param>
public void AdvanceNetworkTime(float deltaTime)
{
    // store old predicted tick to know how many fixed ticks passed
    var previousPredictedTick = PredictedTime.Tick;

    m_NetworkTimeProvider.AdvanceTime(ref m_PredictedTime, ref m_ServerTime, deltaTime);

    // cache times here so that we can adjust them to temporary values while simulating ticks.
    var cachePredictedTime = m_PredictedTime;
    var cacheServerTime = m_ServerTime;

    var currentPredictedTick = PredictedTime.Tick;
    var predictedToServerDifference = currentPredictedTick - ServerTime.Tick;

    for (int i = previousPredictedTick + 1; i <= currentPredictedTick; i++)
    {
        // set exposed time values to correct fixed values
        m_PredictedTime = new NetworkTime(TickRate, i);
        m_ServerTime = new NetworkTime(TickRate, i - predictedToServerDifference);

        OnNetworkTick?.Invoke(m_PredictedTime);
        OnNetworkTickInternal.Invoke(m_ServerTime);
    }

    // Set exposed time to values from tick system
    m_PredictedTime = cachePredictedTime;
    m_ServerTime = cacheServerTime;
}
```

There is one challenge in the implementation of `AdvanceNetworkTime`. It is not clear how to handle the relation ship between `PredictedTime` and `ServerTime` if multiple ticks have passed in the same advance step. The current solution expects that `PredictedTime` and `ServerTime` advanced at roughly the same pace which is true for most of the cases but not always. The alternative to solve this problem would be to have a more fine grained `INetworkTimeProvider` but this approach was not chosen to keep the interface simple.

## Removal of TimeSync

The `TimeSync` message and all associated code has been removed from MLAPI. The concept of re-syncing time is not needed if we apply the learnings from the chapter above as all peers will always advance real time at the same pace. RTT and other factors will affect the `NetworkTime` but the client will be in control over adjusting time without the need of server input.

## RFC 12: Synced Networked Variables

This RFC evolves some of the concepts introduced in [RFC-12](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/12). It does not change the fundamental principles introduced with the RFC. It just adjusts them to work better with the new time system.

## Serializing Ticks

Ticks are stored as int instead of ushort. This allows us to to theoretically run a server for many (414) days while keeping time accurate. The negative range of int is reserved to describe invalid tick values.

NetworkTickWrapping as described in RFC-12 will be removed. To save bandwidth when sending ticks future features like the snapshot system will provide a way to delta compress tick values inside a snapshot packet against the header tick of the snapshot.

### NetworkVariable Writes

NetworkVariable will no longer carry two ticks. Instead of a `local tick` and a `remote tick` there will be only a `last modified tick`. This change is done because there is no longer a need for multiple ticks. Writing NetworkVariables always happens in `PredictedTime` while receiving NetworkVariables happens in `ServerTime`.

How NetworkVariable writes will be changed internally with the time system proposed in this RFC. INetworkVariable will expose a LastModifiedTick value.
```csharp
int LastModifiedTick { get; }
```

The setter of NetworkVariable will set `LastModifiedTick` to predicted tick:
```csharp
/// <summary>
/// The value of the NetworkVariable container
/// </summary>
public T Value
{
    get => m_InternalValue;
    set
    {
        if (EqualityComparer<T>.Default.Equals(m_InternalValue, value))
        {
            return;
        }
        
        LastModifiedTick = NetworkManager.Singleton.PredictedTime.Tick;

        m_IsDirty = true;
        T previousValue = m_InternalValue;
        m_InternalValue = value;
        OnValueChanged?.Invoke(previousValue, m_InternalValue);
    }
}
```

When serializing a NetworkVariable the `LastModifiedTick` will be included.

This change does not yet introduce any changes to how NetworkVariables work. This is base infrastructure which will later be used by the snapshot system.

In the future we might introduce buffering for client NetworkVariable writes. The server will buffer incoming NetworkVariable changes until  `PredictedTick` matches the `LastModifiedTick` in the packet.

# Drawbacks
[drawbacks]: #drawbacks

The new time system has been built around the premise that MLAPI is using a star topology. If we want to add a peer to peer mode in MLAPI in the future many systems which will be integrated into the time systems might need to be rewritten.

The time system will have influence on how major systems of MLAPI such as NetworkVariable synchronize state. For instance it could add additional input delay to buffer inputs. For some types of games this might not be a desired behaviour and while there are options to disable many functions of the time system it might be still a limiting factor in some specific situations.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why is this design the best in the space of possible designs?

The design of this time system is the best possible design because it does define the two relevant time spaces in a star topology. `PredictedTime` being the time of the local peer and `ServerTime` as the time of the server peer. Assuming we don't need to know about the time spaces from other clients, all other relevant time spaces can be derived from the two if needed.

## What other designs have been considered and what is the rationale for not choosing them?

### Alternative: Don't Base Time on Ticks
We can keep MLAPI as just a simple messaging/state replication library which does not take control of the game loop via a tick system. This would mean that the developers themselves would have to implement systems to ensure smooth movement and prediction and we won’t be able to provide production grade components like ‘NetworkTransform’ out of the box. The advantages of this would be that changes to NetworkVariables etc. would be synchronized instantly without the need of being aligned to a tick.

### Client Partial Simulation vs Interpolation

There are two popular ways for dealing with the fixed tick to render frame aliasing issue on the client.
- Interpolation: We run game logic only during a fixed tick. When we need to interpolate a frame at 15.3 (tickrate == 1) then we get the interpolated time 14.3 (15.3 - tickrate) and interpolate between the last two calculated values from the fixed tick (14, 15).
- Partial Simulation: After simulating the fixed tick for 15 we run another simulation for 0.3 seconds so that we alias correctly with the predicted time. During the next frame we revert back to the value at 15 so that we can properly simulate the next fixed tick if needed.

The current time system allows for both approaches but some of the fixed tick API is focuses on the `interpolation` approach. A `Partial Simulation` approach could still be achieved with the current time system but running the partial tick is something which would have to be done manually by the developer in update. `Unity Netcode` uses a partial simulation approach.

The `interpolation` approach was chosen because it works better with other systems which also expect a fixed tick such as Physics and overall has far less edge cases to solve. And advantage of the partial simulation approach is that the player object has less latency caused by the interpolation.

## What is the impact of not doing this?

We need a time system based on fixed ticks for the snapshot system and other systems such as commands and prediction. The current time system can't be used to provide that and would need to be rewritten to something similar like the system proposed in this RFC.

# Prior art
[prior-art]: #prior-art

The [Source Engine](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking) by Valve uses a similar design for network ticks and time. The main difference between our implementation and theirs are that we provide more freedom to the developer for how to handle time with the `INetworkTimeProvider` interface. The Source engine supports using a different snapshot sendrate which is something we could explore.

![source engine tick system](https://developer.valvesoftware.com/w/images/e/ea/Networking1.gif)

Many FPS games such as `Overwatch` or the `Unity FPS Sample` use a similar way of stretching time to adjust network time to latency changes. This way of handling time has been proven by many fast pace production games and should also be a very good way of handling time for more casual co-op games.
- [Overwatch GDC Talk](https://www.youtube.com/watch?v=W3aieHjyNvw&t=553s)
- [Unity FPS Sample Unite Talk](https://www.youtube.com/watch?v=k6JTaFE7SYI)

Glenn Fiedler explains in [this article](https://gafferongames.com/post/fix_your_timestep/) why a fixed time step is important for many systems such as Netcode and Physics and goes over the basics on how to build one.

[Unreal Engine's networking](https://docs.unrealengine.com/en-US/InteractiveExperiences/Networking/index.html) exposes much less information about times and ticks to the developer. This is a different approach which we could consider to take. The current approach might be a bit harder for beginners to get into but will allow for much more flexibility allowing us to support many different multiplayer archetypes.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
    - Special needs from games in regards to time which are not addressed by the system proposed in this RFC.
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
    - Implementation details such as finding good default values which allows time to adjust without oscillating between being too much ahead/behind.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

- We will probably expose a way in `NetworkBehaviour` to run logic for each fixed tick with a function such as `NetworkFixedUpdate`.
- We will improve the time provider to calculated predicted time based on the command buffer size of the command system.
- We will tie the physics simulation to the network tick and run it after each fixed tick.
- The time system provides a single tick value which can be used for anything such as running the gameplay or sending out snapshots but it is by no means restricting any other systems to use it. For instance sending snapshots could be something where we decide to give an option in the future to run only every second tick.
- In the far future we could look into a tighter engine integration of the time system. Potentially override the values of `Time.time` and `Time.fixedTime` etc.
- To make RPCs more predictable we could have them expose the predicted tick value of when they were invoked. For instance if an RPC which creates an explosion gets delayed with the current RPC system the explosion would be delayed and out of place. By exposing the tick at invoke time a developer could write code to adjust for that delay and spawn the explosion particle midway through the animation.
- Profiling tools for time related values. We could display things like buffer sizes or time scale to allow developers to debug time issues.
- We could explore allowing to change the tickrate of the game during a session. This is sometimes used in games genres like Battle Royale. There the server can only run at a low tickrate at the beginning of the game because it needs to simulate 100+ players. Over the course of the game tickrate can be gradually increased as the server load decreases.
