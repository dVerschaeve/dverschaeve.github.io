---
layout: post
title: Exchange Management working title
excerpt: "ToDo"
date: 2021-03-01
---

# Introduction

The release of Exchange Management Shell V2.0.4 comes with support for MacOS and Linux. With certificate based authetentication against the Exchange Online management endpoints released in a previous version,  let's have a look how we can leverage dynamic retreival of a certificate from an Azure KeyVault and use it inside an Azure PowerShell function to connect towards Exchange Online to do some automated tasks. This article outlines the setup and discusses some caveats I encountered during the deployment in my lab environment.

# Main Content

Before we dive into the Azure Function, we first need to ensure that all the required components are configured in our environment. This means that we need to have an Azure KeyVault and an app registration inside Azure Active Directory before we can test out inside our Azure Function. The deployment in this article is done using Azure CLI inside Azure Cloud Shell, this allows you to quicly reproduce the steps inside this article and saves me time taking step by step screenshots.

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


## Prepare Azure Active Directory
Time to move on towards preparing Azure Active Directory where an app registration must be done that will be used to authenticate unattended against the Exchange Online management endpoints. We start with creating an app registration inside Azure Active Directory by executing the following Azure CLI command:

```CLI 
az ad app create --display-name AZF-ExchangeOnlineMGT --native-app
```

Once the command is executed, we need to upload the public portion of our certificate towards the App registrations "SectionToBeAdded" so that we can authenticate using the generated certificate.


The next configuration is allowing access to the Exchange Online Management API, we can configure this in the "API section" of our app registration. We require the following API permission to be configured:

<ToBeAdded>

As this is an application permission, we need to execute an administrative consent before the api permissions are usable.

The final step to be done on the app registration, is to add our app registration towards the Exchange Online administrative role in Azure Active Directory.

## Azure Function
We now finally can switch our attentention towards the subject of this blog post, retreiving a certificate from Azure Keyvault an use it to authenticate against Exchange Online inside an Azure Function. So let's start by creating our Azure Function by executing the following Azure CLI commands:

```
# Create Storage Account for Function app
az storage account create -n savddesignlabexchfa1234 -g RG-AzureFunctionDemo -l westeurope --sku Standard_LRS

# Create a Windows Based Service Plan
az appservice plan create -g RG-AzureFunctionDemo -n APP-ExchangeOnline --hyper-v --sku B1

# Create Function App
az functionapp create -g RG-AzureFunctionDemo  -p APP-ExchangeOnline -n FA-VddesignlabExchFA1234 -s savddesignlabexchfa1234 --functions-version 3 --runtime powershell --runtime-version 7.0
```
Important here is that we create a 64-Bit Azure Function app, this is needed as the Exchange Online modules are compiled for a 64-Bit architecture. A 32-Bit Azure function won't be able to load the Exchange Online V2 management modules.

Once Azure resource manager is finished creating the components, let's navigate towards our Azure Function App and make the nescessary configurations. So go to https://portal.azure.com > Resource Groups > YourResourceGroup > YourAzureFunctionApp. Here we need to do some modifications.








# Conclusion

Todo 

Update on MD file