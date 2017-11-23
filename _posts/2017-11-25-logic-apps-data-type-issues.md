---
title:  "How to handle data type issues of data from Azure Storage blobs"
date:   2017-11-23 10:05:00 +0100
categories: Azure AppServices
---

# Summary

Handling data from Azure Storage blobs is not straightforward. The return value is binary (`application/octet-stream`) at first and needs to be casted into a data type you want to process; in our case into `application/json`.

This write-up is an easy to follow and real walk through of errors beginners may encounter handling Azure Storage blobs in Azure Logic Apps. It has happened to me.

# Requirements

As soon a new file (blob) in an Azure Storage container arrives this file should be processed by an Azure Function app.

# Preparing the Playground

1) Create a new Azure Storage Account.  
2) Create a new container in the storage account.  
3) Create a new Azure Logic App.  
4) Design the Logic App.  

![Azure Portal: Create an Azure Logic App](/images/2017-11-logic-apps/2017-11-create-logic-app.png)

# Challenges

A first draft could look like this.

![Azure Logic App: 1st draft](/images/2017-11-logic-apps/2017-11-logic-app-design-1.png)

With this configuration we have three steps.  

1) When one or more blobs are added or modified (metadata only) (Preview)  
  * Created a connection to the newly created storage account.  
  * Configured to look in the container `files`.  
  * Configured to look for changes every 10 seconds.  
2) Get blob content  
  * Gets the content of a blob by ID. (There is a similar action called "Get blob content using path" if you need to get blob contents via paths.)  
3) Azure Function  
  * We call an Azure Function with the content of the blob.  

Unfortunately, this configuration does not work because of two errors:
1) there is no array of blobs  
2) data type issues

# Trigger & Test the Logic App

I use the [Microsoft Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/) to upload files into the container of my Azure Storage account to trigger my Azure Logic App. In every try I increment the number of my test file. The test file contains a simple JSON which can be interpreted by my Azure Function.

```json
{
	"first":"First Name",
	"last":"Last Name"
}
```

![Azure Storage Explorer: upload first test file](/images/2017-11-logic-apps/2017-11-azure-storage-explorer-upload-1.png)

After each upload I go back into the Azure Portal to look for new **trigger events** and shortly afterwards for new **run events**. To avoid flooding of my trigger history I disable the logic app after each upload to inspect the run results. Before each new try I enable the logic app again.

![Azure Logic App: trigger and run history](/images/2017-11-logic-apps/2017-11-logic-app-history-1.png)

As you can see our first configuration of the Azure Logic App did not run successfully. Let's inspect the first error!

# Error 1: No Array

![Azure Logic App: run history](/images/2017-11-logic-apps/2017-11-logic-app-run-history-1.png)

The return value of the first step is no array! If we look at the raw data, we see in the `body` that there is no array. Maybe this is an exception because we uploaded only a single file? Try again with two files at the same time.

```json
{
    "headers": {
        "Pragma": "no-cache",
        "Transfer-Encoding": "chunked",
        "Retry-After": "15",
        "Vary": "Accept-Encoding",
        "x-ms-request-id": "4774fda2-2cf4-4f8b-bdfd-fb6a7ab4b483",
        "Timing-Allow-Origin": "*",
        "Cache-Control": "no-cache",
        "Date": "Tue, 21 Nov 2017 14:05:46 GMT",
        "Location": "https://logic-apis-westeurope.azure-apim.net/apim/azureblob/899c302122154f6191b7ca34b8528833/datasets/default/triggers/batch/onupdatedfile?folderId=L2ZpbGVz&maxFileCount=10&triggerstate=eyJGaWxlSWQiOiIiLCJTdGF0dXMiOjEsIldpbmRvd1N0YXJ0VGltZSI6IjIwMTctMTEtMjFUMTQ6MDU6MjguMDAxWiIsIldpbmRvd0VuZFRpbWUiOiIyMDE3LTExLTIxVDE0OjA1OjI4LjAwMVoiLCJMYXN0UHJvY2Vzc2VkRmlsZVRpbWUiOiIyMDE3LTExLTIxVDE0OjA1OjI4LjAwMVoiLCJMYXN0Q29uZmxpY3RUaW1lIjoiMjAxNy0xMS0yMVQxNDowNTo0MC42MTI0MThaIn0%3d",
        "Set-Cookie": "ARRAffinity=6f8192aeb45a330cdb5f642ecc217dcd76db8eb70939a923860f458883127859;Path=/;HttpOnly;Domain=azureblob-logic-cp-westeurope.logic-ase-westeurope.p.azurewebsites.net",
        "X-AspNet-Version": "4.0.30319",
        "X-Powered-By": "ASP.NET",
        "Content-Type": "application/json",
        "Expires": "-1",
        "Content-Length": "675"
    },
    "body": {
        "Id": "L2ZpbGVzL25ld2ZpbGUwMDMuanNvbg==",
        "Name": "newfile003.json",
        "DisplayName": "newfile003.json",
        "Path": "/files/newfile003.json",
        "LastModified": "2017-11-21T14:05:28Z",
        "Size": 46,
        "MediaType": "application/octet-stream",
        "IsFolder": false,
        "ETag": "\"0x8D530E8EB65A61F\"",
        "FileLocator": "L2ZpbGVzL25ld2ZpbGUwMDMuanNvbg==",
        "LastModifiedBy": null
    }
}
```

