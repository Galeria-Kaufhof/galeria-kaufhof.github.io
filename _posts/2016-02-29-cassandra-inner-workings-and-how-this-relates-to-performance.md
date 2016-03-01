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

<h2>Network and storage architecture</h2>

<h3>The network</h3>

A production Cassandra setup always consists of multiple nodes, where a node is one Cassandra server process on one
system. All nodes are connected via the network.

In this post, we always assume that our setup is a cluster of 5 nodes, numbered 1 to 5.

Cassandra does not have any kind of "master" node - all nodes are created equal.

We further assume that we have created a keyspace "galeria" (which is going to hold the data tables we are going to
create) as follows:

    CREATE KEYSPACE galeria WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor': 3 };

The replication factor defines that whatever row we insert into a table within this keyspace is stored on three
different nodes.

We can now create a table "users" like this:

    CREATE TABLE users (
        username TEXT,
        firstname TEXT,
        lastname TEXT,
        PRIMARY KEY (username)
    );

When we insert a row into this table as follows:

    INSERT INTO users (username, firstname, lastname) VALUES ('jdoe', 'John', 'Doe');

the following happens network-wise. Because `username` is the first (and in this case only) part of the primary key, it
acts as the partition key. The value of the partition key column is used by the cluster to determine the node onto which
to store the row. To do so, a hash function - the so-called partitioner - is applied on the value, and the result is a
token. This token then tells the cluster about the target node, because each node is responsible for a certain range of
nodes. Assumed that tokens would run from 0 to 49, 
