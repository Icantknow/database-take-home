
## Solution

### Approach & Analysis

[Describe how you analyzed the query patterns and what insights you found]

### Optimization Strategy

[Explain your optimization strategy in detail]

### Implementation Details

[Describe the key aspects of your implementation]

### Results

No modification: Initial (80.0% / 578.50), Optimized (70.0% / 456.0), Score = 127.34
Put everything in a line: Initial (80.0% / 548.50), Optimized (89.0% / 2281.0), (-9.0% / +315.95%), Score = 108.18
Stratify nodes, randomly assign outgoing edges: Initial (80.0% / 539.0), Optimized (96.0% / 237.25), (+16.0% / -56.0%), Score = 209.79
Stratify nodes, class 1 is a cycle: Initial (79.5% / 452.5), Optimized (100.0% / 575), (+20.5% / +27.1%), Score = 158.05
Stratify nodes, randomly assign outgoing edges, weighted edges: Initial (79.5% / 529.0), Optimized (85.0% / 20.0), (+5.5% / -96.2%), Score = 366.55

### Trade-offs & Limitations

[Discuss any trade-offs or limitations of your approach]

### Iteration Journey

Well there’s no one obvious solution at first glance. Maybe it vaguely reminds me of PageRank? Imagining every node of the graph as corresponding to some point on, say, a plane, with edges meaning points are “close”, if there were a lot of random walks, that’s equivalent to dropping a heat source at a point, letting the heat diffuse for some amount of time, and hoping the heat reaches the point we’re interested in. So one option is to first assign every node of the graph to some location in n-D space, then connect points that are close. Two points that are connected by a query ought to be considered closer. We can update the location of a node to be, say, the centroid of all points so far queried that are connected to said node (this whole vaguely sounds like k-means clustering at this point). It’s probably worth checking out the structure of the queries.

Very interesting (exponential distribution…was I even supposed to look in scripts…then again depending on application it’s a very common prior distribution assumption and also it’s noted in the outputs while running the program even with no modifications). My first thought is to jam all the edges onto the first few nodes, but that’s expressly forbidden. Vaguely reminds me of various things, such as word frequencies, although “randomly generated the next possible word and so on and so forth until you get a word you want” doesn’t sound quite like any NLP task I can think of. There’s probably a different more relevant formulation. I do feel like there should be an exactly solvable continuous version of this problem though that can be derived from first principles, so thinking of analogies may be less helpful.

Random thought but 10 random walks seems a bit small. OTOH, I don’t expect the strategy to change much if more random walks are allowed, so I’ll just assume 10 is large…

Anyway, random silly idea for first try. The more commonly seen nodes get three edges. The middlemost nodes get two. The rarer nodes get one edge.

Actually before that, I’ll try just literally putting everything in a line. With a depth of 10000, a single random walk should explore up to about sqrt(10000) = 100 nodes away, which honestly doesn’t seem too bad. Should really be better than 100 because we run multiple random walks.

(path_graph())

Looking a bit closer at the initial graph seems random (although generated from the same seed each time)…. It also seems like the edges don’t have to be bidirectional. Fair enough.

Everything in a line actually does improve on success rate, which really just tells me the random walk is long enough to think about as if everything were some sort of continuous problem. The path lengths are dramatically worse. I’ll try restricting the last nodes to only have 1 outgoing edge and the first nodes to have 3. The constant used to generate the exponential distribution seems to be 0.1, so I think treating the first and last 5*1/0.1 = 50 nodes separately will be useful. First I’ll assign edges randomly. There will be 3 classes of nodes. Class 3 has 3 outgoing edges, class 2 has 2 outgoing edges, class 1 has 1 outgoing edge. Class 1 nodes only “outgo” to class 2, class 3 should go to class 3 and 2. Maybe two outgoing edges to class 3 for each class 3 node. I’ll let edges go randomly and also land in distinct places. They could be bidirectional, but that’s a) more coding b) I like to fight randomness with randomness.

(topheavy_graph())

I forgot to make sure all class 1 nodes are indeed reachable. I don’t want to spend too long on this, but with 800 outgoing edges going to each of the 50 class 1 nodes with 1/500 probabiility, I think from Chung-Erdos there’s a 72% chance of at least one node being missed. I actually can make sure they are all indeed visited, 100 prisoners problem-style (with a cycle), but that means visiting one node will visit every node. Worth trying both.

Also worth trying to see whether or not the class 2 nodes should be a bidirectional path if yes to cycle.

(topheavy_cycle_graph())

Unfortunately, adding a cycle nearly doubled the length to search for things, while improving percent of successful queries from 96% to 100%. This is a lower score. Maybe the 100% matters for some applications, but I’m happy with 96% for many things. It’s also possible to force each class 1 node to be visited by randomly selecting class 2 nodes to visit those nodes. I’ll try this if I have time. I want to work with edge weights first.

For edge weights, something reasonable seems to assign each edge to the probability of the target node being the node that is queried.

(topheavy_weighted_graph())

This works well!

Fiddling around with the sizes of classes or the edge weight probabilities doesn’t appear to improve things much. I even tried making class 1 larger than class 3, although the success rate dropped as I expected. It’s possible that if I forced class 1 to all be visited

---

* Be concise but thorough - aim for 500-1000 words total
* Include specific data and metrics where relevant
* Explain your reasoning, not just what you did