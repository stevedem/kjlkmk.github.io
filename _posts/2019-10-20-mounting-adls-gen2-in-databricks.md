---
title: "Mounting ADLS Gen2 in Databricks"
date: "2019-10-20"
categories: 
  - "azure"
  - "azure-databricks"
tags: 
  - "adls"
  - "azure"
  - "azure-ad"
  - "databricks"
  - "key-vault"
---

![](images/Azure-DataLake-icon.png)

Azure Data Lake Store Gen2

In modern data estates, many companies store their data centrally in a cloud-based data lake and leverage distributed computing to analyze, combine and produce data from different sources. In Azure, Databricks makes it easy to leverage distributed computing with Spark. Azure Data Lake Store Gen2 (ADLS) is the go-to resource for an enterprise grade data lake.

In this post, I walk through the steps for mounting your ADLS Gen2 storage account in Databricks, with keys stored and backed by Azure Key Vault.

Mounting data stores in Databricks permits you to seamlessly access data without requiring credentials with every request. Generally speaking, there are a few steps:

1. Create an Azure Key Vault-backed Secret Scope in Databricks
2. Provision an Azure Active Directory App Registration
3. Mount the storage in Databricks

## Create an Azure Key Vault-backed secret scope in Databricks

Secret scopes in Databricks allow you to manage the keys that authorize Databricks to reach out and talk to other resources in your cloud environment (e.g., a SQL Server connection string, a Blob storage account SAS token, etc.). You have two options when creating a Secret Scope in Databricks: Azure Key Vault-backed or Databricks-backed. For the purposes of this exercise, we'll use Azure Key Vault as our storage mechanism for keys.

For a step-by-step walk-through of creating the Secret Scope, visit the [Databricks documentation here](https://docs.azuredatabricks.net/security/secrets/secret-scopes.html#create-an-azure-key-vault-backed-secret-scope).

**Tip:** This feature is in **Public Preview** and therefore only allows you to create the Secret Scope through the web-based user interface. If you want to view the Secret Scopes already associated with your Databricks workspace, you can use the [Databricks CLI](https://docs.azuredatabricks.net/dev-tools/databricks-cli.html#databricks-cli).

## Provision an Azure Active Directory App Registration

Next, we'll create the App registration that we'll use to authenticate against our Azure subscription. Think of this as a service account specifically for Databricks to use while accessing other Azure resources. It will have a Client ID and corresponding Client Secret.

There will be a few IDs that we need to collect in this step: Application (client) ID, client secret and Directory (tenant) ID.

![](images/app-reg-in-ad-1024x797.png)

1. Navigate to your Azure Active Directory resource
2. Under Manage, select App registrations
3. Select + New Registration and give the App registration a name
4. Once the App registration is completed (should only take a few seconds), select Certificates & secrets under Manage
5. Click + New client secret and copy the secret that is generated to a notepad (we'll use it in the next series of steps)
6. In the Overview tab of your App registration, copy the **Application (client) ID** (we'll use it later)
7. In the same Overview tab, copy the **Directory (tenant) ID** (we'll also use this later)

Because we don't want this Client Secret to be visible in our Databricks Notebooks, we store it in our Azure Key Vault. Follow these next few steps to store our App registration's client secret in our Key Vault.

![](images/create-secret-in-kv.png)

1. Navigate to your Azure Key Vault instance
2. Under Settings, click Secrets
3. Click + Generate/Import
4. Give your secret a recognizable name
5. Copy your **secret name** (we'll need it later when mounting our ADLS Gen2 storage in Databricks)
6. Paste in the secret from the previous steps
7. Ensure the Secret is toggled to Enabled
8. Click Create

Now, we need to give this App registration access to our ADLS Gen2 resource.

![](images/add-role-adlsg2.png)

1. Navigate to your ADLS Gen2 storage account
2. Select Access Control (IAM)
3. Click + Add
4. Use the role "Storage Blob Data Contributor" (Note: you may need [other role assignments](https://docs.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal) depending on what functionality you require)
5. In the Select text box, type in the name of the App registration we created in Azure AD and select it from the results
6. Click Save

In this section, we created an Azure AD App registration, saved our client secret to Azure Key Vault and gave our App registration Storage Blob Data Contributor access to our ADLS Gen2 storage account.

## Mount the storage in Databricks

At this point, we have all the pieces in place to mount our ADLS Gen2 storage account in a Databricks notebook. There are a few lines of configuration that I've taken from the [Databricks documentation](https://docs.databricks.com/data/data-sources/azure/azure-datalake-gen2.html#mount-adls-filesystem).

configs = {"fs.azure.account.auth.type": "OAuth",
           "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
           "fs.azure.account.oauth2.client.id": "<client-id>",
           "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope = "<scope-name>", key = "<key-name-for-service-credential>"),
           "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/<directory-id>/oauth2/token"}

1. In the code snippet above, be sure to fill in the **Application (client) ID** for our App registration that we created earlier.
2. Fill in the client secret section by inserting the **Secret Scope name** we created in the first series of steps and the **name of the secret** we used in Azure Key Vault.
3. Fill in the **Directory (tenant) ID** in the last line of the configuration.

Using the Databricks File System, we use the configuration we filled out above to mount the ADLS storage account. Fill in the file system or container name along with the storage account name in the source field below. Also remember to give your mount point a name.

dbutils.fs.mount(
  source = "abfss://<file-system-name>@<storage-account-name>.dfs.core.windows.net/",
  mount\_point = "/mnt/<mount-name>",
  extra\_configs = configs)

And that's it! To read data based from this mount point, use a path like you normally would from the mount point (sample below).

df = spark.read\\
  .csv('/mnt/my-mount/folder1/climbing\_statistics.csv')

## Summary

In this post, we walked through mounting an ADLS Gen2 storage account in Databricks. We covered creating an Azure Key Vault-backed Secret Scope in Databricks, provisioning an App registration in Azure AD and leveraging Key Vault to store our App registration's secret.