Now, I upload two files at the same time.

![Azure Storage Explorer: upload two test files](/images/2017-11-logic-apps/2017-11-azure-storage-explorer-upload-2-3.png)

Surprisingly, the Logic App gets triggered for each new file separately.  

![Azure Logic App: trigger and run history](/images/2017-11-logic-apps/2017-11-logic-app-history-2-3.png)

So my assumption was wrong that the action `When one or more blobs are added or modified (metadata only) (Preview)` returns an array. For me the property `Number of blobs` was somewhat misleading that the action would return an array.

We can resolve this error easily by removing the for-each-loop. We can design the flow of the Logic App in such a way that the Logic App gets triggered for each new or modified blob separately.

![Azure Logic App: 2nd draft](/images/2017-11-logic-apps/2017-11-logic-app-design-2.png)

Let's try again by uploading a new file. Again, we see in the run history an error. This time the error is at the third step. The first two steps are running successfully, now. We were successful in getting the content of the blob which has triggered the Logic App. So far, so good! Let's explore the new error!

![Azure Logic App: run history](/images/2017-11-logic-apps/2017-11-logic-app-run-history-2a.png)

Even the raw data of the 2nd step looks fine.

```json
{
    "statusCode": 200,
    "headers": {
        "Pragma": "no-cache",
        "x-ms-request-id": "68bfd114-2db8-45d8-aec3-5c3d2d1cfa11",
        "Timing-Allow-Origin": "*",
        "Cache-Control": "no-cache",
        "Date": "Tue, 21 Nov 2017 14:16:30 GMT",
        "ETag": "\"0x8D530EA6B3DA5D9\"",
        "Location": "https://logic-apis-westeurope.azure-apim.net/apim/azureblob/899c302122154f6191b7ca34b8528833/datasets/default/files/L2ZpbGVzL25ld2ZpbGUwMDQuanNvbg%253D%253D/content?inferContentType=True",
        "Set-Cookie": "ARRAffinity=785f4334b5e64d2db0b84edcc1b84f1bf37319679aefce206b51510e56fd9770;Path=/;HttpOnly;Domain=azureblob-logic-cp-westeurope.logic-ase-westeurope.p.azurewebsites.net",
        "X-AspNet-Version": "4.0.30319",
        "X-Powered-By": "ASP.NET",
        "Content-Length": "46",
        "Content-Type": "application/octet-stream",
        "Expires": "-1"
    },
    "body": {
        "$content-type": "application/octet-stream",
        "$content": "ewoJImZpcnN0IjoiRmlyc3QgTmFtZSIsCgkibGFzdCI6Ikxhc3QgTmFtZSIKfQ=="
    }
}
```

# Error 2: Data Type Issues

The Azure Function action in the third step throws the error `UnsupportedMediaType` with the message: "The WebHook request must contain an entity body formatted as JSON." That error may be confusing at first because our file contains pure JSON data. A look at the content type reveals that the Logic App does not know that we handle JSON data, instead it says something of `application/octet-stream`, which is a binary data type.

![Azure Logic App: run history](/images/2017-11-logic-apps/2017-11-logic-app-run-history-2b.png)

The Azure Function gets the following raw input:

