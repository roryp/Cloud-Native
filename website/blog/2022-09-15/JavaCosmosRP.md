---
slug: 
title: 
authors: [rory]
draft: false
hide_table_of_contents: false
toc_min_heading_level: 2
toc_max_heading_level: 3
keywords: [azure, functions, serverless, concepts]
image: ./img/banner.png
description: "Create a function in Java with an Event Hub trigger and an Azure Cosmos DB output binding." 
tags: [serverless-september, azure-functions, java, serverless]
---

<head>
  <meta name="twitter:url"
    content="https://azure.github.io/Cloud-Native/blog/04-functions-java" />
  <meta name="twitter:title"
    content="Azure Functions: For The Java Developer" />
  <meta name="twitter:description"
    content="#30DaysOfServerless: Azure Functions For The Java Developer" />
  <meta name="twitter:image"
    content="https://azure.github.io/Cloud-Native/assets/images/post-kickoff-4a04995b44f0cc4a784fb4ab5e29cf7c.png" />
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:creator"
    content="@nitya" />
  <meta name="twitter:site" content="@AzureAdvocates" /> 
  <link rel="canonical"
    href="https://azure.github.io/Cloud-Native/blog/04-functions-java" />
</head>

---

Welcome to `Day 15`

---

## What We'll Cover

* Create and configure Azure resources using the Azure CLI.
* Create and test Java functions that interact with these resources.
* Deploy your functions to Azure and monitor them with Application Insights

![](./img/banner.png)

---

## Developer Guidance

