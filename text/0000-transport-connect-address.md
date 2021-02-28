- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

This RFC proposes extending the Transport class with a common set of connection oriented properties.

# Motivation
[motivation]: #motivation

This change will make transport more interchangeable in both user code and library extensions. Some example use cases which need the NetworkAddress/Port properties.

- As a user I want to create a connect UI for my game. During development I want to swap out transports to test different scenarios. I want my existing code to work for any transport I use. We have already encountered this issue in two of our samples and ended up using an ugly switch each time.
- It would be really useful if MLAPI where to provide a built in NetworkingManagerHUD for testing which would be a built in component that ads a simple HUD for starting and connecting a client/server. [Unet](https://docs.unity3d.com/Manual/UNetManagerHUD.html) and Mirror both have this feature.
- There is a variety of additional features which could profit from this extension of the transport interface. For instance if we were to add a [LAN NetworkDiscovery](https://docs.unity3d.com/Manual/UNetDiscovery.html) component to MLAPI then this change would allow us to implement it for any transport.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

MLAPI transports expose NetworkAddress and NetworkPort properties. These properties can be used to change the address and port which an MLAPI client connects to at runtime or change the port on which a server gets hosted.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Transports expose properties defined in this RFC for a common interface. They still use transport specific serialize fields to configurate ip addresses and ports in the editor. Setting connection properties should update value of serialize fields.

NetworkAdress: The address the client uses to connect to the server. This is always the final destination address. For a relay transport this would be a room name and not the address of the relay service.

NetworkPort: For a server this is the port on which the application will be exposed. For a client this is the port used to connect to the server. If there is no direct connection (relay/proxy server involved) the port is used to connect to the intermediary server.

Other features such as a user UI for connecting the the servers can use this properties to create a transport agnostic connection interface. For instance the following code could be used:
```csharp
void OnConnectButtonPressed(){
    var ipAddress = m_IpAddressInputField.text;
    var port = ushort.Parse(m_PortInputField.text);
    var transport = NetworkingManager.Singleton.NetworkConfig.NetworkTransport;

    transport.NetworkAddress = ipAddress;
    transport.port = port;

    NetworkingManager.Singleton.StartClient();
}
```

# Drawbacks
[drawbacks]: #drawbacks

- The current approach does not define well how it would support relay based transports with a concept of 'rooms' and 'regions' etc. How NetworkAddress maps to room names or other identifiers used by relay transports is unclear. Would it just be the room name or something along the lines of `$"{region}:{name}."`
- This will break existing transports from MLAPI v12. I consider this a minor drawback because we have other breaking changes already for the transport interface (channels) and upgrading them should be straightforward.
- There where bigger plans to rework the transport interface this could potentially clash with those efforts.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?

Having address/port exposed as properties instead of just passing them into a transport when connecting allows to retrieve and display them at runtime. A string property for NetworkAddress is extensible and can support any kind of connect identifier for future transports.

## What other designs have been considered and what is the rationale for not choosing them?

### 1. Using separate interfaces instead
We could use an interface like this instead and have transports implement it:
```csharp
interface INetworkTransport
{
    string ConnectAddress { get; set; }
    ushort Port { get; set; }
}
```
The advantages of this approach are a) No changes to the transport class b) If a transport does not support ConnectAddress/Port this can be clearly shown by not implementing the `IIpTransport` interface.

The rationale for not chosing this approach is that a connect identifier (address) is a core part of a transport interface which purpose is to create a connection with a server.

This approach could also justify adding a `IRelayTransport` interface later on, along the lines of:
```csharp
interface IRelayTransport
{
    string RoomName {get; set;}
    string ConnectRegion {get; set;}
}
```

### 2. Passing ConnectAddress via NetworkingManager

We could extend the NetworkingManagers `StartHost`, `Startclient` and `StartServer` could take an overload which accepts an IpAddress/Port. The corresponding Transport functions would need that additional overload as well. We then could expose a NetworkAddress/NetworkPort in the NetworkingManager config similar to UNet did this. The reason this approach was not chosen is because it adds a lot of boilerplate code and breaks the single responsibility principle (Transport should be the one dealing with ConnectAddress and not NetworkingManager. NetworkingManager already has way too much functionality packed into a single class.)


### 3. Defining a `ConnectionData` struct

We could define a struct which more explictly defines different types of connect identifiers:
```csharp
struct ConnectionData{
    public string IpAddress { get; set; }
    public ushort Port { get; set; }

    public string RoomName { get; set; }
    public string ConnectRegion { get; set; }
}
```

## What is the impact of not doing this?

Not doing this leaves the transports in a state where they are not easily interchangeable and any features which interact with transport won't be compatible with any transport out of the box.

# Prior art
[prior-art]: #prior-art

Mirror passes the address to the transport in the ClientConnect (StartClient function of MLAPI). Using a similar approach would allow us to not touch the transport class but it moves the responsibilty of handling connection addresses to NetworkingManager.

UNet does not have a concept of transports. A NetworkAddress and NetworkPort can be passed into the NetworkingManager.

Darkrift 2 has network listeners (basically transports) which have an IpAddress and Port configuration.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
    - N/A
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
    - N/A
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
    - N/A
# Future possibilities
[future-possibilities]: #future-possibilities

I cannot think of any future possibilities.
