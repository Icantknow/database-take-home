
## Solution

(Oops I have a 6–9 PM shift...)

### Approach & Analysis

Both the query_distribution.png/query_frequency and also just the script used to generate the queries (💀) make it clear an exponential distribution is being used to generate the queries. The length of the random walk compared to the size of the graph implies the random walk is likely to visit most things with no optimization, but we could optimize to visit low ID nodes more often.

### Optimization Strategy

Nodes are stratified into 3 classes. "Class 3" nodes are the most commonly queried nodes and have 3 outgoing edges. "Class 2" nodes are next most common and have 2 outgoing edges. "Class 1" nodes are least common and have 1 outgoing edge. Actually, the number of outgoing edges is only tangentially related to how common they are. The intent is to have the most common nodes connect to each other, which implies they need to have more than the average number of vertices, and the best place to steal extra edges from is from the lesser-used nodes.

Additionally, class 1 nodes should probably not instantaneously return to class 3 nodes, or else most class 1 nodes will never get visited, so I'd want the class 1 nodes to not connect to class 3 nodes.

Randomly assigning edges seems like a good strategy to ensure most nodes are connected. If we got unlucky, we could regenerate graphs.

We can dramatically shorten path lengths by making it more likely to take edges that go towards class 3. This can be done by weighting each edge with the probability that it appears in a query.

### Implementation Details

The final code used is under `topheavy_weighted_graph()`. Class 3 nodes are the ones with IDs between 0 and 49 inclusive. Class 2 nodes are between 50 and 449 inclusive. Class 1 nodes are between 450 and 499 inclusive.

Class 3 nodes randomly connect to two other class 3 nodes and one other class 2 node. Class 2 nodes connect to two other random nodes. Class 1 nodes connect to one other random node in either class 2 or class 3.

Weights are assigned in the dictionary. The weight of a node is given by `np.exp(-x/10)/10`, which means lower IDs have a higher probability of being selected.

### Results

- No modification: Initial (80.0% / 578.50), Optimized (70.0% / 456.0), Score = **127.34**
- Put everything in a line: Initial (80.0% / 548.50), Optimized (89.0% / 2281.0), (-9.0% / +315.95%), Score = **108.18**
- Stratify nodes, randomly assign outgoing edges: Initial (80.0% / 539.0), Optimized (96.0% / 237.25), (+16.0% / -56.0%), Score = **209.79**
- Stratify nodes, rare nodes are a cycle: Initial (79.5% / 452.5), Optimized (**100.0%** / 575), (+20.5% / +27.1%), Score = **158.05**
- Stratify nodes, randomly assign outgoing edges, weighted edges: Initial (79.5% / 529.0), Optimized (85.0% / **20.0**), (+5.5% / -96.2%), Score = **366.55**

### Trade-offs & Limitations

There's quite a few trade-offs.

Most obvious is the one between success rate and median path length. I can attain 100% get rate by forcing all rare nodes to be visited, but this dramatically increased the length of cycles. Weighting edges so lower IDs are visited more often shortened the median path by a factor of 20.

Should all rare nodes even be reachable? It feels morally wrong to not do so, but objectively the get rate can be quite high by randomly assigning edges and not *guaranteeing* rare nodes.

It also matters how much you care about performance on rare events. It's probably possible to optimize super well for common nodes and then have a sharp "knee" in the path distribution graph where rare nodes are explored expensively. The median can be small, but if you were interested only in rare events, performance might be dramatically worse

The most subtle trade-off is not even obviously a trade-off. For "making all nodes reachable", while this is possible, it's unclear if this improves best performance. Maybe the average performance improves??? But also, this doubles code length, making it harder to maintain and understand, so not doing this seems to be the better option. Similar time and space constraints on writing the code itself were common.

### Iteration Journey

(I cleaned this up significantly after the 2 hour mark. There's lots of overlap with the above sections.)

If the nodes were points on, say, a plane, a bunch of random walks (10 seems small but there's probably not much difference between 10 and 1000 in terms of final graph structure) is equivalent to dropping a heat source, letting the heat diffuse, and hoping the heat reaches the point we're interested. Vaguely, we'd want to assign every node a location somehow and decide which nodes are close (k-means clustering?). It’s probably worth checking out the structure of the queries.

So queries are exponential distributed across nodes. This reminds me of several real-world scenarios, but none quite match. I can't jam all edges on the first few nodes.

Silly first idea: more commonly seen nodes get three edges. Middlemost nodes get two. Rarer nodes get one.

Actually before that, I’ll try just literally putting everything in a line. With a depth of 10000, one random walk should explore ~sqrt(10000)=100 nodes away, which doesn’t seem too bad, especially with multiple walks.

(This is `path_graph()`)

Everything in a line actually improve success rate, so random walks are sufficiently long to think about things continuously. I’ll try the "silly" idea. The exponential distribution has parameter 0.1, so treating the first and last 5*1/0.1 = 50 nodes separately might be useful. Edges are random and also land in distinct places. They could be bidirectional, but that’s a) more coding b) I like to fight randomness with randomness.

(This is `topheavy_graph()`)

With 800 outgoing edges going to each of the 50 class 1 nodes with 1/500 probabiility, I think from Chung-Erdos there’s a 72% chance at least one node is missed. I actually can make sure they are all indeed visited, 100 prisoners problem-style (with a cycle), but that means visiting one node will visit every node. Worth trying both.

If cycling works, should class 2 nodes should be a bidirectional path?

(This is `topheavy_cycle_graph()`)

Unfortunately, this doubled path length, while only improving percent of successful queries from 96% to 100%. This is a lower score (do the unsuccessful queries count in the median). Maybe 100% matters for some applications, but I’m usually happy with 96%. It’s also possible to force each class 1 node to be visited by randomly selecting a class 2 node to visit it. I’ll try this if I have time. I want to work with edge weights first.

For edge weights, assign each edge to the PDF of the target node ID seems reasonable.

(This is `topheavy_weighted_graph()`)

This works well!

Fiddling around with the sizes of classes or the edge weight probabilities doesn’t appear to improve things much. I even tried making class 1 larger than class 3, although the success rate dropped as I expected.

There's several things I would continue to try. I didn't have time to implement a version without weighting but with forcing every class 1 node to be visitable (and not in a cycle, since that's expensive for lengths). I also wonder about forcing every class 3 node to be visited.