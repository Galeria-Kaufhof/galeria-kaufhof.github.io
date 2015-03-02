---
layout: post
title: "Transformieren und Validieren von JSON Requests in Play2"
description: "In diesem Tutorial möchte ich auf die Verarbeitung des JSON Body eines eingehenden POST Requests innerhalb
              eines Play2 Controllers eingehen, insbesondere in Hinblick auf die fachliche Validierung der JSON Struktur
              und ihrer Überführung in Models in der Applikation."
category: tutorials
author: manuelkiessling
tags: [scala, play2, json]
---
{% include JB/setup %}

Als Anfänger in der Arbeit mit Scala und Play2 stehe ich aktuell vor der Aufgabe, eine einfache Webservice API mit Scala
& Play2 zu realisieren.

Der Webservice hat einen Endpunkt `/api/experiments`, über den via POST ein A/B Test (bzw. *Experiment*) mit 2 oder mehr
Varianten angelegt werden kann.

Der Body eines solchen Requests ist eine JSON Struktur, die den A/B Test beschreibt:

    {
      "name": "Checkout page buttons",
      "scope": 100.0,
      "variations": [
        {
          "name": "Group A",
          "weight": 70.0
        },
        {
          "name": "Group B",
          "weight": 30.0
        }
      ]
    }


Mit dieser Struktur wird ein A/B Test erzeugt, der zwei Gruppen kennt (*A* und *B*), wobei in Gruppe A 70% der Benutzer
wandern, in Gruppe B 30%.

