---
title: Graph traversals in AQL
menuTitle: Traversals
weight: 10
description: >-
  You can traverse named graphs and anonymous graphs with a native AQL
  language construct
---
## Syntax

There are two slightly different syntaxes for traversals in AQL, one for
- [named graphs](../../graphs/_index.md#named-graphs) and another to
- specify a [set of edge collections](#working-with-collection-sets)
  ([anonymous graph](../../graphs/_index.md#anonymous-graphs)).

### Working with named graphs

The syntax for AQL graph traversals using named graphs is as follows
(square brackets denote optional parts and `|` denotes alternatives):

```aql
FOR vertex[, edge[, path]]
  IN [min[..max]]
  OUTBOUND|INBOUND|ANY startVertex
  GRAPH graphName
  [PRUNE [pruneVariable = ]pruneCondition]
  [OPTIONS options]
```

- `FOR`: emits up to three variables:
  - **vertex** (object): the current vertex in a traversal
  - **edge** (object, *optional*): the current edge in a traversal
  - **path** (object, *optional*): representation of the current path with
    two members:
    - `vertices`: an array of all vertices on this path
    - `edges`: an array of all edges on this path
- `IN` `min..max`: the minimal and maximal depth for the traversal:
  - **min** (number, *optional*): edges and vertices returned by this query
    start at the traversal depth of *min* (thus edges and vertices below it are
    not returned). If not specified, it defaults to 1. The minimal
    possible value is 0.
  - **max** (number, *optional*): up to *max* length paths are traversed.
    If omitted, *max* defaults to *min*. Thus only the vertices and edges in
    the range of *min* are returned. *max* cannot be specified without *min*.
- `OUTBOUND|INBOUND|ANY`: follow outgoing, incoming, or edges pointing in either
  direction in the traversal. Note that this can't be replaced by a bind parameter.
- **startVertex** (string\|object): a vertex where the traversal originates from.
  This can be specified in the form of an ID string or in the form of a document
  with the `_id` attribute. All other values lead to a warning and an empty
  result. If the specified document does not exist, the result is empty as well
  and there is no warning.
- `GRAPH` **graphName** (string): the name identifying the named graph.
  Its vertex and edge collections are looked up. Note that the graph name
  is like a regular string, hence it must be enclosed by quote marks, like
  `GRAPH "graphName"`.
- `PRUNE` **expression** (AQL expression, *optional*):
  An expression, like in a `FILTER` statement, which is evaluated in every step of
  the traversal, as early as possible. The semantics of this expression are as follows:
  - If the expression evaluates to `false`, the traversal continues on the current path.
  - If the expression evaluates to `true`, the traversal does not continue on the
    current path. However, the paths up to this point are considered as a result
    (they might still be post-filtered or ignored due to depth constraints).
    For example, a traversal over the graph `(A) -> (B) -> (C)` starting at `A`
    and pruning on `B` results in `(A)` and `(A) -> (B)` being valid paths,
    whereas `(A) -> (B) -> (C)` is not returned because it gets pruned on `B`.

  You can only use a single `PRUNE` clause per `FOR` traversal operation, but
  the prune expression can contain an arbitrary number of conditions using `AND`
  and `OR` statements for complex expressions. You can use the variables emitted
  by the `FOR` operation in the prune expression, as well as all variables
  defined before the traversal.

  You can optionally assign the prune expression to a variable like
  `PRUNE var = <expr>` to use the evaluated result elsewhere in the query,
  typically in a `FILTER` expression.

  See [Pruning](#pruning) for details.
- `OPTIONS` **options** (object, *optional*): See the [traversal options](#traversal-options).

### Working with collection sets

The syntax for AQL graph traversals using collection sets is as follows
(square brackets denote optional parts and `|` denotes alternatives):

```aql
[WITH vertexCollection1[, vertexCollection2[, vertexCollectionN]]]
FOR vertex[, edge[, path]]
  IN [min[..max]]
  OUTBOUND|INBOUND|ANY startVertex
  edgeCollection1[, edgeCollection2[, edgeCollectionN]]
  [PRUNE [pruneVariable = ]pruneCondition]
  [OPTIONS options]
```

- `WITH`: Declaration of collections. Optional for single server instances, but
  required for [graph traversals in a cluster](#graph-traversals-in-a-cluster).
  Needs to be placed at the very beginning of the query.
  - **collections** (collection, *repeatable*): list of vertex collections that
    are involved in the traversal
- **edgeCollections** (collection, *repeatable*): One or more edge collections
  to use for the traversal (instead of using a named graph with `GRAPH graphName`).
  Vertex collections are determined by the edges in the edge collections.
  
  You can override the default traversal direction by setting `OUTBOUND`,
  `INBOUND`, or `ANY` before any of the edge collections.
  
  If the same edge collection is specified multiple times, it behaves as if it
  were specified only once. Specifying the same edge collection is only allowed
  when the collections do not have conflicting traversal directions.

  Views cannot be used as edge collections.
- See the [named graph variant](#working-with-named-graphs) for the remaining
  traversal parameters as well as the [traversal options](#traversal-options).
  The `edgeCollections` restriction option is redundant in this case.

### Traversal options

You can optionally specify the following options to modify the execution of a
graph traversal. If you specify unknown options, query warnings are raised.

#### `order`

Specify which traversal algorithm to use (string):
- `"bfs"` – the traversal is executed breadth-first. The results
  first contain all vertices at depth 1, then all vertices at depth 2 and so on.
- `"dfs"` (default) – the traversal is executed depth-first. It
  first returns all paths from *min* depth to *max* depth for one vertex at
  depth 1, then for the next vertex at depth 1 and so on.
- `"weighted"` - the traversal is a weighted traversal
  (introduced in v3.8.0). Paths are enumerated with increasing cost.
  Also see `weightAttribute` and `defaultWeight`. A returned path has an
  additional attribute `weight` containing the cost of the path after every
  step. The order of paths having the same cost is non-deterministic.
  Negative weights are not supported and abort the query with an error.

#### `bfs`

Deprecated, use `order: "bfs"` instead.

#### `uniqueVertices`

Ensure vertex uniqueness (string):

- `"path"` – it is guaranteed that there is no path returned with a duplicate vertex
- `"global"` – it is guaranteed that each vertex is visited at most once during
  the traversal, no matter how many paths lead from the start vertex to this one.
  If you start with a `min depth > 1` a vertex that was found before *min* depth
  might not be returned at all (it still might be part of a path).
  It is required to set `order: "bfs"` or `order: "weighted"` because with
  depth-first search the results would be unpredictable. **Note:**
  Using this configuration the result is not deterministic any more. If there
  are multiple paths from *startVertex* to *vertex*, one of those is picked.
  In case of a `weighted` traversal, the path with the lowest weight is
  picked, but in case of equal weights it is undefined which one is chosen.
- `"none"` (default) – no uniqueness check is applied on vertices

#### `uniqueEdges`

Ensure edge uniqueness (string):

- `"path"` (default) – it is guaranteed that there is no path returned with a
  duplicate edge
- `"none"` – no uniqueness check is applied on edges. **Note:**
  Using this configuration, the traversal follows edges in cycles.

#### `edgeCollections`

Restrict edge collections the traversal may visit (string\|array).

If omitted or an empty array is specified, then there are no restrictions.

- A string parameter is treated as the equivalent of an array with a single
  element.
- Each element of the array should be a string containing the name of an
  edge collection.

#### `vertexCollections`

Restrict vertex collections the traversal may visit (string\|array).

If omitted or an empty array is specified, then there are no restrictions.

- A string parameter is treated as the equivalent of an array with a single
  element.
- Each element of the array should be a string containing the name of a
  vertex collection.
- The starting vertex is always allowed, even if it does not belong to one
  of the collections specified by a restriction.

#### `parallelism`

Parallelize traversal execution (number).
    
If omitted or set to a value of `1`, the traversal execution is not parallelized.
If set to a value greater than `1`, then up to that many worker threads can be
used for concurrently executing the traversal. The value is capped by the number
of available cores on the target machine.

Parallelizing a traversal is normally useful when there are many inputs (start
vertices) that the nested traversal can work on concurrently. This is often the
case when a nested traversal is fed with several tens of thousands of start
vertices, which can then be distributed randomly to worker threads for parallel
execution.

#### `maxProjections`

Specifies the number of document attributes per `FOR` loop to be used as
projections (number). The default value is `5`.

The AQL optimizer automatically detects which document attributes you access in
traversal queries and optimizes the data loading. This optimization is 
beneficial if you have large documents but only access a few document attributes.
The `maxProjections` option lets you tune when to load individual attributes
versus the whole document.

#### `indexHint`

<small>Introduced in v3.12.1</small>

You can provide index hints for traversals to let the optimizer prefer
the [vertex-centric indexes](../../index-and-search/indexing/working-with-indexes/vertex-centric-indexes.md)
you specify over the regular edge index.

This is useful for cases where the selectively estimate of the edge index
is higher than the ones for suitable vertex-centric indexes (and thus they
aren't picked automatically) but the vertex-centric indexes are known to
perform better.

The `indexHint` option expects an object in the following format:

`{ "<edgeColl>": { "<direction>": { "<level>": <index> } } }`

- `<edgeColl>`: The name of an edge collection for which the index hint
  shall be applied. Collection names are case-sensitive.
- `<direction>`: The direction for which to apply the index hint.
  Valid values are `inbound` and `outbound`, in lowercase.
  You can specify indexes for both directions.
- `<level>`: The level/depth for which the index should be applied. Valid
  values are the string `base` (to define the default index for all levels)
  and any stringified integer values greater or equal to zero.
  You can specify multiple levels.
- `<index>`: The name of an index as a string, or multiple index names as
  a list of strings in the order of preference. The optimizer uses the first
  suitable index.

Because collection names and levels/depths are used as object keys,
enclose them in quotes to avoid query parse errors.

Example:

```aql
FOR v, e, p IN 1..4 OUTBOUND startVertex edgeCollection
OPTIONS {
  indexHint: {
    "edgeCollection": {
      "outbound": {
        "base": ["edge"],
        "1": "myIndex1",
        "2": ["myIndex2", "myIndex1"],
        "3": "myIndex3",
      }
    }
  }
}
FILTER p.edges[1].foo == "bar" AND
        p.edges[2].foo == "bar" AND
        p.edges[2].baz == "qux"
```

Index hints for levels other than `base` are only considered if the
traversal actually uses a specific filter condition for the specified level.
In the above example, this is true for levels 1 and 2, but not for level 3.
Consequently, the index hint for level 3 is ignored here.

An expression like `FILTER p.edges[*].foo ALL == "bar"` cannot utilize the
indexes you specify for individual levels (level 1, level 2, etc.) but uses
the index for the `base` level.

The vertex-centric indexes you specify are only used if they are eligible
and the index hints for traversals cannot be forced.

#### `weightAttribute`

Specifies the name of an attribute that is used to look up the weight of an edge
(string).

If no attribute is specified or if it is not present in the edge document then
the `defaultWeight` is used.

The attribute value must not be negative.

{{< info >}}
Weighted traversals do not support negative weights. If a document
attribute (as specified by `weightAttribute`) with a negative value is
encountered during traversal, the query is aborted with an error.
{{< /info >}}

#### `defaultWeight`

Specifies the default weight of an edge (number). The default value is `1`.

The value must not be negative.

{{< info >}}
Weighted traversals do not support negative weights. If `defaultWeight` is set
to a negative number, then the query is aborted with an error.
{{< /info >}}

#### `useCache`

<small>Introduced in: v3.12.2</small>

Whether to use the in-memory cache for edges. The default is `true`.

You can set this option to `false` to not make a large graph operation pollute
the edge cache.

### Traversing in mixed directions

For traversals with a list of edge collections you can optionally specify the
direction for some of the edge collections. Say for example you have three edge
collections *edges1*, *edges2* and *edges3*, where in *edges2* the direction has
no relevance but in *edges1* and *edges3* the direction should be taken into account.
In this case you can use `OUTBOUND` as general traversal direction and `ANY`
specifically for *edges2* as follows:

```aql
FOR vertex IN OUTBOUND
  startVertex
  edges1, ANY edges2, edges3
```

All collections in the list that do not specify their own direction use the
direction defined after `IN`. This allows to use a different direction for each
collection in your traversal.

### Graph traversals in a cluster

Due to the nature of graphs, edges may reference vertices from arbitrary
collections. Following the paths can thus involve documents from various
collections and it is not possible to predict which are visited in a
traversal. Which collections need to be loaded by the graph engine can only be
determined at run time.

Use the [`WITH` statement](../high-level-operations/with.md) to specify the collections you
expect to be involved. This is required for traversals using collection sets
in cluster deployments.

## Pruning

You can define stop conditions for graph traversals to return specific data and
to improve the query performance. This is called _pruning_ and works by checking
conditions during the traversal as opposed to filtering the results afterwards
(post-filtering). This reduces the amount of data to be checked by stopping the
traversal down specific paths early.

{{< youtube id="4LVeeC0ciCQ" >}}

You can specify one `PRUNE` expression per graph traversal, but it can contain
an arbitrary number of conditions. You can use the vertex, edge, and path
variables emitted by the traversal in a prune expression, as well as all other
variables defined before the `FOR` operation. Note that `PRUNE` is an optional
clause of the `FOR` operation and that the `OPTIONS` clause needs to be placed
after `PRUNE`.

```aql
---
name: GRAPHTRAV_graphPruneExample1
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 0..10 OUTBOUND "places/Toronto" GRAPH "kShortestPathsGraph"
  PRUNE v.label == "Edmonton"
  OPTIONS { uniqueVertices: "path" }
  RETURN CONCAT_SEPARATOR(" -- ", p.vertices[*].label)
```

The above example shows a graph traversal using a
[train station and connections dataset](../../graphs/example-graphs.md#k-shortest-paths-graph):

![Train Connection Map](../../../images/train_map.png)

The traversal starts at **Toronto** (bottom left), the traversal depth is
limited to 10, and every station is only visited once. The traversal could
continue up to **Vancouver** (bottom right) at depth 5, but it is stopped early
on this path (the only path in this example) at **Edmonton** because of the
prune expression.

The traversal along paths is stopped as soon as the prune expression evaluates
to `true` for a given path. The current depth is still included in the result,
however. This can be seen in the query result of the example which includes the
Edmonton vertex at which it stopped.

The following example starts a traversal at **London** (middle right), with a
depth between 2 and 3, and every station is only visited once. The station names
as well as the travel times are returned:

```aql
---
name: GRAPHTRAV_graphPruneExample2
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  OPTIONS { uniqueVertices: "path" }
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

The same example with an added prune expression, with vertex and edge conditions:

```aql
---
name: GRAPHTRAV_graphPruneExample3
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  PRUNE v.label == "Carlisle" OR e.travelTime > 3
  OPTIONS { uniqueVertices: "path" }
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

If either the **Carlisle** vertex or an edge with a travel time of over three
hours is encountered, the subsequent paths are pruned. In the example, this
removes the train connections to **Birmingham**, **Glasgow**, and **York**,
which come after **Carlisle**, as well as the connections to and via
**Edinburgh** because of the four hour duration for the section from **York**
to **Edinburgh**.

If your graph is comprised of multiple vertex or edge collections, you can
also prune as soon as you reach a certain collection, using a condition like
`PRUNE IS_SAME_COLLECTION("stopCollection", v)`.

If you want to only return the results of the depth at which the traversal
stopped due to the prune expression, you can use a `FILTER` in addition. You can
assign the evaluated result of a prune expression to a variable
(`PRUNE var = <expr>`) and use it for filtering:

```aql
---
name: GRAPHTRAV_graphPruneExample4
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  PRUNE cond = v.label == "Carlisle" OR e.travelTime > 3
  OPTIONS { uniqueVertices: "path" }
  FILTER cond
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

Only paths that end at **Carlisle** or with the last edge having a travel time
of over three hours are returned. This excludes the connection to **Cologne**
from the results compared to the previous query.

If you want to exclude the depth at which the prune expression stopped the
traversal, you can assign the expression to a variable and use its negated value
in a `FILTER`:

```aql
---
name: GRAPHTRAV_graphPruneExample5
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  PRUNE cond = v.label == "Carlisle" OR e.travelTime > 3
  OPTIONS { uniqueVertices: "path" }
  FILTER NOT cond
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

This only returns the connection to **Cologne**, which is the opposite of the
previous example.

You may combine the prune variable with arbitrary other conditions in a `FILTER`
operation. For example, you can remove results where the last edge has as lower
travel time than the second to last edge of the path:

```aql
---
name: GRAPHTRAV_graphPruneExample6
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 2..5 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  PRUNE cond = v.label == "Carlisle" OR e.travelTime > 3
  OPTIONS { uniqueVertices: "path" }
  FILTER cond AND p.edges[-1].travelTime >= p.edges[-2].travelTime
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

{{< info >}}
The prune expression is **evaluated at every step of the traversal**. This
includes any traversal depths below the specified minimum depth, despite not
becoming part of the result. It also includes depth 0, which is the start vertex
and a `null` edge.

If you add prune conditions using the edge variable, make sure to account for
the edge at depth 0 being `null`, as it may accidentally stop the traversal
immediately. This may not be apparent due to the depth constraints.
{{< /info >}}

The following examples shows a graph traversal starting at **London**, with a
traversal depth between 2 and 3, and every station is only visited once:

```aql
---
name: GRAPHTRAV_graphPruneExample7
description: ''
dataset: kShortestPathsGraph
---
FOR v, e, p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  OPTIONS { uniqueVertices: "path" }
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

If you add prune conditions to stop the traversal if the station is **Glasgow**
or the travel time less than some number, no results are turned. This is even the
case for a value of `2.5`, for which two paths exist that fulfill the criterion
– to **Cologne** and **Carlisle**:

```aql
---
name: GRAPHTRAV_graphPruneExample8
description: ''
dataset: kShortestPathsGraph
---
FOR v,e,p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  PRUNE v.label == "Glasgow" OR e.travelTime < 2.5
  OPTIONS { uniqueVertices: "path" }
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

The problem is that `null`, `false`, and `true` are all less than any number (`< 2.5`)
because of AQL's [Type and value order](../fundamentals/type-and-value-order.md), and
because the edge at depth 0 is always `null`. The prune condition is accidentally
fulfilled at the start vertex, stopping the traversal too early. This similarly
happens if you check an edge attribute for inequality (`!=`) and compare it to
string, for instance, which evaluates to `true` for the `null` value.

The depth at which a traversal is stopped by pruning is considered as a result,
but in the above example, the minimum depth of `2` filters the start vertex out.
If you lower the minimum depth to `0`, you get **London** as the sole result.
This confirms that the traversal stopped at the start vertex.

To avoid this problem, exclude the `null` value. For example, you can use
`e.travelTime > 0 AND e.travelTime < 2.5`, but more generic solutions are to
exclude depth 0 from the check (`LENGTH(p.edges) > 0`) or to simply ignore the
`null` edge (`e != null`):

```aql
---
name: GRAPHTRAV_graphPruneExample9
description: ''
dataset: kShortestPathsGraph
---
FOR v,e,p IN 2..3 OUTBOUND "places/London" GRAPH "kShortestPathsGraph"
  PRUNE v.label == "Glasgow" OR (e != null AND e.travelTime < 2.5)
  OPTIONS { uniqueVertices: "path" }
  RETURN CONCAT_SEPARATOR(" -- ", INTERLEAVE(p.vertices[*].label, p.edges[*].travelTime))
```

{{< warning >}}
You can use AQL functions in prune expressions but only those that can be
executed on DB-Servers, regardless of your deployment mode. The following
functions cannot be used in the expression:
- `CALL()`
- `APPLY()`
- `DOCUMENT()`
- `V8()`
- `SCHEMA_GET()`
- `SCHEMA_VALIDATE()`
- `VERSION()`
- `COLLECTIONS()`
- `CURRENT_USER()`
- `CURRENT_DATABASE()`
- `COLLECTION_COUNT()`
- `NEAR()`
- `WITHIN()`
- `WITHIN_RECTANGLE()`
- `FULLTEXT()`
- [User-defined functions (UDFs)](../user-defined-functions.md)
{{< /warning >}}

## Using filters

All three variables emitted by the traversals might as well be used in filter
statements. For some of these filter statements the optimizer can detect that it
is possible to prune paths of traversals earlier, hence filtered results are
not emitted to the variables in the first place. This may significantly
improve the performance of your query. Whenever a filter is not fulfilled,
the complete set of `vertex`, `edge` and `path` is skipped. All paths
with a length greater than the `max` depth are never computed.

Filter conditions that are `AND`-combined can be optimized, but `OR`-combined
conditions cannot.

### Filtering on paths

Filtering on paths allows for the second most powerful filtering and may have the
second highest impact on performance. Using the path variable you can filter on
specific iteration depths. You can filter for absolute positions in the path
by specifying a positive number (which then qualifies for the optimizations),
or relative positions to the end of the path by specifying a negative number.

#### Filtering edges on the path

This example traversal filters all paths where the start edge (index 0) has the
attribute `theTruth` equal to `true`. The resulting paths are up to 5 items long:

```aql
---
name: GRAPHTRAV_graphFilterEdges
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..5 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[0].theTruth == true
    RETURN { vertices: p.vertices[*]._key, edges: p.edges[*].label }
```

#### Filtering vertices on the path

Similar to filtering the edges on the path, you can also filter the vertices:

```aql
---
name: GRAPHTRAV_graphFilterVertices
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..5 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.vertices[1]._key == "G"
    RETURN { vertices: p.vertices[*]._key, edges: p.edges[*].label }
```

#### Combining several filters

You can combine filters in any way you like:

```aql
---
name: GRAPHTRAV_graphFilterCombine
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..5 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[0].theTruth == true
       AND p.edges[1].theFalse == false
    FILTER p.vertices[1]._key == "G"
    RETURN { vertices: p.vertices[*]._key, edges: p.edges[*].label }
```

The query filters all paths where the first edge has the attribute
`theTruth` equal to `true`, the first vertex is `"G"` and the second edge has
the attribute `theFalse` equal to `false`. The resulting paths are up to
5 items long.

**Note**: Despite the `min` depth of 1, this only returns results of
depth 2. This is because for all results in depth 1, the second edge does not
exist and hence cannot fulfill the condition here.

#### Filter on the entire path

With the help of array comparison operators filters can also be defined
on the entire path, like `ALL` edges should have `theTruth == true`:

```aql
---
name: GRAPHTRAV_graphFilterEntirePath
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..5 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[*].theTruth ALL == true
    RETURN { vertices: p.vertices[*]._key, edges: p.edges[*].label }
```

Or `NONE` of the edges should have `theTruth == true`:

```aql
---
name: GRAPHTRAV_graphFilterPathEdges
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..5 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[*].theTruth NONE == true
    RETURN { vertices: p.vertices[*]._key, edges: p.edges[*].label }
```

Both examples above are recognized by the optimizer and can potentially use other indexes
than the edge index.

It is also possible to define that at least one edge on the path has to fulfill the condition:

```aql
---
name: GRAPHTRAV_graphFilterPathAnyEdge
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..5 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[*].theTruth ANY == true
    RETURN { vertices: p.vertices[*]._key, edges: p.edges[*].label }
```

It is guaranteed that at least one, but potentially more edges fulfill the condition.
All of the above filters can be defined on vertices in the exact same way.

### Filtering on the path vs. filtering on vertices or edges

Filtering on the path influences the Iteration on your graph. If certain conditions 
aren't met, the traversal may stop continuing along this path.

In contrast filters on vertex or edge only express whether you're interested in the actual value of these
documents. Thus, it influences the list of returned documents (if you return v or e) similar 
as specifying a non-null `min` value. If you specify a min value of 2, the traversal over the first
two nodes of these paths has to be executed - you just won't see them in your result array. 

Similar are filters on vertices or edges - the traverser has to walk along these nodes, since 
you may be interested in documents further down the path.

### Examples

Create a simple symmetric traversal demonstration graph:

![traversal graph](../../../images/traversal_graph.png)

```js
---
name: GRAPHTRAV_01_create_graph
description: ''
---
~addIgnoreCollection("circles");
~addIgnoreCollection("edges");
var examples = require("@arangodb/graph-examples/example-graph");
var graph = examples.loadGraph("traversalGraph");
db.circles.toArray();
db.edges.toArray();
print("once you don't need them anymore, clean them up:");
examples.dropGraph("traversalGraph");
```

To get started we select the full graph. For better overview we only return
the vertex IDs:

```aql
---
name: GRAPHTRAV_02_traverse_all_a
description: ''
dataset: traversalGraph
---
FOR v IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
  RETURN v._key
```

```aql
---
name: GRAPHTRAV_02_traverse_all_b
description: ''
dataset: traversalGraph
---
FOR v IN 1..3 OUTBOUND 'circles/A' edges RETURN v._key
```

We can nicely see that it is heading for the first outer vertex, then goes back to
the branch to descend into the next tree. After that it returns to our start node,
to descend again. As we can see both queries return the same result, the first one
uses the named graph, the second uses the edge collections directly.

Now we only want the elements of a specific depth (min = max = 2), the ones that
are right behind the fork:

```aql
---
name: GRAPHTRAV_03_traverse_3a
description: ''
dataset: traversalGraph
---
FOR v IN 2..2 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
  RETURN v._key
```

```aql
---
name: GRAPHTRAV_03_traverse_3b
description: ''
dataset: traversalGraph
---
FOR v IN 2 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
  RETURN v._key
```

As you can see, we can express this in two ways: with or without the `max` depth
parameter.

### Filter examples

Now let's start to add some filters. We want to cut of the branch on the right
side of the graph, we may filter in two ways:

- we know the vertex at depth 1 has `_key` == `G`
- we know the `label` attribute of the edge connecting **A** to **G** is `right_foo`

```aql
---
name: GRAPHTRAV_04_traverse_4a
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.vertices[1]._key != 'G'
    RETURN v._key
```

```aql
---
name: GRAPHTRAV_04_traverse_4b
description: ''
dataset: traversalGraph
---
FOR v, e, p IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[0].label != 'right_foo'
    RETURN v._key
```

As we can see, all vertices behind **G** are skipped in both queries.
The first filters on the vertex `_key`, the second on an edge label.
Note again, as soon as a filter is not fulfilled for any of the three elements
`v`, `e` or `p`, the complete set of these is excluded from the result.

We also may combine several filters, for instance to filter out the right branch
(**G**), and the **E** branch:

```aql
---
name: GRAPHTRAV_05_traverse_5a
description: ''
dataset: traversalGraph
---
FOR v,e,p IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.vertices[1]._key != 'G'
    FILTER p.edges[1].label != 'left_blub'
    RETURN v._key
```

```aql
---
name: GRAPHTRAV_05_traverse_5b
description: ''
dataset: traversalGraph
---
FOR v,e,p IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.vertices[1]._key != 'G' AND p.edges[1].label != 'left_blub'
    RETURN v._key
```

As you can see, combining two `FILTER` statements with an `AND` has the same result.

## Comparing OUTBOUND / INBOUND / ANY

All our previous examples traversed the graph in `OUTBOUND` edge direction.
You may however want to also traverse in reverse direction (`INBOUND`) or
both (`ANY`). Since `circles/A` only has outbound edges, we start our queries
from `circles/E`:

```aql
---
name: GRAPHTRAV_06_traverse_6a
description: ''
dataset: traversalGraph
---
FOR v IN 1..3 OUTBOUND 'circles/E' GRAPH 'traversalGraph'
    RETURN v._key
```

```aql
---
name: GRAPHTRAV_06_traverse_6b
description: ''
dataset: traversalGraph
---
FOR v IN 1..3 INBOUND 'circles/E' GRAPH 'traversalGraph'
    RETURN v._key
```

```aql
---
name: GRAPHTRAV_06_traverse_6c
description: ''
dataset: traversalGraph
---
FOR v IN 1..3 ANY 'circles/E' GRAPH 'traversalGraph'
    RETURN v._key
```

The first traversal only walks in the forward (`OUTBOUND`) direction.
Therefore from **E** we only can see **F**. Walking in reverse direction
(`INBOUND`), we see the path to **A**: **B** → **A**.

Walking in forward and reverse direction (`ANY`) we can see a more diverse result.
First of all, we see the simple paths to **F** and **A**. However, these vertices
have edges in other directions and they are traversed.

**Note**: The traverser may use identical edges multiple times. For instance,
if it walks from **E** to **F**, it continues to walk from **F** to **E**
using the same edge once again. Due to this, we see duplicate nodes in the result.

Please note that the direction can't be passed in by a bind parameter.

## Use the AQL explainer for optimizations

Now let's have a look what the optimizer does behind the curtain and inspect
traversal queries using [the explainer](../execution-and-performance/query-optimization.md):

```aql
---
name: GRAPHTRAV_07_traverse_7
description: ''
dataset: traversalGraph
explain: true
---
FOR v,e,p IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    LET localScopeVar = RAND() > 0.5
    FILTER p.edges[0].theTruth != localScopeVar
    RETURN v._key
```

```aql
---
name: GRAPHTRAV_07_traverse_8
description: ''
dataset: traversalGraph
explain: true
---
FOR v,e,p IN 1..3 OUTBOUND 'circles/A' GRAPH 'traversalGraph'
    FILTER p.edges[0].label == 'right_foo'
    RETURN v._key
```

We now see two queries: In one we add a `localScopeVar` variable, which is outside
the scope of the traversal itself - it is not known inside of the traverser.
Therefore, this filter can only be executed after the traversal, which may be
undesired in large graphs. The second query on the other hand only operates on the
path, and therefore this condition can be used during the execution of the traversal.
Paths that are filtered out by this condition won't be processed at all.

And finally clean it up again:

```js
---
name: GRAPHTRAV_99_drop_graph
description: ''
---
~examples.loadGraph("traversalGraph");
var examples = require("@arangodb/graph-examples/example-graph");
examples.dropGraph("traversalGraph");
```

If this traversal is not powerful enough for your needs, like you cannot describe
your conditions as AQL filter statements, then you might want to have a look at
the [edge collection methods](../../develop/javascript-api/@arangodb/collection-object.md#edge-documents)
in the JavaScript API.

Also see how to [combine graph traversals](../examples-and-query-patterns/traversals.md).
