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

## About
At Galeria.de, we learned the hard way that it's critical to understand the inner workings of distributed
masterless database Cassandra if you want to experience good performance during reads and writes. This post describes
how Cassandra works under the hood, and shows how understanding these details helps to anticipate which use patterns
work well and which don't.


## Network and node storage architecture

### The network

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

#### Step 1: Client-Coordinator connection
Our client (i.e., the process which issues the CQL statement) connects to a so-called *coordinator node*. This is not a
special node within our cluster - any node that happens to receive a query from a client also happens to act as the
coordinator for this query.

#### Step 2: Mapping the partition key value to a cluster node
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

#### Step 3: Determine replication nodes
"At least" because what also comes into play is the replication factor of the keyspace holding the user table (which
contains the row in question). In our example, this factor is 3, which means that the row we create via the INSERT query
needs to be stored on two more nodes, besides its "main" node, 2. The algorithm for this is simple - additional
replicas of the row are stored on the next nodes clockwise in the ring - in our example, nodes 3 and 4.

Note that the replication of the write in question happens on all three nodes (2, 3, and 4) simultaneously, and not
one-after-the-other. This detail is important because it explains why Cassandra is relatively optimistic regarding the
to-disk-sync of a node's commit log (see chapter "The node storage" for more on this).

As said, the replica order is based on the logical structure of the cluster - the cluster sees itself as an ordered ring
structure, where each node has a "neighbour" node that comes "after" it in the ring.

#### Step 4: Wait for node write operations to finish and report back to the client

Once enough nodes have reported that their local write operation succeeded (see chapter "The node storage" for the
details), the coordinator node in turn reports the success of the INSERT operation back to the client. Here, "enough"
nodes depends on the *replication factor - query consistency level* relation for this operation. If we insert a row into
a table that belongs to a keyspace with a replication factor of `3`, and the query was issued with a consisteny level of
`QUORUM`, then `2` nodes (the quorum of 3 nodes) acknowledging the write is considered a success by the coordinator.


### The node storage

Let's "zoom in" on our node 2 and have a look at what happens in terms of its local storage when it receives the write
request from the coordinator node as a result of the INSERT query issued by the client. As noted, the same operations
happen on nodes 3 and 4 simultaneously.

While the final goal is to have the data stored in an SSTable, several moving parts are involved in storing the row data
on a node:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra node storage.svg">

The very first thing that happens storage-wise is appending the write data to the *commit log*, which resides on the
harddisk (but writes to it are only fsync'd periodically, see below).

The commit log is part of the process because it ensures data durability even in crash scenarios. Even if the server
crashes during a write operation - if the data made it into the commit log, it can be recovered when the node is coming
up again.

However, as mentioned above, Cassandra is quite optimistic in regards to actually syncing the commit log to disk - per
default, this happens every 10 seconds, but the node immediately acknowledges the write to the coordinator (after also
writing to the MemTable, see below), without waiting for the fsync. This means that there is a window of up to 10
seconds during which, in case of a server crash, the data is not persisted on the harddrive of the crashing server node,
although the coordinator will think it is.

What in theory sounds highly problematic in terms of data durability isn't a big deal in practice. Cassandra assumes
that data is always replicated, and two participating server nodes crashing within the same 10 seconds window is very
unlikely.

Next in line is the so-called MemTable. It is an in-memory write-back cache of rows that is regularly flushed into an
SSTable whenever the Memtable or the commit log reach a certain size (at which point they are considered "full").

Why does the MemTable exist? When receiving a read request, Cassandra cannot retrieve the requested data efficiently
from the commit log - it is append-only, which means it contains data in the order that write requests arrived, which in
turn means that in order to retrieve all data for a row would mean to sequentially scan through the whole commit log
from top to bottom, which would be prohibitively expensive in terms of disk I/O.

The layout of SSTables, on the other hand, *is* optimized for efficient row data lookup. However, Cassandra cannot
update the SSTable holding the data for a given row synchronously upon each write, because this would result in a huge
amount of random disk I/O operations, making the write scenario prohibitively expensive in terms of disk I/O. To
circumvent this, SSTables are *never* updated - instead, they are immutable/read-only, and new SSTables are created
regularly.

If row data cannot be retrieved from the commit log, and SSTables are only created once in a while, then another data
structure is required to answer read requests *immediately* and *efficiently*.

This is where MemTables come into play. Each server node has one MemTable for each table it carries. A MemTable lives,
as the name implies, in memory, and is mutable, i.e., row data is read from it an written to it as needed, which thanks
to the I/O performance of computer memory, is not prohibitively expensive - a fact that is nicely illustrated by one of
my all time favorite tables, where typical computer-world timings are compared to typical timings that humans can relate
to:

    1 CPU cycle                      0.3 ns     1 s
    Level 1 cache access             0.9 ns     3 s
    Level 2 cache access             2.8 ns     9 s
    Level 3 cache access            12.9 ns    43 s
    Main memory access               120 ns     6 min
    Solid-state disk I/O          50-150 μs   2-6 days
    Rotational disk I/O             1-10 ms  1-12 months
    Internet: SF to NYC               40 ms     4 years
    Internet: SF to UK                81 ms     8 years
    Internet: SF to Australia        183 ms    19 years
    OS virtualization reboot           4 s    423 years
    SCSI command time-out             30 s   3000 years
    Hardware virtualization reboot    40 s   4000 years
    Physical system reboot             5 m     32 millenia

Thus, reading and writing data from and to main memory versus from and to disk is like waiting for your salad order to
be finished in 6 minutes versus getting the salad somewhere between the day after tomorrow and next year. Sounds like a
MemTable does the job.

So, right after appending the INSERT data to the commit log, the node puts the resulting row data into the MemTable
structure. At this point, the data is both durable (commit log) and retrieveable (MemTable), and thus, the data node can
acknowledge the write to the coordinator node.  


### Performance of Cassandra operations

Musings about the performance of a Cassandra operation boil down to two questions: How much will the coordinator node
have to do in terms of network I/O, and how much work will a participating data node have to do in terms of disk I/O?

The fastest operations are those for which the coordinator has to talk to only one data node, and where the approached
data node can find the requested information by searching through as little SSTables as possible. An expensive
operation is one where the coordinator node has to talk to all nodes on the cluster (possibly multiple times), and where
each of the nodes has to scan through all its SSTables to retrieve the requested information.







https://wiki.apache.org/cassandra/Durability
https://wiki.apache.org/cassandra/ArchitectureCommitLog
https://wiki.apache.org/cassandra/MemtableSSTable
http://www.datastax.com/dev/blog/updates-to-cassandras-commit-log-in-2-2
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlIntro.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlHowDataWritten.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlHowDataMaintain.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlWriteUpdate.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlAboutDeletes.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlIndexInternals.html
http://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlAboutReads.html
