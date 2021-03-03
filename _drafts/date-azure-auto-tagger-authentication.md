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
