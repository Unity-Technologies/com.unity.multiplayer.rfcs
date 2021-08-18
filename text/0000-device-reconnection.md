# Device Reconnection
[feature]: #feature

- Start Date: `2021-08-03`
- RFC PR: TBD
- SDK PR: TBD

# Summary
[summary]: #summary

Transports that support device reconnection automatically re-establish connections when their underlying medium changes (a typical example is an IP address change). Such changes are detected by monitoring communication loss. The re-establishment of the connection is done transparently to layers above the transport (the event is advertised but can be ignored).

Because the feature works by detecting traffic losses, and advertises these losses, it can also be useful to ensure proper re-synchronization between hosts after lag spikes.

This RFC discusses the addition of that feature to the Unity Transport Package (UTP) and its adapter in Netcode for GameObjects (NGO). Other transports are out of scope.

# Motivation
[motivation]: #motivation

Mobile devices present several challenges in terms of network programming, one of them being that they are likely to change networks while in active use. For example, this can happen when riding the bus and hopping from one network to another, or when coming home and switching from the cellular network to your residential Wi-Fi network.

Such network switching often causes a local IP address change, typically causing existing connections to stop working. This is disruptive to end users since it prevents them from continuing their online session. The goal of device reconnection is preventing these disruptions by automatically migrating the connections to the new IP address.

Device reconnection can also be leveraged to be notified of short losses in traffic. This may happen when going through a tunnel on mobile. Even if the loss of traffic may not be long enough to cause a full disconnection, users of the transport might want to be notified of such events. Such notifications could then be used to ensure proper re-synchronization between hosts (note that such re-synchronization is out of scope of the device reconnection feature).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Two new `NetworkEvent`s are added to UTP and NGO:

- The `Reconnecting` event will be generated when loss of traffic is detected by the transport and it is trying to re-establish the connection.
- The `Reconnect` event will be generated when a connection is re-established.

Note that both events are only meant as notifications. No action is required upon receiving these events. They can be ignored entirely and the device reconnection feature will work fine. All operations (either on the `NetworkTransport` in NGO or the `NetworkDriver` in UTP) can still be performed while a reconnection is ongoing. But messages sent on an unreliable channel will obviously be lost.

A `Reconnecting` event will always be followed by either a `Reconnect` or a `Disconnect` event, depending on whether the connection could be re-established or not. `Reconnecting` events are always tied to loss of traffic. To ensure inactivity doesn't trigger a reconnection, the transport uses heartbeats to monitor the liveness of the connection.

A `Reconnect` event may be generated on its own, without a preceding `Reconnecting` event. This is used to indicate that a reconnection occured, but without any discernible loss of traffic. Such reconnections can occur when running directly over UDP (without DTLS), where connections are not tied to IP addresses.

Both events are generated server-side and client-side, but reconnections are only initiated client-side. That is, active servers (servers who change IP address) are not supported. The only exception is when using Relay, where both client and server are acting as “clients” of the Relay server (in this scenario it is active Relay servers that are not supported).

## Examples

For UTP the events will be accessible using `NetworkDriver.PopEvent` and friends:

```cs
var eventType = driver.PopEvent(out connection, out reader);

switch (eventType)
{
    case NetworkEvent.Type.Reconnecting:
        Debug.Log("Lost connection. Attempting to reconnect...");
        break;
    case NetworkEvent.Type.Reconnect:
        Debug.Log("And we're back!");
        break;
}
```

