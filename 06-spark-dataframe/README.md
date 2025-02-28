# Data Reading and Writing using DataFrames

## Introduction

In this workshop we will work with [Apache Spark](https://spark.apache.org/) DataFrames and 
We assume that the **Data platform** described [here](../01-environment) is running and accessible. 

The same data as in the [HDFS Workshop](../02-hdfs/README.md) or [Object Storage Workshop](../03-object-storage/README.md) will be used. We will show later how to re-upload the files, if you no longer have them available.

We assume that you have done Workshop 5 **Getting Started using Spark RDD and DataFrames**, where you have learnt how to use Spark form either `pyspark`, Apache Zeppelin or Jupyter Notebook. 
 
## Prepare the data, if no longer available

The data needed here has been uploaded to MinIO in workshop 03-object-storage. You can skip this section, if you still have the data available in HDFS or MinIO. 

Create the flight bucket:

```
docker exec -ti awscli s3cmd mb s3://flight-bucket
```

Airports:

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/airports.csv s3://flight-bucket/raw-data/airports/airports.csv
```

Plane-Data:

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/plane-data.csv s3://flight-bucket/raw-data/planes/plane-data.csv
```

Carriers:

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/carriers.json s3://flight-bucket/raw-data/carriers/carriers.json
```

Flights (upload them one by one):

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/flights-small/flights_2008_4_1.csv s3://flight-bucket/raw-data/flights/
```

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/flights-small/flights_2008_4_2.csv s3://flight-bucket/raw-data/flights/
```

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/flights-small/flights_2008_5_1.csv s3://flight-bucket/raw-data/flights/
```

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/flights-small/flights_2008_5_2.csv s3://flight-bucket/raw-data/flights/
```

```
docker exec -ti awscli s3cmd put /data-transfer/flight-data/flights-small/flights_2008_5_3.csv s3://flight-bucket/raw-data/flights/
```

## Create a new Zeppelin notebook

For this workshop we will be using Zeppelin discussed above. 

But you can easily adapt it to use either **PySpark** or **Apache Jupyter**.

In a browser window, navigate to <http://dataplatform:28080> and make sure that you configure the **Spark** Interpreter as discussed above.

Now let's create a new notebook by clicking on the **Create new note** link and set the **Note Name** to `SparkDataFrame` and set the **Default Interpreter** to `spark`. 

Click on **Create Note** and a new Notebook is created with one cell which is empty. 

### Add some Markdown first

Navigate to the first cell and start with a title. By using the `%md` directive we can switch to the Markdown interpreter, which can be used for displaying static text.

```
%md # Spark DataFrame sample with flights data
```

Click on the **>** symbol on the right or enter **Shift** + **Enter** to run the paragraph.

The markdown code should now be rendered as a Heading-1 title.

## Working with the Airport Data

First add another title, this time as a Heading-2.

```
%md ## Working with the Airport data
```

Now let's work with the Airports data, which we have uploaded to `s3://flight-bucket/raw-data/airports/`. Let's see the data by performing an `%sh` directive

```
%sh
s3cmd ls -r s3://flight-bucket/raw-data/airports/
```

We can see that there is one file for the airports data.

![Alt Image Text](./images/zeppelin-sh-ls-airports.png "Zeppelin Show Files")

Now let’s start using some code. First we have to import the spark python API. 

```
%pyspark
from pyspark.sql.types import *
```

Next let’s import the flights data into a DataFrame and show the first 5 rows. We use header=true to use the header line for naming the columns and specify to infer the schema.  

```
%pyspark
airportsRawDF = spark.read.csv("s3a://flight-bucket/raw-data/airports", 
    	sep=",", inferSchema="true", header="true")
airportsRawDF.show(5)
```

The output will show the header line followed by the 5 data lines.

![Alt Image Text](./images/zeppelin-show-airports-raw.png "Zeppelin Welcome Screen")

Now let’s display the schema, which has been derived from the data:

```	
%pyspark
airportsRawDF.printSchema()
```

You can see that both string as well as double datatypes have been used and that the names of the columns are derived from the header row of the CSV file. 

```
root
 |-- iata: string (nullable = true)
 |-- airport: string (nullable = true)
 |-- city: string (nullable = true)
 |-- state: string (nullable = true)
 |-- country: string (nullable = true)
 |-- lat: double (nullable = true)
 |-- long: double (nullable = true)
``` 
 
Next let’s ask for the total number of rows in the dataset. Should return a total of **3376**. 

```
%pyspark
airportsRawDF.count()
```

You can also transform data easily into another format, just by writing the DataFrame out to a new file or object. 

Let’s create a JSON representation of the data in the refined folder. 
For HDFS:

```
%pyspark
airportsRawDF.write.json("hdfs://namenode:9000/user/hue/refined-data/airports")
```
	
For MinIO:

```
%pyspark
airportsRawDF.write.json("s3a://flight-bucket/refined-data/airports")
```

Check that the file has been written to HDFS or MinIO using either one of the techniques seen before. 

 
## Working with Flights Data

First add another title, this time as a Heading-2.

```
%md ## Working with the Flights data
```

Let’s now start working with the Flights data, which we have uploaded with the various files within the `s3://flight-bucket/raw-data/flights/`.

Navigate to the first cell and start with a title. By using the `%md` directive we can switch to the Markdown interpreter, which can be used for displaying static text.
 
Make sure that the data is in the right place. You can use the directive `%sh` to execute a shell action.

```
%sh
s3cmd ls -r s3://flight-bucket/raw-data/flights/
```

You should see the five files inside the `flights` folder

![Alt Image Text](./images/zeppelin-sh-ls.png "Zeppelin Show Files")

The CSV files in this case do not contain a header line, therefore we cannot use the same technique as before with the airports and derive the schema from the header. 

We first have to manually define a schema. One way is to use a DSL as shown in the next code block. 

```
%pyspark
flightSchema = """`year` INTEGER, `month` INTEGER, `dayOfMonth` INTEGER,  `dayOfWeek` INTEGER, `depTime` INTEGER, `crsDepTime` INTEGER, `arrTime` INTEGER, `crsArrTime` INTEGER, `uniqueCarrier` STRING, `flightNum` STRING, `tailNum` STRING, `actualElapsedTime` INTEGER,
                   `crsElapsedTime` INTEGER, `airTime` INTEGER, `arrDelay` INTEGER,`depDelay` INTEGER,`origin` STRING, `destination` STRING, `distance` INTEGER, `taxiIn` INTEGER, `taxiOut` INTEGER, `cancelled` STRING, `cancellationCode` STRING, `diverted` STRING, 
                   `carrierDelay` STRING, `weatherDelay` STRING, `nasDelay` STRING, `securityDelay` STRING, `lateAircraftDelay` STRING"""
```

Now we can import the flights data into a DataFrame using this schema and show the first 5 rows. 

We use  to use the header line for naming the columns and specify to infer the schema. We specify `schema=fligthSchema` to use the schema from above.  

```
%pyspark
flightsRawDF = spark.read.csv("s3a://flight-bucket/raw-data/flights", 
    	sep=",", inferSchema="false", header="false", schema=flightSchema)
flightsRawDF.show(5)
```
	
The output will show the header line followed by the 5 data lines.

![Alt Image Text](./images/zeppelin-show-flights-raw.png "Zeppelin Welcome Screen")

Let’s also see the schema, which is not very surprising

```	
%pyspark
flightsRawDF.printSchema()
```

The result should be a rather large schema only shown here partially. You can see that both string as well as integer datatypes have been used and that the names of the columns are derived from the header row of the CSV file. 

```
root
 |-- year: integer (nullable = true)
 |-- month: integer (nullable = true)
 |-- dayOfMonth: integer (nullable = true)
 |-- dayOfWeek: integer (nullable = true)
 |-- depTime: integer (nullable = true)
 |-- crsDepTime: integer (nullable = true)
 |-- arrTime: integer (nullable = true)
 |-- crsArrTime: integer (nullable = true)
 |-- uniqueCarrier: string (nullable = true)
 |-- flightNum: string (nullable = true)
 |-- tailNum: string (nullable = true)
 |-- actualElapsedTime: integer (nullable = true)
 |-- crsElapsedTime: integer (nullable = true)
 |-- airTime: integer (nullable = true)
 |-- arrDelay: integer (nullable = true)
 |-- depDelay: integer (nullable = true)
 |-- origin: string (nullable = true)
 |-- destination: string (nullable = true)
 |-- distance: integer (nullable = true)
 |-- taxiIn: integer (nullable = true)
 |-- taxiOut: integer (nullable = true)
 |-- cancelled: string (nullable = true)
 |-- cancellationCode: string (nullable = true)
 |-- diverted: string (nullable = true)
 |-- carrierDelay: string (nullable = true)
 |-- weatherDelay: string (nullable = true)
 |-- nasDelay: string (nullable = true)
 |-- securityDelay: string (nullable = true)
 |-- lateAircraftDelay: string (nullable = true)
```
	
Next let’s ask for the total number of rows in the dataset. Should return **50'000**. 

```
%pyspark
flightsRawDF.count()
```
	
You can also transform data easily into another format, just by writing the DataFrame out to a new file or object. 

Let’s create a Parquet representation of the data in the refined folder. Additionally we partition the data by `year` and `month`. 

For HDFS:

```
%pyspark
flightsRawDF.write.partitionBy("year","month").parquet("hdfs://namenode:9000/user/hue/refined-data/flights")
```
	
For MinIO:

```
%pyspark
flightsRawDF.write.partitionBy("year","month").parquet("s3a://flight-bucket/refined-data/flights")
```

Check that the file has been written to HDFS or MinIO using either one of the techniques seen before. 

For example for the object storage solution, we can add

```
%sh
s3cmd ls -r s3://flight-bucket/refined-data/flights
```	
	
Should you want to execute the write a 2nd time, then you first have to delete the output folder, otherwise the 2nd execution of the write will throw an error. 

You can directly execute the remove from within Zeppelin, using the `%sh` directive. 

For HDFS:

```
%sh
hadoop fs -rm -R hdfs://namenode:9000/user/hue/refined-data/flights
```
	
For MinIO:

```
%sh
s3cmd rm -r s3://flight-bucket/refined-data/flights
```
	
By now we have imported the airports and flights data and made it available as a Data Frame. 

Additionally we have also stored the data to a file in json format. 

## Let’s use some SQL to work with the data

First let's read the data from the parquet refined structure just created before. 

```
%pyspark
flightsRefinedDF = spark.read.format("parquet").load("s3a://flight-bucket/refined-data/flights")
```

With the `flightsRefinedDF` DataFrame in place, register the two DataFrames as temporary tables in Spark SQL

```
%pyspark
flightsRefinedDF.createOrReplaceTempView("flights")
airportsRawDF.createOrReplaceTempView("airports")
```

We can always display the registered tables by using the following statement:

```
%pyspark
spark.sql("show tables").show()
```

We can use `spark.sql()` to now execute an SELECT statement using one of the two tables

```
%pyspark
spark.sql("SELECT * FROM airports").show()
```

But in Zeppelin, testing such a statement is even easier. You can use the `%sql` directive to directly perform an SQL statement without having to wrap it in a `spark.sql()` statement. This simplifies ad-hoc testing quite a bit. 

```
%sql
SELECT * 
FROM airports
```

Let's see a `GROUP BY` in action

```
%sql
SELECT country, state, count(*)
FROM airports
GROUP BY country, state
```

If we only want to see the ones for the USA, we add a `WHERE` clause

```
%sql
SELECT country, state, count(*)
FROM airports
WHERE country = 'USA'
GROUP BY country, state
```

Once you are ready, you can wrap it in a `spark.sql()` using the convenient tripe double quotes. 

```
%pyspark
usAirportsByStateDF = spark.sql("""
        SELECT country, state, count(*)
        FROM airports
        WHERE country = 'USA'
        GROUP BY country, state
          """)
usAirportsByStateDF.show()
```


If you perform a SELECT on the flights table using one or more of the partition columns, the query will prune the non-used partitions and only read the necessary files for the needed partitions

```
%sql
SELECT * 
FROM flights
WHERE year = 2008 
AND month = 04
```

## Use Spark SQL to perform some analytics on the data

Let's see the the 10 longest flights in descending order with `origin` and `destination`

```
%sql
SELECT distance, origin, destination 
FROM flights 
WHERE distance > 1000 
ORDER BY distance DESC
```

```
%sql
SELECT arrDelay, origin, destination,
    CASE
         WHEN arrDelay > 360 THEN 'Very Long Delays'
         WHEN arrDelay > 120 AND arrDelay < 360 THEN 'Long Delays'
         WHEN arrDelay > 60 AND arrDelay < 120 THEN 'Short Delays'
         WHEN arrDelay > 0 and arrDelay < 60 THEN 'Tolerable Delays'
         WHEN arrDelay = 0 THEN 'No Delays'
         ELSE 'Early'
    END AS flight_delays
         FROM flights
         ORDER BY arrDelay DESC
```

## Use Spark SQL to join flights with airports

Last but not least let's use the `airports` table to enrich the values returned by the `flights` table so we have more information on the origin and destination airport. 

If we know SQL, we know that this can be done using a JOIN between two tables. The same syntax is also valid in Spark SQL. Following the techniques learned above, let's first test it using the handy %sql directive. 

```
%sql
SELECT ao.airport AS origin_airport
		, ao.city AS origin_city
		, ad.airport AS desitination_airport
		, ad.city AS destination_city
		, f.*
FROM flights  AS f
LEFT JOIN airports AS ao
ON (f.origin = ao.iata)
LEFT JOIN airports AS ad
ON (f.destination = ad.iata)
```

As soon as we are happy, we can again wrap it in a `spark.sql()` statement. 

```
%pyspark
flightEnrichedDF = spark.sql("""
		SELECT ao.airport AS origin_airport
				, ao.city AS origin_city
				, ad.airport AS desitination_airport
				, ad.city AS destination_city
				, f.*
		FROM flights  AS f
		LEFT JOIN airports AS ao
		ON (f.origin = ao.iata)
		LEFT JOIN airports AS ad
		ON (f.destination = ad.iata)
		""")
```

Let's see the result behind the DataFrame

```
%pyspark
flightEnrichedDF.show()
```

Finally let's write the enriched structure as a result to object storage using again the Parquet format:

```
%pyspark
flightEnrichedDF.write.partitionBy("year","month").parquet("s3a://flight-bucket/result-data/flights")
```
