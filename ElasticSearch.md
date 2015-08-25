# Elastic Search Class

## What is elastic search
    
Distributed, highly available, data storage & retrieval

## Why Elastic Search

Suggestion engine

Data Store

## Terminology:

**node**: single running instance of elasticsearch

**clusters**: coordinating node will distribute client request to other nodes

**index**: a collection of shards, similar to a table in a db; makes documents searchable; pay price on write, not read

**shards**: index only references primary shards

**primary shards**: a piece of the index

**replica shards**: a replication of the primary

do not keep replica and primary on the same machine; for fault tolerance

nodes allow parallel search
        

## Consequences of sharding

no joins

no rollback

no foreign keys

atomic for single documents only


## Documents

### metadata

**_source**:    body of the document; Allows for update API, etc.
    
**_timestamp**: sift for newest data like Cassandra?

**_all**: search on a field that they don’t know the index of; slow

**_ttl**: probably don’t use this

**_size**: monitor the size of documents

## Elastic API

### Index API
Apply the ID in the the request for fast lookup

**_create** will create a document with the given ID if_not_exists

timestamp API query is only around for TTL, probably don’t use it

index request is handled like cassandra where the hash is made and sent to the corresponding shard - murmur3

    
### Search API
near real time (~1sec)
    
### Get API
real time guarantee, pulled from log

will round robin to load balance

**_local** will not round robin

**_primary** will send to the primary shard only

**Document Versioning**: every alteration will be tracked as a version
1?version: will let you execute operation only if version you specify is current

### Update API
dynamic scripting will allow groovy scripts to run

disabled by default

plugins will allow for scripts in other langs

### MGet API
get multiple documents

anything in the url does not need to be repeated in the document

### Bulk API
minimizes round trips when performing bluk index, delete, update operations

needs a newline at the end of every operation for parsing

inspect the response, a 200 only means that a single operation of the bulk succeeded

test bulk sizes and responses on hardware and test bulks multithreaded, 100Mb is the max size by default

### API Lab

```json
GET _search
{
  "query": {
    "match_all": {}
  }
}

PUT employees
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}

GET /employees/_mapping
GET /employees/_settings

PUT /employees/sales/1
{
  "age": 40,
  "salary": 40000,
  "fname": "larry",
  "lname": "davidson",
  "roles": ["cashier", "sales manager", "part-time janitor"],
  "start_date": "2015-08-17T00:00:01"
}

PUT /employees/executives/1
{
  "title": "CFO",
  "salary": 250000,
  "bonuses": 45000,
  "stock_options": true
}

GET /employees/executives/1

DELETE /employees/executives/_create

POST /employees/executives/1/_update
{
  "doc": {
    "title": "CEO"
  }
}

GET /employees/_mget
{
  "docs": [
    {
      "_type": "executives",
      "_id": 1
    },
    {
      "_type": "sales",
      "_id": 1
    }
  ]
}
```


## Text Analysis
Stopwords are those that show up too frequently like 'the', etc.

Case insensitive, everything is stored in lowercase

Catagorize words by their stem, ignoring thisngs like '...ing' or '...s'

Search terms get routed through the char filters and tokenizer as well

**keyword**: prevent phrase from getting tokenized

synonyms can be added at search time, or to the default. Default synonyms can only be changed with a node reboot

replace 'foo => bar' will replace the token with the new word

ICU analysis plugin

### Analyzers
**Standard**: grammar based tokenization. Works well on anything but German

