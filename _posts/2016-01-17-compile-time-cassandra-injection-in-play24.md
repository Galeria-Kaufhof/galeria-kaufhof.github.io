---
layout: post
title: "Compile Time Cassandra Injection in Play 2.4"
description: "Play 2.4 supports Compile Time Dependency Injection. This post describes how to inject your own Cassandra repository object into a controller at compile time, while also initializing and closing a Cassandra connection session during application startup and shutdown, and how to mock the repository in the non-integration tests."
category: tutorials
author: manuelkiessling
tags: [scala, play2, cassandra]
---
{% include JB/setup %}


<h2>About</h2>
<p>
<a href="https://www.playframework.com/documentation/2.4.x/Home">Play 2.4</a> supports <a href="https://www.playframework.com/documentation/2.4.x/ScalaCompileTimeDependencyInjection">Compile Time Dependency Injection</a>. This post describes how to inject your own Cassandra repository object into a controller at compile time, while also initializing and closing a Cassandra connection session during application startup and shutdown, respectively.
</p>
<p>
The code of the final application is available at <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di">https://github.com/manuelkiessling/play2-compiletime-cassandra-di</a>.
</p>

<h2>The goal</h2>
<p>
At the end of this post, we have created a small Play 2.4.6 Scala application with which will be able to serve the name of a <em>product</em> with a given <em>id</em> by reading information from a Cassandra database. We will be able to verify the correct behaviour of our application using a real database with an integration test and, because we will be able to mock the repository that is injected into the controller, we will also be able to verify the correct application behaviour without using a database.
</p>
<p>
The resulting code will be runnable and realistic, but also a bit simplistic - lacking error handling, for example - in order to be tractable.
</p>

<h2>Prerequisites</h2>
<p>
This post is aimed at readers who have already written Scala applications with Play2 and know how to work with sbt and Cassandra.
</p>
<p>
In order to compile and run the code, you <a href="https://www.playframework.com/documentation/2.4.x/Migration24#Java-8-support">need</a> a <a href="http://www.oracle.com/technetwork/java/javase/downloads/index.html">Java 8 SE Development Kit</a>, and you need a recent version of <a href="http://www.scala-sbt.org/download.html">sbt</a>.
</p>
<p>
Last but not least, you need to <a href="https://wiki.apache.org/cassandra/GettingStarted">set up a Cassandra cluster</a> - a one-node local setup is sufficient for the application that we'll create.
</p>
<p>
The post is written from the perspective of a Mac OS X system user with <a href="http://brew.sh/">Homebrew</a> installed, but should be adaptable for any Scala-capable environment with minor modifications.
</p>

<h2>Project setup</h2>
<p>
Let's start by creating an sbt-based Play 2.4 project using the <a href="https://www.typesafe.com/community/core-tools/activator-and-sbt">Typesafe Activator</a>, which we install via Homebrew: <code class="inline">brew install typesafe-activator</code>.
</p>
<p>
We can then use Activator to set up the Play2 project: <code class="inline">activator new play2-compiletime-cassandra-di play-scala</code>.
</p>
<p>
The first thing to do now is to switch from specs2 to ScalaTest as our testing framework, as described in <a href="http://manuel.kiessling.net/2015/12/31/play2-switching-from-specs2-to-scalatest">Play2: Switching from specs2 to ScalaTest</a>. Please change files <code class="inline">build.sbt</code>, <code class="inline">test/ApplicationSpec.scala</code>, and <code class="inline">test/IntegrationSpec.scala</code> as described there.
</p>
<p>
Running <code class="inline">sbt test</code> afterwards should just work. At this point, your codebase should look like <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/tree/3a96b61cf4445d80e3563e66d6e319c80eb8afc4">the reference repository at 3a96b61</a>.
</p>

<h2>Introducing the Cassandra driver</h2>

