---
title: "Using Databricks to Extract JSON Schema"
date: "2019-09-10"
categories: 
  - "azure"
  - "azure-databricks"
tags: 
  - "azure"
  - "databricks"
  - "dbfs"
  - "json"
  - "pyspark"
---

![](images/databricks.png)

Databricks is great for leveraging Spark in Azure for many different data types. One challenge I've encountered when using JSON data is manually coding a complex schema to query nested data in Databricks.

In this post, I'll walk through how to use Databricks to do the hard work for you. By leveraging a small sample of data and the Databricks File System (DBFS), you can automatically infer the JSON schema, modify the schema and apply the modified schema to the rest of your data.

_If you'd rather just see the code, [here is a link](https://github.com/stevedem/DatabricksJSON.git) to the DBC archive file._

Here is a nicely formatted, single record of JSON data that was produced by my Azure IoT DevKit (see my previous [post about it here](http://www.stevedem.com/exploring-azure-iot-hub-with-an-azure-iot-devkit/)). When I set up my IoT DevKit in Azure, I configured a route in IoT Hub to store all incoming messages as JSON files in Blob storage. We'll use this record as a sample throughout this post.

{
  "EnqueuedTimeUtc": "2019-09-05T04:06:23.4350000Z",
  "Properties": {
    "temperatureAlert": "false"
  },
  "SystemProperties": {
    "connectionDeviceId": "MyNodeDevice",
    "connectionAuthMethod": {
      "scope": "device",
      "type": "sas",
      "issuer": "iothub",
      "acceptingIpFilterRule": null
    },
    "connectionDeviceGenerationId": "637014173950564514",
    "enqueuedTime": "2019-09-05T04:06:23.4350000Z"
  },
  "Body": "ewogICAgIm1lc3NhZ2VJZCI6IDMxNDIyLAogICAgInRlbXBlcmF0dXJlIjogMTkuMjAwMDAwNzYyOTM5NDUzLAogICAgImh1bWlkaXR5IjogNDUuNTk5OTk4NDc0MTIxMDk0Cn0="
}

This sample record is fairly straightforward on the surface but useful for understanding the schema definition required for Databricks to parse fields from the JSON structure. My reasoning for messing with this JSON schema in the first place was because Databricks incorrectly inferred the data type of the "Body" field.

In the sample above, you can see that it's just a garbled string. In reality, it's an extension of the JSON schema encoded in base64. Databricks guessed that it was simply a StringType field, but it should be read in as a BinaryType field.

The workflow used to extract, modify and apply the JSON schema from this sample follows these steps:

1. Generate a sample JSON record from the source data
2. Upload the sample JSON record to the Databricks File System (DBFS)
3. Read the JSON file from DBFS (with inferred schema)
4. Print and alter the JSON schema
5. Read the JSON file from DBFS (with the modified schema)
6. Extract the relevant fields

## Generate a sample JSON record from the source data

To begin, extract one record from your source data. Ensure that all fields are captured in this sample, because we will use this as a model for additional records.

Tip: After extracting one record, use [a tool like this one](https://codebeautify.org/jsonviewer) to validate your JSON record and "minify" it to remove all white space.

Once you have the minify'd version of your JSON record, store it into a string variable. Here, I'm referring to this string variable as json\_sample.

json\_sample = '{ JSON RECORD HERE }'

## Upload the sample JSON record to the Databricks File System (DBFS)

Using our previously generated string, we'll place the contents of the string into a file in our DBFS. We do this by leveraging the put function from the Databricks file system utilities.

dbfs\_file\_path = 'file:/dbfs/tmp/sample.json'
dbutils.fs.put(dbfs\_file\_path, 
  json\_str, 
  overwrite=True)

## Read the JSON file from DBFS (with inferred schema)

Then, we'll use the default JSON reader from PySpark to read in our JSON file stored in the DBFS and to automatically infer the schema. Inferring the schema is the default behavior of the JSON reader, which is why I'm not explicitly stating to infer the schema below.

df = spark\\
  .read\\
  .json(dbfs\_file\_path)

Now that we have our sample data loaded from the DBFS, we can alter the JSON schema to label the "Body" field as a BinaryType instead of a StringType.

## Print and alter the JSON schema

To see what the schema looks like, we'll extract the inferred schema and store it as a variable to get a better look at what we're working with.

We can see that by just printing the inferred schema, we get a whole mess of PySpark SQL Types (StructField, StructType, StringType, etc.) By extracting the JSON value of this schema and then converting the JSON object to a string, we can print out the schema and modify the output to help us ingest the "Body" field as a BinaryType instead of a StringType.

jsonSchema = df.schema
print("\\nInferred schema:\\n")
print(jsonSchema)

print("\\nJSON dumps of schema:\\n")
print(json.dumps(jsonSchema.jsonValue()))

![](images/json-schema-output-2019-09-10-111633-1-1024x414.png)

Now, if we simply copy the output of our JSON schema (in the second chunk of output), we can modify it to have a binary type instead of a string type.

jsonSchema = '{"type": "struct", "fields": \[{"type": "binary", "nullable": true, "name": "Body", "metadata": {}}, {"type": "string", "nullable": true, "name": "EnqueuedTimeUtc", "metadata": {}}, {"type": {"type": "struct", "fields": \[{"type": "string", "nullable": true, "name": "temperatureAlert", "metadata": {}}\]}, "nullable": true, "name": "Properties", "metadata": {}}, {"type": {"type": "struct", "fields": \[{"type": "string", "nullable": true, "name": "connectionAuthMethod", "metadata": {}}, {"type": "string", "nullable": true, "name": "connectionDeviceGenerationId", "metadata": {}}, {"type": "string", "nullable": true, "name": "connectionDeviceId", "metadata": {}}, {"type": "string", "nullable": true, "name": "enqueuedTime", "metadata": {}}\]}, "nullable": true, "name": "SystemProperties", "metadata": {}}\]}'
jsonSchema = StructType.fromJson(json.loads(jsonSchema))
print("\\nModified schema:\\n")
print(jsonSchema)

![](images/json-schema-output-2-2019-09-10-111633-1024x133.png)

This feels a bit hacky, but if you stick with me, I'll show you that it works! At this point, we've taken the inferred schema, modified it, and created a new schema by leveraging the output of the inferred schema.

## Read the JSON file from DBFS (with the modified schema)

To usethis modified schema, we simply pass the jsonSchema variable as a parameter to .schema while reading in our JSON sample file.

df = spark\\
  .read\\
  .schema(jsonSchema)\\
  .json(dbfs\_file\_path)
print(df.schema)

![](images/json-schema-output-3-2019-09-10-111633-1024x293.png)

Here's confirmation that our modification worked. It now defines the data type of the "Body" field as binary.

## Extract the relevant fields

To understand what's contained in the base64 binary field, we need to cast it as a string. In this code block, I'll also parse the timestamp from our EnqueuedTimeUtc column and then select only the body and date column to move forward.

decoded\_df = df\\
  .withColumn("body", df\["Body"\].cast("string"))\\
  .withColumn("date", from\_utc\_timestamp(df\["EnqueuedTimeUtc"\], 'UTC'))\\
  .select("body", "date")
display(decoded\_df)

![](images/df-display-2019-09-10-111633-1024x211.png)

Surprise! Within the base64 field was actually more JSON. No problem, we can manually create a very simple StructType to parse this field with. I'll also convert the temperature from Celsius to Fahrenheit. The StructType we'll define has three fields: messageId, temperature and humidity.

For all intents and purposes, we're interested in the date (time stamp), the temperature and the humidity. We'll only retain these columns for our analysis up to this point. In the below code block, we manually construct a JSON schema using the StructType and apply that schema to an individual column in our data frame using the from\_json function.

schema = StructType(\[
  StructField("messageId", StringType()), 
  StructField("temperature", DoubleType()),
  StructField("humidity", DoubleType())
\])
final\_df = decoded\_df\\
  .select(from\_json("body", schema).alias("bodyJson"), "date")\\
  .select(
    ((col("bodyJson.temperature") \* 1.8) + 32).alias("temperature"),
    col("bodyJson.humidity").alias("humidity"),
    col("date")
  )
display(final\_df)

![](images/df-display-2-2019-09-10-111633-1024x94.png)

We now have the three columns that we're interested in and can test this workflow on a larger subset of data.

## Summary

To recap, we inferred, modified and applied a JSON schema using the built-in .json reader from PySpark. We used the DBFS to store a temporary sample record for teasing out the JSON schema of our source data. This workflow can be useful because it allows us to quickly generate and modify a complex JSON schema. This is also especially useful if you have streaming data coming into Databricks and need a way to experiment with your JSON schema of the incoming messages without interfering with the stream.
