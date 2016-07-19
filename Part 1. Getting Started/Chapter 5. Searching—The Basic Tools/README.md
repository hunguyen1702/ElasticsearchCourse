# Introduction

A search in Elasticsearch can be any of the following:
- A structured query on concrete fields
- A full-text query, which finds all documents matching the search keywords
- A combination of the two

To understand a search, you have to understand **3** subjects:
- Mapping
- Analysis
- Query DSL

To start this chapter, first run all the query in the `Instruction` path of `Query.txt` file

# The Empty Search

The most basic form of the search API is the empty search, which doesn’t specify any query but simply returns all documents in all indices in the cluster

In the result:

- `hits`: 
	- `total`: number of documents that matched query
	-  `hits`: contains matching documents—the results (`_index`, `_type`, `_id`, `_score`)
- `took`: how many milliseconds the entire search request took to execute
- `shards`: total number of shards that were involved in the query and, of them, how many were successful and how many failed
- `timeout`: The timed_out value tells us whether the query timed out. By default, search requests do not time out

# Multi-index, Multitype

By not limiting our search to a particular index or type, we have searched across all documents in the cluster. Elasticsearch forwarded the search request in parallel to a primary or replica of every shard in the cluster, gathered the results to select the overall top 10, and returned them to us.

When you search within a single index, Elasticsearch forwards the search request to a primary or replica of every shard in that index, and then gathers the results from each shard. Searching within multiple indices works in exactly the same way—there are just more shards involved.

# Pagination

To return a single “page” of results, passing the `from` and `size` parameters:

- `size`: Indicates the number of results that should be returned, defaults to 10
- `from`: Indicates the number of initial results that should be skipped, defaults to 0

# Search Lite

A “lite” query-string version that expects all its parameters to be passed in the query string

- A `+` prefix indicates conditions that must be satisfied for our query to match. 
- A `-` prefix would indicate conditions that must not match. 
- All conditions without a `+` or `-` are optional—the more that match, the more relevant the document.

#### The `_all` Field

When you index a document, Elasticsearch takes the string values of all of its fields and concatenates them into one big string, which it indexes as the special _all field

####  => Don't use `Lite queries`