We are now going to integrate the <a href="https://github.com/datastax/java-driver">Datastax Java Driver for Apache Cassandra</a>, roughly following the steps outlined in [Setting up a Scala sbt multi-project with Cassandra connectivity and migrations]({% post_url 2015-01-14-setting-up-a-scala-sbt-multi-project-with-cassandra-connectivity-and-migrations %}) (but without tests and the migrations stuff to keep the codebase small for this post).

<p>
This means adding the Cassandra driver as a dependency to file <code class="inline">build.sbt</code>, creating a utility class for connection URIs in file <code class="inline">app/cassandra/CassandraConnectionUri.scala</code>, and adding an object that handles database connections in file <code class="inline">app/cassandra/CassandraConnector.scala</code>. See <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/tree/0fa30e4874b1398f1557dbddd2346393d15c41e3">the resulting codebase on GitHub at 0fa30e4</a> or view <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/commit/0fa30e4874b1398f1557dbddd2346393d15c41e3?diff=split">the differences from the previous version of the codebase</a>.
</p>

<h2></h2>
<p>
With this, we can finally start to work on the actual Cassandra repository and learn how to inject it into a controller.
</p>

<p>
Again, I'm trying to keep things simple and the codecase lean. We will inject the repository into the existing <code class="inline">Application</code> controller in file <code class="inline">app/controllers/Application.scala</code>.
</p>

<p>
Our repository has only one job: It allows controllers to retrieve exactly one row from the Cassandra table it manages using a single-column primary key. Imagine we have a Cassandra table called <em>products</em>, with the following structure:
</p>

<p>
<pre><code>+----+-------+
| id | name  |
+----+-------+
|  1 | Chair |
|  2 | Fork  |
|  3 | Lamp  |
+----+-------+</code></pre>
</p>

<p>
We can create the according table structure (and its keyspace) as follows:
</p>

<p>
<pre><code>CREATE KEYSPACE IF NOT EXISTS test WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1 };
USE test;
CREATE TABLE products (id INT PRIMARY KEY, name TEXT);</code></pre>
</p>

<p>
Let's make some assumptions: We expect our repository to provide a method <code class="inline">getOneById(id: Int): ProductModel</code>. Given a product id, the repository takes care of the heavy lifting that is required to return a <code class="inline">ProductModel</code> object which carries the data for this product as retrieved from the database.
</p>

<p>
We can start by declaring the model. Its a simple case class that lives in <code class="inline">app/models/ProductModel.scala</code>:
</p>

<p>
<pre><code>package models

case class ProductModel(id: Int, name: String)
</code></pre>
</p>

<p>
Next, we start building a repository structure which keeps the API towards clients of repositories abstract, and allows to create concrete implementations that work with a Cassandra database.
</p>

<p>
The first step is to define a trait that all repositories, be it concrete Cassandra-based implementations or lightweight mocks in test, will share. To do so, we put the following in file <code class="inline">app/repositories/Repository.scala</code>:
</p>

<p>
<pre><code>package repositories

abstract trait Repository[M, I] {
  def getOneById(id: I): M
}
</code></pre>
</p>

<p>
Simple enough. This ensures that repository implementations will provide a <code class="inline">getOneById</code> method, and type parametrization allows to declare the type of parameter id and the type of the model object that has to be returned.
</p>

<p>
We know that we have to query the repository for integer values, and we know that we want to retrieve a <code class="inline">ProductModel</code> in return. Thus, we can already declare the fact that our Application controller depends on such a repository, in file <code class="inline">app/controllers/Application.scala</code>:
</p>

<p>
<pre><code>package controllers

import models.ProductModel
import play.api._
import play.api.mvc._
import repositories.Repository

class Application(productsRepository: Repository[ProductModel, Int]) extends Controller {

  def index = Action {
    Ok(views.html.index("Your new application is ready."))
  }

}
</code></pre>
</p>

