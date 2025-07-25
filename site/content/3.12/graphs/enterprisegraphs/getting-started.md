---
title: Getting Started with EnterpriseGraphs
menuTitle: Getting Started
weight: 5
description: >-
  This chapter walks you through the first steps you need to follow to create an EnterpriseGraph
---
EnterpriseGraphs **cannot use existing collections**. When switching to
EnterpriseGraph from an existing dataset, you have to import the data into a
fresh EnterpriseGraph.

When creating an EnterpriseGraph, you cannot have different number of shards per
collection. To preserve the sharding pattern, the `_from` and `_to` attributes
of the edges cannot be modified. 
You can define any `_key` value on vertices, including existing ones.

## Migrating to EnterpriseGraphs

If you want to switch from General Graphs to EnterpriseGraphs, you can
bring data from existing collections using the command-line tools `arangoexport`
and `arangoimport`.

`arangoexport` allows you to export collections to formats like `JSON`, `JSONL`, or `CSV`.
For this particular case, it is recommended to export data to `JSONL` format.
Once the data is exported, you need to exclude
the `_key` values from edges. The `enterprise-graph` module does not allow
custom `_key` values on edges. This is necessary for the initial data replication
when using `arangoimport` because these values are immutable.

### Migration by Example

Let us assume you have a `general-graph` in ArangoDB
that you want to migrate over to be an `enterprise-graph` to benefit from
the sharding strategy. In this example, the graph has only two collections:
`old_vertices` which is a document collection and `old_edges` which is the
corresponding edge collection.

**Export `general-graph` data**

The first step is to export the raw data of those
collections using `arangoexport`:

```sh
arangoexport --type jsonl --collection  old_vertices --output-directory docOutput --overwrite true
arangoexport --type jsonl --collection  old_edges --output-directory docOutput --overwrite true
```

Note that the `JSONL` format type is being used in the migration process
as it is more flexible and can be used with larger datasets.
The `JSON` type is limited in amount of documents, as it cannot be parsed line
by line. The `CSV` and `TSV` formats are also fine,
but require to define the list of attributes to export. `JSONL` exports data
as is, and due to its line based layout, can be processed line by line and
therefore has no artificial restrictions on the data.

After this step, two files are generated in the `docOutput` folder, that
should look like this:

1. `docOutput/old_vertices.jsonl`:
   ```
   {"_key":"Alice","_id":"old_vertices/Alice","_rev":"_edwXFGm---","attribute1":"value1"}
   {"_key":"Bob","_id":"old_vertices/Bob","_rev":"_edwXFGm--_","attribute1":"value2"}
   {"_key":"Charly","_id":"old_vertices/Charly","_rev":"_edwXFGm--B","attribute1":"value3"}
   ```    

2. `docOutput/old_edges.jsonl`:
   ```
   {"_key":"121","_id":"old_edges/121","_from":"old_vertices/Bob","_to":"old_vertices/Charly","_rev":"_edwW20----","attribute2":"value2"}
   {"_key":"122","_id":"old_edges/122","_from":"old_vertices/Charly","_to":"old_vertices/Alice","_rev":"_edwW20G---","attribute2":"value3"}
   {"_key":"120","_id":"old_edges/120","_from":"old_vertices/Alice","_to":"old_vertices/Bob","_rev":"_edwW20C---","attribute2":"value1"}
   ```

**Create new Graph**

The next step is to set up an empty EnterpriseGraph and configure it 
according to your preferences.

{{< info >}}
You are free to change `numberOfShards`, `replicationFactor`, or even collection names
at this point.
{{< /info >}}

