+++
title = "About the Offst project"
description = "Introducing the Offst project"
date = 2018-10-25
+++

I want to introduce you to an idea about how to transfer value between people
through the Internet, a technology called **credit switching**. This is a
project I have been working on for a while, and it is not yet done (so nothing
yet to play with), but I figured out that by now the idea is worth sharing,
both for the purpose of clearing my thoughts about it, and also to hopefully
get some feedback.

You can find the project on github. It is called [Offst](https://www.github.com/Freedomlayer/offst),
released under the GNUv3 license, written in Rust.

If you have been reading the news lately, you probably heard about the idea of
the blockchain, and how it can be used to create a shared agreed upon ledger.
The blockchain uses proof of work, and the assumption that no single entity may
obtain a majority of computation power, as its core security assumption.

Credit Switching is somewhat different. It does not rely on a blockchain, nor
it assumes anything about computation power. The security in Credit Switching
is derived from trust between people. To be more precise: from contracts
between pairs of people that trust each other.

There is much that I want to tell you, but we need to start somewhere.
Let's start from a basic example.

## Mutual Credit

Assume that two business owners, a milkman and a baker, live next to each
other. Also assume that they live in a world without money.

One way the milkman and the baker could exchange goods without the use of money
is using the method of barter. The milkman could ask the baker from some bread,
in exchange for some milk. However, what if at a specific time the milkman
wants to get some bread from the baker, but the baker doesn't need any milk
from the milkman? In this case, the method of barter will not allow to perform
a transaction. Barter only works if the two involved parties are interested in
the exchange in a specific point in time.

To solve the problem, the milkman and the baker could use some kind of
temporary credit: If the milkman wants to get bread from the baker, but the
baker doesn't need any milk right now, the baker will give the milkman a loaf
of bread, and write down on a piece of paper that the milkman owes him some
credit. The milkman will also write a similar note for reference. Maybe even
both of them sign the notes, to make sure both sides keep their commitments
later. We call this idea the mutual credit between the milkman and the baker.

A few days later later, if the baker wants to get milk from the milkman, the
baker could use his credit at the milkman to get milk. After the milkman gives
the baker a bottle of milk, the debt of the milkman to the baker is cleared.

This method allows the milkman and the baker to perform exchange of their
goods, even if they do not want to exchange them at exactly the same time. 

One assumption that mutual credit makes is that the milkman and the
baker trust each other. Suppose that the milkman takes 5 loafs of bread from
the baker. The milkman and the baker both write a note that the milkman ows the
baker credit that equals 5 loafs of bread. What if the milkman is gone one day,
without ever repaying the debt to the baker? The baker is left with a note that
the milkman ows him credit, but this note can not be redeemed anywhere for
goods.

The baker, by allowing the milkman to have debt to him, trusts that the milkman
will stick around to be able to pay back the debt in the future, in some way.

The note that the baker owns proves that the milkman have debt to him. If they
ever meet in court, this note (if prepared correctly) should be able to prove
that the milkman owes money to the baker.

## First steps

Some measures to be taken when using mutual credit clearing:

- Mutual credit clearing should only be set up between people that trust each
    other, or in a setting where a debt note could be used in court. 

- The maximum debt possible in the mutual credit clearing between two sides
    should be limited to some maximum amount. For example: If the baker allows the milkman to owe
    him credit of value at most 3 loafs of bread, the baker will not be able to lose
    more than 3 loafs of bread's value if A can't repay the debt.

Given these assumptions, the state of mutual credit between two nodes A and B
consists of the following parameters:

- A allows B maximum debt of `md{AB}`
- B allows A maximum debt of `md{BA}`

md{BA} and md{AB} are not necessarily equal. Drawn on a one dimensional scale,
we get a picture as follows (from the point of view of A):

```
                     d{AB}
     [-----------|-----^-------]
  -md{BA}        0           md{AB}
```

In the picture, B owes A some amount of money. The `^` marks shows the current
credit relationship between A and B. The value `d{AB}` is the mutual credit state
between A and B. It denotes the purchasing power of A with respect to B. It is
always true that `-md{BA} <= d{AB} <= md{AB}`. Note the symmetric property:
`d{AB} = -d{BA}`.

From the point of view of B, the picture looks as follows:

```
            d{BA}
     [-------^-----|-----------]
  -md{AB}          0         md{BA}
```

In the context of credit switching, we call such a pair of identities A and B
**friends**. A pair of friends represents a trust relationship in the real
world between two people.

The economy of mutual credit between two people is simple, and it
requires that each side generates goods that the other side wants. In the
example above, the milkman wants to get bread, and the baker wants to get milk.
This does not always happen in the real world. 

If, for example, the baker decided to stop drinking milk (Maybe he got
allergic?), the simple mutual credit system between the milkman and the baker
will stop working. After a few loafs of bread that the milkman takes from
the baker for credit, the baker will not allow the milkman to take any more,
because the debt is too large.

This problem could be solved if the baker possibly wanted to buy service from
the carpenter, assuming that the carpenter buys milk from the milkman. This
can only happen in an economy of multiple players.


## Chains of Mutual Credit

To extend the idea of mutual credit to the economy of multiple players we use
chains of mutual credit. In this setting, every person has a few other
people he trusts, and maintains mutual credit with.

Assume that a person A wants to send funds to another person B. If A and B
are friends (They directly manage mutual credit), the transaction is
simple: A will decrease `d{AB}` and B will increase `d{BA}`. 

If A and B are not friends, they need the help of a mediator, or a chain of
mediators, to perform the transaction. A and B will look for a chain of the
following form:

```
A -- M1 -- M2 -- M3 -- B
```

Where A, M1 are friends, M1, M2 are friends, M2, M3 are friends and M3, B are
friends. Of course, the length of the chain could be arbitrary.

To transfer funds from A to B, A will first transfer funds to M1, M1 and will
transfer the funds to M2, M2 will transfer the funds to M3 and M3 will transfer
the funds to B. Each mediator (M1, M2, M3) can take a little amount of funds
for himself during the transaction, in exchange for helping A and B perform the
transaction.

We assume that friendship between people in the real world should allow any two
strangers A,B to find a chain of friends that connects them, to allow transfer
of funds between A and B. Note that if no such chain is found, transfer of
funds will not be possible.


## Credit Switching creates a zero sum economy

Consider the following friends network: 

```
   D
   |
C--A--B--F
   |
   E
```
The `A` has a few friends: `B,C,D,E`.
We calculate the amount of credit A has as the sum of all balances it has with
`B,C,D,E`. For example, if: 

- `A` owes `B` 5 credits.
- `A` owes `C` 4 credits.
- `D` owes `A` 10 credits.
- `E` owes `A` 2 credits. 
- `B` owes `F` 8 credits.

We now calculate the balances for all the nodes:

- A: `- 5 - 4 + 10 + 2 = + 3` credits.
- B: `- 8 + 5 = - 3` credits.
- C: `+ 4` credits
- D: `- 10` credits
- E: `- 2` credits
- F: `+ 8` credits

If we sum all the balances, we get: `3 - 3 + 4 - 10 - 2 + 8 = 0` credits.
This is not surprising, as the balance of every node is the sum of all the
balances he has with his direct friends, and for every pair of friends `X--Y`,
we know that `d{XY} = -d{YX}`.

This means that at the beginning, all the participants of the network begin
from 0 credits. Then as time passes, some participants might have negative
balances and others will have positive balances, however, the sum of all
balances will always be 0.

In some sense, credit in this system is created through debt, and is destroyed
when the debt is payed.


## Future topics

I managed to write this time only the basics of what Credit Switching is about.
There are some very important topics I didn't mention in this post at all, but
I hope to talk about in the near future:

- Synchronization: When we push credits along a chain of friends, how can we
    make sure that the transaction either happens or don't happen? Can credits
    be stuck in the middle?

- Routing: How to find a chain between friends to push credits through? How to
    do this in a decentralized manner?

- Networking: How friends in the network communicate? How to do this in a
    decentralized and secure way?

- Implementation details: about implementing Offst with Rust and the Futures
    asynchronous primitives.


## Current plan for the Offst

I believe that much of the core functionality has already been written. This
includes:

- Credit Switching logic
- Secure communication channels
- Relay service

The parts that are not done are:

- Indexing service (A service for finding chains of friends)
- External interface of a node
    - The ability to add plug-ins in a secure manner.


I hope to keep you updated on the interesting parts of developing this project.
If you are interested, you can sign up at the mailing list, or check this blog
from time to time.