Auf die fachlichen Aspekte des Systems möchte ich aber gar nicht weiter eingehen. Wer interessiert ist tiefer
einzusteigen findet unter [https://github.com/meinauto/freeab-server](https://github.com/meinauto/freeab-server) eine
Node.js Implementation, die ich unter
[https://github.com/manuelkiessling/freeab-server-scala](https://github.com/manuelkiessling/freeab-server-scala) derzeit
nach Scala überführe.

An dieser Stelle möchte ich lediglich auf die Verarbeitung des JSON Body eines eingehenden POST Requests innerhalb des
Play2 Controllers eingehen, insbesondere in Hinblick auf die fachliche Validierung der JSON Struktur und ihrer
Überführung in Models in der Applikation.

Aktuell kennt die Applikation zwei fachliche Entitäten: `Experiment` und `Variation`, wobei ein `Experiment`
eine Liste von mindestens zwei oder mehr `Variation` Entitäten beinhaltet.

Jede dieser Entitäten ist im Code repräsentiert in je zwei verschiedenen Models. Ein `FormExperiment` ist eine case
Klasse welche das Model eines von außen über den Request eingehenden, aber noch nicht persistierten `Experiment`
darstellt. Diesem Model fehlt im Gegensatz zum persistierten Model z.B. eine eindeutige ID:

    final case class FormExperiment(
      name: String,
      scope: Double,
      formVariations: List[FormVariation]
    )

Ein `FormExperiment` wiederum beinhaltet eine Liste von `FormVariation` Objekten:

    final case class FormVariation(
      name: String,
      weight: Double
    )

Ein `FormExperiment` mit einer Liste von `FormVariation` Objekten wird durch die Persistierung zu einem `Experiment` mit
einer Liste von `Variation` Objekten, und damit zu "richtigen" fachlichen Entitäten im Sinne der Businesslogik der
Anwendung. Auf die Details hierzu gehe ich jedoch nicht ein - im Folgenden zeige ich, wie aus dem JSON des eingehenden
Requests ein `FormExperiment` mit einer Liste von `FormVariation` Objekten wird, und wie sichergestellt wird dass dies
nur geschieht, wenn die JSON Struktur syntaktisch und fachlich korrekt ist.

"Fachlich korrekt" meint hier beispielsweise die Anforderung, dass die Summe aller `weight` Attribute der Variationen
immer genau 100 (Prozent) ergeben muss, oder dass innerhalb eines Experiments die Namen der Variationen eindeutig
(im Sinne von unique) sein müssen.

Aber eins nach dem anderen. Wie kann man den JSON Body innerhalb eines Scala Play2 Controllers überhaupt lesen? Play2
bringt eine sehr mächtige JSON Bibliothek mit. Diese verfügt über sogenannte *Reads*. Dies ist ein Trait welcher es
ermöglicht, JSON beispielsweise in case Klassen zu überführen. Für eine `FormVariation` sieht das beispielsweise so aus:

    import play.api.libs.json._
    import play.api.libs.functional.syntax._

    final case class FormVariation(
      name: String,
      weight: Double
    )

    implicit val formVariationReads: Reads[FormVariation] = (
      (JsPath \ "name").read[String] and
        (JsPath \ "weight").read[Double]
      )(FormVariation.apply _)

Wie man sieht muss man lediglich einen `Reads` definieren, der die einzelnen Komponenten der JSON Struktur auf die
richtigen Typen mappt und dann in die case Klasse transformiert. Wie dieser dann zum Einsatz kommt, sehen wir später.

Diese Reads können auch geschachtelt werden, so dass wir unsere Entitätenstruktur, bei der ein `FormExperiment` neben
direkten Attributen auch noch eine Liste von `FormVariation` Objekten beinhaltet, einfach abbilden können:

    import play.api.libs.json._
    import play.api.libs.functional.syntax._

    final case class FormVariation(
      name: String,
      weight: Double
    )

    implicit val formVariationReads: Reads[FormVariation] = (
      (JsPath \ "name").read[String] and
        (JsPath \ "weight").read[Double]
      )(FormVariation.apply _)
      
    final case class FormExperiment(
      name: String,
      scope: Double,
      formVariations: List[FormVariation]
    )

    implicit val formExperimentReads: Reads[FormExperiment] = (
      (JsPath \ "name").read[String] and
        (JsPath \ "scope").read[Double] and
          (JsPath \ "variations").read[List[FormVariation]]
      )(FormExperiment.apply _)


Nun haben wir für alle Entitäten eine case Klasse definiert sowie jeweils einen `Reads`, der eine entsprechende JSON
Struktur in diese Entitäten überführen kann. Setzen wir das Ganze in Bewegung:

    package controllers
    
    import play.api.mvc._
    import play.api.libs.json._
    import play.api.libs.functional.syntax._
    
    object Experiments extends Controller {
    
      final case class FormVariation(
        name: String,
        weight: Double
      )
    
      implicit val formVariationReads: Reads[FormVariation] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "weight").read[Double]
        )(FormVariation.apply _)
    
      final case class FormExperiment(
        name: String,
        scope: Double,
        formVariations: List[FormVariation]
      )
    
      implicit val formExperimentReads: Reads[FormExperiment] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "scope").read[Double] and
            (JsPath \ "variations").read[List[FormVariation]]
        )(FormExperiment.apply _)
    
      def save = Action(BodyParsers.parse.json) { request =>
        val formExperimentResult = request.body.validate(formExperimentReads)
        Ok("Got the JSON!")
      }
    }

Dieser einfache Controller mit der Methode `save` nimmt den POST Request entgegen. Er validiert den Request Body, dies
sorgt automatisch auch für eine Transformation in die Zielklassen.

Die Operation kann gelingen oder fehlschlagen - die beiden Ergebnisarten kann man einfach mit einem `match` behandeln:

    package controllers

    import play.api.mvc._
    import play.api.libs.json._
    import play.api.libs.functional.syntax._

    object Experiments extends Controller {

      final case class FormVariation(
        name: String,
        weight: Double
      )

      implicit val formVariationReads: Reads[FormVariation] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "weight").read[Double]
        )(FormVariation.apply _)

      final case class FormExperiment(
        name: String,
        scope: Double,
        formVariations: List[FormVariation]
      )

      implicit val formExperimentReads: Reads[FormExperiment] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "scope").read[Double] and
            (JsPath \ "variations").read[List[FormVariation]]
        )(FormExperiment.apply _)

      def save = Action(BodyParsers.parse.json) { request =>
        val formExperimentResult = request.body.validate(formExperimentReads)
        formExperimentResult match {
          case e: JsError => {
            BadRequest("Something went wrong!")
          }
          case s: JsSuccess[FormExperiment] => {
            val formExperiment = s.get
            Ok("Received the experiments with name " + formExperiment.name)
          }
        }
      }
    }

