# Introduction

Search requires a more complicated execution model because we don’t know which documents will match the query: they could be on any shard in the cluster. A search request has to consult a copy of every shard in the index or indices we’re interested in to see if they have any matching documents.

But finding all matching documents is only half the story. Results from multiple shards must be combined into a single sorted list before the `search` API can return a “page” of results. For this reason, search is executed in a two-phase process called **query then fetch**.

# Query Phase

The query phase consists of the following three steps:

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0901.png)

*Figure 14. Query phase of distributed search*

1. The client sends a `search` request to `Node 3,` which creates an empty priority queue of size `from + size`.
2. `Node 3` forwards the `search` request to a primary or replica copy of every shard in the index. Each shard executes the query locally and adds the results into a local sorted priority queue of size `from + size`.
3. Each shard returns the **doc IDs and sort values of all the docs in its priority queue** to the coordinating node, `Node 3`, which merges these values into its own priority queue to produce a globally sorted list of results.

When a search request is sent to a node, that node becomes the coordinating node. It is the job of this node to broadcast the search request to all involved shards, and to gather their responses into a globally sorted result set that it can return to the client.

# Fetch Phase

The query phase identifies which documents satisfy the search request, but we still need to retrieve the documents themselves. This is the job of the fetch phase

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0902.png)

*Figure 15. Fetch phase of distributed search*

The distributed phase consists of the following steps:

1. The coordinating node identifies which documents need to be fetched and issues a multi `GET` request to the relevant shards.
2. Each shard loads the documents and *enriches* them, if required, and then returns the documents to the coordinating node.
3. Once all documents have been fetched, the coordinating node returns the results to the client.

# Search Options

A few optional query-string parameters can influence the search process.

#### `preference`

The `preference` parameter allows you to control which shards or nodes are used to handle the search request. It accepts values such as `_primary`, `_primary_first`, `_local, _only_node:xyz`, `_prefer_node:xyz`, and `_shards:2,3`, which are explained in detail later.

However, the most generally useful value is some **arbitrary string**

#### `timeout`

The time it takes to run a search request is the sum of the time it takes to process the slowest shard and the time it takes to merge responses. If one node is having trouble, it could slow down the response to all search requests.

The `timeout` parameter tells shards how long they are allowed to process data before returning a response to the coordinating node. If there was not enough time to process all data, results for this shard will be partial, even possibly empty.

#### `routing`

At index time to ensure that all related documents, such as the documents belonging to a single user, are stored on a single shard.

At search time, instead of searching on all the shards of an index, you can specify one or more routing values to limit the search to just those shards

```
GET /_search?routing=user_1,user2
```

#### `search_type`

The default search type is `query_then_fetch`. In some cases, you might want to explicitly set the search `_type` to `dfs_query_then_fetch` to improve the accuracy of relevance scoring

*The `dfs_query_then_fetch` search type has a prequery phase that fetches the term frequencies from all involved shards to calculate global term frequencies*

# Scroll

A `scroll` query is used to retrieve large numbers of documents from Elasticsearch efficiently, without paying the penalty of deep pagination.

```
GET /old_index/_search?scroll=1m
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}
```

`1m`: Keep the scroll window open for 1 minute
`_doc`: is the most efficient sort order.

The response to this request includes a `_scroll_id`

Now we can pass the `_scroll_id` to the `_search/scroll` endpoint to retrieve the next batch of results:

```
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
```

