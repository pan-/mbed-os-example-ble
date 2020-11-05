Introduction 
============

The Socket interface abstract several type of connectivity to communicate with 
remotes endpoint.
It claims to not be protocol-specific and allow all protocol to provide the same 
API regardless of the underlying transport mechanism. 

In practice it is used for IP and non-IP cellular communication. 

![alt text](socket_class_diagram.png)

## Features

The Socket interface is used for connection (TCP) and connectionless communication 
(UDP, ICMP). The API is a set of C++ classes and follows loosely the [BSD/Posix 
socket API]. 

### Connected mode 

The connected mode is based on TCP. The client request a connection from the server 
that accepts it. Once the connection has been setup, both end can send and receive 
data through the connection. 

#### Client 

The client creates a `TCPSocket` then connects it to the server using the `connect`
function. When the connection is established, the function `send` is used to send 
data to the server while the function `recv` is used to receive data from the server.
The connection can be closed with a call to the `close` member function. 

#### Server 

The server creates a `Socket`. It configure its address and port using the `bind` 
function then listen for incoming connection using the `listen` call. The system 
will put in a backlog incoming connection. 

When a client attempt to connect to the server, the server socket listening for 
incoming connection receives the request. 
The application should call `accept` to make the connection effective. The result 
is a new `Socket` object where `recv` receives data from the client and `send` 
transfers data to it. The connection can be terminated by calling `close`.   

The passive socket awaiting for connections continue to wait for connection. Each 
call to accept will create a new Socket of the connection. It continues to listen 
for client connections until `close` is called. 


### Disconnected Mode 

In disconnected mode, a connection between the client and the server is not 
established or maintained. The client and the server just sends datagram to each 
other. The reception of datagrams is not guaranteed; nor is the order of reception.

In Mbed OS netsocket API, two datagram protocols are implemented: UDP and ICMP. 


#### Client 

The client creates a `UDPSocket` or `ICMPSocket`. It can send data to the server 
using the function `sendto` and receive data with `recvfrom`. `sendto` requires 
the destination address while `recvfrom` will return the address of the sender. 
The socket is closed when `close` is called. 
 
#### Server  
 
The server creates a `UDPSocket` or `ICMPSocket` then binds it against its local 
address and port. Like the client, `sendto` and `recvfrom` are used to send and 
receive data. `close` terminates the operations. 

#### Pseudo connected mode

It is convenient for the client to not set repeat the destination address every 
time data is sent or filter data received based on a specified address. The API
changes the meaning of `connect` when connectionless mode is used. If connect is 
called on a UDP socket then `send` and `recv` can be used. `send` will transfer 
the data to the address registered in `connect` while `recv` filter out data 
received from an address different than the address registered in `connect`. 


### Scheduling 

By default all operations are blocking the current thread execution until completion:
`send` will return when the operation has complete, `accept` will block until a
peer attempt to connect, `connect` will block until the connection has been 
established, etc. 

The function `set_timeout` set a timeout for all the blocking operations. The 
goal is to prevent the current thread to be blocked forever if no error are 
detected by the system.
  
The function `set_blocking` allows an application to works with socket asynchronously. 
If an operation can't complete or can't be scheduled then it returns the error 
code `NSAPI_ERROR_WOULD_BLOCK`. 
The application can be notified of the completion of the operations by registering 
a callback in the `sigio` function. The system invokes the callback when the state
of the socket changes. For example, if a TCP server socket listen for incoming 
connections, the callback is invoked when a peer attempt to connect. The application
can then accept the connection. 

This is a key difference with sockets from the POSIX world where the application 
ask the kernel to monitor a set of file descriptor using `poll`, `select` or `epoll`;  
the kernel yielding the control back to the application when something of interest 
has happened on these file descriptor. In Mbed OS socket API, the thread running 
the network stack invoke the callback registered by the application when something 
happened on the socket. 

It would be possible to build a `poll` like function but blocking a thread of 
execution is not very efficient for embedded systems. 

Note that `mbed_poll` is **not** compatible with the Socket subsystem. 


### Extensions

It is possible to set stack specific options to a socket using the `setsockopt`
function. The option can be retrieved using `getsockopt`. 

The address of the connected peer can be retrieved using `getpeername`.

The `InternetSocket` class which is a base class of `TCPSocket`, `UDPSocket` and 
`ICMPSocket` exposes extensions based on `setsockopt`: 
 - `join_multicast_group`
 - `leave_multicast_group`
 - `get_rtt_estimate_to_address`
 - `get_stagger_estimate_to_address`

