---
title: "Exploring Azure IoT Hub with an Azure IoT DevKit"
date: "2019-09-05"
categories: 
  - "azure"
  - "iot-hub"
tags: 
  - "azure"
  - "blob-storage"
  - "databricks"
  - "event-hubs"
  - "iot"
  - "iot-dev-kit"
  - "iot-hub"
  - "powerbi"
  - "stream-analytics"
---

![](images/download.png)

IoT Hub

Recently, I've been working to extract data from sensors and get it pumping into Azure. One main thoroughfare for ingesting IoT data into Azure is via IoT Hub. In order to increase my familiarity with IoT Hub and IoT data in Azure, I purchased an [IoT Developer Kit](https://microsoft.github.io/azure-iot-developer-kit/) to get my hands dirty.

In this post, I'll briefly describe my experience using the IoT Developer Kit and the resources I used within Azure to make sense of the IoT data.

![](images/az-iot-devkit-1024x768.jpg)

The AZ3166 IoT Developer Kit

The IoT DevKit includes a credit card-sized chip with built-in WiFi and a micro-USB cable for power. Out of the box, it comes with a temperature and humidity sensor. Because I was mostly focused on making this an educational endeavor, I decided to set up my IoT device in my bedroom to continually monitor the temperature and humidity.

After flashing firmware onto the IoT DevKit and providing the WiFi password to the device (following [this How-to guide](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-arduino-iot-devkit-az3166-get-started)), I was able to add this device to my IoT Hub instance. Then, I set up custom endpoints and routes depending on where I wanted the data to land.

First, I set up a two different custom endpoints: an Event Hubs endpoint (for streaming data into Azure Databricks) and a Blob storage account (for archiving historical data).

![](images/iot-hub-endpoints-2019-09-05-172624-1024x458.png)

Custom endpoints in Azure IoT Hub

Then, I set up three different routes: one route for sending data to the Event Hubs endpoint, another for sending data to the Blob storage account endpoint and another to the default events endpoint (which is required for using Stream Analytics).

![](images/iot-hub-routes-2019-09-05-172624-1024x279.png)

Routes in Azure IoT Hub

Now with endpoints and routes configured, data was being ingested into Azure. IoT Hub enables you to provision devices, send data into Azure and set up routes for the data to travel, eventually reaching an endpoint. Here, I am using basic functionality of IoT Hub but it can be used as a bi-directional communication tool between the cloud and millions of devices.

My plan was threefold:

- Stash all of the telemetry messages in cold Blob storage for archival and historical analysis purposes.
- Send all of the telemetry messages to Event Hubs for further processing in Azure Databricks.
- Pass only telemetry messages through Stream Analytics that contained both a temperature and humidity reading.

## **Blob Storage**

When sending the data to Blob storage from IoT Hub, you can specify the naming convention to better organize your information. I have a specific container in my Blob storage account called `iot-data` dedicated to storing this data. The virtual directory specifies a file path named by partition, year, month, day, hour and minute.

![](images/iot-data-2019-09-06-133706-1024x476.png)

Storage Explorer showing a virtual directory of telemetry messages from the IoT DevKit

In the above example, I'm using Storage Explorer to show files stored in partition 2, on September 4th, 2019 from 4AM. There are 60 files, with a handful of messages being stored in each file every minute.

I found a few things frustrating when I set up this route in IoT Hub. First, the data is stored with Content Type set to `application/octet-stream`. Second, the body of the telemetry message is encoded in `base64`. And lastly, there are some messages which only contain a temperature reading with no humidity reading and vice-versa.

## Stream Analytics

In IoT Hub, the [default built-in endpoint](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-d2c#routing-endpoints) sends all data to **messages/events**, which is compatible with Event Hubs. But, when you add a custom endpoint, this default behavior stops. Therefore, because I needed data flowing to the default **messages/events** for Stream Analytics to consume the data, I added a route to the **messages/events** endpoint.

Now, with data flowing to the default **messages/events** endpoint, I created a pretty straightforward query to calculate the temperature in Fahrenheit from Celsius and to filter out records that are missing either a humidity or temperature reading. In other words, I only wanted messages flowing through that had both a temperature and humidity reading.

![](images/stream-analytics-2019-09-06-135448-1024x199.png)

Stream Analytics query to calculate Fahrenheit temperature and filter missing records

Armed with cleaner and simpler data passing through Stream Analytics, I sent the data to a PowerBI output for visualizing in a dashboard. From PowerBI with just a few clicks, I had a dashboard that showed the most recent temperature and humidity reading, as well as trend lines that show the temperature and humidity over the past 60 minutes.

![](images/live-tiles-powerbi-2019-09-06-140533-1024x537.png)

My IoT DevKit Dashboard with live tiles of temperature and humidity

## Event Hubs

This last leg of the race passed telemetry messages from the devices through to an Event Hubs instance, with the goal of analyzing the data in real-time using PySpark in Databricks.

In Databricks, I imported the Azure Event Hubs library from the Maven coordinate located here: `com.microsoft.azure:azure-eventhubs-spark_2.11:2.3.6`. Then I created a schema for the streaming data, read it into a data frame and did the same calculation and filtering step as above in Stream Analytics: converting Celsius to Fahrenheit and filtering for records that had both a temperature and humidity reading.

![](images/databricks-stream-2019-09-06-143337-1024x659.png)

PySpark query in Databricks to convert temperature to Fahrenheit and filter null records

Ultimately, after understanding how to query the data from Event Hubs in Databricks, I generated a batch Notebook job in Databricks to pull all of the records written to Blob storage for the previous day, applied the Fahrenheit-to-Celsius conversion and null record filter and summarized the records for the entire day into one CSV file stored back into Blob storage.

![](images/batch-iot-2019-09-06-144043-1024x497.png)

Sample run output from the batch Notebook job in Databricks

## Summary

In this post, I shared my learning experience ingesting IoT data into Azure using the Azure IoT Developer Kit. There were a ton of different Azure resources involved: IoT Hub, Event Hubs, Blob storage, Stream Analytics, Databricks and PowerBI. It was a very interesting project and was really cool to see live data streaming off of a device.

These are my major takeaways:

- It was really easy to plug in the IoT DevKit and start seeing data in Azure. This device is a great way to plug-and-play and learn!
- The diversity and flexibility of IoT Hub is fantastic. It's a great entry point into Azure and there are a lot of built-in options to send data to various Azure resources that can be set up in just a few clicks.
- I need to research how to set up certain system properties on IoT devices. That way, I can specify that the Content Type is `application/json` and change the encoding to `UTF-8`.
