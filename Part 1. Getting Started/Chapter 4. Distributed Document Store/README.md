# Introduction

In this chapter, we dive into the internal, technical details to help you understand how your data is stored in a distributed system.

# Routing Document to a Shard

When you index a document, it is stored on a single primary shard

The process of storing a document on a shard determined by a simple formula:

```
shard = hash(routing) % number_of_primary_shards
```

- `routing`: an arbitrary string, which defaults to the document’s `_id` but can also be set to a custom value

- `routing` string is passed through a hashing function to generate a number, which is divided by the `number of primary shards` in the `index` to return the *remainder*

- The *remainder* will always be in the range `0` to `number_of_primary_shards - 1`, and gives us the number of the shard where a particular document lives.

- This explains why the number of primary shards can be set only when an index is created and never changed: if the number of primary shards ever changed in the future, all previous routing values would be invalid and documents would never be found.

- All document APIs (`get, index, delete, bulk, update, mget`) accept a `routing` parameter that can be used to customize the document-to- shard mapping

-  A custom routing value could be used to ensure that all related documents are stored on the same shard

# How Primary and Replica Shards Interact

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0401.png)

Figure 8. A cluster with three nodes and one index

Every node is fully capable of serving any request. Every node knows the location of every document in the cluster and so can forward requests directly to the required node. 

***In the following examples, we will send all of our requests to Node 1, which we will refer to as the coordinating node.***

# Creating, Indexing, and Deleting a Document

- `create, index, delete` requests are write operations so they must be  successfully completed on the primary shard before they can be copied to any associated replica shards

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0402.png)

Figure 9. Creating, indexing, or deleting a single document

Here is the sequence of steps necessary to successfully create, index, or delete a document on both the primary and any replica shards:

1.  The client sends a create, index, or delete request to `Node 1`.
2. The node uses the document’s `_id` to determine that the document belongs to shard `0`. It forwards the request to `Node 3`, where the primary copy of shard `0` is currently allocated.
3. `Node 3` executes the request on the primary shard. If it is successful, it forwards the request in parallel to the replica shards on `Node 1` and `Node 2`. Once all of the replica shards report success, `Node 3` reports success to the coordinating node, which reports success to the client.

There are a number of optional request parameters that allow you to influence this process, possibly increasing performance at the cost of data security:

- `consistency`: By default, the primary shard requires a `quorum` of shard copies (where a shard copy can be a primary or a replica shard) to be available before even attempting a write operation. This is to prevent writing data to the “wrong side” of a network partition. A quorum is defined as follows:

```
int( (primary + number_of_replicas) / 2 ) + 1
```

The allowed values for consistency are `one` (just the primary shard), `all` (the primary and all replicas), or the default `quorum` of shard copies.

Note that the `number_of_replicas` is the number of replicas specified in the index settings

- `timeout`: The time Elasticsearch wait for sufficient shard copies available, default is `1 mins`. If you need to, you can use the timeout parameter to make it abort sooner: `100` is 100 milliseconds, and `30s` is 30 seconds.


# Retrieving a Document

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0403.png)

Figure 10. Retrieving a single document

Here is the sequence of steps to retrieve a document from either a primary or replica shard:

1. The client sends a get request to `Node 1`.
2. The node uses the document’s `_id` to determine that the document belongs to shard `0`. Copies of shard `0` exist on all three nodes. On this occasion, it forwards the request to `Node 2`.
3. `Node 2` returns the document to `Node 1`, which returns the document to the client.

For read requests, the coordinating node will choose a different shard copy on every request in order to balance the load; it round-robins through all shard copies.

# Partial Updates to a Document

The `update` API combines the read and write patterns explained previously.

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0404.png)

Figure 11. Partial updates to a document

Here is the sequence of steps used to perform a partial update on a document:

1. The client sends an update request to `Node 1`.
2. It forwards the request to `Node 3`, where the primary shard is allocated.
3. `Node 3` retrieves the document from the primary shard, changes the JSON in the `_source` field, and tries to reindex the document on the primary shard. If the document has already been changed by another process, it retries step 3 up to `retry_on_conflict` times, before giving up.

If `Node 3` has managed to update the document successfully, it forwards the new version of the document in parallel to the replica shards on `Node 1` and `Node 2` to be reindexed. Once all replica shards report success, `Node 3` reports success to the coordinating node, which reports success to the client.

The update API also accepts the routing, replication, consistency, and timeout parameters 

# Multidocument Patterns

### `_mget`

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0405.png)

Figure 12. Retrieving multiple documents with `mget`

Here is the sequence of steps necessary to retrieve multiple documents with a single mget request:

1. The client sends an mget request to Node 1.
2. Node 1 builds a multi-get request per shard, and forwards these requests in parallel to the nodes hosting each required primary or replica shard. Once all replies have been received, Node 1 builds the response and returns it to the client.

A `routing` parameter can be set for each document in the docs array.

### `_bulk`

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0406.png)

Figure 13. Multiple document changes with `bulk`

The sequence of steps followed by the `bulk` API are as follows:

1. The client sends a `bulk` request to `Node 1`.
2. `Node 1` builds a bulk request per shard, and forwards these requests in parallel to the nodes hosting each involved primary shard.
3. The primary shard executes each action serially, one after another. As each action succeeds, the primary forwards the new document (or deletion) to its replica shards in parallel, and then moves on to the next action. Once all replica shards report success for all actions, the node reports success to the coordinating node, which collates the responses and returns them to the client.

The `bulk` API also accepts the `consistency` parameter at the top level for the whole `bulk` request, and the `routing` parameter in the metadata for each request.

#### Why does the bulk API require the funny format with the newline characters, instead of just sending the requests wrapped in a JSON array, like the mget API?

- Each document referenced in a `bulk` request may belong to a different primary shard, each of which may be allocated to any of the nodes in the cluster. This means that every action inside a `bulk` request needs to be forwarded to the correct shard on the correct node.

- If the individual requests were wrapped up in a JSON array and would work, it would need a lot of RAM to hold copies of essentially the same data, and would create many more data structures that the Java Virtual Machine (JVM) would have to spend time garbage collecting.
