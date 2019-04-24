+++
title = "Announcing Offst 0.1.0 🎉🎉🎉"
description = "Announcing Offst 0.1.0"
date = 2019-04-24
+++

We are glad to announce the first public release of Offst!  
Offst is a decentralized payment system, allowing to pay and process payments
efficiently and safely. Offst is an open source project, written in Rust, licensed under
[AGPL3](https://opensource.org/licenses/AGPL-3.0).

You don't need a bank account, credit card, phone number or even an email
address to use Offst. On the other hand, Offst currently doesn't have an
interface to the traditional banking system, and therefore you can not use it
yet to buy food at the grocery store.


**!Warning!** This release of Offst is very initial and not ready to use in
production. **!Warning!**

## Quick links

- [Releases page](https://github.com/freedomlayer/offst/releases)
- [Repository](https://github.com/freedomlayer/offst)
- [Documentation](https://offst.readthedocs.io/en/latest/?badge=latest)
- [Quick Offst tutorial](https://offst.readthedocs.io/en/latest/tutorial/)

## The core idea

Offst is a digital payment system based on [mutual credit](https://en.wikipedia.org/wiki/Mutual_credit).

The common method of running an economy is to divide some amount of money bills
between people and let them play. Usually there are some people in charge of
printing more money bills or destroying money bills to keep the economy going.

Offst works in a different way. 

Quoting From the [documentation](https://offst.readthedocs.io/en/latest/?badge=latest):

Consider Alice, who has two friends, Bob and [Charli](https://en.wikipedia.org/wiki/Charli_XCX).
Suppose that at the same time:

- Bob owes Alice 50 dollars
- Alice also owes Charli 50 dollars. 

```text
    (-50,+50)       (-50, +50)
Bob --------- Alice ---------- Charli

```

Suppose that Charli meets Bob one day, and Bob has a bicycle he wants to sell
for exactly 50 dollars. Suppose also that Charli is exactly looking to buy a
bicycle, and he decides to buy it from bob. In this case Charli can take the
bicycle, and they both ask Alice to forget about the debts on both sides.

In a sense, Charli made a payment to Bob by asking Alice to **offset** the debts.
No real money bills had to pass hands to make this happen.

```text
      (0, 0)          (0, 0)
Bob --------- Alice ---------- Charli

```

If Bob and Charli perform these kinds of transactions very often, it is
reasonable for Alice to charge for forwarding the transactions. For example,
Alice could charge 1 dollar for every transaction she forwards.

Offst is a system that works this way. People or organizations can set up
mutual credit with each other, and leverage those mutual credits to send
payments to anyone in the network. A participant that mediates a transaction
earns 1 credit.

In Offst we call two entities that have set up together mutual credit account
**friends**.

## Setting credit limit

Suppose that Alice and Bob has a mutual credit account. Initially the account
is balanced. If Bob starts buying lots of things along chains that go through
Alice, eventually Bob will have a very large debt to Alice. Alice might not
want that.

Why not? Alice is afraid that Bob will default on this debt one
day. Maybe Alice can trust Bob with certain amount of debt, but not with too
much debt. 

The solution Offst offers to this trust problem is that both sides configure
credit limits.

For example:

- Alice configures a credit limit for Bob: 100 credits.
- Bob configures a credit limit for Alice: 80 credits.

This means Bob can never owe Alice more than 80 credits, and Alice can never
owe Bob more than 100 credits.

From the point of view of Alice:

```text
                    balance
 ----[-----------|-----^-------]--->
    -80          0            100
```

(In the figure above: Alice has control over the right cap, and Bob has control
over the left cap)


Conversely, By putting a limit to the maximum
amount of credits Bob can owe to Alice, Alice has in fact put a limit on the
maximum amount of credits it can own.

In other words: In the closed system of Alice and Bob, The richest Alice can be
is 100 credits. The richest Bob can be is 80 credits. 

As a corollay, the amount of trust (credit limit) a participant puts on other
participants limits the amount of maximum credits the participant can own. This
amount is also the maximum amount of credits a participant can lose by trusting
other participants.


## The value of one credit

The basic amount of value in Offst is **one credit**. One credit is exactly the
amount of value you earn when you help mediate a transaction. 

What is the value of one credit with respect to known currencies? We are not sure
yet. These are questions that we should probably let the market decide. We
might be able to have some estimates though.

1 credit should not be too expensive, because usually one might need to pay a
few credits [^1] as a fee for sending a transaction, and nobody wants to
use a payment system where the fees are too high.

On the other hand, if the value of 1 credit is too small, there is no real
incentive for people to run nodes. From this side, we might conjecture that the
amount of transactions mediated by one Offst node for a day should be able to
cover the costs of running that node.


## Total amount of credits

All participants begin initially with 0 credits, and **the total amount of
credits in the system always remains 0**.
The reason for this is that every excess in credits in one participant is the
dual of lack of credits in another participant.


## How Offst works?

The core of Offst is the credit pushing mechanism. It allows to send credits
along a route of mutual credit edges in a secure way. You can read more about it
in the [project documentation](https://offst.readthedocs.io/en/latest/theory).

Offst does not use a [blockchain](https://en.wikipedia.org/wiki/Blockchain) and
does not contain any form of proof of work. In addition, Offst does not attempt
to achieve a global consensus [^2]. Offst is very efficient. Every transaction
sent using Offst only affects a few other computers in the network (according to the
length of the route).


## Network topology

Offst is made of a few main services:

- Offst node (stnode)
- Offst relay (strelay)
- Offst index (stindex)

Example for part of the Offst network topology:

```text
                  +-------+
                  | Index |
                  +-+---+-+
                    |   |           
              +-----+   +------+
              |                |
         +----+--+         +---+---+          +-------+
         | Index +---------+ Index +----------+ Index |
         +-+-----+         +-------+          +-----+-+
           |                                        |
           |                                        +---+
           |     +-------+         +-------+            |    +-------+
           |     | Relay |         | Relay |            |    | Relay |
           |     +--+----+         +-+---+-+            |    +---+---+               
           |        |                |   |              |        |
           |+-------+                |   |              |        |
           || +----------------------+   +-----------+  |+-------+
           || |                                      |  ||
         /-++-+-\                                  /-+--++\
         | Node |                                  | Node |
         \-+--+-/                                  \---+--/
           |  |                                        |
        +--+  +---+                                    |
        |         |                                    |
     +--+--+   +--+--+                              +--+--+
     | App |   | App |                              | App |
     +-----+   +-----+                              +-----+

```

In the figure above: Two nodes can communicate using relays and find routes
using index servers. Some applications are connected to the nodes.
 

**stnode** is the binary that runs the Offst node service. It is the core part of
Offst that executes the credit pushing mechanism. This binary is what most
users will run on their end device.

**strelay** is a binary that runs the Offst relay service. It is used to relay
communication between nodes. The communication between nodes is end-to-end
encrypted, and therefore strelay can not read the communication. Anyone can run
a relay. We use relays because most users are behind one or more
[NAT](https://en.wikipedia.org/wiki/Network_address_translation)s and
firewalls, and may not be able to listen for TCP connections [^3].

Relay servers are a form of a network meeting point between nodes. It allows
nodes to communicate. Every node must configure at least one relay server to
be able to communicate with other nodes.

**stindex** is a binary that runs the Offst index service. It is used to index
the relationships between nodes and supply routes that can be used to send
funds. 

Every node sends periodic information to a (preconfigured) index server about
his relationship with his friend nodes. The index servers share this
information with each other and use it to index the whole network.

Whenever a node wants to send funds to another node, it first queries an Offst
index for a route. Then the node uses the provided route to send the funds to
the remote node.

Anyone can run an index, but an index is not useful if it is not part of the
index servers federation, because it will not be able to get information about
the full state of the Offst network. Therefore Index server owners should
configure their index servers to talk to each other.


## Offst applications

The way to communicate with Offst node is through Offst apps.
Offst comes with a simple command line app called stctrl. Apps can be
configured to have certain permissions. Offst Apps communicate with the Offst
node using TCP communication (Capnp serialized), therefore apps can be written
in any language by anyone.

An example for `stctrl` invocation:

```bash
$ stctrl -I app0/app0.ident -T node0/node0.ticket info friends
+----+-------+---------------+
| st | name  | balance       |
+====+=======+===============+
| E+ | node1 | C: LR=+, RR=+ |
|    |       | B  =100       |
|    |       | LMD=150       |
|    |       | RMD=200       |
|    |       | LPD=0         |
|    |       | RPD=0         |
+----+-------+---------------+
```

The arguments given to stctrl are: `-I`: the identity of the app (Used for
authentication against the node) and `-T`: the node ticket. The node ticket
contains the node connection info. The subcommand `info friends` shows the list
of currently configured friends.

In the example above the node has one friend (named node1), with balance of 100
credits. The local max debt (credit limit imposed by the remote node) is 150
credits, and the remote max debt (credit limit imposed by the local node) is
200 credits.


## Quick tutorial

How to set up your own node? Check out the
[quick tutorial](https://offst.readthedocs.io/en/latest/tutorial/).


## Public relays and index servers

We set up a pair of relay servers and a pair of index servers to get you
started. We include here their tickets:

- index0:
    - [client_ticket](index0_client.ticket) (For node configuration)
    - [server_ticket](index0_server.ticket) (For index server federation)
- index1:
    - [client_ticket](index1_client.ticket) (For node configuration)
    - [server_ticket](index1_server.ticket) (For index server federation)
- relay0: [ticket](relay0.ticket)
- relay1: [ticket](relay1.ticket)


It doesn't matter which ones you pick. Learn how to configure your offst node
[here](https://offst.readthedocs.io/en/latest/tutorial/).


## Cryptography

Offst depends on [ring](https://github.com/briansmith/ring) for its cryptographic code.
Offst relies on the following cryptographic primitives:

- Diffie Hellman
- Symmetric encryption
- Cryptographic signatures
- Hash functions

All the communication is encrypted.

We chose not to rely on the [traditional TLS certificates authority
hierarchy](https://en.wikipedia.org/wiki/Certificate_authority) because:

- It forces users to buy domain names.
- Using it could give power to a small
    amount of organizations to shut down Offst services.


## License

The Offst project is licensed under the GNU Affero General Public License
version 3 (AGPL3), but Offst applications can be written under any license and
can also be written for commercial usage.

We plan to convert all the interface related crates to the
MIT license, to allow Rust developers to reuse Offst interface to write Rust applications.

Why AGPL3? Because we believe payment related code should always be open
source. This is important both from security and transparency perspectives. If
you have a different opinion about this subject, please [send us a
message](./about/_index.md).


## Future work

Offst is far from done. The following is a very general outline of what we plan
to do next:

- Gathering feedback from preview release

- Offst core (stnode, stindex, strelay):
    - Refactoring codebase
    - Documentation
    - Stabilizing Offst protocol and interfaces
    - Migrating to use an sqlite3 database for node
    - Adding tests
    - Security review

- Offst Applications:
    - Creating a GUI desktop application. (Possibly a browser addon?)

- A mobile GUI application for Offst.


## Contributors

This is a good time to thank people who helped making Offst possible:

- [Kamyuen Tse](https://github.com/kamyuentse)
- [A4Vision](https://github.com/A4Vision)
- [juchiast](https://github.com/juchiast)
- [pzmarzly](https://github.com/pzmarzly)
- [Cy6erlion](https://github.com/Cy6erlion)
- [Nemo157](https://github.com/Nemo157)
- [vitalyd](https://users.rust-lang.org/u/vitalyd)
- [Sam](https://morph.is/v0.8/)
- [amosonn](https://github.com/amosonn)
- BronzeAge
- DreadLord

We welcome contributions from anyone. Don't hesitate to reach out!



[^1]: 
If we believe the idea of [Six degrees of
separation](https://en.wikipedia.org/wiki/Six_degrees_of_separation), then the
average transaction might require a fee of 6 credits.  

[^2]: 
Instead, only a weak form of consensus is obtained along the route where the
funds are pushed.

[^3]:
In the future we can implement smarter logic that attempts to form a real
p2p connection between nodes. At this point we chose the trivial and simple
solution to develop.

