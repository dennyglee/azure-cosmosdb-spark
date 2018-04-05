<img src="https://raw.githubusercontent.com/dennyglee/azure-cosmosdb-spark/master/docs/images/azure-cosmos-db-icon.png" width="75">  &nbsp; Azure Cosmos DB Connector for Apache Spark
==========================================
# Configuration Reference Guide

This guide contains the available configurations of `azure-cosmosdb-spark`. Depending on your scenario, different configurations should be used to optimize your performance and throughput.  Note, these configurations are passed to the [Azure Cosmos DB Java SDK](https://github.com/Azure/azure-documentdb-java).

> Note that the configuration key is case-insensitive and for now, the configuration value is always a string.


<strong><em>Table of Contents</em></strong>

* [Reading from a Cosmos DB Collection](#reading-from-a-cosmos-db-collection)
  * [Required Parameters](#reading-required-parameters)
  * [Common Batch Parameters](#reading-common-batch-parameters)
  * [Required Change Feed Parameters](#reading-required-change-feed-parameters)
  * [Optional Change Feed Parameters](#reading-optional-change-feed-parameters)
  * [Optional Parameters](#reading-optional-parameters)
* [Writing to a Cosmos DB Collection](#writing-to-a-cosmos-db-collection)
  * [Required Parameters](#rwriting-required-parameters)
  * [Common Parameters](#writing-common-parameters)
  * [Miscellaneous Optional Parameters](#writing-miscellaneous-optional-parameters)


&nbsp;


## Reading from Cosmos DB Collection

This section contains the configuration parameters used by `azure-cosmosdb-spark` to read an Azure Cosmos DB collection. For example, below are code-snippets in `Python` and `Scala`    

<em>_Python Example_</em>

```python
# Configure connection to your collection (Python)
readConfig = {
  "Endpoint" : "https://doctorwho.documents.azure.com:443/",
  "Masterkey" : "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" : "DepartureDelays",
  "Collection" : "flights_pcoll" 
}
```


<em>Scala Example</em>
```scala
// Configure connection to your collection (Scala)
val readConfig = Config(Map(
  "Endpoint" -> "https://doctorwho.documents.azure.com:443/",
  "Masterkey" -> "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" -> "DepartureDelays",
  "Collection" -> "flights_pcoll"
))
```



### Reading: Required Parameters

These parameters are required for `azure-cosmosdb-spark` to read from an Azure Cosmos DB collection.

| Parameter | Description |
| --------- | ----------- | 
| `Collection` | This specifies the name of your Azure Cosmos DB collection |
| `Database` | This specifies the name of your Azure Cosmos DB database |
| `Endpoint` | This specifies the endpoint in the format of a URI to connect to your Azure Cosmos DB account, e.g. `https://doctorwho.documents.azure.com:443/` |
| `Masterkey` | This specifies the master key that allows `azure-cosmosdb-spark` to connect to your endpoint, database, and collection. |


&nbsp;


### Reading: Common Batch Parameters

While these parameters are optional, these are often used to improve read batch performance.

| Parameter | Description |
| --------- | ----------- | 
| `query_custom` | Use this parameter to override `azure-cosmosdb-spark` predicate pushdown by parsing the Spark SQL `WHERE` clause, e.g. `SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'` |
| `query_pagesize` | Sets the size of the query result page for each query request. To optimized for throughput, <ul><li> Use a large page size to reduce the number of round trips to fetch queries results when using a predicate <li> Use a small page size to reduce the likelihood of 429 errors (request rate too large) when specifying a `query_custom` that does not have a predicate pushdown. </ul> |
| `PreferredRegions` | This specifies the name of your Azure Cosmos DB collection |
| `SamplingRatio` | Typically set to `1.0` | 
| `schema_samplesize` | Number of documents / rows to scan so that Apache Spark can infer the schema.  In a collection you can have multiple document types; it may be necessary to increase this number to scan enough of the documents in your collection to Spark can infer the schema of all of the different schemas and merge them into a single Spark DataFrame. |  

&nbsp;

<details>
<summary><em>Click for <strong>Python</strong> Example</em></summary>
<p>

```python
# Configure connection to your collection (Python)
readConfig = {
  "Endpoint" : "https://doctorwho.documents.azure.com:443/",
  "Masterkey" : "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" : "DepartureDelays",
  "Collection" : "flights_pcoll",
  "PreferredRegions" : "West US; Southeast Asia"
  "schema_samplesize" : "1000",
  "query_pagesize" : "200000",
  "query_custom" : "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'" 
}
```
</p>
</details>

&nbsp;

<details>
<summary><em>Click for <strong>Scala</strong> Example</em></summary>
<p>

```scala
// Configure connection to your collection (Scala)
val readConfig = Config(Map(
  "Endpoint" -> "https://doctorwho.documents.azure.com:443/",
  "Masterkey" -> "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" -> "DepartureDelays",
  "Collection" -> "flights_pcoll",
  "PreferredRegions" -> "West US; Southeast Asia"
  "schema_samplesize" -> "1000",
  "query_pagesize" -> "200000",
  "query_custom" -> "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'" 
))

```

</p>
</details>


&nbsp;

### Reading: Required Change Feed Parameters

To read the Azure Cosmos DB change feed, you need only to add the `ChangeFeedCheckpointLocation` and `ChangeFeedQueryName` to read it instead of collection (i.e. batch) directly.  But you should also add the `ConnectionMode` and `query_custom` parameters for better performance.


| Parameter | Description |
| --------- | ----------- | 
| `ChangeFeedCheckpointLocation` | The path to the local file storage to persist continuation tokens in case of node failures, e.g. `/tmp/checkpointlocation1406` |
| `ChangeFeedQueryName` | A custom string to identify the query. The connector keeps track of the collection continuation tokens for different change feed queries separately. If `readchangefeed` is true, this is a **required** configuration. | 
| `ConnectionMode` | Specifies if the connection is through the data partitions or gateway, for change feed, please specify `Gateway`. |
| `query_custom` | This is the same `query_custom` from [Common Batch Parameters](#reading-common-batch-parameters). For any large documents (e.g. Twitter JSON, use this parameter to reduce the size of the change feed read, e.g. `SELECT c.id, c.created_at, c.user.screen_name,  c.user.lang, c.user.location, c.text, c.retweet_count, c.entities.hashtags, c.entities.user_mentions, c.favorited, c.source FROM c` |


&nbsp;

<details>
<summary><em>Click for <strong>Python</strong> Example</em></summary>
<p>

```python
# Configure connection to your collection (Python)
readConfig = {
  "Endpoint" : "https://doctorwho.documents.azure.com:443/",
  "Masterkey" : "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" : "DepartureDelays",
  "Collection" : "flights_pcoll",
  "ConnectionMode" : "Gateway",
  "ChangeFeedQueryName" : "CF1406",
  "ChangeFeedCheckPointLocation" : "/tmp/checkpointlocation1406",
  "query_custom" : "SELECT c.id, c.created_at, c.user.screen_name,  c.user.lang, c.user.location, c.text, c.retweet_count, c.entities.hashtags, c.entities.user_mentions, c.favorited, c.source FROM c" 
}
```
</p>
</details>

&nbsp;

<details>
<summary><em>Click for <strong>Scala</strong> Example</em></summary>
<p>

```scala
// Configure connection to your collection (Scala)
val readConfig = Config(Map(
  "Endpoint" -> "https://doctorwho.documents.azure.com:443/",
  "Masterkey" -> "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" -> "DepartureDelays",
  "Collection" -> "flights_pcoll",
  "ConnectionMode" -> "Gateway"
  "ChangeFeedQueryName" -> "CF1406",
  "ChangeFeedCheckPointLocation" -> "/tmp/checkpointlocation1406",
  "query_custom" -> "SELECT c.id, c.created_at, c.user.screen_name,  c.user.lang, c.user.location, c.text, c.retweet_count, c.entities.hashtags, c.entities.user_mentions, c.favorited, c.source FROM c" 
))

```

</p>
</details>


&nbsp;

### Reading: Optional Change Feed Parameters

Below are optional change feed parameters.


| Parameter | Description |
| --------- | ----------- | 
| `changefeedstartfromthebeginning` | Sets whether change feed should start from the beginning (`true`) or from the current point in time by default (`false`). |
| `changefeedusenexttoken` | A boolean value to support processing failure scenarios. It is used to indicate that the current change feed batch has been handled gracefully and the RDD should use the next continuation tokens to get the subsequent batch of changes. |
| `readchangefeed` | Indicates that the collection content is fetched from CosmosDB Change Feed. The default value is `false` |
| `rollingchangefeed` | Indicates whether the change feed should be from the last query. The default value is `false`, which means the changes will be counted from the first read of the collection. | 


&nbsp;



### Reading: Optional Parameters

These miscellaneous optional parameters allow you finer grain control of your read if so desired.

| Parameter | Description |
| --------- | ----------- |
| `query_emitverbosetraces` | Sets the option to allow queries to emit out verbose traces for investigation. | 
| `query_enablescan` | Sets the option to enable scans on the queries which could not be served by an Azure Cosmos DB index because it was disabled (indexes on all attributes are enabled by default in Azure Cosmos DB). |
| `query_maxbuffereditemcount` | This sets the maximum number of items that can be buffered client side during parallel query execution. A positive property value limits the number of buffered items to the set value. If it is set to less than 0, the system automatically decides the number of items to buffer. |
| `query_maxdegreeofparallelism` | This sets the number of concurrent operations that execute from the client side during parallel query execution. A positive property value limits the number of concurrent operations to the set value. If it is set to less than 0, the system automatically decides the number of concurrent operations to run. *As `azure-cosmosdb-spark` maps each collection partition with an executor, this value will have minimal effect on the reading operation.* |


[<em>Back to the top</em>](#configuration-reference-guide)


