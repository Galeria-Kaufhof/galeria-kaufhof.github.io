---
layout: post
title: "Jump - Ein Technologie-Sprung bei Galeria Kaufhof"
description: "Jetzt ist es offiziell: Seit 6 Monaten arbeiten wir von inoio zusammen mit weiteren Dienstleistern und Galeria Kaufhof an deren neuer Multi-Channel Online Plattform - Projektname “Jump”. Mit dem neuen System soll die Time-to-Market erheblich reduziert werden, wenn es um die Einbindung und Entwicklung neuer Features geht."
category: general
author: martingrotzke
tags: [inoio, scala, cassandra, architecture]
---
{% include JB/setup %}

*Dies ist ein Cross-Post vom
[Blog der Agentur inoio](http://www.inoio.de/blog/2014/09/20/technologie-sprung-bei-galeria-kaufhof/)*

Jetzt ist es
[offiziell](http://www.galeria-kaufhof.de/ueber-uns/presse/pressemitteilungen/pressemitteilung-140904.html):
Seit 6 Monaten arbeiten wir von [inoio](http://www.inoio.de/) zusammen mit weiteren Dienstleistern und Galeria Kaufhof
an deren neuer Multi-Channel Online Plattform - Projektname "Jump". Mit dem neuen System soll die Time-to-Market
erheblich reduziert werden, wenn es um die Einbindung und Entwicklung neuer Features geht.


## Makroarchitektur

Die monolithische Architektur des bisherigen Systems wird im Rahmen des Projektes durch eine moderne, skalierbare
["Shared Nothing"-Architektur](http://en.wikipedia.org/wiki/Shared_nothing_architecture) abgelöst. Die verschiedenen
fachlichen Domänen werden dabei im Sinne einer funktionalen Modularisierung durch voneinander getrennte Systeme
umgesetzt. Die Systeme arbeiten ähnlich wie Microservices - passender ist hingegen der Begriff der "Self Contained
Systems", wie ihn Stephan Tilkov in [Sustainable Architecture](https://speakerdeck.com/stilkov/sustainable-architecture)
beschreibt.


## Domänen und Querschnittsteams

![Galeria Kaufhof Jump Makroarchitektur]({{ site.url }}/assets/images/jump-arch.jpg)

* Frontend-Integration: Integration der Vertikalen zu einer zusammenhängenden Website
* Explore: Teaser-Steuerung
* Search: Produktsuche u. Navigation
* Evaluate: Produktdetails
* Order: Bestellen
* Control: Kundenkonto
* Foundation Systems: Querschnittsdienste wie Media oder Feature Toggles
* Platform Engineering: Tools, Deployment und Betrieb

Die einzelnen Domänen werden dabei von externen, spezialisierten Teams zusammen mit Galeria Kaufhof-internen
Programmierern entwickelt. Dabei ist jedes Team für eine einzelne Domäne verantwortlich. Für das Zusammenspiel zwischen
den Systemen (Makroarchitektur) gibt es einige Regeln: Um eine lose Kopplung zu gewährleisten darf es z.B. kein
fachliches Code Sharing (für vermeintlich gemeinsame Datenklassen) geben, genausowenig eine gemeinsame Datenhaltung.
Die Kommunikationen zwischen den Systemen erfolgt ausschliesslich über REST-Schnittstellen. Die Kommunikation muss dabei
asynchron erfolgen, um verteilte Call-Stacks im Client-Request zu vermeiden. Alle Systeme müssen zustandslos
implementiert sein, damit Ausfallsicherheit und Skalierbarkeit gewährleistet werden. Während die Makroarchitektur
weitestgehend vorgegeben ist, ist die Mikroarchitektur, also der Aufbau eines einzelnen Systems in der Verantwortung des
Teams. So gibt es beispielsweise keine verpflichtende gemeinsame Programmiersprache/Technologie.


## Mikroarchitektur / Technologien

Von den 5 Domänen haben sich 4 für eine Lösung auf Basis von Scala und Play Framework/Akka entschieden und ein Team für
Ruby on Rails. Die von inoio entwickelten Systeme Explore und Search werden auf Basis von Scala und Play implementiert.
Für Scala haben wir uns zum einen aufgrund der Paradigmen der funktionalen Programmierung entschieden -
weil wir davon überzeugt sind, dass Programmieren unter Vermeidung von Seiteneffekten letztendlich zu qualitativ
besserer Software führt und dies durch einen funktionalen Programmierstil einfacher zu erreichen ist. Zum anderen läuft
Scala auf der JVM und man hat Zugriff auf eine Vielzahl von Java-Bibliotheken. Das Play Framework wiederum passt durch
seinen HTTP-freundlichen Ansatz, seine hohe Skalierbarkeit durch Zustandslosigkeit und Unterstützung von asynchronen,
nichtblockierenden Implementierungen (Buzz: "reactive") sehr gut in das Projekt. Für Persistenz wird Cassandra
eingesetzt - u.a. wegen seiner sehr guten Skalierungs- und Verfügbarkeitseigenschaften. In unserem Search Team verwenden
wir Solr als Suchtechnologie, hauptsächlich weil Solr im Vergleich zu ElasticSearch bessere Eingriffsmöglichkeiten bzgl.
Suchqualität bietet.

An diesem Punkt ist die Integration eines asynchron/nicht-blockierend arbeitenden Frameworks mit Backend-Schnittstellen
(wie Datenbank etc.) interessant: Häufig existieren für Backend-Technologien nämlich nur synchrone/blockierende Treiber,
sodass der Nutzen eines asynchron arbeitenden Frameworks (u.a. effizientere Verarbeitung und bessere Skalierbarkeit
durch Minimierung der Anzahl benötigter Threads) zunichte gemacht wird. Entweder man verwendet das Web-Framework dann
synchron (was das Play Framework auch anbietet), oder man muss die sychronen Zugriffe durch eine separate Schicht mit
einer asynchronen API versehen (z.B. mit Akka kapseln). Glücklicherweise bietet der
[Datastax Java Treiber für Cassandra](https://github.com/datastax/java-driver) bereits eine asynchrone API (die
Implementierung basiert auf Netty), die sich sehr einfach in Scala Futures übersetzen lässt. Für Solr wiederum haben wir
uns einen eigenen [asynchronen Client](https://github.com/inoio/solrs) geschrieben (auf Basis des
[Async Http Client](https://github.com/AsyncHttpClient/async-http-client)), der die gleiche API wie SolrJ anbietet, aber
eben asynchron. Somit lässt er sich im Wesentlichen genauso bedienen wie SolrJ.


## Wie geht's weiter?

Wir sind sehr glücklich, in Galeria Kaufhof einen Partner gefunden zu haben, der diese Technologiewahl (v.a. die Wahl
von Scala) befürwortet. Für Galeria Kaufhof ist das besonders relevant, weil sie bald die Weiterentwicklung der Platform
komplett im eigenen Haus haben wollen. Durch die Wahl moderner Technologien gibt es sicherlich einen Vorsprung im ["War
for Talents"](http://en.wikipedia.org/wiki/The_war_for_talent), um neue Entwickler anzuwerben - bei inoio haben wir
diese Erfahrung jedenfalls gemacht.

Zu diesem Zweck
[stellt Galeria Kaufhof das Projekt Jump auch "persönlich" vor](http://www.startplatz.de/event/galeria-sucht-hacker/),
am nächsten Donnerstag (25.9.2014) im Startplatz (Köln). Zunächst wird Nina Ehrenberg die fachlichen Hintergründe des
Projekts erläutern. Jan Algermissen (Plattform-Architekt) wird dann etwas zur Makro-Architektur und dem Zusammenspiel
der verschiedenen Systeme erzählen.
Im Anschluss werde ich noch etwas mehr Einblick in die technologischen Details geben, und anhand von Beispielen zeigen,
wie wir in unserer Domäne Search Play/Akka/Scala einsetzen, welche Themen hinter uns und welche Aufgaben noch vor uns
liegen. Natürlich sind Entwickler, die gerade neue Herausforderungen suchen besonders gern gesehen :-) Wer also Teil
dieses Projekts werden will: [Galeria Kaufhof sucht Scala Entwickler](http://www.wir-lieben-ecommerce.de/).