Eine eigene fachliche Validierung kann nun sehr einfach hinzugefügt werden, indem man den zu prüfenden `reads` Schritt
mit einem *Filter* verknüpft. Ein Filter prüft das jeweilige transformierte Element auf Basis einer eigenen Funktion,
die `true` bei Erfolg und `false` bei einem Fehlschlag zurückliefern muss. Ein Fehlschlag erzeugt dann einen
`ValidationError`, der zum Abbruch der Transformation und zu einem `JsError` führt:

    package controllers

    import play.api.data.validation.ValidationError
    import play.api.mvc._
    import play.api.libs.json._
    import play.api.libs.functional.syntax._

    object Experiments extends Controller {

      final case class FormVariation(
        name: String,
        weight: Double
      )

      implicit val formVariationReads: Reads[FormVariation] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "weight").read[Double]
        )(FormVariation.apply _)

      final case class FormExperiment(
        name: String,
        scope: Double,
        formVariations: List[FormVariation]
      )

      implicit val formExperimentReads: Reads[FormExperiment] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "scope").read[Double] and
            (JsPath \ "variations").read[List[FormVariation]]
              .filter(ValidationError("The sum of the variation weights must be 100.0"))(
                _.map(_.weight).sum == 100.0
              )
        )(FormExperiment.apply _)

      def save = Action(BodyParsers.parse.json) { request =>
        val formExperimentResult = request.body.validate(formExperimentReads)
        formExperimentResult match {
          case e: JsError => {
            BadRequest("Something went wrong!")
          }
          case s: JsSuccess[FormExperiment] => {
            val formExperiment = s.get
            Ok("Received the experiments with name " + formExperiment.name)
          }
        }
      }
    }

Hier ist die Prüfung sehr einfach und kann inline als anonyme Funktion deklariert werden. Bei komplexeren Prüfungen kann
man dies natürlich auch über eine zusätzliche Funktion lösen:

    package controllers

    import play.api.data.validation.ValidationError
    import play.api.mvc._
    import play.api.libs.json._
    import play.api.libs.functional.syntax._

    object Experiments extends Controller {

      private def areVariationsNamesUnique_?(formVariations: List[FormVariation]): Boolean = {
        formVariations.map(_.name).distinct.size == formVariations.size
      }

      final case class FormVariation(
        name: String,
        weight: Double
      )

      implicit val formVariationReads: Reads[FormVariation] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "weight").read[Double]
        )(FormVariation.apply _)

      final case class FormExperiment(
        name: String,
        scope: Double,
        formVariations: List[FormVariation]
      )

      implicit val formExperimentReads: Reads[FormExperiment] = (
        (JsPath \ "name").read[String] and
          (JsPath \ "scope").read[Double] and
            (JsPath \ "variations").read[List[FormVariation]]
              .filter(ValidationError("The sum of the variation weights must be 100.0"))(
                _.map(_.weight).sum == 100.0
              )
              .filter(ValidationError("Variations names must be unique"))(areVariationsNamesUnique_?)
        )(FormExperiment.apply _)

      def save = Action(BodyParsers.parse.json) { request =>
        val formExperimentResult = request.body.validate(formExperimentReads)
        formExperimentResult match {
          case e: JsError => {
            BadRequest("Something went wrong!")
          }
          case s: JsSuccess[FormExperiment] => {
            val formExperiment = s.get
            Ok("Received the experiments with name " + formExperiment.name)
          }
        }
      }
    }

Wie man hier weiterhin sieht, können mehrere Filteroperationen einfach aneinandergekettet werden. Das bei einem
Fehlschlag entstehende `JsError` Objekt enthält alle Validierungsfehler mit ihren Fehlertexten, die z.B. wie folgt zu
einer zusammenhängenden Fehlermeldung transformiert werden können:

    formExperimentResult match {
      case e: JsError => {
        BadRequest("The following validation errors occured: " +
                   e.errors.flatMap(_._2.map(_.message)) mkString ". ")
      }
      // ...
    }
