+++
title = "The Trusted Supernode and Distributed Banking"
description = ""
date = 2015-12-08
+++


## Abstract

The Trusted Supernode is an abstract idea for a distributed secure and
efficient banking system. This system allows payment operations that disturb
only small amount of participants. It overcomes adversarial attacks by applying
a useful proof of work, combined with node mixing.

The Trusted Supernode bank system relies at its core on a special form of
trusted entity called the supernode. In addition to its ability to manage
payments, the supernode should allow to securely exchange computation and
storage services for money.

## Motivation

Assume a [mesh network](https://en.wikipedia.org/wiki/Mesh_networking) of
computers (nodes), working together to somehow navigate messages efficiently
across the network.

Life is all good, until at some point one computer (Let's call him $x$) starts
sending a lot of messages. He sends so many messages, that other computers have
problem sending and receiving messages. $x$'s use of the network makes it hard
for other users to use the network correctly.

$x$ is Bob's computer. Bob is a cool guy, but is also a pretty heavy network
user. He transfers large video files, and he also has a digital business that
relies on the networking services of the network.

If this was the normal Internet, at this point Bob will get a call from his
[ISP](https://en.wikipedia.org/wiki/Internet_service_provider) representative,
telling him either to stop what he is doing with the internet, or maybe upgrade
his internet package to a business class client.

But Bob's computer, $x$, is part of a mesh network. Mesh networks are not
managed by any central authority, hence there is no ISP to supervise Bob's
network usage.

It would be nice if the network itself could somehow supervise the network
usage of all the computers. If any computer uses too much bandwidth, he will
have to **pay something of value** to the other computers in the network. This
way, every user of the network could use the network as much as he wants, and
pay according to his use.

If we want to use payments in our network, we need to maintain some kind of a
bank: A distributed bank over the mesh network. In this document we discuss an
idea for creating such a bank. 

This document will not really consider the structure of the mesh network.
Instead, we will assume some kind of underlying communication channel, and
explain how to create a distributed bank on top of it.

## Requirements for Distributed Banking

Let's be more specific about what we want to achieve with our bank. For every
user $x$, a balance value $v_x$ is stored. $v_x$ is the amount of money that
$x$ owns.

Our bank should be able to perform the following operations **efficiently**:

1. Money Transfer: A user $x$ can send user $y$ amount money $u$ if $u \leq
   v_x$. After the money transfer, $x$'s balance value will be decreased by
   $u$, and $y$'s balance value will increase by $u$.

2. Inquiry: A user $x$ can get his current balance $v_x$.

Assume that there are $n$ nodes on the mesh network. Also assume that the
longest path between two nodes in the network is diam(G). (The
[diameter](https://en.wikipedia.org/wiki/Distance_%28graph_theory%29) of the
network G). Whenever we write "efficiently" in this document, we mean that every
operation will disturb at most $diam(G)\cdot polylog(n)$ nodes of the network.

We also want our bank to be somewhat decentralized. We don't have a formal
definition for that, but generally it means that we don't want it to be managed
by a selected few nodes. We want it to be resilient to large amounts of
misbehaving nodes.

## Routing and Distributed Banking

In mesh networks there is a special relation between the problem of message
Routing and Distributed Banking.

Recall that at this document we seek a solution for Distributed Banking in
order to make our message routing algorithm fair. If user $x$ makes a heavier
use of the network than user $y$, then $x$ should pay more than user $y$.

It is interesting to observe the opposite direction of this relation: **If all
that we have is a solution for Distributed Banking, we can use it to implement
efficient routing.**

We roughly describe here this idea:
Assume that we have a mesh network with Distributed Banking, and let $x$ and
$y$ be two nodes. Also assume that $v_x \geq 2$. We want to efficiently route a
message from $x$ to $y$. For the sake of simplicity, assume that we want to
route a single bit $b \in \{0,1\}$ from $x$ to $y$.

First, $y$ performs Inquiry operation, to obtain his current balance. Next, $x$
will transfer to $y$'s $b+1$ money. $y$ will perform another Inquiry. If his
balance has increased by $1$, it means that $x$ has sent the bit $0$. If his
balance has increased by $2$, it means that $x$ has sent the bit $1$.

We assumed that all the operations in the distributed bank are efficient.
Therefore the bit was sent from $x$ to $y$ efficiently.


## About using the Bitcoin network

What about using the [Bitcoin](https://en.wikipedia.org/wiki/Bitcoin) network
as a distributed bank for our mesh network?

In the Bitcoin network every money transfer between two nodes must be
broadcasted to all the nodes in the network. In other words, the Bitcoin
network does not allow to perform efficient money transfer.

Recall that our original motivation to implement a distributed bank is to be
able to route messages in a fair and efficient manner along the network. If our
distributed bank does not allow efficient money transfer, we can not rely on it
to implement efficient routing.


## Initial Attempts

Consider a world of $n$ people. How could we create an economy in this world,
where every person can pay another person, and payments have some meaningful
value?

A common method to achieve such an economy would be to use cash: Bills and
coins. Those are artifacts that are hard to produce. A limited amount of bills
are printed, and people can exchange those bills as payment.

In the digital world of computers, we don't have an exact equivalent for cash.
This happens because digital objects can be duplicated easily. This problem is
sometimes referred to as [Double
Spending](https://en.wikipedia.org/wiki/Double-spending):
Assume that a user $x$ owns a digital bill. He could duplicate the digital bill
and pay the same bill simultaneously both to $y$ and $z$.


Instead of using Cash, we could use some concept of a bank instead. A balance
will be kept for every person. People can pay each other by updating their
balances accordingly (Increasing the destination balance, and decreasing the
source balance).

Now we have the problem of where and how to keep the balances.

A naive solution would be to let every person $x$ keep his own balance value,
$v_x$. This solution is also distributed: Every person has to remember just one
value. A transaction between people $x$ and $y$ only involves $x$ and $y$. If
$x$ wants to transfer $k$ money to $y$, $x$ decreases his own balance by $k$,
and $y$ increases his own balance by $k$.

The problem with this model is obvious: What if $x$ becomes greedy, and decides
to increase his own balance? Our economic system will be compromised.

At this point we understand that we can not trust all people to be honest.
Another solution would be to let just one special person $z$, to keep the
balances for all the nodes in the network.

Whenever $x$ wants to send $k$ money to $y$, $x$ will file a request
to $z$, saying that he wants to send $k$ money to node $y$.

$z$ will inspect $x$'s balance and make sure that $x$ has at least $k$ money.
If this is the case, $z$ will subtract $k$ from $x$'s balance, and add $k$ to
$y$'s balance.

![A transaction processed by z](centralized_bank_z.svg)

*In the picture: A transaction as processed by a centralized bank $z$.*

This solution is somewhat equivalent to the usual centralized bank. (The banks
that we have in the real world).

The weak point of the centralized bank is with $z$: The person that manages the
bank. He has full control over all the balances of all the participants.

In addition, every money transfer between two people $x$ and $y$ must go
through $z$. Therefore the economy can be only as efficient as the ability of
$z$ to process transactions.


## Introducing the trusted subset

> So the Lord scattered them abroad from there over the face of all the earth,
> and they ceased building the city. (Genesis 11:8)

Let $S$ be set of $n$ people. We have noticed that if we want to create an
economy between those people, we have to take into account the fact that some
people might try to cheat, given the opportunity.

To be able to talk formally, we need to have some model of when people might
cheat. We choose the following simplistic model: Every person in $S$ is either
**Good** or **Bad**. A good person always goes by the rules. All the Bad people
collude together to try to subvert the system.

We are going to assume that there are more good people than bad people in $S$.
There is some philosophy behind this assumption and we won't get to it now. For
now, think about it as giving the good people some kind of advantage. All
other things equal, it is hard to believe that a minority of good people in $S$
will be able to hold together a secure distributed bank. 

On the opposite side of things, it is reasonable to believe that a set with a
majority of good people is able to make correct decisions somehow with respect
to banking operations: Maybe through some kind of a smart voting system.

Let $x$ be some person in the network. We want to keep $x$'s balance on the
network, somehow. We have already noted that keeping this balance with just one
person is too centralized and fragile. On the other end of the spectrum,
making all the people in $S$ remember $x$'s balance at all times is very
resilient, but too inefficient. As a middle ground, we will choose a small
subset of people and let them remember $x$'s balance.

To make sure the members of the small subset do not collude to do something
bad, we pick the small set randomly.

Let $T \subseteq S$ a random subset of $S$. As there are more good people than
bad people in $S$, we expect that $T$ will also contain more good people than
bad people. 

Let's put in some numbers. Assume that the total amount of people is
$n=2^{r}$, and that the amount of bad people is $\alpha$ of all the people.
Next, we choose random sets $T$ of size $\log{n} = r$. What is the probability
$p_r$ that the amount of bad people in $T$ is more than $\beta$ of $T$?

$n$ is very large with respect to $|T|$, so for the ease of computation we will
assume that our population is infinite (And every person is bad with
probability of $\alpha$). This will give us a slightly wrong result, but not
too wrong. Now that the draw of a person from the pool of infinite person is
independent of other draws, we can use the [Multiplicative form of
Chernoff bound](https://en.wikipedia.org/wiki/Chernoff_bound) to
estimate $p_r$.

$$ Pr[X > (1+\delta)\mu] < {\left(\frac{e^{\delta}}{(1+\delta)^{(1 +
\delta)}}\right)}^\mu $$

In our case $\mu = r\cdot\alpha$, which is the expected amount of bad people
in $T$. We pick $\delta = \frac{\beta - \alpha}{\alpha}$ to get:

$$ p_r \approx Pr[X > {\beta}\cdot r] < {\left(\frac{e^{\delta}}{(1+\delta)^{(1 +
\delta)}}\right)}^\mu $$

If one out of 9 people is bad ($\alpha = 1/9$), the probability of having a
random subset $T$ of size $r=256$ with more than $\beta=1/3$ bad people is
(Calculation done with python):

    In : math.log2(calc_bad_prob(256,1/9,1/3))
    Out: -53.176815513188735

We got a probability of at most $2^{-53}$. This is a very unlikely event. So
with huge probability such a random set $T$ has less than $1/3$ fraction of bad
people.

This is the code used to obtain this result (Python3):

```python
import math

def calc_bad_prob(r,alpha,beta):
    """
    r - Size of a subset.
    alpha - Probability of a node to be bad.
    beta - Amount of bad nodes in a subset to turn it into a bad subset.

    Calculate upper bound for Pr[X > beta*r] using the Chernoff bound.
    This calculation assumes infinite sampling population.
    """
    delta = (beta - alpha) / alpha
    miu = alpha * r # miu = E[x]

    bound = (math.exp(delta) / ((1 + delta)**(1 + delta))) ** miu

    return bound
```


The above choice of the number $1/3$ was not arbitrary. There is a family of
algorithms called **Asynchronous Byzantine Agreement**. (We will call it ABA in
this document). An ABA algorithm can turn a set of $M$ of people with less than
$1/3$ bad people into one "virtual good person".

How does it work? An ABA algorithm is some kind of a smart vote between the
members of the set. It allows the set of people to make correct decisions with
very high probability, despite the existence of a minority of bad people in the
set. (We will give the details of ABA in this document, but you can learn
about it from Ran Canetti's Thesis: Studies in Secure Multiparty Computation and
Applications, chapter 4.)

You might be wondering: Why didn't we use ABA from the first place, on the
whole set $S$? The reason is efficiency: Making all the people in $S$ vote over
every money transfer operation is too inefficient. Instead, we pick small
subsets of size about $\log(n)$, and run the ABA algorithm on them.

As a result we get "trusted subsets" of $S$.


## Basic banking operations

Now that we have the concept of trusted subsets, we can use it to store money
balances and perform money transfers. We describe here the general idea behind
the basic banking operations.

For every person $x$, we match a random subset $T_x \subseteq S$. The members
of $T_x$ will always keep the current balance of $x$.
Note that it is possible for two different people, $x$ and $y$, to have
intersecting random subsets. In other words, it is possible that $T_x \cap T_y
\neq \emptyset$

**Inquiry:** $x$ can perform an Inquiry operation by sending a request to the
members of $T_x$. The members of $T_x$ work together as one virtual good
person, using the ABA algorithm. Hence they will send back to $x$ his current
balance.

**Money Transfer:** If $x$ wants to send $k$ money to another person, $y$, $x$
can send a request to $T_x$ to transfer $k$ money to $y$. The members of $T_x$
will work together as one virtual good person: First they will check that the
current balance of $x$ is at least $k$. Then $T_x$ will decrease the balance of
$x$ by $k$. Next, $T_x$ will send a request to $T_y$ to increase the balance of
$y$ by $k$. The members of $T_y$ *should be able* to trust $T_x$, because $T_x$
is a randomly chosen subset of $S$, and it acts like a virtual good person.
Finally the members of $T_y$ will add $k$ to the balance of $y$.

Note that we implicitly rely here on some method for supernodes to be able to
find each other. We will discuss it later.

The introduction of trusted subsets of $S$ creates a world of virtual
participants, where all the participants are good. This setting allows some new
possibilities that we will talk about in the future.


## The lifetime of a Supernode

To make our discussion more exact, let's go back to talk about computers.
We are given a set of $S$ nodes that can somehow communicate efficiently, and
we want to implement a distributed bank. Every node $x$ is matched with a
random subset $T_x \subseteq S$ that keeps his balance.

We begin by addressing an important issue with respect to computers: They are
not always online.

Let $x$ be some node. Assume that some of the nodes in $T_x$ get offline, or
end their lifetime (It happens to computers too!). Over time it is possible
that no nodes from $T_x$ will be online. How can we make sure that $x$'s
balance will not be erased?

To solve this issue, $x$'s subset $T_x$ must be dynamic. It must be able to
change its members over time, with respect to changes in the network. We will
call the dynamic set of nodes that keep $x$'s balance $sup_x$, or: $x$'s
**Supernode**. This new name denotes the fact that $x$'s balance is kept by a
dynamic entity, not by a static set.

If some node dies, it should be somehow replaced by a new randomly selected
node. In addition, members of $sup_x$ should be removed randomly from time to
time, to keep the random structure of $sup_x$. (We don't want bad nodes to stay
forever in $sup_x$ while good nodes leave). The supernode has the
responsibility of keeping itself alive.

However, The dynamic nature of Supernodes poses some new trust questions.
Consider this example: Let $x$ and $y$ be two nodes, and assume that $x$ wants
to transfer $k$ money to $y$. $x$ contacts $sup_x$, which subtracts $k$ from
$x$'s balance, and then sends a request to $sup_y$ to add $k$ to $y$'s balance.

How can $sup_y$ know that it can trust $sup_x$? Maybe $sup_x$ is some
artificial set that was invented by $y$, because $y$ wants to increase his
balance? 

We break this question to smaller parts. 
Assume that a node $z$ once knew that a set $G$ of nodes is a supernode. After
a while while he encounters a different set $G'$, which claims to be the same
supernode as the original set, $G$. How can $z$ verify that $G'$ is the same
supernode as the original $G$?

To "transfer the trust" from $G$ to $G'$, we can use [digital
signatures](https://en.wikipedia.org/wiki/Digital_signature). Assume that each
node can cryptographically sign a block of data. 

Let $G_1$ be the set of nodes of some supernode.
Assume that some node $z$ knows that the set of nodes $G_1$ is a supernode. 
Now assume that some node has joined $G$, or some node has left $G$. We mark
the new structure of the supernode to be $G_2$. Every node from $G_1$ signs the
new structure of the $G_2$. The set of all those signatures is a proof that the
supernode has changed its structure from $G_1$ to $G_2$.

The supernode keeps changing over time. $G_2$ changes to $G_3$ and so on.
Whenever it changes, a new set of signatures is added, as an evidence for the
structure change. The members of the supernode keep the chain of sets of
signatures at all times.

After a while, $z$ encounters a some set $G' = G_u$ of nodes. $G'$ can then
prove that it is the same supernode as the original $G_1$ by showing $z$ the
chain of all signatures, leading from the original structure $G_1$, all the way
to $G'$.

$z$ knows that the original set, $G_1$, was a supernode, therefore its members
were somehow sampled randomly from $S$. Hence in the set of signatures proving
the change from $G_1$ to $G_2$, there must be a signature of a good node. $z$
doesn't know which one of those signatures was signed by a good node, but he
knows that at least $2/3$ of those signatures were signed by good nodes.

A good node will not sign over fake structure changes. Therefore $z$ can be
sure that indeed the new set $G_2$ is also a supernode. If, for example, the
change was adding a new node, $z$ can be sure that this new node was sampled
randomly from $S$.

In the same way, $z$ can be sure that the change from $G_2$ to $G_3$ is valid,
and so on. Finally, given a valid chain of signature sets, $z$ can be sure that
$G'$ is the same supernode as the original one, $G_1$.


## An example for sets of signature chains

The above explanation will not be complete without an example, so let's create
one.

Assume that a supernode $sup$ is made of the following nodes:

    (A,B,C,D,E)

Now a new node $F$ joins the supernode. The new structure of the supernode is
as follows:

    (A,B,C,D,E,F)


The signatures that prove the change from the old structure to the new
structure are as follows:

    sign_A((A,B,C,D,E,F))
    sign_B((A,B,C,D,E,F))
    sign_C((A,B,C,D,E,F))
    sign_D((A,B,C,D,E,F))
    sign_E((A,B,C,D,E,F))

Now assume that the node $B$ went offline, and this was recognized by the
supernode. The supernode removes $B$ from the set of its members. This is the
new structure of the supernode:

    (A,C,D,E,F)

And these are the signatures that prove the change from the old structure to
the new structure:

    sign_A((A,C,D,E,F))
    sign_C((A,C,D,E,F))
    sign_D((A,C,D,E,F))
    sign_E((A,C,D,E,F))
    sign_F((A,C,D,E,F))

Note that $B$ has not signed the new structure, because he went offline. The
signatures of the other nodes $\{A,C,D,E,F\}$ should be enough, as many of them
are good nodes.


If the supernode $sup$ wants to prove the changes that happened to him since
his initial structure $(A,B,C,D,E)$ to his current structure $(A,C,D,E,F)$, he
could show the two sets of signatures from above.


## The birth of a Supernode

The section about "lifetime of a supernode" explains how supernodes change over
time. However, it does not explain how they are initially created.

Assume that $x$ has some supernode $sup_x$. $y$ can ask $x$ to create a new
supernode for him. In that case, $x$ will send a request to $sup_x$ to create a
new supernode $sup_y$ for $y$.

The supernode $sup_x$ will then randomly select a subset of $S$, and call it
$sup_y$. It will also initialize some balance value for $y$. All the members of
$sup_x$ will sign over the structure of the newly created supernode, $sup_y$,
and hand those signatures to the members of $sup_y$, together with all the
signatures of the history of changes $sup_x$.

Let $z$ be some node. Assume that $z$ knew in the past that some set $G$ is
the set of members of the supernode $sup_x$. If $z$ ever encounters a set of
nodes $P$ that claim to be the supernode $sup_y$, $z$ will be able to verify
this statement by verifying the chain of signature sets all the way from the
supernode structure $G$ to the creation of $sup_y$. 

$z$ will have to verify all the changes that happened to $sup_x$ since the
member set $G$, the creation of the supernode $sup_y$, and then all the changes
that happened to the supernode $sup_y$ until its current structure, $P$.

It is hard to create a new fake supernode, because the signatures of members of
a real supernode are required, and a real supernode contains a majority of good
nodes, which will not be willing to sign.

We described here a way to create a new supernode, given that we have a
supernode already. We are going to assume that our network have some initial
supernode, and all the supernodes in the network will be his children, in some
sense.

Every supernode $sup$ will keep in memory a chain of set of signatures,
describing everything that happened until its creation. This chain is used as a
proof that $sup$ is a valid supernode.


You must be worried right now about the size of the chains that every supernode
has to remember, and you are correct: The chains keep growing as time goes by
and the supernodes change. Generally, every supernode should remember a chain
which is of size about $O(t)$, where $t$ is the time that has passed since the
beginning of the network. We will talk about a solution for this problem later.


## The death of a Supernode

Given that $x$ has created a some supernode $sup_x$, should this supernode stay
alive forever? 

Managing a Supernode is not a full time job for its members, but it does take
some resources. The supernode members have to respond to various requests to
the supernode, find new members or remove old members from time to time.

To make sure that supernodes are not created carelessly, it is possible to
deduct some money from the supernode owner on a timely basis.
The supernode $sup_x$ will decrease $x$'s balance by a small amount every few
minutes, for example. This is easy to implement, because the supernode $sup_x$
is the supernode who maintains the $x$'s money balance.

Using this method, the supernode will only keep living if $x$ has enough money
to "support" it. When $x$'s balance becomes $0$, the supernode will disassemble
and die.

Using this method we can also allow one node $x$ creating more than one
supernode. $x$ can use each supernode that he creates as a separate wallet, or
even, a separate instance of a virtual trusted computer.


## Introducing Chain Shortening

It could be nice if every supernode $sup$ will have to remember just a short
proof for his realness, instead of remembering a full chain of sets of
signatures all the way to the first supernode ever created.

I searched for a magic cryptographic technology that will enable to compress a
chain of signature sets into a small structure, but I never found one. (I did
find something called [Transitive
Signatures](https://eprint.iacr.org/2004/215.pdf), but never managed to use it
to solve this problem)

Instead, we will use some communication. (It's a general truth of life that
lack of cryptographic technology forces one to use more communication).

We begin by uniquely naming supernodes. Whenever a supernode creates a new
supernode, it gives the new supernode a new random name.

Let $W$ be a special supernode. It does not belong to any node in the network,
and it lives forever. Every node in the network was created by $W$ or one of
his descendent supernodes. The network is bootstrapped by creating $W$, and
then letting $W$ create an initial $sup_x$ supernode for some node $x$.


Being a special supernode, $W$ is the only supernode with the privilege of
broadcasting messages to the whole network. Every time a change happens to the
structure of $W$, it broadcasts the new structure, including a proof: A set of
signatures over the new structure.

For every node $y$ on the network, on receipt of a message from $W$,
broadcasts the message to all of his immediate neighbours. Every node in the
network should know the current structure of $W$, and also remember the
structure of $W$ in the last $C$ changes. 

$C$ could be some large number, for example $C = 50000$. This should not cost
too much memory for every node in the network, but at the same time allow every
node to remember all the history of changes of $W$ in the last half year.


Consider some supernode $sup$ in the network. $sup$ members maintain a chain of
signature sets that describe and prove the history of changes of $sup$ since
the beginning of the network. $sup$ can use the existence of the ever living
supernode $W$ to shorten this chain.

$sup$ will send $W$ his current chain. The supernode $W$ will then inspect the
chain. If $W$ manages to recognize the beginning of the chain (The beginning of
the chain should be some ancient structure of $W$), and all the signatures are
correct, then the members of $W$ will produce a new set of signatures over the
supernode $sup$. 

From now on, instead of using the old chain of signatures, $sup$ can use $W$'s
signature over his current structure as a proof for his validity.

We call the above process **chain shortening.**

![Basic Chain Shortening](basic_chain_shortening.svg)


$sup$ should perform the chain shortening process regularly to keep his proof
chain short, and to stay alive. If $sup$ stops shortening his chain for a long
enough time, no node will be able to recognize him. This amount of time depends
on the choice of constant $C$.

How can a node $z$ verify such a proof chain of a supernode $sup$?
Every node $z$ knows the current and previous structure (Up to $C$ steps
backwards) of the supernode $W$. $z$ will verify that the proof chain begins
with some known structure of the supernode $W$, and that all the signatures up
to the current structure of $sup$ are valid.

The problem with the described mechanism is the large amount of work $W$ needs
to do. $W$ has to deal with chain shortening requests from all the supernodes
in the network. This would be too much for one supernode to deal with.


## Efficient Chain Shortening

We need to take off some of work of chain shortening from the supernode $W$. We
could, for example, let $W$ do the chain shortening for just a few supernodes.
Then each of those supernodes will do the chain shortening for a few other
supernodes.

This kind of hierarchy can leave every supernode with a proof chain of length
at most $O(log(N))$, where $N$ is the amount of supernodes in the network. At
the same time, every supernode has to perform the work of chain shortening only
for a few other supernodes, which is not very expensive (With respect to
computation or communication).

Assuming that we have some kind of efficient communication infrastructure, we
can arrange the supernodes into a DHT in order to achieve this kind of
hierarchy. We are going to use the [Chord
DHT](@/research/dht_intro/index.md) here.
Recall that a DHT is a structure where every participant is directly connected
to a logarithmic amount of other participants. In addition, every participant
can reach another participant using logarithmic amount of hops.

Every supernode is given a unique name on its creation time. This name will be
used as his DHT Identity. $W$ will also be on this DHT. Every supernode in the
DHT will perform chain shortening for all his connections in the DHT (If
possible), every constant amount of time.

This means that every supernode $sup$ on the DHT receives proposals for proof
chains every constant amount of time. $sup$ will always maintain the shortest
valid proof chain he was offered.

Hence, Every supernode in the DHT will have a proof chain which represents his
shortest path to $W$ on the DHT. In the chord DHT the longest distance is
logarithmic. Therefore the shortest proof chain should be of size at most
$O(\log(N))$, where $N$ is the amount of supernodes in the network.

Note that the resulting DHT will be connected, because of the ability of $W$ to
broadcast.

We can use this same DHT as a tool for **discovery of supernodes**. A supernode
$sup_x$ can find another supernode $sup_y$ by searching for $sup_y$'s unique
name in the DHT.

## Network Stickiness

Assume that some node $x$ connects to the network for the first time. $x$
learns about the current structure of $W$, and is given a copy of the history
of structure of $W$ (Of size $C$ steps backwards).

If $x$ ever connects again to the network after some not too long amount of
time, $x$ can show his last known structure of the supernode $W$ to any other
node $y$, and receive back a proof chain from that old structure all the way to
the current structure of $W$. This is possible because all the nodes in the
network remember a long enough history of changes to the structure of the
supernode $W$.

Using this method, $x$ can also verify that he has arrived to the same network
like the one he saw in his first visit. $x$ can not be put into a different
network, and be fooled to believe that it is the original one.

We call this property **Network Stickiness.**

## Limiting the amount of bad nodes

Earlier we have chosen the following model: We have a world $S$ of $n$ nodes:
Most of them are good, and the rest are bad.

How can we enforce this kind of ratio between the nodes in the network? What
stops an adversary from inserting many bad nodes in the network? ( This kind of
attack is also known as [Sybil
Attack](https://en.wikipedia.org/wiki/Sybil_attack).)

The solution would be to require nodes joining the network to sacrifice some
rare resource. In some philosophical sense, this allows us to link real value
from the outside world into the network.

A simple idea would be to use [Proof of
Work](https://en.wikipedia.org/wiki/Proof-of-work_system). Every node that
wants to be part of a supernode must perform difficult calculations regularly.
Assuming that no adversary holds majority of computation power, we can assume
that it should be computationally difficult for any adversary to have majority
of bad nodes inside the network.

The main disadvantage of using this raw form of "Proof of Work" is that we
waste resources. 

Another option would be to require an indirect proof of work: **Instead of
requiring nodes to perform difficult calculations, we can ask nodes to pay.**
As money in our bank stands for real value, we can use it as a rare resource
that nodes sacrifice.

In this method, every node $x$ that wants to join a supernode must lost money
regularly. For example: $x$ has to lose $k$ money every few minutes. Where
does the money $x$ pays go? It is just gone. This is equivalent of spreading it
equally between all the nodes of the network, but easier to do.


## The Supernode as a trusted general purpose computer

Recall that the supernode is a set of nodes, with a majority of good nodes,
that work together like one virtual good node. So far we have talked about the
supernode as means for storing balance for one node, but the supernode can do
much more. We mention here some very abstract ideas of what can be done.

Let $x$ be some node, and $sup_x$ be a supernode owned by $x$. The supernode
$sup_x$ can be used as a trusted remote computer owned by $sup_x$, supplying
various services in exchange for $x$'s money. Using the Asynchronous Byzantine
Agreement algorithm $sup_x$ acts a virtual one good node, so $x$ can trust
$sup_x$. In addition, $sup_x$ has direct access to $x$'s money balance. These
two facts make a perfect setting for $x$ and $sup_x$ to do some business.

Also note that $sup_x$ keeps living as long as $x$ still has money in its
balance. Therefore $sup_x$ can keep working even when $x$ is offline.


Letting the members of $sup_x$ perform computational or storage services might
not be a good idea, because $sup_x$ has only so many members, and they might be
limited in their computational or storage abilities.

Another possibility would be to let $sup_x$ manage workers for $x$. The workers
will supply storage or computational services. The supernode $sup_x$ will
handle the communication. 

Assume that $x$ wants to store some files. $x$ will first send a request to
$sup_x$ to allocate storage space. Next, $sup_x$ will hire some random nodes
from the network (That requested to be hired for storage purposes). $x$ will
then send blocks of the files he wants to store to $sup_x$, and $sup_x$ will
store those blocks redundantly on the workers it has just hired.

The storage workers will be paid from $x$'s balance, and they will be payed on
regularly (For example: Every few minutes/seconds) for their storage work.

At a later time, $x$ could request the blocks he stored earlier from $sup_x$.
The supernode will then contact the workers and retrieve the stored blocks.
Finally, the stored blocks will be sent back to $x$.

A similar idea could work for computation workers: The supernode $sup_x$ can
hire a few random nodes from the network (That requested to be hired for
computation purposes). Then it can run the same computation on all of those
nodes, and take the majority answer. Computation workers could be paid by the
instruction (Some kind of basic computational unit).

$x$ can also use his supernode $sup_x$, as some kind of a server. He can load
it with some code and data, and allow it to run and serve other nodes of the
network. $sup_x$ might be programmed to charge money for its services to other
nodes of the network. This way, $sup_x$ can stay alive as long as it keeps
giving value to users of the network, even long time after $x$ has died.


## Future Research

The Trusted Supernode model of distributed bank was presented here as pretty
abstract idea. Many pieces are still missing to make this idea a working
system. We list some of them here:

1. How to balance the economic system? Recall that a node has to lose money
   when working inside a supernode. Where does new money come from into the
   network?

2. Incentives for nodes to work in a supernode: If working as part of a
   supernode costs money for a node, why would a node want to do that?
   Translating to the language of Bitcoin: If miners don't make money from
   mining, will they mine anyways?

3. Efficiency: The supernode works as one trusted general purpose computer, but
   it is pretty slow, because it is spread across the whole network (His
   members are chosen randomly from the network). How practical would it be to
   run a supernode of ~$256$ members over the Internet? Are there other ways to
   choose supernodes so that their members are not so far away, and still
   maintain secure banking?

4. Connecting the Trusted Supernode bank system to the Mesh network: How to pay
   for sending messages using the proposed banking system? How to avoid giving
   money to nodes in the mesh that don't route correctly? How can supernodes
   communicate efficiently in a mesh network? How to deal with a network split?

5. Creating an efficient Asynchronous Byzantine Agreement algorithm. Some
   efficient ABA algorithms are already known, but it takes some work to apply
   those algorithms for our supernode use case.


## Further reading

- [Making Chord Robust to Byzantine
Attacks](https://www.cs.unm.edu/~saia/papers/swarm.pdf) By Amos Fiat, Jared
Saia and Maxwell Young

- [How to Spread Adversarial Nodes?
Rotate!](http://www14.in.tum.de/personen/scheideler/papers/f85-scheideler.ps.gz) By
Christian Scheideler

- [Studies in Secure Multiparty Computation and
Applications](http://www.cs.tau.ac.il/~canetti/materials/thesis.ps) By Ran
Canetti

- [Random Oracles in Constantinople: Practical Asynchronous Byzantine Agreement
using Cryptography](https://www.zurich.ibm.com/~cca/papers/abba.pdf) By
Christian Cachin, Klaus Kursawe, Victor Shoup

- [Bitcoin: A Peer-to-Peer Electronic Cash
System](https://bitcoin.org/bitcoin.pdf) By Satoshi Nakamoto

