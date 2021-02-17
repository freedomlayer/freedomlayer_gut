+++
title = "Design of Offset account book"
description = "Thoughts about the design of the Offset account book"
date = 2021-02-17
+++

This post is about to describe some thoughts I had regarding
the design of the Offset account book. Just in case this is your first time hearing
about Offset, [Offset](https://www.offsetcredit.org) is a decentralized
infrastructure allowing trading without money. Members can trade with each
other on the basis of credit accounting. To save some energy for both the
readers and me, Offset is not a blockchain, and it will not make you rich.

I haven't updated here for a while. Recently a friend asked me what I was
working on, and my reply was that I was having some problems with the design of
the Offset account book. "Well, what's so difficult about designing a phone
book?" was the immediate reaction. I tried to explain, but the best I could do
in the 2 minutes I had was to mumble something about the fact that Offset is
decentralized, transactions are asynchronous, things might get cancelled, etc. 
Obviously this was not enough to convince my friend, but this got me thinking,
maybe I need to write this all down, to get a better understanding myself of
what is so difficult with designing Offset's account book.

# Motivation

A good start might be to explain why I want to have an account book in the
first place. The first version of Offset has a very simple mobile app that
allows payments using the Offset network. One repeating feedback I received
from users of Offset v1 was that once they received a payment successfuly,
there was no record showing that a transaction ever happend. This occured
because indeed Offset v1 did not save any record for any past transaction! 

Offset v1 saves only the final balance against every friend. The mobile app
goes one step forward and keeps track of succesful (sent) payments by keeping
a cryptographic receipt, but as this information is only saved in the mobile
app, it can not be synchronized between multiple devices that connect to the
same Offset node.

From the feedback I realized that people want to know their history of
transactions. As I tried to dig deeper I realized some of the reasons behind
this feature request. Some people wanted to obtain better understanding of
their own earnings or spendings. Others mentioned that they would like to have
neat records to help them do their taxes, having a convincing and auditable
account book.

In the spirit of the rest of Offset's design, this feature should never
compromise user's privacy. A user transactions' history will only be saved
inside his own node, and can be deleted if the user wanted to. There should
also be a way to cancel this feature altogether, for users that do not want to
ever remember past transactions.


# Requirements

The only account book I see regularly is my own account book at my bank's
website. It looks somewhat like this:


```
Date       | Balance |  Change | Description 
===========+=========+=========+==================
10.02.2021 | +20000  |   -100  | Supermarket
12.02.2021 | +29000  |  +9000  | Salary
13.02.2021 | +28500  |   -500  | Electricy bill
15.02.2021 | +28400  |   -100  | Water bill
```

Every row in the table describes a transaction that took place at a certain
date. Balance denotes the balance right after the transaction took place. So in
the table above: `20000 + 9000 = 29000`, and `29000 - 500 = 28500`. The change
column is either positive (if I received money), or negative (if I sent money).
The description column is a very short textual description of the transaction.

I would like to have something as similar as possible to this account book for
Offset. However, due to the nature of Offset having an exact copy of this
account book design might not be easily acheived. Hence, I would like to
somehow extract the core characteristics of this simple account book, and at
least somehow satisfy those characteristics.

One of the most important characteristic of the above simple account book I
would like to imitate is what I call the *locality principle*. Wherever you are
at the account book, you can audit the balance by looking the the previous
transaction's balance, add the change from the new transaction, and be able to
calculate the balance of the next row. This applies even if you are given a
partial account book, for example, only beginning from a certain date.

The other problem I would like the account book to solve is to somehow allow
a third party (Like a tax collecting entity) to somehow audit the account
book, or even cross audit account books of different people that traded with
each other [^1].


# Intro to how Offset works

To better understand how to create a sound account book for Offset, some
understanding about how Offset works is required. I will try to outline the
main credit related parts, and elegantly abstract away all the technical
details of low level communication, encryption etc.

The basic component of the Offset protocol is a **pair of friends**. Offset
friends give certain amount of credit to each other. A relationship between two
Offset friends is called mutual credit. Offset allows credits to be pushed
along chains of friends, hence allowing trading without money.

TODO: Images of Bob paying to Dan through Charli.

At its core, the **Offset protocol is made of three messages: Request, Response
and Cancel**. Those messages are sent between Offset friends.

Assume that Bob's Offset node wants to send credits to Dan's Offset node,
through Charli. Bob's node will first send a Request message. A request message
contains a few fields. You can take a look here (From Offset v2):

```rust
pub struct RequestSendFundsOp {
    /// Id number of this request. Used to identify the whole transaction
    /// over this route.
    pub request_id: Uid,
    /// Currency used for this request
    pub currency: Currency,
    /// A hash lock created by the originator of this request
    pub src_hashed_lock: HashedLock,
    /// Amount paid to destination
    pub dest_payment: u128,
    /// hash(hash(actionId) || hash(totalDestPayment) || hash(description) || hash(additional))
    pub invoice_hash: HashResult,
    /// List of next nodes to transfer this request
    pub route: Vec<PublicKey>,
    /// Amount of fees left to give to mediators
    /// Every mediator takes the amount of fees he wants and subtracts this
    /// value accordingly.
    pub left_fees: u128,
}
```

The only field important to us at this point is the `dest_payment`, which
specifies the amount of credits Bob wants to push to Dan.

When the Request message arrives to Charli, Charli makes sure that she has
enough capacity to push this amount of credits along the mutual credit channel
connecting Bob and Charli, and also on the mutual credit channel connecting
Charli and Dan. If there is indeed enough capacity on both channels, Charli
will freeze `dest_payment` credits on both channels, and forward the Request
message to Bob. Otherwise, if there is not enough capacity at least on one of
the mutual credit channels, Charli will return Bob a Cancel message, and Bob
will conclude that the payment failed. 

A Cancel message is pretty boring:

```rust
pub struct CancelSendFundsOp {
    /// Id number of this request. Used to identify the whole transaction
    /// over this route.
    pub request_id: Uid,
}
```

It only contains the `request_id` of the originally sent Request message.

By freezing credits, we mean that Charli has saved a certain amount of credits
that can not be used by any other transaction. Those credits wait frozen, until
the transaction initiated by Bob is either done or cancelled.

We continue with the transation initiated by Bob. If Charli had enough
Capacity, her node will forward the Request message to Dan and freeze the
corresponding credits on the mutual credit channel between Charli and Dan.
Dan's node will also check that the mutual credit channel he has with Charli
has enough capacity to receive the credits. If so, Dan will issue a Response
message and send it back to Charli. 

When Charli receives the Response message, it first checks if the Response
message is valid. If the Response is indeed valid, Charli will unfreeze the credits
with Dan, effectively transferring the credits to Dan. Next, Charli sends the
Response message back to Bob. When Bob receives the Response message, it
verifies its validity, and then unfreezes the credits it froze earlier,
effectively transferring the credits to Charli.

This is what a Response message looks like:

```rust
pub struct ResponseSendFundsOp {
    /// Id number of this request. Used to identify the whole transaction
    /// over this route.
    pub request_id: Uid,
    pub src_plain_lock: PlainLock,
    /// Serial number used for this collection of invoice money.
    /// This should be a u128 counter, increased by 1 for every collected
    /// invoice.
    pub serial_num: u128,
    /// Signature{key=destinationKey}(
    ///   hash("FUNDS_RESPONSE") ||
    ///   hash(request_id || src_plain_lock || dest_payment) ||
    ///   hash(currency) ||
    ///   serialNum ||
    ///   invoiceHash)
    /// )
    pub signature: Signature,
}
```

A Request message contains a hash lock, created by Bob. Dan can only produce a
corresponding Response message after Bob has given him the corresponding plain lock.


[^1]: Offset will allow the user to produce an account book, but will never
  send this account book anywhere without explicit user's request.
