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

Logically, a cluster is organized into keyspaces, which contain tables. Tables contain rows, and rows have columns.

Physically, the content of a table row is always stored on the hard drive of at least one node in the cluster, and is
depending on how the keyspace has been defined upon creation, this row content is replicated to 0 or more other nodes
in the cluster. If all of this doesn't make sense now, it will once you've read this post.

In this post, we always assume that our setup is a cluster of 5 nodes, numbered 1 to 5. A 5 node Cassandra cluster is
visualized as follows:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra cluster.svg">
     
Note that this is a logical visualization, not a network diagram - in reality each node talks to all other nodes in the
cluster, not to its neighbours only.

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

then the following happens network-wise. Because `username` is the first (and in this case only) part of the primary
key, it acts as the partition key. The value of the partition key column is used by the cluster to determine the node
onto which to store the row. To do so, a hash function - the so-called partitioner - is applied on the value, and the
result is a token. This token then tells the cluster about the target node, because each node is responsible for a
certain range of tokens. Assumed that tokens would run from 0 to 49, we can visualize the tokens-to-node mapping as
follows:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra cluster with tokens.svg">

That is, node 3 holds all the rows of table "users" in keyspace "galeria" where the value of column "username" for a
given row results in one of the hash function tokens from 20-29.

For example, let's just make up that the column value "jdoe" would result in token value 17. This means that the cluster
must store the according row on node 2.

What also comes into play is the replication factor of the keyspace holding the user table (which contains the row in
question). In our example, this factor is 3, which means that the row we create via the INSERT query needs to be stored
on two more nodes, besides it's "main" node, 2. The algorithm for this is simple - additional replicas of the row are
stored on the next nodes clockwise in the ring - in our example, nodes 3 and 4.

As said, this applies to the logical structure of the cluster - the cluster views itself as a logical ring structure,
where each node has a "neighbour" node that comes "after" it.


<h3>The node storage</h3>

Let's "zoom in" on our node 2 and have a look at what happens in terms of its local storage when it receives the row we
inserted into the cluster.

Several moving parts are involved in storing the row data on this node:

<img width="50%"
     src="{{ site.url }}/assets/images/cassandra/Cassandra node storage.svg">