```json
{
    "function": {
        "name": "FunctionAppHackShop/ProcessBlob2",
        "id": "/subscriptions/fb5f59d4-570b-4979-8aa1-787abe3c65c1/resourceGroups/logic-app-heute/providers/Microsoft.Web/sites/FunctionAppHackShop/functions/ProcessBlob2",
        "type": "Microsoft.Web/sites/functions"
    },
    "body": {
        "$content-type": "application/octet-stream",
        "$content": "ewoJImZpcnN0IjoiRmlyc3QgTmFtZSIsCgkibGFzdCI6Ikxhc3QgTmFtZSIKfQ=="
    }
}
```
The raw output of the Azure Function action looks like this.

```json
{
    "statusCode": 415,
    "headers": {
        "Pragma": "no-cache",
        "Cache-Control": "no-cache",
        "Date": "Tue, 21 Nov 2017 14:16:35 GMT",
        "Set-Cookie": "ARRAffinity=eb5760d533a5ec6fa8bfcabd58a9f9fa34e9daab8b6fde8c63430ffdc0269857;Path=/;HttpOnly;Domain=functionapphackshop.azurewebsites.net",
        "Server": "Microsoft-IIS/8.0",
        "X-AspNet-Version": "4.0.30319",
        "X-Powered-By": "ASP.NET",
        "Content-Length": "80",
        "Content-Type": "application/json; charset=utf-8",
        "Expires": "-1"
    },
    "body": {
        "Message": "The WebHook request must contain an entity body formatted as JSON."
    }
}
```
And for reference the function stub of the Azure Function looks like this:

```C#
using System.Net;
using Newtonsoft.Json;
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Azure.WebJobs.Host;

namespace FunctionAppShopHack
{
    public static class ProcessBlob2
    {
        [FunctionName("ProcessBlob2")]
        public static async Task<object> Run([HttpTrigger(WebHookType = "genericJson")]HttpRequestMessage req, TraceWriter log)
        {
            /* body omitted */
        }
    }
}
```

## Convert into JSON data type

The documentation states, that Logic Apps can handle natively `application/json` and `text/plain` (see [Handle content types in logic apps](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-content-type)). As we have already JSON data we can use the function `@json()` to cast the data type to `application/json`.

![Azure Logic App: 3rd draft](/images/2017-11-logic-apps/2017-11-logic-app-design-3a.png)

Unfortunately, this approach cannot be saved by the Logic App Designer.

Error message
> **Save logic app failed**
> Failed to save logic app logicapp. The template validation failed: 'The template action 'ProcessBlob2' at line '1' and column '43845' is not valid: "The template language expression 'json(@{body('Get_blob_content')})' is not valid: the string character '@' at position '5' is not expected.".'.

![Azure Logic App: 3rd draft](/images/2017-11-logic-apps/2017-11-logic-app-design-3b.png)

Fortunately, this only a small shortcoming of the Azure Logic App Designer. We need to look at the configuration in code view. For that reason click on the tree dots `...` in the upper right corner of the Azure Function action and select `Peek code` in the menu.

![Azure Logic App: 3rd draft](/images/2017-11-logic-apps/2017-11-logic-app-design-3c.png)

```json
{
    "inputs": {
        "body": "@json(@{body('Get_blob_content')})",
        "function": {
            "id": "/subscriptions/fb5f59d4-570b-4979-8aa1-787abe3c65c1/resourceGroups/logic-app-heute/providers/Microsoft.Web/sites/FunctionAppHackShop/functions/ProcessBlob2"
        }
    }
}
```

![Azure Logic App: 2nd draft](/images/2017-11-logic-apps/2017-11-logic-app-design-3d.png)

