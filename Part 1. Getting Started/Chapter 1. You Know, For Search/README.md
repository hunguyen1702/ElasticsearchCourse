# Introduction

- Elasticsearch is an open-source search engine built on top of Apache Lucene™, a full-text search-engine library

- Elasticsearch is also written in Java and uses Lucene internally for all of its indexing and searching, but it aims to make full-text search easy by hiding the complexities of Lucene behind a simple, coherent, RESTful API.

## Characteristics

- A distributed real-time document store where every field is indexed and searchable

- A distributed search engine with real-time analytics

- Capable of scaling to hundreds of servers and petabytes of structured and unstructured data (clusters & nodes)

- It packages up all this functionality into a standalone server that your application can talk to via a simple RESTful API, using a web client from your favorite programming language, or even from the command line.

## Installation: [Instructions](https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html)

# Talking with Elasticsearch

## JavaAPI: 

- Two built-in client (port 9300)
    + `Node Client`: The node client joins a local cluster as a non data node. In other words, it doesn’t hold any data itself, but it knows what data lives on which node in the cluster, and can forward requests directly to the correct node.
    
    + `Transport client`: The lighter-weight transport client can be used to send requests to a remote cluster. It doesn’t join the cluster itself, but simply forwards requests to a node in the cluster.

## RESTful API \w JSON over HTTP

- All languages (except Java) can communicate with Elasticsearch over port 9200 using a RESTful API

- [ElasticsearchClient](https://www.elastic.co/guide/en/elasticsearch/client/index.html) 

- A Elasticsearch HTTP request consists:

```python
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

# Document Oriented

- Elasticsearch is document oriented, meaning that it stores entire objects or documents, indexes the contents of each document in order to make them searchable

- In Elasticsearch, you index, search, sort, and filter documents—not rows of columnar data

- Elasticsearch uses JavaScript Object Notation, or JSON, as the serialization format for documents

# Finding your feet

***Let’s Build an Employee Directory***

We happen to work for Megacorp, and as part of HR’s new "We love our drones!" initiative, we have been tasked with creating an employee directory. The directory is supposed to foster employer empathy and real-time, synergistic, dynamic collaboration, so it has a few business requirements:

- Enable data to contain multi value tags, numbers, and full text.
- Retrieve the full details of any employee.
- Allow structured search, such as finding employees over the age of 30.
- Allow simple full-text search and more-complex *phrase* searches.
- Return highlighted search snippets from the text in the matching documents.
- Enable management to build analytic dashboards over the data.

# Indexing Documents

- The act of storing data in Elasticsearch is called indexing, but before we can index a document, we need to decide where to store it.

- Relational databases add an index, such as a B-tree index, to specific columns in order to improve the speed of data retrieval. Elasticsearch and Lucene use a structure called an inverted index for exactly the same purpose. 

- By default, every field in a document is indexed (has an inverted index) and thus is searchable. A field without an inverted index is not searchable

- There was no need to perform any administrative tasks first, like creating an index or specifying the type of data that each field contains. We could just index a document directly. Elasticsearch ships with defaults for everything, so all the necessary administration tasks were taken care of in the background, using default values.

***So for our employee directory, we are going to do the following:***

- Index a document per employee, which contains all the details of a single employee.
- Each document will be of type `employee`.
- That type will live in the `megacorp` index.
- That index will reside within our Elasticsearch cluster.
- Those actions can be performed in a single command:

```python
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

Notice that the path /megacorp/employee/1 contains three pieces of information:

- `megacorp`: The index name

- `employee`: The type name

- `1`: The ID of this particular employee

***Run query in this section***

# Retrieving a Document

We simply execute an HTTP GET request and specify the address of the document—the `index`, `type`, and `ID`. Using those three pieces of information, we can return the original JSON document. And the response contains some metadata about the document, and original JSON document as the `_source` field

***Run query in this section***

# Search Lite

Using combination of the `index`, `type` and `_search` endpoint API instead of `ID`

***Run query in this section***

# Search with Query DSL

Elasticsearch provides a rich, flexible, query language called the `query DSL` **(domain-specific language)** , which allows us to build much more complicated, robust queries.

***Run query in this section***

# More-Complicated Searches

Let’s make the search a little more complicated. We still want to find all employees with a last name of Smith, but we want only employees who are older than 30. Our query will change a little to accommodate a filter, which allows us to execute structured searches efficiently

***Run query in this section***

# Full-Text Search

The searches so far have been simple: single names, filtered by age. Let’s try a more advanced, full-text search—a task that traditional databases would really struggle with.

By default, Elasticsearch sorts matching results by their **relevance** score, that is, by how well each document matches the query

***Run query in this section***

# Phrase Search

Finding individual words in a field is all well and good, but sometimes you want to match exact sequences of words or phrases

***Run query in this section***

# Highlighting Our Searches

Many applications like to highlight snippets of text from each search result so the user can see why the document matched the query. Retrieving highlighted fragments is easy in Elasticsearch.

***Run query in this section***

# Analytics

Finally, we come to our last business requirement: allow managers to run analytics over the employee directory. Elasticsearch has functionality called `aggregations`, which allow you to generate sophisticated analytics over your data. It is similar to `GROUP BY` in SQL, but much more powerful.

***Run query in this section***

# Distributed Nature

- The distributed aspect of Elasticsearch is largely transparent
- Elasticsearch tries hard to hide the complexity of distributed systems. Here are some of the operations happening automatically under the hood:

    + Partitioning your documents into different containers or shards, which can be stored on a single node or on multiple nodes
    + Balancing these shards across the nodes in your cluster to spread the indexing and search load
    + Duplicating each shard to provide redundant copies of your data, to prevent data loss in case of hardware failure
    + Routing requests from any node in the cluster to the nodes that hold the data you’re interested in
    + Seamlessly integrating new nodes as your cluster grows or redistributing shards to recover from node loss