<p>
At this point (<a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/tree/d58d681c07b9aec98a96a568b7d2a8f48b6954ab">commit d58d681 in the repo</a>, <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/commit/d58d681c07b9aec98a96a568b7d2a8f48b6954ab?diff=split">diff</a>), the application can no longer run and the existing test cases fail, because we have not yet implemented the mechanisms needed to actually inject the declared dependency into the controller. We will fix this later - first, we write a generic Cassandra repository implementation and use it to create a concrete implementation for the <code class="inline">Repository[ProductModel, Int]</code> type.
</p>

<p>
To do so, we create an abstract <code class="inline">CassandraRepository</code> class that does the heavy lifting, in file <code class="inline">app/repositories/CassandraRepository.scala</code>:
</p>

<p>
<pre><code>package repositories

import com.datastax.driver.core.querybuilder.QueryBuilder
import com.datastax.driver.core.querybuilder.QueryBuilder._
import com.datastax.driver.core.{Row, Session}

abstract class CassandraRepository[M, I](session: Session, tablename: String, partitionKeyName: String)
  extends Repository[M, I] {
  def rowToModel(row: Row): M

  def getOneRowBySinglePartitionKeyId(partitionKeyValue: I): Row = {
    val selectStmt =
      select()
        .from(tablename)
        .where(QueryBuilder.eq(partitionKeyName, partitionKeyValue))
        .limit(1)

    val resultSet = session.execute(selectStmt)
    val row = resultSet.one()
    row
  }

  override def getOneById(id: I): M = {
    val row = getOneRowBySinglePartitionKeyId(id)
    rowToModel(row)
  }
}
</code></pre>
</p>

<p>
Some notes on this class: in addition to the type parameters for the model and id field, a concrete class that extends this abstract CassandraRepository needs to provide a session representing the connection to a cassandra cluster, the name of the table that is to be wrapped, and the name of the field that is to be queried via getOneById. Furthermore, implementations must override the <code class="inline">rowToModel</code> method, because only a concrete implementation knows how to create a valid model from the values of a table row.
<br>
Finally, <code class="inline">getOneRowBySinglePartitionKeyId</code> is where stuff happens: using the session and the means provided by the DataStax driver, the database is queried and the resulting row is returned. Because CassandraRepository extends the Repository trait, it must override <code class="inline">getOneById</code> - in this case, that's simply a matter of retrieving the row using the code from the abstract class itself, and transforming it into a model using the to-be-overridden rowToModel method.
</p>

<p>
With this in place, the concrete implementation of a Cassandra-backed <code class="inline">ProductsRepository</code> class that matches the <code class="inline">Repository[ProductModel, Int]</code> type looks like this, in file <code class="inline">app/repositories/ProductsRepository.scala</code>:
</p>

<p>
<pre><code>package repositories

import com.datastax.driver.core.{Row, Session}
import models.ProductModel

class ProductsRepository(session: Session)
  extends CassandraRepository[ProductModel, Int](session, "products", "id") {
  override def rowToModel(row: Row): ProductModel = {
    ProductModel(
      row.getInt("id"),
      row.getString("name")
    )
  }
}
</code></pre>
</p>

<p>
With these changes in place (<a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/tree/cb7a9b48c470f277e86b21b80a7f7ca540335981">commit cb7a9b4 in the repo</a>, <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/commit/cb7a9b48c470f277e86b21b80a7f7ca540335981?diff=split">diff</a>), we can approach the injection. How can we control the way the <code class="inline">Application</code> controller is created, i.e. how can we control compile time dependency injection? In Play 2.4, this happens by providing a class that extends the <code class="inline">play.api.ApplicationLoader</code> trait.
</p>

<p>
To do so, create file <code class="inline">app/AppLoader.scala</code> with the following content:
</p>

<p>
<pre><code>import components.CassandraRepositoryComponents
import play.api.ApplicationLoader.Context
import play.api.routing.Router
import play.api.{Application, ApplicationLoader, BuiltInComponentsFromContext}
import router.Routes

