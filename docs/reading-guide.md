<img src="https://raw.githubusercontent.com/dennyglee/azure-cosmosdb-spark/master/docs/images/azure-cosmos-db-icon.png" width="75">  &nbsp; Azure Cosmos DB Connector for Apache Spark
==========================================
# Reading from Cosmos DB Guide

`azure-cosmosdb-spark` is the official connector for [Azure CosmosDB](http://cosmosdb.com) and [Apache Spark](http://spark.apache.org). The connector allows you to easily read to and write from Azure Cosmos DB via Apache Spark DataFrames in `python` and `scala`.  It also allows you to easily create a lambda architecture for batch-processing, stream-processing, and a serving layer while being globally replicated and minimizing the latency involved in working with big data. 


<!-- <details> -->

<strong><em>Table of Contents</em></strong>

* [Introduction](#introduction)
* [Reading Batch](#reading-batch)
  * [Aggregations](#aggregations)
  * [To cache or not to cache](#to-cache-or-not-to-cache)
* [Reading Change Feed](#reading-change-feed)


<!-- </details> -->


## Introduction

This guide provides code examples and references in `Python` and `Scala` on how to create a Spark DataFrame to read from Auzre Cosmos DB. It is important to note the following:

* The connector parses the `WHERE` clause for predicate pushdown to utilize the Cosmos DB indexes.  For more information on Cosmos DB indexes, please refer to [How does Azure Cosmos DB index data?](https://docs.microsoft.com/en-us/azure/cosmos-db/indexing-policies)
* If you supply a `query_custom` parameter, this will override the pushdown predicate and use the `query_custom` instead.
* For more information on the various configuration parameters, please refer to [Configuration Reference Guide](./configuration-reference-guide.md)
* Please refer to [TODO: Best Practices]() for reading best practices 

As well, it is important to note that Azure Cosmos DB, you can read from **batch** or from the Azure Cosmos DB **change feed** (more info at [Working with the change feed support in Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/change-feed)) if you want to read and store your events in *real time*.


## Reading Batch

The examples below are using `azure-cosmosdb-spark` to read in `python` and `scala` from the `DepartureDelays` database `flights_pcoll` collection within the Azure Cosmos DB account `doctorwho`.  This collection contains on-time flight performance data from the [United States Department of Transportation: Bureau of Transportation Statistics (TranStats)](https://www.transtats.bts.gov/DL_SelectFields.asp?Table_ID=236&DB_Short_Name=On-Time) between 2014-01-01 and 2014-03-31. These examples are based on the original blog post [On-Time Flight Performance with GraphFrames for Apache Spark](https://databricks.com/blog/2016/03/16/on-time-flight-performance-with-graphframes-for-apache-spark.html) and notebook [On-Time Flight Performance with GraphFrames for Apache Spark](https://cdn2.hubspot.net/hubfs/438089/notebooks/Samples/Miscellaneous/On-Time_Flight_Performance.html).



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

> Output: 23,078


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

> Output: 23,078



Note that now you have created the `flights` Spark DataFrame, from this point onwards you will be interacting with a Spark DataFrame.  


[<em>Back to the top</em>](#reading-from-cosmos-db-guide)

### Aggregations

Using the above on-time flight performance data from the `doctorwho` collection, we can easily create aggregations from our Cosmos DB collection.  Below are examples from Jupyter and Azure Databricks notebooks.  You can get the notebooks at: [TODO: Add links to Jupyter and Databricks notebooks]


#### Running LIMIT and COUNT queries

Now that you're using Spark SQL, you can run queries such as `select * from flights limit 10` or `select count(1) from flights` as noted in the Jupyter notebook screenshot below.

<img src="https://github.com/Azure/azure-documentdb-spark/blob/master/docs/images/aggregations/1.%20Spark%20SQL%20Query.png" width=650px>

This allows us to quickly see that there are 23,078 flights originating from Seattle in this dataset.

<img src="https://github.com/Azure/azure-documentdb-spark/blob/master/docs/images/aggregations/2.%20Count%20Query.png" width=650px>


#### Running GROUP BY Queries

To determine the top 10 cities where flights were delayed (`delay > 0`) originating from Seattle, we can run the Spark SQL query as noted in this Azure Databricks notebook screenshot.  

```python
%sql
select a.city as destination, sum(f.delay) as TotalDelays, count(1) as Trips
from flights f
join airports a
  on a.IATA = f.destination
where f.origin = 'SEA'
and f.delay > 0
group by a.city 
order by sum(delay) desc limit 10
```  


As expected, San Francisco had the most delayed flights departing from Seattle.

<img src="https://raw.githubusercontent.com/dennyglee/azure-cosmosdb-spark/master/docs/images/aggregations/top-10-delays-origin-Seattle.png">


#### Calculating Distributed Median

A median calculation can be resource intensive in a distributed environment.  Fortunately, Spark has the `percentile_approx` function that allows us to efficiently approximately calculate the median as noted in the Spark SQL statement below.

```python
%sql
select a.city as destination, percentile_approx(f.delay, 0.5) as median_delay
from flights f
join airports a
  on a.IATA = f.destination
where f.origin = 'SEA'
group by a.city 
order by percentile_approx(f.delay, 0.5)
```


The results can be seen in this Azure Databricks notebook noting that Jackson Hole had the least median delays while Cleveland had the most median delays.

<img src="https://github.com/dennyglee/azure-cosmosdb-spark/blob/master/docs/images/aggregations/median-delays-departing-Seattle.png?raw=true">

[<em>Back to the top</em>](#reading-from-cosmos-db-guide)


### To Cache or Not to Cache...



## Reading Change Feed

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

// Start reading change feed as a stream
var streamData = spark.readStream.format(classOf[CosmosDBSourceProvider].getName).options(sourceConfigMap).load()

// Start streaming query to memory
val query = streamData.groupBy("lang").count().sort($"count".desc).writeStream.outputMode("complete").format("memory").queryName("counts").start()
```


TODO: 
- Need to add Databricks notebook screenshots to showcase this in action
- Need to add notebook examples here




[<em>Back to the top</em>](#reading-from-cosmos-db-guide)

