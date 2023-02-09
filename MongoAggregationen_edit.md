Praktische MongoDb Aggregationen

- [Definition](#definition)
  - [Kompositionsfähigkeit für mehr Produktivität](#kompositionsfähigkeit-für-mehr-produktivität)
  - [Bessere Alternativen zu einer $project -Stage](#bessere-alternativen-zu-einer-project--stage)
  - [Wann sollte man $set \& $unset verwenden?](#wann-sollte-man-set--unset-verwenden)
    - [Wann $project zu verwenden ist](#wann-project-zu-verwenden-ist)
  - [Verwendung von Explain-Plänen](#verwendung-von-explain-plänen)
    - [Den Explain-Plan verstehen](#den-explain-plan-verstehen)
- [Überlegungen zur Pipeline-Leistung](#überlegungen-zur-pipeline-leistung)
  - [1. Beachten Sie die Reihenfolge von Streaming und Blocking Stage](#1-beachten-sie-die-reihenfolge-von-streaming-und-blocking-stage)
  - [$sort Speicherverbrauch und Risikominderung](#sort-speicherverbrauch-und-risikominderung)
  - [$group-Speicherverbrauch und Risikominderung](#group-speicherverbrauch-und-risikominderung)
- [Erklärung und Verwendung von Ausdrücken](#erklärung-und-verwendung-von-ausdrücken)
  - [Was erzeugen Ausdrücke?](#was-erzeugen-ausdrücke)
    - [Können alle Stufen Ausdrücke verwenden?](#können-alle-stufen-ausdrücke-verwenden)
  - [Einschränkungen bei der Verwendung von Ausdrücken mit $match](#einschränkungen-bei-der-verwendung-von-ausdrücken-mit-match)
  - [Sharding-Betrachtung](#sharding-betrachtung)
- [Kurzübersicht über geshardete Clusters](#kurzübersicht-über-geshardete-clusters)
- [Einschränkungen bei der geshardeten Aggregation](#einschränkungen-bei-der-geshardeten-aggregation)
- [Wo wird eine geshardete Aggregation ausgeführt?](#wo-wird-eine-geshardete-aggregation-ausgeführt)
    - [Spaltung der Pipeline zur Laufzeit](#spaltung-der-pipeline-zur-laufzeit)
    - [Ausführung des Shards-Teils der gespaltenen Pipeline](#ausführung-des-shards-teils-der-gespaltenen-pipeline)
    - [Ausführung des Merger-Teils der gespaltenen Pipeline (falls vorhanden)](#ausführung-des-merger-teils-der-gespaltenen-pipeline-falls-vorhanden)
  - [Zusammenfassung der Ansätze zur Ausführung geshardeter Pipelines](#zusammenfassung-der-ansätze-zur-ausführung-geshardeter-pipelines)
  - [Leistungstipps für geshardete Aggregationen](#leistungstipps-für-geshardete-aggregationen)
  - [Erweiterte Verwendung von Ausdrücken zur Verarbeitung von Arrays](#erweiterte-verwendung-von-ausdrücken-zur-verarbeitung-von-arrays)
      - ["If-Else" bedingter Vergleich](#if-else-bedingter-vergleich)
      - [Die "Macht" der Array-Operatoren](#die-macht-der-array-operatoren)
      - ["For-Each"-Schleife zur Umwandlung eines Arrays](#for-each-schleife-zur-umwandlung-eines-arrays)
      - ["For-Each"-Schleife zum Berechnen eines Summenwerts aus einem Array](#for-each-schleife-zum-berechnen-eines-summenwerts-aus-einem-array)
      - ["For-Each"-Schleife zum Auffinden eines Array-Elements](#for-each-schleife-zum-auffinden-eines-array-elements)
      - [Reproduzieren des $map-Verhaltens mit $reduce](#reproduzieren-des-map-verhaltens-mit-reduce)
      - [Hinzufügen neuer Felder zu bestehenden Objekten in einem Array](#hinzufügen-neuer-felder-zu-bestehenden-objekten-in-einem-array)
      - [Rudimentäre Schemareflexion mit Arrays](#rudimentäre-schemareflexion-mit-arrays)

Definition des Aggregation Frameworks: 
<ul>
<li>Aggregations-API zur Definition von Aggregations-Tasks (Pipeline)</li>
<li>Aggregations-Laufzeit</li>
</ul>

## Definition
Eigenschaften der Aggregations-Sprache von Mongo DB:
 Sie ist Turing-vollständig und in der Lage, jedes Geschäftsproblem zu lösen *. Umgekehrt ist sie eine stark eigenwillige domänenspezifische Sprache [(DSL)](https://en.wikipedia.org/wiki/Domain-specific_language).
Man kann sogar einen [Bitcoin-Miner](https://github.com/johnlpage/MongoAggMiner/blob/master/aggmine3.js) mit MongoDB-Aggregationen programmieren.
Die Aggregations-Pipeline-Sprache von MongoDB ist auf datenorientierte Problemlösungen ausgerichtet. Es handelt sich im Wesentlichen um eine deklarative Programmiersprache und nicht um eine imperative Programmiersprache. Außerdem kann sie als funktionale Programmiersprache und nicht als prozedurale Programmiersprache betrachtet werden. Eine Aggregationspipeline ist eine geordnete Reihe von deklarativen Anweisungen, die als Stufen bezeichnet werden, wobei die gesamte Ausgabe einer Stufe die gesamte Eingabe der nächsten Stufe bildet, und so weiter, ohne Seiteneffekte. 

### Kompositionsfähigkeit für mehr Produktivität

Eine Aggregationspipeline ist eine geordnete Reihe von deklarativen Anweisungen, die als Phasen bezeichnet werden.

### Bessere Alternativen zu einer $project -Stage

Geschichte, Altlasten und Verhalten  von \$addFields -> \$set/\$unset
In MongoDB Version 3.4 wurden einige der Nachteile von \$project behoben, indem eine neue Stufe \$addFields eingeführt wurde, die das gleiche Verhalten wie \$set aufweist. \$set kam später als \$addFields und \$set ist eigentlich nur ein Alias für \$addFields. Damals bot das Aggregation Framework jedoch kein direktes Äquivalent zu \$unset. Sowohl die \$set- als auch die \$unset-Stufen sind in modernen Versionen von MongoDB verfügbar, und ihr jeweiliger Zweck lässt sich aus ihren Namen (\$set vs. \$unset) ableiten. Der Name \$addFields spiegelt nicht vollständig wider, dass Sie bestehende Felder ändern können, anstatt nur neue Felder hinzuzufügen. In diesem Artikel wird \$set gegenüber \$addFields bevorzugt, um die Konsistenz zu fördern und eine Verwechslung zu vermeiden. Wenn Sie jedoch \$addFields bevorzugen, verwenden Sie es stattdessen, da es keinen Unterschied im Verhalten gibt.

### Wann sollte man \$set & \$unset verwenden?

\$set & \$unset sollten verwenden werden, wenn die meisten Felder in den Eingabe-Datensätzen beibehalten werden müssen und eine kleine Teilmenge von Feldern hinzufügen, geändert oder entfernen werden soll.


<code>

    // INPUT  (a record from the source collection to be operated on by an aggregation)
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
    
    // OUTPUT  (a record in the results of the executed aggregation)
    {
        card_name: "Mrs. Jane A. Doe",
            card_num: "1234567890123456",
        card_expiry: ISODate("2023-08-31T23:59:59.736Z"), // Field type converted from text
        card_sec_code: "123",
        card_provider_name: "Credit MasterCard Gold",
        transaction_id: "eb1bd77836e8713656d9bf2debba8900",
        transaction_date: ISODate("2021-01-13T09:32:07.000Z"),
        transaction_curncy_code: "GBP",
        transaction_amount: NumberDecimal("501.98"),
        reported: true,
        card_type: "CREDIT"                               // New added literal value field
    }
</code>

#### Wann \$project zu verwenden ist
Eine \$project-Stufe sollte verwendet werden, wenn sich die gewünschte Form der Ausgabedokumente stark von der Form der Eingabedokumente unterscheidet. Dies ist häufig der Fall, wenn die meisten der ursprünglichen Felder nicht einbeziehen werden müssen.
Die Struktur jedes Ausgabedokuments muss sich stark von der Struktur der Eingabedokumente unterscheiden, und Sie müssen viel weniger Originalfelder beibehalten:

<code>

    [{
        "$set": {
            // Modified + new field
            "card_expiry": {"$dateFromString": {"dateString": "$card_expiry"}},
            "card_type": "CREDIT"
        }
    },
    {"$unset": ["_id"]}
    ]
</code>

<code>

    // OUTPUT  (a record in the results of the executed aggregation)
    {
        transaction_info: {
            date: ISODate("2021-01-13T09:32:07.000Z"),
                amount: NumberDecimal("501.98")
        },
        status: "REPORTED"
    }

    [
    {
        "$project": {
            // Add some fields
            "transaction_info.date": "$transaction_date",
            "transaction_info.amount": "$transaction_amount",
            "status": {"$cond": {"if": "$reported", "then": "REPORTED", "else": "UNREPORTED"}},

            // Remove _id field
            "_id": 0,
        }
    },
    ]
</code>

Zusammenfassend lässt sich sagen, dass \$set (oder \$addFields) und \$unset für die Einbeziehung und den Ausschluss von Feldern immer statt \$project verwenden werden sollten. Die wichtigste Ausnahme ist, wenn Sie eine offensichtliche Anforderung für eine sehr unterschiedliche Struktur für Ergebnisdokumente haben, bei der Sie nur eine kleine Teilmenge der Eingabefelder beibehalten müssen.

### Verwendung von Explain-Plänen

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

In den meisten Fällen werden Sie feststellen, dass die Variante executionStats der informativste Modus ist. Sie zeigt nicht nur den Denkprozess des Abfrageplaners, sondern liefert auch tatsächliche Statistiken über den "erfolgreichen" Ausführungsplan (z. B. die insgesamt untersuchten Schlüssel, die insgesamt untersuchten Dokumente usw.). Dies ist jedoch nicht die Standardeinstellung, da zusätzlich zur Formulierung des Abfrageplans auch die Aggregation ausgeführt wird  Wenn die Quell-Kollektion groß oder die Pipeline suboptimal ist, wird es eine Weile dauern, bis das Ergebnis des Explain-Plans zurückgegeben wird.
Es sollte darauf geachtet werden, dass die aggregate()-Funktion auch eine rudimentäre explain-Option bietet, mit der ein explain-Plan erstellt und zurückgegeben werden kann. Diese ist jedoch eingeschränkter und umständlicher in der Anwendung, so dass Sie sie vermeiden sollten.

#### Den Explain-Plan verstehen
Die Sammlung der Kundenaufträge enthält Dokumente ähnlich dem folgenden Beispiel:

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

Sie haben einen Index für das Feld customer_id definiert. Sie erstellen die folgende Aggregationspipeline, um die drei teuersten Bestellungen eines Kunden mit der ID tonijones@myemail.com anzuzeigen:

<code>

    var pipeline = [
    // Unpack each order from customer orders array as a new separate record
    {"$unwind": {
            "path": "$orders",
        }},

    // Match on only one customer
    {"$match": {
            "customer_id": "tonijones@myemail.com",
        }},

    // Sort customer's purchases by most expensive first
    {"$sort" : {
            "orders.value" : -1,
        }},

    // Show only the top 3 most expensive purchases
    {"$limit" : 3},

    // Use the order's value as a top level field
    {"$set": {
            "order_value": "$orders.value",
        }},

    // Drop the document's id and orders sub-document from the results
    {"$unset" : [
            "_id",
            "orders",
        ]},
    ];
</code>

Sie fordern dann den Query Planner-Teil des Explain-Plans an:
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
Die erste Stufe der datenbankoptimierten Version der Pipeline ist eine interne \$cursor-Stufe.
Um die Aggregation weiter zu optimieren, hat die Datenbank-Engine die Stufen \$sort und \$limit in eine einzige spezielle interne Sortierstufe zusammengefasst, die beide Aktionen in einem Durchgang ausführen kann.
Die Ausführungsstatistiken als Teil des Explain-Plans aufgerufen werden:

<code>

    db.customer_orders.explain("executionStats").aggregate(pipeline);
</code>

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

Hier zeigt dieser Teil des Plans auch, dass die Aggregation den vorhandenen Index verwendet. Da totalKeysExamined und totalDocsExamined übereinstimmen, nutzt die Aggregation diesen Index in vollem Umfang, um die benötigten Datensätze zu identifizieren, was grundsätzlich positiv ist. Dennoch bedeutet der Zielindex nicht unbedingt, dass der Abfrageteil der Aggregation vollständig optimiert ist. Wenn es beispielsweise notwendig ist, die Latenzzeit weiter zu verringern, können Sie eine Analyse durchführen, um festzustellen, ob der Index die Abfrage vollständig abdecken kann. Angenommen, der Cursor-Abfrageteil der Aggregation wird vollständig durch den Index abgedeckt und muss keine Rohdokumente untersuchen. In diesem Fall sehen Sie totalDocsExamined: 0 im Explain-Plan.

## Überlegungen zur Pipeline-Leistung
### 1. Beachten Sie die Reihenfolge von Streaming und Blocking Stage

Bei der Ausführung einer Aggregationspipeline zieht die Datenbank-Engine Datensatzstapel aus dem anfänglichen Abfragecursor, der für die Quellenkollektion erstellt wurde. Das Datenbankmodul versucht dann, jeden Stapel durch die Stufen der Aggregationspipeline zu leiten. Bei den meisten Stufentypen, die als Streaming-Stufen bezeichnet werden, nimmt das Datenbankmodul die verarbeitete Charge aus einer Stufe und leitet sie sofort in den nächsten Teil der Pipeline weiter. Sie tut dies, ohne zu warten, bis alle anderen Stapel in der vorherigen Stufe angekommen sind. Zwei Arten von Stufen müssen jedoch blockieren und darauf warten, dass alle Stapel ankommen und sich in dieser Stufe ansammeln. Diese beiden Stufen werden als blockierende Stufen bezeichnet:
- \$sort
- \$group 
  
*Wenn von \$group die Rede ist, schließt dies auch andere, weniger häufig verwendete "Gruppierung"-Stufen ein, nämlich: \$bucket, \$bucketAuto, \$count, \$sortByCount & \$facet (es ist etwas weit hergeholt, \$facet als Gruppenstufe zu bezeichnen, aber im Zusammenhang mit diesem Thema ist es am besten, es so zu sehen)

### \$sort Speicherverbrauch und Risikominderung

Bei naiver Anwendung muss eine \$sort-Stufe alle Eingabedatensätze auf einmal sehen, und daher muss der Hostserver über genügend Kapazität verfügen, um alle Eingabedaten im Speicher zu halten. Der benötigte Speicherplatz hängt stark von der anfänglichen Datengröße und dem Ausmaß ab, in dem die vorherigen Stufen die Größe reduzieren können. Außerdem können mehrere Instanzen der Aggregationspipeline gleichzeitig im Einsatz sein, zusätzlich zu anderen Datenbank-Workloads. Daher erzwingt MongoDB, dass jede Stufe auf 100 MB verbrauchten Arbeitsspeicher begrenzt ist. Die Datenbank gibt einen Fehler aus, wenn sie diese Grenze überschreitet. Um das Hindernis der Speicherbegrenzung zu umgehen, kann die Option allowDiskUse:true für die Gesamtaggregation zur Verarbeitung großer Ergebnisdatensätze gesetzt werden. Folglich wird der Sortiervorgang der Pipeline bei Bedarf auf die Festplatte ausgelagert und die Pipeline wird nicht mehr durch die 100 MB-Grenze eingeschränkt. Der Preis dafür ist jedoch eine deutlich höhere Latenzzeit und die Ausführungszeit wird sich wahrscheinlich um Größenordnungen erhöhen.

Um zu vermeiden, dass die Aggregation den gesamten Datensatz im Speicher manifestieren oder auf die Festplatte auslagern muss, versuchen Sie, Ihre Pipeline so umzugestalten, dass sie einen der folgenden Ansätze enthält (in der Reihenfolge des effektivsten zuerst):


1. Verwenden Sie Indexsortierung. Wenn die Stufe "\$sort" nicht von einer vorangehenden Stufe "\$unwind", "\$group" oder "\$project" abhängt, verschieben Sie die Stufe "\$sort" in die Nähe des Beginns Ihrer Pipeline, um einen Index für die Sortierung anzuvisieren. Die Aggregations-Runtime muss daher keine teure In-Memory-Sortieroperation durchführen. Die Stufe "\$sort" ist nicht unbedingt die erste Stufe in Ihrer Pipeline, da es auch eine Stufe "\$match" geben kann, die denselben Index nutzt. Überprüfen Sie immer den explain-Plan, um sicherzustellen, dass Sie das beabsichtigte Verhalten herbeiführen.

2. Verwenden Sie Limit mit Sort. Wenn Sie nur die erste Teilmenge der Datensätze aus dem sortierten Datensatz benötigen, fügen Sie direkt nach der Stufe \$sort eine Stufe \$limit ein, die die Ergebnisse auf die von Ihnen benötigte feste Menge (z. B. 10) begrenzt. Zur Laufzeit fasst die Aggregations-Engine \$sort und \$limit zu einem einzigen speziellen internen Sortierschritt zusammen, der beide Aktionen gemeinsam durchführt. Der laufende Sortierprozess muss nur die limitierte Menge (im Beispiel oben: 10) im Speicher nachverfolgen, die der gerade ausgeführten Sortier-/Limitregel entsprechen. Er muss nicht den gesamten Datensatz im Speicher halten, um die Sortierung erfolgreich durchzuführen.

3. Reduzieren Sie die zu sortierenden Datensätze. Verschieben Sie die \$sort-Stufe so spät wie möglich in Ihrer Pipeline und stellen Sie sicher, dass frühere Stufen die Anzahl der Datensätze, die in diese späte blockierende \$sort-Stufe fließen, deutlich reduzieren. Diese blockierende Stufe hat weniger Datensätze zu verarbeiten und benötigt weniger RAM.

### \$group-Speicherverbrauch und Risikominderung

In der Realität konzentrieren sich die meisten Gruppierungsszenarien auf das Sammeln von zusammenfassenden Daten wie Summen, Zählungen, Durchschnittswerten, Höchst- und Tiefstwerten und nicht auf Einzeldaten. In diesen Situationen werden erheblich reduzierte Ergebnisdatensätze erzeugt, die weit weniger Verarbeitungsspeicher benötigen als eine \$sort-Stufe. Im Gegensatz zu vielen Sortierungsszenarien benötigen Gruppierungsoperationen normalerweise nur einen Bruchteil des Arbeitsspeichers des Hosts.

Um sicherzustellen, dass Sie einen übermäßigen Speicherverbrauch vermeiden, wenn Sie eine \$group-Stufe verwenden möchten, sollten Sie die folgenden Grundsätze beachten:
1. Unnötige Gruppierungen vermeiden. 
2. Nur Zusammenfassungsdaten gruppieren. Wenn der Anwendungsfall es zulässt, verwenden Sie die Gruppenstufe nur zum Akkumulieren von Dingen wie Summen, Zählungen und zusammenfassenden Roll-ups, anstatt alle Rohdaten jedes zu einer Gruppe gehörenden Datensatzes zu speichern. 
   
Vermeiden Sie das Abwickeln und Umgruppieren von Dokumenten, nur um Array-Elemente zu verarbeiten


2. Es sollte vermieden werden, Dokumente zu entpacken und neu zu gruppieren, nur um Array-Elemente zu verarbeiten
Manchmal benötigen Sie eine Aggregationspipeline, um den Inhalt eines Array-Feldes für jeden Datensatz zu verändern oder zu reduzieren. Zum Beispiel:
- Es sollte vielleicht alle Werte im Array zu einem Gesamtfeld addieren werden.
- Es sollte nur das erste und das letzte Element des Arrays beibehalten werden
- Es sollte vielleicht nur ein wiederkehrendes Feld für jedes Unterdokument im Array aufbewahren werden
- ...oder zahlreiche andere Array-Reduktionsszenarien

Um dies zu verdeutlichen, stellen Sie sich eine Sammlung von Einzelhandelsbestellungen vor, bei der jedes Dokument eine Reihe von Produkten enthält, die im Rahmen der Bestellung gekauft wurden, wie im folgenden Beispiel dargestellt:

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
        // Unpack each product from the each order's product 
        // as a new separate record
        {
            "$unwind": {
                "path": "$products",
            }
        },

        // Match only products valued over 15.00
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
    // Filter out products valued 15.00 or less
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

Um es noch einmal zu wiederholen: Es sollte niemals notwendig sein, eine \$unwind/\$group-Kombination in einer Aggregationspipeline zu verwenden, um die Elemente eines Array-Feldes für jedes Dokument einzeln zu transformieren. Ein guter Indikator dafür, dass dieses Anti-Pattern zutrifft, ist, wenn Ihre Pipeline eine \$group für ein \$_id-Feld enthält. Verwenden Sie stattdessen Array-Operatoren, um die Einführung einer blockierenden Phase zu vermeiden. 
Die primäre Verwendung einer \$unwind/\$group-Kombination besteht darin, Muster über die Arrays vieler Datensätze hinweg zu korrelieren, anstatt nur den Inhalt innerhalb des Arrays eines jeden Eingabedatensatzes zu transformieren.


 Fördern Sie das Verwendung von Match-Filtern in der Pipeline

\Untersuchen, ob das Vorziehen einer vollständigen Übereinstimmung möglich ist
Prüfen, ob das Vorziehen eines teilweisen Matches möglich ist

Zusammenfassend lässt sich sagen: Wenn Sie eine Pipeline haben, die eine \$match-Stufe nutzt, und der Explain-Plan zeigt, dass diese nicht an den Anfang der Pipeline verschoben wird, sollten Sie prüfen, ob ein manuelles Refactoring helfen kann. Wenn der \$match-Filter von einem berechneten Wert abhängt, prüfen Sie, ob Sie diesen ändern oder einen zusätzlichen \$match hinzufügen können, um eine effizientere Pipeline zu erhalten.

## Erklärung und Verwendung von Ausdrücken
Zusammenfassende Aggregationsausdrücke
Aggregationsausdrücke gibt es in einer der drei Hauptvarianten:
1. Operatoren. Der Zugriff erfolgt als Objekt mit einem ``$-Präfix``, gefolgt vom Namen der Operatorfunktion. Der "dollar-operator-name" wird als Hauptschlüssel für das Objekt verwendet.  Beispiele: ``{$arrayElemAt: ...}, {$cond: ...}, {$dateToString: ...}``
2. Feldpfade. Zugriff als Zeichenkette mit einem \$-Präfix, gefolgt von dem Feldpfad in jedem verarbeiteten Datensatz.  Beispiele: ``"$Konto.sortcode", "$Adressen.Adresse.Stadt"``
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

### Was erzeugen Ausdrücke?
Ein Ausdruck kann ein Operator (z. B. ``{$concat: ...}``), eine Variable (z. B. ``"$$ROOT"``) oder ein Feldpfad (z. B. ``"$address"``) sein. In all diesen Fällen ist ein Ausdruck einfach etwas, das dynamisch ein neues JSON/BSON-Datentyp-Element auffüllt und zurückgibt, das eines der folgenden sein kann:
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


### Einschränkungen bei der Verwendung von Ausdrücken mit $match

Sie sollten sich darüber im Klaren sein, dass es Einschränkungen gibt, wann die Laufzeit von einem Index profitieren kann, wenn Sie einen \$expr-Operator innerhalb einer \$match-Stufe verwenden. Dies hängt teilweise von der Version von MongoDB ab, die Sie verwenden. Mit \$expr können Sie einen \$eq-Vergleichsoperator mit einigen Einschränkungen nutzen, darunter die Unmöglichkeit, einen Multi-Schlüssel-Index zu verwenden. Bei MongoDB-Versionen vor 5.0 kann bei Verwendung eines "Bereichs"-Vergleichsoperators (\$gt, \$gte, \$lt und \$lte) kein Index verwendet werden, um das Feld abzugleichen, aber in Version 5.0 funktioniert dies problemlos.

### Sharding-Betrachtung


MongoDB-Sharding:

-   Skalierung der Datenbank, um mehr Daten zu speichern und die Transaktionsdurchsatzrate zu erhöhen
-   Skalierung der Analysearbeiten, um Aggregationen schneller abzuschließen (möglicherweise werden Aggregationen parallel auf mehreren Shards ausgeführt)


## Kurzübersicht über geshardete Clusters

In einem geshardeten Cluster läuft jeder Shard auf einem separaten Satz von Host-Computern und speichert einen Teil einer Datenmenge. Basierend auf einem Shard-Schlüssel jedes Dokuments (von Benutzer definiert), gruppiert das System Untermengen von Dokumenten zu "Chunks" zusammen. Der Cluster verteilt diese Chunks auf seine Shards. In der gleichen Datenbank können sowohl geshardete als auch ungeshardete Sammlungen gehalten werden. Alle ungeshardeten Sammlungen werden auf einem bestimmten Shard im Cluster gespeichert, das als "Primary Shard" bezeichnet wird.

## Einschränkungen bei der geshardeten Aggregation

Einige der MongoDB-Stadien unterstützen nur teilweise geshardete Aggregationen, abhängig von der DB-Version. Diese Stadien beziehen sich alle auf eine zweite Sammlung zusätzlich zur Pipelinesourcen-Eingabesammlung. In jedem Fall kann die Pipeline eine geshardete Sammlung als ihre Quelle verwenden, aber die zweite referenzierte Sammlung muss ungeshardet sein. Die betroffenen Stadien und Versionen sind:

-   $lookup: In Versionen vor 5.1 muss die zweite referenzierte Sammlung, mit der zusammengeführt werden soll, ungeshardet sein
-   $graphLookup: In Versionen vor 5.1 muss die zweite referenzierte Sammlung, mit der zusammengeführt werden soll, ungeshardet sein
-   $out: In allen Versionen muss die zweite referenzierte Sammlung, die als Ziel der Ausgabe der Aggregation verwendet wird, ungeshardet sein. Du kannst stattdessen einen $merge-Abschnitt verwenden, um das Aggregationsergebnis in eine geshardete Sammlung zu schreiben.


## Wo wird eine geshardete Aggregation ausgeführt?

Geshardete Clusters bieten die Möglichkeit, die Antwortzeiten von Aggregationen zu verringern. Dies wird jedoch nicht immer der Fall sein, da bestimmte Arten von Pipelines erfordern, dass erhebliche Mengen von Daten von mehreren Shards zusammengeführt werden, um weiterverarbeitet zu werden. Die Antwortzeit der Aggregation könnte sich in solchen Fällen in die entgegengesetzte Richtung entwickeln und aufgrund des erheblichen Netzwerkübertragungs- und Marshalling-Overheads deutlich länger dauern.

#### Spaltung der Pipeline zur Laufzeit

Ein geshardeter Cluster wird versuchen, die Stadien einer Pipeline parallel auf jedem Shard auszuführen, der die erforderlichen Daten enthält. Sortier- und Gruppierungsstadien ($sort, $bucket, $bucketAuto, $count und $sortByCount) müssen jedoch an allen Daten an einem Ort ("blockierende Stadien", beschrieben in _Performanceüberlegungen für Pipelines_) ausgeführt werden. Sobald ein blockierendes Stadium in der Pipeline auftritt, wird das Aggregationssystem die Pipeline an dem Punkt, an dem das blockierende Stadium auftritt, in zwei Teile aufteilen. Der erste Teil der gespaltenen Pipeline (benannt als "Shards Part" im Explain-Plan) kann parallel auf mehreren Stadien ausgeführt werden, während der restliche Teil der gespaltenen Pipeline (benannt als "Merge part" im Explain-Plan) an einem Ort ausgeführt wird.

#### Ausführung des Shards-Teils der gespaltenen Pipeline

Wenn ein Mongos eine Anfrage zur Ausführung einer Aggregationspipeline erhält, muss es festlegen, wo der Shards-Teil der Pipeline zielgerichtet werden soll. Es wird versuchen, dies auf einer relevanten Teilmenge von Shards auszuführen, anstatt die Arbeit an alle zu senden. Wenn beispielsweise der Filter für $match den Shard-Schlüssel oder einen Präfix des Shard-Schlüssels enthält, kann das Mongos eine zielgerichtete Operation ausführen und den Shards-Teil der gespaltenen Pipeline auf die entsprechenden Shards leiten. Wenn der $match-Filter beispielsweise eine exakte Übereinstimmung mit einem Shard-Schlüsselwert für die Souce-Sammlung enthält, kann die Pipeline auf einen einzigen Shard zielen und muss nicht in zwei Teile aufgeteilt werden.

#### Ausführung des Merger-Teils der gespaltenen Pipeline (falls vorhanden)

Die Aggregations-Runtime wendet eine Reihe von Regeln an, um festzulegen, wo der Merger-Teil einer Aggregationspipeline für einen geshardeten Cluster ausgeführt werden soll und ob überhaupt eine Spaltung erforderlich ist. Die Erreichung von entweder der Targeted-Shard-Ausführung (2) oder Mongos-Merge (4) ist in der Regel das bevorzugte Ergebnis für optimale Leistung:

1.  **Primary-Shard Merge**: Die Pipeline enthält einen Stadium, das auf eine zweite ungeshardete Sammlung verweist:
	-   $out oder $lookup und $graphLookup in der MongoDB-Version vor 5.1
	-   Referenzierung einer ungeshardeten Sammlung von einem $merge-Stadium oder von $lookup oder $graphLookup (MongoDB 5.1 oder höher)
1.  **Targeted-Shard-Ausführung**: Die Runtime kann sicherstellen, dass die Pipeline mit dem erforderlichen Teilmenge der Sourcen-Sammlungsdaten auf nur einem Shard übereinstimmt. Das Verhalten, das auf einen einzigen Shard festgelegt wird, tritt auch dann auf, wenn die Pipeline ein $merge, $lookup oder $graphLookup-Stadium enthält, das auf eine zweite geshardete Sammlung verweist, die Aufzeichnungen über mehrere Shards verteilt enthält.
2.  **Any-Shard Merge**: Wenn allowDiskUsage:true konfiguriert ist und eines der folgenden auch zutrifft, muss die Aggregations-Runtime den Merger-Teil der gespaltenen Pipeline auf einem zufällig gewählten Shard ausführen:
    -   Die Pipeline enthält ein Gruppierungsstadium
    -   Die Pipeline enthält ein $sort-Stadium und ein darauf folgendes blockierendes Stadium (ein Gruppierungs- oder $sort-Stadium) tritt später auf
3. **Mongos Merge**: (Standardverfahren) Wenn der Merger-Teil der Pipeline nur streaming-Stadien enthält, geht die Runtime davon aus, dass es sicher ist, den verbleibenden Teil der Pipeline dem Mongos zu übergeben.


### Zusammenfassung der Ansätze zur Ausführung geshardeter Pipelines

Zusammenfassend sucht die Aggregations-Runtime danach, eine Pipeline auf der Teilmenge von Shards auszuführen, die die erforderlichen Daten enthalten. Wenn die Runtime die Pipeline splitten muss, um Gruppierung oder Sortierung durchzuführen, führt sie die endgültige Merge-Arbeit auf einem Mongos durch, wenn möglich. Das Zusammenführen auf einem Mongos hilft, die Anzahl der erforderlichen Netzwerk-Hops und die Ausführungszeit zu reduzieren.

### Leistungstipps für geshardete Aggregationen

Alle empfohlenen Aggregationsoptimierungen, die in den Leistungsüberlegungen zu Pipeline-Leistungen dargelegt werden, gelten ebenfalls für geshardete Cluster, sind aber bei der Ausführung von Aggregationen in gesharten Clustern sogar noch kritischer:

1.  Sortierung - Es sollte Indexsortierung verwendet werden: Der Shard-Teil der gespaltenen Pipeline, der auf jedem Shard parallel ausgeführt wird, wird dadurch vermeiden, dass der In-Memory-Sortierungsvorgang durchgeführt wird.
2.  Sortierung - Es sollte Limit mit Sortierung verwendet werden:  Die Runtime muss weniger Zwischenaufzeichnungen über das Netzwerk übertragen, von jedem Shard, der den Shard-Teil einer gespaltenen Pipeline ausführt, an den Ort, an dem der Merger-Teil der Pipeline ausgeführt wird.
5.  Sortierung - Die zu sortierenden Aufzeichnungen sollten verringert werden: Wenn die ersten beiden Punkte unmöglich sind, sollte ein $sort-Stadium so spät wie möglich verschoben werden.
6. Gruppierung - Unnötige Gruppierung sollte vermeiden werden: Möglichst Array-Operatoren anstelle von $unwind- und $group-Stadien sollten verwendet werden, um das Splitting der Pipeline in ein unnötig eingeführtes $group-Stadium zu vermeiden.
7.  Gruppierung - Nur Zusammenfassungsdaten sollten gruppiert werden: Die Runtime muss weniger berechnete Aufzeichnungen über das Netzwerk von jedem Shard, auf dem der Shard-Teil einer gespaltenen Pipeline ausgeführt wird, an den Standort des Merger-Teils übertragen werden.
8.  Es sollte ermutigt werden, dass Filter mit Match früh im Pipeline-Prozess erscheinen, damit die Runtime weniger Aufzeichnungen an den Standort des Merger-Teils streamen muss.

Es sollten insbesondere für geshardete Cluster zwei weitere Leistungsoptimierungen angestrebt werden:

1.  Möglichkeiten, Aggregationen auf einem Shard zu zielen, sollten gesucht werden: Wenn möglich, sollte ein $match-Stadium mit einem Filter auf einen Shard-Schlüsselwert (oder Shard-Schlüsselpräfixwert) hinzugefügt werden.
2.  Möglichkeiten, dass eine gespaltene Pipeline auf einem Mongos zusammengeführt wird, sollten gesucht werden: Wenn eine Pipeline ein $group-Stadium (oder ein $sort-Stadium, gefolgt von einem $group- oder $sort-Stadium) hat, das dazu führt, dass sich die Pipeline teilt, sollte allowDiskUse:true vermieden werden, falls möglich. Dies reduziert die Menge an Zwischendaten, die über das Netzwerk übertragen werden und somit die Latenz.
   
### Erweiterte Verwendung von Ausdrücken zur Verarbeitung von Arrays

Das Aggregationsframework bietet eine reiche Auswahl an Aggregationsoperatoren-Ausdrücken für die Analyse und Manipulation von Arrays. Beim Optimieren für die Leistung sind diese Array-Ausdrücke von entscheidender Bedeutung, um das Aufrollen und Neugruppieren von Dokumenten zu vermeiden, wenn Sie nur jedes Array eines Dokuments isoliert verarbeiten müssen. Für die meisten Situationen, in denen Sie ein Array manipulieren müssen, gibt es normalerweise einen einzigen Array-Operator-Ausdruck, auf den Sie zurückgreifen können, um Ihre Anforderung zu erfüllen. Gelegentlich müssen Sie jedoch immer noch eine Kombination aus mehreren niedrigeren Ausdrücken zusammenstellen, um eine herausfordernde Array-Manipulation durchzuführen. Wie bei Aggregationspipelines im Allgemeinen ist ein großer Teil der Herausforderung darin zu adaptieren, Ihren Geist auf eine funktionale Programmierparadigma anstatt eines prozeduralen. Die Vergleich mit prozeduralen Ansätzen kann helfen, Klarheit bei der Beschreibung der Array-Manipulationspipeline-Logik zu bringen.

##### "If-Else" bedingter Vergleich

Betrachten Sie das trivialisierte Szenario eines Einzelhändlers, der die Gesamtkosten einer Kundenbestellung berechnen möchte. Der Kunde kann mehrere identische Produkte bestellen und der Anbieter gewährt einen Rabatt, wenn mehr als 5 Artikel des Produkts in der Bestellung sind.

In einem prozeduralen Stil von JavaScript könnten Sie den folgenden Code schreiben, um die Gesamtkosten der Bestellung zu berechnen:

```
let order = {"product" : "WizzyWidget", "price": 25.99, "qty": 8};
// Procedural style JavaScript
if (order.qty > 5) {
  order.cost = order.price * order.qty * 0.9;
} else {
  order.cost = order.price * order.qty;
}
```


Dieser Code modifiziert die Kundenbestellung wie folgt, um die Gesamtkosten einzuschließen:

```{product: 'WizzyWidget', qty: 8, price: 25.99, cost: 187.128}```

Um ein ähnliches Ergebnis in einer Aggregationspipeline zu erzielen, könnten Sie Folgendes verwenden:

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

Diese Pipeline liefert folgende Ausgabe mit dem transformierten Kundenbestelldokument:

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

Der JavaScript-Beispielcode arbeitet nur mit einem Dokument auf einmal und müsste geändert werden, um eine Liste von Datensätzen in einer Schleife zu durchlaufen. Dieser JavaScript-Code müsste jedes Dokument aus der Datenbank zurück zu einem Client holen, die Änderungen anwenden und dann das Ergebnis zurück in die Datenbank schreiben. Stattdessen arbeitet die Logik der Aggregationspipeline mit jedem Dokument direkt in der Datenbank, was eine weitaus bessere Leistung und Effizienz ermöglicht.

##### Die "Macht" der Array-Operatoren

When data from an array field should to be transformed or extracted, and a single high-level array operator (e.g. $avg, $max, $filter) does not give the needed, the tools to turn to are the $map and $reduce array operators. These two "power" operators enable  an array iteration, perform whatever complexity of logic against each array element and collect together the result for inclusion in a stage's output.
Depending on specific requirements, one or the other could be used to process an array's field, but not both together. Here's an explanation of these two "power" operators:

- \$map. Ermöglicht die Angabe einer Logik, die für jedes Element im Array ausgeführt werden soll, das der Operator iteriert, und als Endergebnis ein Array zurückgibt. Typischerweise wird $map verwendet, um jedes Array-Element zu verändern und dann das veränderte Array zurückzugeben. Der $map-Operator gibt den Inhalt des aktuellen Array-Elements über eine spezielle Variable mit dem Standardnamen \$\$this an die erforderliche Logik weiter.

- \$reduce. In ähnlicher Weise könnte eine Logik angegeben werden, die für jedes Element in einem Array ausgeführt wird, das der Operator durchläuft, aber stattdessen einen einzelnen Wert (und nicht ein Array) als Endergebnis zurückgibt. Der Operator \$reduce wird typischerweise verwendet, um eine Zusammenfassung zu berechnen, nachdem jedes Array-Element analysiert wurde. Wie der \$map-Operator bietet der \$reduce-Operator eine Logik mit Zugriff auf das aktuelle Array-Element über die Variable \$$value, damit die erforderliche Logik beim Akkumulieren des Einzelergebnisses (z. B. des Multiplikationsergebnisses) aktualisiert werden kann.

 ##### "For-Each"-Schleife zur Umwandlung eines Arrays
 
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

Dieser Code ändert die Produktnamen der Bestellung wie folgt, wobei die Produktnamen jetzt in Großbuchstaben geschrieben sind:

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

Unter Verwendung des funktionalen Stils in JavaScript würde der Schleifencode eher dem folgenden ähneln, um das gleiche Ergebnis zu erzielen:

```
// Functional style JavaScript
order.products = order.products.map(
  product => {
    return product.toUpperCase(); 
  }
);
```

Der Vergleich eines $map-Operatorausdrucks für die Aggregation mit einer JavaScript-Array-Funktion map() ist viel aufschlussreicher, um zu erklären, wie der Operator funktioniert.

##### "For-Each"-Schleife zum Berechnen eines Summenwerts aus einem Array

Angenommen, es soll eine Liste der von einem Kunden bestellten Produkte verarbeiten, aber ein einziges zusammenfassendes Zeichenfolgenfeld aus diesem Array erzeugen, indem alle Produktnamen aus dem Array verkettet werden. In einem prozeduralen JavaScript-Stil könnte Folgendes geschrieben werden, um das Zusammenfassungsfeld für die Produktnamen zu erstellen:

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

Dieser Code ergibt die folgende Ausgabe mit einem neuen erzeugten Zeichenfolgenfeld "productList", das die Namen aller Produkte in der Bestellung enthält, getrennt durch Semikolons:
```
{
  orderId: 'AB12345',
  products: [ 'Laptop', 'Kettle', 'Phone', 'Microwave' ],
  productList: 'Laptop; Kettle; Phone; Microwave; '
}
```

Es könnte die folgende Pipeline verwendet werden, um ein ähnliches Ergebnis zu erzielen:
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

Hier durchläuft ein \$reduce-Operatorausdruck jedes Produkt im Eingabearray und verkettet den Namen jedes Produkts zu einer akkumulierenden Zeichenfolge. Der `$$this`-Ausdruck, der verwendet wird, um während jeder Iteration auf den Wert des aktuellen Array-Elements zuzugreifen. Für jede Iteration wird der Ausdruck `$$value` verwendet, um auf den endgültigen Ausgabewert zu verweisen, an den die aktuelle Produktzeichenfolge (+ Trennzeichen) angehängt wird.

Diese Pipeline erzeugt dieselbe Ausgabe wie die obere, die mit Javascript erstellt wurde, wo sie das Bestelldokument umwandelt. Mit einem funktionalen Ansatz in JavaScript hätte es den folgenden Code verwenden können, um dasselbe Ergebnis zu erzielen:
```
// Functional style JavaScript
order.productList = order.products.reduce(
  (previousValue, currentValue) => {
    return previousValue + currentValue + "; ";
  },
  ""
);
```


##### "For-Each"-Schleife zum Auffinden eines Array-Elements

Stellen Sie sich vor, Daten über Gebäude auf einem Campus zu speichern, wo jedes Gebäudedokument eine Reihe von Räumen mit ihren Größen (Breite und Länge) enthält. Ein Raumreservierungssystem kann erfordern, dass der erste Raum im Gebäude mit ausreichender Grundfläche für eine bestimmte Anzahl von Besprechungsteilnehmern gefunden wird. Unten sehen Sie ein Beispiel für die Daten eines Gebäudes, das in die Datenbank geladen werden könnte, mit seiner Anordnung von Räumen und ihren Abmessungen in Metern:
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

Es sollte eine Pipeline erstellt werden, um einen geeigneten Besprechungsraum zu finden, der eine Ausgabe ähnlich der folgenden erzeugt. Das Ergebnis sollte ein neu hinzugefügtes Feld "firstLargeEnoughRoomArrayIndex" enthalten, um die Array-Position des ersten Raums anzugeben, bei dem genügend Kapazität gefunden wurde.

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

Unten ist eine geeignete Pipeline, die durch die Raum-Array-Elemente iteriert und die Position des ersten Elements mit einer berechneten Fläche von mehr als 60 m² erfasst:

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

Hier wird der \$reduce-Operator erneut verwendet, um eine Schleife zu durchlaufen und schließlich einen einzelnen Wert zurückzugeben. Die Pipeline verwendet jedoch anstelle des vorhandenen Array-Felds in jedem Quelldokument eine generierte Folge aufsteigender Zahlen für ihre Eingabe. Der `$range`-Operator wird verwendet, um diese Sequenz zu erstellen, die die gleiche Größe wie das Zimmer-Array-Feld jedes Dokuments hat. Die Pipeline verwendet diesen Ansatz, um die Array-Position des übereinstimmenden Raums mithilfe der Variablen `$$this` zu verfolgen. Für jede Iteration berechnet die Pipeline die Fläche des Array-Room-Elements. Wenn die Größe größer als 60 ist, weist die Pipeline die aktuelle Array-Position (dargestellt durch `$$this`) dem Endergebnis (dargestellt durch `$$value`) zu.

Die „Iterator“-Array-Ausdrücke haben kein Konzept eines _break_ Befehls, den prozedurale Programmiersprachen normalerweise bereitstellen. Obwohl die ausführende Logik möglicherweise bereits einen Raum ausreichender Größe lokalisiert hat, wird der Schleifenprozess daher durch die verbleibenden Array-Elemente fortgesetzt. Folglich muss die Pipeline-Logik bei jeder Iteration eine Prüfung enthalten, um zu vermeiden, dass der endgültige Wert (die `$$value`-Variable) überschrieben wird, wenn er bereits einen Wert hat. Bei massiven Arrays, die einige Hundert oder mehr Elemente enthalten, wird eine Aggregationspipeline natürlich eine merkliche Latenzauswirkung haben, wenn die verbleibenden Array-Mitglieder iteriert werden, obwohl die Logik das erforderliche Element bereits identifiziert hat.

Angenommen, es soll nur das erste passende Array-Element für einen Raum mit ausreichender Grundfläche zurückgegeben werden, nicht sein Index. In diesem Fall kann die Pipeline einfacher sein, indem Sie `$filter` verwenden, um die Array-Elemente auf diejenigen mit ausreichend Platz zuzuschneiden, und dann den Operator `$first`, um nur das erste Element aus dem Filter zu holen. Es könnte eine Pipeline ähnlich der folgenden verwendet werden:

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

##### Reproduzieren des $map-Verhaltens mit $reduce

Es ist möglich, das `$map`-Verhalten mithilfe von `$reduce` zu implementieren, um ein Array zu transformieren. Diese Methode ist komplexer, aber Sie müssen sie möglicherweise in einigen seltenen Fällen verwenden. Bevor wir uns ein Beispiel dafür ansehen, vergleichen wir zuerst ein einfacheres Beispiel für die Verwendung von `$map` und dann `$reduce`, um dasselbe zu erreichen.

Angenommen, einige Sensormesswerte für ein Gerät sollten erfasst werden:

```
db.deviceReadings.insertOne({
  "device": "A1",
  "readings": [27, 282, 38, -1, 187]
});
```

Stellen Sie sich vor, es sollte eine transformierte Version des Arrays „Messwerte“ erstellt werden, wobei die Geräte-ID mit jedem Messwert im Array verkettet wird. Es könnte die Pipeline erstellt werden, um eine Ausgabe ähnlich der folgenden mit dem neu eingefügten Array-Feld zu erzeugen:

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

Da es sich hierbei um das Aggregations-Framework handelt, gibt es natürlich mehrere Möglichkeiten, dasselbe Problem zu lösen.
Zusammenfassend lässt sich sagen, dass eine `$map`-Phase typischerweise verwendet wird, wenn das Verhältnis von Eingabeelementen zu Ausgabeelementen gleich ist (d. h. viele zu viele oder _M:M_). Eine `$reduzieren`-Stufe, die verwendet wird, wenn das Verhältnis von Eingabeelementen zu Ausgabeelementen viele zu eins ist (d. h. _M:1_). In Situationen, in denen das Verhältnis der Eingabeelemente viele zu wenige ist (d. h. _M:N_), werden Sie anstelle von `$map` ausnahmslos nach `$reduce` mit seinem "Null-Array-Verkettungs"-Trick greifen, wenn `$filter` reicht nicht aus.


##### Hinzufügen neuer Felder zu bestehenden Objekten in einem Array

Eine der Hauptverwendungen des Operatorausdrucks "$map" besteht darin, jedem vorhandenen Objekt in einem Array weitere Daten hinzuzufügen. Angenommen, eine Reihe von Einzelhandelsbestellungen wird beibehalten, wobei jedes Bestelldokument eine Reihe von Bestellpositionen enthält. Jede Bestellposition im Array erfasst den Produktnamen, den Stückpreis und die gekaufte Menge, wie im folgenden Beispiel gezeigt:

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

Jetzt müssen die Gesamtkosten für jeden Produktartikel (Menge x Einheitspreis) berechnet und diese Kosten dem entsprechenden Bestellartikel im Array hinzugefügt werden. Es könnte eine Pipeline ähnlich der folgenden verwendet werden, um dies zu erreichen:
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

Hier erstellt die Pipeline für jedes Element im Quellarray ein Element im neuen Array, indem sie explizit die drei Felder aus dem alten Element (`product`, `unitPrice` und `quantity`) zieht und ein neues berechnetes Feld hinzufügt ( „Kosten“). Die Pipeline erzeugt die folgende Ausgabe:

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

Ähnlich wie bei den Nachteilen der Verwendung einer `$project`-Stufe in einer Pipeline wird der Code `$map` durch die explizite Benennung jedes zu behaltenden Felds im Array-Element belastet. Es könnte mühsam sein, wenn jedes Array-Element viele Felder hat. Wenn sich das Datenmodell weiterentwickelt und im Laufe der Zeit neue Feldtypen in den Elementen des Arrays erscheinen, müssen Sie außerdem zu Ihrer Pipeline zurückkehren und sie jedes Mal umgestalten, um diese neu eingeführten Felder einzubeziehen. Genau wie bei der Verwendung von `$set` anstelle von `$project` für eine Pipeline-Phase gibt es eine bessere Lösung, um alle vorhandenen Array-Elementfelder beizubehalten und neue hinzuzufügen, wenn Arrays verarbeitet werden. Eine gute Lösung besteht darin, den Operatorausdruck [`$mergeObjects`](https://docs.mongodb.com/manual/reference/operator/aggregation/merge/) zu verwenden, um alle vorhandenen Felder plus die neu berechneten Felder in jedem neuen zu kombinieren Array-Element. `$mergeObjects` nimmt ein Array von Objekten und kombiniert die Felder aller Objekte des Arrays zu einem einzigen Objekt.
Um in dieser Situation `$mergeObjects` zu verwenden, sollte das aktuelle Array-Element als erster Parameter für `$mergeObjects` bereitgestellt werden. Der zweite bereitgestellte Parameter ist ein neues Objekt, das jedes berechnete Feld enthält. Im folgenden Beispiel fügt der Code nur ein generiertes Feld hinzu, aber wenn es erforderlich ist, könnte er mehrere generierte Felder in dieses neue Objekt aufnehmen:
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

Diese Pipeline erzeugt die gleiche Ausgabe wie die vorherige Pipeline für „fest codierte Feldnamen“, jedoch mit dem Vorteil, dass sie für neue Arten von Feldern, die in Zukunft im Quellarray erscheinen, sympathisch ist.
Anstatt "$mergeObjects" zu verwenden, gibt es eine alternative und etwas ausführlichere Kombination aus drei verschiedenen Array-Operatorausdrücken, die auf ähnliche Weise verwendet werden könnten, um alle vorhandenen Array-Elementfelder beizubehalten und neue hinzuzufügen. Diese drei Operatoren sind:

- [`$objectToArray`](https://docs.mongodb.com/manual/reference/operator/aggregation/objectToArray/). Dadurch wird ein Objekt, das verschiedene Feld-Schlüssel/Wert-Paare enthält, in ein Array von Objekten konvertiert, wobei jedes Objekt zwei Felder hat: "k", das den Feldnamen enthält, und "v", das den Feldwert enthält. Zum Beispiel: `{height: 170, weight: 60}` wird zu `[{k: 'height', v: 170}, {k: 'weight', v: 60}]`
- [`$concatArrays`](https://docs.mongodb.com/manual/reference/operator/aggregation/concatArrays/). Dies kombiniert den Inhalt mehrerer Arrays zu einem einzigen Array-Ergebnis.
- [`$arrayToObject`](https://docs.mongodb.com/manual/reference/operator/aggregation/arrayToObject/). Dadurch wird ein Array in ein Objekt umgewandelt, indem die Umkehrung des Operators "$objectToArray" ausgeführt wird. Zum Beispiel: `{k: 'height', v: 170}, {k: 'weight', v: 60}, {k: 'shoeSize', v: 10}]` wird zu `{height: 170, weight: 60, Schuhgröße: 10}`

Die folgende Pipeline zeigt die Kombination in Aktion für denselben Einzelhandelsbestelldatensatz wie zuvor, wobei die neu berechneten Gesamtkosten für jedes Produkt hinzugefügt werden:
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

Wenn dies dasselbe bewirkt wie die Verwendung von `$mergeObjects`, aber ausführlicher ist, warum sollte man sich dann die Mühe machen, dieses Muster zu verwenden? Nun, in den meisten Fällen würde es nicht. Eine Situation, in der die ausführlichere Kombination verwendet werden würde, ist, wenn der Name des Felds eines Array-Elements zusätzlich zu seinem Wert dynamisch festgelegt werden muss. Anstatt das berechnete Gesamtfeld als "Kosten" zu benennen, nehmen wir an, dass der Name des Felds auch den Namen des Produkts widerspiegeln sollte (z. B. "KostenFürWizzyWidget", "KostenFürHighEndGizmo"). Dies könnte erreicht werden, indem der Ansatz `$arrayToObject`/`$concatArrays`/`$objectToArray`anstelle der Methode `$mergeObjects` wie folgt verwendet wird:
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

Die Pipeline hat alle Felder der vorhandenen Array-Elemente beibehalten und jedem Element ein neues Feld mit einem dynamisch generierten Namen hinzugefügt.

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

##### Rudimentäre Schemareflexion mit Arrays

Als letztes "lustiges" Beispiel sehen wir uns an, wie Sie einen `$objectToArray`-Operatorausdruck verwenden, um [Reflektion](https://en.wikipedia.org/wiki/Reflective_programming) zu verwenden, um die Form einer Sammlung von Dokumenten als zu analysieren Teil eines benutzerdefinierten Schemaanalyse-Tools. Solche Reflexionsfunktionen sind in Datenbanken, die ein flexibles Datenmodell bereitstellen, wie MongoDB, wo die enthaltenen Felder von Dokument zu Dokument variieren können, von entscheidender Bedeutung.

Stellen Sie sich vor, es gibt eine Sammlung von Kundendokumenten, ähnlich der folgenden:
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

In der Schemaanalyse-Pipeline wird das "$objectToArray" verwendet, um den Namen und Typ jedes Felds der obersten Ebene im Dokument wie folgt zu erfassen:
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

Für die beiden Beispieldokumente in der Sammlung gibt die Pipeline Folgendes aus:
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

Die Schwierigkeit bei diesem grundlegenden Pipeline-Ansatz besteht darin, dass die Ausgabe zu lang und komplex ist, sobald viele Dokumente in der Sammlung vorhanden sind, um allgemeine Schemamuster zu erkennen. Stattdessen sollte eine Phasenkombination aus „$unwind“ und „$group“ hinzugefügt werden, um übereinstimmende wiederkehrende Felder zu sammeln. Das generierte Ergebnis sollte auch hervorheben, wenn derselbe Feldname in mehreren Dokumenten, aber mit unterschiedlichen Datentypen vorkommt. Hier ist die verbesserte Pipeline:
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

Die Ausgabe dieser Pipeline bietet nun eine viel verständlichere Zusammenfassung, wie unten gezeigt:
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

Dieses Ergebnis zeigt, dass das Feld "telNums" in Dokumenten einen von zwei verschiedenen Datentypen haben kann.