class AppLoader extends ApplicationLoader {
  override def load(context: ApplicationLoader.Context): Application =
    new AppComponents(context).application
}

class AppComponents(context: Context) extends BuiltInComponentsFromContext(context) with CassandraRepositoryComponents {

  lazy val applicationController = new controllers.Application(productsRepository)
  lazy val assets = new controllers.Assets(httpErrorHandler)

  override def router: Router = new Routes(
    httpErrorHandler,
    applicationController,
    assets
  )
}
</code></pre>
</p>

<p>
Play2 is not able to find out about this AppLoader itself, which is why we need to configure it in file <code class="inline">conf/application.conf</code> by adding the line <code class="inline">play.application.loader="AppLoader"</code>.
</p>

<p>
As you can see, we are overriding the part of Play2 that creates a runnable <code class="inline">play.api.Application</code> by instantiating the Application controller ourselves (while injecting the products repository, more on this later), and by overriding router creation. In order to get access to a products repository instance, the AppComponents class extends the <code class="inline">CassandraRepositoryComponents</code> trait. Within this trait, we connect to the database, set up the repository, and hook into the application lifecycle in order to shut down the database connection when the application shuts down. The code for all this goes into <code class="inline">app/components/CassandraRepositoryComponents.scala</code>:
</p>

<p>
<pre><code>package components

import cassandra.{CassandraConnector, CassandraConnectionUri}
import com.datastax.driver.core.Session
import models.ProductModel
import play.api.inject.ApplicationLifecycle
import play.api.{Configuration, Environment, Mode}
import repositories.{Repository, ProductsRepository}
import scala.concurrent.Future

trait CassandraRepositoryComponents {
  // These will be filled by Play's built-in components; should be `def` to avoid initialization problems
  def environment: Environment
  def configuration: Configuration
  def applicationLifecycle: ApplicationLifecycle

  lazy private val cassandraSession: Session = {
    val uriString = environment.mode match {
      case Mode.Test => "cassandra://localhost:9042/test"
      case _         => "cassandra://localhost:9042/prod"
    }
    val session: Session = CassandraConnector.createSessionAndInitKeyspace(
      CassandraConnectionUri(uriString)
    )
    // Shutdown the client when the app is stopped or reloaded
    applicationLifecycle.addStopHook(() => Future.successful(session.close()))
    session
  }

  lazy val productsRepository: Repository[ProductModel, Int] = {
    new ProductsRepository(cassandraSession)
  }
}
</code></pre>
</p>

<p>
As you can see, this approach also allows to adapt to the application environment: In our case, we connect to a different Cassandra cluster URI if we are running in the test environment.
</p>

