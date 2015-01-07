---
layout: post
title: "Setting up a Scala sbt multi project with Cassandra connectivity and migrations, Part 1"
description: ""
category: tutorials
author: manuelkiessling
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
(one being plain old Scala, one being based on Play2, and one on Spray) using `sbt`. I'll also describe how to enable
all three modules to speak with an Apache Cassandra database, and how to add automatically applied database migrations
(in order to allow using the project in a Continuous Delivery setup like the one we use at Galeria Kaufhof).

Part 1 (which you are currently reading) covers the basic multi-project setup and describes how to get up and running
with Cassandra migrations. Part 2 (which will be published in the near future) will discuss how to integrate a Play and
a Spray application into the setup, respectively.

This tutorial is well suited for reasonably experienced software developers with some first basic experience in Scala
and Cassandra.


## Prerequisites

The project setup we are going to create is based on Java 8, Scala 2.11, sbt 0.13 and Cassandra 2.1.

You'll need a working Scala environment (on Mac OS X with Homebrew, `brew install scala sbt` should do the job). You'll
also need a running and reachable Cassandra installation - setting this up is out of the
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
          val cut = CassandraConnectionUri("cassandra://localhost:9042/test")
          cut.host should be ("localhost")
          cut.hosts should be (Seq("localhost"))
          cut.port should be (9042)
          cut.keyspace should be ("test")
        }
        it("should parse a URI with additional hosts") {
          val cut = CassandraConnectionUri(
            "cassandra://localhost:9042/test" +
              "?host=otherhost.example.net" +
              "&host=yet.anotherhost.example.com")
          cut.hosts should contain allOf ("localhost", "otherhost.example.net", "yet.anotherhost.example.com")
          cut.port should be (9042)
          cut.keyspace should be ("test")
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

The next step is to add code that actually connects to Cassandra. This will lay the ground for adding automatic
database migrations to the project. But first things first. Here is the code for an object which provides a method
that returns a Cassandra database connection session; put it into
`common/src/test/scala/common/utils/cassandra/Helper.scala`:

    package common.utils.cassandra

    import com.datastax.driver.core._

    object Helper {

      def createSessionAndInitKeyspace(uri: CassandraConnectionUri,
                                       defaultConsistencyLevel: ConsistencyLevel = QueryOptions.DEFAULT_CONSISTENCY_LEVEL) = {
        val cluster = new Cluster.Builder().
          addContactPoints(uri.hosts.toArray: _*).
          withPort(uri.port).
          withQueryOptions(new QueryOptions().setConsistencyLevel(defaultConsistencyLevel)).build

        val session = cluster.connect
        session.execute(s"USE ${uri.keyspace}")
        session
      }

    }

This is a relatively straight-forward implementation which can be used like this:

    import common.utils.cassandra._
    
    val uri = CassandraConnectionUri("cassandra://localhost:9042/test")
    val session = Helper.createSessionAndInitKeyspace(uri)
    
    session.execute(/* Some CQL string */)

Let's see how we can make use of this in a spec in order to prove that we can actually talk to our database. To do so,
please create the file `common/src/test/scala/common/utils/cassandra/ConnectionAndQuerySpec.scala` and add the following
code:

    package common.utils.cassandra
    
    import com.datastax.driver.core.querybuilder.QueryBuilder
    import com.datastax.driver.core.querybuilder.QueryBuilder._
    import org.scalatest.{Matchers, FunSpec}
    
    class ConnectionAndQuerySpec extends FunSpec with Matchers {
      
      describe("Connecting and querying a Cassandra database") {
        it("should just work") {
          val uri = CassandraConnectionUri("cassandra://localhost:9042/test")
          val session = Helper.createSessionAndInitKeyspace(uri)
          
          session.execute("CREATE TABLE IF NOT EXISTS things (id int, name text, PRIMARY KEY (id))")
          session.execute("INSERT INTO things (id, name) VALUES (1, 'foo');")
    
          val selectStmt = select().column("name")
            .from("things")
            .where(QueryBuilder.eq("id", 1))
            .limit(1)
          
          val resultSet = session.execute(selectStmt)
          val row = resultSet.one()
          row.getString("name") should be("foo")
          session.execute("DROP TABLE things;")
        }
      }
    
    }

Now, there is one thing that needs to be done that is outside of the scope of our code, and that is the creation of the
Cassandra keyspace. Please do this manually using the `cqlsh`:

    CREATE KEYSPACE IF NOT EXISTS test WITH replication = { 'class': 'SimpleStrategy', 'replication_factor': 1 };

Running `test` once again from within `sbt` should yield a successful spec run.

We are now at a point where we can add automatic database migrations for Cassandra. We will use an existing library for
this named [Pillar](https://github.com/comeara/pillar), and we'll add some wrapper and helper code in order to integrate
it into our project.

The first thing to do is to add the *Pillar* library as a dependency in our `build.sbt` file. This results in the
*cassandraDependencies* block being changed from

    lazy val cassandraDependencies = Seq (
      "com.datastax.cassandra" % "cassandra-driver-core" % "2.1.2"
    )

to

    lazy val cassandraDependencies = Seq (
      "com.datastax.cassandra" % "cassandra-driver-core" % "2.1.2",
      "com.chrisomeara" % "pillar_2.11" % "2.0.1"
    )

Next, we need some helper code. What we need is a method which allows to read the contents of a directory, no matter
if this directory resides in the file system or within a JAR file. Greg Briggs has written a nice little class for this
which we are going to use:

    package common.utils.cassandra;
    
    import java.io.File;
    import java.io.IOException;
    import java.net.URISyntaxException;
    import java.net.URL;
    import java.net.URLDecoder;
    import java.util.Enumeration;
    import java.util.HashSet;
    import java.util.Set;
    import java.util.jar.JarEntry;
    import java.util.jar.JarFile;
    
    public class JarUtils {
    
        /**
         * List directory contents for a resource folder. Not recursive.
         * This is basically a brute-force implementation.
         * Works for regular files and also JARs.
         *
         * @param clazz Any java class that lives in the same place as the resources you want.
         * @param path  Should end with "/", but not start with one.
         * @return Just the name of each member item, not the full paths.
         * @throws java.net.URISyntaxException
         * @throws java.io.IOException
         * @author Greg Briggs
         */
        public static String[] getResourceListing(Class clazz, String path) throws URISyntaxException, IOException {
            URL dirURL = clazz.getClassLoader().getResource(path);
            if (dirURL != null && dirURL.getProtocol().equals("file")) {
            /* A file path: easy enough */
                return new File(dirURL.toURI()).list();
            }
    
            if (dirURL == null) {
            /*
             * In case of a jar file, we can't actually find a directory.
             * Have to assume the same jar as clazz.
             */
                String me = clazz.getName().replace(".", "/") + ".class";
                dirURL = clazz.getClassLoader().getResource(me);
            }
    
            if (dirURL.getProtocol().equals("jar")) {
            /* A JAR path */
                String jarPath = dirURL.getPath().substring(5, dirURL.getPath().indexOf("!")); //strip out only the JAR file
                JarFile jar = new JarFile(URLDecoder.decode(jarPath, "UTF-8"));
                Enumeration<JarEntry> entries = jar.entries(); //gives ALL entries in jar
                Set<String> result = new HashSet<String>(); //avoid duplicates in case it is a subdirectory
                while (entries.hasMoreElements()) {
                    String name = entries.nextElement().getName();
                    if (name.startsWith(path)) { //filter according to the path
                        String entry = name.substring(path.length());
                        int checkSubdir = entry.indexOf("/");
                        if (checkSubdir >= 0) {
                            // if it is a subdirectory, we just return the directory name
                            entry = entry.substring(0, checkSubdir);
                        }
                        result.add(entry);
                    }
                }
                return result.toArray(new String[result.size()]);
            }
    
            throw new UnsupportedOperationException("Cannot list files for URL " + dirURL);
        }
    
    }

Note that this is Java code, not Scala. We therefore need to put it into
`common/src/main/java/common/utils/cassandra/JarUtils.java`.

Next up is some wrapper code that gives us an easy to handle *Pillar* object to work with from our own code. This one
goes into `common/src/main/scala/common/utils/casssandra/Pillar.scala`:

    package common.utils.cassandra
    
    import com.chrisomeara.pillar._
    import com.datastax.driver.core.Session
    
    object Pillar {
    
      private val registry = Registry(loadMigrationsFromJarOrFilesystem())
      private val migrator = Migrator(registry)
    
      private def loadMigrationsFromJarOrFilesystem() = {
        val migrationsDir = "migrations/"
        val migrationNames = JarUtils.getResourceListing(getClass, migrationsDir).toList.filter(_.nonEmpty)
        val parser = Parser()
    
        migrationNames.map(name => getClass.getClassLoader.getResourceAsStream(migrationsDir + name)).map {
          stream =>
            try {
              parser.parse(stream)
            } finally {
              stream.close()
            }
        }.toList
      }
    
      def initialize(session: Session, keyspace: String, replicationFactor: Int): Unit = {
        migrator.initialize(
          session,
          keyspace,
          new ReplicationOptions(Map("class" -> "SimpleStrategy", "replication_factor" -> replicationFactor))
        )
      }
    
      def migrate(session: Session): Unit = {
        migrator.migrate(session)
      }
    }

We can now approach running our very first migration. Let's create the file carrying the migration statements first. It
goes into `/common/src/main/resources/migrations/1_create_things_table.cql`:

    -- description: create status structure
    -- authoredAt: 1418718253000
    -- up:

    CREATE TABLE things (
      id int,
      name text,
      PRIMARY KEY (id)
    );

    -- down:

    DROP TABLE things;

Note that the `authoredAt` field is important. Migrations are applied in ascending order according to the value of this
field. Its format is *milliseconds since start of the Unix epoch*.

Now we need a place where migrations are actually applied. One solution is to do this right after connecting
to the database, which might not be the most sensible solution in a production environment, but does the job for the
scope of this tutorial.

We achieve this by simply adding two lines to our *Helper* object in
`common/src/main/scala/common/utils/casssandra/Helper.scala`:

    package common.utils.cassandra

    import com.datastax.driver.core._

    object Helper {

      def createSessionAndInitKeyspace(uri: CassandraConnectionUri,
                                       defaultConsistencyLevel: ConsistencyLevel = QueryOptions.DEFAULT_CONSISTENCY_LEVEL) = {
        val cluster = new Cluster.Builder().
          addContactPoints(uri.hosts.toArray: _*).
          withPort(uri.port).
          withQueryOptions(new QueryOptions().setConsistencyLevel(defaultConsistencyLevel)).build

        val session = cluster.connect
        session.execute(s"USE ${uri.keyspace}")

        Pillar.initialize(session, uri.keyspace, 1)
        Pillar.migrate(session)

        session
      }

    }

We can now change our *ConnectionAndQuery* spec in file
`common/src/test/scala/common/utils/cassandra/ConnectionAndQuerySpec.scala`, because we no longer need to create the
table we test against within the spec itself - our migration will take care of this:

    package common.utils.cassandra
    
    import com.datastax.driver.core.querybuilder.QueryBuilder
    import com.datastax.driver.core.querybuilder.QueryBuilder._
    import org.scalatest.{Matchers, FunSpec}
    
    class ConnectionAndQuerySpec extends FunSpec with Matchers {
      
      describe("Connecting and querying a Cassandra database") {
        it("should just work") {
          val uri = CassandraConnectionUri("cassandra://localhost:9042/test")
          val session = Helper.createSessionAndInitKeyspace(uri)
          
          session.execute("INSERT INTO things (id, name) VALUES (1, 'foo');")
    
          val selectStmt = select().column("name")
            .from("things")
            .where(QueryBuilder.eq("id", 1))
            .limit(1)
          
          val resultSet = session.execute(selectStmt)
          val row = resultSet.one()
          row.getString("name") should be("foo")
          session.execute("TRUNCATE things;")
        }
      }
    
    }