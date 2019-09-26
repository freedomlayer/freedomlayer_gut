+++
title = "Version 0.1.1 release"
description = "v0.1.1 Changes and future plans"
date = 2019-09-24
+++

Offst is a digital credit card, allowing secure and free payments.

We have just released version 0.1.1 of Offst. This is a short update about changes
from the last version (v0.1.0), and also about some future plans. 

Quick links:

- [Releases page](https://github.com/freedomlayer/offst/releases)
- [Repository](https://github.com/freedomlayer/offst)
- [First version announcement](@/offst/offst_release/index.md)


## Main changes

### Atomic payments

Proposal can be found [here](https://github.com/freedomlayer/offst/blob/c8bc6aa9c14acd76156752fb75d1ced50a6515cb/doc/docs/atomic_proposal.md).

In the previous version of Offst, a payment could take indefinite amount of
time. From the point of view of the buyer, sending funds is a risky operation
because funds might take indefinite amount of time to arrive, and there is no
way to stop the operation.

The new implemented feature protects the buyer from indefinite waiting for the
payment to be processed using a Hash Lock. In other words, funds are released
to the seller only after the buyer gives the seller a final commitment. Until
then, the funds can not be collected by the seller.


Diagram of the old protocol:

```
Invoice        <=====[inv]========    (Out of band)

Request        ------[req]------->
Response       <-----[resp]-------

Receipt        ======[receipt]===>    (Out of band)
Goods          <=====[goods]======    (Out of band)

               B --- C --- D --- E
```

Diagram of the new protocol:

```
Invoice        <=====[inv]========    (Out of band)

Request        ------[req]------->
Response       <-----[resp]-------

Commit         ======[commit]====>    (Out of band)
Goods          <=====[goods]======    (Out of band)

Collect        <-----[collect]----
(Receipt)
               B --- C --- D --- E
```

Note that the diagram of the new protocol contains two extra messages:
`Commit`: Sent by the buyer, and `Collect`: Sent by the seller.

Special thanks to [spolu](https://github.com/spolu) for suggesting this
solution.


### Added capnp ergonomics

Serialization and deserialization in Offst is done using Capnp, relying on the
[capnp-rust](https://github.com/capnproto/capnproto-rust) crate.

Using `capnp-rust` in its current form requires to write very similar code
multiple times, for every struct that we may want to serialize. This makes code
changes very difficult to do.

Example (taken from [here](https://github.com/capnproto/capnproto-rust/issues/136)):

1. Capnp structure:

```capnp
struct FriendsRoute {
        publicKeys @0: List(PublicKey);
        # A list of public keys
}
```

2. Rust structure:

```rust
pub struct FriendsRoute {
    pub public_keys: Vec<PublicKey>,
}
```

3. Middle layer code, converting between capnp Rust structs and Offst's Rust's
   structs:

```rust
pub fn ser_friends_route(
    friends_route: &FriendsRoute,
    friends_route_builder: &mut funder_capnp::friends_route::Builder,
) {
    let public_keys_len = usize_to_u32(friends_route.public_keys.len()).unwrap();
    let mut public_keys_builder = friends_route_builder
        .reborrow()
        .init_public_keys(public_keys_len);

    for (index, public_key) in friends_route.public_keys.iter().enumerate() {
        let mut public_key_builder = public_keys_builder
            .reborrow()
            .get(usize_to_u32(index).unwrap());
        write_public_key(public_key, &mut public_key_builder);
    }
}
```


The plan for a solution was a Rust procedural macro of this form:

```rust
#[capnp_conv(funder_capnp::friends_route)]
pub struct FriendsRoute {
    pub public_keys: Vec<PublicKey>,
}
```

This single declaration of `FriendsRoute` automatically generates the middle
layer code. Verification of matching of field names between the Rust code and
Capnp schemas is done during compile time.

This code for capnp-conv can be found
[here](https://github.com/freedomlayer/offst/tree/c8bc6aa9c14acd76156752fb75d1ced50a6515cb/components/capnp_conv).
When it gets stabilized we hope to release it as a separate crate, or possibly
as part of the `capnp-rust` crate.


Adding this feature allowed to remove many lines of redundant code from the
code base. In addition, we can now make protocol related changes quickly, with much more
confident, and with style.


### Upgrading "Futures"

Offst was recently updated to run on the `nightly-2019-09-13` toolchain.

The first change that required fixing was the new [await
postfix](https://boats.gitlab.io/blog/post/await-decision/).  We had to change
code that looked like `await!(func(arg))` into `func(arg).await` at the whole
codebase. Initially I began doing it manually, until I realized that there are
just too many occurences of `await` at the codebase to change manually.
Eventually I wrote a python script to perform the conversion automatically.

The second major change that required attention was a change to the behaviour
of `futures::channel::mpsc` at the Futures crate. 
It seems like `mpsc::channel(0)` of `0.3.0-alpha.16` is equivalent to
`mpsc::channel(1)` of `0.3.0-alpha.18`. 

It occurred to me that I created many `mpsc::channel(0)` in the codebase, and as
a result the behaviour of large parts of the code have changed: Some tests
resulted in a deadlock. It took me about a day to track down all the deadlocks
that arised from this modification.

Example (works in the new version `0.3.0-alpha.18` of the futures crate):

```rust
#[test]
fn test_channel_full() {
    let mut test_executor = TestExecutor::new();

    let arc_mutex_res = Arc::new(Mutex::new(0usize));
    let c_arc_mutex_res = Arc::clone(&arc_mutex_res);
    test_executor
        .spawn(async move {
            // Channel has limited capacity:
            let (mut sender, _receiver) = mpsc::channel::<u32>(8);

            // We keep sending into the channel.
            // At some point this loop should be stuck, because the channel is full.
            loop {
                sender.send(0).await.unwrap();
                let mut res_guard = c_arc_mutex_res.lock().unwrap();
                *res_guard = res_guard.checked_add(1).unwrap();
            }
        })
        .unwrap();

    test_executor.run_until_no_progress();
    let res_guard = arc_mutex_res.lock().unwrap();
    assert_eq!(*res_guard, 8);
}
```

In the older version `0.3.0-alpha.16` of the futures crate we should create a
channel of capacity 9 instead:
```rust
let (mut sender, _receiver) = mpsc::channel::<u32>(9);
```

This code is taken from the tests of `TestExecutor`, which can be found
[here](https://github.com/freedomlayer/offst/blob/c8bc6aa9c14acd76156752fb75d1ced50a6515cb/components/common/src/test_executor.rs)
This is a Futures Executor we wrote for the purpose of writing deterministic
futures tests in Offst. It also helps with time travelling and discovering
deadlocks. I hope to be able to explain more about it in a different post.


## Future plans

### Multiple currencies

A [question by bltavares](https://github.com/freedomlayer/offst/issues/228) led
to a long discussion about using Offst with different currencies.

Offst currently allows to manage mutual credit between pairs of entities using
a single currency. What `bltavares` actually asked is why can't we use Offst to
manage mutual credit in other currencies.

The current version of Offst should allow trading multiple currencies, but not
in a very convenient way. If a user want to use multiple currencies, he has to
open multiple wallets, each one containing a different currency. In addition, a
user using currenty of type B will have to be extra careful to not accidentally
join an Offst network using currency of type C, as this could result in loss of
money.

This state is not acceptable from safety point of view, and therefore if we
ever want to allow using multiple currencies with Offst we will have to change
something in the way it works.

The two ideas I had for this are:

1. Adding the name of the currency for the protocol messages. This will make
    sure that a user of currency B can never accidentally join an Offst network
    that trades with currency C. 

2. Implementing support for multi currencies inside the Offst protocol,
   allowing to keep multiple balances (in different currencies) between every two nodes.


Option (1) is simpler to implement. Option (2) requires a large modification to
the Offst codebase.

A diagram of solution (1):

```
 E  (PK1, USD) -- (PK2, USD)  F
    (PK3, EUR) -- (PK4, EUR)
```

At the diagram above, user `E` owns the wallets: `PK1, PK3`, user `F` owns the
wallets `PK2, PK4`.

A diagram of solution (2):

```
 E  (PK1, [USD,EUR]) -- (PK2, [USD,EUR])  F
```

At the diagram above, user `E` owns the wallets `PK1`, user `F` owns the
wallet `PK2`.

Solution (2) requires maintaining less wallet keys, at the expense of having a
bit more complex protocol (which can maintain multiple balances).

### Replace cryptographic library

Offst currently relies on the `ring` crate to perform cryptographic operations:

- Hash functions (Sha512)
- Symmetric encryption (Chacha20 + Poly1305)
- Diffie-Hellman (X25519)
- Digital signatures (Ed25519)

The ring crate has a very strict interface, to ensure proper and safe usage.
However, this strictness also makes it very difficult to write tests when using
cryptographic primitives from ring, mostly ones that are related to random
generation.

To be more specific: Ring does not allow the user of the library to have a
determinstic source of random for testing in a convenient way. Our current
solution is using an old an unmaintained Ring version that used to allow
mocking a random generator. Newer versions of the `ring` crate use a `Sealed`
trait, and therefore it is impossible to mock the random generator.

More information can be found at the [issue page](https://github.com/briansmith/ring/issues/661)

The usage of a very old version of ring is a technical debt we will have to
deal with at some point. The way I see it, our options would be:

- Convince the maintainers of the `ring` crate to add a mechanism for mocking a
    random generator.
- Replace the cryptographic library Offst depends on.
- Use a spectacular mocking hack for testing, like conditional compilation or
    deep usage of traits.


### Graphical user interface

We consider creating a graphical user interface for using Offst, possibly as:

- Browser extension
- Mobile application
- Desktop application

Hopefully we will get to it soon after dealing with the multi-currency feature
and some API stabilization.


## Contributors

Thanks to everyone who helped making this release happen:

- [amosonn](https://github.com/amosonn)
- [bltavares](https://github.com/bltavares)
- [GorbulasDev](https://github.com/GorbulasDev)
- [SonOfLilit](https://github.com/SonOfLilit)
- [spolu](https://github.com/spolu) 
- [pzmarzly](https://github.com/pzmarzly)
- [Atul9](https://github.com/Atul9)
- [cy6erlion](https://github.com/cy6erlion)

