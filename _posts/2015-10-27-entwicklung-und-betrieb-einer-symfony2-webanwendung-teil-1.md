---
layout: post
title: "Entwicklung und Betrieb einer Symfony2 Webanwendung - Teil 1"
description: "Diese Artikelserie erläutert ausführlich und anhand eines realen Projekts
              die Entstehung einer vollwertigen Webanwendung auf Basis von Symfony2
              unter Verfolgung von testgetriebener Entwicklung und Continuous Delivery.
              Teil 1 betrachtet die Anforderungen an die Anwendung und das Aufsetzen einer
              Grundstruktur nebst Datenbankmigrationen und ersten funktionalen Tests."
category: tutorials
author: manuelkiessling
tags: [php, symfony2, continuous delivery, migrations, tdd]
---
{% include JB/setup %}


## Über diesen Artikel

Vor kurzem standen wir vor der Herausforderung, eine kleine Onlineanwendung für eine zeitlich begrenzte Rabattaktion zu realisieren, die keinerlei Verbindung mit dem Galeria.de Webshop hatte.

Während [der Technologiestack rund um unseren Onlineshop]({% post_url 2014-09-20-jump-ein-technologiesprung-bei-galeria-kaufhof %}) auf Scala, Ruby und Casssandra basiert, wurde hier die Entscheidung gefällt, die Anwendung außerhalb unserer bestehenden Dienste und Systeme zu realisieren, und auch nicht im Kontext unserer Scala und Ruby Teams, mit dem Ziel den normalen Produktentwicklungsprozess nicht mit diesem Sonderprojekt zu "stören".

Weiterhin waren hier die für den Online-Shop geltenden Skalierungsanforderungen, die ein zentraler Treiber hinter den technologischen Entscheidungen unseres Hauptstacks sind, nicht von Belang.

Da die Deadline für dieses Projekt sehr knapp gesteckt war, hat man sich der technologischen Lösung sehr pragmatisch genähert. Es fiel die Entscheidung, die Anwendung mit dem PHP-basierten Symfony2 Framework und MySQL als Datenbank zu bauen. Diese Kombination ist sehr gut etabliert und hat sich als genau die richtige Wahl für diese Art von Projekt herausgestellt.

Ich möchte dieses Projekt aus der realen Welt heranziehen um den Leser durch all jene Details des Produktentwicklungsprozesses zu führen, die eine relevante Rolle spielen im Zusammenhang mit dem Schreiben und Betreiben von Anwendungen auf Basis von Symfony2 - hierbei gehe ich ein auf Aspekte wie Projektsetup, Testing, Datenbankmigrationen, Continuous Delivery, Sicherheit, und vieles mehr.

Der hierzu gewählte Ansatz hebt alle signifikanten Entscheidungen hervor, erklärt die Implementationsdetails die sich aus diesen Entscheidungen ergaben, und diskutiert die Vor- und Nachteile dieser Entscheidungen. Ich werde weiterhin diejenigen Teile der Anwendungen herausstellen, die weiter verbessert werden könnten.


## Zielgruppe

Dieser Beitrag richtet sich an PHP-Entwickler, die mindestens erste Erfahrungen in der Arbeit mit Symfony2 haben, und für die bspw. die Arbeit mit Composer bekanntes Terrain ist.


## Die Anforderungen

Mit Wirkung zum 1. Oktober 2015 wurde die GALERIA Kaufhof GmbH Teil der Hudson's Bay Company. Zuvor waren wir Teil der METRO GROUP. Vor Abschluss dieser Transaktion wurde eine allerletzte Rabattaktion für unsere nunmehr ehemaligen Kolleginnen und Kollegen der METRO aufgesetzt: *Good Buy METRO*.

Der Use Case sah wie folgt aus: Für eine begrenzte Zeit konnten sich Mitarbeiterinnen und Mitarbeiter bestimmter METRO-Tochterunternehmen über die hier beschriebene Webanwendung für die Rabattaktion registrieren, basierend auf ihrer Mailadresse und Personalnummer. Nach Abschluss der Registrierung erhielt jeder Benutzer eine Mail mit einem PDF-Anhang, auf dem insgesamt sechs personalisierte Gutscheine abgedruckt waren. Jeder Gutschein enthielt einen QR Code, der an der Kasse einer unserer Filialen eingescannt werden konnte, um den Rabatt auf den Einkauf zu erhalten.

