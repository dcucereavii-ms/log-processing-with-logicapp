# Log Processing with Logic App 

![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/5c2cd40a-642f-44aa-9482-5dbeee84c1eb)

This how-to guide around building logic app workflow set to monitor and then process Blob storage account events using Event Grid trigger. In order to accomplish, we are going to use the following Azure Service:

* Blob Storage
* Event Grid Connector for Logic App
* Logic Apps
* Event Hub

## Goal
Upon logs arival in the Blob Storage account, an Azure Function will be triggered to decompress GZip files and copy them to a new Container as json files. A logic app is set up to monitor the arival of the new json file in the second conatiner. Logic App retrieves the event message, extracts file details, and subsequently sends events to Even Hub. The even is then available in the Event Hub.

## Guide

### Connect everything with Logic Apps and Event Grid

1. Create a Logic App. We will use this and Event Grid to tie everything together.
   
   ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/f2371e5c-47f8-4f50-8332-d2b6031b9f01)

2. Navigate to the Logic Apps Designer and select **When an Event Grid event occurs**.

 ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/aba29bf3-52b9-4fce-adad-c6ae1fc9f159)

3. Create an Event Subscrition for the Storage Account we want to monitor. You may have to login to Logic Apps.

    * Set the Subscription field to the Azure Subscription your resoruces are in.
    * Set the Resource Type to `Microsoft.Storage.StorageAccounts`.
    * Set the Resource Name to the Storage Account we want to monitor.
    * Click Edit to change the default paramaters and select Eveny Type `Microsoft.Storage.BlobCreated`.
        > We only want to trigger this workflow when we have a file added, not deleted.
    * Click Advanced Options and create a suffix filter for `jsonslog` container.
        > See the list of Azure Storage events that allow applications to react to events, such as the creation and deletion of blobs here: https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-event-overview
  
   ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/a8d33a4b-5a40-479b-b69c-b8e3e09bf7bb)

    * Give your Event Subscription a name.
      ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/a323c3ce-036d-4f3d-b069-223a63f5fbc5)
