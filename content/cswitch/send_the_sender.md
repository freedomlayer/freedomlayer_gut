+++
title = "Send the Sender trick"
description = "About how Rust allows you to send the sending part of a channel through a channel"
date = 2018-10-29
+++

One very cool trick I learned about Rust is the ability to send the sender part
of a channel through another channel. It began for me as a strange experiment, but
later I have seen it happen inside Servo's code, so I got enough confident to
use this trick in my own code.

I will talk only about [channels in Rust
Future's](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.9/futures/channel/index.html),
but this trick may also apply to Rust std's channels. I never tried though.

If you try to reproduce these results in the future, note that this post works on:

```bash
$ rustc --version
rustc 1.31.0-nightly (96064eb61 2018-10-28)
```

with `futures-preview = "0.3.0-alpha.9"`

## Basic code for sending a Sender


Let's begin with some code that is not very meaningful by itself, but shows in
Rust code what it means to "send a Sender":

```rust
let (sender_a, receiver_a) = mpsc::channel(0);
let (sender_b, receiver_b) = mpsc::channel(0);

await!(sender_b.send(sender_a)).unwrap();
```

Why would you ever want to send the sender part of a channel through a channel?
This can turn out very useful if we want to create a request-response based
service. Instead of having to remember which request matches which response
(Possibly using some kind of a random token), we can instead:

1. Create a new oneshot channel. (A channel that sends a message only once).
2. Send the request to the service, together with the sender part of the
   oneshot channel. We keep the oneshot receiver on our side.
3. Wait on the oneshot receiver until we get a response.


I used this trick many times in the code of [CSwitch](about_cswitch.md). A
simple example is the Timer component. Check out [the code on
github](https://github.com/realcr/cswitch) if you want to learn more.


## Example Adder service

To make this use case more concrete, let's write an example service of our own:
A service that takes a number, adds one to the number and returns the result.

So a request to this service will be as follows:

```rust
struct Request {
    x: u32,
    response_sender: oneshot::Sender<u32>,
}
```

The request contains `x`, which is the number we want to increase, and a
`response_sender`. The service will use the `response_sender` to send back the
result of the computation: `x + 1`.

This is what the service looks like:

```rust
async fn adder_service(mut incoming: mpsc::Receiver<Request>) {
    while let Some(request) = await!(incoming.next()) {
        let Request {x, response_sender} = request;
        let _ = response_sender.send(x.wrapping_add(1));
    }
}
```

The `adder_service` is a future that keeps running in a loop. It takes
`incoming` as argument: This is a stream of incoming requests. `adder_service`
keeps running until the stream of `incoming_requests` ends (possibly because it
was closed).

We obtain an item from the incoming stream using the `next()` method.
Then for each request, the adder service takes the value `x`, adds 1 to it (We
do it using `wrapping_add` to avoid a panic in case `x` wraparounds). Then it
takes the result and sends it back using the provided sender.

Note that the call to `response_sender.send()` is not futuristic, so it does
not require `await!`. This happens because `response_sender` is a sender of
`oneshot` channel, not an `mpsc` channel.

Let's take a look at the client:

```rust
#[derive(Clone)]
struct AdderClient {
    request_sender: mpsc::Sender<Request>,
}

impl AdderClient {
    pub async fn request_add(&mut self, x: u32) -> Result<u32,()> {
        let (response_sender, response_receiver) = oneshot::channel();
        let request = Request {x, response_sender};
        await!(self.request_sender.send(request))
            .map_err(|_| ())?;
        await!(response_receiver)
            .map_err(|_| ())
    }
}

```
The client contains only a request sender. This allows the client to send
requests to the service. We implement the `request_add` method for `AdderClient`.
This is an asynchronous call. It sends a request to the adder service, and
waits for a response. Finally the response is returned to the caller.

Being more detailed, AdderClient does the following:
1. Create a new `oneshot` channel `(response_sender, response_receiver)`. This
   channel will be used by the service to send back the result.

2. A request is sent to the service, containing the value `x` to transform, and
   the sender part of the oneshot, `response_sender`.

3. The client waits on the receiving part of the oneshot, `response_receiver`.
   When it arrives, the result is returned from the function.


An important detail about AdderClient is that by the proposed design we managed
to make is cloneable. If we hold one `adder_client`, we can clone it to create
any amount of adder clients that we want. We can even send those clients to
different places (possibly through more channels).

finally, we present the user of our code one function to create a new service,
together with a bound client:

```rust
fn create_adder_service(mut spawner: impl Spawn) -> AdderClient {
    let (request_sender, request_receiver) = mpsc::channel(0);
    spawner.spawn(adder_service(request_receiver)).unwrap();

    AdderClient {
        request_sender,
    }
}
```

This function first creates a channel for transport of requests, to be sent from the
client to the adder service. It then spawns an adder service. Note that the
adder service is given the receiver part of the channel (`requests_receiver`). Then an AdderClient is
created. Note that it is given the sender part of the requests channel
(`requests_sender`).


Let's take a look at an example of using this code:

```rust
fn main() {
    let mut thread_pool = ThreadPool::new().unwrap();
    let mut adder_client = create_adder_service(thread_pool.clone());
    let res = thread_pool.run(adder_client.request_add(3)).unwrap();
    assert_eq!(res, 4);
}
```

We start a new adder service and obtain a client to this service. Then we send
one request (3), which results in the value (4) as expected.


## Fully working example

Putting it all together, we get:

```rust
#![feature(futures_api, pin, async_await, await_macro, arbitrary_self_types)]

use futures::channel::{mpsc, oneshot};
use futures::{StreamExt, SinkExt};
use futures::executor::ThreadPool;
use futures::task::{Spawn, SpawnExt};

struct Request {
    x: u32,
    response_sender: oneshot::Sender<u32>,
}

async fn adder_service(mut incoming: mpsc::Receiver<Request>) {
    while let Some(request) = await!(incoming.next()) {
        let Request {x, response_sender} = request;
        let _ = response_sender.send(x.wrapping_add(1));
    }
}

#[derive(Clone)]
struct AdderClient {
    request_sender: mpsc::Sender<Request>,
}

impl AdderClient {
    pub async fn request_add(&mut self, x: u32) -> Result<u32,()> {
        let (response_sender, response_receiver) = oneshot::channel();
        let request = Request {x, response_sender};
        await!(self.request_sender.send(request))
            .map_err(|_| ())?;
        await!(response_receiver)
            .map_err(|_| ())
    }
}

fn create_adder_service(mut spawner: impl Spawn) -> AdderClient {
    let (request_sender, request_receiver) = mpsc::channel(0);
    spawner.spawn(adder_service(request_receiver)).unwrap();

    AdderClient {
        request_sender,
    }
}


fn main() {
    let mut thread_pool = ThreadPool::new().unwrap();
    let mut adder_client = create_adder_service(thread_pool.clone());
    let res = thread_pool.run(adder_client.request_add(3)).unwrap();
    assert_eq!(res, 4);
}
```

My Cargo.toml contains:

```toml
[package]
name = "send_the_sender_trick"
version = "0.1.0"
authors = ["real <some@email.org>"]
edition = "2018"

[dependencies]

futures-preview = "0.3.0-alpha.9"
```

## Summary

Using the "send the Sender" trick saved me more than once when developing
futuristic services in Rust. Note that we didn't need to use any kind of
lock or mutex here. I think that it is very elegant and pretty easy to code. I
hope that I inspired you, when the time comes, to "send the Sender" on your
code too.

