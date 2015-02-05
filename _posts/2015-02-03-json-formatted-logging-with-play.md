---
layout: post
title: "JSON Formatted Logging With Play"
description: ""
category:
tags: [scala,play,logging]
---
{% include JB/setup %}

The [multi-channel retailing platform we are building at GALERIA Kaufhof](http://www.startplatz.de/event/galeria-sucht-hacker/) provides
centralised logging for all deployed applications and services. In order to leverage this common logging facility for 
upcoming metrics and analytics use cases we picked JSON as the agreed upon log format across all
domains.

We are primarily using Scala and the Play Framework for our application development and
therefore had to get Play to log in JSON. This note briefly describes our solution.

## Logback JSON Encoder

The logstash team provides a [JSON formatter for logback logs](https://github.com/logstash/logstash-logback-encoder). You can
add this to your sbt project as the following dependency in the build.sbt:

    libraryDependencies += "net.logstash.logback" % "logstash-logback-encoder" % "4.0"

Then configure logging in logger.xml with the minimal addition of this encoder to
the desired appender:

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    </appender>

Logback will now start to write one-line JSON documents to your log file, properly escaping double quotes
and newlines.

Just logging a bunch of rather technical fields and one field 'message' containing what you
actually log in the code doesn't make for a good analytics harvest later on. Let's try to
add some structure to our logs.

The facility Logback JSON Encoder uses to add user defined fields to the emitted JSON are
[slf4j Markers](http://slf4j.org/api/org/slf4j/Marker.html):

    import net.logstash.logback.marker.Markers._
    import scala.collection.JavaConversions._ // appendEntries() expects a Java Map
    
    logger.info(append("name", "value"), "log message")
    
    logger.info(append("name1", "value1").and(append("name2", "value2")), "log message")
    
    val fields = Map( "productId" -> "123456" , "traceId" -> "98765" )
    logger.info(appendEntries(fields), "log message")

See [Event-Specific Custom Fields](https://github.com/logstash/logstash-logback-encoder#loggingevent_custom_event)

## Play Logging with Markers - the Built-In Logger

Given we are using Play, the straight forward way to get hold of a Logger instance is
to import Play's play.api.Logger:

    import play.api.Logger
    import net.logstash.logback.marker.Markers._
    
    Logger.info(append("name", "value"), "log message")

When you try to run this, the compiler will complain that Logger.info isn't applicable to
Marker. The reason for this is that [LoggerLike](https://github.com/playframework/playframework/blob/master/framework/src/play/src/main/scala/play/api/Logger.scala#L15)
simply does not define these corresponding methods of the underying Logger interface.

One solution is to augment LoggerLike with an implicit trait, but we found that this messes up the
source location information in error log events. Not good.

## Logging with Markers - without Play

An alternative to augmenting Play's logger is to use a logging library that properly implements
the Marker-variants of the log methods.

Adding [Scala Logging](https://github.com/typesafehub/scala-logging) to the project solved the
issue right away:

    libraryDependencies += "com.typesafe.scala-logging" %% "scala-logging" % "3.1.0"

In addition Scala Logging comes with nice features such as the
[LazyLogging](https://github.com/typesafehub/scala-logging#using-scala-logging) trait.

## Verbosity...

Logstash JSON Encoder is a little verbose (to say the least):

    {"@timestamp":"2015-02-05T08:13:48.566+01:00","@version":1,"message":"What we logged...","logger_name":"application", \
      "thread_name":"application-akka.actor.default-dispatcher-11","level":"INFO","level_value":20000,"HOSTNAME":"myhost.mydomain", \
      "application.home":"/Users/.../universal/stage"}

Besides reducing the amount of data logged we also wanted to rename some fields to avoid
overlap with fields added by other tools in the logging chain. We also wanted to shorten
the names.

Logstash JSON Encoder [lets you do this](https://github.com/logstash/logstash-logback-encoder#custom_field_names)
 in the logger configuration. For example:

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <fieldNames>
          <timestamp>ts</timestamp>
          <message>msg</message>
          <thread>[ignore]</thread>
          <levelValue>[ignore]</levelValue>
          <logger>logger</logger>
          <version>[ignore]</version>
        </fieldNames>
      </encoder>
    </appender>

It took some digging to find out the corresponding XML tag names but the above removes
most of the clutter.  Unfortunately [Logback Access](http://logback.qos.ch/access.html)
adds some [field names of its own](https://github.com/logstash/logstash-logback-encoder/blob/master/src/main/java/net/logstash/logback/fieldnames/LogstashAccessFieldNames.java),
even for log events that do not pertain to any HTTP requests. For example
the `HOSTNAME` and `application.home` fields seem to originate from that
module.

We were not able to date to remove or rename these unnecessary (to us anyway) fields.

## Scala-related method ambiguity

Finally, an issue came up when using these Marker based variants of the
log methods:

    logger.info(append("name", "value"), "log message")
    
    logger.info(append("name1", "value1").and(append("name2", "value2")), "log message")

For Scala these are ambiguous with other logging methods so the above did
not compile in our case (using Scala up to 2.11.5) - Thinking about it, this might just be the
reason why Play's LoggerLike does not provide the methods in the first place...

Anyhow, using the 

    val fields = Map( "productId" -> "123456" , "traceId" -> "98765" )
    logger.info(appendEntries(fields), "log message")

variant only solves this problem and seems like the nicer method to use
in most cases anyhow.

That's your JSON logging in Play Framework.




    
