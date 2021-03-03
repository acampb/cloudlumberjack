---
title: Azure AutoTagger
author: Andrew Campbell
date: 2021-01-06
categories: []
tags: [Azure, ARM Template, DevOps, PowerShell, Serverless, Functions]
pin: false
image: /assets/img/autotagger/autotagger.png
description: Quickly deploy a serverless solution to automate tagging of Azure resources with last modified data
---

# Overview

Azure AutoTagger is a lightweight, low-cost serverless solution that can easily be deployed to an Azure subscription. Once deployed Azure AutoTagger monitors for `ResourceWriteSucess` events within the subscription and triggers an Azure Function to automatically apply a `LastModifiedTimestamp` and `LastModifiedBy` tag.

![azureautotagger](/assets/img/autotagger/autotagger.png)

* **https://github.com/acampb/AzureAutoTagger**: Contains the ARM template code to deploy the infrastructure and role assignments to the subscription

* **https://github.com/acampb/AzureAutoTaggerFunction**: Contains the Azure Function PowerShell code

![tags](/assets/img/autotagger/tags.png)

## Deployment

> Important: You must have **Owner** permissions on the subscription you intend to deploy this to. The template will create a managed identity and assign it to the `Reader` and `Tag Contributor` roles.

Use the **Deploy to Azure** button to easily deploy this solution in a subscription

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Facampb%2FAzureAutoTagger%2Fmain%2Fazuredeploy.json)

-- OR --

1. Clone the GitHub repo locally

```shell
git clone https://github.com/acampb/AzureAutoTagger.git
```

2. Initiate an ARM Template deployment with Azure PowerShell or Azure CLI

Azure PowerShell:

```powershell
New-AzDeployment -Location "East US" -TemplateFile ".\azuredeploy.json" -resourceGroupName "rg-autotagger" -Verbose
```

Azure CLI:

```python
az deployment create --location "West US" --template-file ".\azuredeploy.json" --parameters resourceGroupName=rg-autotagger
```

## Event Grid

The solution starts by creating an Event Grid System Topic connected to the Azure subscription it is deployed in. An Event Grid Subscription is then configured to consume the events emitted by the Azure subscription. The Event Grid Subscription only consumes events of the type `ResourceWriteSuccess`, so we are only sending events to this subscription when a resource is written (created or changed).

The events emitted by the Azure Subscription are standard JSON payloads describing what resource was changed, and the authentication claim of the identity initiating performing the resource write operation. Below is a sample event JSON from the Azure documentation:

```json
[{
  "subject": "/subscriptions/{subscription-id}/resourcegroups/{resource-group}/providers/Microsoft.Storage/storageAccounts/{storage-name}",
  "eventType": "Microsoft.Resources.ResourceWriteSuccess",
  "eventTime": "2018-07-19T18:38:04.6117357Z",
  "id": "4db48cba-50a2-455a-93b4-de41a3b5b7f6",
  "data": {
    "authorization": {
      "scope": "/subscriptions/{subscription-id}/resourcegroups/{resource-group}/providers/Microsoft.Storage/storageAccounts/{storage-name}",
      "action": "Microsoft.Storage/storageAccounts/write",
      "evidence": {
        "role": "Subscription Admin"
      }
    },
    "claims": {
      "aud": "{audience-claim}",
      "iss": "{issuer-claim}",
      "iat": "{issued-at-claim}",
      "nbf": "{not-before-claim}",
      "exp": "{expiration-claim}",
      "_claim_names": "{\"groups\":\"src1\"}",
      "_claim_sources": "{\"src1\":{\"endpoint\":\"{URI}\"}}",
      "http://schemas.microsoft.com/claims/authnclassreference": "1",
      "aio": "{token}",
      "http://schemas.microsoft.com/claims/authnmethodsreferences": "rsa,mfa",
      "appid": "{ID}",
      "appidacr": "2",
      "http://schemas.microsoft.com/2012/01/devicecontext/claims/identifier": "{ID}",
      "e_exp": "{expiration}",
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname": "{last-name}",
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname": "{first-name}",
      "ipaddr": "{IP-address}",
      "name": "{full-name}",
      "http://schemas.microsoft.com/identity/claims/objectidentifier": "{ID}",
      "onprem_sid": "{ID}",
      "puid": "{ID}",
      "http://schemas.microsoft.com/identity/claims/scope": "user_impersonation",
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier": "{ID}",
      "http://schemas.microsoft.com/identity/claims/tenantid": "{ID}",
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name": "{user-name}",
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn": "{user-name}",
      "uti": "{ID}",
      "ver": "1.0"
    },
    "correlationId": "{ID}",
    "resourceProvider": "Microsoft.Storage",
    "resourceUri": "/subscriptions/{subscription-id}/resourcegroups/{resource-group}/providers/Microsoft.Storage/storageAccounts/{storage-name}",
    "operationName": "Microsoft.Storage/storageAccounts/write",
    "status": "Succeeded",
    "subscriptionId": "{subscription-id}",
    "tenantId": "{tenant-id}"
  },
  "dataVersion": "2",
  "metadataVersion": "1",
  "topic": "/subscriptions/{subscription-id}"
}]
```

