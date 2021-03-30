---
layout: post
title: "Bash: Azure CLI Azure AD role assignments"
excerpt: "To be written"
date: 2021-03-30
---



For a customer, I recently needed to create a bash script that prepared their Azure environment for Terraform. Amongst Azure resources, the script also created service principals for Azure DevOps pipelins that would run Terraform. The Terraform code in Azure DevOps included the creation/destruction of Azure AD groups using the AzureAD provider. [The documentation for managing groups](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/group) using of this provider states that the user account running Terraform needs to be a member of the 'User Account Administrator' role to be able to delete created groups. 

The bash script was using Azure CLI to create the different resources in Azure. I intended to use the "az role assignment create" command to setup the nescessary permissions on the Service Principal, however I quickly discovered this was not possible inside Azure CLI for Azure Active Directory Roles and needed to find a solution as PowerShell was not an option.

### Microsoft Graph API
After some research, I discovered in the Microsoft Graph Beta API that you can assign custom roles. The web page itself documented the needed URL's and parameters, but did not include examples on how to use them. The web page itself can be found here: https://docs.microsoft.com/en-us/azure/active-directory/roles/custom-assign-graph

Within Azure CLI, you can execute rest methods using the [az rest](https://docs.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest#az_rest) command. So I embarqued to assign the required permissions using this command inside bash.

First, let's start to retreive a list of roles inside our Azure Active Directory. So start up bash and login into Azure CLI using 'az login --use-device-code', then we can execute the following bash command to retreive our Azure AD roles:

```bash
    URI=$(echo "https://graph.microsoft.com/v1.0/directoryRoles");
    az rest --method GET --uri $URI --header Content-Type=application/json 
```

Once executed, you get a JSON object which contains activated AD roles in our tenant:
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#directoryRoles",
  "value": [
    {
      "deletedDateTime": null,
      "description": "Can manage all aspects of Azure AD and Microsoft services that use Azure AD identities.",
      "displayName": "Global Administrator",
      "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "roleTemplateId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    },
    {
      "deletedDateTime": null,
      "description": "Can read basic directory information. Commonly used to grant directory read access to applications and guests.",
      "displayName": "Directory Readers",
      "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "roleTemplateId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    },
    {
      "deletedDateTime": null,
      "description": "Can manage all aspects of the Exchange product.",
      "displayName": "Exchange Administrator",
      "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "roleTemplateId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    },
    {
      "deletedDateTime": null,
      "description": "Device Administrators",
      "displayName": "Azure AD Joined Device Local Administrator",
      "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "roleTemplateId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  ]
}
```
JSON format is fine for a human eye to read (when not too complex :-)). However when we want to use a value like the ID to actually grant a role assignment, we need to be able to parse the JSON file. The tool JQ is able to  slice, filter, map and transform JSON, so we are going to use this to parse our response from the Graph API. To filter out our displayName, we need to pipe our JSON output into two jq commands:



```bash
URI=$(echo "https://graph.microsoft.com/v1.0/directoryRoles");
az rest --method GET --uri $URI --header Content-Type=application/json | jq '.value[].displayName' -r
```

Because our roles are returned in a value array, we first need to step into the value array with .value[]. Afterwards, we ask for the displayName of each entry and output in raw format (-r) which results in a list of active roles:
![Create Certificate](/assets/posts/20210331-03/GetRoles.PNG){:class="img-responsive"}

When we look at the documentation to add a role assignment towards a user, the Microsoft Graph API expects the following parameters:
* PrincipalID: the object ID of the user we want to add to a role group
* roleDefinitionId: the definition ID of the role we want to assign
* directoryScopId: an optional scope for the role assignment (if you want to limit the role assignment towards a specific administrative unit)

We can use Azure CLI commands and the rest methods to retreive these values:
```bash
# Retreive the user ID for a given user principal name
user_objectID=$(az ad user list --upn "demo.user@domain.onmicrosoft.com" --query [].objectId -o tsv)




```


