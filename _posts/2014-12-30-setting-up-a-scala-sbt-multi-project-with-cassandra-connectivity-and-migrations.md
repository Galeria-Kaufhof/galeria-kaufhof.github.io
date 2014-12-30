---
layout: post
title: "Setting up a Scala sbt multi project with Cassandra connectivity and migrations"
description: ""
category: tutorials
tags: [scala, sbt, cassandra, migrations, pillar, assembly]
---
{% include JB/setup %}

*I have learned the following through the great support and advice of my coworkers Jens Müller and
[Martin Grotzke](https://twitter.com/martin_grotzke).*

## About

I have recently joined the new multi-channel retail eCommerce project at Galeria Kaufhof in Cologne. This meant diving
head-first into a large-scale Scala/Play/Akka/Ruby software ecosystem, and as a consequence, a lot of learning (and
unlearning, and disorientation, and some first small successes), as I'm still quite new to Scala.

Thus, this is a from-the-trenches tutorial about some of the first things I have managed to understand and build. My
goal is to provide a detailed step-by-step tutorial which shows how to set up a new Scala project with three sub-modules
(one being plain old Scala, one being based on Play2, and one on Spray) using sbt. I'll also describe how to enable all
three modules to speak with an Apache Cassandra database, and how to add automatically applied database migrations (in
order to allow using the project in a Continuous Delivery setup like the one we use at Galeria Kaufhof).

This tutorial is well suited for reasonably experienced software developers with some first basic experience with Scala
and Cassandra.


## Prerequisites

The project setup we are going to create is based on Java 8, Scala 2.11, sbt 0.13 and Cassandra 2.0.11

You'll need a working Scala environment (on Mac OS X with Homebrew, `brew install scala sbt`
should do the job). You'll also need a running and reachable Cassandra installation - setting this up is out of the
scope of this text, but
[this tutorial](http://christopher-batey.blogspot.de/2013/05/installing-cassandra-on-mac-os-x.html) might be a good
start if you are using Mac OS X, and there is also
[the official Getting Started page](http://wiki.apache.org/cassandra/GettingStarted).


## Getting started

Let's start by creating a directory for our project: `mkdir myproject`. Within it...