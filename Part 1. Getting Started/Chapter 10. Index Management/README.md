# Introduction

Fine-tuning the indexing and search process is very important in some particular use cases

In this chapter, we introduce the APIs for managing indices and type mappings, and the most important settings

# Creating an Index

The index is created with the default settings when you index a document to it. However, sometimes, we want to controll number of *primary shards*, *analyzer*, and type of fields before we index any data. 

To create the index manually, passing in any settings or type mappings in the request body, as follows:

```
PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

If you want to, you can prevent the automatic creation of indices by adding the following setting to the `config/elasticsearch.yml` file on each node:

```
action.auto_create_index: false
```

# Deleting an Index

To delete an index or multiple indices or even *all* indices, use the following requests:

```
DELETE /my_index

DELETE /index_one,index_two
DELETE /index_*

DELETE /_all
DELETE /*
```

If you want to eliminate the possibility of an accidental mass-deletion, you can set the following to`true` in your `elasticsearch.yml`:

```
action.destructive_requires_name: true
```

# Index Settings

Two of the most important settings are as follows:

- `number_of_shards`: The number of primary shards that an index should have, which defaults to `5`. This setting cannot be changed after index creation.
- `number_of_replicas`: The number of replica shards (copies) that each primary shard should have, which defaults to `1`. This setting can be changed at any time on a live index.


```
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
    }
}
```

When update `number_of_replicas`:

```
PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```

# Configuring Analyzers

The third important index setting is the `analysis` section, which is used to configure existing *analyzers* or to create new custom *analyzers* specific to your index.

In the following example, we create a new analyzer called the `es_std` analyzer, which based on the `standard` analyzer and uses the predefined list of Spanish stopwords:

```
PUT /spanish_docs
{
    "settings": {
        "analysis": {
            "analyzer": {
                "es_std": {
                    "type":      "standard",
                    "stopwords": "_spanish_"
                }
            }
        }
    }
}
```

The `es_std` analyzer is not global—it exists only in the `spanish_docs` index where we have defined it.

To test `es_std` analyzer: 

```
GET /spanish_docs/_analyze?analyzer=es_std
El veloz zorro marrón
```

The result must not contain stopword `El`

# Custom Analyzers

An `analyzer` is a wrapper that combines three functions into a single package, which are executed in sequence:

- Character filters
- Tokenizers
- Token filters

### Creating a Custom Analyzer

We can configure character filters, tokenizers, and token filters in their respective sections under `analysis`:

```
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ...custom character filters...},
            "tokenizer":   { ...custom tokenizers...},
            "filter":      { ...custom token filters...},
            "analyzer":    { ...custom analyzers...}
        }
    }
}
```

After creating the index, use the `analyze` API to test the new analyzer:

```
GET /my_index/_analyze?analyzer=my_analyzer
{
	"text": "test my analyzer"
}
```

The analyzer is not much use unless we tell Elasticsearch where to use it. We can apply it to a `string` field with a mapping such as the following:

```
PUT /my_index/_mapping/my_type
{
    "properties": {
        "title": {
            "type":      "string",
            "analyzer":  "my_analyzer"
        }
    }
}
```

# Types and Mappings

- A `type` in Elasticsearch represents a class of similar documents. A type consists of a *name* and a *mapping*
- The `mapping`, like a database schema, describes the fields or properties that documents of that type may have, the datatype of each field

### How Types Are Implemented

An index may have several types, and documents of any of these types may be stored in the same index.

The type name of each document is stored with the document in a metadata field called `_type`. When we search for documents of a particular type, Elasticsearch simply uses a filter on the `_type` field to restrict results to documents of that type.

### Avoiding Type Gotchas

Each Lucene index contains a single, flat schema for all fields. A particular field is either mapped as a `string`, or a `number`, but not both.
 
Types are just a mechanism added by Elasticsearch on top of Lucene (in the form of a metadata `_type` field), all types in Elasticsearch ultimately share the same mapping.

### Type Takeaways

Types are useful when you need to discriminate between different segments of a single collection. The overall "shape" of the data is identical (or nearly so) between the different segments.

Types are not as well suited for entirely different types of data. If your two types have mutually exclusive sets of fields, that means half your index is going to contain "empty" values (the fields will be sparse), which will eventually cause performance problems

# The Root Object

The uppermost level of a mapping is known as the *root object*. It may contain the following:

### Properties 

Properties is document fields has 3 important settings:

- `type`
- `index`
- `analyzer`

###  Metadata: `_source` Field

- Elasticsearch stores the JSON string representing the document body in the `_source` field
- The `_source` field is compressed before being written to disk
- That said, storing the `_source` field does use disk space
- Besides indexing the values of a field, you can also choose to store the original field value for later retrieval using `store`

### Metadata: `_all` Field

The `_all` field is useful during the exploratory phase of a new application, while you are still unsure about the final structure that your documents will have

If you decide that you no longer need the `_all` field, you can disable it with this mapping:

```
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "_all": { "enabled": false }
    }
}
```

Inclusion in the `_all` field can be controlled on a field-by-field basis by using the `include_in_all` setting, which defaults to `true`. Instead of disabling the `_all` field completely, disable `include_in_all` for all fields by default, and enable it only on the fields you choose:

```
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "include_in_all": false,
        "properties": {
            "title": {
                "type":           "string",
                "include_in_all": true
            },
            ...
        }
    }
}
```

`_all` field uses the default `analyzer` to analyze its values, regardless of which analyzer has been set on the fields where the values originate. And like any `string` field, you can configure which `analyzer` the `_all` field should use:

```
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "_all": { "analyzer": "whitespace" }
    }
}
```

### Metadata: Document Identity

There are four metadata fields associated with document identity:

- `_id`: The string ID of the document. Neither indexed nor stored
- `_type`: The type name of the document. The `_type` field is indexed but not stored
- `_index`: The index where the document lives. Neither indexed nor stored
- `_uid`: The `_type` and `_id` concatenated together as `type#id`. `_uid` is stored (can be retrieved) and indexed (searchable)

Although you can change the `index` and `store` settings for these fields, you almost never need to do so.

# Dynamic Mapping

Elasticsearch uses *dynamic mapping* to determine the datatype for the field and automatically adds the new field to the type mapping.

You can control this behavior with the `dynamic` setting, which accepts the following options:

- `true`: Add new fields dynamically—the default
- `false`: Ignore new fields
- `strict`: Throw an exception if an unknown field is encountered

The `dynamic` setting may be applied to the *root object* or to any field of type object. You could set dynamic to `strict` by default, but enable it just for a specific inner object:

```
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic":      "strict", 
            "properties": {
                "title":  { "type": "string"},
                "stash":  {
                    "type":     "object",
                    "dynamic":  true 
                }
            }
        }
    }
}
```

*Note: Setting `dynamic` to `false` doesn’t alter the contents of the `_source` field at all. The `_source` will still contain the whole JSON document that you indexed. However, any unknown fields will not be added to the mapping and will not be searchable*

# Customizing Dynamic Mapping

### date_detection

When Elasticsearch encounters a new string field, it checks to see if the string contains a recognizable date. If it looks like a date, the field is added as type `date`. Otherwise, it is added as type `string`.

Sometimes this behavior can lead to problems. Imagine that you index a document like this:

```
{ "note": "2014-01-01" }
```

Assuming that this is the first time that the note field has been seen, it will be added as a `date` field. But what if the next document looks like this:

```
{ "note": "Logged out" }
```

The field is already a date field and so this “malformed date” will cause an exception to be thrown.

Date detection can be turned off by setting `date_detection` to `false` on the root object:

```
PUT /my_index
{
    "mappings": {
        "my_type": {
            "date_detection": false
        }
    }
}
```

With this mapping in place, a string will always be a `string`. If you need a `date` field, you have to add it manually.

### dynamic_templates

With `dynamic_templates`, you can take complete control over the mapping that is generated for newly detected fields

Each template has a name, which you can use to describe what the template does, a mapping to specify the mapping that should be applied, and at least one parameter (such as match) to define which fields the template should apply to.

Templates are checked in order; the first template that matches is applied

Example: 

```
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic_templates": [
                { "es": {
                      "match":              "*_es", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "spanish"
                      }
                }},
                { "en": {
                      "match":              "*", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "english"
                      }
                }}
            ]
}}}
```

# Default Mapping

Often, all types in an index share similar fields and settings. It can be more convenient to specify these common settings in the `_default_` mapping, instead of having to repeat yourself every time you create a new type

The `_default_` mapping acts as a template for new types. All types created after the `_default_` mapping will include all of these default settings, unless explicitly overridden in the type mapping itself.

```
PUT /my_index
{
    "mappings": {
        "_default_": {
            "_all": { "enabled":  false }
        },
        "blog": {
            "_all": { "enabled":  true  }
        }
    }
}
```

# Reindexing Your Data

Although you can add new types to an index, or add new fields to a type, you can’t add new analyzers or make changes to existing fields

The simplest way to apply these changes to your existing data is to reindex: create a new index with the new settings and copy all of your documents from the old index to the new index.

To reindex all of the documents from the old index efficiently, use `scroll` to retrieve batches of documents from the old index, and the `bulk` API to push them into the new index.

# Index Aliases and Zero Downtime

An index *alias* is like a shortcut or symbolic link, which can point to one or more indices, and can be used in any API that expects an index name. Aliases give us an enormous amount of flexibility

There are two endpoints for managing aliases: `_alias` for single operations, and `_aliases` to perform multiple operations atomically.

To create and add new index (`my_index_v1`) to an alias (`my_index`):

```
PUT /my_index_v1 
PUT /my_index_v1/_alias/my_index
```

You can check which index the alias points to:

```
GET /*/_alias/my_index
```

Or which aliases point to the index:

```
GET /my_index_v1/_alias/*
```

Later, we decide that we want to change the mappings for a field in our index. We create `my_index_v2` with the new mappings:

```
PUT /my_index_v2
{
    "mappings": {
        "my_type": {
            "properties": {
                "tags": {
                    "type":   "string",
                    "index":  "not_analyzed"
                }
            }
        }
    }
}
```

Then we reindex our data from `my_index_v1` to `my_index_v2`. Once we are satisfied that our documents have been reindexed correctly, we switch our alias to point to the new index.

```
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```

***Be prepared**: use aliases instead of indices in your application. Then you will be able to reindex whenever you need to. Aliases are cheap and should be used liberally.*

