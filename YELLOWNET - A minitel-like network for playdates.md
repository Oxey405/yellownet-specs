# YELLOWNET - A minitel-like network for Playdates
YELLOWNET communication protocol for Playdate described here.
> Copyright ©️ 2024 - Oxey405 | All rights reserved | under MIT license
## 1. YELLOWNET GOALS :
### 1.1 YELLOWNET is :
- A fun project for playdate users and tinkerers who want to use small online services
- A meta-protocol inspired from HTTP that standardizes connections
- An Open-source project (so **you** can contribute !)
### 1.2 YELLOWNET isn't :
- Something really serious
- Secure (**DO NOT SEND PRIVATE DATA OVER THIS NETWORK** or at least make sur your encryption is strong)

## 2. YELLOWNET basic concepts and inner workings
![[YELLOWNET-playdate-connection.excalidraw]]
Fig. 1 : A typical YELLOWNET setup

YELLOWNET works using message-based packets. Each packet is a string formatted in a special way. (See packet format for more informations)

YELLOWNET clients connect to a server through a gateway with which the client can interact using any interface (typically a Serial over USB connection). The client can send messages destined to the gateway to configure it.

YELLOWNET is **limited by design** but should offer enough flexibility for various use cases and act like very minimal HTTP or Minitel.

## 3. The YELLOWNET protocol
### 3.1 Gateways
#### 3.1.1 Definitions
Gateways are used to bridge the gap between the client and the rest of the network. Therefore the underlying protocol can be anything and depends on the implementation. For all practical purposes and because the client is typically a Playdate ; the standard YELLOWNET gateway implementation will support Serial messages over USB. 
Gateways are to be set up by a client to forward their request to a service.

Gateways are made of two parts : 
- The local handler
- The gateway forwarder
#### 3.1.2 Local handler
The local handler is the part of the gateway that receives messages from the client (generally over Serial USB) and transmits them. It should also validate the packets before sending them.

The local handler must intercept any packets of type `GTW` which are used to send commands to the gateway and change the gateway configuration accordingly. See [[YELLOWNET - A minitel-like network for playdates#3.2.2 Reading and interpreting `GTW` packets|3.2.2 Reading and interpreting `GTW` packets]] for more informations.

#### 3.2.3 Gateways commands
When the gateway receives a `GTW` packet, it should interpret the packet as a setup command. 
See [[YELLOWNET - A minitel-like network for playdates#3.2.2 Reading and interpreting `GTW` packets|3.2.2 Reading and interpreting `GTW` packets]] to understand how `GTW`packets are read.

Here are the basic control commands of a gateway and their expected behavior 
(format : `command <arg1> <arg2> etc...`) :
- `set_address <IP address>` : sets the address towards which the packets should be sent
  response : `forward_address <forward address>` (where forward address is the new address)
- `get_address` : gets the address towards which the packets should be sent
  response  `forward_address <forward address>`
- `close` : closes the socket and terminates the connection
  response : `closed`
- `open` : (re-)opens the socket towards the forward address
- `version` : sends the version string of the gateway. This depends on the implementation of the gateway. 

Note that more commands can be implemented by the gateway but this list are the required commands.

#### 3.2.4 Gateway forwarder
The gateway forwarder is the part of the gateway that is responsible for sending the packets to the actual server on the internet. It must : 
1. Connect to the current address specified by the `set_address` command
2. Send the packets (*except `GTW` packets*) directly and **without tampering them**.
3. Forward received packets directly to the client

### 3.2 Packets 
#### 3.2.1 Anatomy of a packet
All YELLOWNET packets are **ASCII encoded** and follow a precise pattern :
```
<ID>.<TYPE>;<RESOURCE>|<BODY>
```
- `ID` (int16) : The request ID which increments by 1 for each request to a service. It resets per session.
- `TYPE` (string) : The request type. Either `GTW`, `REQ`, `ASW`, `MSG` or `SYS`.
- `RESOURCE` (128 char string) : The resource concerned by the packet. It can be thought of as the last part of a URL but doesn't not have to follow the URL format.
- `BODY` (string) : The content of the request or the response. It needs to always contain at least one byte.


#### 3.2.2 Reading and interpreting `GTW` packets
`GTW` packets are used to set up the gateway. 
The `RESOURCE` of a packet indicates the command that is requested
The `BODY` of a packet includes the argument for that command

#### 3.2.3 `REQ` and `ASW` standard request/answer system
The `REQ` and  `ASW` packets are standard request/answer packets that can be thought of as a minimal HTTP packet. The idea is : the client requests something and the server provides an answer.
The `ID` of the answer (`ASW`) is the same as the `ID` of the request (`REQ`) for simplicity and ease of use.
So `REQ` packets expect and `ASW` packet at some point with the same `ID`. `ASW` packets are not supposed to get sent without a prior `REQ` packet.
![[YELLOWNET-REQ-ASW.excalidraw]]
Fig 2. A REQ/ASW request schema

##### Recommended limitations :
- Request body limit : 4Ko
- Response body limit : 12Ko
##### Typical use-cases
- Requests to a website-like service (like weather, news etc...)
- Analytics
- Update-able content (community levels etc...)
- Social networks (BRING YOUR OWN SECURITY!)

#### 3.2.4 Long-lived connections with `MSG`
Long-lived connections work using the `MSG` packets and can be triggered by anything and unlike `ASW` require no interaction from the client. The ID is still shared and should increment for each message. Both the client and server must keep track of the ID of the last message although `MSG` packets with IDs under the latest one shouldn't be ignored but just reorganized if needed.

#### 3.2.5 The `SYS` packets to communicate about the state of the session
`SYS` (or system) packets are used to signal the state of the session. Those states are signaled to the other party using the `method` field and additional informations are sent in the body.
Here are the exhaustive `SYS` methods and what they tell about the session :
- `gateway_init` : Sent by the gateway to the client to signal its presence and "allows" the client to send packages
- `server_init` : Sent by the server to the client when the client is connected to allow for the client to react accordingly.
- `gateway_disconnect` : Sent by the gateway when it is about to disconnect. This signals the end of the session.
- `server_disconnect` : Sent by the gateway or the server when the server is about to or has been disconnected. Also sent if the server disconnected a client.
- `available_protocols` : Sent by the client to ask for available. Those specs are left without much detail to allow for extension of the YELLOWNET protocol but note that `websocket` and `webrtc`/`udp` are recommended.
## 4. Servers

### 4.1 Basic server stuff
Server must open a UDP or Websocket server and must be able to parse YELLOWNET packets.
There are no real limitations on how a YELLOWNET server works except for the fact that it must respect the specifications of this document.

#### 4.1.1 Security recommendations
Whilst YELLOWNET isn't designed to be really secure, you may want to secure your server. The following list of suggestions may help you avoid the worse :
- [ ] Sanitize user inputs
- [ ] Add Anti-DOS measures (Ratelimits, etc...)
- [ ] Verify your code
- [ ] Implement secure encryption if necessary
- [ ] Isolate the process server