If you're a Java developer new to serverless on Azure, start by exploring the [Azure Functions Java Developer Guide](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption). It covers:
 * Quickstarts with [Visual Studio Code](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-java) and [Azure CLI](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-cli-java?tabs=bash%2Cazure-cli%2Cbrowser)
 * Building with Maven-based tooling for [Gradle](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-java-gradle), [Eclipse](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-maven-eclipse) & [IntelliJ IDEA](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-maven-intellij)
 * Exploring [project scaffolding](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption#project-scaffolding) & [JDK runtimes](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption#jdk-runtime-availability-and-support) (Java 8 and Java 11)
 * Using [Java annotations for Triggers, Bindings](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption#triggers-and-annotations) - with [reference](https://docs.microsoft.com/en-us/java/api/com.microsoft.azure.functions.annotation?view=azure-java-stable) docs.
 * Adopting [best practices](https://docs.microsoft.com/en-us/azure/azure-functions/functions-best-practices?tabs=java) for hosting, reliability and efficiency.
 * Java [code samples](https://docs.microsoft.com/en-us/samples/azure-samples/azure-functions-samples-java/azure-functions-java/) and [integration tutorials](https://docs.microsoft.com/en-us/azure/azure-functions/functions-event-hub-cosmos-db?tabs=bash)

In this blog post, we'll dive into one quickstart and briefly discuss other resources for awareness! Do check out the recommended exercises and resources for self-study!

---

## Create a function in Java with an Event Hub trigger and an Azure Cosmos DB output binding

Today's post will walk through the [Quickstart: Azure Functions](https://docs.microsoft.com/en-gb/azure/azure-functions/functions-event-hub-cosmos-db?tabs=bash) tutorial using the [Azure Functions Core Tools](https://www.npmjs.com/package/azure-functions-core-tools). In the process, we'll set up our development environment with the relevant command-line tools to make building Function apps simpler.

_Note: Completing this exercise may incur a cost of a few USD cents based on your Azure subscription. Explore [pricing details](https://azure.microsoft.com/en-us/pricing/details/functions/#pricing) to learn more_.

First, make sure you have your development environment set up and configured.

:::info PRE-REQUISITES

 1. **An Azure account with an active subscription** - [Create an account for free](https://azure.microsoft.com/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio)
 2. **The Java Development Kit, version 11 or 8.** - [Install](https://docs.microsoft.com/azure/developer/java/fundamentals/java-support-on-azure)
 3. **Apache Maven, version 3.0 or above.** - [Install](https://maven.apache.org/)
 4. **Azure Functions Core Tools version 2.6.666 or above** [Install](https://www.npmjs.com/package/azure-functions-core-tools)
:::

### 1. Setup your environmental variables

First, we need to set up some environment variables for the names and location of the resources you'll create. Use the following commands, replacing the <value> placeholders with values of your choosing. For the LOCATION variable, use one of the values produced by the ```az functionapp list-consumption-locations``` command.

```bash
RESOURCE_GROUP=<value>
EVENT_HUB_NAMESPACE=<value>
EVENT_HUB_NAME=<value>
EVENT_HUB_AUTHORIZATION_RULE=<value>
COSMOS_DB_ACCOUNT=<value>
STORAGE_ACCOUNT=<value>
FUNCTION_APP=<value>
LOCATION=<value>
```

### 2. Create your Azure resources

Use the following command to create a resource group:

```bash
az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION
```

The Event Hubs namespace contains the actual event hub and its authorization rule. The authorization rule enables your functions to send messages to the hub and listen for the related events. One function sends messages that represent telemetry data. Another function listens for events, analyzes the event data, and stores the results in Azure Cosmos DB.
Next, create an Azure Event Hubs namespace, event hub, and authorization rule using the following commands:

```bash
az eventhubs namespace create \
    --resource-group $RESOURCE_GROUP \
    --name $EVENT_HUB_NAMESPACE
az eventhubs eventhub create \
    --resource-group $RESOURCE_GROUP \
    --name $EVENT_HUB_NAME \
    --namespace-name $EVENT_HUB_NAMESPACE \
    --message-retention 1
az eventhubs eventhub authorization-rule create \
    --resource-group $RESOURCE_GROUP \
    --name $EVENT_HUB_AUTHORIZATION_RULE \
    --eventhub-name $EVENT_HUB_NAME \
    --namespace-name $EVENT_HUB_NAMESPACE \
    --rights Listen Send
```

Next, create an Azure Cosmos DB account, database, and collection using the following commands:

```bash
az cosmosdb create \
    --resource-group $RESOURCE_GROUP \
    --name $COSMOS_DB_ACCOUNT
az cosmosdb sql database create \
    --resource-group $RESOURCE_GROUP \
    --account-name $COSMOS_DB_ACCOUNT \
    --name TelemetryDb
az cosmosdb sql container create \
    --resource-group $RESOURCE_GROUP \
    --account-name $COSMOS_DB_ACCOUNT \
    --database-name TelemetryDb \
    --name TelemetryInfo \
    --partition-key-path '/temperatureStatus'
```

Next, create an Azure Storage account, which is required by Azure Functions, then create the function app. Use the following commands:

```bash
az storage account create \
    --resource-group $RESOURCE_GROUP \
    --name $STORAGE_ACCOUNT \
    --sku Standard_LRS
az functionapp create \
    --resource-group $RESOURCE_GROUP \
    --name $FUNCTION_APP \
    --storage-account $STORAGE_ACCOUNT \
    --consumption-plan-location $LOCATION \
    --runtime java \
    --functions-version 3
```

### 3. Configure your function app

Your function app will need to access the other resources to work correctly. The following sections show you how to configure your function app to run on your local machine.

Use the following commands to retrieve the storage, event hub, and Cosmos DB connection strings and save them in environment variables:

```bash
AZURE_WEB_JOBS_STORAGE=$( \
    az storage account show-connection-string \
        --name $STORAGE_ACCOUNT \
        --query connectionString \
        --output tsv)
echo $AZURE_WEB_JOBS_STORAGE
EVENT_HUB_CONNECTION_STRING=$( \
    az eventhubs eventhub authorization-rule keys list \
        --resource-group $RESOURCE_GROUP \
        --name $EVENT_HUB_AUTHORIZATION_RULE \
        --eventhub-name $EVENT_HUB_NAME \
        --namespace-name $EVENT_HUB_NAMESPACE \
        --query primaryConnectionString \
        --output tsv)
echo $EVENT_HUB_CONNECTION_STRING
COSMOS_DB_CONNECTION_STRING=$( \
    az cosmosdb keys list \
        --resource-group $RESOURCE_GROUP \
        --name $COSMOS_DB_ACCOUNT \
        --type connection-strings \
        --query 'connectionStrings[0].connectionString' \
        --output tsv)
echo $COSMOS_DB_CONNECTION_STRING
```

Next, use the following command to transfer the connection string values to app settings in your Azure Functions account:

```bash
az functionapp config appsettings set \
    --resource-group $RESOURCE_GROUP \
    --name $FUNCTION_APP \
    --settings \
        AzureWebJobsStorage=$AZURE_WEB_JOBS_STORAGE \
        EventHubConnectionString=$EVENT_HUB_CONNECTION_STRING \
        CosmosDBConnectionString=$COSMOS_DB_CONNECTION_STRING
```

Your Azure resources have now been created and configured 🎉.
Next, you'll create a project on your local machine, add Java code, and test it. After you get the functions working locally, we'll use Maven to deploy them to Azure.

### 4. Create a local functions project

Use the following Maven command to create a functions project and add the required dependencies.

```bash
mvn archetype:generate --batch-mode \
    -DarchetypeGroupId=com.microsoft.azure \
    -DarchetypeArtifactId=azure-functions-archetype \
    -DappName=$FUNCTION_APP \
    -DresourceGroup=$RESOURCE_GROUP \
    -DappRegion=$LOCATION \
    -DgroupId=com.example \
    -DartifactId=telemetry-functions
```

This command generates several files inside a telemetry-functions folder:

* A _pom.xml_ file for use with Maven
* A _local.settings.json_ file to hold app settings for local testing
* A _host.json_ file that enables the Azure Functions Extension Bundle, required for Cosmos DB output binding in your data analysis function
* A _Function.java_ file that includes a default function implementation
* A few test files that this tutorial doesn't need

Delete the test files. Run the following commands to navigate to the new project folder and delete the test folder:

```bash
cd telemetry-functions
rm -r src/test
```

### 4.1 Retrieve your function app settings for local use

For local testing, your function project will need the connection strings you added to your function app in Azure earlier in this tutorial. Use the following Azure Functions Core Tools command, which retrieves all the function app settings stored in the cloud and adds them to your local.settings.json file:

```bash
func azure functionapp fetch-app-settings $FUNCTION_APP
```

#### 4.2 Add Java code

Next, open the _Function.java_ file and replace the contents with the following code.

```java
package com.example;

import com.example.TelemetryItem.status;
import com.microsoft.azure.functions.annotation.Cardinality;
import com.microsoft.azure.functions.annotation.CosmosDBOutput;
import com.microsoft.azure.functions.annotation.EventHubOutput;
import com.microsoft.azure.functions.annotation.EventHubTrigger;
import com.microsoft.azure.functions.annotation.FunctionName;
import com.microsoft.azure.functions.annotation.TimerTrigger;
import com.microsoft.azure.functions.ExecutionContext;
import com.microsoft.azure.functions.OutputBinding;

public class Function {

    @FunctionName("generateSensorData")
    @EventHubOutput(
        name = "event",
        eventHubName = "", // blank because the value is included in the connection string
        connection = "EventHubConnectionString")
    public TelemetryItem generateSensorData(
        @TimerTrigger(
            name = "timerInfo",
            schedule = "*/10 * * * * *") // every 10 seconds
            String timerInfo,
        final ExecutionContext context) {

        context.getLogger().info("Java Timer trigger function executed at: "
            + java.time.LocalDateTime.now());
        double temperature = Math.random() * 100;
        double pressure = Math.random() * 50;
        return new TelemetryItem(temperature, pressure);
    }

    @FunctionName("processSensorData")
    public void processSensorData(
        @EventHubTrigger(
            name = "msg",
            eventHubName = "", // blank because the value is included in the connection string
            cardinality = Cardinality.ONE,
            connection = "EventHubConnectionString")
            TelemetryItem item,
        @CosmosDBOutput(
            name = "databaseOutput",
            databaseName = "TelemetryDb",
            collectionName = "TelemetryInfo",
            connectionStringSetting = "CosmosDBConnectionString")
            OutputBinding<TelemetryItem> document,
        final ExecutionContext context) {

        context.getLogger().info("Event hub message received: " + item.toString());

        if (item.getPressure() > 30) {
            item.setNormalPressure(false);
        } else {
            item.setNormalPressure(true);
        }

        if (item.getTemperature() < 40) {
            item.setTemperatureStatus(status.COOL);
        } else if (item.getTemperature() > 90) {
            item.setTemperatureStatus(status.HOT);
        } else {
            item.setTemperatureStatus(status.WARM);
        }

        document.setValue(item);
    }
}
```

As you can see, this file contains two functions, _generateSensorData_ and _processSensorData_. The generateSensorData function simulates a sensor that sends temperature and pressure readings to the event hub. A timer trigger runs the function every 10 seconds, and an event hub output binding sends the return value to the event hub.

When the event hub receives the message, it generates an event. The processSensorData function runs when it receives the event. It then processes the event data and uses an Azure Cosmos DB output binding to send the results to Azure Cosmos DB.

The data used by these functions is stored using a class called _TelemetryItem_, which you'll need to implement.
Create a new file called _TelemetryItem.java_ in the same location as Function.java and add the following code:

```Java
package com.example;

public class TelemetryItem {

    private String id;
    private double temperature;
    private double pressure;
    private boolean isNormalPressure;
    private status temperatureStatus;
    static enum status {
        COOL,
        WARM,
        HOT
    }

    public TelemetryItem(double temperature, double pressure) {
        this.temperature = temperature;
        this.pressure = pressure;
    }

    public String getId() {
        return id;
    }

    public double getTemperature() {
        return temperature;
    }

    public double getPressure() {
        return pressure;
    }

    @Override
    public String toString() {
        return "TelemetryItem={id=" + id + ",temperature="
            + temperature + ",pressure=" + pressure + "}";
    }

    public boolean isNormalPressure() {
        return isNormalPressure;
    }

    public void setNormalPressure(boolean isNormal) {
        this.isNormalPressure = isNormal;
    }

    public status getTemperatureStatus() {
        return temperatureStatus;
    }

    public void setTemperatureStatus(status temperatureStatus) {
        this.temperatureStatus = temperatureStatus;
    }
}
```

### 4.3 Run locally

You can now build and run the functions locally and see data appear in your Azure Cosmos DB.

Use the following Maven commands to build and run the functions:

```bash
mvn clean package
mvn azure-functions:run
```

After some build and startup messages, you'll see output similar to the following example for each time the functions run:

```bash
[10/22/22 4:01:30 AM] Executing 'Functions.generateSensorData' (Reason='Timer fired at 2019-10-21T21:01:30.0016769-07:00', Id=c1927c7f-4f70-4a78-83eb-bc077d838410)
[10/22/22 4:01:30 AM] Java Timer trigger function executed at: 2019-10-21T21:01:30.015
[10/22/22 4:01:30 AM] Function "generateSensorData" (Id: c1927c7f-4f70-4a78-83eb-bc077d838410) invoked by Java Worker
[10/22/22 4:01:30 AM] Executed 'Functions.generateSensorData' (Succeeded, Id=c1927c7f-4f70-4a78-83eb-bc077d838410)
[10/22/22 4:01:30 AM] Executing 'Functions.processSensorData' (Reason='', Id=f4c3b4d7-9576-45d0-9c6e-85646bb52122)
[10/22/22 4:01:30 AM] Event hub message received: TelemetryItem={id=null,temperature=32.728691307527015,pressure=10.122563042388165}
[10/22/22 4:01:30 AM] Function "processSensorData" (Id: f4c3b4d7-9576-45d0-9c6e-85646bb52122) invoked by Java Worker
[10/22/22 4:01:38 AM] Executed 'Functions.processSensorData' (Succeeded, Id=1cf0382b-0c98-4cc8-9240-ee2a2f71800d)
```

You can then go to the Azure portal and navigate to your Azure Cosmos DB account. Select Data Explorer, expand TelemetryInfo, then select Items to view your data when it arrives.

:::image type="content" source="./img/java/data-explorer.png" alt-text="Data Explorer view":::

### 5. Deploy to Azure and view app telemetry

You can deploy your app to Azure and verify that it continues to work the same way it did locally.

Deploy your project to Azure using the following command:

```bash
mvn azure-functions:deploy
```

Your functions now run in Azure, and continue accumulating data in your Azure Cosmos DB.
When the ```az functionapp create``` command created your function app, it also created an Application Insights resource with the same name.
In the [Azure portal](https://portal.azure.com/), you can view your app's telemetry through the connected Application Insights resource, as shown in the following screenshots:

**Live Metrics Stream:**
:::image type="content" source="./img/java/application-insights-live-metrics-stream.png" alt-text="application insights live metrics":::

**Performance:**
:::image type="content" source="./img/java/application-insights-performance.png" alt-text="application insights performance":::

### 6. Clean up

Use the following command to delete the resource group and all its contained resources to avoid incurring further costs.

```bash
az group delete --name $RESOURCE_GROUP
```

## Next Steps

So, where can you go from here? The example above used a familiar `HTTP Trigger` scenario with a single Azure service (Azure Functions). Now, think about how you can build richer workflows by using other triggers and integrating with other Azure or third-party services.

### Other Triggers, Bindings

Check out [Azure Functions Java developer guide](https://docs.microsoft.com/en-gb/azure/azure-functions/functions-reference-java) for detailed information to help you succeed in developing Azure Functions using Java.

### Scenario with Integrations

Once you've tried out the samples, try building an end-to-end scenario by using these triggers to integrate seamlessly with other Services. Here are a couple of useful tutorials:
 * [Use Key Vault references for App Service and Azure Functions](https://docs.microsoft.com/azure/app-service/app-service-key-vault-references?tabs=azure-cli)
 * GitHub Star Count app with [SignalR trigger](https://docs.microsoft.com/en-us/azure/azure-signalr/signalr-quickstart-azure-functions-java?toc=%2Fazure%2Fazure-functions%2Ftoc.json)

## Exercise

Time to put this into action and validate your development workflow:
 * Walk through this tutorial yourself, and deploy your first function!
 * Complete the [Develop Java serverless Functions on Azure using Maven](https://docs.microsoft.com/learn/modules/develop-azure-functions-app-with-maven-plugin/) module

## Resources
 * [Azure Functions: Java Quickstarts](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-java)
 * [Best Practices for Java Apps On Azure](https://docs.microsoft.com/en-us/learn/paths/best-practices-java-azure/)
 * [Java at Microsoft](https://developer.microsoft.com/en-us/java/)
 * [Java Integrations: Azure Functions and SignalR](https://docs.microsoft.com/en-us/azure/azure-signalr/signalr-quickstart-azure-functions-java?toc=%2Fazure%2Fazure-functions%2Ftoc.json)
 * [Java Samples: Azure Functions](https://docs.microsoft.com/en-us/samples/browse/?products=azure-functions&languages=java)
