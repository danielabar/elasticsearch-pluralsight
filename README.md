# Getting Started with Elasticsearch

> My course notes from learning Elasticsearch with [Pluralsight Course](https://app.pluralsight.com/library/courses/elasticsearch-for-dotnet-developers/table-of-contents)

## Concepts

Document is saved in _index_ (which is kind of like a Database in relational world).

Indexes are stored across multiple _shards_, which are logical ways of slicing the data into individual chunks so they can be stored across multiple servers.

Shards are stored in one or more servers, which are called _nodes_.

A collection of nodes is considered a _cluster_.

Clustering is what makes ES easy to scale horizontally - i.e. adding more servers to the cluster so the shares can be distributed to them, to help balance out the load.

## Installation

Needs at least jre7. Download install zip from site, unpack and run the startup bat or sh specified in instructions.

## Configuration

Open `config/elasticsearch.yml` in installation folder and set cluster and node name, for example:


```
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: esdemo
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node01
#
```

Start elastic search, for example on Windows:

```shell
cd bin
elasticsearch.bat
```

Use Postman or CURL to verify that [http://localhost:9200](http://localhost:9200) returns a response.


Also install Kibana, Marvel and Sense follow installation and configuration instructions at:
* [Kibana](https://www.elastic.co/downloads/kibana)
* [Marvel](https://www.elastic.co/downloads/marvel)
* [Sense](https://github.com/elastic/sense/)

Note that Kibana is a trial license, not free.

Sense is a Kibana plugin.

Sense is at [http://localhost:5601/app/sense](http://localhost:5601/app/sense)

Given host of: http://localhost:9200, some sample queries are:

```
GET _search
{
  "query": {
    "match_all": {}
  }
}

GET /
```

Remainder of this course will use Sense plugin.

## Schemas (Mappings)

Example comparing to a relational database:

```sql
-- Create the database
CREATE DATABASE my_blog DEFALT CHARACTER SET latin1 COLLATE latin1_swedish_ci;

-- Create blog posts table
CREATE TABLE IF NOT EXISTS post (
  id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  user_id int(10) NOT NULL,
  post_text varchar(255) DEFAULT NULL,
  post_date datetime NOT NULL,
  PRIMARY KEY (id)
) ENGINE=MyISAM DEFAULT CHARSET=UTF8 AUTO_INCREMENT=3;
```

To create an index with this as a schema (aka mapping) in ES, POST a mapping request as follows:

`POST http://localhost:9200/my_blog`

```json
{
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date"
        }
      }
    }
  }
}
```

`my_blog` will become the index name.

Instead of database "table", blog post is considered a "type", which has "properties".

Properties in ES are like columns in a relational database. Each has its own data type, similar to sql data types.

Notice the mapping above has no "id" field. This is because id's are treated specially in ES. By default, it will generate the id automatically.

To verify schema was created, check what indicies are present:

```
GET /_cat/indices
```

To verify mapping was created successfully:

```
GET /my_blog/_mapping
```

## Inserting documents

To insert a document:

```
POST /my_blog/post
{
  "post_date": "2015-08-20",
  "post_text": "This is a real blog post!",
  "user_id": 1
}
```

Insert another:
```
POST /my_blog/post
{
  "post_date": "2015-08-25",
  "post_text": "This is another real blog post!",
  "user_id": 2
}
```

## Basic search

To search ALL blog posts:

```
GET /my_blog/post/_search
```

Sample output for two docs inserted:

```json
{
  "took": 37,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_blog",
        "_type": "post",
        "_id": "AVImVnsdIRq3mAlBy4Fh",
        "_score": 1,
        "_source": {
          "post_date": "2015-08-25",
          "post_text": "This is another real blog post!",
          "user_id": 2
        }
      },
      {
        "_index": "my_blog",
        "_type": "post",
        "_id": "AVImVNA-IRq3mAlBy4Dn",
        "_score": 1,
        "_source": {
          "post_date": "2015-08-20",
          "post_text": "This is a real blog post!",
          "user_id": 1
        }
      }
    ]
  }
}
```
To query by id (as generated by ES):

```
GET /my_blog/post/AVImVNA-IRq3mAlBy4Dn
```

If you don't want the ES auto generated id's and instead want to use your own id system, insert as follows, notice the "/1" at end of POST url means create this document using id=1:

```
POST /my_blog/post/1
{
  "post_date": "2015-08-26",
  "post_text": "This post should have an _id of 1",
  "user_id": 2
}
```

## Data Types

ES comes with 5 core data types out of the box:

* String - most common, similar to varchar in relational database. Use it to store any collection of characters. Many options and settings for flexibility.
* Boolean - pretty basic, either true or false
* Number - many options and settings, can be byte (8 bit integer), short (16 bit integer) Float (single precision 32 bit floating point), double. These correspond to core types in Java.
* Date - lots of formatting options. Stored as UTC, default format is "Date Optional Time", which means ES will append time string to value that is inserted. Can specify multiple formats at once.
* Binary - for example images, stored as base64 string, not indexed by default.

### Data Type Options

Most data types support additional options and settings to tweak their usage in an index, for example, to specify that entries must have date in YYYY-MM-DD format:

```json
{
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```

If a document is inserted with `post_date` not in specified format, ES will return a 400 error.

### Create Index with Settings

First delete previous index we created:

```
DELETE /my_blog/
```

To create an index specifying number of shards as a setting (good practice to always specify num shards in multi-node cluster):

`POST http://localhost:9200/my_blog`

```json
{
  "settings": {
    "index": {
      "number_of_shards": 5
    }
  },
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```

## Index Options

Index options affect how data is stored and retrieved.

### Mappings \_source

By default, ES stores both original copies of data, and indexed version. This can lead a large index.

This behavior can be modified for a slimmed down index. To do so, set \_source to disabled in mapping, then for each field, specify which properties to store.


In the example below, the post_text and post_date fields will NOT be stored, only indexed. This means you can search on indexed versions of the fields, but not retrieve the original data.

```json
{
  "mappings": {
    "post": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "user_id": {
          "type": "integer",
          "store": true
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```

Now when searching for docs, none of the properties are returned. Need to specify fields:

```
GET /my_blog/post/AVInjopcIRq3mAlBy9iM?fields=user_id
```

```json
{
  "_index": "my_blog",
  "_type": "post",
  "_id": "AVInjopcIRq3mAlBy9iM",
  "_version": 1,
  "found": true,
  "fields": {
    "user_id": [
      1
    ]
  }
}
```

Only use source false in production AND where disk space is a concern.

### Mappings \_all

By default, ES maintains \_all field, which is concatenation of all the data in the type, "post" in this example. By leaving \_all enabled, you can search it and it will include every property value.

This feature can be disabled as shown in the example below:

```json
{
  "mappings": {
    "post": {
      "_all": {
        "enabled": false
      },
      "properties": {
        "user_id": {
          "type": "integer",
          "store": true
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```

 \_all can be useful for searches, but it will increase size of index to have it enabled.

### Indexes Routing

Purpose of routing is to direct ES to the shard that contains the data you're looking for. Can drastically affect query performance.

Example: One cluster with three nodes: Node 01, Node 02, Node 03 with three shards each. Now execute a search query:

```
GET http://localhost:9200/my_blog/post/_search?q=post_text:awesome
```

This search will run against every shard in the cluster and check against all the blog posts.

Routing gives ES a hint as to which shard to search. To use it, specify in mapping, for example:

```json
{
  "mappings": {
    "post": {
      "_routing": {
        "required": true,
        "path": "user_id"
      },
      "properties": {
        "user_id": {
          "type": "integer",
          "store": true
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```

Setting "required" to true means queries MUST use the routing parameter when searching.

"path" specifies field we want to route with, in this case, "user_id".

Telling ES to route by "user_id" means ES will divide the blog posts across shards NOT arbitrarily, but by user_id. This will cause each users data to exist on one shard, thus narrowing down search parameters.

A query with routing:

```
GET http://localhost:9200/my_blog/post/_search?routing=2?post_text:awesome
```

Specifying "routing=2" in query will cause it to be routed directly to the shard with all of the data owned by user_id 2.

### Indexes Aliases

For example, keeping an index of application logs on a per day basis. Each index is named "eventLog-{currentDate}". For example: "eventLog-2015-08-01", "eventLog-2015-08-02", etc. Naming indexes with date makes it easy to delete them on a rolling basis.

Issue with this setup is we don't know which index to target for any given query, for example, to find all error events:

```
GET http://localhost:9200/???/Event/_search?q=event:error
```

Solution is to use index _alias_, which creates a "nickname", example:

```
POST http://localhost:9200/_aliases
{
  "actions" : [
    { "add" : { "index" : "eventLog-2015-08-02", "alias" : "eventLog" } }
  ]
}
```

Now can use the nickname in searches, for example:

```
GET http://localhost:9200/eventLog/event/_search?q=event:error
```

The above query will get forwarded to the "eventLog-2015-08-02" index.

An alias can also be assigned to multiple indexes.

## Queries and Analysis

### Basic Queries

aka "search lite"
Simple query string search using http get, searching "post_text" field for occurrence of "awesome":


```
GET http://localhost:9200/my_blog/_search?q=post_text:awesome
```

Use this when you want to find something by unique id or a simple field.

### Query DSL

This is where the real power of ES is. ES accepts specially composed JSON snippets as queries.

For example, to perform the same search as `GET http://localhost:9200/my_blog/_search?q=post_text:awesome`:

```
POST http://localhost:9200/my_blog/_search
{
  "query" : {
    "match" : {
      "post_text" : "awesome"
    }
  }
}
```

"query" tells ES to perform a _query_ command.

"match" is the _type_ of query to be performed.

ES has many built-in types of query commands.

Within the "match" command type, "post_text" tells ES which _field_ is being searched on and the value, example "awesome" specifies what data or _term_ to look for.

"match" will search for multiple terms, for example, to find all blog posts that have the word "wonderful" or "blog":

```
"match" : {
  "post_text" : "wonderful blog"
}
```

Note that documents having both terms will score higher.

ES will sort the results by score, from highest to lowest, leading to better relevance in search results.

#### Match Phrase

To find exact phrase match. Good for full text searching and more predictable results.
For example, find documents having the exact phrase "wonderful blog":

```json
{
  "query" : {
    "match_phrase" : {
      "post_text" : "wonderful blog"
    }
  }
}
```

#### Query Filters

To run more complex queries than the basic match phrase.

Filters efficiently _reduce_ the results returned from ES with logical operators. For example:

```
{
  "query" : {
    "filtered" : {
      "filter" : {
        "range" : {
          "post_date" : {
            "gt" : "2015-08-01"
          }
        }
      },
      "query" : {
        "match" : {
          "post_text" : "wonderful blog"
        }
      }
    }
  }
}
```

Top level "query" tells ES this is a query command.

"filtered" specifies that its a filtered query.

"filter" specifies a filter to be applied to results.

"range" is a good filter to use on date fields, since ES can use logical operators such as greater than, less than etc. on dates.

Below the filter, is the "query" from before.

Another example using "term" filter to reduce results by user_id:

```json
{
  "query" : {
    "filtered" : {
      "query" : {
        "match" : {
          "post_text" : "blog"
        }
      },
      "filter" : {
        "term" : {
          "user_id" : "2"
        }
      }
    }
  }
}
```

### Highlighting

Use "highlight" command, for example, to highlight matched words in the "post_text" field:

```json
{
  "query": {
    "match": {
      "post_text": "blog"
    }
  },
  "highlight": {
    "fields": {
      "post_text": {}
    }
  }
}
```

## Analytics

Elasticsearch comes with a lot of built in analytic functions to make performing calculations on data fast and easy. These are called _aggregations_.

In SQL terms, aggregations are kind of like "group by" but more powerful.

Some features include hierarchical rollups, min/max, percentiles, histograms, distance between geographical points.

General structure:

```json
{
  "query": {
    "match": {
      "post_text": "blog"
    }
  },
  "aggs": {
    "NAME": {
      "AGG_TYPE": {}
    }
  }
}
```

"aggs" is Elasticsearch keyword specifying the aggregation feature.

"NAME" can be any value you specify, for example "all_words".

"AGG_TYPE" is one of the aggregation types, for example "terms".

For example, find all docs that have the word "blog" in "post_text" field, and then for these results, return how often each of the terms occur.

```json
{
  "query": {
    "match": {
      "post_text": "blog"
    }
  },
  "aggs": {
    "all_words": {
      "terms": {
        "field": "post_text",
        "size": 10
      }
    }
  }
}
```

"avg" is another aggregation type, for example, if the data contains a numeric field such as "post_word_count", and want to know, for the docs that match a given query, what's the average word count:

```json
{
  "query": {
    "match": {
      "post_text": "blog"
    }
  },
  "aggs": {
    "all_words": {
      "terms": {
        "field": "post_text",
        "size": 10
      }
    },
    "avg_word_count" : {
      "avg": {
        "field": "post_word_count"
      }
    }
  }
}
```

## Index

Consider the following sentences:

* i can not wait to drive my new car (Document 1)
* my new car looks great today (Document 2)

Elasticsearch will index each continguous block of letters and numbers as a _term_.
These terms are maintained in an _inverted index_, which stores which documents each term belongs in.

![Image of Inverted Index](images/inverted-index.JPG)

In an inverted index, the terms are considered the _primary data_. This makes searching by these terms very fast.

## Analysis

When documents with sentences are inserted into Elasticsearch, the sentences are broken down into their constituent parts, called _terms_.

The work of breaking down strings of text into terms or _tokens_ is performed by an _analyzer_, by a process referred to as _tokenization_.

Analyzer performs three main jobs in tokenizing:

1. Apply character filter (first pass at cleaning up string): remove html, convert numerals to words such as "9" to "nine".

1. Tokenize by whitespace, period, comma etc.

1. Token filter to remove stop words like "and", "the", etc.

### Analyzers

Elasticsearch comes with built in analyzers, and custom ones can also be created.

__standard__:  Default. General purpose analyzer for multiple languages. Breaks up text strings by natural word boundaries, removes punctuation and lower-cases terms.

__whitespace__: Breaks up text by whitespace only. More useful for strings like computer code or logs.

__simple__: Breaks up strings by anything that isn't a number, lowercases terms.

#### Standard: Example

![Image of Standard Analyzer](images/standard-analyzer.JPG)

* Dropped hyphenation & parens
* Converted to lower case
* Separated by natural word endings

These terms are what's compared against when running term queries. So searching for "(string)" would not yield any matches because parens were stripped out.

#### Analysis: API

Elasticsearch comes with an API for testing the analysis of strings, which effectively, _explains_ the analysis.

```
POST http://localhost:9200/_analyze?analyzer=standard
"string to be searched"
```

Use Postman or other REST client to send raw text in POST body because Sense does not accept a request without brackets.

For example:

```
POST http://localhost:9200/_analyze?analyzer=standard
Convert the title-case text using the ToLower(String) command.
```

Returns:

```json
{
    "tokens": [
        {
            "token": "convert",
            "start_offset": 0,
            "end_offset": 7,
            "type": "<ALPHANUM>",
            "position": 0
        },
        {
            "token": "the",
            "start_offset": 8,
            "end_offset": 11,
            "type": "<ALPHANUM>",
            "position": 1
        },
        {
            "token": "title",
            "start_offset": 12,
            "end_offset": 17,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "case",
            "start_offset": 18,
            "end_offset": 22,
            "type": "<ALPHANUM>",
            "position": 3
        },
        {
            "token": "text",
            "start_offset": 23,
            "end_offset": 27,
            "type": "<ALPHANUM>",
            "position": 4
        },
        {
            "token": "using",
            "start_offset": 28,
            "end_offset": 33,
            "type": "<ALPHANUM>",
            "position": 5
        },
        {
            "token": "the",
            "start_offset": 34,
            "end_offset": 37,
            "type": "<ALPHANUM>",
            "position": 6
        },
        {
            "token": "tolower",
            "start_offset": 38,
            "end_offset": 45,
            "type": "<ALPHANUM>",
            "position": 7
        },
        {
            "token": "string",
            "start_offset": 46,
            "end_offset": 52,
            "type": "<ALPHANUM>",
            "position": 8
        },
        {
            "token": "command",
            "start_offset": 54,
            "end_offset": 61,
            "type": "<ALPHANUM>",
            "position": 9
        }
    ]
}
```

#### Analysis: Setting

When to choose analyzer? Most commonly, specify in "mapping" settings of index, for example:

`POST http://localhost:9200/my_blog`

```json
{
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string",
          "analyzer": "standard"
        },
        "post_date": {
          "type": "date"
        }
      }
    }
  }
}
```

Now any blog posts inserted, will have their "post_text" field tokenized according to the standard analyzer.

If you don't want a field to be analyzed, i.e. inserted exactly as is, specify "not_analyzed" in mapping, for example:

```json
"user_id": {
   "type": "integer",
   "index": "not_analyzed"
 }
```

Do this on fields for which you need predictable results such as user id, state fields like "status", etc.
