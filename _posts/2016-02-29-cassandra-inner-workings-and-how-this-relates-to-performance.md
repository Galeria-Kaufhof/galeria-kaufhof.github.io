---
layout: post
title: "Apache Cassandra: Its inner workings, and how this relates to performance"
description: "At Galeria.de, we learned the hard way that it's critical to understand the inner workings of distributed
masterless database Cassandra if you want to experience good performance. Here are some of our takeaways."
category: tutorials
author: manuelkiessling
tags: [cassandra]
---
{% include JB/setup %}

<h2>About</h2>
At Galeria.de, we learned the hard way that it's critical to understand the inner workings of distributed
masterless database Cassandra if you want to experience good performance during reads and writes. This post describes
how Cassandra works under the hood, and shows how understanding these details helps to anticipate which use patterns
work well and which don't.

<h2>Network and node storage architecture</h2>

<h3>The network</h3>

A production Cassandra setup always consists of multiple nodes, where a node is one Cassandra server process on one
system. All nodes are connected via the network. There isn't any kind of "master" node - all nodes are created equal.

Logically, the data in a cluster is organized into keyspaces, which contain tables. Tables contain rows, and rows have
columns.

Physically, the content of a table row is always stored on the hard drive of at least one node in the cluster, and,
depending on how the keyspace has been defined upon creation, this row content is replicated to 0 or more other nodes
in the cluster. If all of this doesn't make sense now, it will once you've read this post.

In this post, we always assume that our setup is a cluster of 5 nodes, numbered 1 to 5. A 5 node Cassandra cluster is
visualized as follows:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra cluster.svg">
     
Note that this is a logical visualization, not a network diagram - in reality each node talks to all other nodes in the
cluster, not only to its neighbours.

We further assume that we have created a keyspace "galeria" (which is going to hold the data tables we are going to
create) as follows:

    CREATE KEYSPACE galeria WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor': 3 };

The replication factor defines that whatever row we insert into a table within this keyspace is stored on three
different nodes in the cluster.

We can now create a table "users" within this keyspace like this:

    USE galeria;
    
    CREATE TABLE users (
        username TEXT,
        firstname TEXT,
        lastname TEXT,
        PRIMARY KEY (username)
    );

When we insert a row into this table as follows:

    INSERT INTO users (username, firstname, lastname) VALUES ('jdoe', 'John', 'Doe');

then the following happens network-wise:

<h4>Step 1: Client-Coordinator connection</h4>
Our client (i.e., the process which issues the CQL statement) connects to a so-called *coordinator node*. This is not a
special node within our cluster - any node that happens to receive a query from a client also happens to act as the
coordinator for this query.

<h4>Step 2: Mapping partition key values to cluster nodes</h4>
The first thing the coordinator node needs to do upon receiving the insert query is to find out where in the cluster the
row data for this INSERT needs to be persisted. Because `username` is the first (and in this case only) part of the
primary key, it acts as the partition key. The value of the partition key column is what's used by the coordinator to
determine the first node onto which to store the row. To do so, a hash function - the so-called *partitioner* - is
applied on the value, and the result is a token. This token then tells the cluster about the target node, because each
node is responsible for a certain range of tokens. Assumed that tokens would run from 0 to 49, we can visualize the
tokens-to-nodes mapping as follows:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra cluster with tokens.svg">

That is, node 3 holds all the rows of table "users" in keyspace "galeria" where the value of column "username" for a
given row results in one of the hash function tokens from 20 to 29.

For example, let's just make up that the username column value "jdoe" would result in token value 17. This means that
the cluster must store the according row at least on node 2.

"At least" because what also comes into play is the replication factor of the keyspace holding the user table (which
contains the row in question). In our example, this factor is 3, which means that the row we create via the INSERT query
needs to be stored on two more nodes, besides it's "main" node, 2. The algorithm for this is simple - additional
replicas of the row are stored on the next nodes clockwise in the ring - in our example, nodes 3 and 4. Note that the
replication of the write in question happens on all three nodes (2, 3, and 4) in parallel, and not one-after-the-other.
This detail is important because it explains why Cassandra is relatively optimistic regarding the to-disk-sync of a
node's commit log (see chapter "The node storage" for more on this).

As said, the replica order is based on the logical structure of the cluster - the cluster sees itself as an ordered ring
structure, where each node has a "neighbour" node that comes "after" it in the ring.


<h3>The node storage</h3>

Let's "zoom in" on our node 2 and have a look at what happens in terms of its local storage when it receives the write
request from the coordinator node as a result of the INSERT query issued by the client. As noted, the same operations
happen on nodes 3 and 4 simultaneously.

While the final goal is to have the data stored in an SSTable, several moving parts are involved in storing the row data
on a node:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra node storage.svg">

The very first thing that happens storage-wise is writing the data to the *commit log*, which resides on the harddisk.
The commit log is an append-only file, and it is part of the process because it ensures data durability even in crash
scenarios. Even if the server crashes during a write - if the data made it into the commit log, it can be recovered when
the node is recovered. However, as mentioned above, Cassandra is quite optimistic in regards to actually syncing the
commit log to disk - per default, this happens every 10 seconds, but the node immediately acknowledges the write to the
client. This means 

Next in line is the so-called Memtable. It is an in-memory write-back cache of rows that are regularly flushed into an
SSTable whenever the Memtable reaches a certain size (at which point it is considered "full").

https://wiki.apache.org/cassandra/Durability
https://wiki.apache.org/cassandra/ArchitectureCommitLog
https://wiki.apache.org/cassandra/MemtableSSTable
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlIntro.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlHowDataWritten.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlHowDataMaintain.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlWriteUpdate.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlAboutDeletes.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlIndexInternals.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlAboutReads.html
