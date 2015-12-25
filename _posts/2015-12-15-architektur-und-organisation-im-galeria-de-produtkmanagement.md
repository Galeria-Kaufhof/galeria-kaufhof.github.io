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

<img width="100%"
     src="{{ site.url }}/assets/images/architektur-update/Ueberblick_Architektur_GALERIA_Kaufhof_Online_Plattform.svg">


## Grundlagen der Architektur

Zwei Grundideen bilden das Fundament der architektonischen Strukturierung: Eine vertikale Orientierung der High-Level
Komponenten in sogenannten Self-contained Systems, und eine fachliche motivierte Trennung und Gruppierung dieser
Komponenten in sogenannten Domänen.

Das Verhältnis von Domänen zu Systemen ist wie folgt: eine Domäne liegt immer dann vor, wenn ein oder
mehrere Systeme einen logisch zusammenhängenden Ausschnitt der fachlichen Use-Cases eines Benutzers vollumfänglich
abbilden. Konkretes Beispiel: die Domäne SEARCH bei Galeria.de umfasst diejenigen Systeme, welche von der
Benutzeroberfläche bis zur Datenhaltung das Suchen und Finden von Produkten für den Benutzer von galeria.de ermöglichen.

Der Domäne SEARCH ist also mindestens ein System zugeordnet, welches sowohl die Weboberflächen-Elemente (wie zum
Beispiel die Suchbox mit Auto-Complete, Suchergebnisseite usw.) bereitstellt, als auch den Import von Produktdaten und
deren Überführung in eine spezialisierte Such-Datenbank implementiert.

Untereinander sprechen diese Systeme - innerhalb einer Domänengrenze und darüber hinaus - nur über definierte
Schnittstellen miteinander, unter Vermeidung von verteilten Callstacks.

In gewissem Sinne wird hier das bekannte Paradigma von loser Kopplung und hoher Kohäsion, welches klassischerweise auf
Ebene eines Softwaresystems betrachtet wird, auf einer höheren Ebene fortgesetzt.

Die Kohäsion entsteht, weil fachlich verwandte Themen vereint werden in den Self-contained Systems einer Domäne. Die
lose Kopplung wird abgebildet dadurch, dass die verschiedenen Systeme nur über Schnittstellen miteinander kommunizieren.

Damit gilt für das Gesamtsystem dieselbe Eigenschaft, die auch innerhalb eines Softwaresystems gilt, welches nach diesem
Paradigma entworfen wurde: Änderungen in einer Komponente bedingen nur dann Änderungen in einer anderen Komponente,
wenn die Änderungen die Schnittstelle betreffen.

