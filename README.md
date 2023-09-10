# Using User Assigned Managed Identity (UAMI) to Read Key Vault Secrets

**Steps:**

## 1.	Create Identity
### First, define the UAMI and the resource group:
```
$vmname='winvm-new' 
$uamiName = "UID1"  
$resourceGroup = "rg01"  
$subscriptionID = "<sub-id>"  
$keyVaultName = "kv01"  
```
### Then, create the UAMI:
```
New-AzUserAssignedIdentity -ResourceGroupName $resourceGroup -Name $uamiName
```
## 2.	Assign Identity to VM
### Bind the created UAMI to a VM:
```
$vm = Get-AzVM -ResourceGroupName $resourceGroup -Name $vmname

Update-AzVM -ResourceGroupName $resourceGroup -VM $vm -IdentityType "UserAssigned" `
-IdentityID "/subscriptions/$subscriptionID/resourcegroups/$resourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$uamiName"
```
## 3.	Retrieve UAMI Principal ID
### Fetch the principal ID for the UAMI:
```
$uami = az identity show --name $uamiName --resource-group $resourceGroup `
    --query principalId --output tsv  
echo $uami 
```
## 4.	Grant Key Vault Permissions for UAMI
### Assign necessary permissions to access the Key Vault secrets:
```
az keyvault set-policy --name $keyVaultName --object-id $uami `
    --secret-permissions get list --key-permissions get list 
```
### Ensure the permissions are correctly set:
```
az keyvault show --name $keyVaultName --query "properties.accessPolicies"
```
## 5.	Acquire Access Token for UAMI
### Fetch an access token using the UAMIs Client ID:
```
$clientID = az identity show --name $uamiName `
    --resource-group $resourceGroup --query clientId --output tsv

$baseUri = 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net&client_id='

$uri = $baseUri + $clientID  
echo $uri
```

### Invoke GET to query the Azure Instance Metadata Service (IMDS) to obtain the access token:
```
$Response = Invoke-RestMethod -Uri $uri -Method GET -Headers @{Metadata="true"}

$KeyVaultToken = $Response.access_token  
echo $KeyVaultToken
```
## 6.	Retrieve Secret from Key Vault
### Finally, use the acquired token to fetch secrets from the Key Vault:
```
$secretName = 'secret1' 
$apiVersion = "2016-10-01" 
$baseUri = "https://$keyVaultName.vault.azure.net/secrets/" 

$uri = $baseUri + $secretName + "?api-version=" + $apiVersion 

Invoke-RestMethod -Uri $uri `
-Method GET -Headers @{Authorization="Bearer $KeyVaultToken"} 
```

# Using System Assigned Managed Identity (SAMI) to Read Key Vault Secrets

**Steps:**

## 1.	Enable SAMI on VM
### Activate the system-assigned managed identity for your Azure VM:
```
$vmname='winvm-new'  
$resourceGroup = "rg01"  
$keyVaultName = "kv01"  

$vm = Get-AzVM -ResourceGroupName $resourceGroup -Name $vmname 

Update-AzVM -ResourceGroupName $resourceGroup -VM $vm `
    -IdentityType SystemAssigned 
```
## 2.	Retrieve SAMI Principal ID
### Get the principal ID for SAMI:
```
$identity = az vm identity show --name $vmname `
    --resource-group $resourceGroup --query principalId `
    --output json | ConvertFrom-Json  
echo $identity 
```
## 3.	Grant the "Key Vault Reader" Role to SAMI
### Assign the required permissions:
```
$keyVaultId = (az keyvault show --name $keyVaultName --query id --output tsv) 

az role assignment create --assignee $identity --role "Key Vault Reader" `
    --scope $keyVaultId 
```
## 4.	Create Access policy for SAMI:
### Assign necessary permissions to access the Key Vault secrets:
```
az keyvault set-policy --name $keyVaultName --object-id $identity `
    --secret-permissions get list --key-permissions get list 
```
### Ensure the permissions are correctly set:
```
az keyvault show --name $keyVaultName --query "properties.accessPolicies"
```
## 5.	Acquire Access Token for SAMI
### For SAMI, you dont need a client ID:
```
$Response = Invoke-RestMethod -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net' -Method GET -Headers @{Metadata="true"} 

$KeyVaultToken = $Response.access_token  
echo $KeyVaultToken 
```
## 6.	Retrieve Secret from Key Vault with SAMI
### Fetch the secret:
```
$baseUri = "https://$keyVaultName.vault.azure.net/secrets/" 

$uri = $baseUri + $secretName + "?api-version=" + $apiVersion 

Invoke-RestMethod -Uri $uri -Method GET `
    -Headers @{Authorization="Bearer $KeyVaultToken"}
```