Surprisingly enough, `join_multicast_group` is disabled in `TCPSocket` but not 
`leave_multicast_group`. It is unclear why these two functions were not defined 
in the `InternetDatagramSocket` class. 


### Cellular non IP Socket 

The choice has been made to use the `Socket` abstraction for sending and receiving
3GPP non-IP datagrams (NIDD). When a Socket operates in that mode the following 
functions are not available because they require a local or distant IP address: 
- connect: No address to connect to 
- bind: No address to bind
- listen: No TCP connection to listen for
- accept: Not TCP connection to accept 
- sendto: Not possible to send data to a specific IP address
- recvfrom: Not possible to receive data from an IP address. 
- getpeername: There's no IP address for the peer

There is no common function to set stack specific options to: `setsockopt` and 
`getsockopt`.

## Architecture 

All the `Socket` implementation relies on the `NetworkStack` interface which is 
tied to the socket instance using the `open` member function. 

The `NetworkStack` class offers all the features required to implement a socket:
- `socket_connect`
- `socket_bind`
- `socket_send`
- `socket_recv`
- ...

However all the `NetworkStack` implementations are non blocking and doesn't offer
thread safety. These two features are implemented by the `Socket` layer. 

### Interface 

The BSD socket wasn't designed with subclassing or object oriented as a goal. As 
a result the `Socket` subsystem is odd from an object oriented perspective as the 
Liskov Substitution Principle is broken: It is not possible to replace an implementation 
of `Socket` with another while preserving the program functionalities. 


|              | Connected                            | Connectionless                                           | Non IP         |
|--------------|--------------------------------------|----------------------------------------------------------|----------------|
| connect      | Connect to the peer                  | Set an address for  the default peer                     | Not applicable |
| send         | Send data to the connected peer      | Send to the default peer                                 | Send NIDD      |
| recv         | receive data from the connected peer | Receive data from the default peer. Filter other packets | Send NIDD      |
| bind         | ✅                                    | ✅                                                        | Not applicable |
| listen       | ✅                                    | Not applicable                                           | Not applicable |
| accept       | ✅                                    | Not applicable                                           | Not applicable |
| sendto       | Dismiss the address and send to the connected peer | ✅                                                        | Not applicable |
| recvfrom     | ✅                                    | ✅                                                        | Not applicable |
| set_timeout  | ✅                                    | ✅                                                        | ✅              |
| set_blocking | ✅                                    | ✅                                                        | ✅              |
| setsockopt   | ✅                                    | ✅                                                        | Not applicable |
| getsockopt   | ✅                                    | ✅                                                        | Not applicable |


### Implementations 

Most of the implementations are contained in four classes:
- `CellularNonIPSocket`: Implementation for cellular when `NIDD` is used.
- `InternetSocket`: Contains the common implementation of IP sockets (`open`, 
`setsockopt`, `getsockopt`, `close`, `sigio`)
- `TCPSocket`: Implementation for the TCP protocol. 
- `InternetDatagramSocket`: Implementation for connectionless protocol. The 
`UDPSocket` and `ICMPSocket` classes only provide the _id_ of the protocol used.

The triple `InternetSocket`, `TCPSocket` and `InternetDatagramSocket` is tightly 
coupled due to the use of _protected_ variables sets in the certain way in the 
parent class and their use in the child class. 

## Testing 

All Socket classes are tested with extensive unit tests present in the tests folder
of the netsocket subsystem. 

Being the user facing API these are also tested in the integration test of the 
netsocket subsystem. 

## Conclusion

The socket subsystem is a fair C++ translation of the BSD/Posix socket API. It 
is the front facing part of the netsocket system.  

We can question its application to non IP protocols. The API being closely 
related to IP addresses there is no benefits in using a non consistent API which 
doesn't offer re-usability through substitution for existing software and adds 
confusion to the programmer.

As we're heading into Shadowfax we can question if providing a C++ abstraction is 
the only choice we should offer to users. A compatibility layer with the BSD/POSIX 
C API would be welcomed for compatibility with existing software built around it 
including embedded projects such as CHIP. Other embedded projects like Zephyr or 
LwIP offers such compatibility layer. 

Finally, a bare metal mode that doesn't use blocking operations per default would 
save a bit of code for bare metal mode. The savings would be marginal it is 
probably not worth the effort except if a serious business use case is brought in. 


 
[BSD/Posix socket API]: https://pubs.opengroup.org/onlinepubs/9699919799/functions/socket.html
 
 
