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
Weiterentwicklungsmodus ist, wird auch die Frage des fachlichen Wachstums spannend. So klar die bestehende Struktur ist
- was sind die architektonischen Leitlinien, wenn der fachliche Themenumfang wächst und die Plattform sich
inhaltich weiterentwickelt? Schliesslich stehen wir noch am Anfang unserer Mission, MCR-Marktführer in Europa zu werden.

Zusätzlich klingt in diesem Dokument auch das Verhältnis zwischen Produkt-Architektur und
Produktentwicklungsorganisation stärker an (ohne dabei den Anspruch zu erheben, die Aufbau- und Ablauforganisation der
Galeria.de Produktentwicklung vollumfänglich aufzuzeigen - dies muss im Zuge anderer Beiträge erfolgen).


## Die Visualisierung

Einen Überblick über die verschiedenen Architekturkomponenten und ihr Verhältnis zueinander soll das folgende Schaubild
ermöglichen:

<img width="960"
     src="{{ site.url }}/assets/images/architektur-update/Ueberblick_Architektur_GALERIA_Kaufhof_Online_Plattform.svg">

Zwei Grundideen bilden das Fundament der architektonischen Strukturierung: Eine vertikale Orientierung der High-Level
Komponenten in sogenannten Self-contained Systems, und eine fachliche Motivation für Trennung und Gruppierung dieser
Komponenten in sogenannten Domänen.

Das Verhältnis von Domänen zu Systemen ist wie folgt: eine Domäne liegt immer dann vor, wenn ein oder
mehrere Systeme einen logisch zusammenhängenden Ausschnitt der fachlichen Use-Cases eines Benutzers vollumfänglich
abbilden. Konkretes Beispiel: die Domäne SEARCH bei Galeria.de umfasst diejenigen Systeme, welche von der
Benutzeroberfläche bis zur Datenhaltung das Suchen und Finden von Produkten für den Benutzer von galeria.de ermöglichen.

Der Domäne SEARCH ist also mindestens ein System zugeordnet, welches sowohl die Weboberflächen-Elemente (wie Suchbox mit
Auto-Complete, Suchergebnisseite usw.) bereitstellt, als auch den Import von Produktdaten und deren Überführung in eine
spezialisierte Such-Datenbank implementiert.

Warum dann noch die Unterscheidung zwischen Domäne und System? Warum nicht 1 Domäne gleich 1 System? Die Motivation
hierfür ist, dass ein Komponentenschnitt einerseits fachlich motiviert sein kann, andererseits technisch.




In gewissem Sinne wird hier das bekannte Paradigma von loser Kopplung und hoher Kohäsion, welches klassischerweise auf
Ebene eines Softwaresystems betrachtet wird, auf einer höheren Ebene fortgesetzt.

Die Kohäsion entsteht, weil fachlich verwandte Themen vereint werden in den Self-contained Systems einer Domäne. Die
lose Kopplung wird abgebildet dadurch, dass die verschiedenen Systeme nur über Schnittstellen miteinander kommunizieren.

Damit gilt für das Gesamtsystem dieselbe Eigenschaft, die auch innerhalb eines Softwaresystems gilt, welches nach diesem
Paradigma entworfen wurde: Änderungen in einer Komponente bedingen nur dann Änderungen in einer anderen Komponente,
wenn die Änderungen die Schnittstelle betreffen.
