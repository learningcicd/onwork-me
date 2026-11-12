{% raw %}
```yaml

```
{% endraw %}

```bash
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "2.0.0.1",
    "parameters": {
        "principalName": {
            "value": "rxi-flag-api-e2e-02-identity",
            "metadata": {
                "description": "[REQUIRED] https://dev.azure.com/WBA/IT%20Services/_git/ccp-ent-infra-build?path=/library/role-assignment-service/PARAMETERS.md&version=GBmaster&_a=preview&anchor=%60parameters.principalname%60"
            }
        },
        "principalId": {
            "value": "dd45624b-7fc2-4cf9-904d-b74b0ee34daa",
            "metadata": {
                "description": "[REQUIRED] https://dev.azure.com/WBA/IT%20Services/_git/ccp-ent-infra-build?path=/library/role-assignment-service/PARAMETERS.md&version=GBmaster&_a=preview&anchor=%60parameters.principalId%60"
            }
        },
        "resourceRoleAssignments": {
            "value": [
                {
                    "resourceGroup": "rxr-rxi-nprod-cus-rg-functions",
                    "resourceType": "Microsoft.Storage/storageAccounts",
                    "resourceName": "rxrrxie2e02cusfuncstrg01",
                    "roleDefinitionNames": [
                        "Storage Table Data Reader"
                    ]
                }
            ],
            "metadata": {
                "description": "[REQUIRED] https://dev.azure.com/WBA/IT%20Services/_git/ccp-ent-infra-build?path=/library/role-assignment-service/PARAMETERS.md&version=GBmaster&_a=preview&anchor=%60parameters.resourceRoleAssignments%60"
            }
        }
    },
    "deploymentParameters": {
        "subscription": "rpu-nprod-rxrenewal-05",
        "template": "role-assignment-service"
    }
}
```
