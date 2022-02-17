---
title: "Streaming Data in Azure"
date: "2020-06-11"
tags: 
  - "azure"
  - "azure-sql-db"
  - "blob-storage"
  - "databricks"
  - "iot"
  - "iot-hub"
  - "powerbi"
  - "stream-analytics"
---

![](images/stream-architecture-2-1024x703.png)

Streaming data is continuously generated data from a variety of sources. Think of these data points, real-time events or telemetry as a continuously flowing stream. Numerous devices, sensors, applications or server logs are all capable of generating data that can be consumed in real-time. In this post, I'll walk through a solution accelerator that can get your data streaming into Azure in no (real-) time!

Recently, my colleagues and I built an end-to-end streaming demonstration. After the customer engagement concluded, we decided to roll up our sleeves and make the solution re-usable, minimizing the time to value when discussing streaming use cases with other customers. Huge shout out to [Howard Ginsburg](https://www.linkedin.com/in/howardginsburg/) and [Steve Flowers](https://www.linkedin.com/in/steven-flowers/) for diving in and collaborating on this!

When it comes to streaming data in Azure, there are lots of options, but ultimately the way to go is the [Azure IoT Reference Architecture](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/iot) from the [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/). Here you will find best practices and tried-and-true solution frameworks that have been battle-tested from numerous customer implementations.

To deploy this solution in your Azure subscription today, head over to the [streaming-demo repository on GitHub.](https://github.com/GLRAzure/streaming-demo) In this repository, you will find these key resources to get you up-and-running:

- [ARM template](https://github.com/GLRAzure/streaming-demo/blob/master/deploy-template/DeployStreamingTemplate.json) to deploy the components of this solution
- [.NET core application](https://github.com/GLRAzure/streaming-demo/tree/master/IoTDeviceSimulator) that acts as a device simulator
- [Documentation](https://github.com/GLRAzure/streaming-demo/blob/master/README.md) for configuring the deployed resources

Below is a slightly modified version of the Azure IoT Reference Architecture that diagram the specific resources provisioned using the provided ARM template and how they interact.

![](images/stream-architecture-1-1024x703.png)

Now that we have a working diagram, let's talk a bit more about the capabilities these resources provide.

## Ingesting Data

If you don't have devices that are readily available and already streaming data, we have provided a basic .NET core application that simulates devices streaming data. Device connection strings (provided by IoT Hub) are used as parameters for the device simulator. Once this application is running, data will be sent from multiple, simulated devices into Azure IoT Hub.

## Routing Data

From within IoT Hub, you have a [number of options](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-d2c) to route your messages to other services. Think of IoT Hub like a traffic guard; when a message is ingested via IoT Hub, there are rules (or routes) that the telemetry message can follow. In this implementation, we establish two routes: (1) to a custom storage endpoint (Blob storage) and (2) to the default events endpoint. This second route leverages [the built-in Event Hubs compatible endpoint](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-read-builtin) that we can consume from two other components in our architecture: Azure Stream Analytics and Azure Databricks.

_A quick note on storage:_ In some cases, saving off the streaming data to "cold storage" for use in downstream data pipelines is invaluable. Typically this is the case when there is a dual need to extract insights from data in near-real-time and also extract insights from the data using a wider time frame.

A great example is in manufacturing: there is a need to visualize the performance and quality of manufacturing machines throughout the day in near-real-time. As an added benefit, saving the data permanently provides data scientists with a treasure trove of data to train machine learning algorithms to assist with predictive maintenance and quality assurances.

## Analyzing & Visualizing Data

When dealing with streaming data, time is of the essence. There are two fantastic services in Azure that provide us with the ability to quickly manipulate, cleanse, transform and aggregate streaming data: [Azure Databricks](https://docs.microsoft.com/en-us/azure/databricks/getting-started/spark/streaming) and [Azure Stream Analytics](https://docs.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction). These services are sourced with streaming data via the Event Hubs-compatible endpoint provided by IoT Hub.

In Azure Stream Analytics, we specify an input, a query and an output. For those familiar with SQL syntax, this should feel very similar. Here is a screenshot of the query that allows us to stream data directly to a Power BI dataset and to an Azure SQL Database for aggregated reporting.

![](images/stream-analytics.png)

The beauty of the Azure Stream Analytics service is in its simplicity. The power of Power BI shines when combining both our live, real-time stream of data (line chart and bar chart) and a constantly refreshed aggregate of data (data table for historical averages over five minutes) from the Azure SQL Database. Below, you can see both the real-time data and aggregated data in one report!

![](images/powerbi-dashboard.png)

In Azure Databricks, Structured Streaming allows us to ingest data from the Event Hubs-compatible endpoint in IoT Hub. All the power, customization, complexity and scalability of Databricks is available to analyze your streaming data. Those who have experience with Big Data technologies like Spark and prefer a code-first experience, Databricks is the perfect environment for handling streaming data. Here is a code snippet that shows how to ingest data from Event Hubs.

![](images/stream-databricks.png)

Although not quite as robust as the feature set in Power BI, Azure Databricks provides some visualization capabilities (that not all are aware of). Below is a screenshot of a similar dashboard to Power BI.

![](images/databricks-dashboard.png)

## Summary

In this post, I discussed a solution accelerator for streaming data use cases. In Azure, there are a variety of services and technologies that help enable the extraction of insights from data. The major capabilities provided in this solution accelerator are ingesting, routing, storing, analyzing and visualizing real-time telemetry.
