---
title: "Securing Azure Data Services using a VNET and Private Endpoints"
date: "2020-03-24"
tags: 
  - "azure"
  - "blob-storage"
  - "data-factory"
  - "self-hosted-integration-runtime"
  - "sql-server"
  - "virtual-machine"
  - "virtual-newtwork"
  - "vm"
  - "vnet"
---

![](images/vnet.png)

Recently, I've been working with my customers who are leveraging Microsoft Azure to build out data processing pipelines in Azure Data Factory. Often, the resources that Data Factory requires access to are deployed to a private endpoint in a Virtual Network (VNET). In this article, I'm going to explain how to permit Data Factory access to "see" these private endpoints within your VNET.

Fair warning: this will be a bit of a longer blog post as there is a lot of content to cover. Also, this article assumes some fundamental knowledge of working with resources in Azure - it does not go step-by-step. Rather, I link to step-by-step documentation and highlight important deployment configuration steps.

When deploying production environments and workloads in Azure, it is hard to get around dealing with the complexities of a Virtual Network. In many customer scenarios, we build a "quick-and-dirty" proof-of-concept with test data that does not have any confidentiality implications. Therefore, security often gets put in the parking lot for a later conversation.

We're going to focus on that often parked item in this blog post. When customers get serious about leveraging Azure services, they set up a VNET with [ExpressRoute](https://azure.microsoft.com/en-us/services/expressroute/). This connects their on-premises data centers to Azure, allowing for a faster, private connection to Azure data centers.

For the scenario I'm going to discuss in this blog post, there are a handful of major components to focus on:

- Azure Virtual Network (VNET)
- Azure Virtual Machine (VM)
- Azure Data Factory (ADF)
- Azure SQL
- Storage account

We will deploy the VM, the Azure SQL Server and the storage account to the VNET using private endpoints within that VNET. Here is a (not-so scientific) diagram of these components interacting:

![](images/vnet-private-endpoints-1024x709.png)

An important note here is that in order for Azure Data Factory to "see" the Azure Data Services that were provisioned into the VNET with private endpoints, the self-hosted integration runtime (IR) must be running on a VM within the same VNET.

## Azure Virtual Network (VNET)

First, we'll start off by creating a new VNET in Azure. You can follow this [Quickstart found on the Microsoft documentation](https://docs.microsoft.com/en-us/azure/virtual-network/quick-create-portal). For my example, the IPv4 address space will be: 55.1.0.0/16 and will contain one Subnet address range of: 55.1.0.0/24. With our VNET deployed, let's start adding resources to it.

## Azure Virtual Machine (VM)

To deploy an Azure VM to a VNET, you can follow the same Quickstart linked above. Don't worry about creating two VMs, we will only need one. Ensure that the VM you deploy will land in the VNET you created initially. It will be given its own IP address within the Subnet you created. We will allow specific ports to be open (HTTP and RDP) to permit communication with the VM.

![](images/create-vm-1-1024x718.png)

After waiting for the VM to provision, RDP into the VM to ensure you can access it. We'll be installing the following software to make it easier to work with the private endpoints we provision next. Install the following software:

- [Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/)
- [SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15)

## Storage Account

Next, we'll provision a blob storage account by following the [How to guide on the Microsoft documentation](https://docs.microsoft.com/en-us/azure/private-link/create-private-endpoint-storage-portal). We've already created the VNET and VM, so just focus on provisioning the storage account. The key information to focus on is the Networking tab of the storage account deployment. Ensure that you add a new private endpoint within the VNET you created earlier.

![](images/create-storage-account-1024x434.png)

## Azure SQL Server

Now we will provision a SQL Server to the same VNET we've been working with by following this [Quickstart on the Microsoft documentation](https://docs.microsoft.com/en-us/azure/private-link/create-private-endpoint-portal). As above, we really only need to focus on the section of the Quickstart that deploys the SQL Server.

In the linked Quickstart above, the documentation has you provision the SQL Server and then add a private endpoint after the fact. If you'd like to do it in one step, you have the option to deploy it with a private endpoint in the Networking tab, similar to the Storage account configuration.

![](images/create-sql-server-1024x433.png)

## Azure Data Factory

Following this [Quickstart, you can provision a Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-portal). Again, you really only need to focus on the piece that provisions the Data Factory service itself. We have already created the storage account we plan to use.

At this point, we have provisioned all the Azure resources we need. Next, after you RDP into your VM, sign into the Azure portal (this will expedite the setup of the self-hosted IR) and then open up the Author & Monitor experience for Azure Data Factory. You can follow this [walkthrough for more detail on creating the self-hosted IR](https://docs.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime).

Because we opened the Data Factory workspace on the VM that will host the integration runtime, we can select Option 1: Express setup. This will download the self-hosted IR software on the VM and link it to the Azure Data Factory resource we just provisioned.

![](images/create-ir-1024x916.png)

At this point, everything is set up for us to access the storage account and SQL Server by using their private endpoints. To show that you can't access these Azure services from outside the VNET, I'll post a series of screenshots below comparing the response you get from within the VNET and from outside the VNET.

Here are the expected responses from a machine **outside** the VNET:

![](images/outside-vnet.png)

And the expected responses from the VM that is **within** the VNET:

![](images/inside-vnet.png)

As you can see, the response from within the VNET routes the request to the appropriate private endpoint within the Subnet of the VNET.

The really cool aspect about this is that you can leverage these private endpoints how you normally do within Azure Data Factory. In other words, this routing happens behind the scenes and we can still use the default connection strings for the Azure SQL Server and storage account.

## Creating sample data in SQL Server

For the purposes of this demonstration, we'll create a very simple table in the SQL Server we created (this is where having SSMS installed on the VM helps). You can use the following two scripts to create a table and insert two sample records. This assumes you have a database created called 'mydatabase'.

First, here is the script to create a table:

SET ANSI\_NULLS ON
GO

SET QUOTED\_IDENTIFIER ON
GO

CREATE TABLE \[dbo\].\[myTable\](
	\[c1\] \[nchar\](10) NULL,
	\[c2\] \[nchar\](10) NULL
) ON \[PRIMARY\]
GO

And to populate two records in the table:

USE \[mydatabase\]
GO

INSERT INTO \[dbo\].\[myTable\]
           (\[c1\]
           ,\[c2\])
     VALUES
           ('hi', 'there'),
           ('hello', 'world')
GO

We will copy the data in this Azure SQL Server to the storage account we created previously.

## Copy activity in Data Factory

In the Author & Monitor workspace in Data Factory, drag a Copy data activity onto the canvas. First, we'll define the Source and then the Sink for this activity. You'll notice that we specify the self-hosted integration runtime for both of these linked services. This is a critical step to allow the Data Factory runtime to "see" the two private endpoints within our VNET.

For the Source, we will create a linked service to the Azure SQL Database:

![](images/sql-linked-service-874x1024.png)

As you can see, we really don't specify anything special to tell Data Factory to leverage the private endpoint. Similarly for the Sink, we create a linked service to the storage account:

![](images/blob-linked-service-1-937x1024.png)

Once these linked services are set up, we can Validate and Publish our changes to this pipeline and run it. For completeness, here is a screenshot of the data in SQL Server:

![](images/hello-world.png)

And the copied data into our storage account:

![](images/blob-data-1024x642.png)

## Summary

We just successfully ran a Data Factory pipeline that copies data from an Azure SQL Database private endpoint to a blob storage account private endpoint, all secured in a VNET!
