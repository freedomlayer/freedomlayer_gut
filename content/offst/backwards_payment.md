+++
title = "Backwards credit payment"
description = "Backwards credit payment"
date = 2018-11-07
+++

Backwards credit payment is the core of the payments mechanism used in
Offst. It allows to send credits securely through a chain of mutually trusting
friends.

Recall that in the [previous post](./offst/about_offst.md) we showed that
transfer of funds between two nodes could be accomplished by "pushing" credit
along a chain of friends.

```
A -- M1 -- M2 -- M3 -- B
```

In the figure above: `A` can send credits to `B` by "pushing" the credits along the a
chain of friends `(A, M1, M2, M3, B)`.

To examine the security of this setup, we need to consider the incentives of
the nodes participating in the transaction. In this section we introduce a
method to perform a transaction between friends.

## Naive approach

Consider the following graph of friends:

```
A -- B -- C -- D
```

`A,B` are friends, `B,C` are friends, `C,D` are friends. Suppose that `A` wants
to send funds to `D`. We described earlier the general idea for sending funds
from `A` to `B` together with payments to the mediators, but we haven't yet gave
the specifics of how to do this, to make this transaction somewhat atomic and
secure.

Consider first the following naive scenario: `A` wants to send 10 credits to `D`.
Also assume that each node is expecting `1` credit in return for the service of
passing credits.

`A` will calculate how much credit he needs to give to each of the mediators as a
payment for passing the funds all the way to `D`. For example: 1 credit to `B`, 1
credit to `C` and 10 credit to `D`. Next, A will send 12 = 1 + 1 + 10 credits to `B`.
`A` trusts `B` to take 1 credit to himself, sending the rest of the credits (11
credits) to `C`. In the same way, `A` could trust `C` to take 1 credit for
himself, and pass the remaining 10 credits to `D`.

`B` could keep 1 credits to himself, passing 11 credits to `C`. However, it is of
greater benefit for `B` to keep the full 12 credits to himself and not pass any
credits to `C`.

We solve this problem using the idea of **backwards payment of credits**. This idea
is crucial to the operation of payments in Offst.

## Introducing Backwards payment of credits

When `A` wants to send funds to `D`, instead of directly passing all the 12
credits to `B`, `A` will send `B` a promise for 12 credits. This promise will
be fulfilled given a proof that `D` received the correct amount of credits.  In
addition, `A` freezes 12 credits in his mutual credit with `B`. Those 12
credits will not be unfrozen until A gets a proof from `B` that the funds were
delivered to `D`, or until `A` gets a message from `B` notifying that an error
happened while processing the request. In case of an error, all the credits
will be unfrozen and `B` will not be paid.

Next, `B` passes a promise to `C` to pay 11 credits if `C`
brings a proof that the funds were delivered to `D`. `B` freezes 11 credits
in his mutual credit with `C`. `C` then sends a promise to `D` to pay
10 credits if `D` issues a signed message which proves `D` received the funds.

`D` receives the promise message, creates a signature of receipt, and sends it back to `C`.
`C` pays 10 credits to `D`. Next, `C` sends the signature back to `B` and receives 11
credits from `B`. Finally `B` sends the signature back to `A` and receives 12
credits.

Eventually, `A` paid 12 credits, and `B, C` each earned 1 credit. `D` received 10
credits. In addition, `A` has a proof that the funds were received by `D`. 

We distinguish between two stages in this transaction: We call the forward
stage (Sending the message from `A` to `D`) the `RequestSendFunds`, and the backwards stage
(Sending the signature from `D` to `A`) the `ResponseSendFunds`.


```
 ---------[Request]------>
 <---[Response/Failure]---

     A -- B -- C -- D
```

What happens if one of the mediators can not pass the message during the
request stage? For example, if `C` wants to pass the message to `D`, but `C` knows
that `D` is currently not online? 
In this case, `C` will send back a `FailureSendFunds` message to `B`, claiming that the
funds could not be delivered, together with `C`'s signature (as the failure reoprter). 
`B` will unfreeze the 11 credits in his mutual credit with `C`. `B` will then
forward the failure message to `A`. Seeing the provided failure message, A will
unfreeze the 12 credits in his mutual credit with `B`.

In other words: In the case of a failure, no node gets paid, and all frozen
credits are unfrozen.


## Analyzing incentives in backwards credit payment

We now observe various examples of possible scenarious during a backwards credit payment
transaction. Our goal here is to show that it will be the most profitable for
every participant to pass the funds to their destination (When possible).

Note that analyzing the cases below does not mean that the backwards credit
payment is proved to be safe, but currently we do not know of any holes in its
design.


Consider the following network formation between neighbors:

```
A -- B -- C -- D -- E -- F
```

And consider the following cases:

### Not forwarding a RequestSendFunds

`B` receives a `RequestSendFunds` message that he should forward to `C`, but he
doesn't forward the message to C.

This is not a reasonable for `B`, because `B` could potentially earn credits
from this transaction.


### Sending a RequestSendFunds through nonexistent chain

`A` sends a message to a nonexistent remote node `T`, along the "imaginary" chain: `A -- B -- C -- D -- T`.

As a result, the node `D` will send back a `FailedSendFund`
message, signed by `D`, back to `C`. The error response message will eventually
arrive `A`. No payment will occur.


### Receiving RequestSendFunds without paying

`B` receives a `ResponseSendFund` message from `C` but doesn't pay `C`.

An inconsistency will be created in the mutual credit between `B` and `C`,
and new transactions will not continue between `B` and `C` until this inconsistency is
solved manually. 

Usually we assume that `B` will not want to destory his
relationship with `C` for one transaction. Also, note that the amount of
credits earned by `B` by not paying `C` can not exceed the trust `C` configured
for `B`.


### Not forwarding ResponseSendFunds

`C` receives a `ResponseSendFunds` message from `D` but does not pass it to `B`.

This means that `C` gives up on credit, as passing the `ResponseSendFund` message to `B`
will earn `C` credit. Therefore, `C` should probably prefer to pass the `ResponseSendFund`
message to `B`.


### Claiming to be many nodes

An attacker node can claim to be many nodes and seemingly earn more credits for
every transaction. For example, consider the following friends graph:

```
A -- B -- C -- D -- E
     \---------/
       Attacker
```

Where the nodes `B, C, D` all belong to the same attacker. Those nodes possibly run on the
same machine, simulating multiple machines on the friends graph. It could be
very difficult for other nodes to notice that `B, C, D` run on the same machine.

Hence, whenever `A` sends funds to `E` through the chain `A -- B -- C -- D --
E`, the attacker obtains more credit compared to the honest friends graph: `A -
M - E`. This is somewhat equivalent for letting the node `E` ask for more credit for
forwarding funds. 

If a node has control over a critical passage, he could employ such strategy in
order to earn more credits for passing funds. However, we assume that usually
there will be other chains of friends to transfer funds. We also assume that
nodes will try to find a chain that is as short as possible (and as cheap as
possible). Hence when a node simulates an imaginary long chain of nodes, he
earns more credits for passing funds, however, it becomes less likely that his
long chain will be chosen by other nodes.