Dies sorgt für hohe Robustheit des Gesamtsystems (ohne verteilte Callstacks können andere Systeme weiter operieren, auch
wenn ein angebundenes System nicht-verfügbar wird), ermöglicht weitgehend autarkes Arbeiten pro Domäne (nicht zuletzt in
Hinblick auf die Releasefrequenz), bietet die Möglichkeit, Systeme nach ihren unterschiedlichen Anforderungen auch
unterschiedlich zu skalieren, und erlaubt eine in Hinblick auf die erforderliche Funktionalität passgenaue Wahl der
Technologien pro System. Weitere Informationen hierzu liefert [scs-architecture.org](http://scs-architecture.org).

Das Konzept der Domäne ist weiterhin der Brückenschlag zwischen Architektur und Aufbauorganisation. Optimalerweise steht
hinter jeder Domäne ein Team - in unserem Fall ein Scrum-Team - welches von Anforderungsmanagent über
Software-Entwicklung und QA bis hin zum Betrieb die Domäne mit ihren Systemen fachlich und technisch vollumfänglich
"owned".


## Domänen und ihre Systeme

Warum dann noch die Unterscheidung zwischen Domäne und System? Warum nicht 1 Domäne gleich 1 System? An dieser Stelle
findet derzeit eine Evolution unseres bisherigen Modells statt, in dem die Begriffe bisher deckungsgleich verwendet
wurden.

Die Grund für eine Unterscheidung ist, dass ein Komponentenschnitt einerseits fachlich motiviert sein kann, andererseits
technisch. Eine rein technische Motivation führt hierbei zu einem neuen System, eine fachliche Motivation zu einer neuen
Domäne. Der Begriff der "technischen Motivation" ist allerdings recht weit gefasst - es muss nicht zwangsläufig die
Einführung einer neuen Technologie (Programmiersprache, Framework, Datenbanksystem usw.) vorliegen: selbst bei
gleichbleibendem Stack kann es die technische Motivation geben, eine weiterhin saubere Codebase gewährleisten zu wollen
oder feingranularer releasen zu wollen.

Für beide Motivationen gibt es derzeit Beispiele im Projekt. Das Team der bestehenden Domäne EXPLORE, welches sich
bisher vornehmlich um Teaser und Störer im Shop kümmert, soll in Zukunft die Verantwortung übernehmen für die
Infrastruktur von Inhalten auf galeria.de, die nicht direkt mit dem Shopping-Erlebnis des Kunden zu tun haben,
beispielsweise Presseseiten und Unternehmensinformationen sowie Inhalte rund um das Recruiting.

Abgesehen davon, dass hier auf Basis eines Open Source CMS auch technisch eine neue Lösung entsteht, wird schnell klar,
dass die bisherige EXPLORE Fachlichkeit verlassen wird. Daher begründet das Team derzeit eine neue Domäne CONTENT. Auch
wenn hier vorerst dieselben handelnden Personen an Bord sind, sehen wir das zu behandelnde Thema aufgrund der
andersartigen fachlichen Ausrichtung als neue fachliche Einheit.

Anders im Team ORDER, welches sich um alle Fachlichkeiten rund um das Thema Bestellungen kümmert. Hier zeichnet sich ab,
dass es sinnvoll sein könnte, die Webshop-orientierten Aspekte des Bestellens von der nachgelagerten Bestellverarbeitung
zu trennen - sinnvoll zum Beispiel vor dem Hintergrund der möglicherweise sehr unterschiedlichen
Skalierungsanforderungen für den Shop einerseits und die nachgelagerte Verarbeitung von Bestellungen andererseits;
weiterhin ist zu erwarten, dass eine Entkopplung dieser beiden Aspekte auch in Hinblick auf Releasezyklen und Sauberkeit
der lokalen Architektur Vorteile bringt. Self-contained Systems sind stets Monolithen - das ist nicht per se negativ,
aber jeder Monolith kann irgendwann zu groß werden; "zu groß" ist eine sehr subjektive Eigenschaft, aber ein Maßstab ist
sicherlich die mittlerweile weitverbreitete Formulierung "passt nicht mehr vollständig in den Kopf eines Teammitglieds".


## Systeme und Umgebungen

Das Schaubild spricht weiterhin von Umgebungen. Gemeint ist damit der Kontext, in dem die Anwendungen von Systemen
laufen können. Aufgrund der vertikalen Orientierung von Systemen ist der Ausführungskontext ihrer Anwendungen potentiell
verteilt: Eine Play2-basierte Scala Anwendung wird auf den Servern von Galeria.de ausgeführt, also in der sogenannten
Plattformumgebung - diese Backend-Anwendung liefert aber vielleicht eine Single-Page Application oder eine andere Form
von JavaScript-Anwendung aus; diese läuft dann im Browser des Benutzers, also in dessen Umgebung.

Weiterhin sprechen viele Systeme der Plattformumgebung mit Fremdsystemen, im Falle von Galeria.de beispielsweise mit
Systemen der Warenwirtschaft. Diese werden ausgeführt in einer Fremdumgebung, also einer Umgebung die technisch und
organisatorisch außerhalb der Kontrolle von Galeria.de liegt.


## Schnittstellen

