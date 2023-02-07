---
fileID: smart-and-satellite-graphs-single-server
title: SmartGraphs and SatelliteGraphs on a Single Server
weight: 680
description: >-
  ## General Idea
  Simulate a SmartGraph or a SatelliteGraph on a single server and then to port it to an ArangoDB
  Cluster
layout: default
---
The graphs are created in a single server instance and tested there. Internally, the graphs are General
Graphs supplemented by formal properties such as `isSmart`, which however play no role in the behavior of the graphs. The
same is true for vertex and edge collections: they have the corresponding properties, which are non-functional.

After a test phase such a graph can be dumped and then restored in a cluster instance. The graph itself and the vertex
and edge collections now obtain true SmartGraph or SatelliteGraph sharding properties as if they were created in the
cluster.

## The Procedure

On a single server, create [SmartGraphs](smartgraphs/graphs-smart-graphs-management) or [SatelliteGraphs](satellitegraphs/graphs-satellite-graphs-management)
graphs by using `arangosh` as usual. Then you can set all the cluster relevant properties of graphs and collections: `numberOfShards`, `isSmart`,
`isSatellite`, `replicationFactor`, `smartGraphAttribute`, `satellites` and `shardingStrategy`. 
After that you can [dump](../programs-tools/arangodump/programs-arangodump-examples) the graphs with `arangodump` as usual.

Now [restore](../programs-tools/arangorestore/programs-arangorestore-examples) the dumped data into a running instance of ArangoDB Cluster. As a result,
all cluster relevant properties are restored correctly and affect now the sharding and the performance.