# Introduction

What exactly is a shard and how does it work? In this chapter, we answer these questions:

- Why is search near *real-time*?
- Why are document `CRUD` (create-read-update-delete) operations *real-time*?
- How does Elasticsearch ensure that the changes you make are durable, that they won’t be lost if there is a power failure?
- Why does deleting documents not free up space immediately?
- What do the `refresh`, `flush`, and `optimize` APIs do, and when should you use them?

# Making Text Searchable

The data structure that best supports the *multiple-values-per-field* requirement is the *inverted index*

The *inverted index* contains a sorted list of all of the unique values, or *terms*, that occur in any document and, for each term, a list of all the documents that contain it.

```
Term  | Doc 1 | Doc 2 | Doc 3 | ...
------------------------------------
brown |   X   |       |  X    | ...
fox   |   X   |   X   |  X    | ...
quick |   X   |   X   |       | ...
the   |   X   |       |  X    | ...
```

### Immutability

The inverted index that is written to disk is *immutable*: it doesn’t change.

Of course, an immutable index has its downsides too. If you want to make new documents searchable, you have to rebuild the entire index. 

This places a significant limitation either on the amount of data that an index can contain, or the frequency with which the index can be updated.

# Dynamically Updatable Indices

How to make an inverted index updatable without losing the benefits of immutability?

The answer turned out to be: use more than one index.

Instead of rewriting the whole inverted index, add new supplementary indices to reflect more-recent changes. 

Each inverted index can be queried in turn—starting with the oldest—and the results combined.

Lucene introduced the concept of *per-segment search*. Notations:

- *segment*: invert index
- *index* (in Lucene): a collection of *segments* plus a *commit point*

A per-segment search works as follows:

1. New documents are collected in an in-memory indexing buffer

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1102.png)
	
2. Every so often, the buffer is commited:
	- A new segment—a supplementary inverted index—is written to disk.
	- A new commit point is written to disk, which includes the name of the new segment.
	- The disk is `fsync`’ed—all writes waiting in the filesystem cache are flushed to disk, to ensure that they have been physically written.

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1103.png)
	
3. The new segment is opened, making the documents it contains visible to search.
4. The in-memory buffer is cleared, and is ready to accept new documents.

When a query is issued, all known segments are queried in turn

### Deletes and Updates

Segments are immutable, so documents cannot be removed from older segments, nor can older segments be updated to reflect a newer version of a document

Every commit point includes a `.del` file that lists which documents in which segments have been deleted. When a document is “deleted”/"updated" it/it's-old-version is actually just marked as deleted in the `.del` file.

A document that has been marked as deleted can still match a query, but it is removed from the results list before the final query results are returned.


# Near Real-Time Search

With the development of *per-segment* search, the delay between indexing a document and making it visible to search dropped dramatically.

Commiting a new segment to disk requires an `fsync`. An `fsync` is costly; it cannot be performed every time a document is indexed without a big performance hit.

What was needed was a more lightweight way to make new documents visible to search, which meant removing `fsync` from the equation.

- Sitting between Elasticsearch and the disk is the filesystem cache.
- As before, documents in the in-memory indexing buffer are written to a new segment. 
- The new segment is written to the filesystem cache first—which is cheap—and only later is it flushed to disk—which is expensive.
- Once a file is in the cache, it can be opened and read, just like any other file.

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1104.png)

*A Lucene index with new documents in the in-memory buffer*

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1105.png)

*The buffer contents have been written to a segment, which is searchable, but is not yet commited*

### `refresh` API

In Elasticsearch, this lightweight process of writing and opening a new segment is called a refresh.

By default, every shard is refreshed automatically once every second

You can use `refresh` API to manually refresh the shards:

```
POST /_refresh 
POST /index/_refresh 
```

 You can change the frequency of refreshes on a per-index basis by setting the `refresh_interval`

```
PUT /my_logs
{
  "settings": {
    "refresh_interval": "30s" 
  }
}

# Disable automatic refresh
PUT /my_logs/_settings
{ "refresh_interval": -1 } 

# Update
PUT /my_logs/_settings
{ "refresh_interval": "1s" } 
```

# Making Changes Persistent

While we refresh once every second to achieve near real-time search, we still need to do full commits regularly to make sure that we can recover from failure. 

And, when the document changes that happen between commits. We don’t want to lose those either.

Elasticsearch added a *translog*, or transaction log, which records every operation in Elasticsearch as it happens. With the translog, the process now looks like this:

1. When a document is indexed, it is added to the in-memory buffer and appended to the translog

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1106.png)

2. Once every second, the shard is refreshed:
	- The docs in the in-memory buffer are written to a new segment, without an `fsync`.
	- The segment is opened to make it visible to search.
	- The in-memory buffer is cleared. 

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1107.png)

3. This process continues with more documents being added to the in-memory buffer and appended to the transaction log

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1108.png)

4. Every so often—such as when the translog is getting too big—the index is flushed; a new translog is created, and a full commit is performed:
	- Any docs in the in-memory buffer are written to a new segment.
	- The buffer is cleared.
	- A commit point is written to disk.
	- The filesystem cache is flushed with an `fsync`.
	- The old translog is deleted.

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1109.png)

### `flush` API

The action of performing a commit and truncating the translog is known in Elasticsearch as a *flush*. Shards are flushed automatically every 30 minutes, or when the translog becomes too big

The `flush` API can be used to perform a manual flush:

```
POST /blogs/_flush 

# Flush all indices and wait until all flushes have completed before returning
POST /_flush?wait_for_ongoing
```

The benefit of using `flush` is before restarting a node or closing an index.

But, you seldom need to issue a manual flush yourself; usually, automatic flushing is all that is required.

# Segment Merging

The automatic refresh process will create a new segment every second, so the number of segments is increases rapidly. Which leads to problems:

-  Each segment consumes file handles, memory, and CPU cycles
-  Every search request has to check every segment in turn; the more segments there are, the slower the search will be

Elasticsearch solves this problem by merging segments in the background: 

1. While indexing, the refresh process creates new segments and opens them for search.
2. The merge process selects a few segments of similar size and merges them into a new bigger segment in the background. This does not interrupt indexing and searching.

	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1110.png)

3. When the merge completes:
	- The new segment is flushed to disk.
	-A new commit point is written that includes the new segment and excludes the old, smaller segments.
	- The new segment is opened for search.
	- The old segments are deleted.
	
	![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_1111.png)

*The merging of big segments can use a lot of I/O and CPU, which can hurt search performance if left unchecked*

### `optimize` API

The `optimize` API is best described as the forced merge API. It forces a shard to be merged down to the number of segments specified in the `max_num_segments` parameter

```
POST /index/_optimize?max_num_segments=1 
```
