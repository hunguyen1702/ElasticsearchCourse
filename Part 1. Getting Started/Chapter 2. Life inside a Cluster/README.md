# Introduction

This chapter will explain commonly used terminology like cluster, node, and shard, the mechanics of how Elasticsearch scales out, and how it deals with hardware failure.

Elasticsearch is built to be always available, and to scale with your needs. Scale can come from buying bigger servers (vertical scale, or scaling up) or from buying more servers (horizontal scale, or scaling out).

Real scalability comes from horizontal scale—the ability to add more nodes to the cluster and to spread load and reliability between them.

Elasticsearch is distributed by nature: it knows how to manage multiple nodes to provide scale and high availability.

# An Empty Cluster

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0201.png)

*Figure 1. A cluster with one empty node*

- `Node`: A node is a running instance of Elasticsearch
- `Cluster`: a cluster consists of one or more nodes with the same cluster.name that are working together to share their data and workload. As nodes are added to or removed from the cluster, the cluster reorganizes itself to spread the data evenly.
- `Master Node`: One node in the cluster is elected to be the master node, which is in charge of managing cluster-wide changes like creating or deleting an index, or adding or removing a node from the cluster. Any node can become the master

# Cluster Health

Many statistics can be monitored in an Elasticsearch cluster, but the single most important one is `cluster health`, which reports a status of either `green`, `yellow`, or `red`. The status field provides an overall indication of how the cluster is functioning:

- `green`: All primary and replica shards are active.
- `yellow`: All primary shards are active, but not all replica shards are active.
- `red`: Not all primary shards are active.

# Add an Index

To add data to Elasticsearch, we need an index—a place to store related data. In reality, an `index` is just a logical namespace that points to one or more physical shards

- `Shard`: A shard is a low-level worker unit that holds just a slice of all the data in the index, and is a complete search engine in its own right. Our documents are stored and indexed in shards, but our applications don’t talk to them directly. Instead, they talk to an index.
- `Primary Shard`: Each document in your index belongs to a single primary shard, so the number of primary shards that you have determines the maximum amount of data that your index can hold.
- `Replica Shard`: A replica shard is just a copy of a primary shard. Replicas are used to provide redundant copies of your data to protect against hardware failure, and to serve read requests like searching or retrieving a document.
- The number of primary shards in an index is fixed at the time that an index is created, but the number of replica shards can be changed at any time.

***Run query in this section***

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0202.png)

*Figure 2, “A single-node cluster with an index”.*


# Add Failover

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0203.png)

*Figure 3. A two-node cluster—all primary and replica shards are allocated*

- Starting a Second Node: When you run a second node on the same machine, it automatically discovers and joins the cluster as long as it has the same `cluster.name` as the first node. However, for nodes running on different machines to join the same cluster, you need to configure a list of unicast hosts the nodes can contact to join the cluster.
- Any newly indexed document will first be stored on a primary shard, and then copied in parallel to the associated replica shard(s). This ensures that our document can be retrieved from a primary shard or from any of its replicas.

***Run query in this section***

# Scale Horizontally

If we start a third node, our cluster reorganizes itself to look like Figure 4

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0204.png)

*Figure 4. A three-node cluster—shards have been reallocated to spread the load*

One shard each from Node 1 and Node 2 have moved to the new Node 3, and we have two shards per node, instead of three. This means that the hardware resources (CPU, RAM, I/O) of each node are being shared among fewer shards, allowing each shard to perform better.

A shard is a fully fledged search engine in its own right, and is capable of using all of the resources of a single node.

***Run query in this section***

## Then Scale Some More

The number of primary shards is fixed at the moment an index is created. Effectively, that number defines the maximum amount of data that can be stored in the index. (The actual number depends on your data, your hardware and your use case.) However, read requests—searches or document retrieval—can be handled by a primary or a replica shard, so the more copies of data that you have, the more search throughput you can handle.

The number of replica shards can be changed dynamically on a live cluster, allowing us to scale up or down as demand requires

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0205.png)

*Figure 5. Increasing the number_of_replicas to 2*

***Run query in this section***

# Coping with Failure

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0206.png)

*Figure 6. Cluster after killing one node*

- If the `master node` was killed -> elected a new `master node`
- If any `Primary Shard` was killed -> promote their `Replica Shard` to `Primary Shard`
- Status from `green` to `yellow`

***Run query in this section***

# Conclusion

- Shards are how Elasticsearch distributes data around your cluster. Think of shards as containers for data. Documents are stored in shards, and shards are allocated to nodes in your cluster. As your cluster grows or shrinks, Elasticsearch will automatically migrate shards between nodes so that the cluster remains balanced.

- As users, we can talk to any node in the cluster, including the master node. An `index` is just a logical namespace that points to one or more physical shards
