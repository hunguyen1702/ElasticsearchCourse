# Introduction

- Elasticsearch has dynamically generated a mapping for us, based on what it could guess about our field types
- You might expect that each of the core data types—strings, numbers, Booleans, and dates—might be indexed slightly differently. And this is true: there are slight differences.
- But by far the biggest difference is between fields that represent exact values (which can include string fields) and fields that represent full text.

# Exact Values Versus Full Text

- Exact values are exactly what they sound like. Examples are a date or a user ID, but can also include exact strings such as a username or an email address. Exact values are easy to query
- Full text, on the other hand, refers to textual data—usually written in some human language — like the text of a tweet or the body of an email.

# Inverted Index

- Designed to allow very fast full-text searches
- Consists of a list of all the unique words that appear in any document, and for each word, a list of the documents in which it appears

# Analysis & Analyzer

Analysis is a process that consists of the following:

- First, tokenizing a block of text into individual terms suitable for use in an inverted index,
- Then normalizing these terms into a standard form to improve their “searchability,” or recall

This job is performed by analyzers. An analyzer is really just a wrapper that combines three functions into a single package:

- Character filters: strip out `HTML`, convert `&` to `and`
- Tokenizer: split the text
- Token filters: change terms (lowercase), remove terms (remove stopwords), add terms (synonyms)

### Built-in Analyzers

However, Elasticsearch also ships with prepackaged analyzers that you can use directly: Standard analyzer, Simple analyzer, Whitespace analyzer, Language analyzers

### When Analyzers Are Used

- When we index a document, its full-text fields are analyzed into terms that are used to create the inverted index

- When we search on a full-text field, we need to pass the query string through the same analysis process, to ensure that we are searching for terms in the same form as those that exist in the index

### Testing Analyzers

 To better understand what is going on, you can use the analyze API to see how text is analyzed

### Specifying Analyzers

Perhaps you want to apply a different analyzer that suits the language your data is in. And sometimes you want a string field to be just a string field—to index the exact value that you pass in, without any analysis, such as a string user ID or an internal status field or tag.

To achieve this, we have to configure these fields manually by specifying the mapping.

# Mapping

- Each document in an index has a type. Every type has its own mapping, or schema definition
- A mapping defines the fields within a type, the datatype for each field, and how the field should be handled by Elasticsearch. 
- A mapping is also used to configure metadata associated with the type.

### Core Simple Field Types

Elasticsearch supports the following simple field types:

- String: `string`
- Whole number: `byte`, `short`, `integer`, `long`
- Floating-point: `float`, `double`
- Boolean: `boolean`
- Date: `date`

When you index a document that contains a new field—one previously not seen—Elasticsearch will use *dynamic* mapping to try to guess the field type from the basic datatypes available in JSON

### Viewing the Mapping

We can view the mapping that Elasticsearch has for one or more types in one or more indices by using the `/_mapping` endpoint.

### Customizing Field Mappings

Custom mappings allow you to do the following:

- Distinguish between full-text string fields and exact value string fields
- Use language-specific analyzers
- Optimize a field for partial matching
- Specify custom date formats
- And much more

The most important attribute of a field is the `type`

The two most important mapping attributes for string fields are `index` and `analyzer`

The index attribute controls how the string will be indexed. It can contain one of three values:

- `analyzed`: First analyze the string and then index it. In other words, index this field as full text.
- `not_analyzed`: Index this field, so it is searchable, but index the value exactly as specified. Do not analyze it.
- `no`: Don’t index this field at all. This field will not be searchable.

### Updating a Mapping

**Although you can add to an existing mapping, you can’t change existing field mappings. **

### Testing the Mapping

*You can use the analyze API to test the mapping for string fields by name. Compare the output of these two requests*

# Complex Core Field Types

Besides the simple scalar datatypes that we have mentioned, JSON also has null values, arrays, and objects, all of which are supported by Elasticsearch.

### Multivalue Fields

There is no special mapping required for arrays. Any field can contain zero, one, or more values, in the same way as a full-text field is analyzed to produce multiple terms

*Note: Arrays are indexed—made searchable—as multivalue fields, which are unordered. At search time, you can’t refer to “the first element” or “the last element.” Rather, think of an array as a bag of values.*

### Empty Fields

These three fields would all be considered to be empty, and would not be indexed:

```elasticsearch
"null_value":               null,
"empty_array":              [],
"array_with_null_value":    [ null ]
```

### Multilevel Objects

The last native JSON datatype that we need to discuss is the `object` — known in other languages as a hash, hashmap, dictionary or associative array.

#### Mapping for Inner Objects

Elasticsearch will detect new object fields dynamically and map them as type object, with each inner field listed under properties

Example:

```elasticsearch
{
  "gb": {
    "tweet": { 
      "properties": {
        "tweet":            { "type": "string" },
        "user": { 
          "type":             "object",
          "properties": {
            "id":           { "type": "string" },
            "gender":       { "type": "string" },
            "age":          { "type": "long"   },
            "name":   { 
              "type":         "object",
              "properties": {
                "full":     { "type": "string" },
                "first":    { "type": "string" },
                "last":     { "type": "string" }
              }
            }
          }
        }
      }
    }
  }
}
```

#### How Inner Objects are Indexed

In order for Elasticsearch to index inner objects usefully, it converts our document into something like this:

```elasticsearch
{
    "tweet":            [elasticsearch, flexible, very],
    "user.id":          [@johnsmith],
    "user.gender":      [male],
    "user.age":         [26],
    "user.name.full":   [john, smith],
    "user.name.first":  [john],
    "user.name.last":   [smith]
}
```

#### Arrays of Inner Objects

 Finally, consider how an array containing inner objects would be indexed. Let’s say we have a followers array that looks like this:
 
```elasticsearch
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
```

This document will be flattened as we described previously, but the result will look like this:

```elasticsearch
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}
```

*Note: The correlation between `{age: 35}` and `{name: Mary White}` has been lost as each multivalue field is just a bag of values, not an ordered array*