We have to change the evaluation in the body property. It must not contain more than one expression wrapper `@()`. The documentation does not say explicitly how to nest expressions (see [https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language#expressions](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language#expressions)), but after some trial and error, we know, we just need to remove the nested expression wrapper `@{`and `}` and leave everything else.

```
@json(body('Get_blob_content'))
```

![Azure Logic App: 3rd draft](/images/2017-11-logic-apps/2017-11-logic-app-design-3e.png)

Check again, if it's working. Upload a new file.

We check the run history, again, and all actions did run successfully. Let's check the raw input and output.

![Azure Logic App: run history](/images/2017-11-logic-apps/2017-11-logic-app-run-history-3.png)

Raw input:

```json
{
    "function": {
        "name": "FunctionAppHackShop/ProcessBlob2",
        "id": "/subscriptions/fb5f59d4-570b-4979-8aa1-787abe3c65c1/resourceGroups/logic-app-heute/providers/Microsoft.Web/sites/FunctionAppHackShop/functions/ProcessBlob2",
        "type": "Microsoft.Web/sites/functions"
    },
    "body": {
        "first": "First Name",
        "last": "Last Name"
    }
}
```
Raw output:

```json
{
    "statusCode": 200,
    "headers": {
        "Pragma": "no-cache",
        "Vary": "Accept-Encoding",
        "Cache-Control": "no-cache",
        "Date": "Tue, 21 Nov 2017 14:33:38 GMT",
        "Set-Cookie": "ARRAffinity=eb5760d533a5ec6fa8bfcabd58a9f9fa34e9daab8b6fde8c63430ffdc0269857;Path=/;HttpOnly;Domain=functionapphackshop.azurewebsites.net",
        "Server": "Microsoft-IIS/8.0",
        "X-AspNet-Version": "4.0.30319",
        "X-Powered-By": "ASP.NET",
        "Content-Type": "application/json; charset=utf-8",
        "Expires": "-1",
        "Content-Length": "42"
    },
    "body": {
        "greeting": "Hello First Name Last Name!"
    }
}
```

There is a difference in the input data of the Azure Function action, as there is no explicit content type, just pure JSON data.

# Final Logic App

The final Logic App looks like this in the designer. Unfortunately, you don't see all expressions. You need to peek inside the code, as seen in the step before.

![Azure Logic App: 4th draft](/images/2017-11-logic-apps/2017-11-logic-app-design-4a.png)

To see everything switch to code view. That's not nice to design, but it's good enough to check our configuration.

![Azure Logic App: 4th draft](/images/2017-11-logic-apps/2017-11-logic-app-design-4b.png)

```json
{
    "$connections": {
        "value": {
            "azureblob": {
                "connectionId": "/subscriptions/fb5f59d4-570b-4979-8aa1-787abe3c65c1/resourceGroups/hackfestwriteup/providers/Microsoft.Web/connections/azureblob",
                "connectionName": "azureblob",
                "id": "/subscriptions/fb5f59d4-570b-4979-8aa1-787abe3c65c1/providers/Microsoft.Web/locations/westeurope/managedApis/azureblob"
            }
        }
    },
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Get_blob_content": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azureblob']['connectionId']"
                        }
                    },
                    "method": "get",
                    "path": "/datasets/default/files/@{encodeURIComponent(encodeURIComponent(triggerBody()?['Id']))}/content",
                    "queries": {
                        "inferContentType": true
                    }
                },
                "runAfter": {},
                "type": "ApiConnection"
            },
            "ProcessBlob2": {
                "inputs": {
                    "body": "@json(body('Get_blob_content'))",
                    "function": {
                        "id": "/subscriptions/fb5f59d4-570b-4979-8aa1-787abe3c65c1/resourceGroups/logic-app-heute/providers/Microsoft.Web/sites/FunctionAppHackShop/functions/ProcessBlob2"
                    }
                },
                "runAfter": {
                    "Get_blob_content": [
                        "Succeeded"
                    ]
                },
                "type": "Function"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "triggers": {
            "When_one_or_more_blobs_are_added_or_modified_(metadata_only)": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azureblob']['connectionId']"
                        }
                    },
                    "method": "get",
                    "path": "/datasets/default/triggers/batch/onupdatedfile",
                    "queries": {
                        "folderId": "L2ZpbGVz",
                        "maxFileCount": 10
                    }
                },
                "metadata": {
                    "L2ZpbGVz": "/files"
                },
                "recurrence": {
                    "frequency": "Second",
                    "interval": 10
                },
                "splitOn": "@triggerBody()",
                "type": "ApiConnection"
            }
        }
    }
}
```
# Helpful Links

## Azure Logic Apps

* https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-content-type
* https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language#expressions
* https://stackoverflow.com/questions/43598188/azure-logic-apps-get-blob-content-setting-content-type
* https://stackoverflow.com/questions/38748561/parsing-json-array-in-azure-logic-app-from-base64-encoded-string-to-use-in-for-e
* https://social.technet.microsoft.com/wiki/contents/articles/33999.azure-logic-apps-calling-azure-functions-inside-logic-apps.aspx

## Azure Function Apps

* https://stackoverflow.com/questions/10399324/where-is-httpcontent-readasasync#_=_
* https://stackoverflow.com/questions/37301379/azure-logic-app-is-not-displaying-functions-when-show-azure-functions-in-the-sam