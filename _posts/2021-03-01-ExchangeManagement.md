---
layout: post
title: Exchange Management working title
excerpt: "ToDo"
date: 2021-03-01
---

# Introduction

The release of Exchange Management Shell V2.0.4 comes with support for MacOS and Linux. With certificate based authetentication against the Exchange Online management endpoints released in a previous version,  let's have a look how we can leverage dynamic retreival of a certificate from an Azure KeyVault and use it inside an Azure PowerShell function to connect towards Exchange Online to do some automated tasks. This article outlines the setup and discusses some caveats I encountered during the deployment in my lab environment.

# Main Content

Before we dive into the Azure Function, we first need to ensure that all the required components are configured in our environment. This means that we need to have an Azure KeyVault and an app registration inside Azure Active Directory before we can test out inside our Azure Function. *Note: some parts in this article are done using Azure CLI while others are executed manually.*

## Prepare Azure Key Vault
All journey's start somewhere and in this case we first need to do create an Azure KeyVault using Azure Cloud Shell:

```CLI 
az group create --name "RG-AzureFunctionDemo" -l "westeurope"
az keyvault create --name "kvvddesignlab1234" --resource-group "RG-AzureFunctionDemo" --location "westeurope"
```

The outcome of these commands is that an Azure Key Vault is created inside a resource group. When an Azure Key vault is created with above commend, you are automatically assigned an access policy that allows you to manage secrets inside the Keyvault. At this point we are ready to generate the certificate that will be used to authenticate against Exchange Online. For this purpose, we can generate a self-signed certificate inside KeyVault going to our KeyVault resource in the Azure Portal (portal.azure.com > Resource Groups > YourResourceGroup > YourKeyVaultName > Certificates > Generate / Import). For this example, I used the following settings:

* Name: ExchangeOnlineManagement
* Common Name: CN=ExchangeOnlineManagement
* Content Type: PKCS #12
* Lifetime action type: E-mail all contacts at a given percentage lifetime

![Create Certificate](/assets/posts/20210302-02/CreateExchangeOnlineCertificate.png){:class="img-responsive"}

We need the public portion of the certificate, so we need to download it by executing the following command:
```
az keyvault certificate download --vault-name kvvddesignlab1234 -n ExchangeOnlineManagement -f cert.crt -e DER
```


## Prepare Azure Active Directory
Time to move on towards preparing Azure Active Directory where an app registration must be done that will be used to authenticate unattended against the Exchange Online management endpoints. We start with creating an app registration inside Azure Active Directory by executing the following Azure CLI command:

```CLI 
az ad app create --display-name AZF-ExchangeOnlineMGT --native-app
```

![Create Certificate](/assets/posts/20210302-02/CreateAppRegistration.png){:class="img-responsive"}

Take a note of the appID value, we will need this later on when we authenticate against Exchange Online via PowerShell code.

Once the application is created, we need to upload the public portion of our certificate towards the App registrations "Certificate & Secrets" so that we can authenticate using the generated certificate. You can do this by going to the certificate section on your app registration Azure Active Directory management portal > App registrations > All Applications > YourApplicationName > Certificates & secrets and click the "Upload Certificate" button where you select the certificate cert.crt file downloaded in the previous step.
![Create Certificate](/assets/posts/20210302-02/UploadCertificate.png){:class="img-responsive"}


The next configuration is allowing access to the Exchange Online Management API, we can configure this in the "API section" of our app registration. We an do this by changing the app's manifest file. So head over to your Azure Active Directory management portal > App registrations > All Applications > YourApplicationName > Manifest and modify the requiredResourceAccess to reflect the following configuration:
```
"requiredResourceAccess": [
   {
      "resourceAppId": "00000002-0000-0ff1-ce00-000000000000",
      "resourceAccess": [
         {
            "id": "dc50a0fb-09a3-484d-be87-e023b12c6440",
            "type": "Role"
         }
      ]
   }
],
```

As this is an application permission, we need to execute an administrative consent before the api permissions are usable. Head over to the API Permissions section of the Azure AD App registration and click the "Grant admin consent for YourDirectory" button. Confirm by clicking "Yes".
![Create Certificate](/assets/posts/20210302-02/GrantAdminConsent.png){:class="img-responsive"}


The final step to be done on the app registration, is to add our app registration towards the Exchange Online administrative role in Azure Active Directory. We can assign the "Exchange Administrator" role in the "Roles and administrators" blade of Azure Active Directory, so head over to your Azure AD portal > Roles and administrators > Exchange administrator and click the "Add assignments button". Here you search for your created app and add it as an "Exchange Administrator"
button. Confirm by clicking "Yes".
![Create Certificate](/assets/posts/20210302-02/ExchangeAdminRoleAssignment.png){:class="img-responsive"}


## Azure Function
We now finally can switch our attentention towards the subject of this blog post, retreiving a certificate from Azure Keyvault an use it to authenticate against Exchange Online inside an Azure Function. So let's start by creating our Azure Function by executing the following Azure CLI commands:

