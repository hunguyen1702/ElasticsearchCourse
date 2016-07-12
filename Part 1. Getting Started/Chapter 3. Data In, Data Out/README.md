# Introduction

In this chapter, we present the APIs that we use to create, retrieve, update, and delete documents. For the moment, we don’t care about the data inside our documents or how to query them. All we care about is how to store our documents safely in Elasticsearch and how to get them back again.

# What Is a Document?

Distinguish between 2 terms:

- `Object`: An object is just a JSON object—similar to what is known as a hash, hashmap, dictionary, or associative array. Objects may contain other objects
- `Document`: refers to the top-level, or root object that is serialized into JSON and stored in Elasticsearch under a unique ID

# Document Metadata

- `_index`: Where the document lives

The `index` name must be lowercase, cannot begin with an underscore, and cannot contain commas

- `_type`: The class of object that the document represents

Elasticsearch exposes a feature called types which allows you to logically partition data inside of an index. Documents in different types may have different fields, but it is best if they are highly similar. 

- `_id`: The ID is a string that, when combined with the _index and _type, uniquely identifies a document in Elasticsearch.

- Other Metadata: types, fields...

# Indexing a Document

#### Using Our Own ID

Using `PUT` (“store this document at this URL”) combine with `_index`, `_type`, `_id`

#### Autogenerating IDs

Using `POST` (“store this document under this URL”) combine with `_index`, `_type`

# Retrieving a Document

- To get the document out of Elasticsearch, we use the same `_index`, `_type`, and `_id`, but the HTTP method changes to `GET`

- *Adding `pretty` to the query-string parameters for any request, as in the preceding example, causes Elasticsearch to pretty-print the JSON response to make it more readable*

### Retrieving Part of a Document

- By default, a `GET` request will return the whole document, as stored in the _source field. But perhaps all you are interested in some fields. Individual fields can be requested by using the `_source` parameter

- Or if you want just the _source field without any metadata, you can use the `_source` endpoint

# Checking Whether a Document Exists

If all you want to do is to check whether a document exists—you’re not interested in the content at all—then use the HEAD method instead of the GET method. HEAD requests don’t return a body, just HTTP headers 

# Updating a Whole Document

- Documents in Elasticsearch are immutable; we cannot change them. Instead, if we need to `update` an existing document, we `reindex` or `replace` it

- Steps:

1. Retrieve the JSON from the old document

2. Change it

3. Delete the old document (*Elasticsearch cleans up deleted documents in the background as you continue to index more data*)

4. Index a new document

# Creating a New Document

- To be sure that we are not overwriting the existing document

- 3 ways to create a document:

	- Using `POST /_index/_type/`
	- Using `PUT /_index/_type/_id?op_type=create`
	- Using `PUT /_index/_type/_id/_create`

# Deleting a Document

- The syntax for deleting a document follows the same pattern that we have seen already, but uses the `DELETE` method

- Deleting a document doesn’t immediately remove the document from disk; it just marks it as deleted

# Dealing with Conflicts

- Updating a document with the index API, we read the original document, make our changes, and then reindex the whole document in one go.

- The most recent indexing request wins: whichever document was indexed last is the one stored in Elasticsearch

- If somebody else had changed the document in the meantime, their changes would be lost

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0301.png)

In the database world, two approaches are commonly used to ensure that changes are not lost when making concurrent updates:

- Pessimistic concurrency control
	- Blocks access
	- In order
	- Prevent conflicts

- Optimistic concurrency control
	- Doesn’t block operations
	- Can be attempted
	- Update failed when data has been modified

# Optimistic Concurrency Control

- Every document has a `_version` number that is incremented whenever a document is changed. 

- Elasticsearch uses this `_version` number to ensure that changes are applied in the correct order.

- Version numbers must be integers greater than zero and less than about `9.2e+18`--a positive `long` value in Java

## Using Versions from an External System

Instead of checking that the current _version is the same as the one specified in the request, Elasticsearch checks that the current _version is less than the specified version

# Partial Updates to Documents

- The `update` API simply manages the same `retrieve-change-reindex` process that we have already described. 

- The difference is that this process happens within a shard, thus avoiding the network overhead of multiple requests. By reducing the time between the retrieve and reindex steps, we also reduce the likelihood of there being conflicting changes from other processes.

- Using Scripts to Make Partial Updates: Scripts can be used in the update API to change the contents of the `_source` field, which is referred to inside an update script as `ctx._source`

- Updating a Document That May Not Yet Exist: we can use the upsert parameter to specify the document that should be created if it doesn’t already exist

## Updates and Conflicts

- To avoid losing data, the update API retrieves the current `_version` of the document in the retrieve step, and passes that to the index request during the reindex step. If another process has changed the document between retrieve and reindex, then the `_version` number won’t match and the update request will fail.

- For many uses of partial update, it doesn’t matter that a document has been changed. For instance, if two processes are both incrementing the page-view counter, it doesn’t matter in which order it happens; if a conflict occurs, the only thing we need to do is reattempt the update.

- This can be done automatically by setting the `retry_on_conflict` parameter to the number of times that update should retry before failing; it defaults to 0

# Retrieving Multiple Documents

- Combining multiple requests into one avoids the network overhead of processing each request individually

-  Using the multi-get, or `mget` API

- The `mget` API expects a `docs` array, each element of which specifies the `_index`,` _type`, and `_id` metadata of the document you wish to retrieve

# Cheaper in Bulk

- In the same way that `mget` allows us to retrieve multiple documents at once, the bulk API allows us to make multiple create, index, update, or delete requests in a single step

- Format: 

```
{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
...
```

- Two important points to note:
	- Every line must end with a newline character (\n)
	- The JSON must not be pretty-printed.