The event JSON contains some interesting data that we will utilize later within our Azure function, notably the `ResourceUri` and some claim information about the user or service principal performing the operation. The Event Subscription is configured to trigger our Azure Function to execute on a new event and will pass this JSON data object to the Function as a parameter in our PowerShell script.

You may be thinking "wait, wont the Azure Function writing tags create an endless loop of insanity?". Yes, yes it will, unless we configure some additional filtering. Within the Event Grid Subscription configure we also create an advanced filter to exclude the event if the `$data.claims.appid` matches the appid of the Azure Function itself.

## Azure Function

The Azure Function executes PowerShell code which parses the JSON data provided from the Event Grid for which user or service principal modified the Azure resource, and what the Resource Uri is. The script then creates a hashtable and updates the tags on the resource. The code performs an `Update-AzTag -Merge` operation so any existing tags are preserved.

These additional resources are deployed in support of the Azure Function:

* **Storage Account**: A storage account is required for Azure Functions to operate.
* **App Service Plan**: The App Service Plan is the hosting plan for the Azure Function. The plan provides the compute and memory for the function, and controls additional functionality. The deployment template defaults to the `Consumption` plan for the lowest possible cost, but this could be upgraded later if required.
* **Application Insights**: Application Insights is deployed and configured in order to provide troubleshooting and log streaming capabilities.
* **User Assigned Managed Identity**: A User assigned managed identity is created and assigned to the Azure Function. When the Function's PowerShell code is executed it is authenticated to the Azure subscription using this identity. This managed identity is also assigned to the `Reader` and `Tag Contributor` RBAC roles.

## Authentication

Getting authentication working correctly with this solution actually proved to be a bit of a challenge I wasn't expecting. Initially I created a System Assigned Managed Identity with the Function app, and added it to the `Reader` and `Tag Contributor` roles. The PowerShell runtime for Azure Functions actually handles this scenario natively very well. When your function is started it executes the `profile.ps1` file, and the default file has this section of code to authenticate to Azure built in.

```powershell
# Authenticate with Azure PowerShell using MSI.
# Remove this if you are not planning on using MSI or Azure PowerShell.
if ($env:MSI_SECRET) {
    Disable-AzContextAutosave -Scope Process | Out-Null
    Connect-AzAccount -Identity
}
```

As soon as you create a system assigned managed identity for the Function App it will automatically create an environmental variable named `MSI_SECRET`. When this variable is present the `profile.ps1` authenticates to Azure using that identity.

I used this approach for several weeks while I was developing and testing and it worked fine to get the Function authenticated. However, I was noticing the Function App performing the `Update-AzTag` operation was itself a `ResourceWriteSuccess` which caused the Event Grid to emit another event and trigger another execution of the Function. Initially I just focused on events where a user email address was present to use as the `LastModifiedBy` tag, but I also wanted to support resource changes made by other sources (ARM template deployments, Terraform, Ansible, Azure Policy, etc).

Those sources were identifiable by `Application Id` and `Object Id`. I figured it would be easy to filter the `ResourceWriteSuccess` events caused by the Function itself from the Event Grid Subscription. I knew I could reference the `ObjectId` of a System Assigned Managed Identity from the ARM template deployment, I just needed to filter my events based on that. The `ObjectId` was present in JSON event data at `data.claims.http://schemas.microsoft.com/identity/claims/objectidentifier`. Unfortunately the Event Grid Advanced Filters have a limitation where keys with a . (dot) character cannot be used for a filter. So back to the drawing board.

