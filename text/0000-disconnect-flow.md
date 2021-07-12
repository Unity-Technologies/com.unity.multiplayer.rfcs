- Feature Name: `disconnect_flow`
- Start Date: 2021/06/02
- RFC PR: 
- Issue: [[Unity-Technologies/com.unity.multiplayer.mlapi#772](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/issues/772)

# Summary

This feature standardizes the flow for communicating disconnects, both between server and client, and between SDK and user.

# Motivation

The motivation is to better enable users to react to network events, including servers reacting to disconnecting clients, clients reacting to lost connections to the server, and clients reacting to other clients joining and leaving the game.

# Guide-level explanation

When network topology changes (via new or lost connections, whether intentionally or unintentionally), it's often desirable for the processes that are (or were) involved in the network to be informed of it so that they can react. In order to reactto these events, MLAPI provides the following callbacks that can be registered to NetworkManager:

- `OnClientConnectedCallback` - Called on the **server** when a new client connection is **formed**. Expects a delegate with the following signature:
  
  ```C#
  void OnClientConnected(int clientId){}
  ```

- `OnClientDisconnectCallback` - Called on the **server** when a client connection is **lost**. Expects a delegate with the following signature:
  
  ```C#
  void OnClientDisconnected(int clientId, NetworkManager.DisconnectReason reason)
  ```

- `OnPeerConnectedCallback` - Called on the **client** when another client has **connected** to the same server. Expects a delegate with the following signature:
  
  ```C#
  void OnPeerConnected(int clientId){}
  ```

- `OnPeerDisconnectedCallback` - Called on the **client** when another client has **disconnected** from the same server. Expects a delegate with the following signature:
  
  ```C#
  void OnPeerDisconnected(int clientId, NetworkManager.DisconnectReason reason)
  ```

- `OnServerConnectionEstablishedCallback` - Called on the **client** when a server connection is **established**. Expects a delegate with the following signature, where `peers` is the list of clients who are already in the game:
  
  ```C#
  void OnServerConnectionEstablished(int[] peers){}
  ```

- `OnServerConnectionLostCallback` - Called on the **client** when it is **no longer connected** to the server (i.e., no longer a part of the game at all). Expects a delegate with the following signature:
  
  ```C#
  void OnServerConnectionLost(NetworkManager.DisconnectReason reason)
  ```

The `reason` parameter is an enum currently defined as follows:

```C#
public enum DisconnectReason
{
    TimeOut, // Connection to the remote process timed out
    Stopped, // The remote process shut down its network cleanly
    Kicked, // The client was kicked from the server
    ConfigurationMismatch, // The client and server have mismatched configurations
    NotAuthorized, // The client is not (or is no longer) authorized to be connected
}
```

# Reference-level explanation

There are basically three components to this implementation.

The first component is to enable to server to broadcast to clients which other clients are connected. This is implemented with two commands: a new `CLIENT_CONNECTED` command and the existing `CONNECTION_APPROVED` command. When a new client connects, a `CLIENT_CONNECTED` message with that clients ID is broadcast to all other connected clients to inform them of the new peer, and when the `CONNECTION_APPROVED` command is sent to the new peer, a list of existing connected client IDs is included. 

 Upon receipt of one of these messages:

- If the message is `CLIENT_CONNECTED`, the `OnPeerConnectedCallback` is invoked.

- If the message is `CONNECTION_APPROVED`, the `OnServerConnectionEstablished` callback is invoked. **This is a change: The client will no longer receive the OnClientConnected callback, which is now server-only.**

The other new message that's added is a `DISCONNECT` message, which contains in it the `DisconnectReason` value and the disconnecting client's clientId. A `DISCONNECT` message is sent:

- From client to server any time `NetworkManager.StopClient()` or `NetworkManager.StopHost()` is called, providing `DisconnectReason.ShutDown` as the reason. **Server does not honor or use the clientId value for security reasons. It infers this value from the identity of the sender.**

- From server to all clients with the `ServerClientId` any time `NetworkManager.StopServer()` or `NetworkManager.StopHost()` is called, providing `DisconnectReason.ShutDown` as the reason

- From server to all clients with the disconnecting client's clientId any time it receives a `DISCONNECT` from a client, forwarding the received reason

- From server to all clients with the disconnecting client's clientId any time connection to a client times out, providing `DisconnectReason.TimedOut` as the reason

- From server to a client that has sent a `CONNECTION_REQUEST` message with a mismatched configuration, providing `DisconnectReason.ConfigurationMismatch` as the reason

- From server to a client that has sent a `CONNECTION_REQUEST` message that's rejected by the `ConnectionApprovalCallback`, providing `DisconnectReason.NotAuthorized` as the reason

- From server to all clients any time the server closes an established client's connection for any reason, providing `DisconnectReason.Kicked` as the reason

When a `DISCONNECT` message is received:

- If the recipient is a client and the clientId is its own `LocalClientId` or the `ServerClientId`, it calls the `OnServerConnectionLostCallback` with the provided reason and `NetworkManager.StopClient()` or `NetworkManager.StopHost()` is called automatically as appropriate. **This is a change: The client will no longe receive the OnClientDisconnected callback, which is now server-only**

- If the recipient is a client and a clientId other than `LocalClientId` or `ServerClientId` is provided, it calls the `OnPeerDisconnectedCallback` with the provided reason.

- If the recipient is a server, it calls the `OnClientDisconnectedCallback` with `DisconnectReason.Stopped`

Disconnect callbacks can also be invoked in the following circumstances:

- When the server detects a timeout on a client connection, it will call the `OnClientDisconnectedCallback` with `DisconnectReason.TimedOut`

- When the client detects a timeout on the server connection, it will call the `OnServerConnectionLostCallback` with `DisconnectReason.TimedOut`

- When a server closes an established client's connection for any reason, it will call the `OnClientDisconnectedCallback` with `DisconnectReason.Kicked`

**Note: ConnectedClients and ConnectedClientsList do not get updated, as clients cannot talk directly to each other so there is no NetworkClient to provide.**

# Drawbacks

Other than adding a nearly negligible amount of network traffic, there are no drawbacks to speak of.

# Rationale and alternatives

Providing this information to clients allows clients to more intelligently react to connection changes. In particular, the `DisconnectReason` field is useful for messaging about events to a player - while the code may handle all types of disconnects the same, it is useful to be able to communicate to the player the difference between a network timeout, a server shutdown, an authorization failure, and a kick.

# Prior art

TODO

- 

# Unresolved questions

- Knowing that we can't update ConnectedClients and ConnectedClientsList, should we have a `HashSet<ulong> PeerList` available to users so they don't have to maintain such a list themselves?
- Is there any possibility that providing a client with a list of anonymous client IDs for other clients poses a security risk? It's not clear what they could do with that information that would pose such a risk, especially since any objects owned by other clients will expose that same client ID.

# Future possibilities

There is the possibility additional disconnect reasons may be added in the future, or even the possibility we may want to provide the ability for the user to provide their own disconnect reasons. A `NetworkManager.Kick(clientId, reason)` method seems valuable. In that situation, we may consider making the reason a ulong with reserved values so users can add additional disconnect codes, or possibly even a string.
