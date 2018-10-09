+++
title = "Approximating the size of a mesh network"
description = ""
date = 2017-07-28
+++

$$
\newcommand{\E}{\mathrm{E}}
\newcommand{\Var}{\mathrm{Var}}
\newcommand{\Cov}{\mathrm{Cov}}
$$

## Abstract

We present here a method for approximating the amount of nodes
in a mesh network. We also introduce an experiment to test our approximation
method. The size approximation is done by taking the minimum hashes of various
node ids and performing a transformation over those values.

The experiments are written in [Rust](https://www.rust-lang.org) and could be
found [here
[github]](https://github.com/realcr/freedomlayer_code/tree/master/approximate_net).


## Hashing node ids

Let $G$ be a connected network of nodes. Every node is connected to a few other nodes
that we call his neighbors. How can a node get an approximation of the amount
of nodes in the network?

We will discuss here a distributed algorithm that allows all the nodes of the
network to calculate an approximation.

Assume that every node has a unique identifier. For example, a public key or a
hash of a public key. Let $I$ be the set of all possible node ids. Assume that
we have a hash functions $h: I \rightarrow [0,1]$.  This function takes a node id as
input, and returns a value between 0 and 1 as output. 

How to practically obtain this kind of function? We could use some
cryptographic hash function, like
[SHA256](https://en.wikipedia.org/wiki/SHA-2), as follows:

$$h(x) = sha256("myhash" || x) / 2^{256}$$

Where || means concatenation. We divide by $2^{256}$ to normalize, so that we
get a value between $0$ and $1$. 

## Observing the minimum hash value

Consider the minimum value of the function $h$ over the set $V(G)$. (We
implicitly assume that $V(G) \subseteq I$). In other words, we apply $h$ over
all the node ids in the network, and pick the minimum value of $h$. We call
this value $min$.

$min$ has some relation to the amount of nodes in the network. If there are
more nodes in the network, we expect $min$ to be smaller. This is because it is
more likely that some node $v \in V(G)$ will get a small value for $h(v)$.

If there is only one node in the network, we expect $min$ to be about $1/2$, on
average. Extending this idea, if there are two nodes in the network, we expect
$min$ to be about $1/3$. Generally for $n$ nodes, we expect $min$ to have the
value $1/(n+1)$.

We can prove this idea more rigorously. Consider $n$ random node ids $v_1,
\dots v_{n}$. What is the probability that $min$ is going to be larger than
$t$? This requires all of the values $h(v_j)$ to be larger than $t$. The
probability of this event is $\Pr[min > t] = (1-t)^n$, because $\Pr[h(v_j) > t] =
1-t$, and the values of $h$ for different nodes are independent. Therefore
$F_{min}(t) = \Pr[min \leq t] = 1 - (1-t)^n$. This is also known as the cumulative
probability function of $min$.

$F(t)$ is differentiable, and so we can obtain the density function: $f(t) =
F'(t) = n(1-t)^{n-1}$. To obtain the mean of $min$ we can then calculate:
$$\E[min] = \int_{0}^{1} tf(t)dt = \int_{0}^{1} nt(1-t)^{n-1}dt$$

To solve this we first calculate:

$$\begin{split}
    \int_{0}^{1}t(1-t)^{n-1}dt = & \left[-\frac{t(1-t)^n}{n}\right]_{0}^{1} -
    \int_{0}^{1} 1 \cdot \frac{(1-t)^n}{-n}dt \\
    & = \int_{0}^{1}\frac{(1-t)^n}{n}dt = 
    \left[\frac{(1-t)^{n+1}}{-n(n+1)}\right]_{0}^{1} \\
    & = \frac{1}{n(n+1)}
\end{split}$$

Hence $\E[min] = n\cdot\frac{1}{n(n+1)} = \frac{1}{n+1}$. This result is similar
to our intuitive idea earlier.

For the sake of completeness, we are adding here the computation for
$Var[min]$. $Var[min] = \E[min^2] - \E[min]^2$. We already know $\E[min]$, we are
now left to calculate $\E[min^2]$.

$$\begin{split}
    \E[min^2] = & \int_{0}^{1} nt^2(1-t)^{n-1}dt =
    \left[\frac{nt^2(1-t)^n}{-n}\right]_{0}^{1} - 
    \int_{0}^{1} n\cdot 2\cdot t\cdot\frac{(1-t)^n}{-n}dt \\
    & = 2\int_{0}^{1}t(1-t)^n dt = 
    \left[\frac{2t(1-t)^{n+1}}{-(n+1)}\right]_{0}^{1} - 
    2\int_{0}^{1} \frac{(1-t)^{n+1}}{-(n+1)}dt \\
    & = 2\int_{0}^{1} (1-t)^{n+1} dt = 
    \left[2\cdot\frac{(1-t)^{n+2}}{-(n+1)(n+2)}\right]_{0}^{1} \\
    & = \frac{2}{(n+1)(n+2)}
\end{split}$$


Therefore $$\begin{split}
    \Var[min] = & \E[min]^2 - \E[min^2] = 
    \frac{2}{(n+1)(n+2)} - \left(\frac{1}{n+1}\right)^2 \\
    & = \frac{2(n+1)}{(n+1)^2(n+2)} - \frac{n+2}{(n+1)^2(n+2)} = 
    \frac{n}{(n+1)^2(n+2)}
\end{split}$$

We can also obtain that $\Var[min] \leq \frac{n+1}{(n+1)^2(n+2)} = 
\frac{1}{(n+1)(n+2)} \leq \frac{1}{(n+1)^2}$.

## Using multiple hash functions

Having the minimum of the hash function $h$ over all the node ids in the
network gives us some idea about the amount of nodes in the network. $min$ is
about $1 / (n+1)$, so we approximate $n$, the amount of nodes in the network,
to be $(1 / min) - 1$.

As we have seen above, the variance of $min$ is pretty large. This means that
there is a possibility for large errors in our approximation method. One idea
to deal with this problem is to add more hash functions. Instead of just one
hash function $h$, we will have $r$ hash functions, defined for example as:

$$h_i(x) = sha256("myhash" || x || i) / 2^{256}$$

Where $0 \leq i < r$. With $r$ hash functions we can obtain $r$ different min
values: $min_i$ for $0 \leq i < r$. Those $r$ minimum values hold more
information about the network size than just one minimum value, however we
still have to figure out how to combine all those minimum values into one
approximation of the network size.


## Experimenting with combining multiple minimums

We wrote a basic experiment (In Rust) to check the various methods for combining minimum
values for approximating the amount of nodes. It can be found [here
[github]](https://github.com/realcr/freedomlayer_code/tree/master/approximate_net).

We have tried various methods:

- Averaging all the minimum values and then calculating $(1 / avg) - 1$.
- Calculating $(1 / min_i) - 1$ for every $0 \leq i < r$ and then averaging the
    values.
- Calculating harmonic average over the minimum values and then calculating
  $(1 / avg) - 1$.
- Calculating $(1 / min_i) - 1$ for every $0 \leq i < r$ and then calculating 
    harmonic average over the values.

This is how the various combination schemes look like on code:

```rust
pub fn approx_size_harmonic_before(mins: &[u64]) -> usize {

    let fmeans = mins.iter()
        .map(|&m| m as f64)
        .collect::<Vec<f64>>();
    let hmean_min = harmonic_mean(&fmeans);
    (((u64::max_value() as f64)/ hmean_min) as usize) - 1
}

pub fn approx_size_harmonic_after(mins: &[u64]) -> usize {
    let trans = mins.iter()
        .map(|&m| (u64::max_value() / m) - 1)
        .map(|x| x as f64)
        .collect::<Vec<f64>>();

    harmonic_mean(&trans) as usize
}

pub fn approx_size_mean_before(mins: &[u64]) -> usize {
    let fmeans = mins.iter()
        .map(|&m| m as f64)
        .collect::<Vec<f64>>();
    let mean_min = mean(&fmeans);
    (((u64::max_value() as f64)/ mean_min) as usize) - 1

}

pub fn approx_size_mean_after(mins: &[u64]) -> usize {
    let trans = mins.iter()
        .map(|&m| (u64::max_value() / m) - 1)
        .map(|x| x as f64)
        .collect::<Vec<f64>>();

    mean(&trans) as usize
}
```

These are the results:

```bash
$ cargo run --release
   Compiling approximate_net v0.1.0 
    Finished release [optimized] target(s) in 1.44 secs
     Running `target/release/approximate_net`
Calculating error ratios for approximation functions...
num_iters = 100
num_mins  = 40
num_elems = 1000000

err_ratio for approximation functions:
approx_size_harmonic_before    : 10.208276536189299
approx_size_harmonic_after     : 0.14301947584857105
approx_size_mean_before        : 0.14301974775883225
approx_size_mean_after         : 10.208276278935292
```

Explaining the experiment:

We generate a set of size `num_elems`. Each element is of type `u64`: a 64 bit
number. We then use `num_mins` different hash functions to find the minimums
over all the elements. We then obtain `num_mins` different minimums for all the
different hash functions. Finally we use one of `4` methods to approximate the
amount of elements in the set.

We do this whole process `num_iters` times, to get a general idea of how much
deviation each of those methods have from the original amount of elements in
the set. To check if an approximation method works well, we calculate the error ratio.
It is calculated as the standard deviation divided by `num_elems`, the amount
of elements in the set. We approximate the standard deviation using this
calculation:

$$\sigma \approx \sqrt{\frac{1}{numIters}
    \sum_{0 \leq j < numIters}(approx_j - numElems)^2}$$

And we then calculate $errRatio = \frac{\sigma}{numElems}$.

Looking at the results, approx_size_harmonic_after and approx_size_mean_before
give very similar results, and they have the best results in this group. The
other two methods (approx_size_harmonic_before and approx_size_mean_after) have
very bad results.

A friend noted that `approx_size_mean_after` and `approx_size_harmonic_before`
should be identical (up to floating point math errors). This is because:
$$\begin{split}
    mean_i{ \left(\frac{1}{a_i} - 1 \right) } & = \frac{\sum_{i}{\left(\frac{1}{a_i} - 1\right)}}{n} \\
    & = \frac{\sum_{i}{\frac{1}{a_i}}}{n} - 1 = \frac{1}{\frac{n}{\sum_{i}{\frac{1}{a_i}}} } - 1 \\
    & = \frac{1}{harmonic_{i}{(a_i)}} - 1
\end{split}$$

For similar reasons, `approx_size_mean_before` and `approx_size_harmonic_after`
should be pretty close.

Our choice for combining the min values is going to be the
`approx_size_mean_before`, as it is the simplest to reason about.


## Distributed algorithm for calculating the hashes minimums

One of the reasons we chose the minimum hashes method for approximating the
amount of nodes in the network is that it is easy to calculate in a mesh
network.

Every node in the network maintains an array of the nodes with lowest hashes
that he ever seen, as follows: $minHash[i] = (nodeId, timestamp, sign[nodeId](timestamp))$
for $0 \leq i < r$.

$minHash[i]$ corresponds to the node with lowest hash value for the function
$h_i$. $sign[nodeId](timestamp)$ is a digital signature by the node $nodeId$
over some timestamp. This proves that $nodeId$ was recently alive.

Given that every node knows the set of minimums for all the $r$ different hash
functions, every node in the network can calculate an approximation for the
amount of nodes in the network.

We now describe the distributed algorithm for a given node $x$:

**Initialization**

When a node $x$ enters the network, he first fills:
$minHash_x[i] = (x, sign[x](timestamp))$ for $0 \leq i < r$. In other words, $x$
fills himself as the node with lowest hash for all the hash functions.

**Periodic cleaning**

For every node $x$, every constant period of time the following happens: 

1. $x$ checks his list of $minHash$ nodes. If any timestamp is too old, the
entry $minHash[i] = (nodeId, timestamp, sign[nodeId](timestamp))$ is removed
from the list and $x$ puts himself instead: $minHash_x[i] = (x ,
currentTimestamp, sign[x](currentTimestamp))$. 

2. $x$ sends all his direct network neighbors a message the contains the
updated $minHash$ entry: $UpdateMinNodeId(minHash_x[i])$


**Receipt of UpdateMinNodeId message**

Assume that a node $x$ receives an UpdateMinNodeId message with $minHash[i] = (nodeId,
timestamp, sign[nodeId](timestamp))$. Also assume that $x$ currently have
a current entry: $minHash_x[i]$ as the node with lowest value for $h_i$.

1. If the timestamp is too old, abort.
2. If the signature by $nodeId$ over the timestamp is not valid, abort.
3. If $h_i(nodeId) > h_i(minHash_x[i].nodeId)$ then abort. ($x$'s node with
    lowest $h_i$ value has lower value than the candidate).
4. If $h_i(nodeId) = h_i(minHash_x[i].nodeId)$ and the timestamp of the
    candidate is older then abort.
5. Update $minHash_x[i] = minHash[i]$
6. $x$ sends all his direct network neighbors a message that contains the
updated $minHash$ entry: $UpdateMinNodeId(minHash_x[i])$


## Further research

### Accuracy

It is known that there are more accurate methods to approximate the unique
amount of elements in a set. However, not all of those methods are adequate to
work as a distributed algorithm.

We know that the arithmetic mean gives good enough results for approximating
the network size for our purposes, however it is possible that better methods
exist to approximate the network size only given various minimum values of
random hash functions over the node ids.

### Security

If a node can choose his own node id, he may generate many node ids until his
node id gives a low value for one of the hashes. This can allow one participant
in the network to greatly affect the approximation for the amount of nodes in
the network. Currently the only way we know of to avoid this problem is to not
let nodes pick their own node ids, and to invalidate node ids from time to
time. See the article "How to spread Adversarial Nodes? Rotate!" by Christian
Scheideler for more about this idea.


## Further reading

- [Estimating counts of distinct values with kmv](https://blog.demofox.org/2015/02/03/estimating-counts-of-distinct-values-with-kmv/)
- "HyperLogLog: the analysis of a near-optimal cardinality estimation
    algorithm" by Philippe Flajolet, Éric Fusy, Olivier Gandouet, Frédéric 
    Meunier
- [HyperLogLog and MinHash](http://tech.adroll.com/blog/data/2013/07/10/hll-minhash.html)
- [Jaccard Index (wikipedia)](https://en.wikipedia.org/wiki/Jaccard_index)
- [MinHash (wikipedia)](https://en.wikipedia.org/wiki/MinHash)
- "How to spread Adversarial Nodes? Rotate!" by Christian Scheideler.

