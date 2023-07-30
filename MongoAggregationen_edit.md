# Praktische MongoDb Aggregationen

- [Praktische MongoDb Aggregationen](#praktische-mongodb-aggregationen)
  - [Definition](#definition)
  - [Tipps und Richtlinien](#tipps-und-richtlinien)
    - [Kompositionsfähigkeit für mehr Produktivität](#kompositionsfähigkeit-für-mehr-produktivität)
    - [Bessere Alternativen zu einer $project -Stage](#bessere-alternativen-zu-einer-project--stage)
      - [Wann sollte man $set \& $unset verwenden?](#wann-sollte-man-set--unset-verwenden)
      - [Wann $project zu verwenden ist](#wann-project-zu-verwenden-ist)
    - [Verwendung von Explain-Plänen](#verwendung-von-explain-plänen)
      - [Verständnis des Explain-Plans](#verständnis-des-explain-plans)
    - [Überlegungen zur Pipeline-Leistung](#überlegungen-zur-pipeline-leistung)
      - [Streaming vs. Blockierungsphasen beachten Reihenfolge](#streaming-vs-blockierungsphasen-beachten-reihenfolge)
        - [$sort Speicherverbrauch und Risikominderung](#sort-speicherverbrauch-und-risikominderung)
        - [$group-Speicherverbrauch und Risikominderung](#group-speicherverbrauch-und-risikominderung)
      - [Vermeidung des Unwinding und Umgruppierens von Dokumenten, nur um Array-Elemente zu verarbeiten](#vermeidung-des-unwinding-und-umgruppierens-von-dokumenten-nur-um-array-elemente-zu-verarbeiten)
      - [Förderung von Match-Filtern, die früh in der Pipeline auftauchen](#förderung-von-match-filtern-die-früh-in-der-pipeline-auftauchen)
        - [Prüfen, ob das Vorziehen einer 'Full Match' möglich ist](#prüfen-ob-das-vorziehen-einer-full-match-möglich-ist)
        - [Prüfen, ob das Vorziehen eines 'Partial Match' möglich ist](#prüfen-ob-das-vorziehen-eines-partial-match-möglich-ist)
    - [Expressions Erklärungen](#expressions-erklärungen)
      - [Aggregation Expression zusammenfassen](#aggregation-expression-zusammenfassen)
      - [Was erzeugen Ausdrücke?](#was-erzeugen-ausdrücke)
      - [Können alle Stufen Ausdrücke verwenden?](#können-alle-stufen-ausdrücke-verwenden)
      - [Was hat es mit $expr innerhalb von $match auf sich?](#was-hat-es-mit-expr-innerhalb-von-match-auf-sich)
      - [Einschränkungen bei der Verwendung von Ausdrücken mit $match](#einschränkungen-bei-der-verwendung-von-ausdrücken-mit-match)
    - [Überlegungen zum Thema Sharding](#überlegungen-zum-thema-sharding)
      - [Kurzübersicht über geshardete Cluster](#kurzübersicht-über-geshardete-cluster)
      - [Einschränkungen bei der geshardeten Aggregation](#einschränkungen-bei-der-geshardeten-aggregation)
      - [Wo wird eine geshardete Aggregation ausgeführt?](#wo-wird-eine-geshardete-aggregation-ausgeführt)
      - [Aufteilung der Pipeline zur Laufzeit](#aufteilung-der-pipeline-zur-laufzeit)
      - [Ausführung des Shards-Teils der gespaltenen Pipeline](#ausführung-des-shards-teils-der-gespaltenen-pipeline)
      - [Ausführung des Merger-Teils der gespaltenen Pipeline (falls vorhanden)](#ausführung-des-merger-teils-der-gespaltenen-pipeline-falls-vorhanden)
      - [Zusammenfassung der Ansätze zur Ausführung geshardeter Pipelines](#zusammenfassung-der-ansätze-zur-ausführung-geshardeter-pipelines)
      - [Performanstipps für geshardete Aggregationen](#performanstipps-für-geshardete-aggregationen)
    - [Erweiterte Verwendung von Ausdrücken zur Verarbeitung von Arrays](#erweiterte-verwendung-von-ausdrücken-zur-verarbeitung-von-arrays)
      - ["If-Else" Bedingter Vergleich](#if-else-bedingter-vergleich)
      - [Die "Macht" der Array-Operatoren](#die-macht-der-array-operatoren)
      - ["For-Each"-Schleife zur Transformation eines Arrays](#for-each-schleife-zur-transformation-eines-arrays)
      - ["For-Each"-Schleife zum Berechnen eines Summenwerts aus einem Array](#for-each-schleife-zum-berechnen-eines-summenwerts-aus-einem-array)
      - ["For-Each"-Schleife zum Auffinden eines Array-Elements](#for-each-schleife-zum-auffinden-eines-array-elements)
      - [Reproduzieren des $map-Verhaltens mit $reduce](#reproduzieren-des-map-verhaltens-mit-reduce)
      - [Hinzufügen neuer Felder zu bestehenden Objekten in einem Array](#hinzufügen-neuer-felder-zu-bestehenden-objekten-in-einem-array)
      - [Rudimentäre Schemareflexion mit Arrays](#rudimentäre-schemareflexion-mit-arrays)



## Definition

Das Aggregations-Framework von MongoDB ermöglicht es Benutzern, eine Analyse- oder Datenverarbeitungs-Anfrage, die mit einer speziellen Aggregations-Sprache geschrieben wurde, an die Datenbank zu senden und sie anhand der darin enthaltenen Daten auszuführen. Das Framework besteht aus zwei Teilen:

- Die Aggregations-API, die in den MongoDB-Treiber eingebettet ist und es der Applikation ermöglicht, eine Aggregationsaufgabe, eine sogenannte Pipeline, zu definieren und an die Datenbank zu senden, damit diese sie verarbeitet.
- Die Runtime (Laufzeitumgebung) für die Aggregation, die in der Datenbank läuft und die Pipeline-Anforderung von der Applikation erhält, um die Pipeline anhand der gespeicherten Daten auszuführen.

Die Aggregations-Sprache von MongoDB ist Turing-vollständig und in der Lage, jedes Geschäftsproblem zu lösen. Gleichzeitig ist sie jedoch eine eigenwillige domänenspezifische Sprache [(DSL)](https://en.wikipedia.org/wiki/Domain-specific_language). Es ist sogar möglich, einen [Bitcoin-Miner](https://github.com/johnlpage/MongoAggMiner/blob/master/aggmine3.js) mit MongoDB-Aggregationen zu programmieren, wie in einem Beispielprojekt auf GitHub gezeigt wird.

Die Aggregations-Pipeline-Sprache von MongoDB ist auf datenorientierte Problemlösungen ausgerichtet. Sie ist im Wesentlichen eine deklarative Programmiersprache und keine imperative Programmiersprache. Man kann sie auch als funktionale Programmiersprache und nicht als prozedurale Programmiersprache betrachten. Eine Aggregationspipeline ist eine geordnete Abfolge von deklarativen Anweisungen, die als Stufen bezeichnet werden. Dabei bildet die gesamte Ausgabe einer Stufe die gesamte Eingabe der nächsten Stufe und so weiter, ohne Seiteneffekte.

## Tipps und Richtlinien
### Kompositionsfähigkeit für mehr Produktivität

Eine Aggregationspipeline ist eine geordnete Folge von Anweisungen, die als Stufen bezeichnet werden. Der gesamte Ausgabewert einer Stufe bildet die Eingabe für die nächste Stufe, und so weiter, ohne Nebeneffekte. Pipelines weisen eine hohe Komponierbarkeit auf, bei der Stufen zustandslose, eigenständige Komponenten sind, die in verschiedenen Kombinationen (Pipelines) ausgewählt und zusammengestellt werden können, um spezifische Anforderungen zu erfüllen. Diese Komponierbarkeit fördert iteratives Prototyping, wobei nach jedem Inkrement eine einfache Testdurchführung möglich ist. Mit den Aggregationen von MongoDB können komplexe Probleme in unkomplizierte, einzelne Stufen unterteilt werden, wobei jeder Schritt zuerst isoliert entwickelt und getestet werden kann.

### Bessere Alternativen zu einer $project -Stage

Das wichtigste Werkzeug in der MongoDB-Abfragesprache (MQL) zur Definition oder Einschränkung der zurückzugebenden Felder ist eine Projektion. Im MongoDB Aggregation Framework ist die $project-Stufe die analoge Möglichkeit zur Angabe von Feldern, die ein- oder ausgeschlossen werden sollen.
Allerdings bringt $project einige Probleme mit der Benutzerfreundlichkeit mit sich:
1. \$project ist verwirrend und nicht intuitiv. Es kann nur gewählt werden, um Felder in einer einzigen Phase einzuschließen oder auszuschließen, aber nicht beides.
2. \$project ist wortreich und unflexibel. Sollte es sich um ein neu definiertes oder ein überarbeitetes Feld handeln, so sind alle anderen in der Projektion genannten Felder einzubeziehen.

In MongoDB Version 4.2 wurden die Stages "$set" und "$unset" eingeführt, die in den meisten Fällen der Verwendung von "$project" zur Deklaration von Feldein- und -ausschlüssen vorzuziehen sind. Sie machen die Absicht des Codes viel klarer, führen zu weniger ausführlichen Pipelines und reduzieren vor allem die Notwendigkeit, eine Pipeline zu refaktorieren, wenn sich das Datenmodell weiterentwickelt

Geschichte, Altlasten und Verhalten  von \$addFields -> \$set/\$unset
In MongoDB Version 3.4 wurden einige der Nachteile von \$project behoben, indem eine neue Stufe \$addFields eingeführt wurde, die das gleiche Verhalten wie \$set aufweist. \$set kam später als \$addFields und \$set ist eigentlich nur ein Alias für \$addFields. Damals bot das Aggregation Framework jedoch kein direktes Äquivalent zu \$unset. Sowohl die \$set- als auch die \$unset-Stufen sind in modernen Versionen von MongoDB verfügbar, und ihr jeweiliger Zweck lässt sich aus ihren Namen (\$set vs. \$unset) ableiten. Der Name \$addFields spiegelt nicht vollständig wider, dass Sie bestehende Felder ändern können, anstatt nur neue Felder hinzuzufügen. In diesem Artikel wird \$set gegenüber \$addFields bevorzugt, um die Konsistenz zu fördern und eine Verwechslung zu vermeiden. Wenn Sie jedoch \$addFields bevorzugen, verwenden Sie es stattdessen, da es keinen Unterschied im Verhalten gibt.

#### Wann sollte man \$set & \$unset verwenden?

\$set & \$unset sollten verwendet werden, wenn die meisten Felder in den Eingabe-Datensätzen beibehalten werden müssen und eine kleine Teilmenge von Feldern hinzufügt, geändert oder entfernen werden soll.


Zum Beispiel gibt es eine Sammlung von Kreditkartenzahlungsbelegen, die den folgenden ähneln:
<code>

    // INPUT (ein Datensatz aus der Quellen-Sammlung, der von einer Aggregation bearbeitet werden soll)
    {
        _id: ObjectId("6044faa70b2c21f8705d8954"),
        card_name: "Mrs. Jane A. Doe",
        card_num: "1234567890123456",
        card_expiry: "2023-08-31T23:59:59.736Z",
        card_sec_code: "123",
        card_provider_name: "Credit MasterCard Gold",
        transaction_id: "eb1bd77836e8713656d9bf2debba8900",
        transaction_date: ISODate("2021-01-13T09:32:07.000Z"),
        transaction_curncy_code: "GBP",
        transaction_amount: NumberDecimal("501.98"),
        reported: true
    }
</code>

  Eine Aggregationspipeline ist erforderlich, um geänderte Versionen der Dokumente zu erstellen:

<code>
    // OUTPUT  (ein Datensatz in den Ergebnissen der durchgeführten Aggregation)
    {
        card_name: "Mrs. Jane A. Doe",
            card_num: "1234567890123456",
        card_expiry: ISODate("2023-08-31T23:59:59.736Z"), // Feldtyp aus Text konvertiert
        card_sec_code: "123",
        card_provider_name: "Credit MasterCard Gold",
        transaction_id: "eb1bd77836e8713656d9bf2debba8900",
        transaction_date: ISODate("2021-01-13T09:32:07.000Z"),
        transaction_curncy_code: "GBP",
        transaction_amount: NumberDecimal("501.98"),
        reported: true,
        card_type: "CREDIT"                               // Neu hinzugefügtes Feld für den literalen Wert
    }
</code>

Naiverweise könnte man beschließen, eine Aggregationspipeline zu erstellen, die eine $project-Stufe verwendet, um diese Transformation zu erreichen:

<code>
// SCHLECHT
[
  {"$project": {
    // Ändern eines Feldes + Hinzufügen eines neuen Feldes
    "card_expiry": {"$dateFromString": {"dateString": "$card_expiry"}},
    "card_type": "CREDIT",        

    // Alle anderen Felder müssen nun benannt werden, damit diese Felder erhalten bleiben
    "card_name": 1,
    "card_num": 1,
    "card_sec_code": 1,
    "card_provider_name": 1,
    "transaction_id": 1,
    "transaction_date": 1,
    "transaction_curncy_code": 1,
    "transaction_amount": 1,
    "reported": 1,                
    
    // Feld _id entfernen
    "_id": 0,
  }},
]
</code>

Die Stufe der Pipeline ist recht lang, und da eine $project-Stufe zum Ändern/Hinzufügen von zwei Feldern verwendet wird, sollte auch jedes andere vorhandene Feld aus den Quelldatensätzen für die Aufnahme explizit genannt werden. Ein besserer Ansatz wäre, stattdessen $set und $unset zu verwenden:

<code>
// GUT
[
  {"$set": {
    // Geändertes + neues Feld
    "card_expiry": {"$dateFromString": {"dateString": "$card_expiry"}},
    "card_type": "CREDIT",        
  }},
  
  {"$unset": [
    // Feld _id entfernen
    "_id",
  ]},
]
</code>

#### Wann \$project zu verwenden ist

Eine \$project-Stufe sollte verwendet werden, wenn sich die gewünschte Form der Ausgabedokumente stark von der Form der Eingabedokumente unterscheidet. Dies ist häufig der Fall, wenn die meisten der ursprünglichen Felder nicht einbezogen werden müssen.

Eine weitere Anforderung ist, dass derselbe Eingabewert für den nächsten Ergebnisbeleg verwendet wird:
<code>
    // OUTPUT  (ein Datensatz in den Ergebnissen der durchgeführten Aggregation)
    {
      transaction_info: { 
        date: ISODate("2021-01-13T09:32:07.000Z"),
        amount: NumberDecimal("501.98")
      },
      status: "REPORTED"
    }
</code>

Die Verwendung von $set/$unset in der Pipeline, um diese Ausgabestruktur zu erreichen, wäre sehr wortreich und würde die Benennung aller Felder erfordern:

<code>
  // SCHLECHT
  [
    {"$set": {
      // Einige Felder hinzufügen
      "transaction_info.date": "$transaction_date",
      "transaction_info.amount": "$transaction_amount",
      "status": {"$cond": {"if": "$reported", "then": "REPORTED", "else": "UNREPORTED"}},
    }},
    
    {"$unset": [
      // Feld _id entfernen
      "_id",

      // Muss alle anderen bestehenden Felder nennen, die weggelassen werden sollen
      "card_name",
      "card_num",
      "card_expiry",
      "card_sec_code",
      "card_provider_name",
      "transaction_id",
      "transaction_date",
      "transaction_curncy_code",
      "transaction_amount",
      "reported",         
    ]}, 
  ]
</code>

Durch die Verwendung von $project für diese spezielle Aggregation, um die gleichen Ergebnisse zu erreichen, wird die Pipeline jedoch weniger wortreich sein. Die Pipeline ist so flexibel, dass sie nicht geändert werden muss, wenn spätere Ergänzungen des Datenmodells mit neuen, bisher unbekannten Feldern vorgenommen werden:

<code>
// GUT
[
  {"$project": {
    // Einige Felder hinzufügen
    "transaction_info.date": "$transaction_date",
    "transaction_info.amount": "$transaction_amount",
    "status": {"$cond": {"if": "$reported", "then": "REPORTED", "else": "UNREPORTED"}},
    
    // Feld _id entfernen
    "_id": 0,
  }},
]
</code>

Zusammenfassend lässt sich sagen, dass \$set (oder \$addFields) und \$unset für die Einbeziehung und den Ausschluss von Feldern immer statt \$project verwenden werden sollten. Die wichtigste Ausnahme ist, wenn Sie eine offensichtliche Anforderung für eine sehr unterschiedliche Struktur für Ergebnisdokumente haben, bei der Sie nur eine kleine Teilmenge der Eingabefelder beibehalten müssen.

### Verwendung von Explain-Plänen

Bei der Verwendung der MongoDB Query Language (MQL) zur Entwicklung von Queries ist es wichtig, den Explain-Plan für eine Query anzusehen, um festzustellen, ob der richtige Index verwendet wird und ob andere Aspekte der Query oder des Datenmodells optimiert werden sollten. Ein Explain-Plan ermöglicht das Verständnis der Leistungsauswirkungen der erstellten Abfrage.

Um den Explain-Plan für eine Aggregationspipeline anzuzeigen, kann der folgende Befehl ausgeführt werden:
<code>
    db.coll.explain().aggregate([{"$match": {"name": "Jo"}}]);
</code>

Wie bei MQL gibt es drei verschiedene Stufen, was und in welcher Ausführlichkeit vom Explain-Plan ausgegeben wird:

<code>

    // QueryPlanner verbosity  (default if no verbosity parameter provided)
    db.coll.explain("queryPlanner").aggregate(pipeline);

    // ExecutionStats verbosity
    db.coll.explain("executionStats").aggregate(pipeline);

    // AllPlansExecution verbosity
    db.coll.explain("allPlansExecution").aggregate(pipeline);
</code>

In den meisten Fällen ist die Variante executionStats der informativste Modus. Sie zeigt nicht nur den Denkprozess des Suchanfragen-Planers, sondern liefert auch tatsächliche Statistiken über den "erfolgreichen" Ausführungsplan (z. B. die insgesamt untersuchten Schlüssel, die insgesamt untersuchten Dokumente usw.). Dies ist jedoch nicht die Standardeinstellung, da zusätzlich zur Formulierung des Abfrageplans auch die Aggregation ausgeführt wird. Wenn die Quellkollektion groß oder die Pipeline suboptimal ist, wird es eine Weile dauern, bis das Ergebnis des Explain-Plans zurückgegeben wird.

Es sei darauf hingewiesen, dass die aggregate()-Funktion auch eine rudimentäre explain-Option bietet, mit der ein explain-Plan erstellt und zurückgegeben werden kann. Diese ist jedoch eingeschränkter und umständlicher in der Anwendung und sollte vermieden werden.

#### Verständnis des Explain-Plans
Die 'customer orders' Kollektion enthält Dokumente ähnlich dem folgenden Beispiel:
<code>

    {
        "customer_id": "elise_smith@myemail.com",
        "orders": [
        {
            "orderdate": ISODate("2020-01-13T09:32:07Z"),
            "product_type": "GARDEN",
            "value": NumberDecimal("99.99")
        },
        {
            "orderdate": ISODate("2020-05-30T08:35:52Z"),
            "product_type": "ELECTRONICS",
            "value": NumberDecimal("231.43")
        }
        ]
    }
</code>

Ein Index für das Feld customer_id wurde definiert. Es wurde die folgende Aggregationspipeline erstellt, um die drei teuersten Bestellungen eines Kunden mit der ID tonijones@myemail.com anzuzeigen:

<code>

    var pipeline = [
    // jede Bestellung aus dem Array Kundenbestellungen als neuen separaten Datensatz auspacken
    {"$unwind": {
            "path": "$orders",
        }},

    // Abgleich auf nur einen customer
    {"$match": {
            "customer_id": "tonijones@myemail.com",
        }},

    // Einkäufe des "customer's" nach dem teuersten zuerst Sortiert
    {"$sort" : {
            "orders.value" : -1,
        }},

    // Nur die 3 teuersten Käufe anzeigen
    {"$limit" : 3},

    // Den Wert der Bestellung als Feld der obersten Ebene verwenden
    {"$set": {
            "order_value": "$orders.value",
        }},

    // Die ID des Dokuments und die Aufträge des Unterdokuments aus den Ergebnissen entfernen
    {"$unset" : [
            "_id",
            "orders",
        ]},
    ];
</code>

Bei der Ausführung dieser Aggregation gegen einen umfangreichen Beispieldatensatz erhält man folgendes Ergebnis:
<code>
[
  {
    customer_id: 'tonijones@myemail.com',
    order_value: NumberDecimal("1024.89")
  },
  {
    customer_id: 'tonijones@myemail.com',
    order_value: NumberDecimal("187.99")
  },
  {
    customer_id: 'tonijones@myemail.com',
    order_value: NumberDecimal("4.59")
  }
]
</code>

dann wird der Abfrageplaner Teil des explain-Plans angefordert:
<code>
  db.customer_orders.explain("queryPlanner").aggregate(pipeline);
</code>

Die Ausgabe des Abfrageplans für diese Pipeline sieht folgendermaßen aus (wobei einige Informationen der Kürze halber weggelassen wurden):

<code>

    stages: [
        {
            '$cursor': {
                queryPlanner: {
                    parsedQuery: { customer_id: { '$eq': 'tonijones@myemail.com' } },
                                    winningPlan: {
                        stage: 'FETCH',
                        inputStage: {
                            stage: 'IXSCAN',
                            keyPattern: { customer_id: 1 },
                            indexName: 'customer_id_1',
                            direction: 'forward',
                            indexBounds: {
                                customer_id: [
                                    '["tonijones@myemail.com", "tonijones@myemail.com"]'
                                ]
                            }
                        }
                    },
                }
            }
        },

        { '$unwind': { path: '$orders' } },

        { '$sort': { sortKey: { 'orders.value': -1 }, limit: 3 } },

        { '$set': { order_value: '$orders.value' } },

        { '$project': { _id: false, orders: false } }
    ]
</code>

Aus diesem Abfrageplan lassen sich einige aufschlussreiche Erkenntnisse ableiten:

Um die Aggregation zu optimieren, hat die Datenbank-Engine die Pipeline neu geordnet und den zum \$match gehörenden Filter an den Anfang der Pipeline gestellt. 
Die erste Stufe der für Datenbanken optimierten Version der Pipeline ist eine interne \$cursor-Stufe, unabhängig von der Reihenfolge, in der Sie die Pipelinestufen angeordnet haben. Die \$cursor-Laufzeitstufe ist immer die erste Aktion, die für eine Aggregation ausgeführt wird.
Um die Aggregation weiter zu optimieren, hat die Datenbank-Engine die Stufen \$sort und \$limit in eine einzige spezielle interne Sortierstufe zusammengefasst, die beide Aktionen in einem Durchgang ausführen kann.
Die Ausführungsstatistiken als Teil des Explain-Plans aufgerufen werden:

Die Ausführungsstatistiken als Teil des Erklärungsplans abgefragt werden könnten:
<code>
    db.customer_orders.explain("executionStats").aggregate(pipeline);
</code>


Hier zeigt dieser Teil des Plans auch, dass die Aggregation den vorhandenen Index verwendet. Da totalKeysExamined und totalDocsExamined übereinstimmen, nutzt die Aggregation diesen Index in vollem Umfang, um die benötigten Datensätze zu identifizieren, was eine gute Nachricht ist. Dennoch bedeutet der gezielte Index nicht unbedingt, dass der Abfrageteil der Aggregation vollständig optimiert ist. Angenommen, der Cursor-Abfrageteil der Aggregation wird vollständig durch den Index erfüllt und muss keine Rohdokumente untersuchen. In diesem Fall wird es totalDocsExamined: 0 im Explain-Plan:

<code>
    executionStats: {
        nReturned: 1,
        totalKeysExamined: 1,
        totalDocsExamined: 1,
        executionStages: {
        stage: 'FETCH',
            nReturned: 1,
            works: 2,
            advanced: 1,
            docsExamined: 1,
            inputStage: {
            stage: 'IXSCAN',
                nReturned: 1,
                works: 2,
                advanced: 1,
                keyPattern: { customer_id: 1 },
            indexName: 'customer_id_1',
                direction: 'forward',
                indexBounds: {
                customer_id: [
                    '["tonijones@myemail.com", "tonijones@myemail.com"]'
                ]
            },
            keysExamined: 1,
        }
    }
    }
</code>

### Überlegungen zur Pipeline-Leistung

Wie bei jeder Programmiersprache gibt es auch hier einen Nachteil, wenn eine Aggregationspipeline zu früh optimiert wird. Es könnte eine zu komplizierte Lösung entstehen, die den Leistungsanforderungen nicht gerecht wird. Der Explain-Plan wird in der Regel in den letzten Phasen der Pipeline-Entwicklung verwendet.
#### Streaming vs. Blockierungsphasen beachten Reihenfolge

Bei der Ausführung einer Aggregationspipeline zieht die Datenbank-Engine Datensatzstapel aus dem anfänglichen Abfragecursor, der für die Quellenkollektion erstellt wurde. Das Datenbankmodul versucht dann, jeden Stapel durch die Stufen der Aggregationspipeline zu leiten. Bei den meisten Stufentypen, die als Streaming-Stufen bezeichnet werden, nimmt das Datenbankmodul die verarbeitete Charge aus einer Stufe und leitet sie sofort in den nächsten Teil der Pipeline weiter. Sie tut dies, ohne zu warten, bis alle anderen Stapel in der vorherigen Stufe angekommen sind. Zwei Arten von Stufen müssen jedoch blockieren und darauf warten, dass alle Stapel ankommen und sich in dieser Stufe ansammeln. Diese beiden Stufen werden als blockierende Stufen bezeichnet:
- \$sort
- \$group 
  
*Wenn von \$group die Rede ist, schließt dies auch andere, weniger häufig verwendete "Gruppierung"-Stufen ein, nämlich: \$bucket, \$bucketAuto, \$count, \$sortByCount & \$facet (es ist etwas weit hergeholt, \$facet als Gruppenstufe zu bezeichnen, aber im Zusammenhang mit diesem Thema ist es am besten, es so zu sehen)

##### \$sort Speicherverbrauch und Risikominderung

Bei naiver Anwendung muss eine \$sort-Stufe alle Eingabedatensätze auf einmal sehen, und daher muss der Hostserver über genügend Kapazität verfügen, um alle Eingabedaten im Speicher zu halten. Der benötigte Speicherplatz hängt stark von der anfänglichen Datengröße und dem Ausmaß ab, in dem die vorherigen Stufen die Größe reduzieren können. Außerdem können mehrere Instanzen der Aggregationspipeline gleichzeitig im Einsatz sein, zusätzlich zu anderen Datenbank-Workloads. Daher erzwingt MongoDB, dass jede Stufe auf 100 MB verbrauchten Arbeitsspeicher begrenzt ist. Die Datenbank gibt einen Fehler aus, wenn sie diese Grenze überschreitet. Um das Hindernis der Speicherbegrenzung zu umgehen, kann die Option allowDiskUse:true für die Gesamtaggregation zur Verarbeitung großer Ergebnisdatensätze gesetzt werden. Folglich wird der Sortiervorgang der Pipeline bei Bedarf auf die Festplatte ausgelagert und die Pipeline wird nicht mehr durch die 100 MB-Grenze eingeschränkt. Der Preis dafür ist jedoch eine deutlich höhere Latenzzeit und die Ausführungszeit wird sich wahrscheinlich um Größenordnungen erhöhen.

Um zu vermeiden, dass die Aggregation den gesamten Datensatz im Speicher manifestieren oder auf die Festplatte auslagern muss, versuchen Sie, Ihre Pipeline so umzugestalten, dass sie einen der folgenden Ansätze enthält (in der Reihenfolge des effektivsten zuerst):


1. Index-Sortierung verwenden. Wenn die \$sort-Stufe nicht von einer \$unwind-, \$group- oder \$project-Stufe abhängt, die ihr vorausgeht, sollte die \$sort-Stufe in die Nähe des Starts der Pipeline verschoben werden, um einen Index für die Sortierung zu erreichen. Die Aggregations-Runtime muss daher keine teure In-Memory-Sortieroperation durchführen. Die Stufe "\$sort" ist nicht unbedingt die erste Stufe in der Pipeline, da es auch eine Stufe "\$match" geben kann, die denselben Index nutzt. Der Erklärungsplan sollte immer überprüft werden, um sicherzustellen, dass das beabsichtigte Verhalten eingeleitet wird.

2. Limit mit Sortierung sollte verwendet werden. Wenn nur die erste Teilmenge von Datensätzen aus dem sortierten Datensatz benötigt wird, sollte eine \$limit-Stufe direkt nach der \$sort-Stufe hinzugefügt werden, die die Ergebnisse auf die benötigte feste Menge begrenzt (z. B. 10). Zur Laufzeit fasst die Aggregations-Engine \$sort und \$limit zu einem einzigen speziellen internen Sortierschritt zusammen, der beide Aktionen gemeinsam durchführt. Der laufende Sortierprozess muss nur die zehn Datensätze im Speicher nachverfolgen, die der gerade ausgeführten Sortier-/Limitregel entsprechen. Er muss nicht den gesamten Datensatz im Speicher halten, um die Sortierung erfolgreich durchzuführen.

3. Datensätze zum Sortieren reduzieren. Die \$sort-Stufe sollte so spät wie möglich in die Pipeline verschoben werden, und es sollte sichergestellt werden, dass frühere Stufen die Anzahl der Datensätze, die in diese späte blockierende \$sort-Stufe fließen, deutlich reduzieren. Diese blockierende Stufe hat weniger Datensätze zu verarbeiten und benötigt weniger RAM.

##### \$group-Speicherverbrauch und Risikominderung

In der Realität konzentrieren sich die meisten Gruppierungsszenarien auf das Sammeln von zusammenfassenden Daten wie Summen, Zählungen, Durchschnittswerten, Höchst- und Tiefstwerten und nicht auf Einzeldaten. In diesen Situationen werden erheblich reduzierte Ergebnisdatensätze erzeugt, die weit weniger Verarbeitungsspeicher benötigen als eine \$sort-Stufe. Im Gegensatz zu vielen Sortierungsszenarien benötigen Gruppierungsoperationen normalerweise nur einen Bruchteil des Arbeitsspeichers des Hosts.

Um einen übermäßigen Speicherverbrauch zu vermeiden, wenn eine $group-Stufe verwendet werden soll, sind die folgenden Grundsätze zu beachten:
1. Unnötige Gruppierungen vermeiden. 
2. Nur Zusammenfassungsdaten gruppieren. Wenn der Anwendungsfall es zulässt, sollte die Gruppenphase nur dazu verwendet werden, Summen, Zählungen und zusammenfassende Roll-ups zu sammeln, anstatt alle Rohdaten jedes Datensatzes, der zu einer Gruppe gehört, zu speichern.
   
#### Vermeidung des Unwinding und Umgruppierens von Dokumenten, nur um Array-Elemente zu verarbeiten

Manchmal benötigt man eine Aggregationspipeline, um den Inhalt eines Array-Feldes für jeden Datensatz zu verändern oder zu reduzieren. Zum Beispiel::
- Es sollte alle Werte im Array zu einem Gesamtfeld addiert werden.
- Es sollte nur das erste und das letzte Element des Arrays beibehalten werden
- Es sollte nur ein wiederkehrendes Feld für jedes Unterdokument im Array aufbewahren werden
- ...oder zahlreiche andere Array-Reduktionsszenarien

Um dies zu verdeutlichen, stelle man sich eine Sammlung von Einzelhandelsbestellungen vor, bei der jedes Dokument eine Reihe von Produkten enthält, die im Rahmen der Bestellung gekauft wurden, wie im folgenden Beispiel gezeigt:

<code>

    [
        {
            _id: 1197372932325,
            products: [
                {
                    prod_id: 'abc12345',
                    name: 'Asus Laptop',
                    price: NumberDecimal('429.99')
                }
            ]
        },
        {
            _id: 4433997244387,
            products: [
                {
                    prod_id: 'def45678',
                    name: 'Karcher Hose Set',
                    price: NumberDecimal('23.43')
                },
                {
                    prod_id: 'jkl77336',
                    name: 'Picky Pencil Sharpener',
                    price: NumberDecimal('0.67')
                },
                {
                    prod_id: 'xyz11228',
                    name: 'Russell Hobbs Chrome Kettle',
                    price: NumberDecimal('15.76')
                }
            ]
        }
    ]
</code>

Der Einzelhändler möchte einen Bericht über alle Bestellungen sehen, der jedoch nur die teuren Produkte enthält, die von den Kunden gekauft wurden (z. B. nur Produkte mit einem Preis von mehr als 15 Dollar). Folglich ist eine Aggregation erforderlich, um die preiswerten Produkte aus dem Array der einzelnen Bestellungen herauszufiltern.

Ein naiver Weg, diese Umwandlung zu erreichen, besteht darin, das Produkt-Array jedes Auftragsdokuments zu entflechten, um eine Zwischenmenge einzelner Produktdatensätze zu erzeugen.

<code>
    
    // SUBOPTIMAL
    var pipeline = [
        // Auspacken jedes Produkts aus dem Produkt jeder Bestellung als einen neuen separaten Datensatz 
        {
            "$unwind": {
                "path": "$products",
            }
        },

        // Nur passende Produkte im Wert von über 15,00
        {
            "$match": {
                "products.price": {
                    "$gt": NumberDecimal("15.00"),
                },
            }
        },

        // Group by product type
        {
            "$group": {
                "_id": "$_id",
                "products": {"$push": "$products"},
            }
        }
    ];
</code>

Diese Pipeline ist suboptimal, da eine \$group-Stufe eingeführt wurde, die, wie bereits in diesem Kapitel beschrieben, eine blockierende Stufe ist. Sowohl der Speicherverbrauch als auch die Ausführungszeit steigen erheblich, was bei einem großen Eingabedatensatz fatal sein kann. Es gibt eine weitaus bessere Alternative, indem man stattdessen einen der Array-Operatoren verwendet. Array-Operatoren sind manchmal weniger intuitiv zu programmieren, aber sie vermeiden die Einführung einer Blockierungsphase in die Pipeline. Folglich sind sie wesentlich effizienter, insbesondere bei großen Datensätzen. Die folgende Abbildung zeigt eine weitaus wirtschaftlichere Pipeline, die den Array-Operator \$filter anstelle der Kombination \$unwind/\$match/\$group verwendet, um das gleiche Ergebnis zu erzielen:

<code>

    // OPTIMAL

    var pipeline = [
    // Produkte im Wert von 15,00 oder weniger herausfiltern
    {
        "$set": {
            "products": {
                "$filter": {
                    "input": "$products",
                    "as": "product",
                    "cond": {"$gt": ["$$product.price", NumberDecimal("15.00")]},
                }
            },
        }
    },
    ];
</code>

Im Gegensatz zur suboptimalen Pipeline enthält die optimale Pipeline "leere Aufträge" in den Ergebnissen für die Aufträge, die nur preiswerte Artikel enthalten. Wenn dies ein Problem darstellt, können Sie am Anfang der optimalen Pipeline eine einfache \$match-Stufe mit demselben Inhalt wie die \$match-Stufe im suboptimalen Beispiel einfügen.

Um es noch einmal zu wiederholen: Es sollte niemals notwendig sein, eine \$unwind/\$group-Kombination in einer Aggregationspipeline zu verwenden, um die Elemente eines Array-Feldes für jedes Dokument einzeln zu transformieren. Eine Möglichkeit, dieses Anti-Pattern zu erkennen, ist, wenn die Pipeline eine \$group für ein \$_id-Feld enthält. Um zu vermeiden, dass eine Blockierungsphase eingeführt wird, sollte man stattdessen Array-Operatoren verwenden.
Die primäre Verwendung einer \$unwind/\$group-Kombination besteht darin, Muster über die Arrays vieler Datensätze hinweg zu korrelieren, anstatt nur den Inhalt innerhalb des Arrays eines jeden Eingabedatensatzes zu transformieren.


#### Förderung von Match-Filtern, die früh in der Pipeline auftauchen
##### Prüfen, ob das Vorziehen einer 'Full Match' möglich ist

Der Inhalt der obersten Ebene von $match ist Teil des Filters, den die Engine zunächst als erste Abfrage ausführt. Die Aggregation hat dann die beste Chance, einen Index zu nutzen. Manchmal wird jedoch eine $match-Stufe später in einer Pipeline definiert, um einen Filter für ein Feld durchzuführen, das die Pipeline in einer früheren Stufe berechnet hat. Das berechnete Feld ist in der ursprünglichen Eingabeauflistung der Pipeline nicht vorhanden.
Einige Beispiele hierfür sind:
- Eine Pipeline, in der eine $group-Stufe ein neues Gesamtfeld auf der Grundlage eines Akkumulatoroperators erstellt. Später in der Pipeline filtert eine $match-Stufe Gruppen, bei denen die Summe jeder Gruppe größer als 1000 ist.
- Eine Pipeline, in der eine $set-Stufe einen neuen Gesamtfeldwert berechnet, der auf der Addition aller Elemente eines Array-Feldes in jedem Dokument basiert. Später in der Pipeline filtert eine $match-Stufe Dokumente, bei denen die Summe weniger als 50 beträgt.

##### Prüfen, ob das Vorziehen eines 'Partial Match' möglich ist

Es kann Fälle geben, in denen ein berechneter Wert nicht aufgedröselt werden kann. Es kann jedoch immer noch möglich sein, eine zusätzliche $match-Stufe einzufügen, um eine teilweise Übereinstimmung mit dem Abfragecursor der Aggregation durchzuführen. Beispiel: Eine Pipeline, die die Werte der sensiblen Felder für das Geburtsdatum maskiert (durch berechnete maskierte 'masked_date' ersetzt). Das berechnete Feld fügt eine zufällige Anzahl von Tagen (eins bis sieben) zu jedem aktuellen Datum hinzu. Die Pipeline enthält bereits eine $match-Stufe mit dem Filter masked_date > 01-Jan-2020. Die Laufzeit kann dies aufgrund der Abhängigkeit von einem berechneten Wert nicht an den Anfang der Pipeline optimieren. Es könnte jedoch möglich sein, manuell eine zusätzliche $match-Stufe am Anfang der Pipeline mit dem Filter date_of_birth > 25-Dec-2019 hinzuzufügen. Diese neue $match-Stufe nutzt einen Index und filtert Datensätze, die sieben Tage vor der bestehenden $match-Stufe liegen, aber die endgültige Ausgabe der Aggregation ist dieselbe. Der neue $match kann einige Datensätze mehr weitergeben als beabsichtigt. Später wendet die Pipeline jedoch den vorhandenen Filter masked_date > 01-Jan-2020 an, der die überzähligen Datensätze natürlich entfernt, bevor die Pipeline abgeschlossen wird.

Zusammenfassend lässt sich sagen: Wenn eine Pipeline eine $match-Stufe nutzt und der Explain-Plan zeigt, dass diese nicht an den Anfang der Pipeline verschoben wird, sollte untersucht werden, ob ein manuelles Refactoring helfen kann. Wenn der $match-Filter von einem berechneten Wert abhängt, wäre zu prüfen, ob dieser geändert oder ein zusätzliches $match hinzugefügt werden könnte, um eine effizientere Pipeline zu erhalten.

### Expressions Erklärungen
#### Aggregation Expression zusammenfassen
Aggregationsausdrücke gibt es in einer der drei Hauptvarianten:
1. Operatoren. Der Zugriff erfolgt als Objekt mit einem ``$ -Präfix``, gefolgt vom Namen der Operatorfunktion. Der "dollar-operator-name" wird als Hauptschlüssel für das Objekt verwendet.  Beispiele: ``{$arrayElemAt: ...}, {$cond: ...}, {$dateToString: ...}``
2. Feldpfade. Zugriff als Zeichenkette mit einem ``\$ -Präfix``, gefolgt von dem Feldpfad in jedem verarbeiteten Datensatz.  Beispiele: ``"$Konto.sortcode", "$Adressen.Adresse.Stadt"``
3. Variablen. Der Zugriff erfolgt über eine Zeichenkette mit einem ``$$-Präfix``, gefolgt von dem festen Namen, und fällt in drei Unterkategorien:
- Kontext-Systemvariablen. Bei Werten, die aus der Systemumgebung stammen, wird nicht jeder Eingabedatensatz, sondern eine Aggregationsstufe verarbeitet.  Beispiele: ``"$$NOW", "$$CLUSTER_TIME"``
- Marker-Flag-Systemvariablen. Zur Angabe des gewünschten Verhaltens, das an die Aggregations-Pipeline zurückgegeben werden soll.  Beispiele: ``"$$ROOT", "$$REMOVE", "$$PRUNE"``
- Benutzer-Variablen. Zum Speichern von Werten, die Sie mit einem \$let-Operator deklarieren (oder mit der let-Option einer \$lookup-Stufe oder als Option einer \$map- oder \$filter-Stufe).  Beispiele: ``"$$product_name_var", "$$orderIdVal"``
Sie können diese drei Kategorien von Aggregationsausdrücken bei der Bearbeitung von Eingabedatensätzen kombinieren und so komplexe Vergleiche und Transformationen von Daten durchführen. 
 
 <code>
 
    json"customer_info": {"$cond": {
        "if":   {"$eq": ["$customer_info.category", "SENSITIVE"]},
        "then": "$$REMOVE",
            "else": "$customer_info",
    }}
</code>

#### Was erzeugen Ausdrücke?
Ein Ausdruck kann ein Operator (z. B. ``{$concat: ...}``), eine Variable (z. B. ``"$$ROOT"``) oder ein Feldpfad (z. B. ``"$address"``) sein. In all diesen Fällen ist ein Ausdruck einfach etwas, das dynamisch ein neues JSON/BSON-Datentyp-Element auffüllt und zurückgibt und eines der folgenden sein kann:
- a Number (einschließlich integer, long, float, double, decimal128)
- a String (UTF-8)
- a Boolean
- a DateTime (UTC)
- an Array
- an Object

Ein bestimmter Ausdruck kann jedoch die Rückgabe auf einen oder wenige dieser Typen beschränken. Zum Beispiel der ``{$concat: ...}`` Operator, der mehrere Strings kombiniert, kann nur einen String-Datentyp (oder null) liefern. Die Variable ``"$$ROOT"`` kann nur ein Objekt zurückgeben, das sich auf das Stammdokument bezieht, das gerade in der Pipelinestufe verarbeitet wird. Ein Feldpfad (z. B. "$address") ist anders und kann ein Element eines beliebigen Datentyps zurückgeben, je nachdem, worauf sich das Feld im aktuellen Eingabedokument bezieht. 
Zusammenfassend lässt sich sagen, dass Feldpfade und Bindung von Benutzervariablen Ausdrücke sind, die abhängig von ihrem Kontext zur Laufzeit jeden JSON/BSON-Datentyp zurückgeben können. Bei den anderen Arten von Ausdrücken (Operatoren, Kontext-Systemvariablen und Marker-Flag-Systemvariablen) ist der Datentyp, den sie zurückgeben können, auf einen oder eine bestimmte Anzahl dokumentierter Typen festgelegt. Um den genauen Datentyp zu ermitteln, der von diesen spezifischen Operatoren erzeugt wird, müssen Sie die[ Dokumentation der Aggregation Pipeline Quick Reference](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/) konsultieren.
Bei Ausdrücken der Kategorie Operator kann ein Ausdruck auch andere Ausdrücke als Parameter annehmen, wodurch sie kombinierbar werden.

#### Können alle Stufen Ausdrücke verwenden?
Es gibt viele Arten von Stufen im Aggregation Framework, die die Einbettung von Ausdrücken nicht zulassen. Beispiele für einige der am häufigsten verwendeten dieser Stufen sind:
- \$match
- \$limit
- \$skip
- \$sort
- \$count
- \$lookup
- \$out
Einige dieser Stufen können eine Überraschung sein, wenn man vorher nie wirklich darüber nachgedacht hat. Es könnte sein, dass $match der überraschendste Punkt in dieser Liste ist. Der Inhalt einer $match-Stufe ist lediglich eine Reihe von "Query Conditions" mit der gleichen Syntax wie MQL und kein Aggregationsausdruck. Dafür gibt es einen guten Grund. Die Aggregations-Engine verwendet die MQL Query Engine wieder, um eine "reguläre" Abfrage gegen die Sammlung durchzuführen, sodass die Query Engine alle ihre üblichen Optimierungen verwenden kann. Die Abfragebedingungen werden unverändert von der $match-Stufe am Anfang der Pipeline übernommen. Daher muss der $match-Filter die gleiche Syntax wie MQL verwenden. In den meisten Fällen muss nur eine der aufgeführten Stufen aussagekräftiger sein: die $match-Stufe, aber diese Stufe ist bereits flexibel, da sie auf MQL-Abfragebedingungen basiert. Manchmal ist jedoch selbst MQL nicht aussagekräftig genug, um eine Regel zur Identifizierung von Datensätzen, die in einer Aggregation beibehalten werden sollen, ausreichend zu definieren. 

#### Was hat es mit $expr innerhalb von $match auf sich?

Die zuvor geäußerte Allgemeinheit, dass $match keine Expressions unterstützt, ist eigentlich nicht ganz korrekt. Mit Version 3.6 von MongoDB wurde der Operator $expr eingeführt, der in eine $match-Stufe (oder in MQL) integriert werden kann, um beim Filtern von Datensätzen Aggregationsausdrücke zu verwenden. Im Wesentlichen ermöglicht dies der Abfragelaufzeit von MongoDB (die den $match einer Aggregation ausführt), Ausdrücke wiederzuverwenden, die von der Aggregationslaufzeit von MongoDB bereitgestellt werden. Innerhalb eines $expr-Operators kann jeder zusammengesetzte Ausdruck aus $-Operatorfunktionen, $-Feldpfaden und $$-Variablen enthalten sein. In einigen Situationen ist es erforderlich, $expr innerhalb einer $match-Stufe zu verwenden. Beispiele hierfür sind:

- Die Anforderung, zwei Felder desselben Datensatzes zu vergleichen und auf der Grundlage des Vergleichsergebnisses zu entscheiden, ob der Datensatz behalten werden soll

- Die Anforderung, eine Berechnung auf der Grundlage von Werten aus mehreren vorhandenen Feldern in jedem Datensatz durchzuführen und dann die Berechnung mit einer Konstanten zu vergleichen

Diese sind in einer Aggregation (oder MQL find()) unmöglich, wenn reguläre $match-Abfragebedingungen verwendet werden.

#### Einschränkungen bei der Verwendung von Ausdrücken mit $match

Es sollte beachtet werden, dass es Einschränkungen gibt, wann die Laufzeit von einem Index profitieren kann, wenn ein \$expr-Operator innerhalb einer \$match-Stufe verwendet wird. Dies hängt teilweise von der MongoDB-Version ab. Mit \$expr können Sie einen \$eq-Vergleichsoperator mit einigen Einschränkungen nutzen, darunter die Unmöglichkeit, einen Multi-Schlüssel-Index zu verwenden. Bei der Verwendung von $expr wird ein \$eq-Vergleichsoperator mit einigen Einschränkungen genutzt, einschließlich der Unmöglichkeit, einen Index mit mehreren Schlüsseln zu verwenden. Bei MongoDB-Versionen vor 5.0 kann bei Verwendung eines "range"-Vergleichsoperators (\$gt, \$gte, \$lt und \$lte) kein Index verwendet werden, um das Feld abzugleichen, aber ab Version 5.0 funktioniert dies problemlos.

### Überlegungen zum Thema Sharding

MongoDB-Sharding:

 // Zweck/ Ziel von Sharding

-   Horizontale Skalierung der Datenbank, um mehr Daten zu speichern und die Transaktionsdurchsatzrate zu erhöhen
-   Skalierung der Analysearbeiten, um Aggregationen schneller abzuschließen (möglicherweise werden Aggregationen parallel auf mehreren Shards ausgeführt)


#### Kurzübersicht über geshardete Cluster

In einem geshardeten Cluster läuft jeder Shard auf einer separaten dedizierten Gruppe von Host-Computern und speichert einen Teil einer Datenmenge. Basierend auf einem Shard-Schlüssel jedes Dokuments (vom Benutzer definiert), gruppiert das System Untermengen von Dokumenten zu "Chunks" zusammen. Der Cluster verteilt diese Chunks auf seine Shards. In der gleichen Datenbank können sowohl geshardete als auch ungeshardete Sammlungen gehalten werden. Alle ungeshardeten Sammlungen werden auf einem bestimmten Shard im Cluster gespeichert, das als "Primary Shard" bezeichnet wird.

#### Einschränkungen bei der geshardeten Aggregation

Einige der MongoDB-Stufen unterstützen nur teilweise geshardete Aggregationen, abhängig von der DB-Version. Diese Stufen beziehen sich alle auf eine zweite Sammlung zusätzlich zur Pipelinesourcen-Eingabesammlung. In jedem Fall kann die Pipeline eine geshardete Sammlung als ihre Quelle verwenden, aber die zweite referenzierte Sammlung muss ungeshardet sein. Die betroffenen Stufen und Versionen sind:

-   $lookup: In Versionen vor 5.1 muss die zweite referenzierte Sammlung, mit der zusammengeführt werden soll, ungeshardet sein
-   $graphLookup: In Versionen vor 5.1 muss die zweite referenzierte Sammlung, mit der zusammengeführt werden soll, ungeshardet sein
-   $out: In allen Versionen muss die zweite referenzierte Sammlung, die als Ziel der Ausgabe der Aggregation verwendet wird, ungeshardet sein. Es kann stattdessen einen $merge-Abschnitt verwendet werden, um das Aggregationsergebnis in eine geshardete Collection zu schreiben.


#### Wo wird eine geshardete Aggregation ausgeführt?

Geshardete Cluster bieten die Möglichkeit, die Antwortzeiten von Aggregationen zu verringern. Dies wird jedoch nicht immer der Fall sein, da bestimmte Arten von Pipelines erfordern, dass erhebliche Datenmengen von mehreren Shards zusammengeführt werden, um weiterverarbeitet zu werden. Die Antwortzeit der Aggregation könnte sich in solchen Fällen in die entgegengesetzte Richtung entwickeln und aufgrund des erheblichen Netzwerkübertragungs- und Marshalling-Overheads deutlich länger dauern.

#### Aufteilung der Pipeline zur Laufzeit

Ein geshardeter Cluster wird versuchen, die Stufen einer Pipeline parallel auf jedem Shard auszuführen, der die erforderlichen Daten enthält. Sortier- und Gruppierungsstufen ($sort, $bucket, $bucketAuto, $count und $sortByCount) müssen jedoch an allen Daten an einem Ort ("blockierende Stufen", beschrieben in _Performanceüberlegungen für Pipelines_) ausgeführt werden. Sobald ein blockierende Stufe in der Pipeline auftritt, wird das Aggregationssystem die Pipeline an dem Punkt, an dem das blockierende Stufen auftritt, in zwei Teile aufteilen. Der erste Teil der Pipeline (benannt als "Shards Part" im Explain-Plan) kann parallel auf mehreren Stufen ausgeführt werden, während der restliche Teil der Pipeline (benannt als "Merge part" im Explain-Plan) an einem Ort ausgeführt wird.

#### Ausführung des Shards-Teils der gespaltenen Pipeline

Wenn eine MongoDb eine Anfrage zur Ausführung einer Aggregationspipeline erhält, muss es festlegen, wo in der Pipeline sich der Shards-Teil befindet. Es wird versuchen, dies auf einer relevanten Teilmenge von Shards auszuführen, anstatt die Arbeit an alle zu senden. Wenn beispielsweise der Filter für $match den Shard-Schlüssel oder einen Präfix des Shard-Schlüssels enthält, kann das Mongos eine zielgerichtete Operation ausführen und den Shards-Teil der gespaltenen Pipeline auf die entsprechenden Shards leiten. Wenn der $match-Filter beispielsweise eine exakte Übereinstimmung mit einem Shard-Schlüsselwert für die Source-Sammlung enthält, kann die Pipeline auf einen einzigen Shard zielen und muss nicht in zwei Teile aufgeteilt werden.

#### Ausführung des Merger-Teils der gespaltenen Pipeline (falls vorhanden)

Die Aggregations-Runtime wendet eine Reihe von Regeln an, um festzulegen, wo der Merger-Teil einer Aggregationspipeline für einen geshardeten Cluster ausgeführt werden soll und ob überhaupt eine Spaltung erforderlich ist. Die Erreichung von entweder der Targeted-Shard-Ausführung (2) oder Mongos-Merge (4) ist in der Regel das bevorzugte Ergebnis für optimale Leistung:

1.  **Primary-Shard Merge**: Die Pipeline enthält eine Stufe, das auf eine zweite ungeshardete Sammlung verweist:
	-   $out oder $lookup und $graphLookup in der MongoDB-Version vor 5.1
	-   Referenzierung einer ungeshardeten Sammlung von einer $merge-Stufe oder von $lookup oder $graphLookup (MongoDB 5.1 oder höher)
2.  **Targeted-Shard-Ausführung**: Die Runtime kann sicherstellen, dass die Pipeline mit der erforderlichen Teilmenge der Quell-Collection-Daten auf nur einem Shard übereinstimmt. Das Verhalten, das auf einen einzigen Shard festgelegt wird, tritt auch dann auf, wenn die Pipeline eine $merge, $lookup oder $graphLookup-Stufe enthält, das auf eine zweite geshardete Sammlung verweist, die Dokumente über mehrere Shards verteilt enthält.
3.  **Any-Shard Merge**: Wenn allowDiskUsage:true konfiguriert ist und eines der folgenden auch zutrifft, muss die Aggregations-Runtime den Merger-Teil der gespaltenen Pipeline auf einem zufällig gewählten Shard ausführen:
    -   Die Pipeline enthält ein Gruppierungsstufe
    -   Die Pipeline enthält ein $sort-Stufe und ein darauf folgendes blockierendes Stufe (ein Gruppierungs- oder $sort-Stufe) tritt später auf
4. **Mongos Merge**: (Standardverfahren) Wenn der Merger-Teil der Pipeline nur streaming-Stufen enthält, geht die Runtime davon aus, dass es sicher ist, den verbleibenden Teil der Pipeline dem Mongos zu übergeben.


#### Zusammenfassung der Ansätze zur Ausführung geshardeter Pipelines

Zusammenfassend anstreben die Aggregations-Runtime danach, eine Pipeline auf der Teilmenge von Shards auszuführen, die die erforderlichen Daten enthält. Wenn die Runtime die Pipeline splitten muss, um Gruppierung oder Sortierung durchzuführen, führt sie die endgültige Merge-Arbeit auf einem Mongos durch, wenn möglich. Das Zusammenführen auf einem Mongos hilft, die Anzahl der erforderlichen Netzwerk-Hops und die Ausführungszeit zu reduzieren.

#### Performanstipps für geshardete Aggregationen

Alle empfohlenen Aggregationsoptimierungen, die in den Pipeline-Leistungen dargelegt werden, gelten ebenfalls für geshardete Cluster, sind aber bei der Ausführung von Aggregationen in geshardeten Clustern sogar noch kritischer:

1.  Sortierung - Es sollte Indexsortierung verwendet werden: Der Shard-Teil der gespaltenen Pipeline, der auf jedem Shard parallel ausgeführt wird, wird dadurch vermeiden, dass ein In-Memory-Sortierungsvorgang durchgeführt wird.
2.  Sortierung - Es sollte Limit mit Sortierung verwendet werden:  Die Runtime muss weniger Zwischeneinträge über das Netzwerk übertragen, von jedem Shard, der den Shard-Teil einer gespaltenen Pipeline ausführt, an den Ort, an dem der Merger-Teil der Pipeline ausgeführt wird.
3.  Sortierung - Die zu sortierenden Einträge sollten verringert werden: Wenn die ersten beiden Punkte unmöglich sind, sollte eine $sort-Stufe so spät wie möglich verschoben werden.
4. Gruppierung - Unnötige Gruppierung sollte vermieden werden: Möglichst Array-Operatoren anstelle von $unwind- und $group-Stufe sollten verwendet werden, um das Splitting der Pipeline in ein unnötig eingeführtes $group-Stufe zu vermeiden.
5.  Gruppierung - Nur Zusammenfassungsdaten sollten gruppiert werden: Die Runtime muss weniger berechnete Einträge über das Netzwerk von jedem Shard, auf dem der Shard-Teil einer gespaltenen Pipeline ausgeführt wird, an den Standort des Merger-Teils übertragen.
6.  Es sollte angestrebt werden, dass Filter mit Match früh im Pipeline-Prozess erscheinen, damit die Runtime weniger Einträge an den Standort des Merger-Teils streamen muss.

Es sollten insbesondere für geshardete Cluster zwei weitere Leistungsoptimierungen angestrebt werden:

1.  Aggregationen sollten nach Möglichkeit auf einen Shard zielen, sollten gesucht werden: Wenn möglich, sollte ein $match-Stufe mit einem Filter auf einen Shard-Schlüsselwert (oder Shard-Schlüsselpräfixwert) hinzugefügt werden.
2.  Eine gespaltene Pipeline sollte nach Möglichkeit auf einem Mongos zusammengeführt wird, sollten gesucht werden: Wenn eine Pipeline ein $group-Stufe (oder ein $sort-Stufe, gefolgt von einem $group- oder $sort-Stufe) hat, was dazu führt, dass sich die Pipeline teilt, sollte allowDiskUse:true vermieden werden, falls möglich. Dies reduziert die Menge an Zwischendaten, die über das Netzwerk übertragen werden und somit die Latenz.
   
### Erweiterte Verwendung von Ausdrücken zur Verarbeitung von Arrays

Das Aggregation-Framework bietet einen umfangreichen Set von Aggregationsoperator ausdrücken für die Analyse und Manipulation von Arrays. 
Bei der Leistungsoptimierung sind diese Array-Ausdrücke von entscheidender Bedeutung, um das Unwinding und die Neugruppierung von Dokumenten zu vermeiden, bei denen Sie nur das Array jedes Dokuments isoliert verarbeiten müssen. 
Für die meisten Situationen, in denen ein Array manipuliert werden muss, gibt es in der Regel einen einzigen Array-Operator-Ausdruck, der zur Lösung der Anforderung verwendet werden kann.
Gelegentlich kann es dennoch erforderlich sein, einen Verbund aus mehreren Expressions auf niedrigerer Ebene zusammenzustellen, um eine komplexe Array-Manipulationsaufgabe zu erledigen. 
Wie bei Aggregationspipelines im Allgemein, besteht ein großer Teil der Herausforderung darin, die Denkweise an ein funktionales Programmierparadigma anzupassen, anstatt an ein prozedurales. Ein Vergleich mit prozeduralen Ansätzen kann bei der Beschreibung der Array-Manipulations-Pipelinelogik für mehr Klarheit sorgen.

#### "If-Else" Bedingter Vergleich

Nehmen wir das trivialisierte Szenario eines Einzelhändlers, der die Gesamtkosten der Bestellung eines Kunden berechnen möchte. Der Kunde könnte mehrere Artikel desselben Produkts bestellen, und der Verkäufer erhält einen Rabatt, wenn mehr als 5 Artikel des Produkts in der Bestellung enthalten sind.

In einem prozeduralen Stil von JavaScript, könnte es den folgenden Code zur Berechnung der gesamten Bestellung Kosten geschrieben:

```
let order = {"product" : "WizzyWidget", "price": 25.99, "qty": 8};
// Procedural style JavaScript
if (order.qty > 5) {
  order.cost = order.price * order.qty * 0.9;
} else {
  order.cost = order.price * order.qty;
}
```

Mit diesem Code wird die Bestellung des Kunden wie folgt geändert, um die Gesamtkosten einzubeziehen:

```{product: 'WizzyWidget', qty: 8, price: 25.99, cost: 187.128}```

Um ein ähnliches Ergebnis in einer Aggregationspipeline zu erzielen, könnte Folgendes verwendet werden:

```
db.customer_orders.insertOne(order);

var pipeline = [
  {"$set": {
    "cost": {
      "$cond": { 
        "if":   {"$gte": ["$qty", 5]}, 
        "then": {"$multiply": ["$price", "$qty", 0.9]},
        "else": {"$multiply": ["$price", "$qty"]},
      }    
    },
  }},
];

db.customer_orders.aggregate(pipeline);
```

Diese Pipeline erzeugt die folgende Ausgabe, bei der das Kundenauftragsdokument transformiert wird:
```{product: 'WizzyWidget', price: 25.99, qty: 8, cost: 187.128}```

Wenn ein funktionaler Programmieransatz in JavaScript verwendet werden könnte, würde der Code eher wie folgt aussehen, um das gleiche Ergebnis zu erzielen:
```
// Functional style JavaScript
order.cost = (
  (order.qty > 5) ?
  (order.price * order.qty * 0.9) :
  (order.price * order.qty)
 );
```

Der Aufbau des JavaScript-Codes im funktionalen Stil entspricht eher der Struktur der Aggregationspipeline.

Der JavaScript-Beispielcode funktioniert nur für jeweils ein Dokument und müsste geändert werden, um eine Liste von Datensätzen zu durchlaufen. Dieser JavaScript-Code müsste jedes Dokument aus der Datenbank zurück zum Client holen, die Änderungen vornehmen und dann das Ergebnis zurück in die Datenbank schreiben. Stattdessen arbeitet die Logik der Aggregationspipeline mit jedem Dokument vor Ort in der Datenbank, was die Leistung und Effizienz deutlich erhöht.

#### Die "Macht" der Array-Operatoren

Wenn Daten transformiert oder aus einem Array-Feld extrahiert werden sollen und ein einzelner High-Level-Array-Operator (z.B. $avg, $max, $filter) nicht das gewünschte Ergebnis liefert, sind die Array-Operatoren $map und $reduce das Mittel der Wahl. Diese beiden "Power"-Operatoren ermöglichen es, durch ein Array zu iterieren, jede beliebige komplexe Logik für jedes Array-Element auszuführen und das Ergebnis zu sammeln, um es in die Ausgabe einer Stufe aufzunehmen.
Je nach den spezifischen Anforderungen kann der eine oder der andere verwendet werden, um das Feld eines Arrays zu verarbeiten, aber nicht beide zusammen. Hier finden Sie eine Erklärung dieser beiden "Power"-Operatoren:

- \$map. Ermöglicht die Angabe einer Logik, die für jedes Element im Array, das der Operator durchläuft, ausgeführt werden soll, wobei als Endergebnis ein Array zurückgegeben wird. Typischerweise wird $map verwendet, um jedes Array Element zu mutieren und dann das transformierte Array zurückzugeben. Der $map-Operator gibt den Inhalt des aktuellen Array-Elements über eine spezielle Variable mit dem Standardnamen \$\$this an eine Logik weiter.

- \$reduce.  In ähnlicher Weise könnte eine Logik angegeben werden, die für jedes Element in einem Array ausgeführt wird, das der Operator durchläuft, aber stattdessen einen einzelnen Wert (und nicht ein Array) als Endergebnis zurückgibt. Normalerweise verwenden Sie \$reduce, um eine Zusammenfassung zu berechnen, nachdem Sie jedes Array-Element analysiert haben. Wie der \$map-Operator bietet der \$reduce-Operator einer Logik Zugriff auf das aktuelle Array-Element über die Variable \$\$value, damit eine Logik bei der Akkumulation des Einzelergebnisses (z. B. das Multiplikationsergebnis) aktualisiert werden kann.

#### "For-Each"-Schleife zur Transformation eines Arrays
 
Nehmen wir an, eine Liste der von einem Kunden bestellten Produkte soll verarbeitet und das Array der Produktnamen in Großbuchstaben umgewandelt werden. In einem prozeduralen JavaScript-Stil könnte der folgende Code geschrieben werden, um in einer Schleife durch jedes Produkt im Array zu gehen und seinen Namen in Großbuchstaben umzuwandeln:

```
let order = {
  "orderId": "AB12345",
  "products": ["Laptop", "Kettle", "Phone", "Microwave"]
};
 
// Procedural style JavaScript
for (let pos in order.products) {
  order.products[pos] = order.products[pos].toUpperCase();
}
```

Dieser Code ändert die Produktnamen der Bestellung wie folgt, wobei die Produktnamen jetzt in Großbuchstaben geschrieben werden:
``` {orderId: 'AB12345', products: ['LAPTOP', 'KETTLE', 'PHONE', 'MICROWAVE']}```

Um ein ähnliches Ergebnis in einer Aggregationspipeline zu erzielen, könnte Folgendes verwendet werden:
```
db.orders.insertOne(order);

var pipeline = [
  {"$set": {
    "products": {
      "$map": {
        "input": "$products",
        "as": "product",
        "in": {"$toUpper": "$$product"}
      }
    }
  }}
];

db.orders.aggregate(pipeline);
```

Diese Pipeline erzeugt die folgende Ausgabe, wobei das Bestelldokument umgewandelt wird in:

```{orderId: 'AB12345', products: ['LAPTOP', 'KETTLE', 'PHONE', 'MICROWAVE']```

Bei Verwendung des funktionalen Stils in JavaScript würde der Schleifencode eher dem folgenden ähneln, um das gleiche Ergebnis zu erzielen:

```
// Functional style JavaScript
order.products = order.products.map(
  product => {
    return product.toUpperCase(); 
  }
);
```

Der Vergleich eines Aggregationsausdrucks des \$map-Operators mit einer JavaScript map()-Array-Funktion ist viel besser geeignet, um die Funktionsweise des Operators zu erklären.

#### "For-Each"-Schleife zum Berechnen eines Summenwerts aus einem Array

Angenommen, es ist notwendig, eine Liste der von einem Kunden bestellten Produkte zu verarbeiten, aber ein einzelnes zusammenfassendes Stringfeld aus diesem Array durch Verkettung aller Produktnamen aus dem Array zu erzeugen. In einer prozeduralen JavaScript-Stil, könnte es die folgende codiert werden, um die Produktnamen Zusammenfassung Feld zu produzieren:
```
let order = {
  "orderId": "AB12345",
  "products": ["Laptop", "Kettle", "Phone", "Microwave"]
};
 
order.productList = "";

// Procedural style JavaScript
for (const pos in order.products) {
  order.productList += order.products[pos] + "; ";
}
```

Dieser Code bringt die folgende Ausgabe mit einem neuen String-Feld `productList` hervor, das die Namen aller Produkte in der Bestellung enthält, getrennt durch Semikolons:

```
{
  orderId: 'AB12345',
  products: [ 'Laptop', 'Kettle', 'Phone', 'Microwave' ],
  productList: 'Laptop; Kettle; Phone; Microwave; '
}
```

Es könnte in der folgenden Pipeline verwendet werden, um ein ähnliches Ergebnis zu erzielen:

```
db.orders.insertOne(order);

var pipeline = [
  {"$set": {
    "productList": {
      "$reduce": {
        "input": "$products",
        "initialValue": "",
        "in": {
          "$concat": ["$$value", "$$this", "; "]
        }            
      }
    }
  }}
];

db.orders.aggregate(pipeline);
```

Hier durchläuft ein Ausdruck des Operators `$$reduce` jedes Produkt in der Eingabearray in einer Schleife und verkettet den Namen jedes Produkts zu einer akkumulierenden Zeichenfolge. Der Ausdruck `$$this` wird verwendet, um bei jeder Iteration auf den Wert des aktuellen Array-Elements zuzugreifen. Bei jeder Iteration wird der Ausdruck `$$value` verwendet, um auf den endgültigen Ausgabewert zu referenzieren, an den der aktuelle Produktstring (+ Begrenzer) angehängt wird.

Diese Pipeline erzeugt die gleiche Ausgabe, wie die obere, mit Javascript gebaut, wo es die Bestellung Dokument umwandelt. Mit einem funktionalen Ansatz in JavaScript, könnte es den folgenden Code verwendet haben, um das gleiche Ergebnis zu erzielen:

```
// Functional style JavaScript
order.productList = order.products.reduce(
  (previousValue, currentValue) => {
    return previousValue + currentValue + "; ";
  },
  ""
);
```

#### "For-Each"-Schleife zum Auffinden eines Array-Elements

Man kann sich vorstellen, dass Daten über Gebäude auf einem Campus gespeichert werden, wobei jedes Gebäudedokument eine Reihe von Räumen mit ihren Größen (Breite und Länge) enthält. Für ein Raumreservierungssystem könnte es erforderlich sein, den ersten Raum im Gebäude zu finden, der über eine ausreichende Grundfläche für eine bestimmte Anzahl von Besprechungsplattformen verfügt. Nachfolgend ist ein Beispiel für die Daten eines Gebäudes zu sehen, die in die Datenbank geladen werden könnten, mit einer Reihe von Räumen und deren Abmessungen in Metern:
```
db.buildings.insertOne({
  "building": "WestAnnex-1",
  "room_sizes": [
    {"width": 9, "length": 5},
    {"width": 8, "length": 7},
    {"width": 7, "length": 9},
    {"width": 9, "length": 8},
  ]
});
```

Es sollte eine Pipeline erstellt werden, um einen geeigneten Besprechungsraum zu finden, der eine Ausgabe ähnlich der folgenden erzeugt. Das Ergebnis sollte ein neu hinzugefügtes Feld `firstLargeEnoughRoomArrayIndex` enthalten, um die Array-Position des ersten gefundenen Raums mit ausreichender Kapazität anzugeben.

```
{
  building: 'WestAnnex-1',
  room_sizes: [
    { width: 9, length: 5 },
    { width: 8, length: 7 },
    { width: 7, length: 9 },
    { width: 9, length: 8 }
  ],
  firstLargeEnoughRoomArrayIndex: 2
}
```

Nachfolgend ist eine geeignete Pipeline dargestellt, die durch die Elemente des Raumarrays iteriert und die Position des ersten Elements mit einer berechneten Fläche von mehr als 60 m² erfasst:

```
var pipeline = [
  {"$set": {
    "firstLargeEnoughRoomArrayIndex": {
      "$reduce": {
        "input": {"$range": [0, {"$size": "$room_sizes"}]},
        "initialValue": -1,
        "in": {
          "$cond": { 
            "if": {
              "$and": [
                // IF ALREADY FOUND DON'T CONSIDER SUBSEQUENT ELEMENTS
                {"$lt": ["$$value", 0]}, 
                // IF WIDTH x LENGTH > 60
                {"$gt": [
                  {"$multiply": [
                    {"$getField": {"input": {"$arrayElemAt": ["$room_sizes", "$$this"]}, "field": "width"}},
                    {"$getField": {"input": {"$arrayElemAt": ["$room_sizes", "$$this"]}, "field": "length"}},
                  ]},
                  60
                ]}
              ]
            }, 
            // IF ROOM SIZE IS BIG ENOUGH CAPTURE ITS ARRAY POSITION
            "then": "$$this",  
            // IF ROOM SIZE NOT BIG ENOUGH RETAIN EXISTING VALUE (-1)
            "else": "$$value"  
          }            
        }            
      }
    }
  }}
];

db.buildings.aggregate(pipeline);
```

Hier wird der Operator `$reduce` erneut verwendet, um eine Schleife zu bilden und schließlich einen einzigen Wert zurückzugeben. Die Pipeline verwendet jedoch eine generierte Folge inkrementeller Zahlen als Eingabe und nicht das bestehende Array-Feld in jedem Quelldokument. Der Operator `$range` wird verwendet, um diese Sequenz zu erstellen, die die gleiche Größe wie das Array-Feld "rooms" in jedem Dokument hat. Die Pipeline verwendet diesen Ansatz, um die Array-Position des passenden Raums mit Hilfe der Variablen `$$this` zu verfolgen. Für jede Iteration berechnet die Pipeline den Bereich des Array-Raumelements. Wenn die Größe größer als 60 ist, ordnet die Pipeline die aktuelle Array-Position (dargestellt durch `$$this`) dem Endergebnis (dargestellt durch `$$value`) zu.

Die "Iterator"-Array-Ausdrücke haben kein Konzept eines _break_-Befehls, den prozedurale Programmiersprachen normalerweise bieten. Auch wenn die Ausführungslogik bereits einen ausreichend großen Raum gefunden hat, wird der Schleifenprozess daher durch die verbleibenden Array-Elemente fortgesetzt. Folglich muss die Pipeline-Logik bei jeder Iteration eine Prüfung beinhalten, um zu vermeiden, dass der Endwert (die Variable `$$value`) überschrieben wird, wenn er bereits einen Wert hat. Bei großen Arrays mit einigen hundert oder mehr Elementen hat eine Aggregationspipeline natürlich einen spürbaren Einfluss auf die Latenzzeit, wenn sie die verbleibenden Arrayelemente durchläuft, obwohl die Logik das erforderliche Element bereits identifiziert hat.

Nehmen wir an, es soll nur das erste passende Array-Element für einen Raum mit ausreichender Bodenfläche zurückgegeben werden, nicht sein Index. In diesem Fall kann die Pipeline einfacher sein, indem man `$filter` verwendet, um die Array-Elemente auf diejenigen mit ausreichend Platz zu trimmen und dann den `$first` Operator, um nur das erste Element aus dem Filter zu holen. Es würde eine Pipeline ähnlich der folgenden verwendet werden:

```
var pipeline = [
  {"$set": {
    "firstLargeEnoughRoom": {
      "$first": {
        "$filter": { 
          "input": "$room_sizes", 
          "as": "room",
          "cond": {
            "$gt": [
              {"$multiply": ["$$room.width", "$$room.length"]},
              60
            ]
          } 
        }    
      }
    }
  }}
];

db.buildings.aggregate(pipeline);
```

Diese Pipeline erzeugt dieselbe Ausgabe.

#### Reproduzieren des $map-Verhaltens mit $reduce

Es ist möglich, das `$map`-Verhalten mit `$reduce` zu implementieren, um ein Array zu transformieren. Diese Methode ist komplexer, aber man muss sie vielleicht in einigen seltenen Fällen verwenden. Bevor ein Beispiel gezeigt wird, warum das so ist, wollen wir zunächst ein einfacheres Beispiel für die Verwendung von `$map` und dann `$reduce` vergleichen, um das Gleiche zu erreichen.

Nehmen wir an, man hat einige Sensormesswerte für ein Gerät aufgezeichnet:

```
db.deviceReadings.insertOne({
  "device": "A1",
  "readings": [27, 282, 38, -1, 187]
});
```

Man stelle sich vor, dass eine umgewandelte Version des Arrays "Lesungen" erzeugt werden soll, wobei die ID des Geräts mit jeder Lesung im Array verkettet wird. Man könnte die Pipeline so aufbauen, dass sie eine Ausgabe ähnlich der folgenden erzeugt, mit dem neu eingefügten Array-Feld:

```
{
  device: 'A1',
  readings: [ 27, 282, 38, -1, 187 ],
  deviceReadings: [ 'A1:27', 'A1:282', 'A1:38', 'A1:-1', 'A1:187' ]
}
```

Dies könnte durch die Verwendung des Operatorausdrucks `$map` in der folgenden Pipeline erreicht werden:
```
var pipeline = [
  {"$set": {
    "deviceReadings": {
      "$map": {
        "input": "$readings",
        "as": "reading",
        "in": {
          "$concat": ["$device", ":", {"$toString": "$$reading"}]
        }
      }
    }
  }}
];

db.deviceReadings.aggregate(pipeline);
```

Dasselbe könnte auch mit dem Operatorausdruck `$reduce` in der folgenden Pipeline erreicht werden:
```
var pipeline = [
  {"$set": {
    "deviceReadings": {
      "$reduce": {
        "input": "$readings",
        "initialValue": [],
        "in": {
          "$concatArrays": [
            "$$value",
            [{"$concat": ["$device", ":", {"$toString": "$$this"}]}]
          ]
        }
      }
    }
  }}
];

db.deviceReadings.aggregate(pipeline);
```

Da es sich um einen Aggregationsrahmen handelt, gibt es natürlich mehrere Möglichkeiten, das gleiche Problem zu lösen.
Zusammenfassend lässt sich sagen, dass eine "map"-Stufe typischerweise verwendet wird, wenn das Verhältnis von Eingabeelementen zu Ausgabeelementen gleich ist (d.h. many-to-many oder _M:M_). Eine `$reduce`-Stufe wird verwendet, wenn das Verhältnis von Eingabeelementen zu Ausgabeelementen viele-zu-eins ist (d.h. _M:1_). In Situationen, in denen das Verhältnis der Eingabeelemente viele zu wenige ist (d.h. _M:N_), wird man statt `$map` immer zu `$reduce` mit seinem "Null-Array-Verkettungstrick" greifen, wenn `$filter` nicht ausreicht.


#### Hinzufügen neuer Felder zu bestehenden Objekten in einem Array

Einer der wichtigsten Verwendungszwecke des Operatorausdrucks "$map" ist das Hinzufügen weiterer Daten zu jedem vorhandenen Objekt in einem Array. Nehmen wir an, dass ein Satz von Einzelhandelsbestellungen persistiert, wobei jedes Bestelldokument ein Array von Bestellpositionen enthält. Jeder Auftragsposten im Array enthält den Namen des Produkts, den Stückpreis und die gekaufte Menge, wie im folgenden Beispiel gezeigt:

```
db.orders.insertOne({
    "custid": "jdoe@acme.com",
    "items": [
      {
        "product" : "WizzyWidget", 
        "unitPrice": 25.99,
        "qty": 8,
      },
      {
        "product" : "HighEndGizmo", 
        "unitPrice": 33.24,
        "qty": 3,
      }
    ]
});
```

Jetzt müssen die Gesamtkosten für jedes Produkt (Menge x Stückpreis) berechnet und zu den entsprechenden Auftragspositionen im Array hinzugefügt werden. Um dies zu erreichen, könnte eine Pipeline verwendet werden, ähnlich der folgenden:
```
var pipeline = [
  {"$set": {
    "items": {
      "$map": {
        "input": "$items",
        "as": "item",
        "in": {
          "product": "$$item.product",
          "unitPrice": "$$item.unitPrice",
          "qty": "$$item.qty",
          "cost": {"$multiply": ["$$item.unitPrice", "$$item.qty"]}},
        }
      }
    }
  }
];

db.orders.aggregate(pipeline);
```

Hier erstellt die Pipeline für jedes Element im Quell-Array ein Element im neuen Array, indem die drei Felder des alten Elements (`product`, `unitPrice` und `quantity`) explizit eingefügt werden und ein neues berechnetes Feld (`cost`) hinzugefügt wird. Die Pipeline erzeugt die folgende Ausgabe:

```
{
  custid: 'jdoe@acme.com',
  items: [
    {
      product: 'WizzyWidget',
      unitPrice: 25.99,
      qty: 8,
      cost: 187.128
    },
    {
      product: 'HighEndGizmo',
      unitPrice: 33.24,
      qty: 3,
      cost: 99.72
    }
  ]
}
```

Ähnlich wie bei den Nachteilen der Verwendung einer `$project`-Stufe in einer Pipeline, wird der `$map`-Code durch die explizite Benennung jedes Feldes in dem zu speichernden Array-Element belastet.  Das könnte ermüdend sein, wenn jedes Array-Element viele Felder hat. Wenn sich das Datenmodell weiterentwickelt und im Laufe der Zeit neue Feldtypen in den Array-Elementen auftauchen, ist man außerdem gezwungen, jedes Mal zu seiner Pipeline zurückzukehren und sie neu zu strukturieren, um diese neu eingeführten Felder aufzunehmen. Genau wie bei der Verwendung von `$set` anstatt von `$project` für eine Pipelinestufe, gibt es eine bessere Lösung, die es erlaubt, alle existierenden Array-Elementfelder beizubehalten und neue hinzuzufügen, wenn Arrays verarbeitet werden. Eine gute Lösung ist die Verwendung des [`$$mergeObjects`](https://docs.mongodb.com/manual/reference/operator/aggregation/merge/) Operator Ausdrucks, um alle existierenden Felder plus die neu berechneten Felder in jedem neuen Array Element zu kombinieren. `$mergeObjects` nimmt ein Array von Objekten und kombiniert die Felder von allen Objekten des Arrays zu einem einzigen Objekt.
Um `$mergeObjects` in dieser Situation zu verwenden, sollte das aktuelle Array-Element als erster Parameter an `$mergeObjects` übergeben werden. Der zweite übergebene Parameter ist ein neues Objekt, das jedes berechnete Feld enthält. Im folgenden Beispiel fügt der Code nur ein generiertes Feld hinzu, kann aber bei Bedarf auch mehrere generierte Felder in dieses neue Objekt aufnehmen:
```
var pipeline = [
  {"$set": {
    "items": {
      "$map": {
        "input": "$items",
        "as": "item",
        "in": {
          "$mergeObjects": [
            "$$item",            
            {"cost": {"$multiply": ["$$item.unitPrice", "$$item.qty"]}},
          ]
        }
      }
    }
  }}
];

db.orders.aggregate(pipeline);
```

Diese Pipeline erzeugt die gleiche Ausgabe wie die vorherige "hardcoded field names" Pipeline, aber mit dem Vorteil, dass sie neue Feldtypen, die in der Zukunft im Quell-Array erscheinen, berücksichtigt.
Anstatt `$mergeObjects` zu verwenden, gibt es eine alternative und etwas ausführlichere Kombination von drei verschiedenen Array-Operator-Ausdrücken, die in ähnlicher Weise verwendet werden können, um alle bestehenden Array-Element-Felder zu erhalten und neue hinzuzufügen. Diese drei Operatoren sind:
- [`$ObjectToArray`](https://docs.mongodb.com/manual/reference/operator/aggregation/objectToArray/). Dies konvertiert ein Objekt, das verschiedene Feld-Schlüssel/Wert-Paare enthält, in ein Array von Objekten, wobei jedes Objekt zwei Felder hat: `k`, das den Namen des Feldes enthält, und `v`, das den Wert des Feldes enthält. Zum Beispiel: `{Höhe: 170, Gewicht: 60}` wird zu `[{k: 'Höhe', v: 170}, {k: 'Gewicht', v: 60}]`
- [`$concatArrays`](https://docs.mongodb.com/manual/reference/operator/aggregation/concatArrays/). Dies kombiniert den Inhalt mehrerer Arrays zu einem einzigen Array-Ergebnis.
- [`$arrayToObject`](https://docs.mongodb.com/manual/reference/operator/aggregation/arrayToObject/). Dies wandelt ein Array in ein Objekt um, indem es die Umkehrung des `$objectToArray` Operators durchführt. Zum Beispiel: `{k: 'height', v: 170}, {k: 'weight', v: 60}, {k: 'shoeSize', v: 10}]` wird zu `{height: 170, weight: 60, shoeSize: 10}`

Die Pipeline unten zeigt die Kombination in Aktion für denselben Datensatz von Einzelhandelsbestellungen wie zuvor, wobei die neu berechneten Gesamtkosten für jedes Produkt hinzugefügt werden:
```
var pipeline = [
  {"$set": {
    "items": {
      "$map": {
        "input": "$items",
        "as": "item",
        "in": {
          "$arrayToObject": {
            "$concatArrays": [
              {"$objectToArray": "$$item"},            
              [{
                "k": "cost",
                "v": {"$multiply": ["$$item.unitPrice", "$$item.qty"]},
              }]              
            ]
          }
        }
      }
    }}
  }
];

db.orders.aggregate(pipeline);
```

Wenn dies dasselbe erreicht wie die Verwendung von `$mergeObjects`, aber ausführlicher ist, warum sollte man sich dann die Mühe machen, dieses Muster zu verwenden? Nun, in den meisten Fällen würde es das nicht. Eine Situation, in der die ausführlichere Kombination verwendet wird, ist, wenn der Name des Feldes eines Array-Elements dynamisch gesetzt werden muss, zusätzlich zu seinem Wert. Anstatt das berechnete Gesamtfeld als "cost" zu benennen, sollte der Name des Feldes auch den Namen des Produktes wiedergeben (z.B. "costForWizzyWidget", "costForHighEndGizmo"). Dies könnte erreicht werden, indem man die Methode `$arrayToObject`/`$concatArrays`/`$objectToArray` anstelle der Methode `$mergeObjects` verwendet, wie folgt:
```
var pipeline = [
  {"$set": {
    "items": {
      "$map": {
        "input": "$items",
        "as": "item",
        "in": {
          "$arrayToObject": {
            "$concatArrays": [
              {"$objectToArray": "$$item"},            
              [{
                "k": {"$concat": ["costFor", "$$item.product"]},
                "v": {"$multiply": ["$$item.unitPrice", "$$item.qty"]},
              }]              
            ]
          }
        }
      }
    }}
  }
];

db.orders.aggregate(pipeline);
```

Die Pipeline hat alle vorhandenen Felder der Array-Elemente beibehalten und jedem Element ein neues Feld mit einem dynamisch generierten Namen hinzugefügt.
```
{
  custid: 'jdoe@acme.com',
  items: [
    {
      product: 'WizzyWidget',
      unitPrice: 25.99,
      qty: 8,
      costForWizzyWidget: 207.92
    },
    {
      product: 'HighEndGizmo',
      unitPrice: 33.24,
      qty: 3,
      costForHighEndGizmo: 99.72
    }
  ]
}
```

#### Rudimentäre Schemareflexion mit Arrays

Als letztes "lustiges" Beispiel sehen wir uns an, wie man einen "$objectToArray"-Operatorausdruck verwendet, um mit [reflection](https://en.wikipedia.org/wiki/Reflective_programming) die Form einer Sammlung von Dokumenten als Teil eines benutzerdefinierten Schema-Analyse-Tools zu analysieren. Solche Reflection-Fähigkeiten sind in Datenbanken mit einem flexiblen Datenmodell wie MongoDB, wo die enthaltenen Felder von Dokument zu Dokument variieren können, unerlässlich.

Nehmen wir an, man hätte eine Kollektion von Kundendokumenten, ähnlich der folgenden:
```
db.customers.insertMany([
  {
    "_id": ObjectId('6064381b7aa89666258201fd'),
    "email": 'elsie_smith@myemail.com',
    "dateOfBirth": ISODate('1991-05-30T08:35:52.000Z'),
    "accNnumber": 123456,
    "balance": NumberDecimal("9.99"),
    "address": {
      "firstLine": "1 High Street",
      "city": "Newtown",
      "postcode": "NW1 1AB",
    },
    "telNums": ["07664883721", "01027483028"],
    "optedOutOfMarketing": true,
  },
  {
    "_id": ObjectId('734947394bb73732923293ed'),
    "email": 'jon.jones@coolemail.com',
    "dateOfBirth": ISODate('1993-07-11T22:01:47.000Z'),
    "accNnumber": 567890,
    "balance": NumberDecimal("299.22"),
    "telNums": "07836226281",
    "contactPrefernece": "email",
  },
]);
```

In der Schema-Analyse-Pipeline wird "$objectToArray" verwendet, um den Namen und den Typ jedes Feldes der obersten Ebene im Dokument wie folgt zu erfassen:
```
var pipeline = [
  {"$project": {
    "_id": 0,
    "schema": {
      "$map": {
        "input": {"$objectToArray": "$$ROOT"},
        "as": "field",
        "in": {
          "fieldname": "$$field.k",
          "type": {"$type": "$$field.v"},          
        }
      }
    }
  }}
];

db.customers.aggregate(pipeline);
```

Für die beiden Beispieldokumente in der Kollektion gibt die Pipeline das Folgende aus:
```
{
  schema: [
    {fieldname: '_id', type: 'objectId'},
    {fieldname: 'email', type: 'string'},
    {fieldname: 'dateOfBirth', type: 'date'},
    {fieldname: 'accNnumber', type: 'int'},
    {fieldname: 'balance', type: 'decimal'},
    {fieldname: 'address', type: 'object'},
    {fieldname: 'telNums', type: 'array'},
    {fieldname: 'optedOutOfMarketing', type: 'bool'}
  ]
},
{
  schema: [
    {fieldname: '_id', type: 'objectId'},
    {fieldname: 'email', type: 'string'},
    {fieldname: 'dateOfBirth', type: 'date'},
    {fieldname: 'accNnumber', type: 'int'},
    {fieldname: 'balance', type: 'decimal'},
    {fieldname: 'telNums', type: 'string'},
    {fieldname: 'contactPrefernece', type: 'string'}
}
```

Die Schwierigkeit bei diesem grundlegenden Pipeline-Ansatz besteht darin, dass die Ausgabe zu lang und zu komplex wird, sobald viele Dokumente in der Kollektion vorhanden sind, um gemeinsame Schemamuster zu erkennen. Stattdessen sollte eine Kombination der Stufen "$unwind" und "$group" hinzugefügt werden, um wiederkehrende Felder, die übereinstimmen, zu sammeln. Das generierte Ergebnis sollte auch hervorheben, wenn derselbe Feldname in mehreren Dokumenten, aber mit unterschiedlichen Datentypen, vorkommt. Hier ist die verbesserte Pipeline:
```
var pipeline = [
  {"$project": {
    "_id": 0,
    "schema": {
      "$map": {
        "input": {"$objectToArray": "$$ROOT"},
        "as": "field",
        "in": {
          "fieldname": "$$field.k",
          "type": {"$type": "$$field.v"},          
        }
      }
    }
  }},
  
  {"$unwind": "$schema"},

  {"$group": {
    "_id": "$schema.fieldname",
    "types": {"$addToSet": "$schema.type"},
  }},
  
  {"$set": {
    "fieldname": "$_id",
    "_id": "$$REMOVE",
  }},
];

db.customers.aggregate(pipeline);
```

Die Ausgabe dieser Pipeline bietet nun eine viel verständlichere Zusammenfassung, wie unten dargestellt:
```
{fieldname: '_id', types: ['objectId']},
{fieldname: 'address', types: ['object']},
{fieldname: 'email', types: ['string']},
{fieldname: 'telNums', types: ['string', 'array']},
{fieldname: 'contactPrefernece', types: ['string']},
{fieldname: 'accNnumber', types: ['int']},
{fieldname: 'balance', types: ['decimal']},
{fieldname: 'dateOfBirth', types: ['date']},
{fieldname: 'optedOutOfMarketing', types: ['bool']}
```

Dieses Ergebnis zeigt, dass das Feld "telNums" in den Dokumenten einen von zwei verschiedenen Datentypen haben kann.