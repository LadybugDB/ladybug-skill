# Data Science with Graph Databases

## Kuzu (Legacy)

Kuzu encouraged integration with external Python libraries for graph data science:

### NetworkX Integration

```python
import kuzu
import networkx as nx

db = kuzu.Database("myDB")
conn = kuzu.Connection(db)

# Export to NetworkX for analysis
query = "MATCH (n)-[r]->(m) RETURN n, r, m"
G = nx.DiGraph()
# ... custom export logic
```

### Cypher UDFs

Kuzu supported User Defined Functions for custom graph analytics:

```cypher
CREATE MACRO my_udf(x, y) AS x + y;
```

## Ladybug (Preferred)

Ladybug prefers native integration with Icebug for data science workflows.

### Icebug (v13.2)

[Icebug](https://github.com/Ladybug-Memory/icebug) is a fork of [networkit](https://networkit.github.io/) with emphasis on Apache Arrow. It is much more efficient to construct and run algorithms compared to NetworkX. This skill targets Icebug **13.2** (compatible with Ladybug **0.21.2**):

```bash
pip install icebug==13.2
```

- Arrow-native memory layout for efficient data transfer
- Faster algorithm execution on large graphs
- Direct integration with Ladybug's Arrow-backed tables

Icebug inherits 100+ algorithms from networkit:

| Category | Algorithms |
|----------|------------|
| **Centrality** (~20-25) | Betweenness (exact + approximations), PageRank, Closeness, Harmonic, Katz, Eigenvector, CoreDecomposition, LaplacianCentrality, and more |
| **Community Detection** (~10-15) | Louvain/PLM, PLP, LFM, CNM, Infomap, Ego-splitting, SCD variants |
| **Distance/Shortest Paths** (~10-15) | BFS, DFS, Dijkstra, A*, APSP, SPSP, BidirectionalBFS/Dijkstra, SSSP variants, Eccentricity, Diameter |
| **Components** (~5-8) | ConnectedComponents, StronglyConnectedComponents, BiconnectedComponents |
| **Graph Generators** (~15-20) | Erdős-Rényi, Barabási-Albert, Chung-Lu, LFR, Hyperbolic, Dorogovtsev-Mendes, Watts-Strogatz |
| **Sparsification** (~5-8) | Various edge scoring and sparsification methods |
| **Link Prediction** (~10) | Jaccard, Adamic-Adar, Common Neighbors |

```python
import networkit as nk

# Create graph from Arrow data
G = nk.Graph.fromCSR(...)

# Run algorithms efficiently
pagerank = nk.pagerank(G)
communities = nk.louvain(G)
```

### Native Graph Algorithms (legacy)

Use the `algo` extension for in-database graph algorithms:

```cypher
LOAD EXTENSION algo;

-- Create projected graph
CALL PROJECT_GRAPH('my_graph', ['Person'], ['KNOWS']);

-- Run PageRank
CALL algo.pagerank('my_graph') YIELD node, rank ORDER BY rank DESC;

-- Run Louvain community detection
CALL algo.louvain('my_graph') YIELD node, community;

-- K-Core decomposition
CALL algo.kcore('my_graph', k:=3) YIELD node, core;

-- Shortest path
CALL algo.shortest_path('start_node', 'end_node');
```

### Available Algorithms

- PageRank
- Louvain (community detection)
- K-Core Decomposition
- Strongly Connected Components
- Weakly Connected Components
- Shortest Paths

### Python Integration

```python
import ladybug as lb

db = lb.Database(":memory:")
conn = lb.Connection(db)

# Query results to DataFrame
df = conn.execute("MATCH (n) RETURN n.name, n.age").get_as_df()
```

## Migration Notes

- Replace NetworkX export patterns with arrow and icebug algorithms
- Leverage Icebug-format (disk and memory) for large-scale graph analytics
