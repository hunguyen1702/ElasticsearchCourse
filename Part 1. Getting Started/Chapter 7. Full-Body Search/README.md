# Introduction

**Request body `search`** not only handles the query itself, but also allows you to return *highlighted snippets* from your *results, aggregate analytics* across all results or subsets of results, and return *did-you-mean* suggestions, which will help guide your users to the best results quickly. (Use it instead of *Search Lite* query)

# Empty Search

The simplest form of the search API, the empty search, which returns all documents in all indices

```elasticsearch
GET /_search

GET /index1,index2,index*/type1,type2/_search

GET /_search
{
  "from": 30,
  "size": 10
}
```

# Query DSL

The query DSL is a flexible, expressive search language that Elasticsearch uses to expose most of the power of Lucene through a simple JSON interface.

To use the Query DSL, pass a query in the query parameter:

```elasticsearch
GET /_search
{
    "query": YOUR_QUERY_HERE
}
```

### Structure of a Query Clause

A query clause typically has this structure:

```
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}
```

If it references one particular field, it has this structure:

```
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
```

### Combining Multiple Clauses

Query clauses are simple building blocks that can be combined with each other to create complex queries. Clauses can be as follows:

- `Leaf clauses` (like the match clause) that are used to compare a field (or fields) to a query string.
- `Compound clauses` that are used to combine other query clauses. For instance, a `bool` clause allows you to combine other clauses that either `must` match, `must_not` match, or `should` match if possible
- They can also include non-scoring, filters for structured search

Example:

```elasticsearch
{
    "bool": {
        "must": { "match":   { "email": "business opportunity" }},
        "should": [
            { "match":       { "starred": true }},
            { "bool": {
                "must":      { "match": { "folder": "inbox" }},
                "must_not":  { "match": { "spam": true }}
            }}
        ],
        "minimum_should_match": 1
    }
}
```

# Queries and Filters

The DSL used by Elasticsearch has a single set of components called queries, which can be mixed and matched in endless combinations. This single set of components can be used in two contexts: 

- `Querying`
	- The query is said to be a "non-scoring" or "filtering" query
	- The answer is always a simple, binary `yes|no`.
- `Filtering`
	- The query becomes a "scoring" query. Similar to its non-scoring sibling, this determines if a document matches and how well the document matches.

### Performance Differences

- `Querying`
	- Calculate how relevant each document is
	- Not only find matching documents --> makes them heavier
	- Not cacheable
- `Filtering` 
	- Filtering queries are simple checks for set inclusion/exclusion
	- Fast
	- Can be cached in memory for faster access

*The goal of filtering is to reduce the number of documents that have to be examined by the scoring queries.*

### When to Use Which

- Use query clauses for full-text search or for any condition that should affect the relevance score
- Use filters for everything else.

# Most Important Queries

#### `match_all`

- The `match_all` query simply matches all documents
- This query is frequently used in combination with a filter

#### `match`

The `match` query should be the standard query that you reach for whenever you want to query for a full-text or exact value in almost any field:

- If you run a `match` query against a full-text field, it will analyze the query string 
- If you use it on a field containing an exact value then it will search for that exact value

#### `multi_match`

The `multi_match` query allows to run the same `match` query on multiple fields

#### `range`

The`range` query allows you to find numbers or dates that fall into a specified range. The operators that it accepts are as follows:

- gt: Greater than
- gte: Greater than or equal to
- lt: Less than
- lte: Less than or equal to

#### `term`

The `term` query is used to search by exact values or `not_analyzed` exact-value string fields

#### `terms`

The `terms` query is the same as the `term` query, but allows you to specify multiple values to match

#### `exists` and `missing`

The `exists` and `missing` queries are used to find documents in which the specified field either has one or more values (`exists`) or doesn’t have any values (`missing`)

# Combining queries together

To do that, you can use the `bool` query. This query combines multiple queries together in user-defined boolean combinations. This query accepts the following parameters:

- `must`: Clauses that **must** match for the document to be included.
- `must_not`: Clauses that **must not** match for the document to be included.
- `should`: If these clauses match, they increase the `_score`; otherwise, they have no effect.
- `filter`: Clauses that must match, but are run in non-scoring, filtering mode (not contribute to the score, simply include/exclude documents)

*Each sub-query clause will individually calculate a relevance score for the document. Once these scores are calculated, the bool query will merge the scores together and return a single score representing the total score of the boolean operation.*

### Adding a filtering query

Any query can be converted it into a non-scoring query. Simply move a query into the `filter` clause of a `bool` query and it automatically converts to a non-scoring filter.

### `constant_score` Query

You can use this instead of a `bool` that only has `filter` clauses. Performance will be identical, but it may aid in query simplicity/clarity.

# Validating Queries

Queries can become quite complex and, especially when combined with different analyzers and field mappings, can become a bit difficult to follow. The validate-query API can be used to check whether a query is valid.

```elasticsearch
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

### Understanding Errors

To find out why it is invalid, add the `explain` parameter to the query string

```elasticsearch
GET /gb/tweet/_validate/query?explain
{..}
```

### Understanding Queries

Using the explain parameter has the added advantage of returning a human-readable description of the (valid) query, which can be useful for understanding exactly how your query has been interpreted by Elasticsearch
