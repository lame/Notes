# Elastic Search Class

## What is elastic search
    
Distributed, highly available, data storage & retrieval

## Why Elastic Search

Suggestion engine

Data Store

## Terminology:
#### node
single running instance of elasticsearch

#### clusters
coordinating node will distribute client request to other nodes

#### index
a collection of shards, similar to a table in a db 

makes documents searchable

pay price on write, not read

#### shards
index only references primary shards

primary shards: a piece of the index

replica shards: a replication of the primary

- do not keep replica and primary on the same machine; for fault tolerance

nodes allow parallel search
        

## Consequences of sharding

no joins

atomic for single documents only


## Documents

### source

all the information from the document

### metadata

- #### _source:
    body of the document

    Allows for update API, etc.
    
- #### _timestamp:

    sift for newest data like Cassandra?

- #### _all:
    search on a field that they don’t know the index of

    slow

- #### _ttl:
    probably don’t use this

- ##### _size:
    monitor the size of documents

## Elastic API

### Index API
Apply the ID in the the request for fast lookup

_create will create a document with the given ID if_not_exists

timestamp API query is only around for TTL, probably don’t use it

index request is handled like cassandra where the hash is made and sent to the corresponding shard - murmur3

    
### Search API
near real time (~1sec)
    
### Get API
real time guarantee, pulled from log

will round robin to load balance

_local: will not round robin

_primary: will send to the primary shard only

Document Versioning: every alteration will be tracked as a version
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
