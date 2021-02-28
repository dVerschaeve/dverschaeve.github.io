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
az group create --name "RG-AzureFunctionDemo" -l "EuropeWest"
az keyvault create --name "kvvddesignlab1234" --resource-group "RG-AzureFunctionDemo" --location "EuropeWest"
```

The outcome of these commands is that an Azure Key Vault is created inside a resource group. When an Azure Key vault is created, you don't automatically gain the abilitiy to view and manage secrets inside the vault. To ensure our user ID can generate a certificate, we have to create an Access Policy that allows us the list certificates, retreive certificates and the ability to create certificates by executing the following CLI commands in Azure Cloud Shell:

```
export VAR_AZURE_USERID=$(az ad user show --id <Current Logged On User>)
az keyvault set-policy --name kvvddesignlab1234 --object-id $VAR_AZURE_USERID --certificate-permissions list get create
```

At this point we are ready to generate the certificate that will be used to authenticate against Exchange Online. For this purpose, we can generate a self-signed certificate inside KeyVault using by executing the following commands in Azure Cloud Shell:

```
az keyvault certificate create --vault-name "kvvddesignlab1234" -n ExchangeOnlineManagement -p "$(az keyvault certificate get-default-policy)"
```


## Prepare Azure Active Directory

## 

# Conclusion

Todo 

Update on MD file