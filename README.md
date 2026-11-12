{% raw %}
```yaml

```
{% endraw %}

```bash
{
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
Your file has a custom deploymentParameters block, so it can’t go straight to az deployment. Run this in your pwsh terminal: it reads the file, sets the subscription, pulls out the parameters section into a temp file, and deploys.

powershell
```

$cfg = Get-Content .\prodadfroleassignment.json -Raw | ConvertFrom-Json

az account set --subscription $cfg.deploymentParameters.subscription

$tmp = New-TemporaryFile
@{ parameters = $cfg.parameters } | ConvertTo-Json -Depth 10 | Set-Content $tmp

az deployment sub create `
  --location centralus `
  --template-file ".\$($cfg.deploymentParameters.template).json" `
  --parameters "@$tmp"

Remove-Item $tmp
```
