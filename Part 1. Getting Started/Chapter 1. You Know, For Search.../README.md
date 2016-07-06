# Introduction

- Elasticsearch is an open-source search engine built on top of Apache Lucene™, a full-text search-engine library

- Elasticsearch is also written in Java and uses Lucene internally for all of its indexing and searching, but it aims to make full-text search easy by hiding the complexities of Lucene behind a simple, coherent, RESTful API.

# Characteristics

- A distributed real-time document store where every field is indexed and searchable

- A distributed search engine with real-time analytics

- Capable of scaling to hundreds of servers and petabytes of structured and unstructured data (clusters & nodes)

- It packages up all this functionality into a standalone server that your application can talk to via a simple RESTful API, using a web client from your favorite programming language, or even from the command line.

# Installation: [Instructions](https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html)

# JavaAPI: 

- Two built-in client: `Node Client` and `Transport Client` (port: 9300)

- [ElasticsearchClient](https://www.elastic.co/guide/en/elasticsearch/client/index.html)

# RESTful API \w JSON over HTTP

- All languages (except Java) can communicate with Elasticsearch over port 9200 using a RESTful API
 
- A Elasticsearch HTTP request consists:

```python
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

# Document Oriented

- Elasticsearch is document oriented, meaning that it stores entire objects or documents, indexes the contents of each document in order to make them searchable

- In Elasticsearch, you index, search, sort, and filter documents—not rows of columnar data

- Elasticsearch uses JavaScript Object Notation, or JSON, as the serialization format for documents

# Inverted Index

Relational databases add an index, such as a B-tree index, to specific columns in order to improve the speed of data retrieval. Elasticsearch and Lucene use a structure called an inverted index for exactly the same purpose. By default, every field in a document is indexed (has an inverted index) and thus is searchable. A field without an inverted index is not searchable
