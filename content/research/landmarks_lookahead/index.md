+++
title = "Landmarks routing with lookahead experiment results"
description = ""
date = 2017-06-21
+++

## Abstract

We present here some experiment results for a variation of the landmarks
routing method. Those results show that landmarks routing could be used in
practice for efficient routing in a network.
The experiments are written in [Rust](https://www.rust-lang.org) and could be
found on github, [here [github]](https://github.com/Freedomlayer/freedomlayer_code/tree/master/landmarks_lookahead).

## A Short reminder about network coordinates

Given a network $G$ of $n$ nodes, we pick $k$ nodes randomly $\{l_1, \dots ,
l_k\}$ and call them the landmarks of the network. The amount of landmarks we
pick is about $\log(n)^2$, where $n = |V(G)|$, the amount of nodes in the
network.

Every node $x$ maintains shortest paths to all the landmarks. Specifically,
every node should know his distances to each of the landmarks. If we order the
landmarks somehow, this results in every node having a coordinate, specifying
his location in the network. For some node $x$, this coordinate consists of
the distances of $x$ to each of the landmarks $l_i$.

$$Coord(x) := \{d(x,l_1), \dots , d(x, l_k)\}$$

Where $d(x,y)$ is the length of the shortest path between the nodes $x$ and
$y$.

Recall that landmarks based routing (Using random walking) was discussed
earlier in: [Landmarks navigation by random walking](
./research/landmarks_navigation_rw/index.md).

We use the network coordinates to approximate distances between nodes in the
network. This should help us with routing messages between nodes.

The distance between two nodes $x$ and $y$ is approximated as follows: We take
the $x$'s and $y$'s coordinates, subtract the entries pairwise, take absolute
values and find the maximum: 

$$mdist(Coord(x), Coord(y)) := \max_{1 \leq i \leq k} {|Coord(x)_i - Coord(y)_i|}$$

$mdist$ satisfies $mdist(Coord(x), Coord(y)) \leq d(x,y)$. This follows from
the triangle inequality over the metric $d$.
$mdist$ itself is a semi metric. It is symmetric and always satisfies the triangle
inequality. However, it is possible that $mdist(C) = mdist(D)$ for two
different coordinates $C \neq D$.

## Using lookahead

In [Landmarks navigation by random walking](
./research/landmarks_navigation_rw/index.md)
we ran an experiment of routing with network coordinates, relying on a random
walk. It didn't gave us very good results.

In this experiment we will use a different strategy. Assume that we want to
route from a node $x$ to a node $y$. We begin from node $x$, and in every
iteration we pick a neighbor that is closest to $y$ according to $mdist$ over
the network coordinates.

Using this method we arrive quickly at some node $z$ with low $mdist(z,y)$, and
every neighbor $w$ of $z$ has a higher value of $mdist(w,y)$. This means that
we are stuck. 

Whenever we are stuck at some node $z$, we pick a random neighbor of $z$ and continue from
there.

This algorithm can also get stuck. We can solve this by letting every node know
more about the network. For every node $z$, instead of picking the best
neighbor every time, we will pick from all nodes of some distance from $z$.

It appears that this method allows efficient routing in various types of
networks. We knows this from experiments. Unfortunately, we have no formal ways
of proving this.


## Comparing landmarks routing and chord virtual DHT routing

The following experiment is used to check the performance of landmarks routing
with lookahead, and compare its performance with chord [Virtual DHT
routing](./research/exp_virtual_dht_routing/index.md).
Also see [A globally connected overlay for Virtual Ring Routing [pdf]](/articles/chord_connected_routing.pdf).

How to read this table? 

$g$ is the log in base 2 of the size of the network. $g=6$ means a network of
size $64$. The network column shows which type of network the routing is
performed on. There are three types of networks we check: 

- rand: A random network where every node is connected to about $1.5 * log(n)$ other nodes.
- 2d: A two dimensional grid.
- rand+2d: A sum of a two dimensional grid and a random network.

Note that for every type of network we generate three networks, to have more
numbers to look at. $ni$ means network iteration.  

There are three routing methods we consider: 

- chord: Virtual DHT routing. We programmed the idea presented at [A globally
  connected overlay for Virtual Ring Routing [pdf]](/articles/chord_connected_routing.pdf).

- landmarks nei^2: Landmarks routing where every node can see approximately all
  nodes in radius 2 from himself. (His direct neighbors, and the neighbors of his neighbors).

- landmarks nei^3: Landmarks routing where every node can see approximately all
  nodes in radius 3 from himself.

For every routing experiment, we keep the average path length (left), the
maximum path length (middle) and the ratio of routing success (right).

If the maximum path length becomes too large, we stop the specific experiment
and put stars (`***`) in all subsequent routing experiments of the same type.

The experiments are written in [Rust](https://www.rust-lang.org). The results are presented here.
You can get the same results yourself by running:
```
git clone https://github.com/Freedomlayer/freedomlayer_code
cd landmarks_lookahead/net_coords
cargo run --bin full_matrix --release
```

```
      Network        |          chord         |    landmarks nei^2     |     landmarks nei^3     
---------------------+------------------------+------------------------+------------------------+
g= 6; rand    ; ni=0 |     2.87,      9, 1.00 |     1.99,      3, 1.00 |     1.99,      3, 1.00 |
g= 6; rand    ; ni=1 |     2.85,      9, 1.00 |     1.91,      3, 1.00 |     1.91,      3, 1.00 |
g= 6; rand    ; ni=2 |     2.82,      9, 1.00 |     2.02,      3, 1.00 |     2.02,      3, 1.00 |
g= 6; 2d      ; ni=0 |     9.19,     39, 1.00 |     5.51,     13, 1.00 |     5.51,     13, 1.00 |
g= 6; 2d      ; ni=1 |     9.09,     30, 1.00 |     5.17,     10, 1.00 |     5.17,     10, 1.00 |
g= 6; 2d      ; ni=2 |     8.93,     28, 1.00 |     5.34,     13, 1.00 |     5.34,     13, 1.00 |
g= 6; rand+2d ; ni=0 |     2.58,      8, 1.00 |     1.83,      3, 1.00 |     1.83,      3, 1.00 |
g= 6; rand+2d ; ni=1 |     2.53,      8, 1.00 |     1.89,      3, 1.00 |     1.89,      3, 1.00 |
g= 6; rand+2d ; ni=2 |     2.45,      7, 1.00 |     1.82,      3, 1.00 |     1.82,      3, 1.00 |

g= 7; rand    ; ni=0 |     3.66,     10, 1.00 |     2.15,      3, 1.00 |     2.15,      3, 1.00 |
g= 7; rand    ; ni=1 |     3.60,     11, 1.00 |     2.07,      3, 1.00 |     2.07,      3, 1.00 |
g= 7; rand    ; ni=2 |     3.67,     10, 1.00 |     2.13,      3, 1.00 |     2.13,      3, 1.00 |
g= 7; 2d      ; ni=0 |    15.57,     55, 1.00 |     6.91,     15, 1.00 |     6.91,     15, 1.00 |
g= 7; 2d      ; ni=1 |    15.46,     54, 1.00 |     6.70,     16, 1.00 |     6.70,     16, 1.00 |
g= 7; 2d      ; ni=2 |    15.49,     49, 1.00 |     7.62,     17, 1.00 |     7.62,     17, 1.00 |
g= 7; rand+2d ; ni=0 |     3.18,      9, 1.00 |     2.04,      3, 1.00 |     2.04,      3, 1.00 |
g= 7; rand+2d ; ni=1 |     3.00,     11, 1.00 |     1.91,      3, 1.00 |     1.91,      3, 1.00 |
g= 7; rand+2d ; ni=2 |     3.17,      9, 1.00 |     1.92,      3, 1.00 |     1.92,      3, 1.00 |

g= 8; rand    ; ni=0 |     4.59,     13, 1.00 |     2.40,      3, 1.00 |     2.40,      3, 1.00 |
g= 8; rand    ; ni=1 |     4.74,     12, 1.00 |     2.33,      3, 1.00 |     2.33,      3, 1.00 |
g= 8; rand    ; ni=2 |     4.71,     13, 1.00 |     2.30,      3, 1.00 |     2.30,      3, 1.00 |
g= 8; 2d      ; ni=0 |    32.81,    105, 1.00 |     9.75,     23, 1.00 |     9.75,     23, 1.00 |
g= 8; 2d      ; ni=1 |    32.13,     95, 1.00 |    11.60,     27, 1.00 |    11.60,     27, 1.00 |
g= 8; 2d      ; ni=2 |    30.22,     96, 1.00 |    10.11,     23, 1.00 |    10.11,     23, 1.00 |
g= 8; rand+2d ; ni=0 |     4.19,     12, 1.00 |     2.13,      3, 1.00 |     2.13,      3, 1.00 |
g= 8; rand+2d ; ni=1 |     4.18,     12, 1.00 |     2.08,      3, 1.00 |     2.08,      3, 1.00 |
g= 8; rand+2d ; ni=2 |     4.25,     15, 1.00 |     2.17,      3, 1.00 |     2.17,      3, 1.00 |

g= 9; rand    ; ni=0 |     6.05,     18, 1.00 |     3.19,     11, 1.00 |     2.54,      3, 1.00 |
g= 9; rand    ; ni=1 |     5.86,     14, 1.00 |     3.12,      9, 1.00 |     2.55,      3, 1.00 |
g= 9; rand    ; ni=2 |     6.07,     16, 1.00 |     3.26,     13, 1.00 |     2.57,      4, 1.00 |
g= 9; 2d      ; ni=0 |    61.83,    206, 1.00 |    14.72,     37, 1.00 |    14.72,     37, 1.00 |
g= 9; 2d      ; ni=1 |    53.72,    165, 1.00 |    15.30,     32, 1.00 |    15.30,     32, 1.00 |
g= 9; 2d      ; ni=2 |    60.82,    354, 1.00 |    13.41,     40, 1.00 |    13.41,     40, 1.00 |
g= 9; rand+2d ; ni=0 |     5.29,     14, 1.00 |     2.33,      3, 1.00 |     2.33,      3, 1.00 |
g= 9; rand+2d ; ni=1 |     5.27,     15, 1.00 |     2.25,      3, 1.00 |     2.25,      3, 1.00 |
g= 9; rand+2d ; ni=2 |     5.31,     14, 1.00 |     2.40,      3, 1.00 |     2.40,      3, 1.00 |

g=10; rand    ; ni=0 |     7.48,     18, 1.00 |     4.55,     26, 1.00 |     2.62,      4, 1.00 |
g=10; rand    ; ni=1 |     7.54,     21, 1.00 |     4.58,     27, 1.00 |     2.65,      3, 1.00 |
g=10; rand    ; ni=2 |     7.48,     17, 1.00 |     4.20,     24, 1.00 |     2.69,      3, 1.00 |
g=10; 2d      ; ni=0 |   130.53,    462, 1.00 |    20.86,     55, 1.00 |    20.86,     55, 1.00 |
g=10; 2d      ; ni=1 |   129.02,    811, 1.00 |    19.58,     49, 1.00 |    19.58,     49, 1.00 |
g=10; 2d      ; ni=2 |   156.51,   1003, 1.00 |    22.75,     52, 1.00 |    22.75,     52, 1.00 |
g=10; rand+2d ; ni=0 |     6.78,     16, 1.00 |     4.00,     15, 1.00 |     2.61,      3, 1.00 |
g=10; rand+2d ; ni=1 |     6.76,     17, 1.00 |     3.83,     19, 1.00 |     2.58,      3, 1.00 |
g=10; rand+2d ; ni=2 |     6.78,     18, 1.00 |     3.63,     15, 1.00 |     2.52,      3, 1.00 |

g=11; rand    ; ni=0 |     9.57,     22, 1.00 |     7.52,     43, 1.00 |     2.87,      4, 1.00 |
g=11; rand    ; ni=1 |     9.48,     21, 1.00 |     5.56,     22, 1.00 |     2.75,      4, 1.00 |
g=11; rand    ; ni=2 |     9.43,     24, 1.00 |     6.00,     35, 1.00 |     2.76,      3, 1.00 |
g=11; 2d      ; ni=0 |   294.56,   1116, 1.00 |    31.23,     71, 1.00 |    31.23,     71, 1.00 |
g=11; 2d      ; ni=1 |   358.45,   1237, 1.00 |    28.14,     64, 1.00 |    28.14,     64, 1.00 |
g=11; 2d      ; ni=2 |   355.77,   1242, 1.00 |    29.17,     75, 1.00 |    29.17,     75, 1.00 |
g=11; rand+2d ; ni=0 |     8.75,     18, 1.00 |     5.37,     25, 1.00 |     2.70,      3, 1.00 |
g=11; rand+2d ; ni=1 |     8.80,     20, 1.00 |     4.89,     23, 1.00 |     2.57,      3, 1.00 |
g=11; rand+2d ; ni=2 |     8.90,     24, 1.00 |     5.91,     29, 1.00 |     2.74,      3, 1.00 |

g=12; rand    ; ni=0 |    12.55,     31, 1.00 |    11.09,     71, 1.00 |     2.93,      4, 1.00 |
g=12; rand    ; ni=1 |    12.25,     42, 1.00 |    10.40,    141, 1.00 |     2.84,      4, 1.00 |
g=12; rand    ; ni=2 |    12.39,     43, 1.00 |     8.59,     45, 1.00 |     2.94,      4, 1.00 |
g=12; 2d      ; ni=0 |   817.58,   3433, 1.00 |    39.81,     91, 1.00 |    39.81,     91, 1.00 |
g=12; 2d      ; ni=1 |   866.15,   3367, 1.00 |    40.21,     95, 1.00 |    40.21,     95, 1.00 |
g=12; 2d      ; ni=2 |   844.80,   3937, 1.00 |    44.39,    115, 1.00 |    44.39,    115, 1.00 |
g=12; rand+2d ; ni=0 |    11.11,     26, 1.00 |     8.67,     94, 1.00 |     2.81,      4, 1.00 |
g=12; rand+2d ; ni=1 |    11.11,     26, 1.00 |    10.28,    108, 1.00 |     2.84,      4, 1.00 |
g=12; rand+2d ; ni=2 |    11.21,     24, 1.00 |     8.92,     42, 1.00 |     2.79,      4, 1.00 |

g=13; rand    ; ni=0 |    16.69,     43, 1.00 |    15.19,    119, 1.00 |     3.05,      4, 1.00 |
g=13; rand    ; ni=1 |    16.59,     38, 1.00 |    15.89,    145, 1.00 |     2.99,      4, 1.00 |
g=13; rand    ; ni=2 |    16.46,    106, 1.00 |    16.47,     92, 1.00 |     3.10,      4, 1.00 |
g=13; 2d      ; ni=0 |  3580.53,  34424, 1.00 |    63.29,    145, 1.00 |    63.29,    145, 1.00 |
g=13; 2d      ; ni=1 |************************|    60.27,    137, 1.00 |    60.27,    137, 1.00 |
g=13; 2d      ; ni=2 |************************|    54.35,    126, 1.00 |    54.35,    126, 1.00 |
g=13; rand+2d ; ni=0 |    14.64,     35, 1.00 |    12.60,    123, 1.00 |     2.96,      4, 1.00 |
g=13; rand+2d ; ni=1 |    14.78,     32, 1.00 |    14.51,     81, 1.00 |     2.91,      3, 1.00 |
g=13; rand+2d ; ni=2 |    14.69,     33, 1.00 |    12.94,     59, 1.00 |     2.91,      4, 1.00 |

g=14; rand    ; ni=0 |    22.19,    187, 1.00 |    16.68,    171, 1.00 |     3.19,      4, 1.00 |
g=14; rand    ; ni=1 |    23.04,    215, 1.00 |    18.86,    233, 1.00 |     3.17,      4, 1.00 |
g=14; rand    ; ni=2 |    23.23,    497, 1.00 |    12.71,    162, 1.00 |     3.18,      4, 1.00 |
g=14; 2d      ; ni=0 |************************|    99.64,    173, 1.00 |    99.64,    173, 1.00 |
g=14; 2d      ; ni=1 |************************|    88.99,    213, 1.00 |    88.99,    213, 1.00 |
g=14; 2d      ; ni=2 |************************|    84.10,    175, 1.00 |    84.10,    175, 1.00 |
g=14; rand+2d ; ni=0 |    20.40,    176, 1.00 |    23.61,    121, 1.00 |     3.08,      4, 1.00 |
g=14; rand+2d ; ni=1 |    20.07,    307, 1.00 |    19.42,    170, 1.00 |     3.09,      4, 1.00 |
g=14; rand+2d ; ni=2 |    20.64,    530, 1.00 |    18.85,    127, 1.00 |     3.03,      4, 1.00 |

g=15; rand    ; ni=0 |    35.67,   1292, 1.00 |    22.96,    157, 1.00 |     4.14,     16, 1.00 |
g=15; rand    ; ni=1 |    34.16,    949, 1.00 |    23.83,    239, 1.00 |     4.12,     17, 1.00 |
g=15; rand    ; ni=2 |    35.81,   1940, 1.00 |    33.00,    381, 1.00 |     3.98,     16, 1.00 |
g=15; 2d      ; ni=0 |************************|   122.06,    263, 1.00 |   122.06,    263, 1.00 |
g=15; 2d      ; ni=1 |************************|   111.60,    278, 1.00 |   111.60,    278, 1.00 |
g=15; 2d      ; ni=2 |************************|   114.55,    292, 1.00 |   114.55,    292, 1.00 |
g=15; rand+2d ; ni=0 |    41.51,   7527, 1.00 |    32.80,    270, 1.00 |     3.30,      4, 1.00 |
g=15; rand+2d ; ni=1 |    45.24,   7244, 1.00 |    31.98,    210, 1.00 |     3.26,      4, 1.00 |
g=15; rand+2d ; ni=2 |    38.32,   6505, 1.00 |    24.21,    190, 1.00 |     3.32,      4, 1.00 |

g=16; rand    ; ni=0 |    57.93,   1565, 1.00 |    62.13,    619, 1.00 |     5.44,     21, 1.00 |
g=16; rand    ; ni=1 |    69.82,   2856, 1.00 |    67.00,    507, 1.00 |     5.72,     40, 1.00 |
g=16; rand    ; ni=2 |    58.75,   1675, 1.00 |    50.52,    255, 1.00 |     5.73,     29, 1.00 |
g=16; 2d      ; ni=0 |************************|   179.36,    407, 1.00 |   179.36,    407, 1.00 |
g=16; 2d      ; ni=1 |************************|   146.36,    410, 1.00 |   146.36,    410, 1.00 |
g=16; 2d      ; ni=2 |************************|   159.79,    357, 1.00 |   159.79,    357, 1.00 |
g=16; rand+2d ; ni=0 |   215.08,  42298, 1.00 |    48.92,    294, 1.00 |     4.61,     26, 1.00 |
g=16; rand+2d ; ni=1 |************************|    46.13,    357, 1.00 |     4.79,     21, 1.00 |
g=16; rand+2d ; ni=2 |************************|    52.57,    421, 1.00 |     4.58,     15, 1.00 |
```

## Running Landmarks routing on large networks

We could not run the previous experiment for very large networks, because Chord
virtual DHT routing is hard to simulate for large networks. Therefore we
created another experiment that only checks landmarks routing for nei^2 and
nei^3.

In addition, we use here a technique of tie breaking with random distances:
Instead of having a plain distance $1$ between every two neighbors, we pick a
random number between 1000 and 2000. This distorts the geometry of the graph a
little bit, but it surprisingly gives better results in our routing
experiments. This was not done at the previous experiment.


You can run the following experiment by running:
```
git clone https://github.com/Freedomlayer/freedomlayer_code
cd landmarks_lookahead/net_coords
cargo run --bin landmarks_weighted_matrix --release
```


```
      Network        |    landmarks nei^2     |     landmarks nei^3     
---------------------+------------------------+------------------------+
g= 6; rand    ; ni=0 |     1.99,      3, 1.00 |     1.99,      3, 1.00 |
g= 6; rand    ; ni=1 |     1.91,      3, 1.00 |     1.91,      3, 1.00 |
g= 6; rand    ; ni=2 |     2.02,      3, 1.00 |     2.02,      3, 1.00 |
g= 6; 2d      ; ni=0 |     5.51,     13, 1.00 |     5.51,     13, 1.00 |
g= 6; 2d      ; ni=1 |     5.17,     10, 1.00 |     5.17,     10, 1.00 |
g= 6; 2d      ; ni=2 |     5.34,     13, 1.00 |     5.34,     13, 1.00 |
g= 6; rand+2d ; ni=0 |     1.83,      3, 1.00 |     1.83,      3, 1.00 |
g= 6; rand+2d ; ni=1 |     1.89,      3, 1.00 |     1.89,      3, 1.00 |
g= 6; rand+2d ; ni=2 |     1.82,      3, 1.00 |     1.82,      3, 1.00 |

g= 7; rand    ; ni=0 |     2.15,      3, 1.00 |     2.15,      3, 1.00 |
g= 7; rand    ; ni=1 |     2.07,      3, 1.00 |     2.07,      3, 1.00 |
g= 7; rand    ; ni=2 |     2.13,      3, 1.00 |     2.13,      3, 1.00 |
g= 7; 2d      ; ni=0 |     6.91,     15, 1.00 |     6.91,     15, 1.00 |
g= 7; 2d      ; ni=1 |     6.70,     16, 1.00 |     6.70,     16, 1.00 |
g= 7; 2d      ; ni=2 |     7.62,     17, 1.00 |     7.62,     17, 1.00 |
g= 7; rand+2d ; ni=0 |     2.04,      3, 1.00 |     2.04,      3, 1.00 |
g= 7; rand+2d ; ni=1 |     1.91,      3, 1.00 |     1.91,      3, 1.00 |
g= 7; rand+2d ; ni=2 |     1.92,      3, 1.00 |     1.92,      3, 1.00 |

g= 8; rand    ; ni=0 |     2.40,      3, 1.00 |     2.40,      3, 1.00 |
g= 8; rand    ; ni=1 |     2.33,      3, 1.00 |     2.33,      3, 1.00 |
g= 8; rand    ; ni=2 |     2.30,      3, 1.00 |     2.30,      3, 1.00 |
g= 8; 2d      ; ni=0 |     9.75,     23, 1.00 |     9.75,     23, 1.00 |
g= 8; 2d      ; ni=1 |    11.60,     27, 1.00 |    11.60,     27, 1.00 |
g= 8; 2d      ; ni=2 |    10.11,     23, 1.00 |    10.11,     23, 1.00 |
g= 8; rand+2d ; ni=0 |     2.13,      3, 1.00 |     2.13,      3, 1.00 |
g= 8; rand+2d ; ni=1 |     2.08,      3, 1.00 |     2.08,      3, 1.00 |
g= 8; rand+2d ; ni=2 |     2.17,      3, 1.00 |     2.17,      3, 1.00 |

g= 9; rand    ; ni=0 |     2.68,      4, 1.00 |     2.54,      3, 1.00 |
g= 9; rand    ; ni=1 |     2.65,      4, 1.00 |     2.55,      3, 1.00 |
g= 9; rand    ; ni=2 |     2.72,      4, 1.00 |     2.57,      4, 1.00 |
g= 9; 2d      ; ni=0 |    14.74,     37, 1.00 |    14.72,     37, 1.00 |
g= 9; 2d      ; ni=1 |    15.30,     32, 1.00 |    15.30,     32, 1.00 |
g= 9; 2d      ; ni=2 |    13.41,     40, 1.00 |    13.41,     40, 1.00 |
g= 9; rand+2d ; ni=0 |     2.33,      3, 1.00 |     2.33,      3, 1.00 |
g= 9; rand+2d ; ni=1 |     2.25,      3, 1.00 |     2.25,      3, 1.00 |
g= 9; rand+2d ; ni=2 |     2.40,      3, 1.00 |     2.40,      3, 1.00 |

g=10; rand    ; ni=0 |     2.76,      4, 1.00 |     2.62,      4, 1.00 |
g=10; rand    ; ni=1 |     2.75,      4, 1.00 |     2.65,      3, 1.00 |
g=10; rand    ; ni=2 |     2.78,      4, 1.00 |     2.69,      3, 1.00 |
g=10; 2d      ; ni=0 |    20.88,     55, 1.00 |    20.88,     55, 1.00 |
g=10; 2d      ; ni=1 |    19.62,     49, 1.00 |    19.58,     49, 1.00 |
g=10; 2d      ; ni=2 |    22.89,     52, 1.00 |    22.75,     52, 1.00 |
g=10; rand+2d ; ni=0 |     2.71,      4, 1.00 |     2.61,      3, 1.00 |
g=10; rand+2d ; ni=1 |     2.73,      4, 1.00 |     2.58,      3, 1.00 |
g=10; rand+2d ; ni=2 |     2.62,      4, 1.00 |     2.52,      3, 1.00 |

g=11; rand    ; ni=0 |     2.98,      5, 1.00 |     2.87,      4, 1.00 |
g=11; rand    ; ni=1 |     2.87,      6, 1.00 |     2.75,      4, 1.00 |
g=11; rand    ; ni=2 |     2.92,      5, 1.00 |     2.76,      3, 1.00 |
g=11; 2d      ; ni=0 |    31.29,     71, 1.00 |    31.25,     71, 1.00 |
g=11; 2d      ; ni=1 |    28.14,     64, 1.00 |    28.14,     64, 1.00 |
g=11; 2d      ; ni=2 |    29.21,     75, 1.00 |    29.19,     75, 1.00 |
g=11; rand+2d ; ni=0 |     2.83,      6, 1.00 |     2.70,      3, 1.00 |
g=11; rand+2d ; ni=1 |     2.73,      4, 1.00 |     2.57,      3, 1.00 |
g=11; rand+2d ; ni=2 |     2.83,      4, 1.00 |     2.74,      3, 1.00 |

g=12; rand    ; ni=0 |     3.21,      7, 1.00 |     2.93,      4, 1.00 |
g=12; rand    ; ni=1 |     3.05,      9, 1.00 |     2.84,      4, 1.00 |
g=12; rand    ; ni=2 |     3.10,      7, 1.00 |     2.94,      4, 1.00 |
g=12; 2d      ; ni=0 |    40.93,    153, 1.00 |    39.89,     91, 1.00 |
g=12; 2d      ; ni=1 |    40.23,     95, 1.00 |    40.21,     95, 1.00 |
g=12; 2d      ; ni=2 |    44.49,    115, 1.00 |    44.39,    115, 1.00 |
g=12; rand+2d ; ni=0 |     2.98,      5, 1.00 |     2.81,      4, 1.00 |
g=12; rand+2d ; ni=1 |     2.92,      5, 1.00 |     2.84,      4, 1.00 |
g=12; rand+2d ; ni=2 |     2.94,      5, 1.00 |     2.79,      4, 1.00 |

g=13; rand    ; ni=0 |     3.68,     12, 1.00 |     3.05,      4, 1.00 |
g=13; rand    ; ni=1 |     3.30,      8, 1.00 |     2.99,      4, 1.00 |
g=13; rand    ; ni=2 |     3.51,     10, 1.00 |     3.10,      4, 1.00 |
g=13; 2d      ; ni=0 |    63.25,    145, 0.99 |    63.47,    145, 1.00 |
g=13; 2d      ; ni=1 |    60.47,    137, 1.00 |    60.27,    137, 1.00 |
g=13; 2d      ; ni=2 |    54.45,    126, 1.00 |    54.41,    126, 1.00 |
g=13; rand+2d ; ni=0 |     3.14,      7, 1.00 |     2.96,      4, 1.00 |
g=13; rand+2d ; ni=1 |     3.08,      6, 1.00 |     2.91,      3, 1.00 |
g=13; rand+2d ; ni=2 |     3.07,      5, 1.00 |     2.91,      4, 1.00 |

g=14; rand    ; ni=0 |     4.26,     27, 1.00 |     3.19,      4, 1.00 |
g=14; rand    ; ni=1 |     3.87,     15, 1.00 |     3.17,      4, 1.00 |
g=14; rand    ; ni=2 |     4.07,     22, 1.00 |     3.18,      4, 1.00 |
g=14; 2d      ; ni=0 |    99.51,    173, 0.99 |    99.68,    173, 1.00 |
g=14; 2d      ; ni=1 |    89.35,    213, 1.00 |    89.03,    213, 1.00 |
g=14; 2d      ; ni=2 |    84.24,    175, 1.00 |    84.14,    175, 1.00 |
g=14; rand+2d ; ni=0 |     3.83,     23, 1.00 |     3.08,      4, 1.00 |
g=14; rand+2d ; ni=1 |     3.52,      8, 1.00 |     3.09,      4, 1.00 |
g=14; rand+2d ; ni=2 |     3.61,     17, 1.00 |     3.03,      4, 1.00 |

g=15; rand    ; ni=0 |     6.31,     51, 1.00 |     3.52,      5, 1.00 |
g=15; rand    ; ni=1 |     5.56,     23, 1.00 |     3.48,      5, 1.00 |
g=15; rand    ; ni=2 |     6.04,     31, 1.00 |     3.51,      5, 1.00 |
g=15; 2d      ; ni=0 |   122.30,    263, 1.00 |   122.08,    263, 1.00 |
g=15; 2d      ; ni=1 |   112.18,    278, 1.00 |   111.70,    278, 1.00 |
g=15; 2d      ; ni=2 |   114.91,    292, 1.00 |   114.61,    292, 1.00 |
g=15; rand+2d ; ni=0 |     4.42,     22, 1.00 |     3.30,      4, 1.00 |
g=15; rand+2d ; ni=1 |     5.52,     67, 1.00 |     3.26,      4, 1.00 |
g=15; rand+2d ; ni=2 |     4.63,     23, 1.00 |     3.32,      4, 1.00 |

g=16; rand    ; ni=0 |     7.40,     37, 1.00 |     3.67,      5, 1.00 |
g=16; rand    ; ni=1 |     8.84,     87, 1.00 |     3.59,      5, 1.00 |
g=16; rand    ; ni=2 |     8.21,     40, 1.00 |     3.75,      5, 1.00 |
g=16; 2d      ; ni=0 |   178.18,    407, 0.99 |   180.36,    462, 1.00 |
g=16; 2d      ; ni=1 |   146.28,    410, 0.99 |   147.32,    410, 1.00 |
g=16; 2d      ; ni=2 |   159.23,    355, 0.99 |   159.85,    355, 1.00 |
g=16; rand+2d ; ni=0 |     7.10,     33, 1.00 |     3.70,      5, 1.00 |
g=16; rand+2d ; ni=1 |     6.31,     27, 1.00 |     3.54,      5, 1.00 |
g=16; rand+2d ; ni=2 |     7.04,     59, 1.00 |     3.59,      5, 1.00 |

g=17; rand    ; ni=0 |    12.07,     61, 1.00 |     3.81,      5, 1.00 |
g=17; rand    ; ni=1 |    11.60,     59, 1.00 |     3.74,      5, 1.00 |
g=17; rand    ; ni=2 |    15.41,     99, 1.00 |     3.85,      7, 1.00 |
g=17; 2d      ; ni=0 |   234.92,    653, 1.00 |   234.46,    653, 1.00 |
g=17; 2d      ; ni=1 |   255.01,    522, 1.00 |   254.77,    522, 1.00 |
g=17; 2d      ; ni=2 |   243.38,    568, 1.00 |   242.72,    568, 1.00 |
g=17; rand+2d ; ni=0 |     9.30,     77, 1.00 |     3.70,      5, 1.00 |
g=17; rand+2d ; ni=1 |     8.61,     40, 1.00 |     3.79,      5, 1.00 |
g=17; rand+2d ; ni=2 |     9.41,     79, 1.00 |     3.73,      5, 1.00 |

g=18; rand    ; ni=0 |    23.53,    155, 1.00 |     3.96,      7, 1.00 |
g=18; rand    ; ni=1 |    21.44,    117, 1.00 |     3.93,      5, 1.00 |
g=18; rand    ; ni=2 |    23.41,    222, 1.00 |     3.86,      5, 1.00 |
g=18; 2d      ; ni=0 |   364.25,    835, 1.00 |   363.13,    835, 1.00 |
g=18; 2d      ; ni=1 |   319.11,    794, 1.00 |   318.03,    792, 1.00 |
g=18; 2d      ; ni=2 |   335.69,    781, 1.00 |   334.95,    781, 1.00 |
g=18; rand+2d ; ni=0 |    18.00,    103, 1.00 |     3.85,      7, 1.00 |
g=18; rand+2d ; ni=1 |    15.77,    146, 1.00 |     3.92,      5, 1.00 |
g=18; rand+2d ; ni=2 |    18.08,    141, 1.00 |     3.89,      7, 1.00 |

g=19; rand    ; ni=0 |    40.79,    219, 1.00 |     4.16,      7, 1.00 |
g=19; rand    ; ni=1 |    41.68,    270, 1.00 |     4.24,      8, 1.00 |
g=19; rand    ; ni=2 |    43.11,    229, 1.00 |     4.36,     11, 1.00 |
g=19; 2d      ; ni=0 |   492.48,   1141, 1.00 |   492.08,   1141, 1.00 |
g=19; 2d      ; ni=1 |   477.74,   1053, 1.00 |   476.40,   1053, 1.00 |
g=19; 2d      ; ni=2 |   503.55,   1258, 0.98 |   501.88,   1258, 0.98 |
g=19; rand+2d ; ni=0 |    30.63,    193, 1.00 |     3.97,      7, 1.00 |
g=19; rand+2d ; ni=1 |    31.11,    280, 1.00 |     4.11,      7, 1.00 |
g=19; rand+2d ; ni=2 |    38.90,    287, 1.00 |     3.97,      7, 1.00 |

g=20; rand    ; ni=0 |    71.98,    479, 1.00 |     4.96,     11, 1.00 |
g=20; rand    ; ni=1 |    67.75,    566, 1.00 |     4.68,     10, 1.00 |
g=20; rand    ; ni=2 |    67.08,    697, 1.00 |     4.36,     10, 1.00 |
g=20; 2d      ; ni=0 |   653.86,   1626, 1.00 |   652.96,   1626, 1.00 |
g=20; 2d      ; ni=1 |   762.23,   1482, 1.00 |   760.85,   1482, 1.00 |
g=20; 2d      ; ni=2 |   637.62,   1608, 1.00 |   636.42,   1608, 1.00 |
g=20; rand+2d ; ni=0 |    58.23,    617, 1.00 |     4.24,     10, 1.00 |
g=20; rand+2d ; ni=1 |    53.61,    341, 1.00 |     4.22,      8, 1.00 |
g=20; rand+2d ; ni=2 |    56.92,    356, 1.00 |     4.58,     10, 1.00 |
```

