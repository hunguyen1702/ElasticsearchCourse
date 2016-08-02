# Introduction

By default, results are returned sorted by *relevance*—with the most relevant docs first. This chapter will explain what mean *relevance* is and how it is calculated and the `sort` parameter and how to use it.

# Sorting

In Elasticsearch, the *relevance score* is represented by the floating-point number returned in the search results as the `_score`, so the default sort order is `_score` descending.

Sometimes, though, you don’t have a meaningful relevance score. For example, when you use filter queries. Documents will be returned in effectively random order, and each document will have a score of zero.

### Sorting by Field Values

Example: 

```elasticsearch
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
```

 The `_score` and `max_score` are both null. Calculating the `_score` can be quite expensive, and usually its only purpose is for sorting.

If you want the `_score` to be calculated regardless, you can set the `track_scores` parameter to true.

### Multilevel Sorting

Example:

```elasticsearch
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```

Order is important. Results are sorted by the first criterion first. Only results whose first `sort` value is identical will then be sorted by the second criterion, and so on.

### Sorting on Multivalue Fields

When you trying to sort a bag of values (do not have any intrinsic order) fields, you might want to reduce those fields to a single value by using the `min`, `max`, `avg`, or `sum` *sort* modes (`mode` parameter)

# String Sorting and Multifields

Analyzed string fields are also multivalue fields, but sorting on them seldom gives you the results you want.

For example,  if you analyze a string like `fine old art`, it results in three terms. We probably want to sort alphabetically on the first term, then the second term, and so forth, but Elasticsearch doesn’t have this information at its disposal at sort time.

The naive approach to indexing the same string in two ways would be to include two separate fields in the document: one that is `analyzed` for searching, and one that is `not_analyzed` for sorting.

But storing the same string twice in the `_source` field is waste of space. We can use `fields` parameter to transform a simple mapping like:

```elasticsearch
"tweet": {
    "type":     "string",
    "analyzer": "english"
}
```

to:

```elasticsearch
"tweet": { 
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": { 
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
```

Now, we can use the `tweet` field for search and the `tweet.raw` field for sorting:

```elasticsearch
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    },
    "sort": "tweet.raw"
}
```

# What Is Relevance?

What we usually mean by *relevance* is the algorithm that we use to calculate how similar the contents of a full-text field are to a full-text query string.

The standard similarity algorithm used in Elasticsearch is known as *term frequency/inverse document frequency*, or *TF/IDF*

- `Term frequency`: How often does the term appear in the field? The more often, the more relevant.
- `Inverse document frequency`: How often does each term appear in the index? The more often, the less relevant.
- `Field-length norm`: How long is the field? The longer it is, the less likely it is that words in the field will be relevant.

Individual queries may combine the *TF/IDF* score with other factors such as the term proximity in `phrase` queries, or term similarity in `fuzzy` queries.

### Understanding the Score

When debugging a complex query, it can be difficult to understand exactly how a `_score` has been calculated by using `explain` parameter in query-string like we used to validate a query in last chapter

### Understanding Why a Document Matched

While the explain option adds an explanation for every result, you can use the explain API to understand why one particular document matched or, more important, why it didn’t match.

The path for the request is `/index/type/id/_explain` and the request body is the query we use to search

# Doc Values Intro

When you sort on a field, Elasticsearch needs access to the value of that field for every document that matches the query

- When searching, we need to be able to map a term to a list of documents.
- When sorting, we need to map a document to its terms. In other words, we need to “uninvert” the inverted index.

This “uninverted” structure is often called a “column-store” in other systems. Essentially, it stores all the values for a single field together in a single column of data, which makes it very efficient for operations like sorting.

In Elasticsearch, this column-store is known as `doc values`, and is enabled by default. `Doc values` are created at index-time: when a field is indexed, Elasticsearch adds the tokens to the inverted index for search. But it also extracts the terms and adds them to the columnar doc values.

Because doc values are serialized to disk, we can leverage the OS to help keep access fast. When the "working set" is smaller than the available memory on a node, the OS will naturally keep all the doc values hot in memory, leading to very fast access. When the "working set" is much larger than available memory, the OS will naturally start to page doc-values on/off disk without running into the dreaded OutOfMemory exception.

We’ll talk about doc values in much greater depth later. For now, all you need to know is that sorting (and some other operations) happen on a parallel data structure which is built at index-time.


