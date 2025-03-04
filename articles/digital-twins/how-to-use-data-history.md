---
# Mandatory fields.
title: Use data history with Azure Data Explorer
titleSuffix: Azure Digital Twins
description: See how to set up and use data history for Azure Digital Twins, using the CLI or Azure portal.
author: baanders
ms.author: baanders # Microsoft employees only
ms.date: 03/23/2022
ms.topic: how-to
ms.service: digital-twins
ms.custom: event-tier1-build-2022

# Optional fields. Don't forget to remove # if you need a field.
# ms.custom: can-be-multiple-comma-separated
# ms.reviewer: MSFT-alias-of-reviewer
# manager: MSFT-alias-of-manager-or-PM-counterpart
---

# Use Azure Digital Twins data history

[Data history](concepts-data-history.md) is an Azure Digital Twins feature for automatically historizing twin property updates to [Azure Data Explorer](/azure/data-explorer/data-explorer-overview). This data can be queried using the [Azure Digital Twins query plugin for Azure Data Explorer](concepts-data-explorer-plugin.md) to gain insights about your environment over time.

This article shows how to set up a working data history connection between Azure Digital Twins and Azure Data Explorer. It uses the [Azure CLI](/cli/azure/what-is-azure-cli) and the [Azure portal](https://portal.azure.com) to set up and connect the required data history resources, including:
* an Azure Digital Twins instance
* an [Event Hubs](../event-hubs/event-hubs-about.md) namespace containing an event hub
* an [Azure Data Explorer](/azure/data-explorer/data-explorer-overview) cluster containing a database

It also contains a sample twin graph that you can use to see the historized twin property updates in Azure Data Explorer. 

>[!TIP]
>Although this article uses the Azure portal, you can also work with data history using the [2022-05-31](https://github.com/Azure/azure-rest-api-specs/tree/main/specification/digitaltwins/data-plane/Microsoft.DigitalTwins/stable/2022-05-31) version of the rest APIs.

## Prerequisites

[!INCLUDE [azure-cli-prepare-your-environment-h3.md](../../includes/azure-cli-prepare-your-environment-h3.md)]

>[!NOTE]
> You can also use Azure Cloud Shell in the PowerShell environment instead of the Bash environment, if you prefer. The commands on this page are written for the Bash environment, so they may require some small adjustments to be run in PowerShell.

[!INCLUDE [CLI setup for Azure Digital Twins](../../includes/digital-twins-cli.md)]

### Set up local variables for CLI session

This article provides CLI commands that you can use to create the data history resources. In order to make it easy to copy and run those commands later, you can set up local variables in your CLI session now, and then refer to those variables later in the CLI commands when creating your resources. Update the placeholders (identified with `<...>` brackets) in the commands below, and run these commands to create the variables. Make sure to follow the naming rules described in the comments. These values will be used later when creating the new resources.

>[!NOTE]
>These commands are written for the Bash environment. They can be adjusted for PowerShell if you prefer to use a PowerShell CLI environment.

```azurecli-interactive
## General Setup
location="<your-resource-region>"
resourcegroup="<your-resource-group-name>"

## Azure Digital Twins Setup
# Instance name can contain letters, numbers, and hyphens. It must start and end with a letter or number, and be between 4 and 62 characters long.
dtname="<name-for-your-digital-twins-instance>"
# Connection name can contain letters, numbers, and hyphens. It must contain at least one letter, and be between 3 and 50 characters long.
connectionname="<name-for-your-data-history-connection>"

## Event Hub Setup
# Namespace can contain letters, numbers, and hyphens. It must start with a letter, end with a letter or number, and be between 6 and 50 characters long.
eventhubnamespace="<name-for-your-event-hub-namespace>"
# Event hub name can contain only letters, numbers, periods, hyphens and underscores. It must start and end with a letter or number.
eventhub="<name-for-your-event-hub>"

## Azure Data Explorer Setup
# Cluster name can contain only lowercase alphanumeric characters. It must start with a letter, and be between 4 and 22 characters long.
clustername="<name-for-your-cluster>"  
# Database name can contain only alphanumeric, spaces, dash and dot characters, and be up to 260 characters in length.
databasename="<name-for-your-database>"
```

## Create an Azure Digital Twins instance with a managed identity

If you already have an Azure Digital Twins instance, ensure that you've enabled a [system-managed identity](how-to-route-with-managed-identity.md#add-a-system-managed-identity-to-an-existing-instance) for it.

If you don't have an Azure Digital Twins instance, set one up using the instructions in this section.

# [CLI](#tab/cli) 

Use the following command to create a new instance with a system-managed identity. The command uses three local variables (`$dtname`, `$resourcegroup`, and `$location`) that were created earlier in [Set up local variables for CLI session](#set-up-local-variables-for-cli-session).

```azurecli-interactive
az dt create --dt-name $dtname --resource-group $resourcegroup --location $location --assign-identity
```

Next, use the following command to grant yourself the *Azure Digital Twins Data Owner* role on the instance. The command has one placeholder, `<owneruser@microsoft.com>`, that you should replace with your own Azure account information, and uses a local variable (`$dtname`) that was created earlier in [Set up local variables for CLI session](#set-up-local-variables-for-cli-session).

```azurecli-interactive
az dt role-assignment create --dt-name $dtname --assignee "<owneruser@microsoft.com>" --role "Azure Digital Twins Data Owner"
```

>[!NOTE]
>It may take up to five minutes for this RBAC change to apply. 

# [Portal](#tab/portal)

Follow the instructions in [Set up an Azure Digital Twins instance and authentication](how-to-set-up-instance-portal.md) to create an instance, making sure to enable a **system-managed identity** in the [Advanced](how-to-set-up-instance-portal.md#additional-setup-options) tab during setup. Then, continue through the article's instructions to set up user access permissions so that you have the Azure Digital Twins Data Owner role on the instance.

Remember the name you give to your instance so you can use it later.

---

## Create an Event Hubs namespace and event hub

The next step is to create an Event Hubs namespace and an event hub. This hub will receive digital twin property update notifications from the Azure Digital Twins instance and then forward the messages to the target Azure Data Explorer cluster. 

As part of the [data history connection setup](#set-up-data-history-connection) later, you'll grant the Azure Digital Twins instance the *Azure Event Hubs Data Owner* role on the event hub resource.

For more information about Event Hubs and their capabilities, see the [Event Hubs documentation](../event-hubs/event-hubs-about.md).

# [CLI](#tab/cli) 

Use the following CLI commands to create the required resources. The commands use several local variables (`$location`, `$resourcegroup`, `$eventhubnamespace`, and `$eventhub`) that were created earlier in [Set up local variables for CLI session](#set-up-local-variables-for-cli-session).

Create an Event Hubs namespace:

```azurecli-interactive
az eventhubs namespace create --name $eventhubnamespace --resource-group $resourcegroup --location $location
```

Create an event hub in your namespace:

```azurecli-interactive
az eventhubs eventhub create --name $eventhub --resource-group $resourcegroup --namespace-name $eventhubnamespace
```

# [Portal](#tab/portal)

Follow the instructions in [Create an event hub using Azure portal](../event-hubs/event-hubs-create.md) to create an Event Hubs namespace and an event hub. (The article also contains instructions on how to create a new resource group. You can create a new resource group for the Event Hubs resources, or skip that step and use an existing resource group for your new Event Hubs resources.)

Remember the names you give to these resources so you can use them later.

---

## Create a Kusto (Azure Data Explorer) cluster and database

Next, create a Kusto (Azure Data Explorer) cluster and database to receive the data from Azure Digital Twins. 

As part of the [data history connection setup](#set-up-data-history-connection) later, you'll grant the Azure Digital Twins instance the *Contributor* role on at least the database (it can also be scoped to the cluster), and the *Admin* role on the database.

# [CLI](#tab/cli) 

Use the following CLI commands to create the required resources. The commands use several local variables (`$location`, `$resourcegroup`, `$clustername`, and `$databasename`) that were created earlier in [Set up local variables for CLI session](#set-up-local-variables-for-cli-session).

Start by adding the Kusto extension to your CLI session, if you don't have it already.

```azurecli-interactive
az extension add --name kusto
```

Next, create the Kusto cluster. The command below requires 5-10 minutes to execute, and will create an E2a v4 cluster in the developer tier. This type of cluster has a single node for the engine and data-management cluster, and is applicable for development and test scenarios. For more information about the tiers in Azure Data Explorer and how to select the right options for your production workload, see [Select the correct compute SKU for your Azure Data Explorer cluster](/azure/data-explorer/manage-cluster-choose-sku) and [Azure Data Explorer Pricing](https://azure.microsoft.com/pricing/details/data-explorer).

```azurecli-interactive
az kusto cluster create --cluster-name $clustername --sku name="Dev(No SLA)_Standard_E2a_v4" tier="Basic" --resource-group $resourcegroup --location $location --type SystemAssigned
```

Create a database in your new Kusto cluster (using the cluster name from above and in the same location). This database will be used to store contextualized Azure Digital Twins data. The command below creates a database with a soft delete period of 365 days, and a hot cache period of 31 days. For more information about the options available for this command, see [az kusto database create](/cli/azure/kusto/database?view=azure-cli-latest&preserve-view=true#az-kusto-database-create).

```azurecli-interactive
az kusto database create --cluster-name $clustername --database-name $databasename --resource-group $resourcegroup --read-write-database soft-delete-period=P365D hot-cache-period=P31D location=$location
```

# [Portal](#tab/portal)

Follow the instructions in [Create an Azure Data Explorer cluster and database](/azure/data-explorer/create-cluster-database-portal?tabs=one-click-create-database) to create an Azure Data Explorer cluster and a database in the cluster.

Remember the names you give to these resources so you can use them later.

---

## Set up data history connection

Now that you've created the required resources, use the command below to create a data history connection between the Azure Digital Twins instance, the event hub, and the Azure Data Explorer cluster. 

# [CLI](#tab/cli) 

Use the following command to create a data history connection. By default, this command assumes all resources are in the same resource group as the Azure Digital Twins instance. You can also specify resources that are in different resource groups using the parameter options for this command, which can be displayed by running `az dt data-history connection create adx -h`.
The command uses several local variables (`$connectionname`, `$dtname`, `$clustername`, `$databasename`, `$eventhub`, and `$eventhubnamespace`) that were created earlier in [Set up local variables for CLI session](#set-up-local-variables-for-cli-session).

```azurecli-interactive
az dt data-history connection create adx --cn $connectionname --dt-name $dtname --adx-cluster-name $clustername --adx-database-name $databasename --eventhub $eventhub --eventhub-namespace $eventhubnamespace
```

When executing the above command, you'll be given the option of assigning the necessary permissions required for setting up your data history connection on your behalf (if you've already assigned the necessary permissions, you can skip these prompts). These permissions are granted to the managed identity of your Azure Digital Twins instance. The minimum required roles are:
* Azure Event Hubs Data Owner on the event hub
* Contributor scoped at least to the specified database (it can also be scoped to the cluster)
* Database principal assignment with role Admin (for table creation / management) scoped to the specified database

For regular data plane operation, these roles can be reduced to a single Azure Event Hubs Data Sender role, if desired.

>[!NOTE]
> If you encounter the error "Could not create Azure Digital Twins instance connection. Unable to create table and mapping rule in database. Check your permissions for the Azure Database Explorer and run `az login` to refresh your credentials," resolve the error by adding yourself as an *AllDatabasesAdmin* under Permissions in your Azure Data Explorer cluster.
>
>If you're using the Cloud Shell and encounter the error "Failed to connect to MSI. Please make sure MSI is configured correctly," try running the command with a local Azure CLI installation instead.

# [Portal](#tab/portal)

Start by navigating to your Azure Digital Twins instance in the Azure portal (you can find the instance by entering its name into the portal search bar). Then complete the following steps.

1. Select **Data history** from the Connect Outputs section of the instance's menu.
    :::image type="content"  source="media/how-to-use-data-history/select-data-history.png" alt-text="Screenshot of the Azure portal showing the data history option in the menu for an Azure Digital Twins instance." lightbox="media/how-to-use-data-history/select-data-history.png":::

    Select **Create a connection**. Doing so will begin the process of creating a data history connection.

2. **(SOME USERS)** If you **don't** already have a [managed identity enabled for your Azure Digital Twins instance](how-to-route-with-managed-identity.md), you'll see this page first, asking you to turn on Identity for the instance as the first step for the data history connection.

    :::image type="content"  source="media/how-to-use-data-history/authentication.png" alt-text="Screenshot of the Azure portal showing the first step in the data history connection setup, Authentication." lightbox="media/how-to-use-data-history/authentication.png":::

    If you **do** already have a managed identity enabled, your setup will **skip this step** and you'll see the next page immediately.

3. On the **Send** page, enter the details of the [Event Hubs resources](#create-an-event-hubs-namespace-and-event-hub) that you created earlier.
    :::image type="content"  source="media/how-to-use-data-history/send.png" alt-text="Screenshot of the Azure portal showing the Send step in the data history connection setup." lightbox="media/how-to-use-data-history/send.png":::

    Select **Next**.

4. On the **Store** page, enter the details of the [Azure Data Explorer resources](#create-a-kusto-azure-data-explorer-cluster-and-database) that you created earlier and choose a name for your database table.
    :::image type="content"  source="media/how-to-use-data-history/store.png" alt-text="Screenshot of the Azure portal showing the Store step in the data history connection setup." lightbox="media/how-to-use-data-history/store.png":::

    Select **Next**.

5. On the **Permission** page, select all of the checkboxes to give your Azure Digital Twins instance permission to connect to the Event Hubs and Azure Data Explorer resources. If you already have equal or higher permissions in place, you can skip this step.
    :::image type="content"  source="media/how-to-use-data-history/permission.png" alt-text="Screenshot of the Azure portal showing the Permission step in the data history connection setup." lightbox="media/how-to-use-data-history/permission.png":::

    Select **Next**. 

6. On the **Review + create** page, review the details of your resources and select **Create connection**.
    :::image type="content"  source="media/how-to-use-data-history/review-create.png" alt-text="Screenshot of the Azure portal showing the Review and Create step in the data history connection setup." lightbox="media/how-to-use-data-history/review-create.png":::

When the connection is finished creating, you'll be taken back to the **Data history** page for the Azure Digital Twins instance, which now shows details of the data history connection you've created.

:::image type="content"  source="media/how-to-use-data-history/data-history-details.png" alt-text="Screenshot of the Azure portal showing the Data History Details page after setting up a connection." lightbox="media/how-to-use-data-history/data-history-details.png":::

---

After setting up the data history connection, you can optionally remove the roles granted to your Azure Digital Twins instance for accessing the Event Hubs and Azure Data Explorer resources. In order to use data history, the only role the instance needs going forward is *Azure Event Hubs Data Sender* (or a higher role that includes these permissions, such as *Azure Event Hubs Data Owner*) on the Event Hubs resource.

>[!NOTE]
>Once the connection is set up, the default settings on your Azure Data Explorer cluster will result in an ingestion latency of approximately 10 minutes or less. You can reduce this latency by enabling [streaming ingestion](/azure/data-explorer/ingest-data-streaming) (less than 10 seconds of latency) or an [ingestion batching policy](/azure/data-explorer/kusto/management/batchingpolicy). For more information about Azure Data Explorer ingestion latency, see [End-to-end ingestion latency](concepts-data-history.md#end-to-end-ingestion-latency).

## Verify with a sample twin graph

Now that your data history connection is set up, you can test it with data from your digital twins.

If you already have twins in your Azure Digital Twins instance that are receiving property updates, you can skip this section and visualize the results using your own resources. 

Otherwise, continue through this section to set up a sample graph containing twins that receives twin property updates. 

You can set up a sample graph for this scenario using the **Azure Digital Twins Data Simulator**. The Azure Digital Twins Data Simulator continuously pushes property updates to several twins in an Azure Digital Twins instance.

### Create a sample graph

You can use the **Azure Digital Twins Data Simulator** to provision a sample twin graph and push property updates to it. The twin graph created here models pasteurization processes for a dairy company.

Start by opening the [Azure Digital Twins Data Simulator](https://explorer.digitaltwins.azure.net/tools/data-pusher) in your browser. Set these fields:
* **Instance URL**: Enter the host name of your Azure Digital Twins instance. The host name can be found in the [portal](https://portal.azure.com) page for your instance, and has a format like `<Azure-Digital-Twins-instance-name>.api.<region-code>.digitaltwins.azure.net`. 
* **Simulation Type**: Select *Dairy facility* from the dropdown menu.

Select **Generate Environment**. 

:::image type="content"  source="media/how-to-use-data-history/data-simulator.png" alt-text="Screenshot of the Azure Digital Twins Data simulator.":::

You'll see confirmation messages on the screen as models, twins, and relationships are created in your environment. 

When the simulation is ready, the **Start simulation** button will become enabled. Select **Start simulation** to push simulated data to your Azure Digital Twins instance. To continuously update the twins in your Azure Digital Twins instance, keep this browser window in the foreground on your desktop (and complete other browser actions in a separate window). 

To verify that data is flowing through the data history pipeline, navigate to the [Azure portal](https://portal.azure.com) and open the Event Hubs namespace resource you created. You should see charts showing the flow of messages into and out of the namespace, indicating the flow of incoming messages from Azure Digital Twins and outgoing messages to Azure Data Explorer.

:::image type="content"  source="media/how-to-use-data-history/simulated-environment-portal.png" alt-text="Screenshot of the Azure portal showing an Event Hubs namespace for the simulated environment." lightbox="media/how-to-use-data-history/simulated-environment-portal.png":::

### View the historized twin updates in Azure Data Explorer

In this section, you'll view the historized twin updates being stored in Azure Data Explorer.

Start in the [Azure portal](https://portal.azure.com) and navigate to the Azure Data Explorer cluster you created earlier. Choose the **Databases** pane from the left menu to open the database view. Find the database you created for this article and select the checkbox next to it, then select **Query**.

:::image type="content"  source="media/how-to-use-data-history/azure-data-explorer-database.png" alt-text="Screenshot of the Azure portal showing a database in an Azure Data Explorer cluster.":::

Next, expand the cluster and database in the left pane to see the name of the table. You'll use this name to run queries on the table.

:::image type="content"  source="media/how-to-use-data-history/data-history-table.png" alt-text="Screenshot of the Azure portal showing the query view for the database. The name of the data history table is highlighted." lightbox="media/how-to-use-data-history/data-history-table.png":::

Copy the command below. The command will change the ingestion to [batched mode](concepts-data-history.md#batch-ingestion-default) and ingest every 10 seconds.

```kusto
.alter table <table-name> policy ingestionbatching @'{"MaximumBatchingTimeSpan":"00:00:10", "MaximumNumberOfItems": 500, "MaximumRawDataSizeMB": 1024}'
```

Paste the command into the query window, replacing the `<table-name>` placeholder with the name of your table. Select the **Run** button.

:::image type="content"  source="media/how-to-use-data-history/data-history-run-query-1.png" alt-text="Screenshot of the Azure portal showing the query view for the database. The Run button is highlighted." lightbox="media/how-to-use-data-history/data-history-run-query-1.png":::

Next, add the following command to the query window, and run it again to verify that Azure Data Explorer has ingested twin updates into the table.

>[!NOTE]
> It may take up to 5 minutes for the first batch of ingested data to appear.

```kusto
<table_name>
| count
```

You should see in the results that the count of items in the table is something greater than 0.

You can also add and run the following command to view 100 records in the table:

```kusto
<table_name>
| limit 100
```

Next, run a query based on the data of your twins to see the contextualized time series data. 

Use the query below to chart the outflow of all salt machine twins in the Oslo dairy. This Kusto query uses the Azure Digital Twins plugin to select the twins of interest, joins those twins against the data history time series in Azure Data Explorer, and then charts the results. Make sure to replace the `<ADT-instance>` placeholder with the URL of your instance, in the format `https://<instance-host-name>`.

```kusto
let ADTendpoint = "<ADT-instance>";
let ADTquery = ```SELECT SALT_MACHINE.$dtId as tid
FROM DIGITALTWINS FACTORY 
JOIN SALT_MACHINE RELATED FACTORY.contains 
WHERE FACTORY.$dtId = 'OsloFactory'
AND IS_OF_MODEL(SALT_MACHINE , 'dtmi:assetGen:SaltMachine;1')```;
evaluate azure_digital_twins_query_request(ADTendpoint, ADTquery)
| extend Id = tostring(tid)
| join kind=inner (<table_name>) on Id
| extend val_double = todouble(Value)
| where Key == "OutFlow"
| render timechart with (ycolumns = val_double)
```

The results should show the outflow numbers changing over time.

:::image type="content"  source="media/how-to-use-data-history/data-history-run-query-2.png" alt-text="Screenshot of the Azure portal showing the query view for the database. The result for the example query is a line graph showing changing values over time for the salt machine outflows." lightbox="media/how-to-use-data-history/data-history-run-query-2.png":::

## Next steps 

To keep exploring the dairy scenario, you can view [more sample queries on GitHub](https://github.com/Azure-Samples/azure-digital-twins-getting-started/blob/main/adt-adx-queries/Dairy_operation_with_data_history/ContosoDairyDataHistoryQueries.kql) that show how you can monitor the performance of the dairy operation based on machine type, factory, maintenance technician, and various combinations of these parameters.

To build Grafana dashboards that visualize the performance of the dairy operation, read [Creating dashboards with Azure Digital Twins, Azure Data Explorer, and Grafana](https://github.com/Azure-Samples/azure-digital-twins-getting-started/tree/main/adt-adx-queries/Dairy_operation_with_data_history/Visualize_with_Grafana).

For more information on using the Azure Digital Twins query plugin for Azure Data Explorer, see [Querying with the Azure Data Explorer plugin](concepts-data-explorer-plugin.md) and [this blog post](https://techcommunity.microsoft.com/t5/internet-of-things/adding-context-to-iot-data-just-became-easier/ba-p/2459987). You can also read more about the plugin here: [Querying with the Azure Data Explorer plugin](concepts-data-explorer-plugin.md).
