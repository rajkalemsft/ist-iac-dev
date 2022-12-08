# Authentication from GitHub to Azure

The recommended method of Azure login/authentication is with OpenId Connect using a Federated Identity Credential.
Please follow [this guide](https://learn.microsoft.com/azure/developer/github/connect-from-azure) to create the correct credential.

## Scripted Setup

This repository uses a script to provide a simple way to create a GitHub OIDC federated credential, it is based on the steps outlined [here](https://learn.microsoft.com/azure/developer/github/connect-from-azure).

The script will create a new application, assign the correct Azure RBAC permissions for the Subscription **OR** Resource Group containing your AKS cluster, and create Federated Identity Credentials for both an environment and branch.

- Update below values in the script,
        
        - APPNAME --> Azure AD application name to be created for workflow auth.
        - RG --> AKS cluster resource group name. Uncomment only if you are assigning permissions at the Resource Group level
        - GHORG --> Git hub Org name for the current repository
        - GHREPO --> GitHub repository name.
        - GHBRANCH --> GitHub branch name.
        - GHENV --> GitHub environment name from devspike

- Here's what to exepect on execution,
        - You will be asked to login to Azure.
        - Towards the end of the successful execution, you will be given three values which are used to configure environment on GitHub repository,
            1. AZURE_CLIENT_ID --> ClientId of the Azure AD application specified above (APPNAME)
            2. AZURE_TENANT_ID --> TenantId for Azure Subscription
            3. AZURE_SUBSCRIPTION_ID --> Azure subscription Id.

> The script requires AZ CLI >= 2.37

```bash
#Set up user specific variables
APPNAME=myApp
# RG=myAksClusterResourceGroup  # Uncomment only if you are assigning permissions at the Resource Group level
GHORG=Azure
GHREPO=aks-baseline-automation
GHBRANCH=main
GHENV=prod

# Login to your subscription. When prompted for credentials, select a user account who has permission to register applications in Azure AD 
az login --use-device-code 

# Create App/Service Principal
APP=$(az ad app create --display-name $APPNAME)
appId=$(echo $APP | jq -r ".appId"); echo $appId
applicationObjectId=$(echo $APP | jq -r ".id")
SP=$(az ad sp create --id $appId)
assigneeObjectId=$(echo $SP | jq -r ".id"); echo $assigneeObjectId

#Create Role Assignments (Azure Subscription level RBAC)
subscriptionId=$(az account show --query id -o tsv)
az role assignment create --role Owner --scope "/subscriptions/$subscriptionId" --assignee-object-id $assigneeObjectId --assignee-principal-type ServicePrincipal
az role assignment create --role "Azure Kubernetes Service RBAC Cluster Admin" --scope "/subscriptions/$subscriptionId" --assignee-object-id $assigneeObjectId --assignee-principal-type ServicePrincipal

#Create Role Assignments (Azure Resource Group level RBAC)
# Uncomment only if you are assigning permissions at the Resource Group level
#az role assignment create --role Owner --resource-group $RG --assignee-object-id $assigneeObjectId --assignee-principal-type ServicePrincipal
#az role assignment create --role "Azure Kubernetes Service RBAC Cluster Admin" --resource-group $RG --assignee-object-id  $assigneeObjectId --assignee-principal-type ServicePrincipal

#Create federated identity credentials for use from a GitHub Branch
fedReqUrl="https://graph.microsoft.com/beta/applications/$applicationObjectId/federatedIdentityCredentials"
fedReqBody=$(jq -n --arg n "$APPNAME-branch-$GHBRANCH" \
                   --arg r "repo:$GHORG/$GHREPO:ref:refs/heads/$GHBRANCH" \
                   --arg d "Access for GitHub branch $GHBRANCH" \
             '{name:$n,issuer:"https://token.actions.githubusercontent.com",subject:$r,description:$d,audiences:["api://AzureADTokenExchange"]}')
echo $fedReqBody | jq -r
az rest --method POST --uri $fedReqUrl --body "$fedReqBody"

#Create federated identity credentials for use from a GitHub Environment
fedReqUrl="https://graph.microsoft.com/beta/applications/$applicationObjectId/federatedIdentityCredentials"
fedReqBody=$(jq -n --arg n "$APPNAME-env-$GHENV" \
                   --arg r "repo:$GHORG/$GHREPO:environment:$GHENV" \
                   --arg d "Access for GitHub environment $GHENV" \
             '{name:$n,issuer:"https://token.actions.githubusercontent.com",subject:$r,description:$d,audiences:["api://AzureADTokenExchange"]}')
echo $fedReqBody | jq -r
az rest --method POST --uri $fedReqUrl --body "$fedReqBody"

#Create federated identity credentials for use from a GitHub PR
fedReqUrl="https://graph.microsoft.com/beta/applications/$applicationObjectId/federatedIdentityCredentials"
fedReqBody=$(jq -n --arg n "$APPNAME-pr" \
                   --arg r "repo:$GHORG/$GHREPO:pull_request" \
                   --arg d "Access for GitHub PR" \
             '{name:$n,issuer:"https://token.actions.githubusercontent.com",subject:$r,description:$d,audiences:["api://AzureADTokenExchange"]}')
echo $fedReqBody | jq -r
az rest --method POST --uri $fedReqUrl --body "$fedReqBody"

#Retrieving values needed for GitHub secret creation
clientId=$appId
tenantId=$(az account show --query tenantId -o tsv)

echo "Create these GitHub Environment secrets per these instructions: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment"
echo -e "AZURE_CLIENT_ID: $clientId\nAZURE_TENANT_ID: $tenantId\nAZURE_SUBSCRIPTION_ID: $subscriptionId"
```

## Troubleshooting

###  AADSTS70021: No matching federated identity record found for presented assertion.

This error will occur when the Assertion subject (the GitHub environment/branch/tag) does not have Federation Identity Credentials. You should double check that the specific environment or branch that the Action was using has Federated Identity Credentials for the Azure AD Application that was created.
