+++
title = "Node coordinate discovery by uniform hashing"
description = ""
date = 2017-06-22 
+++

### Abstract

Landmarks based routing allows efficient routing of messages between nodes in a
network. Given two nodes $x, y$ in a network $G$, $x$ can send a message to $y$
given that $x$ knows the network coordinate $Coord(y)$ of $y$.

Hence we need some method of translating a node id $y$ into his
network coordinate, $Coord(y)$.

We discuss here one method of translation: hashing of node id into uniform
valid coordinate.

The code for all the experiments is written in
[Rust](https://www.rust-lang.org), and could be found [here [github]](
https://github.com/Freedomlayer/freedomlayer_code/tree/master/landmarks_discovery_hashing/net_coords).


### Hashing node ids into coordinates

Assume that we had some deterministic way of hashing a node id, $y$, into some
coordinate $H(y)$. (Note that $H(y)$ has no relation to $Coord(y)$).

The node $y$ will then calculate $H(y)$, obtaining some network coordinate.
$y$ will then find the node $z$ with network coordinate closest to $H(y)$, and
let $z$ keep $y$'s coordinate: $Coord(y)$.

If some node $x$ wants to send a message to $y$, $x$ will first calculate
$H(y)$. $H$ is deterministic, so $x$ will get the same value of $H(y)$ that $y$
got. $x$ will then find the node $z$ with a coordinate closest to $H(y)$, and
ask him for $Coord(y)$.

Finally, knowing $Coord(y)$, $x$ can send a message to $y$.

There are some requirements to make this technique work:

- The process of applying the function $H$, followed by finding a node with
  closest coordinate to the result of $H$ should distribute close to uniformly
  over the set of nodes. This is because we don't want that few nodes will keep
  the coordinates of most of the nodes in the network.

- The coordinate produced by $H$ should look like a "real coordinate". It
  should satisfy the triangle inequalities, and hopefully look like other
  coordinates in the network.


### Generating random coordinates

We want to create a function $H: V(G) \rightarrow Coordinates$ that produces
deterministic results, even when invoked from different nodes. For this to
happen, we need to rely on some common knowledge that all nodes have.

We will rely on the knowledge of the coordinates of all the landmarks:
$Coord(l_i)$ for $1 \leq i \leq k$.

We examined a few algorithms of generating random coordinates that are uniform
over the set of nodes in the network. We introduce here two algorithms that
gave us good results.

### `randomize_coord_landmarks_coords`

The first algorithm, called `randomize_coord_landmarks_coords`, does the
following to create a random coordinate: For entry $j$, we pick a random entry
of a random coordinate from all the coordinates $Coord(l_i)$, $1 \leq i \leq k$.

Example rust code for this algorithm:

```rust
pub fn randomize_coord_landmarks_coords<R: Rng>(landmarks: &Vec<usize>, coords: &Vec<Vec<u64>>,
                    mut rng: &mut R) -> Vec<u64> {

    let rand_landmark: Range<usize> = 
        Range::new(0, landmarks.len());
    let interval_size: u64 = 2_u64.pow(0_u32);

    let mut coord: Vec<u64> = vec![];
    for _ in 0 .. landmarks.len() {
        let mut cur_value = 0;
        for _ in 0 .. interval_size {
            cur_value += coords[landmarks[rand_landmark.ind_sample(&mut rng)]][rand_landmark.ind_sample(&mut rng)];
        }
        coord.push(cur_value);
    }
    coord
}
```

This algorithm is not very smart. It does not produce coordinates that satisfy
the triangle inequalities, but it does produce coordinates that distribute
somewhat uniformly over all the nodes in the network.

We include here some results for this algorithm, for small sized networks.
Explanation about how to read the result: $g$ means log in base 2 of the size
of the network. (For example, $g=9$ means a network of $512$ nodes).
We check against $4$ types of networks: 
- rand: A random network. Every node is connected to $1.5 * \log(n)$ other
  nodes.
- 2d: A two dimensional grid.
- rand+2d: A sum of a random network with a two dimensional grid.
- planar: A planar graph: Nodes are randomized in a two dimensional plane, and
  every node is connected to the closet $1.5 * \log(n)$ nodes.

$ni$ means network iteration. In this experiment we make two iterations for
every type of network. We do this to have more numbers to look at.

In this experiment we generate $n = |V(G)|$ random coordinates, and for each
coordinate we find the node with a coordinate closets to the generated coordinate.

For each node we count the amount of times it was the closest node to the
generated coordinate. `max_nr` is the maximum number of times we visited a
single node. If `max_nr` is very large, it means that only a few nodes were
visited during the experiment, and the algorithm chosen for randomizing
coordinates is not very uniform.

We also for every randomly generated coordinate how many nodes are fit to be
the node with closest node to this coordinate (There could be more than one).
`average_min_indices` is the average of this amount, over all the randomly
generated coordinates. It is usually $1$, because usually there is only one
closest node.

Parameters for this experiment:

- Amount of landmarks: $\log(n)^2$
- Tie breaking by randomizing edge distances: Done with interval $[0x10000,
  0x20000)$. (This helps with having a low `average_min_indices` value).

```bash
git clone https://github.com/Freedomlayer/freedomlayer_code
cd landmarks_discovery_hashing/net_coords
```

Edit `landmarks_discovery_hashing/net_coords/bin/randomize_coord_balanced` to
use `randomize_coord_landmarks_coords`.

Then run:

```bash
cargo run --bin randomize_coord_balanced --release
```

Results:

```
g=10; rand    ; ni=0 |max_nr =    8| average_min_indices = 1
g=10; rand    ; ni=1 |max_nr =    6| average_min_indices = 1.0009765625
g=10; 2d      ; ni=0 |max_nr =   11| average_min_indices = 1
g=10; 2d      ; ni=1 |max_nr =    7| average_min_indices = 1
g=10; rand+2d ; ni=0 |max_nr =   11| average_min_indices = 1.0009765625
g=10; rand+2d ; ni=1 |max_nr =    8| average_min_indices = 1.0009765625
g=10; planar  ; ni=0 |max_nr =   14| average_min_indices = 1
g=10; planar  ; ni=1 |max_nr =   15| average_min_indices = 1

g=11; rand    ; ni=0 |max_nr =    9| average_min_indices = 1.00048828125
g=11; rand    ; ni=1 |max_nr =    9| average_min_indices = 1.0009765625
g=11; 2d      ; ni=0 |max_nr =   11| average_min_indices = 1
g=11; 2d      ; ni=1 |max_nr =    8| average_min_indices = 1
g=11; rand+2d ; ni=0 |max_nr =   11| average_min_indices = 1.0004938271604937
g=11; rand+2d ; ni=1 |max_nr =    8| average_min_indices = 1
g=11; planar  ; ni=0 |max_nr =   14| average_min_indices = 1
g=11; planar  ; ni=1 |max_nr =   17| average_min_indices = 1

g=12; rand    ; ni=0 |max_nr =   15| average_min_indices = 1.000732421875
g=12; rand    ; ni=1 |max_nr =   16| average_min_indices = 1.000244140625
g=12; 2d      ; ni=0 |max_nr =   14| average_min_indices = 1
g=12; 2d      ; ni=1 |max_nr =   10| average_min_indices = 1
g=12; rand+2d ; ni=0 |max_nr =    8| average_min_indices = 1.000244140625
g=12; rand+2d ; ni=1 |max_nr =   11| average_min_indices = 1.00048828125
g=12; planar  ; ni=0 |max_nr =   13| average_min_indices = 1.000244140625
g=12; planar  ; ni=1 |max_nr =   21| average_min_indices = 1.000244140625

g=13; rand    ; ni=0 |max_nr =   26| average_min_indices = 1
g=13; rand    ; ni=1 |max_nr =   15| average_min_indices = 1.0001220703125
g=13; 2d      ; ni=0 |max_nr =   14| average_min_indices = 1
g=13; 2d      ; ni=1 |max_nr =   12| average_min_indices = 1.000246913580247
g=13; rand+2d ; ni=0 |max_nr =   18| average_min_indices = 1.0003703703703704
g=13; rand+2d ; ni=1 |max_nr =   18| average_min_indices = 1
g=13; planar  ; ni=0 |max_nr =   44| average_min_indices = 1
g=13; planar  ; ni=1 |max_nr =   19| average_min_indices = 1
```


### `randomize_coord_rw_directional`

The first algorithm presented, `randomize_coord_landmarks_coords`, generates
random coordinates that don't satisfy the triangle inequality with respect to
the landmarks coordinates.

Let $d(x,y)$ is the length of the shortest path between $x$ and $y$ in the
network, $d$ is a metric over $V(G)$, the vertices of the network.
Let $\{l_1, \dots , l_k\} \subseteq V(G)$ be the chosen landmarks for the network.
For every node $x$, we define: 

$$Coord(x) := \{d(x,l_1), \dots , d(x, l_k)\}$$

We then define:

$$mdist(Coord(x), Coord(y)) := \max_{1 \leq i \leq k} {|Coord(x)_i - Coord(y)_i|}$$

The virtual distance between $x$ and $y$.  $mdist$ satisfies $mdist(Coord(x),
Coord(y)) \leq d(x,y)$. This follows from the triangle inequality over the
metric $d$.

We call a coordinate $A$ valid if it satisfies all the triangle inequalities
according to the metric $d$. In other words, $A_i + A_j \geq d(l_i,l_j) \geq
|A_i - A_j|$. Hence if two coordinates $A$ and $B$ are valid, $\alpha \cdot A +
(1 - \alpha) \cdot B$ is also a valid coordinate for $\alpha \in [0,1]$.
Therefore the set of valid coordinates is convex.

In addition, if a coordinate $A = (a_1, \dots , a_k)$ is valid, then $A +
\beta \cdot 1 = (a_1 + \beta, \dots, a_k + \beta)$ is also valid. Hence the set
of valid coordinates has infinite volume.

If we add an artificial constraint of $coord_i \leq const * \max_i
{Coord(l_j)_i}$, we get a bounded set of valid coordinates, with contraints
that are linear inequalities. This kind of shape is also known as a polytope.

It appears that the question of choosing a random point inside the volume of a
polytope was studied in the past. (Mostly for approximation of volume for high
dimensional bodies, and finding redundant equations in a set of linear
inequalities). 

See, for example:

- "Hit-And-Run algorithms for the identification of nonredundant linear
  equations"
  (H.C.P. Berbee, C.G.E Boender, A.H.G Rinnooy Kan, C.L. Scheffer, R.L. Smith,
  J. Telgen)
- "Random walks and an O*(n^5) volume algorithm for convex bodies" (Ravi
  Kannan, László Lovász, Miklós Simonovits)

We use an algorithm that is similar to the one presented in "Hit-AndRun
algorithms". It works as follows:

We begin with some coordinate $C$. We can possibly pick one of $Coord(l_i)$ for
some $1 \leq i \leq k$ as a starting point, or possibly the average of all the
coordinates of the landmarks (If we use distance randomization, the average
is usually a valid coordinate).

In every step we pick a random entry index $j$, where $1 \leq j \leq k$. We
check what is the possible range for $C_j$ so that it still satisfies the
triangle inequality. Let $a$ be the small possible value, and $b$ be the highest
possible value. We then pick a random value uniformly in $[a,b]$, and finally
set $C_j$ to be the new value of entry $j$ of the coordinate.

After enough steps, we expect to reach a random coordinate valid coordinate.

This is the code for this algorithm:

```rust
pub fn randomize_coord_rw_directional<R: Rng>(upper_constraints: &Vec<u64>, 
                  landmarks: &Vec<usize>, coords: &Vec<Vec<u64>>,
                    mut rng: &mut R) -> Vec<u64> {

    let entry_range: Range<usize> = Range::new(0, landmarks.len());
    // Start from the average of all landmarks:
    let mut cur_coord = average_landmarks(&landmarks, &coords);
    // Start from a random landmark:
    // let mut cur_coord = coords[landmarks[entry_range.ind_sample(rng)]].clone();

    let mut good_iters = 0;
    while good_iters < landmarks.len().pow(2) * 4 {
        let i = entry_range.ind_sample(rng);

        // Get range of valid values for entry number i:
        let (low, high) = get_entry_rw_range(&cur_coord, i, 
                                             &upper_constraints, landmarks, coords);

        if low < high {
            good_iters += 1;
        }

        // Set the new random value to the entry:
        let value_range: Range<u64> = Range::new(low, high + 1);
        cur_coord[i] = value_range.ind_sample(rng);
    }

    cur_coord
}
```

We check the uniformity of the resulting generated coordinates, the same way we
did with the previous algorithm: `randomize_coord_landmarks_coords`.

```bash
git clone https://github.com/Freedomlayer/freedomlayer_code
cd landmarks_discovery_hashing/net_coords
cargo run --bin randomize_coord_balanced --release
```

```
g=10; rand    ; ni=0 |max_nr =   21| average_min_indices = 1
g=10; rand    ; ni=1 |max_nr =   26| average_min_indices = 1
g=10; 2d      ; ni=0 |max_nr =   21| average_min_indices = 1
g=10; 2d      ; ni=1 |max_nr =   58| average_min_indices = 1
g=10; rand+2d ; ni=0 |max_nr =   24| average_min_indices = 1
g=10; rand+2d ; ni=1 |max_nr =   26| average_min_indices = 1
g=10; planar  ; ni=0 |max_nr =   33| average_min_indices = 1
g=10; planar  ; ni=1 |max_nr =   16| average_min_indices = 1

g=11; rand    ; ni=0 |max_nr =   39| average_min_indices = 1
g=11; rand    ; ni=1 |max_nr =   78| average_min_indices = 1
g=11; 2d      ; ni=0 |max_nr =   63| average_min_indices = 1
g=11; 2d      ; ni=1 |max_nr =   47| average_min_indices = 1
g=11; rand+2d ; ni=0 |max_nr =   27| average_min_indices = 1
g=11; rand+2d ; ni=1 |max_nr =   32| average_min_indices = 1
g=11; planar  ; ni=0 |max_nr =   72| average_min_indices = 1
g=11; planar  ; ni=1 |max_nr =   28| average_min_indices = 1
```


### `find_coord` experiment

We perform an end-to-end experiment, that works as follows.
We pick a pair of nodes $x,y \in V(G)$ randomly. We then generate a random
coordinate using a chosen coordinate generation algorithm. Call that coordinate
$C$.

$x$ then searches for a node with coordinate close to $C$, call that node $z$.
Finally $y$ searches for $z$, given the coordinate $C$.

$y$ searches for $z$ as follows: $y$ first chooses a random coordinate $M$ and
finds a node $m$ close to $M$. From that node, $y$ tries to find the node $z$,
given the coordinate $C$. If many iterations pass and $y$ couldn't find $z$
from $m$, $y$ generates a new random coordinate $M$ and tries again.

In the results tables:
$g$ is the log of the size of the network, $ni$ means the network iteration.
Again we check against $4$ types of networks: rand, 2d, rand+2d, planar.
`avg_path_len` is the average resulting path length for arriving from $y$ to
$z$. 

`avg_num_attemps` is the most important number for this experiment: It
shows how many times $y$ had to generate a new $M$ and try to find $z$.
Ideally this number should be as low as possible: 1.

Results for `randomize_coord_landmarks_coords`:

```bash
git clone https://github.com/Freedomlayer/freedomlayer_code
cd landmarks_discovery_hashing/net_coords
```

Edit `landmarks_discovery_hashing/net_coords/bin/find_coords` to
use `randomize_coord_landmarks_coords`.

Then run:

```bash
cargo run --bin find_coords --release
```

```
max_visits = 2
num_pairs = 10

g= 8; rand    ; ni=0 |avg_path_len =   17.100 |avg_num_attempts =  1.000 |
g= 8; rand    ; ni=1 |avg_path_len =   15.900 |avg_num_attempts =  1.000 |
g= 8; 2d      ; ni=0 |avg_path_len =   47.600 |avg_num_attempts =  2.200 |
g= 8; 2d      ; ni=1 |avg_path_len =   55.500 |avg_num_attempts =  1.700 |
g= 8; rand+2d ; ni=0 |avg_path_len =   15.400 |avg_num_attempts =  1.000 |
g= 8; rand+2d ; ni=1 |avg_path_len =   17.700 |avg_num_attempts =  1.000 |
g= 8; planar  ; ni=0 |avg_path_len =   25.100 |avg_num_attempts =  1.000 |
g= 8; planar  ; ni=1 |avg_path_len =   23.300 |avg_num_attempts =  2.200 |

g= 9; rand    ; ni=0 |avg_path_len =   21.200 |avg_num_attempts =  1.000 |
g= 9; rand    ; ni=1 |avg_path_len =   13.900 |avg_num_attempts =  1.000 |
g= 9; 2d      ; ni=0 |avg_path_len =   37.100 |avg_num_attempts =  1.900 |
g= 9; 2d      ; ni=1 |avg_path_len =   41.700 |avg_num_attempts =  2.700 |
g= 9; rand+2d ; ni=0 |avg_path_len =   19.800 |avg_num_attempts =  1.000 |
g= 9; rand+2d ; ni=1 |avg_path_len =   17.400 |avg_num_attempts =  1.000 |
g= 9; planar  ; ni=0 |avg_path_len =   48.500 |avg_num_attempts =  1.200 |
g= 9; planar  ; ni=1 |avg_path_len =   50.400 |avg_num_attempts =  3.500 |

g=10; rand    ; ni=0 |avg_path_len =   25.800 |avg_num_attempts =  1.000 |
g=10; rand    ; ni=1 |avg_path_len =   16.500 |avg_num_attempts =  1.000 |
g=10; 2d      ; ni=0 |avg_path_len =   48.300 |avg_num_attempts =  2.800 |
g=10; 2d      ; ni=1 |avg_path_len =  126.100 |avg_num_attempts =  2.200 |
g=10; rand+2d ; ni=0 |avg_path_len =   22.400 |avg_num_attempts =  1.000 |
g=10; rand+2d ; ni=1 |avg_path_len =   15.400 |avg_num_attempts =  1.000 |
g=10; planar  ; ni=0 |avg_path_len =   59.400 |avg_num_attempts =  1.200 |
g=10; planar  ; ni=1 |avg_path_len =   81.900 |avg_num_attempts =  3.400 |

g=11; rand    ; ni=0 |avg_path_len =   26.000 |avg_num_attempts =  1.900 |
g=11; rand    ; ni=1 |avg_path_len =   23.400 |avg_num_attempts =  1.000 |
g=11; 2d      ; ni=0 |avg_path_len =   64.900 |avg_num_attempts = 40.200 |
g=11; 2d      ; ni=1 |avg_path_len =   56.100 |avg_num_attempts = 33.600 |
g=11; rand+2d ; ni=0 |avg_path_len =   51.200 |avg_num_attempts =  1.000 |
g=11; rand+2d ; ni=1 |avg_path_len =   25.700 |avg_num_attempts =  1.000 |
g=11; planar  ; ni=0 |avg_path_len =   37.700 |avg_num_attempts =  5.800 |
g=11; planar  ; ni=1 |avg_path_len =   39.700 |avg_num_attempts =  2.800 |
```


Results for `randomize_coord_rw_directional`:

```bash
git clone https://github.com/Freedomlayer/freedomlayer_code
cd landmarks_discovery_hashing/net_coords
cargo run --bin find_coords --release
```

```
max_visits = 2
num_pairs = 10

g= 8; rand    ; ni=0 |avg_path_len =   15.400 |avg_num_attempts =  1.000 |
g= 8; rand    ; ni=1 |avg_path_len =   15.100 |avg_num_attempts =  1.000 |
g= 8; 2d      ; ni=0 |avg_path_len =   41.900 |avg_num_attempts =  1.200 |
g= 8; 2d      ; ni=1 |avg_path_len =   38.200 |avg_num_attempts =  1.100 |
g= 8; rand+2d ; ni=0 |avg_path_len =   16.700 |avg_num_attempts =  1.000 |
g= 8; rand+2d ; ni=1 |avg_path_len =   15.200 |avg_num_attempts =  1.000 |
g= 8; planar  ; ni=0 |avg_path_len =   26.400 |avg_num_attempts =  2.900 |
g= 8; planar  ; ni=1 |avg_path_len =   26.800 |avg_num_attempts =  1.400 |

g= 9; rand    ; ni=0 |avg_path_len =   17.100 |avg_num_attempts =  1.000 |
g= 9; rand    ; ni=1 |avg_path_len =   21.100 |avg_num_attempts =  1.000 |
g= 9; 2d      ; ni=0 |avg_path_len =   47.200 |avg_num_attempts =  2.400 |
g= 9; 2d      ; ni=1 |avg_path_len =   91.300 |avg_num_attempts =  5.900 |
g= 9; rand+2d ; ni=0 |avg_path_len =   14.800 |avg_num_attempts =  1.000 |
g= 9; rand+2d ; ni=1 |avg_path_len =   15.500 |avg_num_attempts =  1.000 |
g= 9; planar  ; ni=0 |avg_path_len =   33.100 |avg_num_attempts =  1.000 |
g= 9; planar  ; ni=1 |avg_path_len =   29.400 |avg_num_attempts =  1.600 |

g=10; rand    ; ni=0 |avg_path_len =   21.000 |avg_num_attempts =  1.000 |
g=10; rand    ; ni=1 |avg_path_len =   20.200 |avg_num_attempts =  1.000 |
g=10; 2d      ; ni=0 |avg_path_len =   71.200 |avg_num_attempts =  4.700 |
g=10; 2d      ; ni=1 |avg_path_len =  132.300 |avg_num_attempts =  3.900 |
g=10; rand+2d ; ni=0 |avg_path_len =   33.900 |avg_num_attempts =  1.000 |
g=10; rand+2d ; ni=1 |avg_path_len =   19.100 |avg_num_attempts =  1.000 |
g=10; planar  ; ni=0 |avg_path_len =   46.800 |avg_num_attempts = 10.100 |
g=10; planar  ; ni=1 |avg_path_len =   54.500 |avg_num_attempts =  2.800 |

g=11; rand    ; ni=0 |avg_path_len =   77.300 |avg_num_attempts =  1.000 |
g=11; rand    ; ni=1 |avg_path_len =   19.000 |avg_num_attempts =  1.000 |
g=11; 2d      ; ni=0 |avg_path_len =  184.200 |avg_num_attempts = 17.900 |
g=11; 2d      ; ni=1 |avg_path_len =  101.100 |avg_num_attempts =  9.300 |
g=11; rand+2d ; ni=0 |avg_path_len =   24.100 |avg_num_attempts =  1.000 |
g=11; rand+2d ; ni=1 |avg_path_len =   18.400 |avg_num_attempts =  1.000 |
g=11; planar  ; ni=0 |avg_path_len =  104.800 |avg_num_attempts = 39.900 |
g=11; planar  ; ni=1 |avg_path_len =  108.900 |avg_num_attempts = 37.800 |
```


Note that `avg_num_attempts` is almost always $1$ for the networks rand,
rand+2d, but it blows up for the networks 2d, planar.

This means that the idea of node discovery by node id hashing is practical for
random looking network. However, at this point we don't know how to make it
work for planar like networks.

