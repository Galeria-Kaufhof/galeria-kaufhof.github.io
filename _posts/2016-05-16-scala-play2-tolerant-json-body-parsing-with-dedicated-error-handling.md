---
layout: post
title: "Scala Play2: Tolerant JSON body parsing with dedicated error handling"
description: "For one of my current projects, I needed to gain full control over the request-body-to-case-class-object transformation of my Scala Play2 controller action. Here’s the solution I came up with."
category: tutorials
author: manuelkiessling
tags: [scala, play2, json]
---
{% include JB/setup %}


<p>
I'm currently rewriting a Scala Play2 based web service that employs the following body parser:
</p>

<p>
<pre><code>def tolerantJsonParser[A](implicit reader: Reads[A]): BodyParser[A] =
  parse.tolerantJson.validate(json =>
    json.validate[A].asEither.left.map(err => Results.BadRequest)
  )(play.api.libs.iteratee.Execution.Implicits.trampoline)

def doSomething = Action.async(tolerantJsonParser[SomeThing]) { request =>
  val someThing: SomeThing = request.body
  ...</code></pre>
</p>

<p>
Not shown here is the implicit <code class="inline">Reads</code> that takes care of transforming the Json object into an object of case class <code class="inline">SomeThing</code> when the <code class="inline">json.validate</code> method is called.
</p>

<p>
As you can see, the existing code took care of handling the incoming Json in a tolerant manner - that is, it didn't bail out if the media type of the body is not <code class="inline">application/json</code>, which is what Play does per default, but which is not what we want in this case because clients send requests with a more specific media type to this webservice.
</p>

<p>
In case that the Json object to case class transformation fails, the body parser correctly answers the request with a <code class="inline">400 Bad Request</code> response status code. This covers all cases where the body is valid Json, but cannot be mapped to the structure of the case class using the implicit Reads.
</p>

<p>
Another case is implicitly covered, too - if the request body isn't even Json to begin with (e.g. because a <code class="inline">{</code> is missing, as in <code class="inline">"foo":"bar"}</code>), then <code class="inline">parse.tolerantJson</code> fails, resulting in a failure response, too.
</p>

<p>
However, the latter case is handled by Play2, resulting in a generic HTML error response - but for the rewrite, I wanted to have dedicated error handling because my goal was to send a specific Json encoded error response.
</p>

<p>
The solution turned out to be quite simple (which didn't stop me from taking several hours to come up with it) - by parsing the Json myself using the <code class="inline">parse.tolerantText</code> body parser, I gained full control over the body parsing process, which allowed me to react to errors in both steps in the process - the text-to-json transformation as well as the json-to-case-class-object transformation:
</p>

<p>
<pre><code>def tolerantTryJsonParser[A](implicit reader: Reads[A]): BodyParser[Try[A]] = {
  parse.tolerantText.map { text =>
    Try {
      Json.parse(text).validate.get
    }
  }
}

def doSomething = Action.async(tolerantTryJsonParser[SomeThing]) { request =>
  request.body match {
    case Success(something) => ...
    case Failure(error) => ...
  }</code></pre>
</p>
