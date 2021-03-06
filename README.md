# Cognitive Services

This repository contains ARM templates and/or (PowerShell) code for deploying Azure Cognitive Services in a test environment and play around with the configuration.

The aim is to deploy the services with security settings in place, such as restrictive network settings or keys stored in key vaults.

---
**Please Note**

None of the templates and samples in this repository should be used in production environments. The intention is to learn stuff and provide a starting point for testing deployments.

---

## Deployment

A few details are needed for executing a resource group deployment based on this template:

* Tenant ID
* Object ID of the user running the deployment
* Public IP Address of the host that is running the deployment (there are several ways to find this out, i.E. `curl -4 ifconfig.co` )

The values can be either put into the azuredeploy.parameter.json file or added when executing the deployment from PowerShell (see deploy.ps1 for a sample):

```azurepowershell
New-AzResourceGroupDeployment -Name $deploymentName `
                              -ResourceGroupName $resourceGroupName `
                              -location $location `
                              -Mode Incremental `
                              -TemplateFile .\azuredeploy.json `
                              -TemplateParameterFile .\azuredeploy.parameter.json `
                              -keyVaultName $keyVaultName `
                              -keyVaultTenant $tenantId `
                              -keyVaultAccessPolicies $keyVaultPolicies `
                              -keyVaultNetworkAcls $keyVaultNetworkAcls `
                              -csName $cognitiveServicesName `
                              -secretName1 $keyName1 `
                              -secretName2 $keyName2
```

## Test the deployment

Using PowerShell and the REST API endpoint, we can test whether this works.

### Test the Cognitive Service through REST API

1. First, we need to get the Cognitive Services Endpoint.
    
    ```azurepowershell
    $csUri = (Get-AzCognitiveServicesAccount -Name $cognitiveServicesName -ResourceGroupName $resourceGroupName).Endpoint+"text/analytics/v3.0/languages"
    ```

2. Then, create the header. For this, we need to get the access key from the key vault.

    ```azurepowershell
    $accessKey = Get-AzKeyVaultSecret -VaultName $keyVaultName -Name $keyName1 -AsPlainText

    $headers = @{
        "Content-Type"              = "application/json"
        "Ocp-Apim-Subscription-Key" = $accessKey
    }
    ```

3. As next step, create the body.

    ```azurepowershell
    $body = @{
        documents = @(
            @{
                id = 1
                text = "Hello"
            } 
        )
    } | ConvertTo-Json
    ```

4. And lastly, execute the REST request.

    ```azurepowershell
    Invoke-RestMethod -Method Post -Uri $csUri -Headers $headers -Body $body | ConvertTo-Json -Depth 3
    ```

    The response should display the detected language.

    ```azurepowershell
    {
      "documents": [
        {
          "id": "1",
          "detectedLanguage": {
            "name": "English",
            "iso6391Name": "en",
            "confidenceScore": 1.0
          },
          "warnings": []
        }
      ],
      "errors": [],
      "modelVersion": "2021-11-20"
    }
    ```

We have now proven that we can execute the request without storing the access key within code.
Please note, that in a real-world scenario we would use a service principal or managed identity, provide it access to they keys in the key vault (`Get` and `List` permissions against Secrets should be sufficient) and use the SPN to query the REST API.

## Rotate the keys

When rotating the keys, we need to make sure that the app is using the key that is not being rotated so that operations are not affected.

1. Create first key and store it in the Key Vault
    
    ```azurepowershell
    $newKey1 = (New-AzCognitiveServicesAccountKey -KeyName Key1 -ResourceGroupName $resourceGroupName -Name $cognitiveServicesName).Key1 | ConvertTo-SecureString -AsPlainText -Force
    Set-AzKeyVaultSecret -VaultName $keyVaultName -Name $keyName1 -SecretValue $newKey1
    ```

2. Change application to use the first key 
3. and rotate the second key

    ```azurepowershell
    $newKey2 = (New-AzCognitiveServicesAccountKey -KeyName Key2 -ResourceGroupName $resourceGroupName -Name $cognitiveServicesName).Key2 | ConvertTo-SecureString -AsPlainText -Force
    Set-AzKeyVaultSecret -VaultName $keyVaultName -Name $keyName2 -SecretValue $newKey2
    ```