<p>
At this point (<a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/tree/4e707cb784e7adbc7c8204f033404690986c9448">commit 4e707cb</a>, <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/commit/4e707cb784e7adbc7c8204f033404690986c9448?diff=split">diff</a>, dependency injection is in place and the application is runnable again.
</p>

<p>
What still doesn't run, however, are the tests. Also, it's a bit sad that we did all those things that give our controller a repository, and then it doesn't even use it. Let's fix both issues.
</p>

<p>
We start with the integration spec in file <code class="inline">test/IntegrationSpec.scala</code>. We are going to extend this quite a bit:
</p>

<p>
<pre><code>import java.io.File
import cassandra.{CassandraConnector, CassandraConnectionUri}
import org.scalatest.BeforeAndAfter
import org.scalatestplus.play._
import play.api
import play.api.{Mode, Environment, ApplicationLoader}

class IntegrationSpec extends PlaySpec with OneBrowserPerSuite with OneServerPerSuite with HtmlUnitFactory with BeforeAndAfter {

  before {
    val uri = CassandraConnectionUri("cassandra://localhost:9042/test")
    val session = CassandraConnector.createSessionAndInitKeyspace(uri)
    val query = "INSERT INTO products (id, name) VALUES (1,  'Chair');"
    session.execute(query)
    session.close()
  }

  override implicit lazy val app: api.Application =
    new AppLoader().load(
      ApplicationLoader.createContext(
        new Environment(
          new File("."), ApplicationLoader.getClass.getClassLoader, Mode.Test)
      )
    )

  "Application" should {

    "work from within a browser and tell us about the first product" in {

      go to "http://localhost:" + port

      pageSource must include ("Your new application is ready. The name of product #1 is Chair.")
    }
  }
}
</code></pre>
</p>

<p>
We need to override the implicit <code class="inline">app</code> value, where we ask our new AppLoader to create an application for the <em>Test</em> environment. Furthermore, we extend our specification and now expect our app to not only greet us, but to also tell us about the name of the product with ID 1, which we insert in the new <em>before</em> step of our specification.
</p>

<p>
We could do the same with the ApplicationSpec, but let's go one step further and mock the ProductRepository, which gives us a specification that in contrast to the integration spec doesn't need an actual Cassandra database to work, and only verifies the behaviour of the Application controller itself, not its dependencies. To do so, we change file <code class="inline">test/ApplicationSpec.scala</code> as follows:
</p>

<p>
<pre><code>import java.io.File
import models.ProductModel
import play.api
import play.api.{Mode, Environment, ApplicationLoader}
import play.api.ApplicationLoader.Context
import play.api.test._
import play.api.test.Helpers._
import org.scalatestplus.play._
import repositories.Repository

class MockProductsRepository extends Repository[ProductModel, Int] {
  override def getOneById(id: Int): ProductModel = {
    ProductModel(1, "Mocked Chair")
  }
}

class FakeApplicationComponents(context: Context) extends AppComponents(context) {
  override lazy val productsRepository = new MockProductsRepository()
}

class FakeAppLoader extends ApplicationLoader {
  override def load(context: Context): api.Application =
    new FakeApplicationComponents(context).application
}

class ApplicationSpec extends PlaySpec with OneAppPerSuite {

  override implicit lazy val app: api.Application = {
    val appLoader = new FakeAppLoader
    appLoader.load(
      ApplicationLoader.createContext(
        new Environment(
          new File("."), ApplicationLoader.getClass.getClassLoader, Mode.Test)
      )
    )
  }

  "Application" should {

    "send 404 on a bad request" in {
      val Some(wrongRoute) = route(FakeRequest(GET, "/boum"))

      status(wrongRoute) mustBe NOT_FOUND
    }

    "render the index page and tell us about the first product" in {
      val Some(home) = route(FakeRequest(GET, "/"))

      status(home) mustBe OK
      contentType(home) mustBe Some("text/html")
      contentAsString(home) must include ("Your new application is ready. The name of product #1 is Mocked Chair.")
    }
  }

}
</code></pre>
</pre>
</p>

<p>
Of course, both specs will only pass if we change the behaviour of the Application controller in file <code class="inline">app/controllers/Application.scala</code> and make us of the injected repository:
</p>

<p>
<pre><code>package controllers

import models.ProductModel
import play.api._
import play.api.mvc._
import repositories.Repository

class Application(productsRepository: Repository[ProductModel, Int]) extends Controller {

  def index = Action {
    val product = productsRepository.getOneById(1)
    Ok(views.html.index(s"Your new application is ready. The name of product #1 is ${product.name}."))
  }

}
</code></pre>
</p>

<p>
And that's it. At <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/tree/796b6c64ea88a160481069261e1474ec36103072">commit 796b6c6</a>, (see <a href="https://github.com/manuelkiessling/play2-compiletime-cassandra-di/commit/796b6c64ea88a160481069261e1474ec36103072?diff=split">the diff</a>), we have a working Play 2.4 app with a compile time injected Cassandra repository.
</p>
