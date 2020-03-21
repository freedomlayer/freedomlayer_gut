+++
title = "Relaying TCP connections in Offset"
description = "Relaying TCP connections in Offset"
date = 2018-12-08
+++

[Offset](https://github.com/freedomlayer/offset) is designed to be a peer to peer
payment system. However, many users do not have direct access to the Internet
because they are behind a NAT or other restrictive boxes.  This means that peer
to peer connections are sometimes [very difficult to
establish](https://en.wikipedia.org/wiki/NAT_traversal), or even not possible.

As a workaround we had to come up with a relay mechanism, allowing servers to
relay communication between any two clients. Relays federate, and anyone can
open a communication relay. 

The Relays are oblivious to the underlying payment protocols employed by Offset.
The underlying communication is encrypted end to end. The Relay job is only to
transfer this communication. In other words: Payments in Offset are done in a
peer to peer fashion, but the communication is relayed through federated
servers.

There are two types of entities in this world of communication: A Client and a
Relay. A Relay allows two clients to talk to each other:

```
Client --[TCP]-- Relay --[TCP]-- Client
```

Let's begin with the schema of the Relay mechanism of [offset](https://github.com/freedomlayer/offset) :

```
# First message sent after a connection was encrypted.
# This message will determine the context of the connection.
# This messeag can be sent only once per encrypted connection.
struct InitConnection {
    union {
        listen @0: Void;
        # Listen to connections
        accept @1: PublicKey;
        # Accepting connection from <PublicKey>
        connect @2: PublicKey;
        # Request for a connection to <PublicKey> 
    }
}

# Client -> Relay
struct RejectConnection {
        publicKey @0: PublicKey;
        # Reject incoming connection by PublicKey
}

# Relay -> Client
struct IncomingConnection {
        publicKey @0: PublicKey;
        # Incoming Connection public key
}
```

The schema above represents [cap'n proto](https://capnproto.org/) serialization
structures. The full schema code can be found 
[here](https://github.com/freedomlayer/offset/blob/24f40e516e78737946d7a27b66d18ee88c5b58e2/components/proto/src/schema/relay.capnp).
Don't worry if you have never read cap'n proto before, I will soon explain what
each structure is supposed to do.


## How does the Relay work?

(1) A client first opens a TCP connection to a Relay.

(2) The client now sends `InitConnection`, which should be one of three types:

- listen
- accept
- connect

This first message determines what kind of communication will be transferred
over this TCP connection.

(2.1) The client sends `InitConnection::listen` to the relay:

From now on this TCP connection is dedicated for only two types of messages:

- The client may only send `RejectConnection` to the relay.
- The relay may only send `IncomingConnection` to the client. 

We call this kind of connection a **listen channel**. Every client should
maintain an open listen channel to the relay. This connection is used by the
relay to send metadata about incoming connections to the client. The client can
use this connection to reject incoming connections.

(2.2) The client C sent `InitConnection::connect(public_key)` to the relay:

We call such a TCP connection between a client and the relay a **connect channel**.

If the relay doesn't have another client with the given public key, it will
close the connection with C.

If the relay has another client D with the given public key connected on a
listen channel, the relay will send `IncomingConnection` to the client D.


```
     (connect channel)
  C ----------------- Relay
                        |
                        |  (listen channel)
                        |
                        D
```

D then has two options. It can either reject the connection by sending back
`RejectConnection` over the listen channel. In that case the Relay will close
the connect channel with client C.

D can also accept the connection by opening a new TCP connection to the
Relay, as described in the next section (2.3). In that case, the connect
channel opened by client C will turn into a TCP connection to the client D,
relayed by the Relay.

(2.3) The client sends `InitConnection::Accept(public_key)` to the relay:

We call this kind of TCP connection an **accept channel**. This should be done
by the client as a response to an incoming `IncomingConnection` message that
was received over the listen channel.

If there was indeed a pending connection from a client with the specified
public key, the accept channel will turn into a TCP connection to the remote
client, relayed by the relay.


## ASCII diagram

Connect and Accept:

```
                          Relay ---------------------- D
                              <-- InitConnection::listen 


                                   (listen channel)
C ----------------------  Relay ---------------------- D


                                   (listen channel)
C ----------------------  Relay ---------------------- D
InitConnection::connect -->


     (connect channel)             (listen channel)
C ----------------------  Relay ---------------------- D
                                IncomingConnection -->



     (connect channel)             (listen channel)
C ----------------------  Relay ---------------------- D
                            |                          |
                            |                          |
                            +--------------------------+
                              <-- InitConnection::accept


     (relayed TCP)                  (listen channel)
C @@@@@@@@@@@@@@@@@@@@@@  Relay ---------------------- D
                            @                          @
                            @                          @
                            @@@@@@@@@@@@@@@@@@@@@@@@@@@@
                                   (relayed TCP)

```

Connect and Reject:

```
                          Relay ---------------------- D
                              <-- InitConnection::listen 


                                   (listen channel)
C ----------------------  Relay ---------------------- D


                                   (listen channel)
C ----------------------  Relay ---------------------- D
InitConnection::connect -->



     (connect channel)             (listen channel)
C ----------------------  Relay ---------------------- D
                                IncomingConnection -->


     (connect channel)             (listen channel)
C ----------------------  Relay ---------------------- D
                                <-- Reject Connection


                                   (listen channel)
C                         Relay ---------------------- D
  (TCP connection closed)

```


## Why new connections?

If you followed the above description, you probably noticed that every
connection between two clients relayed by the Relay requires opening a new TCP
connection to the Relay (From both clients).

At first sight this might seem wasteful. However, if we want to have reasonable
performance with TCP this might be required. Let me explain. Assume that
instead of the proposed method (Where a Client opens multiple TCP connections
to the Relay), we will only allow one TCP connection between a Client and the
Relay.

```
      Relay
     /  |  \
    /   |   \  
   C    D    E (fast)
      (slow)
```

Consider the topology above. Client C wants to send messages both to Client D
and Client E. Now assume that the connection between the Relay and D is very
slow, but the connection between the Relay and E is very fast.

If C sends data at medium capacity to both D and E, at some point D will not be
able to process the incoming data at the speed sent from C, and it will apply
backpressure back to the Relay, on the `Relay -- D`. The Relay in turn will
apply backpressue on the `C--Relay` TCP connection. As a result, C will have to
send data slowly. Although E could in theory get the data at a higher capacity,
it will only be able to obtain data from C at a slow rate.


