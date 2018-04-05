<img src="https://raw.githubusercontent.com/dennyglee/azure-cosmosdb-spark/master/docs/images/azure-cosmos-db-icon.png" width="75">  &nbsp; Azure Cosmos DB Connector for Apache Spark
==========================================
# User Guide


Current Version of Spark Connector: 1.1.0 | [![Build Status](https://travis-ci.org/Azure/azure-cosmosdb-spark.svg?branch=master)](https://travis-ci.org/Azure/azure-cosmosdb-spark)



`azure-cosmosdb-spark` is the official connector for [Azure CosmosDB](http://cosmosdb.com) and [Apache Spark](http://spark.apache.org). The connector allows you to easily read to and write from Azure Cosmos DB via Apache Spark DataFrames in `python` and `scala`.  It also allows you to easily create a lambda architecture for batch-processing, stream-processing, and a serving layer while being globally replicated and minimizing the latency involved in working with big data. 


<!-- <details> -->

<strong><em>Table of Contents</em></strong>

* [Introduction](#introduction)
* [Requirements](#requirements)
* [Working with the connector](#working-with-the-connector)
  * [Using spark-cli](#using-spark-cli)
  * [Using Jupyter notebooks](#using-jupyter-notebooks)
  * [Using Databricks notebooks](#using-databricks-notebooks)
  * [Build the connector](#build-the-connector)
* [Reading from Cosmos DB](#reading-from-cosmos-db)
  * [Reading Batch](#reading-batch)
  * [Reading Change Feed](#reading-change-feed)
  * To cache or not to cache, that is the question
* [Writing to Cosmos DB](#writing-to-cosmos-db)
  * Python
  * Scala
  * Examples
* [Structured Streaming](#structured-streaming)
  * TTL
  * Reading Change Feed
    * Python
    * Scala
* [Lambda Architecture](#lambda-architecture)
  * Point to Lambda Architecture
* [Configuration Reference](#configuration-reference)
    * Parmeters
    * Reading
    * Writing

<!-- </details> -->


## Introduction

`azure-cosmosdb-spark` is the official connector for [Azure CosmosDB](http://cosmosdb.com) and [Apache Spark](http://spark.apache.org). The connector allows you to easily read to and write from Azure Cosmos DB via Apache Spark DataFrames in `python` and `scala`.  It also allows you to easily create a lambda architecture for batch-processing, stream-processing, and a serving layer while being globally replicated and minimizing the latency involved in working with big data. 

The connector utilizes the [Azure Cosmos DB Java SDK](https://github.com/Azure/azure-documentdb-java) via following data flow:

![](https://github.com/Azure/azure-cosmosdb-spark/blob/master/docs/images/diagrams/azure-cosmosdb-spark-flow_600x266.png?raw=true)

The data flow is as follows:

1. Connection is made from Spark driver node to Cosmos DB gateway node to obtain the partition map.  Note, user only specifies Spark and Cosmos DB connections, the fact that it connects to the respective master and gateway nodes is transparent to the user.
2. This information is provided back to the Spark master node.  At this point, we should be able to parse the query to determine which partitions (and their locations) within Cosmos DB we need to access.
3. This information is transmitted to the Spark worker nodes ...
4. Thus allowing the Spark worker nodes to connect directly to the Cosmos DB partitions directly to extract the data that is needed and bring the data back to the Spark partitions within the Spark worker nodes.

The important call out is that communication between Spark and Cosmos DB is significantly faster because the data movement is between the Spark worker nodes and the Cosmos DB data nodes (partitions).


[<em>Back to the top</em>](#user-guide)

&nbsp;

## Requirements

`azure-cosmosdb-spark` has been regularly tested using HDInsight 3.6 (Spark 2.1), 3.7 (Spark 2.2) and Azure Databricks Runtime 3.5 (Spark 2.2.1), 4.0 (Spark 2.3.0).

<em>Review <strong>supported</strong> component versions</em>

| Component | Versions Supported |
| --------- | ------------------ |
| Apache Spark | 2.2.1, 2.3 |
| Scala | 2.10, 2.11 |
| Python | 2.7, 3.6 |
| Azure Cosmos DB Java SDK | 1.16.1, 1.16.2 |


[<em>Back to the top</em>](#user-guide)

&nbsp;

## Working with the connector
You can build and/or use the maven coordinates to work with `azure-cosmosdb-spark`.

<em>Review the connector's <strong>maven versions</strong></em>


| Spark | Scala | Latest version |
|---|---|---|
| 2.2.0 | 2.11 | [azure-cosmosdb-spark_1.0.0-2.2.0_2.11](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.2.0_2.11/1.0.0) |
| 2.2.0 | 2.10 | [azure-cosmosdb-spark_1.0.0-2.2.0_2.10](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.2.0_2.10/1.0.0) |
| 2.1.0 | 2.11 | [azure-cosmosdb-spark_1.0.0-2.1.0_2.11](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.1.0_2.11/1.0.0) |
| 2.1.0 | 2.10 | [azure-cosmosdb-spark_1.0.0-2.1.0_2.10](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.1.0_2.10/1.0.0) |
| 2.0.2 | 2.11 | [azure-cosmosdb-spark_0.0.3-2.0.2_2.11](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.0.2_2.11/0.0.3) |
| 2.0.2 | 2.10 | [azure-cosmosdb-spark_0.0.3-2.0.2_2.10](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.0.2_2.10/0.0.3) |



### Using spark-cli
To work with the connector using the spark-cli (i.e. `spark-shell`, `pyspark`, `spark-submit`), you can use the `--packages` parameter with the connector's [maven coordinates](https://mvnrepository.com/artifact/com.microsoft.azure/azure-cosmosdb-spark_2.2.0_2.11).

```sh
spark-shell --master YARN --packages "com.microsoft.azure:azure-cosmosdb-spark_2.2.0_2.11:1.0.0"

```


### Using Jupyter notebooks
If you're using Jupyter notebooks within HDInsight, you can use spark-magic `%%configure` cell to specify the connector's maven coordinates.

```python
{ "name":"Spark-to-Cosmos_DB_Connector",
  "conf": {
    "spark.jars.packages": "com.microsoft.azure:azure-cosmosdb-spark_2.2.0_2.11:1.0.0",
    "spark.jars.excludes": "org.scala-lang:scala-reflect"
   }
   ...
}
```

> Note, the inclusion of the `spark.jars.excludes` is specific to remove potential conflicts between the connector, Apache Spark, and Livy.



### Using Databricks notebooks
Please create a library using within your Databricks workspace by following the guidance within the Azure Databricks Guide > [Use the Azure Cosmos DB Spark connector](https://docs.azuredatabricks.net/spark/latest/data-sources/azure/cosmosdb-connector.html)



### Build the connector
Currently, this connector project uses `maven` so to build without dependencies, you can run:

```sh
mvn clean package
```


[<em>Back to the top</em>](#user-guide)



## Reading from Cosmos DB
Below are code snippets in `Python` and `Scala` on how to create a Spark DataFrame to read from Cosmos DB. It is important to note the following:

* The connector parses the `WHERE` clause for predicate pushdown to utilize the Cosmos DB indexes.  For more information on Cosmos DB indexes, please refer to [How does Azure Cosmos DB index data?](https://docs.microsoft.com/en-us/azure/cosmos-db/indexing-policies)
* If you supply a `query_custom` parameter, this will override the pushdown predicate and use the `query_custom` instead.
* For more information on the various configuration parameters, please refer to [Configuration Reference Guide](./docs/configuration-reference-guide.md)
* Please refer to [TODO: Best Practices]() for reading best practices 

As well, it is important to note that Azure Cosmos DB, you can read from **batch** or from the Azure Cosmos DB **change feed** (more info at [Working with the change feed support in Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/change-feed)) if you want to read and store your events in *real time*.


### Reading Batch

Below are some common examples of reading from Azure Cosmos DB using `azure-cosmosdb-spark`.   

<em>_Python Example_</em>

```python
# Read Configuration
readConfig = {
  "Endpoint" : "https://doctorwho.documents.azure.com:443/",
  "Masterkey" : "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" : "DepartureDelays",
  "preferredRegions" : "Central US;East US2",
  "Collection" : "flights_pcoll", 
  "schema_samplesize" : "1000",
  "query_pagesize" : "2147483647"
}

# Specify query_custom
query_custom = "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'"
readConfig["query_custom"] = query_custom

# Connect via azure-cosmosdb-spark to create Spark DataFrame
flights = spark.read.format("com.microsoft.azure.cosmosdb.spark").options(**readConfig).load()
flights.count()
```


<em>_Scala Example_</em>
 
```scala
// Import Necessary Libraries
import com.microsoft.azure.cosmosdb.spark.schema._
import com.microsoft.azure.cosmosdb.spark._
import com.microsoft.azure.cosmosdb.spark.config.Config

// Configure connection to your collection
val readConfig = Config(Map(
  "Endpoint" -> "https://doctorwho.documents.azure.com:443/",
  "Masterkey" -> "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" -> "DepartureDelays",
  "PreferredRegions" -> "Central US;East US2;",
  "Collection" -> "flights_pcoll", 
  "SamplingRatio" -> "1.0",
  "query_custom" -> "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'"
))

// Connect via azure-cosmosdb-spark to create Spark DataFrame
val flights = spark.read.cosmosDB(readConfig)
flights.count()
```



### Reading Change Feed

To read the Azure Cosmos DB change feed, you need only to add the `ChangeFeedCheckpointLocation` and `ChangeFeedQueryName` to read it instead of collection (i.e. batch) directly.  But you should also add the `ConnectionMode` and `query_custom` parameters for better performance.

<em>_Python Example_</em>

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


<em>_Scala Example_</em>

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



[<em>Back to the top</em>](#user-guide)


### Writing to Cosmos DB
Below are excerpts in `Python` and `Scala` on how to write a Spark DataFrame to Cosmos DB 

```python
# Write configuration
writeConfig = {
 "Endpoint" : "https://doctorwho.documents.azure.com:443/",
 "Masterkey" : "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
 "Database" : "DepartureDelays",
 "Collection" : "flights_fromsea",
 "Upsert" : "true"
}

# Write to Cosmos DB from the flights DataFrame
flights.write.format("com.microsoft.azure.cosmosdb.spark").options(**writeConfig).save()
```

<details>
<summary><em>Click for <strong>Scala</strong> Excerpt</em></summary>
<p>

```scala
// Configure connection to the sink collection
val writeConfig = Config(Map(
  "Endpoint" -> "https://doctorwho.documents.azure.com:443/",
  "Masterkey" -> "SPSVkSfA7f6vMgMvnYdzc1MaWb65v4VQNcI2Tp1WfSP2vtgmAwGXEPcxoYra5QBHHyjDGYuHKSkguHIz1vvmWQ==",
  "Database" -> "DepartureDelays",
  "PreferredRegions" -> "Central US;East US2;",
  "Collection" -> "flights_fromsea",
  "WritingBatchSize" -> "100"
))

// Upsert the dataframe to Cosmos DB
import org.apache.spark.sql.SaveMode
flights.write.mode(SaveMode.Overwrite).cosmosDB(writeConfig)
```

</p>
</details>

&nbsp;

See other sample [Jupyter](https://github.com/dennyglee/azure-cosmosdb-spark/tree/master/samples/notebooks) and [Databricks]() notebooks as well as [PySpark]() and [Spark]() scripts.



### Connecting Spark to Cosmos DB via the azure-cosmosdb-spark
While the communication transport is a little more complicated, executing a query from Spark to Cosmos DB using `azure-cosmosdb-spark` is significantly faster.

Below is a code snippet on how to use `azure-cosmosdb-spark` within a Spark context.

#### Python
```python
# Base Configuration
flightsConfig = {
"Endpoint" : "https://doctorwho.documents.azure.com:443/",
"Masterkey" : "le1n99i1w5l7uvokJs3RT5ZAH8dc3ql7lx2CG0h0kK4lVWPkQnwpRLyAN0nwS1z4Cyd1lJgvGUfMWR3v8vkXKA==",
"Database" : "DepartureDelays",
"preferredRegions" : "Central US;East US2",
"Collection" : "flights_pcoll", 
"SamplingRatio" : "1.0",
"schema_samplesize" : "1000",
"query_pagesize" : "2147483647",
"query_custom" : "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'"
}

# Connect via Spark connector to create Spark DataFrame
flights = spark.read.format("com.microsoft.azure.cosmosdb.spark").options(**flightsConfig).load()
flights.count()
```

#### Scala
```scala
// Import Necessary Libraries
import org.joda.time._
import org.joda.time.format._
import com.microsoft.azure.cosmosdb.spark.schema._
import com.microsoft.azure.cosmosdb.spark._
import com.microsoft.azure.cosmosdb.spark.config.Config

// Configure connection to your collection
val readConfig2 = Config(Map("Endpoint" -> "https://doctorwho.documents.azure.com:443/",
"Masterkey" -> "le1n99i1w5l7uvokJs3RT5ZAH8dc3ql7lx2CG0h0kK4lVWPkQnwpRLyAN0nwS1z4Cyd1lJgvGUfMWR3v8vkXKA==",
"Database" -> "DepartureDelays",
"preferredRegions" -> "Central US;East US2;",
"Collection" -> "flights_pcoll", 
"query_custom" -> "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.origin = 'SEA'"
"SamplingRatio" -> "1.0"))
 
// Create collection connection 
val coll = spark.sqlContext.read.cosmosDB(readConfig2)
coll.createOrReplaceTempView("c")
```

As noted in the code snippet:

- `azure-cosmosdb-spark` contains the all the necessary connection parameters including the preferred locations (i.e. choosing which read replica in what priority order).
- Just import the necessary libraries and configure your masterKey and host to create the Cosmos DB client.

### Executing Spark Queries via azure-cosmosdb-spark

Below is an example using the above Cosmos DB instance via the specified read-only keys. This code snippet below connects to the DepartureDelays.flights_pcoll collection (in the DoctorWho account as specified earlier) running a query to extract the flight delay information of flights departing from Seattle.

```
// Queries
var query = "SELECT c.date, c.delay, c.distance, c.origin, c.destination FROM c WHERE c.destination = 'SFO'"
val df = spark.sql(query)

// Run DF query (count)
df.count()

// Run DF query (show)
df.show()
```

### Scenarios

Connecting Spark to Cosmos DB using `azure-cosmosdb-spark` are typically for scenarios where:

* You want to use Python and/or Scala
* You have a large amount of data to transfer between Apache Spark and Cosmos DB


To give you an idea of the query performance difference, please refer to [Query Test Runs](https://github.com/Azure/azure-documentdb-spark/wiki/Query-Test-Runs) in this wiki.