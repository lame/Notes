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