```
# Create Storage Account for Function app
az storage account create -n savddesignlabexchfa1234 -g RG-AzureFunctionDemo -l westeurope --sku Standard_LRS

# Create Function App
az functionapp create --consumption-plan-location westeurope --name FA-VddesignlabExchFA1234 --os-type Windows --resource-group RG-AzureFunctionDemo --runtime powershell --storage-account savddesignlabexchfa1234 --functions-version 3
```
Important here is that we use a 64-Bit Azure Function app, this is needed as the Exchange Online modules are compiled for a 64-Bit architecture. A 32-Bit Azure function won't be able to load the Exchange Online V2 management modules. So the very first thing we do on our newly created function app, is to set it to 64 bit by going to the Azure Admin Portal > Resource Groups > YourResourceGroup > YourFunctionAppName > Configuration (under Settings) > Tab General Settings, change the platform setting towards "64 Bit" and save the change. This action will restart your function app.

![Create Certificate](/assets/posts/20210302-02/FunctionApp64Bit.png){:class="img-responsive"}


To be able to retrieve the certificate from our KeyVault, we first need to ensure that our function app can authenticate itself against Azure KeyVault. The advantage of using an Azure function is that we can assign it an identity and have Azure manage the credentials. For this use case, we are going  to use a system assigned managed identity that is tight towards the lifecycle of our Function App. This implies that when we remove the Function app, the managed identity is also automatically purged. To enable the managed identity, go to Azure Admin Portal > Resource Groups > YourResourceGroup > YourFunctionAppName > Identity (under Settings) > Tab System Assigned, change the status to "On" and save the change.

![Create Certificate](/assets/posts/20210302-02/SystemAssignedManagedIdentity.png){:class="img-responsive"}

The managed identity currently won't be able to retrieve any secrets from our Azure KeyVault as their is no KeyVault access policy defined for our managed identity. To create the access policy, we need to go back to our KeyVault configuration by going to Azure Admin Portal > Resource Groups > YourResourceGroup > YourKeyVaultName >  Access Policies (under Settings). Here we create a new access policy by selecting "+ Add Access Policy" 

To ensure the private key can be retrieved from our KeyVault, we need to select the "Get" and "List" permission under "Secret Permissions":

![Create Certificate](/assets/posts/20210302-02/KeyManagementPermissions.png){:class="img-responsive"}

Followed by the "Get" and "List" permission under "Certificate Permissions" to be able to retrieve the certificate:

![Create Certificate](/assets/posts/20210302-02/CertificatePermissions.png){:class="img-responsive"}

Finally, we need to search for our function app principal to apply the access policy. Search for the name you provided to your function app, select it and click "Add" to create the access policy.

![Create Certificate](/assets/posts/20210302-02/KeyVAultAccessPolicy.png){:class="img-responsive"}

Don't forget to hit the save button under your access policies to apply the access policies against your KeyVault.

Until now, we prepared our Azure environment to be able to use certificate authentication against Exchange Online. We finally can now start configuring an Azure function to authenticate against Exchange Online. The first item on our list is creating a function and loading the necessary modules. So we need to head over to our function Azure Portal > Resource Groups > YourResourceGroupName > YourAzureFunctionName >  Functions. Here we add a new function and select the "HTTP" trigger template, provide a name and select "Function" as "Authorization level":

![Create Certificate](/assets/posts/20210302-02/AZFunction-HttpTrigger.png){:class="img-responsive"}

Before we jump into the code of the function, we need to modify the functions dependencies and enlists which modules need to be loaded into the Azure Function. Through the portal, we need to do this with the "App Service Editor" which is currently in preview. You can find this under Azure Portal > Resource Groups > YourResourceGroupName > YourAzureFunctionName > App Service Editor (under Development Tools). Click the "Go" button to open the editor:

![Create Certificate](/assets/posts/20210302-02/AppServiceEditor.png){:class="img-responsive"}

Select the "requirements.psd1" file which contains the dependencies for the Azure Function. In this file, we need to add the "ExchangeOnlineManagent" version 2.0.4 module and set the AZ module to 5.5.0. Your "requirements.psd1" file should look like this now:

```
@{
    'ExchangeOnlineManagement' = '2.0.4'
    'az' = '5.5.0'
}
```
*Note: I fixed the versions to ensure that when new versions are released, we don't potentially automatically break our functions when the modules are updated to the latest versions.* 

Azure functions won't automatically manage dependencies enlisted in the requirements file and ignores the file, we therefore need to enable the Managed dependency feature inside the host.json file.  Open the file with the App Service Editor and ensure that "Enabled" is set to "true" under the managedDependency key:
```
  "managedDependency": {
    "Enabled": true
  }
```