For `UTPTransport` (the UTP adapter for NGO), the events will be generated through the `OnTransportEvent` delegate (`UTPTransport` doesn't support direct polling via `PollEvent`):

```cs
public void HandleEvent(NetworkEvent eventType, ...)
{
    if (eventType == NetworkEvent.Reconnecting || eventType == NetworkEvent.Reconnect)
        Debug.Log("Received a reconnection-related event.");
}

...

utpTransport.OnTransportEvent += HandleEvent;
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Implementation of the feature requires changes both to NGO and UTP.

The changes in NGO are very minor. The `NetworkEvent` enum is expanded with the new `Reconnecting` and `Reconnect` events. The UTP adapter also needs to handle the new events coming from UTP's `NetworkDriver` and map them to the appropriate NGO events.

## Unity Transport Package

The basic idea behind the implementation of device reconnection is that for each connection, we track its inactivity. When no data is received on a connection for some time (exact figure TBD), we trigger a reconnection. To avoid normal inactivity from triggering reconnections, a heartbeat mechanism is added to each protocol.

Heartbeats are implemented as ping/pong messages (every ping is answered with a pong), and are sent on a connection when no data has been received for a while.

### Modifications to `NetworkDriver`

The `CheckTimeouts` function (called as part of the `ScheduleUpdate` job) is also extended with something that looks like the following pseudo-code:

```
for each connection:
    if time from last receive > heartbeat threshold:
        if time of last heartbeat send > heartbeat threshold:
            send heartbeat on connection
    
    if reconnection in progress:
        if connection state is connected:
            generate reconnect event
    else:
        if time from last receive > reconnection threshold:
            generate reconnecting event
            if client side:
                trigger reconnection on connection
```

If the reconnection fails a set amount of times (exact figure TBD), the connection is considered disconnected, and the `Disconnect` event is generated. This replaces the existing inactivity timeout (`disconnectTimeoutMS`).

### Extensions to `NetworkProtocol`

Device reconnection requires `NetworkProtocol` to provide two additional interfaces (both as `TransportFunctionPointer`s):

- `ProcessReconnectDelegate` will re-establish a given connection in a manner suitable for the protocol. For some protocols (like DTLS) this is done by simply creating a whole new connection from scratch. Other protocols might have better ways to do so (like Relay where only a re-binding to the server is required).
- `ProcessSendPing` and `ProcessSendPong` will send ping/pong messages on the connection. The actual implementation of the heartbeats will be provided by `UnityTransportProtocol`, on which all other protocols already rely for control messages. Corresponding `ProcessPacketCommands` are raised to the driver when receiving these messages.

### Special handling of UDP connections

When using the basic UDP protocol (`UnityTransportProtocol`) and nothing else (no DTLS or Relay), reconnections are handled differently. This is because UDP connections are not tied to an IP address. They don't need to be reconnected when the client's IP address changes. (This is assuming that the client driver is not bound to a specific address, which normally won't be the case. In that scenario, the above mechanism with inactivity timeouts would kick in.)

Properly supporting seamless reconnections in that scenario involves modifying `NetworkDriver` to update a connection's address upon receiving a valid message from a new IP (currently such messages are ignored).

While reconnections over UDP may be seamless, we still want to notify users that they happened. On the server side we can simply generate a `Reconnect` event when receiving data from a new address. On the client side this is trickier since the reconnection is completely transparent. To be made aware of the event, the server will resend a `ConnectionAccept` message to the client when a reconnection is detected. This can be used by the client to generate the `Reconnect` event.

### Protection against connection hijacking

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

- The QUIC protocol ([RFC 9600](https://datatracker.ietf.org/doc/html/rfc9000)) is a network protocol built over UDP that automatically handles reconnections the same way we do for the base UDP protocol. (Obviously QUIC is much more complex since it also bundles TLS and congestion control.)
- There's been some proposals regarding mobile DTLS (latest is [here](https://www.ietf.org/archive/id/draft-ietf-tls-dtls-connection-id-13.txt)), but nothing that made it into an official extension.
- IP mobility does not appear to be a common feature of game-centric networking libraries. [ENet](http://enet.bespin.org/) doesn’t support it. Valve’s [GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets) don’t directly support it either, but one can get something similar working through [Steam Datagram Relay](https://partner.steamgames.com/doc/features/multiplayer/steamdatagramrelay). [Unreal's networking](https://docs.unrealengine.com/4.26/en-US/InteractiveExperiences/Networking/) doesn't either, but they can [integrate with Steam's solution](https://docs.unrealengine.com/4.26/en-US/InteractiveExperiences/Networking/HowTo/SteamSockets/) and thus provide mobility through Steam Datagram Relay.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How NGO will react to `Reconnecting` and `Reconnect` events (if it will even react) is left out of scope of this RFC.
- Whether device reconnection will be made optional (and in general how configurable it should be) is still unknown.

# Future possibilities
[future-possibilities]: #future-possibilities

As mentioned in the section above, the question of whether NGO will handle `Reconnecting` and `Reconnect` events (and if so, how) is left open for future discussions.

Another topic to consider is how future evolutions of UTP might impact the implementation of device reconnection. In particular, protocol stacking opens up the possibility of implementing the feature as a protocol. While this might make the implementation slightly more verbose (due to the boilerplate around implementing a protocol), it would provide the benefit of clearly separating device reconnection from the rest of UTP.