Im Kern lauteten die funktionalen Anforderungen daher:

* Erlaube Zugriff auf eine Webanwendung
* Ermögliche über die Webanwendung eine Registrierung auf Basis von Mailadresse und Personalnummer
* Verifiziere die Gültigkeit der Personalnummer über einen internen Prozess, sowie die Gültigkeit der Mailadresse über ein Double Opt-In Verfahren
* Wähle für jeden verifizierten Benutzer aus dem Pool aller Rabattcodes sechs freie Codes aus
* Erstelle für jeden dieser Codes einen QR Code
* Erstelle auf Basis der sechs QR Codes ein PDF Dokument für den Benutzer und sende es ihm per Mail

Hinzu kamen nicht-funktionale Anforderungen. Um in der kurzen Projektphase stets zeitnah und zuverlässig auf Detailänderungen in den funktionalen Anforderungen reagieren zu können, war eine hohe Testabdeckung erforderlich. Weiterhin sollten Änderungen immer umgehend in der Produktionsumgebung verfügbar sein, damit eine enge Feedback-Schleife mit den Anforderern möglich war. Dies wiederum bedingte eine vollautomatische Continuous Delivery Pipeline, und eine der Voraussetzungen hierfür war der Einsatz von Datenbank-Migrations.

Da das System personenbezogene Daten speichern würde, wurde eine externe Sicherheitsüberprüfung eingeplant, und diese mit einem guten Ergebnis zu bestehen war eine weitere Anforderung. Zusätzlich war das Thema Laststabilität im Fokus - zwar wurde die Anwendung nur einem begrenzten Nutzerkreis zur Verfügung gestellt, aber da es sich um eine zeitlich eng begrenzte Sonderaktion handelte, war ein gewisser Ansturm zu Beginn der Aktion zumindest möglich. Daher wurde auch ein Lasttest eingeplant mit der Anforderung, dass die Webanwendung auch bei vielen parallelen Zugriffen gute Antwortzeiten lieferte.

Eine weitere nicht-funktionale Anforderung war, dass die Anwendung auch auf mobilen Geräten angenehm zu bedienen sein sollte. 


## Die Umsetzung

### Aufsetzen des Projekts

