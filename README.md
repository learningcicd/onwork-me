```
#!/bin/bash

# Define your two non-prod subscriptions
SUBSCRIPTIONS=(
  "your-subscription-id-or-name-1"
  "your-subscription-id-or-name-2"
)

MATCH_COUNT=0

for SUBSCRIPTION in "${SUBSCRIPTIONS[@]}"; do
  echo ""
  echo "========================================================"
  echo "Switching to subscription: $SUBSCRIPTION"
  echo "========================================================"
  az account set --subscription "$SUBSCRIPTION"

  # Get all function apps in this subscription
  FUNCTION_APPS=$(az functionapp list --query "[].{name:name, rg:resourceGroup}" -o json)

  APP_COUNT=$(echo "$FUNCTION_APPS" | jq length)
  echo "Found $APP_COUNT function app(s) in this subscription."
  echo ""

  # Loop through each function app
  echo "$FUNCTION_APPS" | jq -c '.[]' | while read -r app; do
    NAME=$(echo "$app" | jq -r '.name')
    RG=$(echo "$app" | jq -r '.rg')

    RUNTIME=$(az functionapp config appsettings list \
      --name "$NAME" \
      --resource-group "$RG" \
      -o json 2>/dev/null | \
      jq -r '.[] | select(.name == "FUNCTIONS_WORKER_RUNTIME") | .value')

    if [[ "$RUNTIME" == "dotnet" ]]; then
      echo "✅ MATCH: $NAME | RG: $RG | Sub: $SUBSCRIPTION"
    else
      echo "⬜ SKIP:  $NAME | RG: $RG |
```
