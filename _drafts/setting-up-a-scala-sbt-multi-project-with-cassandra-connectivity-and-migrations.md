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


## Setting things up

Our first topic is to create the basic project structure, where we end up having three sub-projects that are all managed
through one central `build.sbt` file.

Let's start by creating a directory for our project: `mkdir myproject`. Within this directory, we need to create a
`build.sbt` file. We need to fill it with the following content:

    name := "My Project"

    val commonSettings = Seq(
      organization := "net.example",
      version := "0.1",
      scalaVersion := "2.11.4",
      scalacOptions := Seq("-unchecked", "-deprecation", "-encoding", "utf8")
    )

    lazy val common = project.in(file("common"))
      .settings(commonSettings:_*)

    lazy val sprayApp = project.in(file("sprayApp"))
      .settings(commonSettings:_*)

    lazy val playApp = project.in(file("playApp"))
      .settings(commonSettings:_*)

    lazy val main = project.in(file("."))
      .aggregate(common, sprayApp, playApp)


This is kind of a minimum required configuration, but it's already sufficient to give us an `sbt` project which consists
of three sub-projects: **common**, **sprayApp**, and **playApp**. For each of these, we defined in which folder the code
for the project resides, and which settings apply - in this case, we share a common set of settings.

Note how we defined a fourth project: **main** is the container project which aggregates our three sub-projects. We will
later see how this simplifies certain steps of the development workflow.

Even with this basic setup, we can already explore how `sbt` behaves in a multi-project setup:

    $ sbt
    [info] Set current project to My Project (in build file:/Users/manuelkiessling/myproject/)

    > project common
    [info] Set current project to common (in build file:/Users/manuelkiessling/myproject/)

    > test
    [info] Updating {file:/Users/manuelkiessling/myproject/}common...
    [info] Resolving jline#jline;2.12 ...
    [info] Done updating.
    [success] Total time: 2 s, completed 30.12.2014 19:31:31

    > project main
    [info] Set current project to My Project (in build file:/Users/manuelkiessling/myproject/)

    > test
    [info] Updating {file:/Users/manuelkiessling/myproject/}main...
    [info] Updating {file:/Users/manuelkiessling/myproject/}common...
    [info] Updating {file:/Users/manuelkiessling/myproject/}playApp...
    [info] Updating {file:/Users/manuelkiessling/myproject/}sprayApp...
    [info] Resolving jline#jline;2.12 ...
    [info] Done updating.
    [info] Resolving jline#jline;2.12 ...
    [info] Done updating.
    [info] Resolving org.fusesource.jansi#jansi;1.4 ...
    [info] Done updating.
    [success] Total time: 1 s, completed 30.12.2014 19:40:29

As you can see, the `sbt` console allows us to switch between sub-projects using the `project` command. We can change
into one of the sub-projects, and when executing the `test` command, then it is run in the context of the chosen
sub-project. When we do not switch into a sub-project (or switch back to the main project using `project main`), running
`test` leads to the *test* command being executed in each of our sub-projects.

## Some first code

Let's now add some actual code to our `common` sub-project. We will put general-purpose code, which is
going to be used in all other projects, into `common`. The boilerplate code which allows to connect to Cassandra
instances is a good example.

To do so, we first need to create the required folder structures:

    mkdir -p common/src/test/scala/common/utils/cassandra
    mkdir -p common/src/main/scala/common/utils/cassandra

First, we will create a case class that represents a Cassandra connection URI. Let's write a *scalatest* spec for it,
which resides in file `common/src/test/scala/common/utils/cassandra/CassandraConnectionUriSpec.scala`:

    package common.utils.cassandra
    
    import org.scalatest.{Matchers, FunSpec}
    
    class CassandraConnectionUriSpec extends FunSpec with Matchers {
    
      describe("A Cassandra connection URI object") {
        it("should parse a URI with a single host") {
          val cut = CassandraConnectionUri("cassandra://localhost:9042/control")
          cut.host should be ("localhost")
          cut.hosts should be (Seq("localhost"))
          cut.port should be (9042)
          cut.keyspace should be ("control")
        }
        it("should parse a URI with additional hosts") {
          val cut = CassandraConnectionUri(
            "cassandra://localhost:9042/control" +
              "?host=otherhost.example.net" +
              "&host=yet.anotherhost.example.com")
          cut.hosts should contain allOf ("localhost", "otherhost.example.net", "yet.anotherhost.example.com")
          cut.port should be (9042)
          cut.keyspace should be ("control")
        }
      }
    
    }

Creating this spec leads to `common` depending on the *scalatest* library. We need to declare this dependency in our
`build.sbt` file. Because we are going to use *scalatest* in all our submodules, it makes sense to put the dependency
declaration into a `val` which can be reused:

    name := "My Project"
    
    val commonSettings = Seq(
      organization := "net.example",
      version := "0.1",
      scalaVersion := "2.11.4",
      scalacOptions := Seq("-unchecked", "-deprecation", "-encoding", "utf8")
    )

    lazy val testDependencies = Seq (
      "org.scalatest" %% "scalatest" % "2.2.0" % "test"
    )
    
    lazy val common = project.in(file("common"))
      .settings(commonSettings:_*)
      .settings(libraryDependencies ++= testDependencies)
    
    lazy val sprayApp = project.in(file("sprayApp"))
      .settings(commonSettings:_*)
    
    lazy val playApp = project.in(file("playApp"))
      .settings(commonSettings:_*)
    
    lazy val main = project.in(file("."))
      .aggregate(common, sprayApp, playApp)

Now, starting `sbt`, switching to `project common`, and running `test` will of course still yield an error, because the
spec is against a still missing implementation. Let's change that by putting the following into
`common/src/test/scala/common/utils/cassandra/CassandraConnectionUri.scala`:

    package common.utils.cassandra
    
    import java.net.URI
    
    case class CassandraConnectionUri(connectionString: String) {
    
      private val uri = new URI(connectionString)
    
      private val additionalHosts = Option(uri.getQuery) match {
        case Some(query) => query.split('&').map(_.split('=')).filter(param => param(0) == "host").map(param => param(1)).toSeq
        case None => Seq.empty
      }
    
      val host = uri.getHost
      val hosts = Seq(uri.getHost) ++ additionalHosts
      val port = uri.getPort
      val keyspace = uri.getPath.substring(1)
    
    }

Running `test` in the sbt console will now result in a first successful test run:

    $ sbt
    [info] Set current project to My Project (in build file:/Users/manuelkiessling/myproject/)
    > project common
    [info] Set current project to common (in build file:/Users/manuelkiessling/myproject/)
    > test
    [info] Compiling 1 Scala source to /Users/manuelkiessling/myproject/common/target/scala-2.11/test-classes...
    [info] CassandraConnectionUriSpec:
    [info] A Cassandra connection URI object
    [info] - should parse a URI with a single host
    [info] - should parse a URI with additional hosts
    [info] Run completed in 352 milliseconds.
    [info] Total number of tests run: 2
    [info] Suites: completed 1, aborted 0
    [info] Tests: succeeded 2, failed 0, canceled 0, ignored 0, pending 0
    [info] All tests passed.
    [success] Total time: 6 s, completed 02.01.2015 11:36:45
