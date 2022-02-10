MongoDB aggregations

https://www.practical-mongodb-aggregations.com/intro/introducing-aggregations.html

Aggregation Framework is: 
- Aggregation API to define aggregation tasks (pipeline)
- Aggregation runtime

Mongo DB’s Aggregations Language is:
 It is Turing complete and able to solve any business problem *. Conversely, it is a strongly opinionated [Domain Specific Language (DSL)](https://en.wikipedia.org/wiki/Domain-specific_language).
You can even code a [Bitcoin miner](https://github.com/johnlpage/MongoAggMiner/blob/master/aggmine3.js) using MongoDB aggregations.
MongoDB's aggregation pipeline language is focused on data-oriented problem-solving. It is essentially a declarative programming language, rather than an imperative programming language. Also, it can be regarded as a functional programming language rather than a procedural programming language. An aggregation pipeline is an ordered series of declarative statements, called stages, where the entire output of one stage forms the entire input of the next stage, and so on, with no side effects. 

### Embrace composability for increased productivity

An aggregation pipeline is an ordered series of declarative statements, called stages.

### Better Alternatives To A Project Stage

History&Legacy&Behaviour of \$addFields -> \$set/\$unset
MongoDB version 3.4 addressed some of the disadvantages of \$project by introducing a new \$addFields stage, which has the same behaviour as \$set. \$set came later than \$addFieldsand \$set is actually just an alias for \$addFields. However, back then, the Aggregation Framework provided no direct equivalent to \$unset. Both \$set and \$unset stages are available in modern versions of MongoDB, and their counter purposes are obvious to deduce by their names (\$set Vs \$unset). The name \$addFields doesn't fully reflect that you can modify existing fields rather than just adding new fields. This book prefers \$set over \$addFields to help promote consistency and avoid any confusion of intent. However, if you are wedded to \$addFields, use that instead, as there is no behavioural difference.

### When To Use \$set & \$unset

You should use \$set & \$unset stages when you need to retain most of the fields in the input records, and you want to add, modify or remove a minority subset of fields.

``INPUT  (a record from the source collection to be operated on by an aggregation)``
```json
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
```


``// OUTPUT  (a record in the results of the executed aggregation)``
```json{
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
```

``// GOOD``
```json[
   {"$set": {
           // Modified + new field
           "card_expiry": {"$dateFromString": {"dateString": "$card_expiry"}},
           "card_type": "CREDIT",
       }},

   {"$unset": [
           // Remove _id field
           "_id",
       ]},
]
```

### When To Use \$project
It is best to use a \$project stage when the required shape of output documents is very different from the input documents' shape. This situation often arises when you do not need to include most of the original fields.
This time for the same input payments collection.You need each output document's structure to be very different from the input structure, and you need to retain far fewer original fields:


``// OUTPUT  (a record in the results of the executed aggregation)``
```json{
   transaction_info: {
       date: ISODate("2021-01-13T09:32:07.000Z"),
           amount: NumberDecimal("501.98")
   },
   status: "REPORTED"
}
```


```json[
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
```

In summary, you should always look to use \$set (or \$addFields) and \$unset for field inclusion and exclusion, rather than \$project. The main exception is if you have an obvious requirement for a very different structure for result documents, where you only need to retain a small subset of the input fields.

### Using Explain Plans

To view the explain plan for an aggregation pipeline, you can execute commands such as the following:

``db.coll.explain().aggregate([{"$match": {"name": "Jo"}}]);``

As with MQL, there are three different verbosity modes that you can generate an explain plan with:


``// QueryPlanner verbosity  (default if no verbosity parameter provided)``<br>
``db.coll.explain("queryPlanner").aggregate(pipeline);``

``// ExecutionStats verbosity``<br>
``db.coll.explain("executionStats").aggregate(pipeline);``

``// AllPlansExecution verbosity``<br>
``db.coll.explain("allPlansExecution").aggregate(pipeline);``


In most cases, you will find that running the executionStats variant is the most informative mode. Rather than showing just the query planner's thought process, it also provides actual statistics on the "winning" execution plan (e.g. the total keys examined, the total docs examined, etc.). However, this isn't the default because it actually executes the aggregation in addition to formulating the query plan. If the source collection is large or the pipeline is suboptimal, it will take a while to return the explain plan result.
Note, the aggregate() function also provides a vestigial explain option to ask for an explain plan to be generated and returned. Nonetheless, this is more limited and cumbersome to use, so you should avoid it.

### Understanding The Explain Plan
The customer orders collection contains documents similar to the following example:

```json{
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
```

You've defined an index on the customer_id field. You create the following aggregation pipeline to show the three most expensive orders made by a customer whose ID is tonijones@myemail.com:

```jsonvar pipeline = [
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
```

You then request the query planner part of the explain plan:
The query plan output for this pipeline shows the following (excluding some information for brevity):

```jsonstages: [
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
```


You can deduce some illuminating insights from this query plan:

To optimise the aggregation, the database engine has reordered the pipeline positioning the filter belonging to the \$match to the top of the pipeline. 
The first stage of the database optimised version of the pipeline is an internal \$cursor stage.
To further optimise the aggregation, the database engine has collapsed the \$sort and \$limit into a single special internal sort stage which can perform both actions in one go.
You ask for the execution stats part of the explain plan:

``db.customer_orders.explain("executionStats").aggregate(pipeline);``



```jsonexecutionStats: {
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
```


Here, this part of the plan also shows that the aggregation uses the existing index. Because totalKeysExamined and totalDocsExamined match, the aggregation fully leverages this index to identify the required records, which is good news. Nevertheless, the targeted index doesn't necessarily mean the aggregation's query part is fully optimised. For example, if there is the need to reduce latency further, you can do some analysis to determine if the index can completely cover the query. Suppose the cursor query part of the aggregation is satisfied entirely using the index and does not have to examine any raw documents. In that case, you will see totalDocsExamined: 0 in the explain plan.

### Pipeline Performance Considerations
1. Be Cognizant Of Streaming Vs Blocking Stages Ordering
When executing an aggregation pipeline, the database engine pulls batches of records from the initial query cursor generated against the source collection. The database engine then attempts to stream each batch through the aggregation pipeline stages. For most types of stages, referred to as streaming stages, the database engine will take the processed batch from one stage and immediately stream it into the next part of the pipeline. It will do this without waiting for all the other batches to arrive at the prior stage. However, two types of stages must block and wait for all batches to arrive and accumulate together at that stage. These two stages are referred to as blocking stages and specifically, the two types of stages that block are:
- \$sort
- \$group *
* actually when stating \$group, this also includes other less frequently used "grouping" stages too, specifically:\$bucket, \$bucketAuto, \$count, \$sortByCount & \$facet  (it's a stretch to call \$facet a group stage, but in the context of this topic, it's best to think of it that way)

\$sort Memory Consumption And Mitigation

Used naïvely, a \$sort stage will need to see all the input records at once, and so the host server must have enough capacity to hold all the input data in memory. The amount of memory required depends heavily on the initial data size and the degree to which the prior stages can reduce the size. Also, multiple instances of the aggregation pipeline may be in-flight at any one time, in addition to other database workloads. Therefore, MongoDB enforces every stage is limited to 100 MB of consumed RAM. The database throws an error if it exceeds this limit. To avoid the memory limit obstacle, you can set the allowDiskUse:true option for the overall aggregation for handling large result data sets. Consequently, the pipeline's sort operation spills to disk if required, and the 100 MB limit no longer constrains the pipeline. However, the sacrifice here is significantly higher latency, and the execution time is likely to increase by orders of magnitude.

To circumvent the aggregation needing to manifest the whole data set in memory or overspill to disk, attempt to refactor your pipeline to incorporate one of the following approaches (in order of most effective first):

Use Index Sort. If the \$sort stage does not depend on a \$unwind, \$group or \$project stage preceding it, move the \$sort stage to near the start of your pipeline to target an index for the sort. The aggregation runtime does not need to perform an expensive in-memory sort operation as a result. The \$sort stage won't necessarily be the first stage in your pipeline because there may also be a \$match stage that takes advantage of the same index. Always inspect the explain plan to ensure you are inducing the intended behaviour.

Use Limit With Sort. If you only need the first subset of records from the sorted set of data, add a $limit stage directly after the \$sort stage, limiting the results to the fixed amount you require (e.g. 10). At runtime, the aggregation engine will collapse the $sort and $limit into a single special internal sort stage which performs both actions together. The in-flight sort process only has to track the ten records in memory, which currently satisfy the executing sort/limit rule. It does not have to hold the whole data set in memory to execute the sort successfully.

Reduce Records To Sort. Move the \$sort stage to as late as possible in your pipeline and ensure earlier stages significantly reduce the number of records streaming into this late blocking \$sort stage. This blocking stage will have fewer records to process and less thirst for RAM.

\$group Memory Consumption And Mitigation
In reality, most grouping scenarios focus on accumulating summary data such as totals, counts, averages, highs and lows, and not itemised data. In these situations, considerably reduced result data sets are produced, requiring far less processing memory than a \$sort stage. Contrary to many sorting scenarios, grouping operations will typically demand a fraction of the host's RAM.

To ensure you avoid excessive memory consumption when you are looking to use a \$group stage, adopt the following principles:
Avoid Unnecessary Grouping. 

Group Summary Data Only. If the use case permits it, use the group stage to accumulate things like totals, counts and summary roll-ups only, rather than holding all the raw data of each record belonging to a group. 

2. Avoid Unwinding & Regrouping Documents Just To Process Array Elements
Sometimes, you need an aggregation pipeline to mutate or reduce an array field's content for each record. For example:
You may need to add together all the values in the array into a total field
You may need to retain the first and last elements of the array only
You may need to retain only one recurring field for each sub-document in the array
...or numerous other array "reduction" scenarios

To bring this to life, imagine a retail orders collection where each document contains an array of products purchased as part of the order, as shown in the example below:

```json[
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
```



The retailer wants to see a report of all the orders but only containing the expensive products purchased by customers (e.g. having just products priced greater than 15 dollars). Consequently, an aggregation is required to filter out the inexpensive product items of each order's array.

One naïve way of achieving this transformation is to unwind the products array of each order document to produce an intermediate set of individual product records.
// SUBOPTIMAL

```jsonvar pipeline = [
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
   },
];
```

This pipeline is suboptimal because a $group stage has been introduced, which is a blocking stage, as outlined earlier in this chapter. Both memory consumption and execution time will increase significantly, which could be fatal for a large input data set. There is a far better alternative by using one of the Array Operators instead. Array Operators are sometimes less intuitive to code, but they avoid introducing a blocking stage into the pipeline. Consequently, they are significantly more efficient, especially for large data sets. Shown below is a far more economical pipeline, using the $filter array operator, rather than the \$unwind/\$match/\$group combination, to produce the same outcome:

// OPTIMAL

```jsonvar pipeline = [
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
```


Unlike the suboptimal pipeline, the optimal pipeline will include "empty orders" in the results for those orders that contained only inexpensive items. If this is a problem, you can include a simple \$match stage at the start of the optimal pipeline with the same content as the \$match stage shown in the suboptimal example.

To reiterate, there should never be the need to use an \$unwind/\$group combination in an aggregation pipeline to transform an array field's elements for each document in isolation. One way to recognize if you have this anti-pattern is if your pipeline contains a \$group on a \$_id field. Instead, use Array Operators to avoid introducing a blocking stage. 
The primary use of an \$unwind/\$group combination is to correlate patterns across many records' arrays rather than transforming the content within each input record's array only.

3. Encourage Match Filters To Appear Early In The Pipeline
Explore If Bringing Forward A Full Match Is Possible
Explore If Bringing Forward A Partial Match Is Possible

In summary, if you have a pipeline leveraging a \$match stage and the explain plan shows this is not moving to the start of the pipeline, explore whether manually refactoring will help. If the \$match filter depends on a computed value, examine if you can alter this or add an extra \$match to yield a more efficient pipeline.


## Expressions Explained
### Summarising Aggregation Expressions
Aggregation expressions come in one of three primary flavours:
1. Operators. Accessed as an object with a \$ prefix followed by the operator function name. The "dollar-operator-name" is used as the main key for the object. 
Examples: ``{\$arrayElemAt: ...}, {\$cond: ...}, {\$dateToString: ...}``
2. Field Paths. Accessed as a string with a \$ prefix followed by the field's path in each record being processed.  Examples: "\$account.sortcode", "\$addresses.address.city"
3. Variables. Accessed as a string with a \$$ prefix followed by the fixed name and falling into three sub-categories:
- Context System Variables. With values coming from the system environment rather than each input record an aggregation stage is processing.  Examples: ``"$$NOW", "$$CLUSTER_TIME"``
- Marker Flag System Variables. To indicate desired behaviour to pass back to the aggregation runtime.  Examples: ``"\$\$ROOT", "\$\$REMOVE", "\$\$PRUNE"``
- Bind User Variables. For storing values you declare with a \$let operator (or with the let option of a \$lookup stage, or as option of a \$map or \$filter stage).  Examples: ``"\$\$product_name_var", "\$\$orderIdVal"``
You can combine these three categories of aggregation expressions when operating on input records, enabling you to perform complex comparisons and transformations of data. 


```json
{"customer_info": {"$cond": {
   "if":   {"$eq": ["$customer_info.category", "SENSITIVE"]},
   "then": "$$REMOVE",
       "else": "$customer_info",
}}}
```

### What Do Expressions Produce?
An expression can be an Operator (e.g. {\$concat: ...}), a Variable (e.g. "\$\$ROOT") or a Field Path (e.g. "\$address"). In all these cases, an expression is just something that dynamically populates and returns a new JSON/BSON data type element, which can be one of:
- a Number  (including integer, long, float, double, decimal128)
- a String  (UTF-8)
- a Boolean
- a DateTime  (UTC)
- an Array
- an Object

However, a specific expression can restrict you to returning just one or a few of these types. For example, the ``{\$concat: ...}`` Operator, which combines multiple strings, can only produce a String data type (or null). The Variable "\$\$ROOT" can only return an Object which refers to the root document currently being processed in the pipeline stage. A Field Path (e.g. "\$address") is different and can return an element of any data type, depending on what the field refers to in the current input document. 
In summary, Field Paths and Bind User Variables are expressions that can return any JSON/BSON data type at runtime depending on their context. For the other kinds of expressions (Operators, Context System Variables and Marker Flag System Variables), the data type each can return is fixed to one or a set number of documented types. To establish the exact data type produced by these specific operators, you need to consult the [Aggregation Pipeline Quick Reference documentation](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/).
For the Operator category of expressions, an expression can also take other expressions as parameters, making them composable.

### Can All Stages Use Expressions?
There are many types of stages in the Aggregation Framework that don't allow expressions to be embedded. Examples of some of the most commonly used of these stages are:
- \$match
- \$limit
- \$skip
- \$sort
- \$count
- \$lookup
- \$out

### Restrictions When Using Expressions with \$match

You should be aware that there are restrictions on when the runtime can benefit from an index when using a \$expr operator inside a \$match stage. This partly depends on the version of MongoDB you are running. Using \$expr, you can leverage a \$eq comparison operator with some constraints, including an inability to use a multi-key index. For MongoDB versions before 5.0, if you use a "range" comparison operator (\$gt, \$gte, \$lt and \$lte), an index cannot be employed to match the field, but this works fine in version 5.0.

### Sharding Considerations TBD

### Advanced Use Of Expressions For Array Processing


