---
layout: post
title: "JSON Formatted Logging With Play"
description: ""
category: 
tags: []
---
{% include JB/setup %}

The [multi-channel retailing platform we are building at GALERIA Kaufhof](http://www.startplatz.de/event/galeria-sucht-hacker/) provides
centralised logging for all deployed applications and services. In order to leverage this common logging facility for 
upcoming metrics and analysis use cases we picked JSON as the common log format.

We are primarily using Scala and the Play! Framework for our application development therefore had
to get Play! to log in JSON. This note briefly describes our solution.

# Logback JSON Encoder

The logstash team provides a [JSON formatter for logback logs](https://github.com/logstash/logstash-logback-encoder). You add
this to your sbt project as the following dependency in the build.sbt:

    libraryDependencies += "net.logstash.logback" % "logstash-logback-encoder" % "4.0"

Then, configure logging in logger.xml with the minimal addition of this encoder to
the desired appender:

   <encoder class="net.logstash.logback.encoder.LogstashEncoder">

Logback will now start to write one-line JSON documents to your log file.

# Play! Logging With org.slf4j.Markers - the Built-In Logger

Just logging a bunch of rather technical fields and one field 'message' containing what you
actually log in the code doesn't make for a good analytics harvest later on. Let's try to
add some structure to our logs.

# -Play!- Logging with Markers - 2^nd Try



https://github.com/typesafehub/scala-logging

# Verbosity - Oh Dear!


http://logback.qos.ch/access.html




    