> Die [README des Projekts](https://github.com/Galeria-Kaufhof/goodbuy-metro#good-buy-metro) auf GitHub bietet einen Leitfaden zur Einrichtung eines Mac OS X Systems als Entwicklungsumgebung für die Anwendung.

Der erste Schritt in der Entwicklung war das Anlegen eines neuen Symfony2 Projekts. Ich entschied mich für die aktuelle stabile nicht-LTS Version von Symfony, zum damaligen Zeitpunkt 2.7.3. So verständlich ich die Idee von Long Term Support Versionen finde, ziehe ich dennoch vor, lieber immer mit einer aktuellen stabilen Version zu arbeiten und auch immer zeitnah (vielleicht nach 2-3 minor releases) auf eine neue stabile Version upzugraden, wenn diese verfügbar wird.

Meiner Meinung nach läuft man ein eine Falle wenn man zu lange auf einer älteren Version verharrt, ein Vorgehen, welches durch LTS Versionen begünstigt wird. Man verliert einfach den Anschluss und ein Wechsel, der ja irgendwann erfolgen *muss*, wird immer furcheinflößender, komplexer und teurer. Lieber regelmäßig durch einen kleinen Schmerz gehen (der bei guter Testabdeckung eh überschaubar ist) und nicht in die Falle laufen, irgendwann ein Legacy-System zu haben. Für mich ist dieses Vorgehen ein Beispiel für das agile Prinzip *If It Hurts, Do It More Often* - guten Lesestoff bietet hier zum Beispiel Martin Fowler in [FrequencyReducesDifficulty](http://martinfowler.com/bliki/FrequencyReducesDifficulty.html).

Wie unter [Installing and Configuring Symfony](http://symfony.com/doc/current/book/installation.html) beschrieben wurde der Symfony Installer heruntergeladen und installiert, um dann mittels `symfony new goodbye-metro 2.7.3` das Projekt aufzusetzen.

Symfony2 ist die Basis der Anwendung im Backend, aber eine Webanwendung hat auch ein Frontend, und auch dieses will z.B. in Hinblick auf externe Bibliotheken und Frameworks gemanaged werden. Hierzu wurde *Bower*, der JavaScript Paketmanager, benutzt. Über die Datei *bower.json* im Hauptverzeichnis des Projekts wurde *Bootstrap* als Abhängigkeit definiert:

    {
      "name": "goodbye-metro",
      "version": "0.0.1",
      "dependencies": {
        "bootstrap": "~3.3.5"
      }
    }

Um Bower und Symfony2 sinnvoll zu integrieren ist es wichtig dafür zu sorgen, dass Bower seine Bibliotheken im richtigen Zielverzeichnis ablegt. Der Symfony Best Practice folgend, sollte die Anwendung im Bundle *AppBundle* entstehen. Die öffentlichen Webdateien für dieses Bundle gehören in *src/AppBundle/Resources/public* - über das Assetsystem von Symfony wird dieser Ort nach *web/bundles/app* gespiegelt, und von dort können die Dateien vom Webserver geserved werden. Da wir mit Bower externen Code in unser Projekt holen (analog zu den externen PHP Libraries, die mittels Composer in *vendor* im Wurzelverzeichnis des Projekts landen), macht es Sinn auch diese in einem *vendor* Ordner abzulegen, um sie nicht mit internen Frontend-Dateien zu vermischen.

Um dies zu erreichen, wurde die Datei *.bowerrc* mit folgendem Inhalt angelegt:

    {
      "directory": "src/AppBundle/Resources/public/vendor",
      "interactive": false
    }

`"interactive": false` ist nützlich, um Bower ausführen zu können ohne dass Eingaben an der Kommandozeile abgefragt werden.

Wichtiges Detail: die externen Bibliotheken, die per Bower gemanaged werden, sollen nicht Teil des git Repositories werden. Daher wurde die Zeile `src/AppBundle/Resources/public/vendor` zur *.gitignore* Datei hinzugefügt.

Die Dependencies der PHP Welt hat der Symfony Installer automatisch nach *vendor/* heruntergeladen. Für die Frontend Dependencies müssen wir mittels Bower selber tätig werden:

    bower install


### Migrations als Grundlage für Continuous Delivery

Damit war nun ein Grundgerüst für die zu bauende Anwendung, sowohl in Hinblick auf das Backend als auch das Frontend, verfügbar. Aber dieses Grundgerüst musste noch erweitert werden, um den zukünftigen Anforderungen gerecht zu werden.

Ein ganz zentrales Element für das Erreichen einer Continuous Delivery sind Datenbankmigrationen. Statt händisch Schemaänderungen vorzunehmen, sind Veränderungen an der Struktur einer Datenbank abgebildet in Codedateien, die Teil des Projektrepositories sind wie anderer Code auch. Das Schema der Datenbank ist somit einerseits versioniert, andererseits können Schemaänderungen ohne menschliches Zutun durchgeführt werden.

Ist dieses Verfahren aufgesetzt, kann neuer Code automatisiert auf die Produktionsumgebung ausrollen, selbst wenn dieser Code eine veränderte Datenbankstruktur erwartet - im Zuge des Ausrollens wird die Datenbank automatisch auf die Struktur angepasst, die der neue Code erwartet.

Datenbankmigrationen sind in Symfony2 Projekten sehr leicht zu realisieren, da hierfür ein entsprechendes Bundle existiert. Um dieses zu installieren (und automatisch zu den Composer-verwalteten externen Abhängigkeiten hinzuzufügen), reicht folgender Aufruf:

    composer require doctrine/doctrine-migrations-bundle "^1.0"

Das neue Bundle musste nun dem Kernel der Anwendung bekannt gemacht werden, indem *app/AppKernel.php* um den Eintrag

    new Doctrine\Bundle\MigrationsBundle\DoctrineMigrationsBundle()

erweitert wurde, und die folgenden Konfigurationsparameter mussten in *app/config/config.yml* hinzugefügt werden:

    doctrine_migrations:
        dir_name: "%kernel.root_dir%/DoctrineMigrations"
        namespace: Application\Migrations
        table_name: migration_versions
        name: Application Migrations

Um nun zu ersten Migrations zu kommen machte es Sinn, eine erste Entität zu schaffen und die Erzeugung der dazugehörigen Datenbankstruktur in einer ebensolchen Migration abzubilden. Symfony2 bietet alle Hilfsmittel um diesen Weg nicht komplett zu Fuß gehen zu müssen.

Der naheliegenste Kandidat für diese erste Entität war der Nutzer der Rabattaktion, intern *Customer* genannt - die Namensgebung *User* oder, da es sich grundsätzlich im Konzernmitarbeiter handelte, *Employee*, wäre sicherlich ebenfalls möglich gewesen.

Über `php app/console doctrine:generate:entity` erfolgte die interaktive Erzeugung der Entität *Customer*. Das Ergebnis sieht man unter [src/AppBundle/Entity/Customer.php auf GitHub](https://github.com/Galeria-Kaufhof/goodbuy-metro/blob/ff28bc6d1d9e823e17a5b153b1911527ef45aa55/src/AppBundle/Entity/Customer.php).

Direkt mit Bordmitteln gelangt man von der neuen Entität zur zugehörigen Migrations-Datei: `php app/console doctrine:migrations:diff` stellt die Unterschiede zwischen dem Code (dem die *Customer* Entität bereits bekannt ist) und der Datenbank (die noch keine zugehörige Tabelle kennt) fest und legt unter *app/DoctrineMigrations/* eine Datei mit entsprechenden SQL Statements an (siehe [app/DoctrineMigrations/Version20150828083456.php auf GitHub](https://github.com/Galeria-Kaufhof/goodbuy-metro/blob/ff28bc6d1d9e823e17a5b153b1911527ef45aa55/app/DoctrineMigrations/Version20150828083456.php)).

Um die Datenbank nun mit dem Code zu synchronisieren, führt man schlicht `php app/console doctrine:migrations:migrate` aus.

> Hierbei sollte man beachten, dass neu erzeugte Migrations immer sofort angewendet werden sollten, bevor man weitere Veränderungen an Entitäten vornimmt. Der *diff* Befehl ist sehr gut darin zu erkennen, was die Unterschiede zwischen Entitäten und Datenbank sind, aber er kann nicht berücksichtigen, welche unangewendeten Migrations bereits existieren. Führt man zum Beispiel nach dem Erzeugen der Entität den *diff* Befehl zwei Mal direkt hintereinander aus, dann erhält man zwei Migrationsdateien, die aber bei den den gleichen Inhalt haben (Anlegen der Tabelle für die Entität), und ein Ausführen von *migrate* würde fehlschlagen wenn nach Anwenden der ersten Migrationsdatei die Anwendung der zweiten versucht, die soeben erstellte Tabelle noch mal anzulegen.


### Motivation für Continuous Delivery

Warum eigentlich noch mal das Ganze? Das Ziel ist die Schaffung und Nutzung einer Continuous Delivery Pipeline, und Migrations sind neben Tests ein notwendiges Mittel zum Zweck.

In der Softwareentwicklung bei Galeria.de ist Continuous Delivery ein sehr zentraler Baustein unseres Produktentwicklungsprozesses, deshalb wurde auch bei diesem sehr kleinen Sonderprojekt Wert darauf gelegt.

Wer Software entwickelt, kennt vermutlich das Phänomen: In der eigenen Entwicklungsumgebung funktioniert alles wie gewünscht, aber auf dem Produktionssystem verhält sich die Software anders, und nicht selten fehlerhaft. Oder auch: man ist zu 95% fertig mit dem Projekt, nun muss man es "nur noch" releasen, und stellt fest, dass man nicht 5%, sondern noch 30% des Aufwands vor sich hat, bis man wirklich gelauncht ist.

Das hervorragende Buch *Growing Object-Oriented Software, Driven By Tests* macht dieses Phänomen auf interessante Weise anschaulich. Würde man eine "Stresskurve" über die Projektdauer plotten die anzeigt, welches Level von Stress oder Chaos im Projekt zu einem beliebigen Zeitpunkt auf dem Weg zum Launch herrscht, dann sieht diese klassischerweise wie folgt aus:

    Stress/Chaos                                 
                                                 
          ^                                      
          │                                      
          │                            *         
          │                            *         
          │                            *        
          │                           * *       
          │                           * *       
          │                           * *       
          │ **************************   *      
          +───────────────────────────────> Zeit
                                       |
                                    Launch

In der Zeit vor dem Launch ist es verhältnismäßig ruhig - man arbeitet vor sich hin, die Anwendung entsteht und lebt in der Entwicklungsumgebung, welche überschaubar und gut beherrscht ist. Dann kommt die Launchphase, und es wird hektisch: in Produktion sind Softwarepakete auf einem ganz anderen Stand, einzelne Nodes in der Entwicklungsumgebung werden zu Clustern mit vielen Nodes in Produktion, Netzwerkrouten funktionieren nicht, bei jedem Deployment ist die Seite einige Minuten lang offline und so weiter und so fort.

Eine Anwendung zu bauen hat aus dieser Perspektive zwei Aspekte: Das Herstellen von Funktionalität, und das Herstellen von Betriebsbereitschaft. Im klassischen Vorgehen liegt während nahezu der gesamten Projektphase der Fokus fast ausschließlich auf dem ersten Aspekt, und die Betriebsbereitschaft kommt zu kurz. Continuous Delivery dreht den Spieß um:

    Stress/Chaos                                 
                                                 
          ^                                      
          │                                      
          │
          │
          │
          │ *
          │  *
          │   *                        *
          │    ************************ **
          +───────────────────────────────> Zeit
                                       |
                                    Launch

Bei diesem Ansatz werden die knackigen Herausforderungen, die Software betriebsbereit zu bekommen und einen funktionierenden und zuverlässigen Releaseprozess sicherzustellen, an den Projektanfang gesetzt (*"Do the hard stuff first"*). Das ist durchaus anstrengend, denn es dauert in der Regel einen Moment bis zum allerersten Mal die (zu diesem Zeitpunkt in ihrer Funktionalität und Komplexität natürlich noch äußerst rudimentäre) Software sauber bis zu den Produktionssystemen ausrollt - dabei will man doch "richtig loslegen" und Features bauen, statt sich jetzt schon mit dem Aufsetzen von Serversystemen auseinanderzusetzen.

Aber hat man diese Hürde erst einmal genommen, erntet man für die gesamte Lebensdauer der Anwendung, und ganz besonders in der Launchphase, die Früchte dieses Ansatzes. Der finale Release, d.h. das zur-Verfügung-stellen der Anwendung für den eigentlichen Kunden, ist zum Zeitpunkt des Launches bereits dutzende, wenn nicht hunderte Male geübt und eingespielt. Wenn man ein Feature fertiggestellt hat, ist es *wirklich* fertig: Es liefert die geforderte Funktionalität, *und* rollt zuverlässig zum Kunden aus, *und* funktioniert auch im Live-Betrieb. Eine Funktionalität, die letzteres nicht bietet, ist aus Kundensicht exakt so wertvoll wie die vollständige Abwesenheit der Funktionalität.

Continuous Delivery ist dabei natürlich kein singuläres Event - so, wie die Anwendung in der Projektphase in ihrer Funktionalität wächst, wächst auch der Deliveryprozess mit. Hier ist man nicht davor gefeit, ab und zu eine kleine Überraschung zu erleben und nachbessern zu müssen - aber meiner Erfahrung nach lebt es sich deutlich besser mit seltenen kleineren Überraschungen als mit einer großen zum ungünstigsten denkbaren Zeitpunkt, dem Launch.


### Erste Tests

Zurück zur Anwendung. Migrations waren nun aufgesetzt, und eine funktionierende Continuous Delivery das nächste Ziel. Um dies nicht ganz im luftleeren Raum zu verfolgen, ging es nun darum, erste Funktionalität zu erzeugen - und die Korrektheit dieser Funktionalität mit einem Testfall zu beweisen, denn die Idee einer Continuous Delivery Pipeline ist ja, dass sie automatisch, ohne weiteren menschlichen Eingriff, die Software auf Produktionssystemen veröffentlicht; da also auch kein Mensch die Korrektheit testet, muss die Korrektheit über automatisierte Testfälle gewährleistet werden.

Zu diesem Zeitpunkt existierte lediglich die *Customer* Entität, und diese verfügte nicht wirklich über nennenswertes Verhalten, welches sinnvoll zu testen gewesen wäre. Der nächste Schritt war daher die Schaffung eines ersten Testfalls, der relevantes Verhalten der Anwendung überprüfen würde, und der innerhalb eines Delivery-Durchlaufs bewies, dass die Anwendung auf dem Zielsystem erwartungsgemäß funktionierte. Der Fokus lag daher auch auf einem funktionalen Test, und nicht auf einem Unit-Test; Units wie Methoden und Klassen sind in der Regel so isoliert, dass ihr Funktionieren innerhalb eines Testcases wenig darüber aussagt, ob die Anwendung an sich auf dem Zielsystem korrekt läuft - sprich, selbst eine ganze Batterie an fehlerfrei durchlaufenden Unittests sagt mir nicht, ob meine Continuous Delivery eine für den Benutzer korrekt laufende Anwendung zum Ergebnis hat.

Funktionale Tests zu schreiben ist in Symfony2 Anwendungen glücklicherweise sehr einfach und komfortabel. Man testet hierbei zwar nicht auf Basis realer HTTP-Anfragen und -Antworten, aber man testet dennoch die integrierte Anwendung in ihrer Gesamtheit und kann somit sicherstellen, dass sich alle relevanten Komponenten im Zusammenspiel korrekt verhalten.

Um funktionale Tests schreiben zu können, bedurfte es nur wenig Vorbereitung. Zum einen musste PHPUnit als Dependency definiert werden mittels `require phpunit/phpunit "^4.8"`, und eine *phpunit.xml.dist* musste im Wurzelverzeichnis des Projekts angelegt werden - siehe [phpunit.xml.dist auf GitHub](https://github.com/Galeria-Kaufhof/goodbuy-metro/blob/master/phpunit.xml.dist) für den Inhalt.

Nun kann man über das Schreiben von Testklassen, die *Symfony\Bundle\FrameworkBundle\Test\WebTestCase* erweitern, funktionale Testfälle erzeugen. Der allererste funktionale Testfall im Projekt, in Datei *src/AppBundle/Tests/Functional/RegistrationTest.php*, sah wie folgt aus:

    <?php
    
    namespace AppBundle\Tests\Functional;
    
    use AppBundle\Tests\TestHelpers;
    use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
    
    class RegistrationTest extends WebTestCase
    {
        public function testContents()
        {
            $client = static::createClient();
    
            $client->request('GET', '/');
    
            $this->assertEquals(200, $client->getResponse()->getStatusCode());
        }
    }

Der Testfall selber ist sehr überschaubar, aber sein Funktionieren beweist, dass die integrierte Anwendung in der Lage ist, korrekt auf einen Request gegen die Route / zu antworten. Wie gesagt werden hierbei keine realen HTTP Requests über eine reale Leitung geschickt - der *$client*, den man über den *WebTestCase* von Symfony2 erzeugt, ist lediglich eine clevere Abstraktion, die stets im Kontext der PHP Laufzeit bleibt. Jedoch läuft der Client gegen die vollständig integrierte Symfony-Anwendung, d.h. der Testfall kann nur erfolgreich sein, wenn Dependencies, Konfiguration, Routing, Controller, Datenbank usw. richtig funktionieren und zusammenspielen. Für das angestrebte Ziel ist dies völlig ausreichend.

Ausgeführt wird dieser Testfall nun schlicht mittels `php ./vendor/phpunit/phpunit/phpunit`.

An diesem Punkt war ein wichtiger erster Zwischenstand erreicht: Die Anwendung war grundsätzlich aufgesetzt, Veränderungen an der Datenbank waren dank Migrations codeseitig steuerbar, und die notwendigen Strukturen für einen ersten Testcase waren in Stellung gebracht. Mit anderen Worten: Das erste Paket für die Delivery war geschnürt - nun brauchte es die Pipeline zum Produktivsystem, über die das Paket geliefert werden konnte.

> Im demnächst erscheinenden Teil 2 dieser Serie wird der Aufbau der Continuous Delivery Pipeline in allen Details beleuchtet.
