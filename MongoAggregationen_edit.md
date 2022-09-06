Praktische MongoDb Aggregationen

- [Definition](#definition)
  - [Kompositionsfähigkeit für mehr Produktivität](#kompositionsfähigkeit-für-mehr-produktivität)
  - [Bessere Alternativen zu einer $project -Stage](#bessere-alternativen-zu-einer-project--stage)
  - [Wann sollte man \$set & \$unset verwenden?](#wann-sollte-man-set--unset-verwenden)
    - [Wann \$project zu verwenden ist](#wann-project-zu-verwenden-ist)
  - [Verwendung von Explain-Plänen](#verwendung-von-explain-plänen)
    - [Den Explain-Plan verstehen](#den-explain-plan-verstehen)
- [Überlegungen zur Pipeline-Leistung](#überlegungen-zur-pipeline-leistung)
  - [1. Beachten Sie die Reihenfolge von Streaming und Blocking Stage](#1-beachten-sie-die-reihenfolge-von-streaming-und-blocking-stage)
  - [\$sort Speicherverbrauch und Schadensbegrenzung](#sort-speicherverbrauch-und-schadensbegrenzung)
  - [\$group-Speicherverbrauch und Schadensbegrenzung](#group-speicherverbrauch-und-schadensbegrenzung)
- [Erklärung und Verwendung von Ausdrücken](#erklärung-und-verwendung-von-ausdrücken)
  - [Was erzeugen Ausdrücke?](#was-erzeugen-ausdrücke)
    - [Können alle Stufen Ausdrücke verwenden?](#können-alle-stufen-ausdrücke-verwenden)
  - [Einschränkungen bei der Verwendung von Ausdrücken mit $match](#einschränkungen-bei-der-verwendung-von-ausdrücken-mit-match)
  - [Sharding-Betrachtung TBD](#sharding-betrachtung-tbd)
  - [Advanced Use Of Expressions For Array Processing](#advanced-use-of-expressions-for-array-processing)

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

\$set & \$unset sollten verwenden werden, wenn die meisten Felder in den Eingabe-Datensätzen beibehalten müssen und eine kleine Teilmenge von Feldern hinzufügen, geändert oder entfernen werden soll.


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
Eine \$project-Stufe sollte verwendet werden, wenn sich die gewünschte Form der Ausgabedokumente stark von der Form der Eingabedokumente unterscheidet. Dies ist häufig der Fall, wenn Sie die meisten der ursprünglichen Felder nicht einbeziehen müssen.
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

Zusammenfassend lässt sich sagen, dass \$set (oder \$addFields) und \$unset für die Einbeziehung und den Ausschluss von Feldern immer verwenden werden, statt \$project. Die wichtigste Ausnahme ist, wenn Sie eine offensichtliche Anforderung für eine sehr unterschiedliche Struktur für Ergebnisdokumente haben, bei der Sie nur eine kleine Teilmenge der Eingabefelder beibehalten müssen.

### Verwendung von Explain-Plänen

Um den Explain-Plan für eine Aggregationspipeline anzuzeigen, den folgenden Befehle können ausführen werden:

<code>
    
    db.coll.explain().aggregate([{"$match": {"name": "Jo"}}]);
</code>

Wie bei MQL gibt es drei verschiedene Ausführlichkeitsmodi, mit denen Sie einen Explain-Plan erstellen können:

<code>

    // QueryPlanner verbosity  (default if no verbosity parameter provided)
    db.coll.explain("queryPlanner").aggregate(pipeline);

    // ExecutionStats verbosity
    db.coll.explain("executionStats").aggregate(pipeline);

    // AllPlansExecution verbosity
    db.coll.explain("allPlansExecution").aggregate(pipeline);
</code>

In den meisten Fällen werden Sie feststellen, dass die Variante executionStats der informativste Modus ist. Sie zeigt nicht nur den Denkprozess des Abfrageplaners, sondern liefert auch tatsächliche Statistiken über den "erfolgreichen" Ausführungsplan (z. B. die insgesamt untersuchten Schlüssel, die insgesamt untersuchten Dokumente usw.). Dies ist jedoch nicht die Standardeinstellung, da zusätzlich zur Formulierung des Abfrageplans auch die Aggregation ausgeführt wird (However, this isn't the default because it actually executes the aggregation in addition to formulating the query plan.). Wenn die Quellenkollektion groß oder die Pipeline suboptimal ist, wird es eine Weile dauern, bis das Ergebnis des Explain-Plans zurückgegeben wird.
Beachten Sie, dass die aggregate()-Funktion auch eine rudimentäre explain-Option bietet, mit der ein explain-Plan erstellt und zurückgegeben werden kann. Diese ist jedoch eingeschränkter und umständlicher in der Anwendung, so dass Sie sie vermeiden sollten.

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

Sie fordern dann den Abfrageplanerteil des Explain-Plans an:
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
Sie fragen nach den Ausführungsstatistiken als Teil des Explain-Plans:

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

### \$sort Speicherverbrauch und Schadensbegrenzung

Bei naiver Anwendung muss eine \$sort-Stufe alle Eingabedatensätze auf einmal sehen, und daher muss der Hostserver über genügend Kapazität verfügen, um alle Eingabedaten im Speicher zu halten. Der benötigte Speicherplatz hängt stark von der anfänglichen Datengröße und dem Ausmaß ab, in dem die vorherigen Stufen die Größe reduzieren können. Außerdem können mehrere Instanzen der Aggregationspipeline gleichzeitig im Einsatz sein, zusätzlich zu anderen Datenbank-Workloads. Daher erzwingt MongoDB, dass jede Stufe auf 100 MB verbrauchten Arbeitsspeicher begrenzt ist. Die Datenbank gibt einen Fehler aus, wenn sie diese Grenze überschreitet. Um das Hindernis der Speicherbegrenzung zu umgehen, kann die Option allowDiskUse:true für die Gesamtaggregation zur Verarbeitung großer Ergebnisdatensätze gesetzt werden. Folglich wird der Sortiervorgang der Pipeline bei Bedarf auf die Festplatte ausgelagert und die Pipeline wird nicht mehr durch die 100 MB-Grenze eingeschränkt. Der Preis dafür ist jedoch eine deutlich höhere Latenzzeit und die Ausführungszeit wird sich wahrscheinlich um Größenordnungen erhöhen.

Um zu vermeiden, dass die Aggregation den gesamten Datensatz im Speicher manifestieren oder auf die Festplatte auslagern muss, versuchen Sie, Ihre Pipeline so umzugestalten, dass sie einen der folgenden Ansätze enthält (in der Reihenfolge des effektivsten zuerst):


1. Verwenden Sie Indexsortierung. Wenn die Stufe "\$sort" nicht von einer vorangehenden Stufe "\$unwind", "\$group" oder "\$project" abhängt, verschieben Sie die Stufe "\$sort" in die Nähe des Beginns Ihrer Pipeline, um einen Index für die Sortierung anzuvisieren. Die Aggregations-Runtime muss daher keine teure In-Memory-Sortieroperation durchführen. Die Stufe "\$sort" ist nicht unbedingt die erste Stufe in Ihrer Pipeline, da es auch eine Stufe "\$match" geben kann, die denselben Index nutzt. Überprüfen Sie immer den explain-Plan, um sicherzustellen, dass Sie das beabsichtigte Verhalten herbeiführen.

2. Verwenden Sie Limit mit Sort. Wenn Sie nur die erste Teilmenge der Datensätze aus dem sortierten Datensatz benötigen, fügen Sie direkt nach der Stufe \$sort eine Stufe \$limit ein, die die Ergebnisse auf die von Ihnen benötigte feste Menge (z. B. 10) begrenzt. Zur Laufzeit fasst die Aggregations-Engine \$sort und \$limit zu einem einzigen speziellen internen Sortierschritt zusammen, der beide Aktionen gemeinsam durchführt. Der laufende Sortierprozess muss nur die zehn Datensätze im Speicher nachverfolgen, die der gerade ausgeführten Sortier-/Limitregel entsprechen. Er muss nicht den gesamten Datensatz im Speicher halten, um die Sortierung erfolgreich durchzuführen.

3. Reduzieren Sie die zu sortierenden Datensätze. Verschieben Sie die \$sort-Stufe so spät wie möglich in Ihrer Pipeline und stellen Sie sicher, dass frühere Stufen die Anzahl der Datensätze, die in diese späte blockierende \$sort-Stufe fließen, deutlich reduzieren. Diese blockierende Stufe hat weniger Datensätze zu verarbeiten und benötigt weniger RAM.

### \$group-Speicherverbrauch und Schadensbegrenzung

In der Realität konzentrieren sich die meisten Gruppierungsszenarien auf das Sammeln von zusammenfassenden Daten wie Summen, Zählungen, Durchschnittswerten, Höchst- und Tiefstwerten und nicht auf Einzeldaten. In diesen Situationen werden erheblich reduzierte Ergebnisdatensätze erzeugt, die weit weniger Verarbeitungsspeicher benötigen als eine \$sort-Stufe. Im Gegensatz zu vielen Sortierungsszenarien benötigen Gruppierungsoperationen normalerweise nur einen Bruchteil des Arbeitsspeichers des Hosts.

Um sicherzustellen, dass Sie einen übermäßigen Speicherverbrauch vermeiden, wenn Sie eine \$group-Stufe verwenden möchten, sollten Sie die folgenden Grundsätze beachten:
1. Unnötige Gruppierungen vermeiden. 
2. Nur Zusammenfassungsdaten gruppieren. Wenn der Anwendungsfall es zulässt, verwenden Sie die Gruppenstufe nur zum Akkumulieren von Dingen wie Summen, Zählungen und zusammenfassenden Roll-ups, anstatt alle Rohdaten jedes zu einer Gruppe gehörenden Datensatzes zu speichern. 
   
Vermeiden Sie das Abwickeln und Umgruppieren von Dokumenten, nur um Array-Elemente zu verarbeiten


2. Avoid Unwinding & Regrouping Documents Just To Process Array Elements
Sometimes, you need an aggregation pipeline to mutate or reduce an array field's content for each record. For example:
- You may need to add together all the values in the array into a total field
- You may need to retain the first and last elements of the array only
- You may need to retain only one recurring field for each sub-document in the array
- ...or numerous other array "reduction" scenarios

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
- Benutzer-Variablen binden. Zum Speichern von Werten, die Sie mit einem \$let-Operator deklarieren (oder mit der let-Option einer \$lookup-Stufe oder als Option einer \$map- oder \$filter-Stufe).  Beispiele: ``"$$product_name_var", "$$orderIdVal"``
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

### Sharding-Betrachtung TBD


### Advanced Use Of Expressions For Array Processing
