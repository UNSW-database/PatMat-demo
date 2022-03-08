# Preliminary
We implement distributed subgraph matching system called `PatMat` based on [Timely Dataflow](https://github.com/frankmcsherry/timely-dataflow).

Currently we adopt the most widely used the subgraph isomorphism semantics for subgraph matching. Specifically, given a pattern graph `P(V(P), E(P), Lp)` and a data graph `G(V(G), E(G), Lg)`, where `V(*), E(*) and L*` denote the nodes, edges and labels (of nodes/edges), we want to find all isomorphisms `f: V(P) -> V(G)`, such that:
1. For all `v in V(P)`, let `u = f(v)`, `Lp(v) = Lg(u)`.
2. Given two `v1, v2 in V(P)`, and let `u1 = f(v1), u2 = f(v2)`, if `(v1, v2) in E(P)`, then `(u1, u2) in E(G)` and `Lp((v1, v2)) = Lg((u1, u2))`. 

Our ultimate target is to support all use cases of Neo4j's [Cypher](https://neo4j.com/developer/cypher-query-language/) in the distributed (and/or streaming) context, for both offline query and online incremental query. 

We refer to the following academic works for the core pattern matching component:

* MultiwayJoin [1]: Propose one-round join via duplicating edges.
* StarJoin [2]: Decompose the query graph into a set of stars, then enumerate matches for each star, and finally join all matched stars. 
* TwinTwigJoin [3]: Optimising Star-Join via TwinTwig decomposition. Shown to be instance optimal over Star-Join.
* CliqueJoin [4]: Optimising TT-Join by allowing decomposing the query into cliques and stars. Optimal solution in the paradigm of decomposition-and-join.
* BiGJoin [5]: An implmentation of worst-case optimal join algorithm in Timely dataflow. 
* CrystalJoin [6]: Use vertex cover to compress intermediate results.

# The Basic Storage
We introduce below a very basic storage designed for timely processing. The structure looks like:
```
work_dir
.... persist_data
.... .... DATA
.... .... .... data_prefix
.... .... .... others
.... .... TEMP
```
Suppose we are now handling a dataset named "lj", and the working directory is "/tmp/work_dir", one will expect the graph data stored as `/tmp/work_dir/lj/DATA/<data_prefix>`, where `data_prefix` is the prefix of the partitioned data. Note that one machine can run multiple workers, and they share the same local copy.

# Usage of the bins
We have included four utilities in the project, namely graph partition (`graph_part`), triangle partition (`tri_part`), join plan computation (`compute_join_plan`) and pattern matching (`patmat`). 

`graph_part`: Hash partition the graph data. Specifically, a node and its neighbors will be randomly assigned to one worker in the cluster. Each machine, after collecting the owning nodes, will maintain them as a partitioned graph in its local storage (as indicated by `BasicStorage`). 

`tri_part`: The triangle partition introduced in [3]. Basically, given three nodes v, v1, v2 where v1, v2 are v's neighbors, if (v1, v2) is also an edge of the graph, it will be included into v's partitioned graph.

`compute_join_plan`: Given a query, compute the join plan (execution plan) according to the algorithms. Currently, we include two supported schemes: BinaryJoin and GenericJoin (with BigJoin plan and CrystalJoin plan).

`patmat`: The main routine of doing pattern matching. While specifying the graph data and join plan (by `compute_join_plan`), we call the core pattern matching routine for the query. 

Compile each utility using:
```
cargo build --release --bin [graph_part|tri_part|compute_join_plan|patmat]
```

Each utility will include its own instructions by calling.
```
[graph_part|tri_part|compute_join_plan|patmat] --help
```

# Version Map.
V0.1 Support isomorphism-based subgraph matching for simple labelled graph. The simple labelled graph, there is only one edge between two nodes, and self-loop is not allowed. The only properties of a node (and edge), is a globally unique id and a label (type). 

V0.2 Support [property graph](https://neo4j.com/developer/graph-database/). We will implement a full functional property graph based on [RockDB](https://github.com/facebook/rocksdb).

V0.2.5 PatMat Demo. A demo site of the system.

V0.3 Incremental query support. 
 
# References
1. Enumerating subgraph instances using map-reduce, Afrati et al., ICDE 2013.
2. Efficient subgraph matching on billion node graphs, Sun et al., VLDB 2012.
3. Scalable subgraph enumeration in MapReduce: a cost-oriented approach, Lai et al., VLDBJ 2017.
4. Scalable distributed subgraph enumeration, Lai et al., VLDB 2017.
5. Distributed Evaluation of Subgraph Queries Using Worst-case Optimal Low-Memory Dataflows, Frank et al, VLDB 2018.
6. Subgraph Matching: on compression and Computation, Miao et al, VLDB 2018.