**Keyword**: create a single token out of the text (id's enums)

**Whitespace**: breaks text at whitespace

**Pattern**: tokenize based on regex

**UAX / URL Email**: keeps foo@bar.com as a single 'word'

**Http encode**: breaks text at '/ '

### Token Filters
**lowercase**: for case insensative search

**nGrams / edgeNGrams**: breaks the tokens into ngrams chunks. Helps with fuzzy search or search on unknown languages. This would replace a levenshtein distance at search time with ngrams at index time

**Shingles**: NGrams for words; prevents the need for phrase queries at search time with shingles at index time

**Stemmer**: Algorithmic stemming filters, they have a tendency to be rather aggressive
- Snowball
- porter stem
- kstem

**Hunspell**: Dictionary based stemming filters; Mozilla / Open Office have well maintained ones

### Lab

```json
POST _analyze?analyzer=english&filters=lowercase&text=it is unlikely that I\’m especially good at analysis yet.


POST /analyzertest2
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "my_shingle_tokenizer"]
        }
      },
      "filter": {
        "my_shingle_tokenizer": {
          "type": "shingle",
          "min_shingle_size": "2",
          "max_shingle_size": "5",
          "output_unigrams": false
        }
      }
    }
  }
}

POST /analyzertest2/_analyze?analyzer=my_analyzer&text=it is unlikely that I\’m especially good at analysis yet.
```

## Mappings

### Controlling Dynamic Mappings

### Type Mappings

**String, byte, short, integer, long, float, double, date**: the core data types

**Root object / Object**: represents a document of inner document

**nested**: an embedded document

**ip**: an IPv4 address (IPv6 does is not compatable yet!)

### Text Fields

...

### Numeric Fields

...

### Date Fields

ISO8601 time format is native in elastic

Supports multiple formats like such:

- `yyyy/MM/dd HH:mm:ss || yyyy/MM/dd`

- this runs left to right; have most used format to the left

### Boolean Fields

Entered as 'true' and 'false'

Stored internally as 'T' or 'F'

### Common Attributes

**include_in_all**: indicated whether the value of the field should be included; this is an override value

**null_value**: Defines a default value for the field

**copy_to**: ...

**Store**: whether to store the field's original value; store separately from document content in Lucene
- if you have _source disabled and you still want search
- sorting or aggregations for small fields in large documents
- sounds like a secondary index in Cassandra

### Object Fields

...

### Multi-Fields

enables virtual fields so the document can be indexed in several different ways

very popular during reindexing

### Metadata Fields

**_type**: indexed and related to a mapping

**_all**: is an indexed 'catch all' filter

**_timestamp**: can be used from a field in your data, path works as well; updated at index time

### Custom Metadata

for a mapping, **_meta** can be set to add versioning, for version control

### Default Mappings for Index

**_default\_**: set a default mapping for the entire index

### Dynamic Field Mapping

...

### Index Templates

Elk stack created an index everyday

This is a template for creating multiple indexes

The created template is a deep copy, will not relate back to the template after creation

**order**: the order of which the operations are applied, 0 is run first then in ascending order

### Lab

```json
POST _analyze?analyzer=english&filters=lowercase&text=it is unlikely that I\’m especially good at analysis yet.


POST /analyzertest2
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "my_shingle_tokenizer"]
        }
      },
      "filter": {
        "my_shingle_tokenizer": {
          "type": "shingle",
          "min_shingle_size": "2",
          "max_shingle_size": "5",
          "output_unigrams": false
        }
      }
    }
  }
}

POST /analyzertest2/_analyze?analyzer=my_analyzer&text=it is unlikely that I\’m especially good at analysis yet.

PUT /stack/question/_mapping
{
  "_meta": {
    "comment": "Metadata isnt fun"
  }
}

GET /stack/question/_mapping

PUT /stack/question/_mapping
{
  "properties": {
    "last_editor_display_name": {
      "type": "string",
      "search_analyzer": "keyword"
    }
  }  
}

PUT _template/mystack_wildcard
{
  "template": "mystack*",
  "mappings": {
    "question": {
      "dynamic_templates": [
        {
          "strings": {
            "match": "s_*",
            "mapping": {"type": "string"}
          }
        }
      ]
    }
  }
}
```

## Search

### Structured vs Unstructured

...

### Pagination

A search request is always within a 'page'

The order of the list is determined by the sorting index

```
curl -ivXGET 'localhost...' -d '{
    "from": "3", "size": "7", ...
}'
```

### Sorting Field-data

when sorting by field, all values of the field in all documents will be loaded into memory; java JVM freaks out.

index with dock values to help heap?

### Search - Phases

**Phase 1: Query**

Query runs on individual shards... mapreduce?
    
Returns index and score for that index
    
Runs priority queue sort

**Phase 2: Fetch**

Retrieves the information from the priority queue and returns to client


### Search Types

**QUERY_THEN_FETCH**

...

**DFS_QUERY_THEN_FETCH**

Really good in unittesting and for really small indicies

Create an index with a single shard and run DFS

**COUNT** 

Skips the fetch cycle, just returns the number of documents containing the search term

**SCAN**

Allows to iterate over large amounts of data, giving up on sorting
 
new documents will not show up in scan, it searches at point-in-time

`track_score = true` will show the score, just not sort them

### Query Match

"analyzer" can be specified per query

**Slop**: number of other words allowed between query terms in a document; type: phrase

**Minimum should match**: can be a number of words or a percentage for unknown entry length

**Max_expansion**: the number of terms the last prefix will expand to; type: phrase_prefix

## Detour

### What's in a Logic

**must**: Q1 _AND_ Q2

**must_not**: _NOT_ Q1

**should**: Q1 _OR_ Q2

### Multi-Match

can match queries in title and body for example

score can be boosted on certain fields with a ^

```json
{
  "multi_match” : {
    "fields" : [ “title^10", "body" ],
    "query" : "quick fox"
  }
}
```

### Bool

```json
{
    "bool" : {
        "must" : [
            { "match" : { "color" : "blue" } },
            { "match" : { "title" : "shirt" } }
        ],
        "must_not" : [
            { "match" : { "size" : "xxl" } }
        ],
        "should" : [
            { "match" : { "textile" : "cotton" } }
￼         ]
    }
}
```

can set a minimum_should_match as well; supports percentage format

### Query String

```json
{
    "query_string" : {
        "query" : "(tony OR thor) AND intelligence:[6 TO *]"
￼     }
}
```
Is not an elegant solution, is said to be finicky and picky

Will cause performance issues down the roads

Simple query is much better:

```json
￼{
  “simple_query_string" : {
    "query" : "(tony | ironman) + thor"
  }
}
```
Fuzzy and regexp are supported as well

### Query DSL: Filters

...

### BitSets, Filters, and Lucene

**BitSet**: Array data structure that compactly stores bits

Create segments (ElasticSearch Segment Spy Plugin) and merge them after a given amount. Similar to cassandra leveled compaction

Filters get cached in bitsets to make filters really quickly
- Performs a binary and on hits from each filter and returns the documents relating to the true bit

Things like the filter 'now' are never cached because they are constantly changing

**_cache=false**: flag can be set to disable cache on queries that are unlikely to be repeated

compound filters are not cached, however their pieces are cached

### Highlighting

**Highlighting**: showing the snippets of text that match the search

Types of highlighters:
- Highlighter: requires re-analysis of the text
- FVH Fast Vector Highlighter: uses term vectors; doesn't need to reanalyze text before rendering; not necessairily faster than highlighter, depends on data
- Postings: requires _**index_option**_ be set to offset; sentence based

Highlighter type is automatically chosen based on mappings

### Lab

```json
GET /stack/question/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "title": {
            "query": "java object",
            "type": "phrase",
            "slop": 1
          }
        }},
        {"range": {
          "creation_date": {
            "gte": "2014-09-10",
            "lte": "2014-09-13"
          }
        }}
      ]
    }
  },
  "sort": [
    {
      "creation_date": {
        "order": "desc"
      }
    }
  ],
  "highlight": {
    "fields": {
      "title": {}
    }
  }
}
```
## Relevancy

word matches get lumped into a relevancy vector

### TF-IDF

Term Frequency - inverse document frequency

- The more the term occurs in the document, the weighting is increased
- The more the term occurs in the document collection, weighting is decreased

**explain=true**: will output weighting for results in search

### Influencing Relevancy

**Boost**: make a query clause more important than another; boost is a relative importance to other clauses and boost values; is normalized in results.

boost can work on indicies as well, allows to boost fo more relevant recent data:

```json
{
    "indices_boost": {
        “data_2015_07": 2,
        “data_2015_08": 4
    },
    "query": {
        "match": {
            "text": “some terms to search on"
        }
    }
}
```
**Boost_factor**: multiplicative increase in importance; not normalized, not relative

### Decay Function

Great for reducing relevancy based on range; date ranges and geo-location (seems really important to TrueCar algorithm) are good examples

### Script Scoring

...

### Field Value Factor

Replace the field relevancy value with a custom algorithm

'boost_mode: replace'

### Lab
```json
GET /stack/question/_search
{
  "explain": true, 
  "query": {
    "bool": {
      "should": [
        {"match": {
          "title": {
            "query": "Maven",
            "boost": 2
          }
        }},
        {"function_score": {
          "query": {
            "match": {
              "body": {
                "query": "C++"
              }
            }
          },
          "functions": [
            {
              "field_value_factor": {
                "field": "comment_count",
                "factor": 2,
                "modifier": "none",
                "missing": 0
              }
            }
          ]
        }},
        {"function_score": {
          "query": {
            "match": {
              "title": {
                "query": "Java"
              }
            }
          },
          "functions": [
            {
              "filter": {
                "term": {
                  "tags": "database"
                }
              },
              "boost_factor": 5
            },
            {
              "filter": {
                "term": {
                  "tags": "mysql"
                }
              },  
              "boost_factor": 3
            }
          ],
          "score_mode": "multiply"
        }}
      ]
    }
  },
  "sort": [
    {
      "creation_date": {
        "order": "desc"
      }
    }
  ],
  "highlight": {
    "fields": {
      "title": {}
    }
  }
}
```

## Suggestions

Suggestions work on an index level

Payloads are held in memory, keep them to _**necessary fields only**_

_**Term, Phrase, Completion, & Context**_: the four kinds of suggestions

### Term Suggestion
Based on edit distance; text is broken down into tokens

**Term Modes**:

- missing: Only suggests if term is not in index
- always: Find similar suggestions
- popular: Suggestions occurring more often than the input term

popular seems to be a good strategy

nGrams could be an interesting option; index size becomes a problem with scale; not easily implemented

### Phrase Suggestion
words are not separated, the phrase is the token; same as term past that

- real_world_error_liklihood can be increased depending on platform

### Completion Suggestion
...

### Context Suggestion
...

### Suggestions Lab

```json
POST stack/_search
{
  "query": {
    "match": {
      "title": {
        "query": "javs sprng framewerk"
      }
    }
  },
  "suggest": {
    "mysuggest": {
      "text": "javs sprng framewerk",
      "term": {
        "field": "title",
        "suggest_mode": "missing",
        "sort": "score",
        "size": 3
      }
    }
  }
}

POST stack/_search?search_type=count
{
  "suggest": {
    "text": "rubi on rais",
    "simple_phrase": {
      "phrase": {
        "field": "title",
        "size": 3,
        "max_errors": 2
      }
    }
  }
}
```

## Aggregations

Search for unknown query criteria

Aggregations are the 'next generation' of facets, are put alongside queries in requests. The aggregations should be sent to the backend through a POST filter rather than altering the aggs; allows multiple aggregations on the same field to be made.

**Aggregation Scope**

- **filter_query**: defined in the query part; impacts the main scope
- **post_filter**: defined _outside_ of the query part; does not impact main scope (i.e. document set)

```json
  "query" : {
    "filtered" : {
      "query" : { "match" : { "name" : "John" }},
      "filter" : {
        "range" : { "age" : { "from" : 20, "to" : 50 }}
      }
  },
￼  "aggregations" : {
    "employer" : {
      "terms" : {
        "field" : "company"
      }
    }
  }

```

### Buckets & Metrics

**Buckets**: Aggregations that build buckets. Each bucket is associated with some criteria over documents. During query execution, each document is evaluated against the created buckets and each bucket keeps track of what documents “fall” in it. Each bucket effectively defines a set of documents derived from the document set within the aggregations scope.

- works very well for data such as enum values, date ranges, counts, etc.

**Metrics**: Aggregations that given a set of documents, produce a single/multiple scalar/s. Typically, metrics aggregations generate numeric stats that are computed over a specific document set

### Aggregations

**Sum Agg**: create a sum based on a single field in all results

**Minimum / Maximum Agg**: pull oldest date / newest date, smallest / largest field, etc.

**Stats Agg**: returns min, max, average, sum; works on numbers, dates, geo data

**Extended Agg**: returns stats and sum of squares,variance, standard deviation, and standard deviation bounds; will error on certain field types

**Terms Agg**: will aggregate based on terms field; size and sorting can be specified in the query; we don't know what the terms are - make buckets for all the terms

**Histogram Agg**: created output that would be transferrable to a visual chart in Kabana for instance; invervals can be set in query

**Range Agg**: very similar to histogram, just has self set range buckets as opposed to intervals; bucket names can be overwritten with _key_

**\*_range Agg**: date range, IP range aggregation; IP range aggregation can take advantage on mask to range by subnet for instance

**Date Histogram Agg**: same as histogram, interval understands date related terms such as 'day' or 'month'

**Filters Agg**: buckets can be based upon filters, filter clauses are created by the user. Filters with average aggregation is another approach that allows the _avg_ agg to be implemented; similar to terms, however we know what terms we're searching for

**Missing Agg**: Find documents that don't have information for a field; there is missing data in the document field

**Cardinality Agg**: Aggregates the number of unique values for a defined field

**Significant Terms**: Find the 'uncommonly common' terms. e.g. what terms are significant to 'Java' vs. the entire set of documents. Focused agg of terms based on a query term's results

**Top Hits Agg**: will return a top 10 hit array for each returned bucket; a distributed join

**Percentile Agg**: ...

### Aggregation Filters

**Scripting with Aggs**: ...

**GeoLocation**: distance and bounds from a point

## Document Design

Define questions before the data modeling. Data modeling needs to be created before data uploads. 

Heirarchical inner-objects need to be removed unless you have a 1-to-1 relationship. A 1-to-many relationship will cause problems inside of Lucene storage containers. Lucene stores thisinformation as an unordered array per field.

**Nested Type**: necessary to create proper objects of nested data inside lucene. Nested type can be separated by dot notation; it is not recommended that data have dots in it.

**Parent / Child Type**: will make separate JSON documents for each level. Parent / Child documents must live in the same index and be of different types

```json
curl -XPOST 'localhost:9200/crunchbase/employee?parent=amazon' -d '{
    "first_name" : "Jeff",
    "last_name" : "Bezos"
￼}'
```

* Pay close attention to how the parent is explicitly defined so the child document will be stored on the partent's shard, not on whichever shard would hold _id: 1_ after murmur3.

**has_child Query**: ...

- _inner\_hits_ will give responses from the inner query

Nested is more efficient that parent / child, nested uses far less mem in the JVM. Nested has to create new index for any information changes. Parent / Child can cause hotspots in shards

**Parent / Child vs Flattening Data**: ...

### Suggestions

- Add good metadata to docs such as which app version created this data and what datamodel this data was created with
- Roles and group fields should be added so security measures can be added down the road
- Do not rely only on document types and index name
- Use one index per document type


## Percolator, Search Reversed

Percolator allows you to search a document for possible queries that will work on it. Percolator creates an index of questions that are searched against with documents for questions that are relevant to aforementioned documents. Percolating is done at the time of indexing the document

_.percolator_ is a type and reserved word, _\_percolator_ will return a matches array

**Filtering**: allows for including and excluding specific queries based on provided metadata; only a subset of queries will be run against new document undergoing percolation

Percolators should not be kept in the same index as data; allows for nodes that only do percolation, spees up. Separating specific percolation nodes can allow these nodes to have specalized hardware to increase speed as well as large heap and uncontested resource pools that would be used from indexing documents

## Client Testing and such

...
