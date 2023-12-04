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
    * Click Advanced Options and create a suffix filter for our container.
        > See the list of Azure Storage events that allow applications to react to events, such as the creation and deletion of blobs here: https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-event-overview
  
   ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/a8d33a4b-5a40-479b-b69c-b8e3e09bf7bb)

    * Give your Event Subscription a name.
      
      ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/a323c3ce-036d-4f3d-b069-223a63f5fbc5)

At this point, you might wonder “Why not use the built-in Blob storage trigger instead?”. The key benefit of the Event Grid trigger is it eliminates the need for continuous polling, allowing Logic App to operate in a reactive mode, i.e. it springs into action only when a new Blob is created in Blob Storage, resulting in a reduction of resource consumption and costs.

4. Click on + New Step and search for `Get blob content using path (V2)` to get blob container content.
   
    * Click in the **Storage account name or blob endpoint** field and select the container we want to monitor from the options that appear.
    * Select **Blob path** field, click on the filed and select **Expression** option. Insert **split(triggerBody()?['subject'],'/')[4]** expression in the **Expression** field. This expression extracts a specific segment from a string by splitting it based on a delimiter, in this case, the forward slash (/).
    

    ```JSON
     "Get_blob_content_using_path_(V2)": {
                        "inputs": {
                            "host": {
                                "connection": {
                                    "name": "@parameters('$connections')['azureblob']['connectionId']"
                                }
                            },
                            "method": "get",
                            "path": "/v2/datasets/@{encodeURIComponent(encodeURIComponent('AccountNameFromSettings'))}/GetFileContentByPath",
                            "queries": {
                                "inferContentType": true,
                                "path": "@{split(triggerBody()?['subject'],'/')[4]}/@{split(triggerBody()?['subject'],'/')[6]}",
                                "queryParametersSingleEncoded": true
                            }
                        },
                        "runAfter": {},
                        "type": "ApiConnection"
                    }
    ```

    * The output of this action will be similar to this:

    ```JSON
   {
    "statusCode": 200,
    "headers": {
        "Cache-Control": "no-store, no-cache",
        "Pragma": "no-cache",
        "ETag": "\"0x8DBF29016616CCB\"",
        "Location": "https://logic-apis-eastus.azure-apim.net/apim/azureblob/df30d08cb735432db8adb9d7faa35b9a/v2/datasets/AccountNameFromSettings/GetFileContentByPath?inferContentType=True&path=logsjson%2ff7688ff2-11f9-49dc-af5d-c4de06a458d9.json&queryParametersSingleEncoded=True",
        "Set-Cookie": "ARRAffinity=fa5ce4b13622b0d3617b4398e823c470b20f49c8905d33671c1e72b454b4c01b;Path=/;HttpOnly;Secure;Domain=azureblob-eus.azconn-eus-002.p.azurewebsites.net,ARRAffinitySameSite=fa5ce4b13622b0d3617b4398e823c470b20f49c8905d33671c1e72b454b4c01b;Path=/;HttpOnly;SameSite=None;Secure;Domain=azureblob-eus.azconn-eus-002.p.azurewebsites.net",
        "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
        "x-ms-request-id": "16c8d988-7e29-421f-8729-b0efb224dad5",
        "X-Content-Type-Options": "nosniff",
        "X-Frame-Options": "DENY",
        "x-ms-connection-parameter-set-name": "keyBasedAuth",
        "Timing-Allow-Origin": "*",
        "x-ms-apihub-cached-response": "false",
        "x-ms-apihub-obo": "false",
        "Date": "Fri, 01 Dec 2023 17:08:06 GMT",
        "Content-Length": "295",
        "Content-Type": "application/octet-stream",
        "Expires": "-1"
    },
    "body": {
        "$content-type": "application/octet-stream",
        "$content": "eyJDbGllbnRJUCI6IjI2MDQ6YTk0MDozMDE6MjI1OjA6MTQ6OiIsIkNsaWVudFJlcXVlc3RIb3N0IjoidGVzdHczLm1vbmVyaXMuY29tIiwiQ2xpZW50UmVxdWVzdE1ldGhvZCI6IkdFVCIsIkNsaWVudFJlcXVlc3RVUkkiOiIvIiwiRWRnZUVuZFRpbWVzdGFtcCI6IjIwMjMtMTEtMjRUMDA6NDI6MTBaIiwiRWRnZVJlc3BvbnNlQnl0ZXMiOjQ5NTAsIkVkZ2VSZXNwb25zZVN0YXR1cyI6NDAzLCJFZGdlU3RhcnRUaW1lc3RhbXAiOiIyMDIzLTExLTI0VDAwOjQyOjEwWiIsIlJheUlEIjoiODJhZDljNDU1ODg5MDQ5OCJ9Cg=="
    }
}
    ```
![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/fb9077f7-502f-4b07-a54f-97f327c6d82d)


5. Create a new step and search for `Compose` action. TThis expression is often employed when dealing with binary or encoded data, such as files stored in blobs, where the content needs to be decoded from base64 to a readable text format for further processing, transformation, or analysis within the Logic App workflow.

    * In the **Inputs** field, select **Expression** option. Insert **base64ToString(body('Get_blob_content_using_path_(V2)').$content)** expression in the **Expression** field. The Logic App expression **base64ToString(body('Get_blob_content_using_path_(V2)').$content)** within a Compose action is used to convert base64-encoded content obtained from a blob into a readable string format.

    ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/bff99a35-340a-4505-a423-67ee3da98c2c)

      ```JSON
            "Compose": {
                              "inputs": "@base64ToString(body('Get_blob_content_using_path_(V2)').$content)",
                              "runAfter": {
                                  "Get_blob_content_using_path_(V2)": [
                                      "Succeeded"
                                  ]
                              },
                              "type": "Compose"
                          }
                      }
          ```

6.  Create a new step and search for `Send Event` action. When you add an Event Hubs trigger or action for the first time, you're prompted to create a connection to your event hub. When you're prompted, choose one of the following options:

   * Provide the following connection information:
   ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/13fc4148-26a6-4300-b4a3-826e8306b3f6)

   * Select the Event Hubs policy to use, if not already selected, and then select Create.

   Screenshot showing the provided connection information with "Create" selected.
   ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/cf7aa814-5914-4b32-a67d-57445f523808)

   * In the action, provide information about the events that you want to send.
    ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/708d9092-bce6-4c05-841e-c2579bc2258f)

   * In the **Event Hub name** field select the event hub where you want to send the event
    ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/80db6227-91d0-4c2f-a555-09bd7b9c9c93)

   * In the **Content** field select the outputr of the *Compose** action and send the content of the blob container to Event Hub:
    ![image](https://github.com/dcucereavii-ms/log-processing-with-logicapp/assets/82041010/28a1f432-659e-4399-9ad2-7927cf827dfb)

 ```JSON
      "Send_event": {
                        "inputs": {
                            "body": {
                                "ContentData": "@{base64(outputs('Compose'))}"
                            },
                            "host": {
                                "connection": {
                                    "name": "@parameters('$connections')['<eventhub_connection_name>']['connectionId']"
                                }
                            },
                            "method": "post",
                            "path": "/@{encodeURIComponent('<storage_account_name>')}/events"
                        },
                        "runAfter": {
                            "Compose": [
                                "Succeeded"
                            ]
                        },
                        "type": "ApiConnection"
                    }
                }
    ```