During runtime of the the PowerShell function, you must download and temporary store the private key so that you can proof your identity against Exchange Online. This happens using a "System.Security.Cryptography.X509Certificates.X509Certificate2" object, which will store the private key under the user profile. An Azure function by default does not load the user profile during runtime for performance reasons. To avoid file not found errors in our code, we must configure the Azure function to load the user profile by configuring the WEBSITE_LOAD_USER profile application setting. You can add this under Azure Admin Portal > Resource Groups > YourResourceGroup > YourFunctionAppName > Configuration (under Settings) > Tab Application Settings by clicking the 'New application setting' button. Here you provide the 'WEBSITE_LOAD_USER_PROFILE' name and set the value to '1', confirm by clicking 'OK' and save your new application settings:

![Create Certificate](/assets/posts/20210302-02/AZFunction-LoadUserProfile.png){:class="img-responsive"}

We are now ready to put some code inside the Azure Function and do a first test run in retrieving data. So head over to your Azure Function via Azure Portal > Resource Groups > YourResourceGroupName > YourAzureFunctionName > Functions > YourHttpTriggerName > Code + Test. Here you can paste in the following code:

```
using namespace System.Net

# Input bindings are passed in via param block.
param($Request, $TriggerMetadata)

# Write to the Azure Functions log stream.
Write-Host "PowerShell HTTP trigger function processed a request."

$body = "This HTTP triggered function executed successfully."

Write-Host "Retrieving Exchange Online Management certificate from KeyVault"
$Cert = Get-AzKeyVaultCertificate -VaultName "YourVaultName" -Name "ExchangeOnlineManagement"
$PrivateKey = Get-AzKeyVaultSecret -VaultName "YourVaultName" -Name $Cert.Name -AsPlainText
$secretByte = [Convert]::FromBase64String($PrivateKey)
$x509Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($secretByte, "", "PersistKeySet")


Write-Host "Connecting to Exchange Online"
Connect-ExchangeOnline -AppId "YourApplicationID" -Certificate $x509Cert -Organization "Yourtenant.onmicrosoft.com" -Showbanner:$false

Write-Host "Connected to Exchange Online, retrieving mailbox list"
$body = get-mailbox | Select-object PrimarySMTPAddress,Identity

Write-Host "Disconnecting from Exchange Online"
Disconnect-ExchangeOnline -Confirm:$false

Write-Host "Returning mailbox list in HTTP body"
Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::OK
    Body = $body
})
```

Inside the code we do the following actions:

1. Fetch the ExchangeOnlineManagement certificate from KeyVault ($Cert = Get-AzKeyVaultCertificate ...)
2. Fetch the private key from KeyVault ($PrivateKey = Get-AzKeyVaultSecret...)
3. Convert the private key into a byte array ($SecretByte = [Convert]...)
4. Create an X509 certificate object from the private key ($X509Cert = New-Object...)
5. Connect towards Exchange Online using the X509 object (Connect-ExchangeOnline ...)
6. Retrieve a list of Exchange Online mailboxes ($body = get-mailbox) and minimize information towards Primary SMTP address and the Identity
7. Disconnect from Exchange Online, we can only have 3 simultaneous sessions so we need to properly disconnect.
8. Add the mailbox list towards the Azure's Function response


When you test the function, you should have the output of get-mailbox as a JSON object in your output:

![Create Certificate](/assets/posts/20210302-02/AZFunction-Output.png){:class="img-responsive"}

In the example above we have hard coded variables, this should be avoided at all cost. So let's clean up our code and define our variables as application settings:

| Application Setting | Description |
| ----------- | ----------- |
| EXO_KeyVaultName  | your KeyVault Name |
| EXO_KeyVaultKey | your certificate key name inside KeyVault |
| EXO_AzureADAppID | your application ID in Azure Active Directory |
| EXO_OrgID | your Exchange Online organization ID 'tenantName.onmicrosoft.com' |

These application settings are available inside our PowerShell function as environmental variables and can be used by calling adding $env:ApplicationSettingID in our code. This results in the following final code:

```
using namespace System.Net

# Input bindings are passed in via param block.
param($Request, $TriggerMetadata)

# Write to the Azure Functions log stream.
Write-Host "PowerShell HTTP trigger function processed a request."

$body = "This HTTP triggered function executed successfully."

Write-Host "Retrieving Exchange Online Management certificate from KeyVault"
$cert = Get-AzKeyVaultCertificate -VaultName $env:EXO_KeyVaultName -Name $env:EXO_KeyVaultKey
$secret = Get-AzKeyVaultSecret -VaultName $env:EXO_KeyVaultName -Name $cert.Name -AsPlainText
$secretByte = [Convert]::FromBase64String($secret)
$x509Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($secretByte, "", "PersistKeySet")

Write-Host "Connecting to Exchange Online"
Connect-ExchangeOnline -AppId $env:EXO_AzureADAppID -Certificate $x509Cert -Organization $env:EXO_OrgID -Showbanner:$false

Write-Host "Connected to Exchange Online, retrieving mailbox list"
$body = get-mailbox | Select-object PrimarySMTPAddress,Identity

Write-Host "Disconnecting from Exchange Online"
Disconnect-ExchangeOnline -Confirm:$false

# Associate values to output bindings by calling 'Push-OutputBinding'.
Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::OK
    Body = $body
})
```







# Conclusion

Todo 

Update on MD file