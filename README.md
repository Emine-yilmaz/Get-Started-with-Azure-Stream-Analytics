# Get-Started-with-Azure-Stream-Analytics

This document outlines the steps to provision and configure an Azure Stream Analytics job to process a stream of real-time sales transaction data and store the summarized results in Azure Storage.

## Prerequisites

* An Azure subscription with administrative-level access.

## Steps

### 1. Provision Azure Resources

Provision Azure Event Hubs and Azure Storage using the provided PowerShell script and ARM template.

1.  Sign in to the [Azure portal](https://portal.azure.com).
2.  Open **Cloud Shell** by clicking the **[\>_]** button. Select **PowerShell** if prompted.
3.  Clone the repository:
    ```powershell
    rm -r dp-203 -f
    git clone [https://github.com/MicrosoftLearning/dp-203-azure-data-engineer](https://github.com/MicrosoftLearning/dp-203-azure-data-engineer) dp-203
    ```
4.  Navigate to the lab directory and run the setup script:
    ```powershell
    cd dp-203/Allfiles/labs/17
    ./setup.ps1
    ```
5.  If prompted, choose your subscription.
6.  Wait for the script to complete (approximately 5 minutes).

### 2. View the Streaming Data Source

Observe the simulated sales order data being sent to Azure Event Hubs.

1.  After the script completes, in the Azure portal, navigate to the `dp203-xxxxxxx` resource group. Note the Azure Storage account (`storexxxxxxxxxxxx`) and the Event Hubs namespace (`eventsxxxxxxxx`). Also, note the **Location** of these resources.
2.  Re-open the Cloud Shell.
3.  Run the client application to send simulated orders:
    ```powershell
    node ~/dp-203/Allfiles/labs/17/orderclient
    ```
4.  Observe the output showing `ProductID` and `Quantity` for each order being sent. The application will send 1000 orders and then stop.

### 3. Create an Azure Stream Analytics Job

Create a Stream Analytics job to process the incoming event stream.

1.  In the Azure portal, on the `dp203-xxxxxxx` page, click **+ Create** and search for **Stream Analytics job**.
2.  Create a **Stream Analytics job** with the following properties:
    * **Subscription**: Your Azure subscription
    * **Resource group**: `dp203-xxxxxxx`
    * **Name**: `process-orders`
    * **Region**: The same region where your Event Hubs and Storage account are provisioned.
    * **Hosting environment**: `Cloud`
    * **Streaming units**: `1`
    * **Storage**: Leave the storage account unselected for now.
    * **Tags**: None
3.  Wait for the deployment to complete and then navigate to the deployed **process-orders** Stream Analytics job resource.

### 4. Create an Input for the Event Stream

Configure the input to receive data from the Event Hub.

1.  On the **process-orders** overview page, click **Add input**.
2.  On the **Inputs** page, click **Add stream input** and select **Event Hub**.
3.  Configure the Event Hub input with the following properties:
    * **Input alias**: `orders`
    * **Select Event Hub from your subscriptions**: Selected
    * **Subscription**: Your Azure subscription
    * **Event Hub namespace**: Select the `eventsxxxxxxxx` Event Hubs namespace.
    * **Event Hub name**: Select the `eventhubxxxxxxxx` event hub.
    * **Event Hub consumer group**: Select `$Default`.
    * **Authentication mode**: `Create system assigned managed identity`
    * **Partition key**: Leave blank.
    * **Event serialization format**: `JSON`
    * **Encoding**: `UTF-8`
4.  Click **Save** and wait for the input to be created and the connection test to succeed.

### 5. Create an Output for the Blob Store

Configure the output to store processed data in Azure Blob Storage.

1.  Navigate to the **Outputs** page for the **process-orders** Stream Analytics job.
2.  Click **Add** and select **Blob storage/ADLS Gen2**.
3.  Configure the Blob storage output with the following properties:
    * **Output alias**: `blobstore`
    * **Select Blob storage/ADLS Gen2 from your subscriptions**: Selected
    * **Subscription**: Your Azure subscription
    * **Storage account**: Select the `storexxxxxxxxxxxx` storage account.
    * **Container**: Select the `data` container.
    * **Authentication mode**: `Managed Identity: System assigned`
    * **Event serialization format**: `JSON`
    * **Format**: `Line separated`
    * **Encoding**: `UTF-8`
    * **Write mode**: `Append as results arrive`
    * **Path pattern**: `{date}`
    * **Date format**: `YYYY/MM/DD`
    * **Time format**: Not applicable
    * **Minimum rows**: `20`
    * **Maximum time**: `0 Hours, 1 minutes, 0 seconds`
4.  Click **Save** and wait for the output to be created and the connection test to succeed.

### 6. Create a Query

Define the Stream Analytics query to process and aggregate the incoming data.

1.  Navigate to the **Query** page for the **process-orders** Stream Analytics job.
2.  Observe the input preview (if available).
3.  Modify the default query with the following code:
    ```sql
    SELECT
        DateAdd(second,-10,System.TimeStamp) AS StartTime,
        System.TimeStamp AS EndTime,
        ProductID,
        SUM(Quantity) AS Orders
    INTO
        [blobstore]
    FROM
        [orders] TIMESTAMP BY EventProcessedUtcTime
    GROUP BY ProductID, TumblingWindow(second, 10)
    HAVING COUNT(*) > 1
    ```
4.  Click the **▷ Test query** button to validate the query. Ensure the status indicates **Success**.
5.  Click **Save** to save the query.

### 7. Run the Streaming Job

Start the Stream Analytics job to begin processing real-time data.

1.  Navigate to the **Overview** page for the **process-orders** Stream Analytics job. Verify that the **orders** input and **blobstore** output are listed. Refresh if necessary.
2.  Click the **▷ Start** button at the top and start the job **Now**. Wait for the notification that the streaming job started successfully.
3.  Re-open the Cloud Shell and re-run the order client application:
    ```powershell
    node ~/dp-203/Allfiles/labs/17/orderclient
    ```
4.  While the client app is running, in the Azure portal, navigate to the `dp203-xxxxxxx` resource group and open the `storexxxxxxxxxxxx` storage account.
5.  Go to the **Containers** tab and open the **data** container.
6.  Refresh the view until you see a folder structure based on the current year, month, and day.
7.  Navigate to the folder for the current hour and find the JSON file (e.g., `0_xxxxxxxxxxxxxxxx.json`).
8.  Open the file (View/edit) and review its contents. You should see JSON records showing the aggregated order quantities for each `ProductID` within 10-second windows.
9.  In the Cloud Shell, wait for the order client app to finish.
10. In the Azure portal, refresh the blob storage file to see the complete processed results.
11. Return to the **process-orders** Stream Analytics job in the `dp203-xxxxxxx` resource group.
12. Click the **⬜ Stop** button at the top of the page and confirm to stop the job.

**Important:** After completing this lab, remember to delete the resource group `dp203-xxxxxxx` in the Azure portal to avoid incurring unnecessary costs.
