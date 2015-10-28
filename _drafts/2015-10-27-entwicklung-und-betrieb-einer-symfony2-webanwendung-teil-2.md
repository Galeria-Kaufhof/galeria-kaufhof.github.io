---
layout: post
title: "Entwicklung und Betrieb einer Symfony2 Webanwendung - Teil 2"
description: "Diese Artikelserie erläutert ausführlich und anhand eines realen Projekts
              die Entstehung einer vollwertigen Webanwendung auf Basis von Symfony2
              unter Verfolgung von testgetriebener Entwicklung und Continuous Delivery.
              Teil 2 erläutert den Aufbau einer Continuous Delivery Pipeline für das Projekt."
category: tutorials
author: manuelkiessling
tags: [php, symfony2, continuous delivery, migrations, tdd]
---
{% include JB/setup %}


## Über diesen Artikel

Teil 2 dieser Serie beschreibt den Aufbau einer Coninuous Delivery Pipeline für die in [Teil 1]({% post_url 2015-10-27-entwicklung-und-betrieb-einer-symfony2-webanwendung-teil-1 %}) beschriebene Anwendung.


### Aufbau der Continuous Delivery Pipeline

Was konkret ist eine Continuous Delivery Pipeline? Letztlich ist es eine Software oder ein System mehrerer Softwarekomponenten, in das vorne eine neue Version der Anwendungssoftware einläuft, und aus dem hinten ein kompletter Release herausfällt der dem Endbenutzer vollständig und fehlerfrei zur Verfügung steht:

    Entwicklerin fügt gelben Button zur Seite hinzu,
    testet lokal, und committet in den Releasezweig
    der Codeverwaltung
    
                         |
                         V
    
    Continuous Delivery Pipeline holt die Codeänderung
    ab, macht diese releasefertig, verifiziert die
    Korrektheit auf dem Zielsystem, und released
    
                         |
                         V
    
    Benutzer kann gelben Button klicken.

Das im Open Source Umfeld am weitesten verbreitete Continuous Delivery System ist [Jenkins](https://jenkins-ci.org/), und dieses setzen wir bei Galeria.de auch sehr umfangreich ein. Für die hier vorliegende one-off Anwendung wäre jedoch einerseits Jenkins eine unangemessen große Lösung gewesen, andererseits war wie erwähnt eine der Anforderungen, die Entwicklungsprozesse und -systeme des Shops mit dem *Good Buy METRO* Projekt so wenig wie möglich zu stören.

Daher kam eine sehr viel leichtgewichtigere, nicht zentralisierte Continuous Delivery Lösung auf Basis von [SimpleCD](https://github.com/manuelkiessling/simplecd) zum Einsatz.

Wenn Jenkins der Mercedes unter den CD/CI Systemen ist, dann ist SimpleCD der Tretroller. Im Grunde handelt es sich um ein einziges Shellskript, welches Code aus einem Git Repository abholen kann und weitere Shellskripte, die in einem bestimmten Verzeichnis des abgeholten Repositories liegen, ausführt. Diese durch SimpleCD ausgeführten Skripte können dann Migrations und Unittests ausführen, Dateien in Zielverzeichnisse bewegen, Services neustarten usw. - was auch immer im Einzelnen für das betreffende Projekt notwendig ist um es releasefertig zu machen, auf Korrektheit zu prüfen, und zu veröffentlichen.
