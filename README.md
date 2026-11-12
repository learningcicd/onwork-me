{% raw %}
```yaml

```
{% endraw %}

```bash
{
  "contentVersion": "1.0.0.1",
  "parameters": {
    "principalName": {
      "value": "rxr-rxi-prod-01-cus-adf"
    },
    "principalId": {
      "value": "4e95ba3f-f8c1-4749-99a7-73f712c18283"
    },
    "resourceRoleAssignments": {
      "value": [
        {
          "resourceGroup": "rxr-rxi-prod-cus-FunctionApps-rg",
          "resourceType": "Microsoft.Storage/storageAccounts",
          "resourceName": "rxrrxiprodcusfuncstrg05",
          "roleDefinitionNames": [
            "Storage Table Data Reader"
          ]
        }
      ]
    }
  },
  "deploymentParameters": {
    "subscription": "rpu-prod-rxrenewal-05",
    "template": "role-assignment-service"
  }
}
```

```
az account set --subscription "rpu-prod-rxrenewal-05"
az deployment sub create \
  --location centralus \
  --template-file role-assignment-service.json \
  --parameters @prodadfroleassignment.json
```
