---
layout: post
title: "Die Architektur der Galeria.de Plattform im Kontext der Produktentwicklungsorganisation"
description: "Dieser Artikel erläutert die architektonischen Rahmenbedingungen, die der Arbeit in der
              Softwareentwicklung für das im Galeria.de Produktmanagement ihre Orientierung geben."
category: general
author: manuelkiessling
tags: [scs, self-contained services, architecture, interfaces]
---
{% include JB/setup %}


## Über diesen Artikel

Im Kontext dieses Blogs stellt der vorliegende Artikel ein Update und eine Erweiterung des [Beitrags von September
2014]({% post_url 2014-09-20-jump-ein-technologiesprung-bei-galeria-kaufhof %}) dar.

Schon damals existierte ein klar definiertes Set an Vorgaben, welches den Rahmen für Makro- und Mikroarchitekturfragen
gesteckt hat und das Projekt in Hinblick auf Fragen der System- und Softwarearchitektur leitet.

In den vergangenen Tagen haben wir begonnen, ausgehend von den Erfahrungen bis heute und unserer jetzigen Perspektive,
einige der Grundlagen unserer Architektur noch einmal neu aufzuschreiben.

Anstoß hierzu lieferte unter anderem der Launch von http://scs-architecture.org/, einem (noch überschaubaren) Portal,
welches das Konzept der Self-contained Systems, die den zentralen Baustein auch unserer Architektur bilden, präsentiert.

Inhaltlich haben wir das Konzept SCS seit langem gelebt, aber semantisch war der Ansatz auf unserer Architekturlandkarte
nicht klar verortet. Mit der Überarbeitung haben nun alle zentralen Bausteine einen klaren Platz und einen klaren Namen.

Bei einem Projekt, welches seit nunmehr fast 2 Jahren läuft und seit fast einem Jahr im Betriebs- und
Weiterentwicklungsmodus ist, wird auch die Frage der fachlichen Wachstums spannend. So klar die bestehende Struktur ist
- was sind die architektonischen Leitlinien, wenn der fachliche Themenumfang wächst und die Plattform sich
inhaltich weiterentwickelt? Schliesslich stehen wir noch am Anfang unserer Mission, MCR-Marktführer in Europa zu werden.

Zusätzlich klingt in diesem Dokument auch das Verhältnis zwischen Produkt-Architektur und
Produktentwicklungsorganisation stärker an (ohne dabei den Anspruch zu erheben, die Aufbau- und Ablauforganisation der
Galeria.de Produktentwicklung vollumfänglich aufzuzeigen - dies muss im Zuge anderer Beiträge erfolgen).


