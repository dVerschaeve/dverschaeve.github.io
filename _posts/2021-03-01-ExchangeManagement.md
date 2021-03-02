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
Time to move on towards preparing Azure Active Directory where an app registration must be done that will be used to authenticate unattended against the Exchange Online management endpoints. 

```CLI 
az ad app create --display-name AZF-ExchangeOnlineMGT --native-app
```

## 

# Conclusion

Todo 

Update on MD file