The JSON event data contains another property `data.claims.appid` which doesn't have dots in the key name, so I started down the rabbit hole of trying to query the Application Id of the System Assigned Managed Identity. I could not find any way to query this natively with ARM template `reference` functions as the service principal itself is an Azure Active Directory object, and not an Azure subscription resource. I was able to get an ARM template Deployment Script to execute a script to query Azure Active Directory for the App Id, but the Deployment Script itself the needed an identity with permissions to query Azure AD.

I eventually settled on just creating a User Assigned Managed Identity directly with the ARM template deployment. That managed identity does expose it's Application Id within ARM so I can then configure the Function App to use this identity, and automatically create the advanced filter within the Event Grid Subscription to exclude events with this App Id.

Now my ARM template was looking good, but some additional testing showed that my Function App was now no longer authenticating to Azure. This led me back to the bit of code in `profile.ps1` that handles authenticating to Azure. I confirmed that the environment variable `MSI_SECRET` still existed, but it would not authenticate.

```shell
ERROR: ManagedIdentityCredential authentication failed: Service request failed.Status: 400 (Bad Request)
```

This error led me to some [issues for `azure-sdk-for-net` on GitHub](https://github.com/Azure/azure-sdk-for-net/issues/13564); suggesting that what was missing was another environmental variable named `AZURE_CLIENT_ID` set to the managed identities' client id. This is due to the fact that a resource (Azure Function in this case) can have multiple user-assigned managed identities, and we need to explicitly tell it which to authenticate with. After creating `AZURE_CLIENT_ID` as an App Setting in the Function App (which are exposed as environmental variables at runtime), I was still recieving the same error.

The final piece in the puzzle was the `Connect-AzAccount` cmdlet in `profile.ps1` itself. When using this cmdlet with a user assigned managed identity you need to provide the `-AccountId` parameter with the identity client id as well. I updated `profile.ps1` with the following code and everything came together and authenticated. I'm still utilizing the `AZURE_CLIENT_ID` variable set on the Function App as a method to pass the client id to the function code when it executes.



```powershell
# Authenticate with Azure PowerShell using MSI.
# Remove this if you are not planning on using MSI or Azure PowerShell.
if ($env:MSI_SECRET) {
    Disable-AzContextAutosave -Scope Process | Out-Null
    Connect-AzAccount -Identity -AccountId $env:AZURE_CLIENT_ID
}
```

## ARM Template Deployment

This ARM Template is intended to perform a subscription level deployment. It will perform several linked deployments to deploy and configure all of the resources required in the solution.

* Create a new Resource Group with the name specified in the parameter `resourceGroupName`. Defaults to `rg-autotagger` if no other value is specified

* Perform a linked template deployment to create the Event Grid and Function App resources. Uses [https://github.com/acampb/AzureAutoTagger/blob/main/eventgridfunction.json](https://github.com/acampb/AzureAutoTagger/blob/main/eventgridfunction.json)

    * Create the User Assigned Managed Identity

    * Create the App Service plan

    * Create the Storage Account

    * Create the Function App

    * Configure the Function App source control to deploy the PowerShell function from the application repo: [**https://github.com/acampb/AzureAutoTaggerFunction**](https://github.com/acampb/AzureAutoTaggerFunction)

    * Create the Event Grid System Topic using the subscription as a source

    * Create the Event Grid Subscription, configured to include only `ResourceWriteSuccess` events, and an advanced filter to exclude events from the Managed Identity App Id

    * Create the Application Insights

    * Output the Managed Identity Principal Id

* Perform a linked template deployment to assign the Managed Identity to the `Reader` role. Uses [https://github.com/acampb/AzureAutoTagger/blob/main/rbac.json](https://github.com/acampb/AzureAutoTagger/blob/main/rbac.json)

* Perform a linked template deployment to assign the Managed Identity to the `Tag Contributor` role. Uses [https://github.com/acampb/AzureAutoTagger/blob/main/rbac.json](https://github.com/acampb/AzureAutoTagger/blob/main/rbac.json)
