---
layout: post
title: "Bash: Azure CLI Azure AD role assignments"
excerpt: "Using Azure CLI in bash to add Azure AD role assignments"
date: 2021-03-31
---



For a customer, I recently needed to create a bash script that prepared their Azure environment for Terraform. Amongst Azure resources, the script also created service principals for Azure DevOps pipelins that would run Terraform. The Terraform code in Azure DevOps included the creation/destruction of Azure AD groups using the AzureAD provider. [The documentation for managing groups](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/group) using of this provider states that the user account running Terraform needs to be a member of the 'User Account Administrator' role to be able to delete created groups. 

The bash script was using Azure CLI to create the different resources in Azure. I intended to use the "az role assignment create" command to setup the nescessary permissions on the Service Principal, however I quickly discovered this was not possible inside Azure CLI for Azure Active Directory Roles and needed to find a solution as PowerShell was not an option.

### Microsoft Graph API
After some research, I discovered in the Microsoft Graph Beta API that you can assign custom roles. The web page itself documented the needed URL's and parameters, but did not include examples on how to use them. The web page itself can be found here: https://docs.microsoft.com/en-us/azure/active-directory/roles/custom-assign-graph

Within Azure CLI, you can execute rest methods using the [az rest](https://docs.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest#az_rest) command. So I embarqued to assign the required permissions using this command inside bash.

First, let's start to retreive a list of roles inside our Azure Active Directory. So start up bash and login into Azure CLI using 'az login --use-device-code', then we can execute the following bash command to retreive our Azure AD roles:

```bash
    URI=$(echo "https://graph.microsoft.com/beta/directoryRoles");
    az rest --method GET --uri $URI --header Content-Type=application/json 
```

Once executed, you get a JSON object which contains activated AD roles in our tenant:
```json
{
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#directoryRoles",
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
URI=$(echo "https://graph.microsoft.com/beta/directoryRoles");
az rest --method GET --uri $URI --header Content-Type=application/json | jq '.value[].displayName' -r
```

Because our roles are returned in a value array, we first need to step into the value array with .value[]. Afterwards, we ask for the displayName (.displayName) of each entry and output in raw format (-r) which results in a list of active roles:
![Get Roles](/assets/posts/20210331-03/GetRoles.PNG){:class="img-responsive"}

### Add Role Assignment

When we look at the Microsoft documentation to add a role assignment towards a user, the Microsoft Graph API expects the following parameters:
* PrincipalID: the object ID of the user we want to add to a role group, this is the GUID of the Azure AD user
* roleDefinitionId: the definition ID of the role we want to assign. This is the ID of the directory role which we retreived in very first rest command of this article.
* directoryScopId: an optional scope for the role assignment. You can provide this if you need to limit the role assignment towards a specific administrative unit.

We can use Azure CLI commands and the rest methods to retreive these values. To retreive the GUID of the Azure AD user, we use the standard 'az ad user list' command and store the objectID parameter in a bash variable:
```bash
# Retreive the user ID for a given user principal name and display to console
user_objectID=$(az ad user list --upn "demo.user@xxxxxxxx.OnMicrosoft.com" --query [].objectId -o tsv)
echo $user_objectID
```

I started the article with a list of activated directory roles in Azure AD, to retreive the role definition ID from we will need to filter our results. With JQ we can furter filter our JSON resultset to get the ID of a specific role. For example, the following command will only provide the information of the "Global Administrator" role:
```bash
# Retreive the Global Administrator role
URI=$(echo "https://graph.microsoft.com/beta/directoryRoles");
az rest --method GET --uri $URI --header Content-Type=application/json | jq '.value[] | select(.displayName | contains("Global Administrator"))'
```

Result of fetching the "Global Administrator" role:
![Get Global Admin Role](/assets/posts/20210331-03/GetGlobalAdmin.png){:class="img-responsive"}

JQ by default outputs JSON, so we can pipe it towards another JQ instance to retreive the ID of the role in a raw format (-r):
```bash
# Retreive the Global Administrator role
URI=$(echo "https://graph.microsoft.com/beta/directoryRoles");
az rest --method GET --uri $URI --header Content-Type=application/json | jq '.value[] | select(.displayName | contains("Global Administrator"))' | jq '.id' -r
```

Now all we need to do on the roleDefinitionId is to make is to fine tune our commands to make it more generic and use variables. Our goal is to use this in an script so we want to avoid hard coding names like the 'Global Administrator' name inside our code. JQ supports input arguments (--arg) which present themselfs as variables that can be used in your filter statements.  For example: when we specify the displayName argument, the value can be used inside our select statement using $displayName. Let's refactor our code to make it re-usable:

```bash
user_role="Global Administrator"
# Retreive the role provided in user_role
URI=$(echo "https://graph.microsoft.com/beta/directoryRoles");
roleDefinitionId=$(az rest --method GET --uri $URI --header Content-Type=application/json | jq --arg displayName "${user_role}" '.value[] | select(.displayName | contains($displayName))' | jq '.id' -r)
# Display Result
echo $roleDefinitionId
```

Executing the code above results in a plain text role definition ID stored in a variable that we can use to add a role assignment:
![Get Global Admin Role refactored](/assets/posts/20210331-03/GetGlobalAdminRefactor.png){:class="img-responsive"}

We now have all the information required to create a role assignment using the roleAssignments enpoint. To provide the Graph API the required information, we need to pass our information as a JSON object into the body of our API call. The API expects the following JSON structure:

```json
{
    "principalId":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "roleDefinitionId":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "directoryScopeId":"/" 
}
```

We can use JQ to construct the expected body where we pass the information into our JQ statement using arguments. 
```bash
BODY=$( jq -n \
  --arg principalId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  --arg roleDefinitionId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  --arg directoryScopeId "/" \
  '{principalId: $principalId, roleDefinitionId: $roleDefinitionId, directoryScopeId: $directoryScopeId}'
)
echo $BODY  
```
The result is the expected JSON format stored in a variable:
![Add Role Assignment JSON body](/assets/posts/20210331-03/AddRoleAssignmentJsonBody.PNG){:class="img-responsive"}

All we have to do now is to refactor our code to use our variables and post the information towards the Graph API endpoint by executing:

```bash
URI=$(echo  "https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments")
BODY=$( jq -n \
            --arg principalId "$user_objectID" \
            --arg roleDefinitionId "$roleDefinitionId" \
            --arg directoryScopeId "/" \
        '{principalId: $principalId, roleDefinitionId: $roleDefinitionId, directoryScopeId: $directoryScopeId}' )  
az rest --method POST --uri $URI --header Content-Type=application/json --body "$BODY"
```
On success, you get the role assignment information back in JSON format:
```json
{
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#roleManagement/directory/roleAssignments/$entity",
  "directoryScopeId": "/",
  "id": "5xxx_mJe20exxXXxxJo4sTXX_xxxxxxxxxxxxxxxx_k-1",
  "principalId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "resourceScope": "/",
  "roleDefinitionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```
When navigating to the Global Administrators role in Azure AD, we see our demo user has been added as a global administrator:
![Azure AD Role Assignments](/assets/posts/20210331-03/RoleAssignments.png){:class="img-responsive"}

### Remove Role Assignment

Now that we have seen how to add a role assignment using Azure CLI commands, let's have a quick look how to remove them again. According to the Microsoft documentation, we can use the following Graph API endpoint to remove a role assignment:

```
DELETE https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments/lAPpYvVpN0KRkAEhdxReEJC2sEqbR_9Hr48lds9SGHI-1
```

The documentation is not that extensive, but the Graph API expects that you pass the role assignment ID in the URL using a DELETE action. To do this, we first need to retreive the role assignment ID, so back to our bash commands. We can retereive the different role assignments by making GET call towards the roleAssignments endpoint, however we need to pass in a filter to retreive only the role assignments for our user. So we filter on the principal ID which we already retreived in previous steps:

```bash
# Retreive Object ID
user_objectID=$(az ad user list --upn "demo.user@xxxxxxxx.OnMicrosoft.com" --query [].objectId -o tsv)

#Retreive role assignments
URI=$(echo "https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments?\$filter=principalId%20eq%20'${user_objectID}'")
az rest --method GET --uri $URI --header Content-Type=application/json
```

![Azure AD Role Assignments](/assets/posts/20210331-03/GetAssignedRoles.png){:class="img-responsive"}

The result is a list of all role assignments the user has and needs to be filtered so we can have the ID attribute of the role assignment. The main caveat here is that the roleDefinitionId on the user role assignment list does not match the ID we get from the directoryRoles endpoint, instead it enlists the roleTemplateId:

![Get Global Admin Role](/assets/posts/20210331-03/IDMismatch.png){:class="img-responsive"}


So when we construct our filter, we need to pass in the roleTemplateId. So we need to retreive it first using the following code:
```bash
user_role="Global Administrator"
# Retreive the role provided in user_role
URI=$(echo "https://graph.microsoft.com/beta/directoryRoles");
roleTemplateId=$(az rest --method GET --uri $URI --header Content-Type=application/json | jq --arg displayName "${user_role}" '.value[] | select(.displayName | contains($displayName))' | jq '.roleTemplateId' -r)
# Display Result
echo $roleTemplateId
```
Afterwards, we can filter our roleAssignments using and store the resulting ID in a variable:
```bash
URI=$(echo "https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments?\$filter=principalId%20eq%20'${user_objectID}'")
roleAssignmentId=$(az rest --method GET --uri $URI --header Content-Type=application/json | jq --arg roleTemplateId ${roleTemplateId} '.value[] | select(.roleDefinitionId | contains($roleTemplateId))' | jq '.id' -r)
echo "$roleAssignmentId"
```

Finally, we remove the role assignment by call the Graph API:
```bash
URI=$(echo "https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments/${roleAssignmentId}")
az rest --method DELETE --uri $URI --header Content-Type=application/json
```

Note: uppon success, the command does not return a result. So if you want to check out the result, you need to go to the Azure AD portal and verify.

### Script
I've combined all the principles off this article into a Bash script that is hosted in my Github repository [changeRoleassignment.sh](https://github.com/dVerschaeve/Linux.Helpers/blob/main/Bash/changeRoleassignment.sh). The script works with input parameters and supports both adding a role assignment and removing a role assignment. Expected parameters are:

* Action (Mandatory, possible values: add or remove)
* User Principal Name (Mandatory)
* Role Assignment Display Name (Mandatory)
* Sope (Optional)

To add a role assignment for a given user:
```bash
changeRoleAssignments.sh add demo.user@xxxxxxxx.OnMicrosoft.com "Global Administrator"
```

To remove a role assignment for a given user:
```bash
changeRoleAssignments.sh remove demo.user@xxxxxxxx.OnMicrosoft.com "Global Administrator"
```

### Conclusion
I hope you can use the learnings of this article to solve some of your automation tasks. For me it was quite challenging as I don't do bash scripting often. It's also based on the beta endpoints of the Microsoft Graph API, so documentation was not that extensive, support is limited and it can be modified by Microsoft. During creation of the code, I've also noticed errors in the Microsoft documentation. I've reported them through a Github issue and was quite pleased that my suggestions were incorporated into the document quite quickly. 

Fingers crossed that Azure CLI will support adding Azure AD role assignments in the future like it's the case with PowerShell, untill then feel free to use the knowledge and script of this article.