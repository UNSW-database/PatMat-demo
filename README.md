# Preliminary
We implement distributed subgraph matching system called `PatMat` based on [Timely Dataflow](https://github.com/frankmcsherry/timely-dataflow).

Currently we adopt the most widely used the subgraph isomorphism semantics for subgraph matching. Specifically, given a pattern graph `P(V(P), E(P), Lp)` and a data graph `G(V(G), E(G), Lg)`, where `V(*), E(*) and L*` denote the nodes, edges and labels (of nodes/edges), we want to find all isomorphisms `f: V(P) -> V(G)`, such that:
1. For all `v in V(P)`, let `u = f(v)`, `Lp(v) = Lg(u)`.
2. Given two `v1, v2 in V(P)`, and let `u1 = f(v1), u2 = f(v2)`, if `(v1, v2) in E(P)`, then `(u1, u2) in E(G)` and `Lp((v1, v2)) = Lg((u1, u2))`. 

Our ultimate target is to support all use cases of Neo4j's [Cypher](https://neo4j.com/developer/cypher-query-language/) in the distributed (and/or streaming) context, for both offline query and online incremental query. 

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
We have included four utilities in the project, namely graph partition (`graph_part`), join plan computation (`compute_join_plan`) and pattern matching (`patmat`). 

`graph_part`: Hash partition the graph data. Specifically, graph_part takes as input a csv file containing the edges of the graph. Then each node and its neighbors will be randomly assigned to one worker in the cluster. Each machine, after collecting the owning nodes, will maintain them as a partitioned graph in its local storage (as indicated by `BasicStorage`).  

`compute_join_plan`: Given a query, compute the join plan (execution plan) according to the algorithms. Currently, we include two supported schemes: BinaryJoin and GenericJoin (with BigJoin plan and CrystalJoin plan). The join plan needs to be computed before running the patmat bin for the actual matching.

`patmat`: The main routine of doing pattern matching. While specifying the graph data and join plan (by `compute_join_plan`), we call the core pattern matching routine for the query. 


Each utility will include its own instructions by calling.
```
[graph_part|tri_part|compute_join_plan|patmat] --help
```
