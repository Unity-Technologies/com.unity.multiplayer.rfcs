# Device Reconnection
[feature]: #feature

- Start Date: `2021-08-03`
- RFC PR: [#34](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/34)
- SDK PR: N/A

# Summary
[summary]: #summary

Transports that support device reconnection automatically re-establish connections when their underlying medium changes (a typical example is an IP address change). Such changes are detected by monitoring communication loss. The re-establishment of the connection is done transparently to layers above the transport.

This RFC discusses the addition of that feature to the Unity Transport Package (UTP).

# Motivation
[motivation]: #motivation

Mobile devices present several challenges in terms of network programming, one of them being that they are likely to change networks while in active use. For example, this can happen when riding the bus and hopping from one network to another, or when coming home and switching from the cellular network to your residential Wi-Fi network.

Such network switching often causes a local IP address change, typically causing existing connections to stop working. This is disruptive to end users since it prevents them from continuing their online session. The goal of device reconnection is preventing these disruptions by automatically migrating the connections to the new IP address.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A new `reconnectionTimeoutMS` setting will be provided through the `NetworkSettings.WithNetworkConfigSettings` call. This value (in milliseconds) indicates after how long a reconnection should be attempted if no activity is detected.

That is, if no traffic occurs on a connection for `reconnectionTimeoutMS` milliseconds, UTP will automatically try to re-establish the connection. If reconnection fails, it is re-attempted again `reconnectionTimeoutMS` later until either it succeeds or the normal disconnection timeout is reached (at which point the user is notified of the disconnection).

The entire process is transparent to the user, except of course for the small loss of traffic during the reconnection timeout. For this reason, `reconnectionTimeoutMS` should be set to a relatively low value (the default is 2 seconds).

Device reconnection can be disabled by setting the timeout to 0.

## Example

Here's how device reconnection can be configured on a `NetworkDriver`:

```csharp
var settings = new NetworkSettings();
settings.WithNetworkConfigSettings(reconnectionTimeout: 2000);
var driver = NetworkDriver.Create(settings);
```

Note that device reconnection is enabled by default with a 2000 milliseconds timeout, so the above code would have no effect at all.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The basic idea behind the implementation of device reconnection is that for each connection, we track its inactivity. When no data is received on a connection for some time, we trigger a reconnection. To avoid normal inactivity from triggering reconnections, a heartbeat mechanism is added to each protocol.

Heartbeats are implemented as ping/pong messages (every ping is answered with a pong), and are sent on a connection when no data has been received for a while.

## Modifications to `NetworkDriver`

The `CheckTimeouts` function (called as part of the `ScheduleUpdate` job) is also extended with something that looks like the following pseudo-code:

```
for each connection:
    if time from last receive > heartbeat threshold:
        if time of last heartbeat send > heartbeat threshold:
            send heartbeat on connection
    
    if connection is reconnecting:
        if connection state is connected:
            mark connection as not reconnecting
    else:
        if time from last receive > reconnection threshold:
            trigger reconnection on connection
            mark connection as reconnecting
```

If the reconnection fails, it is retried periodically until either it succeeds or the disconnection timeout (`disconnectTimeoutMS`) is reached. In the latter case, a `Disconnect` event is generated.

## Extensions to `NetworkProtocol`

Device reconnection requires `NetworkProtocol` to provide two additional interfaces (both as `TransportFunctionPointer`s):

- `Reconnect` will re-establish a given connection in a manner suitable for the protocol. For some protocols (like DTLS) this is done by simply creating a whole new connection from scratch. Other protocols might have better ways to do so (like Relay where only a re-binding to the server is required).
- `ProcessSendPing` and `ProcessSendPong` will send ping/pong messages on the connection. The actual implementation of the heartbeats will be provided by `UnityTransportProtocol`, on which all other protocols already rely for control messages. Corresponding `ProcessPacketCommands` are raised to the driver when receiving these messages.

## Special handling of UDP connections

When using the basic UDP protocol (`UnityTransportProtocol`) and nothing else (no DTLS or Relay), reconnections are handled differently. This is because UDP connections are not tied to an IP address. They don't need to be reconnected when the client's IP address changes. (This is assuming that the client driver is not bound to a specific address, which normally won't be the case. In that scenario, the above mechanism with inactivity timeouts would kick in.)

Properly supporting seamless reconnections in that scenario involves modifying `NetworkDriver` to update a connection's address upon receiving a valid message from a new IP (currently such messages are ignored).

## Protection against connection hijacking

With the possibility of connections migrating from one address to another comes the possibility that malicious actors will attempt to hijack existing connections to redirect them elsewhere (e.g. for amplification attacks). UTP will attempt to protect against hijacks from parties that are not privy to existing traffic. If protection against man-in-the-middle attacks is a concern, using DTLS is expected.

Every message processed by UTP includes an internal session ID that is 16 bits long. This can reasonably be brute-forced on the open internet. As such, the session ID will be modified to be at least 32 bits long, so that it can't easily be guessed.

# Drawbacks
[drawbacks]: #drawbacks

- Heartbeats are a very common feature in transport protocols, but considering they are not currently implemented in UTP, their introduction might bring an extra bandwidth overhead. This is especially true for applications that send data very rarely.
- The implementation extends the `NetworkProtocol` API, which imposes a heavier burden on any future implementations of new protocols in UTP. It could also make it harder to refactor the code (say for protocol stacking).
- Implementing most of the reconnection decision logic in `NetworkDriver` is the simplest and most straightforward solution, but it does mean that more complexity is added to the part of the UTP code that is already arguably the hardest to understand.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are two other designs that could have achieved reconnection in a better manner:

- Building UTP over a protocol that already supports seamless mobility (like QUIC; see below). This would have tied UTP to the limitations of that protocol (performance issues, lack of flexibility, limited portability, etc.). It also would have required other services UTP interacts with (like Relay) to support that protocol.
- Building other protocols on top of the base UDP protocol, which can support seamless mobility. For Relay that would require the server to implement that base UDP protocol, which they might not be able/willing to do. This solution would also prevent users from creating fully encrypted connections (since control messages would be unencrypted).

Not implementing device reconnection means that users will be inconvenienced (by having their online sessions stop working) when roaming from one network to another, which is bad UX. Not providing the feature would also go against our goals of being a top-of-the-class multiplayer solution with a mobile-first philosophy.

# Prior art
[prior-art]: #prior-art

- The QUIC protocol ([RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000)) is a network protocol built over UDP that automatically handles reconnections the same way we do for the base UDP protocol. (Obviously QUIC is much more complex since it also bundles TLS and congestion control.)
- There's been some proposals regarding mobile DTLS (latest is [here](https://www.ietf.org/archive/id/draft-ietf-tls-dtls-connection-id-13.txt)), but nothing that made it into an official extension.
- IP mobility does not appear to be a common feature of game-centric networking libraries. [ENet](http://enet.bespin.org/) doesn’t support it. Valve’s [GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets) don’t directly support it either, but one can get something similar working through [Steam Datagram Relay](https://partner.steamgames.com/doc/features/multiplayer/steamdatagramrelay). [Unreal's networking](https://docs.unrealengine.com/4.26/en-US/InteractiveExperiences/Networking/) doesn't either, but they can [integrate with Steam's solution](https://docs.unrealengine.com/4.26/en-US/InteractiveExperiences/Networking/HowTo/SteamSockets/) and thus provide mobility through Steam Datagram Relay.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

# Future possibilities
[future-possibilities]: #future-possibilities

A topic to consider is how future evolutions of UTP might impact the implementation of device reconnection. In particular, protocol stacking opens up the possibility of implementing the feature as a protocol. While this might make the implementation slightly more verbose (due to the boilerplate around implementing a protocol), it would provide the benefit of clearly separating device reconnection from the rest of UTP.

Also, originally the feature entailed generating `Reconnecting` and `Reconnected` events when a reconnection respectively starts and completes. This was removed as it was deemed that it could cause confusion with future session reconnection events, and there was no use for these events. But if a use for these events is found (say improved re-synchronization of game state), then these events could be added back.