Please follow the instructions on how to create an EnterpriseGraph
[using the Web Interface](#create-an-enterprisegraph-using-the-web-interface)
or [using _arangosh_](#create-an-enterprisegraph-using-arangosh).

**Import data while keeping collection names**

This example describes a 1:1 migration while keeping the original graph intact
and just changing the sharding strategy.

The empty collections that are now in the target ArangoDB cluster, 
have to be filled with data.
All vertices can be imported without any change:

```sh
arangoimport --collection old_vertices --file docOutput/old_vertices.jsonl
```

On the edges, EnterpriseGraphs disallow storing the `_key` value, so this attribute
needs to be removed on import:

```
arangoimport --collection old_edges --file docOutput/old_edges.jsonl --remove-attribute "_key"
```

After this step, the graph has been migrated.

**Import data while changing collection names**

This example describes a scenario in which the collections names have changed,
assuming that you have renamed `old_vertices` to `vertices`.

For the vertex data this change is not relevant, the `_id` values are adjusted
automatically, so you can import the data again, and just target the new
collection name:

```sh
arangoimport --collection vertices --file docOutput/old_vertices.jsonl
```

For the edges you need to apply more changes, as they need to be rewired.
To make the change of vertex collection, you need to set
`--overwrite-collection-prefix` to `true`.

To migrate the graph and also change to new collection names, run the following
command:

```sh
arangoimport --collection edges --file docOutput/old_edges.jsonl --remove-attribute "_key" --from-collection-prefix "vertices" --to-collection-prefix "vertices" --overwrite-collection-prefix true
```

Note that:
- You have to remove the `_key` value as it is disallowed for EnterpriseGraphs.
- Because you have changed the name of the `_from` collection, you need
  to provide a `--from-collection-prefix`. The same is true for the `_to` collection,
  so you also need to provide a `--to-collection-prefix`.
- To make the actual name change to the vertex collection, you need to
  allow `--overwrite-collection-prefix`. If this option is not enabled, only values
  without a collection name prefix are changed. This is helpful if your data is not
  exported by ArangoDB in the first place.   

This mechanism does not provide the option to selectively replace
collection names. It only allows replacing all collection names on `_from` 
respectively `_to`.
This means that, even if you use different collections in `_from` and `_to`, 
their names are modified based on the prefix that is specified.  

Consider the following example where `_to` points to a vertex in a different collection,
`users_vertices/Bob`. When using `--to-collection-prefix "vertices"` to rename
the collections, all collection names on the `_to` side are renamed to
`vertices` as this transformation solely allows for the replacement of all
collection names within the edge attribute.

```json
{"_key":"121", "_from":"old_vertices/Bob", "_to":"old_vertices/Charly", ... }
{"_key":"122", "_from":"old_vertices/Charly", "_to":"old_vertices/Alice", ... }
{"_key":"120", "_from":"old_vertices/Alice", "_to":"users_vertices/Bob", ... }
```

## Collections in EnterpriseGraphs

In contrast to General Graphs, you **cannot use existing collections**.
When switching from an existing dataset, you have to import the data into a
fresh EnterpriseGraph.

The creation of an EnterpriseGraph graph requires the name of the graph and a
definition of its edges. All collections used within the creation process are
automatically created by the `enterprise-graph` module. Make sure to only use
non-existent collection names for both vertices and edges.

## Create an EnterpriseGraph using the web interface

The web interface (also called Web UI) allows you to easily create and manage
EnterpriseGraphs. To get started, follow the steps outlined below.

1. In the web interface, navigate to the **Graphs** section.
2. To add a new graph, click **Add Graph**.
3. In the **Create Graph** dialog that appears, select the
   **EnterpriseGraph** tab.
4. Fill in all required fields:
   - For **Name**, enter a name for the EnterpriseGraph.
   - For **Shards**, enter the number of parts to split the graph into.
   - Optional: For **Replication factor**, enter the total number of
     desired copies of the data in the cluster.
   - Optional: For **Write concern**, enter the total number of copies
     of the data in the cluster required for each write operation.
   - Optional: For **SatelliteCollections**, insert vertex collections
     that are being used in your edge definitions. These collections are
     then created as satellites, and thus replicated to all DB-Servers.
   - For **Edge definition**, insert a single non-existent name to define
     the relation of the graph. This automatically creates a new edge
     collection, which is displayed in the **Collections** section of the
     left sidebar menu.
     {{< tip >}}
     To define multiple relations, press the **Add relation** button.
     To remove a relation, press the **Remove relation** button.
     {{< /tip >}}
   - For **fromCollections**, insert a list of vertex collections
     that contain the start vertices of the relation.
   - For **toCollections**, insert a list of vertex collections that
     contain the end vertices of the relation.
   {{< tip >}}
   Insert only non-existent collection names. Collections are automatically
   created during the graph setup and are displayed in the
   **Collections** tab of the left sidebar menu.
   {{< /tip >}}
   - For **Orphan collections**, insert a list of vertex collections
     that are part of the graph but not used in any edge definition.
5. Click **Create**. 
6. Click the name or row of the newly created graph to open the Graph Viewer if
   you want to visually interact with the graph and manage the graph data.

![Create EnterpriseGraph](../../../images/graphs-create-enterprise-graph-dialog312.png)
   
## Create an EnterpriseGraph using *arangosh*

Compared to SmartGraphs, the option `isSmart: true` is required but the
`smartGraphAttribute` is forbidden. 

```js
---
name: enterpriseGraphCreateGraphHowTo1
description: ''
type: cluster
---
var graph_module = require("@arangodb/enterprise-graph");
var graph = graph_module._create("myGraph", [], [], {isSmart: true, numberOfShards: 9});
graph;
~graph_module._drop("myGraph");
```

### Add vertex collections

The **collections must not exist** when creating the EnterpriseGraph. The EnterpriseGraph
module creates them for you automatically to set up the sharding for all
these collections correctly. If you create collections via the EnterpriseGraph
module and remove them from the graph definition, then you may re-add them
without trouble however, as they have the correct sharding.

```js
---
name: enterpriseGraphCreateGraphHowTo2
description: ''
type: cluster
---
~var graph_module = require("@arangodb/enterprise-graph");
~var graph = graph_module._create("myGraph", [], [], {isSmart: true, numberOfShards: 9});
graph._addVertexCollection("shop");
graph._addVertexCollection("customer");
graph._addVertexCollection("pet");
graph = graph_module._graph("myGraph");
~graph_module._drop("myGraph", true);
```

### Define relations on the Graph

Adding edge collections works the same as with General Graphs, but again, the
collections are created by the EnterpriseGraph module to set up sharding correctly
so they must not exist when creating the EnterpriseGraph (unless they have the
correct sharding already).

```js
---
name: enterpriseGraphCreateGraphHowTo3
description: ''
type: cluster
---
~var graph_module = require("@arangodb/enterprise-graph");
~var graph = graph_module._create("myGraph", [], [], {isSmart: true, numberOfShards: 9});
~graph._addVertexCollection("shop");
~graph._addVertexCollection("customer");
~graph._addVertexCollection("pet");
var rel = graph_module._relation("isCustomer", ["shop"], ["customer"]);
graph._extendEdgeDefinitions(rel);
graph = graph_module._graph("myGraph");
~graph_module._drop("myGraph", true);
```

### Create an EnterpriseGraph using SatelliteCollections

When creating a collection, you can decide whether it's a SatelliteCollection
or not. For example, a vertex collection can be satellite as well. 
SatelliteCollections don't require sharding as the data is distributed
globally on all DB-Servers. The `smartGraphAttribute` is also not required.

In addition to the attributes you would set to create a EnterpriseGraph, there is an
additional attribute `satellites` you can optionally set. It needs to be an array of
one or more collection names. These names can be used in edge definitions
(relations) and these collections are created as SatelliteCollections.
However, all vertex collections on one side of the relation have to be of
the same type - either all satellite or all smart. This is because `_from`
and `_to` can have different types based on the sharding pattern.

In this example, both vertex collections are created as SatelliteCollections.

{{< info >}}
When providing a satellite collection that is not used in a relation,
it is not created. If you create the collection in a following
request, only then the option counts.
{{< /info >}}

```js
---
name: enterpriseGraphCreateGraphHowTo4
description: ''
type: cluster
---
var graph_module = require("@arangodb/enterprise-graph");
var rel = graph_module._relation("isCustomer", "shop", "customer")
var graph = graph_module._create("myGraph", [rel], [], {satellites: ["shop", "customer"], isSmart: true, numberOfShards: 9});
graph;
~graph_module._drop("myGraph", true);
```
