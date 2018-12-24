+++
title = "Creating an empty iterator of a certain type in Rust"
description = "Creating an empty iterator of a certain type in Rust"
date = 2018-12-15
+++

I am working these days on the development of
[offst](https://github.com/freedomlayer/offst)'s Index server. I needed to
implement a basic directed graph structure, allowing to run the [BFS
algorithm](https://en.wikipedia.org/wiki/Breadth-first_search) to find routes
with a certain amount of capacity. 

The Index server has the job of collecting capacity reports from nodes in the
network. Nodes can then query the Index server for routes of certain
capacities.

Whenever a node A wants to send `k` credits to node B, the node A should first
send a request to the Index server, requesting a route from A to B of capacity
at least `k` credits. When A obtains a response from the Index server
containing a route, A can push credits along the provided route all the way to
B.

Example: 

A requests a route:

```
RouteRequest  (A -> IndexServer):  (A -> B, capacity >= 20)
RouteResponse (IndexServer -> A):  (A -- M1 -- M2 -- M3 -- B, capacity = 35)
```

Note that in the example `A` requested a route of send capacity at least `20`
credits, and he obtained as response a route with capacity `35`, which is more
than what A asked for.

Now A can use the obtained route `A -- M1 -- M2 -- M3 -- B` to push credits all
the way to B.


## Returning an empty iterator

During the work on the Index server I had the problem of wanting to return an
empty iterator in an early flow of a function. 

Let's begin with some context to the problem. The graph we am working with has
the following structure:

```rust
pub struct SimpleCapacityGraph<N> {
    nodes: HashMap<N, HashMap<N,(u128, u128)>>,
}
```

This is a map, mapping every node in the graph to a map of his neighbors.
The map of his neighbors maps each neighbor to a pair of capacities, send
capacity and receive capacity.

Example:

The graph:

```
 0 -- 1 
 |
 |
 2
```

Could be described with the following map:

```
0 -> {1 -> (10, 20), 2 -> (30, 15)}
1 -> {0 -> (20, 10)}
2 -> {0 -> (15, 35)}
```

In the example above, note that the node `0` reported a send capacity of `10`
credits to the node `1` and a receive capacity of `20` credits to the node `1`.
The report of node `1` about capacities to node `0` matches the report of node
`0`.

However, note that this will not always be the case. For example,  node `2`
reported receive capacity of `35` credits from node `0`, but node `0` reported
send capacity of `30` credits to node `2`. 

The details about the send and receive capacities are not very relevant to the
discussion here, but I felt obligated to explain the non matching numbers in
the example above.

Going back to the code, somewhere inside the implementation of
`SimpleCapacityGraph` there is the `neighbors_with_send_capacity(&self, a: N,
capacity: u128)`. This function returns all the neighbors of the node `a`
with capacity which is at least the given capacity argument:


```rust
impl<N> SimpleCapacityGraph<N> 
where
    N: cmp::Eq + hash::Hash + Clone + std::fmt::Debug,
{
    /* ... */

    fn neighbors_with_send_capacity(&self, a: N, capacity: u128) -> ??? {
        // ???
    }

    /* ... */
}
```

## Returning a Vec

The steps we should follow are roughly as follows:

1. Try to obtain the node `a`'s map, using `self.nodes.get(&a)`. If this node is not
   present, `a` has no matching neighbors.

2. Iterate over `a_map`, the map of `a`'s neighbors. For every neighbor make
   sure that it is of a large enough capacity. Return all the neighbors that
   have large enough capacity.


What should be the return value for `neighbors_with_send_capacity`? We could
choose something like `Vec`, obtaining this implementation:


```rust
fn neighbors_with_send_capacity_vec(&self, a: N, capacity: u128) -> Vec<&N> {
    let a_map = match self.nodes.get(&a) {
        Some(a_map) => a_map,
        None => return vec![],
    };

    a_map
        .keys()
        .filter(move |b| self.get_send_capacity(&a,b) >= capacity)
        .collect::<Vec<&N>>()
}
```

This is a working solution, however, note that we allocate a vector every
time we call this function, and where is the fun in using Rust if we don't have
**zero cost abstractions**? 

Another important detail is that every usage of
`neighbors_with_send_capacity_vec()` in our code uses it to iterate over the
resulting neighbors. It would have been nice if we could somehow return an
Iterator from the function.

## Returning Option of impl Iterator

Using the [impl trait](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md) 
feature we can do the following:

```rust
fn neighbors_with_send_capacity_option(&self, a: N, capacity: u128) -> Option<impl Iterator<Item=&N>> {
    let a_map = match self.nodes.get(&a) {
        Some(a_map) => a_map,
        None => return None,
    };
    let iter = a_map.keys().filter(move |b| self.get_send_capacity(&a,b) >= capacity);
    Some(iter)
}
```

However, this solution is not very ergonomic, as it requires the user of the
function to first check if the returned value is `Some(iterator)` and only then
iterate over the iterator. The user code will look like this:

```rust
if let Some(neighbors) = self.neighbors_with_send_capacity_option(a, capacity) {
    for neighbor in neighbors {
        // Do something here
    }
}
```

## std::iter::empty() attempt

After some searching in Rust's std documentation we can find [std::iter::empty()](https://doc.rust-lang.org/std/iter/fn.empty.html).
This is a function that creates an empty iterator. A naive attempt to use it
will look like this:

```rust
fn neighbors_with_send_capacity_iter_empty(&self, a: N, capacity: u128) -> impl Iterator<Item=&N> {
    let a_map = match self.nodes.get(&a) {
        Some(a_map) => a_map,
        None => return std::iter::empty(),
    };
    let iter = a_map.keys().filter(move |b| self.get_send_capacity(&a,b) >= capacity);
    iter
}
```

This function will not compile. The compiler throws the following error:

```rust
error[E0308]: mismatched types
  --> components/index_server/src/graph/simple_capacity_graph.rs:87:9
   |
87 |         iter
   |         ^^^^ expected struct `std::iter::Empty`, found struct `std::iter::Filter`
   |
   = note: expected type `std::iter::Empty<&N>`
              found type `std::iter::Filter<std::collections::hash_map::Keys<'_, N, (u128, u128)>, [closure@components/index_server/src/graph/simple_capacity_graph.rs:86:40: 86:89 self:_, a:_, capacity:_]>`
```

The compiler points out that `std::iter::Empty<&N>` and our other returned filter are
not the same. The compiler has to choose a single return value for the
function, and therefore the function could not be compiled.


## Boxing

To make this function compile we can try to box the the return values:

```rust
fn neighbors_with_send_capacity_iter_empty_box(&self, a: N, capacity: u128) -> Box<dyn Iterator<Item=&N>> {
    let a_map = match self.nodes.get(&a) {
        Some(a_map) => a_map,
        None => return Box::new(std::iter::empty()),
    };
    let iter = a_map.keys().filter(move |b| self.get_send_capacity(&a,b) >= capacity);
    Box::new(iter)
}
```

This function will not compile. The compilation error:

```rust
error: unsatisfied lifetime constraints
  --> components/index_server/src/graph/simple_capacity_graph.rs:98:9
   |
92 |     fn neighbors_with_send_capacity_iter_empty_box(&self, a: N, capacity: u128) -> Box<dyn Iterator<Item=&N>> {
   |                                                    - let's call the lifetime of this reference `'1`
...
98 |         Box::new(iter)
   |         ^^^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'static`

```

But with some anonymous lifetime annotations magic this function will compile:

```rust
fn neighbors_with_send_capacity_iter_empty_box(&self, a: N, capacity: u128) -> Box<dyn Iterator<Item=&N> + '_> {
    let a_map = match self.nodes.get(&a) {
        Some(a_map) => a_map,
        None => return Box::new(std::iter::empty()),
    };
    let iter = a_map.keys().filter(move |b| self.get_send_capacity(&a,b) >= capacity);
    Box::new(iter)
}
```

## OptionIterator

The Boxing solution still feels like some kind of a compromise.

To solve this problem, let's introduce a helper type called `OptionIterator`.
OptionIterator will be able to wrap an `Option<I>`, where I is some Iterator.
If the Option is None, the wrapper iterator will finish immediately. Otherwise,
if the Option is `Some(iterator)`, the wrapping iterator will do exactly what
the internal iterator is doing.


```rust
pub struct OptionIterator<I> {
    opt_iterator: Option<I>,
}

impl<I,T> Iterator for OptionIterator<I> 
where
    I: Iterator<Item=T>,
{
    type Item = T;
    fn next(&mut self) -> Option<T> {
        match &mut self.opt_iterator {
            Some(iterator) => iterator.next(),
            None => None,
        }
    }
}

impl<I> OptionIterator<I> {
    pub fn new(opt_iterator: Option<I>) -> OptionIterator<I> {
        OptionIterator {
            opt_iterator,
        }
    }
}
```

As can be seen, the only field of `OptionIterator` is `opt_iterator`, which
either contains an iterator or nothing. Next, let's look at the `next()` method
of the `Iterator` implementation for `OptionIterator`. If there exists an
internal iterator, the internal iterator's `next()` method is called.
Otherwise, `None` is returned, which means that the iterator has finished.

In other words, for some `iterator`, `OptionIterator::new(Some(iterator))` will
behave exactly like `iterator`. However, `OptionIterator::new(None)` will
behave like an empty iterator.

The advantage of using the `OptionIterator` wrapper is that both the existing
internal iterator and the empty internal iterator cases produce the same type:
OptionIterator. Recall our non compiling attempt of
`neighbors_with_send_capacity_iter_empty()` above.  We can now rewrite this
function as follows:

```rust
fn neighbors_with_send_capacity(&self, a: N, capacity: u128) -> OptionIterator<impl Iterator<Item=&N>> {
    let a_map = match self.nodes.get(&a) {
        Some(a_map) => a_map,
        None => return OptionIterator::new(None),
    };
    let iter = a_map.keys().filter(move |b| self.get_send_capacity(&a,b) >= capacity);
    OptionIterator::new(Some(iter))
}
```

Zero cost abstractions, as promised (:

This is the actual implementation currently chosen for the code in offst. If
you are interested, you can view the full code of `SimpleCapacityGraph`
implementation [on
github](https://github.com/freedomlayer/offst/blob/6f91692cc751c3c47e2ad54ce285306c07cde4a3/components/index_server/src/graph/simple_capacity_graph.rs).


## Further reading

- [Discussion on reddit](https://www.reddit.com/r/rust/comments/a6ii2e/creating_an_empty_iterator_of_a_certain_type/).
    Contains some very interesting advice for other possible solutions. For
    example, using `either::Either` or `res.into_iter().flat_map(|iter| iter)`.

- One idea we did not explore here is the possiblity of creating a generator and
somehow converting it to an iterator. See: [generator_to_iterator](https://users.rust-lang.org/t/a-problem-with-some-generators/12827).

- "How to create an empty iterator for a certain collection type?" on [Stack Overflow](https://stackoverflow.com/questions/32284362/how-to-create-an-empty-iterator-for-a-certain-collection-type-list-set-map-in)


