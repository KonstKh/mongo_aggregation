Praktische MondoDb Aggregationen

- [Definition](#definition)

Aggregations Framework ist: 
<ul>
<li>Aggregations-API zur Definition von Aggregations-Taks (Pipeline)</li>
<li>Aggregations-Laufzeit</li>
</ul>

# Definition
Die Aggregations-Sprache von Mongo DB ist:
 Sie ist Turing-komplett und in der Lage, jedes Geschäftsproblem zu lösen *. Umgekehrt ist sie eine stark eigenwillige domänenspezifische Sprache [(DSL)](https://en.wikipedia.org/wiki/Domain-specific_language).
Sie können sogar einen [Bitcoin-Miner](https://github.com/johnlpage/MongoAggMiner/blob/master/aggmine3.js) mit MongoDB-Aggregationen programmieren.
Die Aggregations-Pipeline-Sprache von MongoDB ist auf datenorientierte Problemlösungen ausgerichtet. Es handelt sich im Wesentlichen um eine deklarative Programmiersprache und nicht um eine imperative Programmiersprache. Außerdem kann sie als funktionale Programmiersprache und nicht als prozedurale Programmiersprache betrachtet werden. Eine Aggregationspipeline ist eine geordnete Reihe von deklarativen Anweisungen, die als Stufen bezeichnet werden, wobei die gesamte Ausgabe einer Stufe die gesamte Eingabe der nächsten Stufe bildet, und so weiter, ohne Seiteneffekte. 

